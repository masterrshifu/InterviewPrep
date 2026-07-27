# Java Memory Management & Garbage Collection — Deep Dive & Interview Guide

> A single-source study guide covering the JVM memory model, the object lifecycle, every mainstream garbage collector, GC pressure, and how to diagnose GC-induced CPU overhead in production. Written to be read end-to-end for learning **and** skimmed as an interview cheat sheet.

---

## Table of Contents

1. [How to use this guide](#0-how-to-use-this-guide)
2. [The JVM runtime memory model](#1-the-jvm-runtime-memory-model)
3. [The object lifecycle & allocation path](#2-the-object-lifecycle--allocation-path)
4. [Reachability, GC roots & reference types](#3-reachability-gc-roots--reference-types)
5. [Core GC algorithms](#4-core-gc-algorithms)
6. [The generational heap](#5-the-generational-heap)
7. [The collectors, one by one](#6-the-collectors-one-by-one)
8. [Safepoints, stop-the-world & GC threads](#7-safepoints-stop-the-world--gc-threads)
9. [GC pressure & allocation rate](#8-gc-pressure--allocation-rate)
10. [Diagnosing GC CPU overhead in production](#9-diagnosing-gc-cpu-overhead-in-production)
11. [Reading GC logs](#10-reading-gc-logs)
12. [Tuning playbook & JVM flags](#11-tuning-playbook--jvm-flags)
13. [Memory leaks & OutOfMemoryError](#12-memory-leaks--outofmemoryerror)
14. [Off-heap & native memory](#13-off-heap--native-memory)
15. [Interview question bank](#14-interview-question-bank)
16. [Rapid-fire flashcards](#15-rapid-fire-flashcards)
17. [Glossary](#16-glossary)

---

## 0. How to use this guide

Each concept is presented so you can both learn it and answer on it. If you only have an hour before an interview, read sections 1, 5, 6, 8, 9, then the question bank (14) and flashcards (15).

Numbers, defaults, and flag names are given for **HotSpot** (the OpenJDK/Oracle JVM). Where a default changed across versions, the version is called out. Assume Java 8 through 21 unless noted.

---

## 1. The JVM runtime memory model

When the JVM runs, it divides memory into several runtime data areas. Some are shared across all threads; some are per-thread. Understanding this split is the foundation for everything else, because **garbage collection only manages the heap** — nothing else.

### Shared across all threads

**Heap.** The single largest region and the only one the garbage collector manages. Every object instance and every array lives here. Created at JVM startup, sized by `-Xms` (initial) and `-Xmx` (maximum). When it fills and cannot be reclaimed, you get `OutOfMemoryError: Java heap space`. The heap is subdivided into generations (young/old) by most collectors — see section 5.

**Metaspace** (Java 8+). Stores class metadata: the runtime representation of classes, method bytecode, field/method structures, and the runtime constant pool. It replaced the old **PermGen** (permanent generation) in Java 8. The critical difference: Metaspace lives in **native memory**, not the Java heap, and grows automatically by default (bounded only by native memory unless you set `-XX:MaxMetaspaceSize`). PermGen had a fixed max and was a notorious source of `OutOfMemoryError: PermGen space`, especially in app servers that reload classes. Metaspace throws `OutOfMemoryError: Metaspace` if capped and exhausted.

**String pool (interned strings).** Lives inside the heap (moved out of PermGen into the heap in Java 7). `String.intern()` and string literals are deduplicated here.

**Code cache.** Where the JIT compiler stores compiled native code. If it fills, the JIT stops compiling and performance degrades (you may see "CodeCache is full" warnings). Sized with `-XX:ReservedCodeCacheSize`.

### Per-thread (not GC-managed)

**JVM stack.** Each thread has its own stack. Every method call pushes a **stack frame** holding local variables, operand stack, and the return address. Primitives declared as locals and **object references** (not the objects themselves) live here. Overflow → `StackOverflowError` (e.g., deep/infinite recursion). Sized with `-Xss`.

**Program counter (PC) register.** Per-thread pointer to the currently executing bytecode instruction.

**Native method stack.** Supports native (JNI/C) method calls.

### The classic interview line

> **"Where do objects live vs. where do references live?"** — Objects (and arrays) always live on the **heap**. The reference variable pointing to an object lives wherever it's declared: on the **stack** if it's a local variable, or **inside the object on the heap** if it's an instance field. This is why passing an object to a method passes the reference by value, not the object.

**"Is the stack garbage collected?"** No. Stack frames are reclaimed automatically (LIFO) when a method returns — there's no GC involved. GC only touches the heap.

### Escape analysis & stack allocation (nuance worth knowing)

The "all objects go on the heap" rule has a JIT-era caveat. HotSpot's **escape analysis** can prove that an object never "escapes" the method that created it (never stored in a field, never returned, never passed to unknown code). When it can, the JIT may apply **scalar replacement** — breaking the object into its scalar fields and keeping them in registers/stack, so no heap allocation happens at all. This is why microbenchmarks sometimes show zero allocation for code that "obviously" news up an object. It's an optimization, not a language guarantee.

---

## 2. The object lifecycle & allocation path

Understanding the allocation fast path explains why allocation in Java is cheap and why the young generation exists.

**Step 1 — Allocation via bump-the-pointer.** New objects are almost always allocated in the young generation's **Eden** space. Allocation is just incrementing a pointer to the next free byte — O(1), nearly as fast as a stack push. This is possible because young-gen collectors compact, leaving one contiguous free region.

**Step 2 — TLABs (Thread-Local Allocation Buffers).** To avoid threads contending on a shared allocation pointer, each thread gets its own private chunk of Eden called a TLAB. Allocation inside a TLAB needs no synchronization. When a thread's TLAB fills, it grabs a new one (a synchronized but rare operation). Large objects that don't fit a TLAB are allocated directly in the shared Eden (or straight into old gen if huge). Controlled by `-XX:+UseTLAB` (on by default), `-XX:TLABSize`.

**Step 3 — Living in Eden, then surviving.** Most objects die young (the **weak generational hypothesis**). Those that survive a young collection get copied to a **Survivor** space and their **age** counter increments.

**Step 4 — Promotion (tenuring).** After surviving enough young collections (the **tenuring threshold**, `-XX:MaxTenuringThreshold`, default 15 for most collectors), an object is **promoted** to the old generation. Objects can also be promoted early if the survivor space can't hold them (**premature promotion**).

**Step 5 — Death.** When an object becomes unreachable, it's eligible for collection. The GC reclaims its space during the next collection of that region. `finalize()` (deprecated) or `Cleaner`/`PhantomReference` hooks may run before the memory is actually freed.

### Object header & memory layout

Every Java object carries overhead. On HotSpot (64-bit, compressed oops on):

- **Mark word** (8 bytes): identity hash code, GC age bits, lock state / biased-locking info.
- **Klass pointer** (4 bytes with compressed class pointers): points to class metadata in Metaspace.
- **Array length** (4 bytes, arrays only).
- Fields, then **padding** to an 8-byte boundary.

So an object with no fields still costs ~16 bytes. This matters for GC pressure: millions of tiny objects (e.g., boxed `Integer`s) carry huge header overhead and allocation churn. **Compressed oops** (`-XX:+UseCompressedOops`, default when heap ≤ ~32 GB) store 32-bit object references instead of 64-bit, saving memory and improving cache behavior — a key reason to keep heaps under 32 GB when possible.

---

## 3. Reachability, GC roots & reference types

### How the GC decides what's garbage

Java does **not** use reference counting (which can't reclaim cycles). It uses **reachability analysis / tracing**: starting from a set of **GC roots**, the collector traverses the object graph. Anything reachable is live; everything else is garbage. This correctly collects cyclic structures (A points to B, B points to A, but nothing outside points to either — both are collected).

### What counts as a GC root

- **Local variables and operand stack** of every live stack frame in every thread.
- **Active threads** themselves.
- **Static fields** of loaded classes.
- **JNI references** (native code holding Java objects).
- **Synchronization monitors** (objects used as locks).
- **Classes loaded by the bootstrap class loader** and other "always live" system objects.

A common interview trap: **"Does a static field keep an object alive forever?"** Yes — as long as the class is loaded, its static fields are roots. This is the single most common cause of memory leaks in Java (static collections that only grow).

### Reference types (java.lang.ref)

Java exposes four reference strengths, which change GC behavior. This is a favorite interview topic.

| Type | Collected when… | Typical use |
|------|-----------------|-------------|
| **Strong** (`Object o = new ...`) | Never, while reachable via a strong ref | Normal references |
| **SoftReference** | Only when the JVM is under memory pressure (about to OOM) | Memory-sensitive caches |
| **WeakReference** | At the next GC, if no strong refs remain | Canonicalizing maps, metadata keyed by object (`WeakHashMap`) |
| **PhantomReference** | After the object is finalized; `get()` always returns null | Precise post-mortem cleanup, replacing `finalize()` |

**Reference queues.** Soft/weak/phantom references can be registered with a `ReferenceQueue`; when the referent is cleared, the reference is enqueued so your code can react (e.g., evict a cache entry). `PhantomReference` + `Cleaner` is the modern, safe replacement for `finalize()`, which is deprecated because finalizers run on an unpredictable schedule, can resurrect objects, and stall GC.

**WeakHashMap gotcha:** the *keys* are weakly referenced, not the values. If a value strongly references its key, the entry never clears — a classic leak.

---

## 4. Core GC algorithms

Every collector is built from a small set of primitive algorithms. Know these and you can reason about any collector.

### Mark-Sweep

Two phases. **Mark:** trace from roots, flag every reachable object. **Sweep:** scan the heap, reclaim every unmarked object by adding its space to a free list. **Problem:** leaves **fragmentation** — free memory is scattered in non-contiguous holes, so a large allocation may fail even when total free space is sufficient, and allocation needs a free-list search (slower than bump-the-pointer).

### Mark-Sweep-Compact

Adds a **compact** phase: live objects are slid to one end of the region, leaving one contiguous free block. Eliminates fragmentation and restores fast bump-pointer allocation, but compaction is expensive (moving objects + fixing up all references to them). Used by the old-gen phase of Serial and Parallel collectors.

### Copying (Mark-Copy / evacuation)

Divide the region in two halves (or use survivor spaces). Copy all live objects from the "from" space to the "to" space, then discard the entire "from" space at once. **Pros:** no fragmentation, extremely fast reclamation (dead objects cost nothing — you just don't copy them), allocation stays bump-the-pointer. **Cons:** you "waste" half the space, and cost is proportional to the number of *live* objects. This is ideal for the **young generation**, where most objects are dead — few survivors to copy. This is the basis of "evacuation" in G1/ZGC/Shenandoah.

### The generational insight

Copying is cheap when survivors are few (young gen). Compaction/mark-sweep is better when most objects live long (old gen). So generational collectors use **different algorithms per generation** — copying for young, mark-sweep-compact for old. This is why "which algorithm does Java use?" has no single answer.

### Concurrent vs. parallel vs. incremental

Three orthogonal axes people confuse:

- **Serial:** one GC thread, application fully paused.
- **Parallel:** multiple GC threads working at once, but application still paused (throughput-oriented).
- **Concurrent:** GC threads run *alongside* application threads, minimizing pauses (latency-oriented). Requires extra machinery (write barriers, remembered sets) to track mutations happening during marking.
- **Incremental:** break GC work into small chunks interleaved with the app.

The hard part of concurrent GC is that the object graph **changes while you're tracing it**. Solutions: **tri-color marking** (white = unvisited, gray = visited but children not scanned, black = fully scanned) plus a **write barrier** (SATB — snapshot-at-the-beginning, used by G1/Shenandoah; or incremental-update, used by CMS) to catch references the mutator changes mid-cycle so no live object is missed.

---

## 5. The generational heap

### The weak generational hypothesis

Two empirical observations drive the whole design:

1. **Most objects die young** (the vast majority become garbage shortly after allocation).
2. **Few references point from old objects to young objects.**

If most objects die young, you should collect the young region often and cheaply, and collect the expensive old region rarely.

### Layout

```
                 HEAP
+---------------------------+-------------------+
|      Young Generation      |   Old Generation  |
|  +------+  +----+  +----+  |                   |
|  | Eden |  | S0 |  | S1 |  |   (Tenured)       |
|  +------+  +----+  +----+  |                   |
+---------------------------+-------------------+
```

- **Eden:** where new objects are born.
- **Survivor 0 / Survivor 1 (S0/S1):** two spaces; at any moment one is empty. Young GC copies live objects from Eden + the active survivor into the other survivor, ages them, and clears the rest. This "ping-pong" is the copying algorithm in action.
- **Old / Tenured:** long-lived objects that survived promotion.

Default young:old ratio is controlled by `-XX:NewRatio` (default 2 → old is 2× young), and Eden:Survivor by `-XX:SurvivorRatio` (default 8 → each survivor is 1/8 of Eden).

### Minor vs. Major vs. Full GC

- **Minor GC (Young GC):** collects only the young generation. Frequent, fast, always stop-the-world (but short). Uses copying.
- **Major GC:** collects the old generation. Much rarer, much longer.
- **Full GC:** collects the **entire** heap (young + old) and usually Metaspace, typically with compaction. The most expensive event; a frequent full-GC pattern is a red flag. Triggered by old-gen exhaustion, `System.gc()`, Metaspace pressure, promotion failure, or concurrent-mode failure.

> Terminology is loose in practice — people say "major" and "full" interchangeably. In an interview, define your terms: *minor = young only; full = whole heap with compaction*.

### Cross-generational references & the card table / remembered set

Problem: during a **minor GC** you trace from roots into the young gen — but an old-gen object might hold the only reference to a young object. Scanning the entire old gen to find such references would make minor GC as expensive as full GC, defeating the purpose.

Solution: track old→young references cheaply.

- **Card table:** the old gen is divided into fixed-size "cards" (typically 512 bytes). A **write barrier** marks a card "dirty" whenever application code writes a reference into it. At minor GC, only dirty cards are scanned for old→young pointers. Used by Serial/Parallel/CMS.
- **Remembered sets (RSets):** G1 and region-based collectors maintain, per region, a set of locations in *other* regions that point into it. This lets them collect any single region independently.

The **write barrier** — a tiny snippet of code the JIT inserts around every reference-field store — is the hidden runtime tax of generational and concurrent GC. It's cheap per-write but ever-present.

---

## 6. The collectors, one by one

### The three goals you're always trading between

Every collector optimizes some balance of:

- **Throughput** — % of total CPU time spent running your application vs. GC. Batch/analytics jobs want this high.
- **Latency (pause time)** — how long individual stop-the-world pauses last. Interactive services, trading, low-latency APIs want this low.
- **Footprint** — memory and CPU overhead the collector itself consumes.

You cannot maximize all three. Concurrent low-pause collectors buy short pauses by spending more CPU (lower throughput) and more memory (higher footprint). This tradeoff triangle is the single most important framing for any "which GC?" interview question.

### How to pick (`-XX:+UseXxxGC`)

| Collector | Flag | Generational | Pauses | Best for |
|-----------|------|--------------|--------|----------|
| Serial | `-XX:+UseSerialGC` | Yes | STW, single-thread | Small heaps, single-core, containers with 1 CPU |
| Parallel | `-XX:+UseParallelGC` | Yes | STW, multi-thread | Max throughput batch jobs |
| CMS *(removed in 14)* | `-XX:+UseConcMarkSweepGC` | Yes | Mostly concurrent | Legacy low-latency (pre-G1) |
| **G1** | `-XX:+UseG1GC` | Yes (logical) | STW but bounded | **Default since Java 9**; general-purpose |
| ZGC | `-XX:+UseZGC` | No→Yes (gen'l in 21) | <1 ms, concurrent | Very large heaps, strict latency |
| Shenandoah | `-XX:+UseShenandoahGC` | No (mostly) | <10 ms, concurrent | Low-latency, OpenJDK builds |
| Epsilon | `-XX:+UseEpsilonGC` | N/A | Never collects | Testing, ultra-short-lived jobs |

---

### 6.1 Serial GC

The simplest collector: a **single** GC thread, and the application is **fully paused** for the entire collection. Young gen uses copying; old gen uses mark-sweep-**compact**.

**Why it still exists:** on a single-core machine, parallel/concurrent collectors add coordination overhead with no benefit. Serial has the smallest footprint and is the default when the JVM detects a single available processor (common in small containers). Great for CLI tools and tiny microservices.

**Weakness:** pauses grow linearly with heap size — unusable for large heaps.

---

### 6.2 Parallel GC (Throughput Collector)

The default from Java 5 through 8. Uses **multiple threads** for both young (copying) and old (mark-sweep-compact) collections, but is still **fully stop-the-world**. It maximizes throughput by finishing GC as fast as possible using all cores, accepting that pauses can be long.

**Adaptive sizing (ergonomics):** with `-XX:+UseAdaptiveSizePolicy` (on by default), it dynamically resizes generations and adjusts the tenuring threshold to hit throughput/pause goals set by `-XX:GCTimeRatio` and `-XX:MaxGCPauseMillis`.

**Best for:** batch processing, ETL, scientific computing — anything where total completion time matters more than individual pause length. Thread count via `-XX:ParallelGCThreads`.

**Weakness:** a full GC on a big heap can pause for seconds. Not suitable for latency-sensitive services.

---

### 6.3 CMS — Concurrent Mark Sweep *(deprecated in Java 9, removed in Java 14)*

The first mainstream **low-pause** collector. It runs most of its marking work **concurrently** with the application so old-gen collection doesn't require a long STW pause. Know it for legacy systems and because interviewers love the "why was it removed?" question.

**Phases:**
1. **Initial Mark** (STW, brief) — mark objects directly reachable from roots.
2. **Concurrent Mark** — trace the graph while the app runs.
3. **Concurrent Preclean / Remark** — Remark is a short STW to catch changes made during concurrent marking (uses incremental-update write barrier).
4. **Concurrent Sweep** — reclaim dead objects concurrently.

**Two fatal flaws that got it removed:**
- **No compaction.** CMS sweeps but doesn't compact the old gen, so it **fragments** over time. Eventually a promotion fails because there's no contiguous space → it falls back to a **full STW single-threaded compacting collection** (the dreaded "concurrent mode failure"), which is a very long pause — the opposite of what you wanted.
- **CPU cost & complexity.** Concurrent work steals CPU from the app, and CMS had many fragile tuning knobs. G1 does the job better, so CMS was deprecated (JEP 291) and removed in Java 14 (JEP 363).

---

### 6.4 G1 GC (Garbage-First) — the default since Java 9

G1 is the modern general-purpose collector. Its big idea: **divide the heap into many equal-sized regions** (typically 1–32 MB, ~2048 regions) instead of large contiguous generations. Each region is tagged at runtime as Eden, Survivor, Old, or **Humongous** (for objects larger than half a region).

Generations become **logical, not physical** — any region can play any role, and the set of regions in each generation changes every cycle.

**The "Garbage-First" name:** G1 tracks how much garbage each region holds and collects the regions with the **most garbage first**, giving the best reclamation for the least work. This is what lets it hit a **pause-time target**.

**Pause target:** `-XX:MaxGCPauseMillis` (default **200 ms**). G1 tries to keep pauses under this by only collecting as many regions per pause as it estimates it can within the budget (the **collection set** / CSet). This is a soft goal, not a guarantee.

**Cycle overview:**
- **Young collections** (STW): evacuate live objects from Eden/Survivor regions into new Survivor/Old regions (copying).
- **Concurrent marking cycle** (triggered when old-gen occupancy crosses `-XX:InitiatingHeapOccupancyPercent`, IHOP, default 45%): initial mark (piggybacks on a young pause), concurrent mark, remark (STW, SATB), cleanup.
- **Mixed collections** (STW): subsequent young collections that *also* evacuate a chosen set of the most-garbage old regions. G1 avoids a full old-gen collection by spreading old-region cleanup across several mixed GCs.

**Compaction:** G1 compacts incrementally via evacuation (copying survivors into fresh regions), so it does **not** suffer CMS-style fragmentation.

**Full GC fallback:** if G1 can't keep up (allocation outpaces reclamation, or evacuation fails), it falls back to a full GC — historically single-threaded, made **parallel** in Java 10 (JEP 307). A full GC in G1 means your tuning is off.

**Humongous objects:** objects ≥ 50% of a region size get their own contiguous run of regions and are allocated directly in old gen. Lots of humongous allocations fragment the region space and can force early marking cycles — a real production gotcha (e.g., large byte arrays).

**Best for:** the default choice for most server apps with heaps from a few GB up to tens of GB that want a balance of throughput and predictable, bounded pauses.

---

### 6.5 ZGC — the scalable low-latency collector

ZGC targets **sub-millisecond max pause times regardless of heap size** — heaps from a few hundred MB to **16 TB**. Pause times do **not** grow with heap or live-set size, which is its headline property.

**How it stays concurrent:** almost everything — marking, relocation (compaction), and reference processing — happens **concurrently** with the application. The few STW pauses are only for root scanning and are bounded to microseconds/low-milliseconds.

**Key mechanisms:**
- **Colored pointers:** ZGC stores metadata (marked, remapped, etc.) directly in unused bits of 64-bit object pointers. This lets the collector know an object's GC state from the pointer itself.
- **Load barriers:** instead of a write barrier, ZGC uses a **load barrier** — when the app reads a reference, the barrier checks the pointer color and, if the object has been relocated, fixes up the reference on the fly ("self-healing"). This is what allows concurrent relocation without stopping the app.

**Generational ZGC (Java 21, JEP 439):** original ZGC was **non-generational** (it treated the whole heap uniformly, which wastes effort since most objects die young). Generational ZGC adds young/old generations, dramatically improving efficiency and reducing CPU/allocation-stall overhead. As of Java 21 it's the recommended mode (`-XX:+UseZGC -XX:+ZGenerational`; becomes default ZGC behavior in Java 23+).

**Costs:** higher memory footprint (colored pointers, multi-mapped memory) and CPU overhead vs. throughput collectors. Requires a 64-bit platform.

**Best for:** large heaps + strict latency SLAs (real-time analytics, large caches, low-latency trading, big in-memory datasets).

---

### 6.6 Shenandoah — concurrent compaction, any heap size

Red Hat's low-pause collector, contributed to OpenJDK. Like ZGC, it does **concurrent evacuation/compaction**, so pause times are low and **independent of heap size**. Historically it targeted heaps of any size (including smaller ones) where ZGC was originally tuned for very large heaps.

**Key mechanism — Brooks forwarding pointers (older versions) / load-reference barriers (newer):** each object carried an extra forwarding pointer so it could be relocated while the app still reads it via the indirection. Newer Shenandoah uses load-reference barriers similar in spirit to ZGC's approach.

**Mostly non-generational** (a generational mode is in development). Pauses are typically a few milliseconds, dominated by root scanning.

**Best for:** low-latency needs on OpenJDK distributions (it's available in many builds; note it isn't in Oracle's own JDK binaries). Practically, ZGC and Shenandoah occupy the same niche; the choice often comes down to which is available and better-tested on your JDK build.

---

### 6.7 Epsilon — the "no-op" collector (Java 11, JEP 318)

Epsilon **allocates but never reclaims**. When the heap fills, the JVM exits with OOM. Uses: performance testing (isolate allocation cost from GC), extremely short-lived jobs that finish before exhausting memory, and measuring the true GC overhead of your app by comparing against a baseline with no GC. Not for production services.

---

### Quick decision guide

- **1 CPU / tiny heap / container:** Serial.
- **Batch throughput, pauses don't matter:** Parallel.
- **General server app, want good defaults:** G1 (you already have it).
- **Large heap (>tens of GB) and/or strict p99 latency:** ZGC (generational) or Shenandoah.
- **Benchmarking/GC-free short jobs:** Epsilon.
- **Legacy system on CMS:** plan migration to G1/ZGC — CMS is gone as of Java 14.

---

## 7. Safepoints, stop-the-world & GC threads

### What "stop-the-world" really means

A **stop-the-world (STW) pause** is a window in which **all application (mutator) threads are suspended** so the GC can work on a stable object graph. Even "concurrent" collectors have brief STW pauses (root scanning, remark). Understanding *how* threads stop is a strong interview differentiator.

### Safepoints

Threads can't be stopped at an arbitrary machine instruction — the GC needs each thread at a point where its stack and registers are in a known, walkable state. These points are **safepoints**, and the JIT inserts safepoint **polls** (cheap checks) at method returns and loop back-edges. To start a pause, the JVM sets a global "please stop" flag; each thread stops at its next safepoint poll.

**Time-to-safepoint (TTSP):** the time between "stop requested" and "all threads actually stopped." This is *not* counted in some naive GC timers but is very real. A single thread stuck in a long **counted loop with no safepoint poll**, or blocked in native code, or paged out, can stall *every other thread* that already reached its safepoint — producing a long pause that GC logs blame on GC but is actually a TTSP problem. Diagnose with `-Xlog:safepoint` (Java 9+) or `-XX:+PrintSafepointStatistics` (older). This is an advanced-but-loved production war story.

### Not just GC uses safepoints

Safepoints are also used for biased-lock revocation, `Thread.getStackTrace`, deoptimization, and some JVMTI operations — so "STW pause" and "GC" aren't synonyms. A `Deoptimization` or `RevokeBias` safepoint can masquerade as a GC hiccup.

### GC thread counts

- `-XX:ParallelGCThreads` — threads for STW parallel phases (young evacuation, parallel full GC). Default derives from CPU count (roughly `#CPUs` up to 8, then `5/8 × #CPUs`).
- `-XX:ConcGCThreads` — threads for concurrent phases (concurrent marking in G1/ZGC/Shenandoah). Fewer than ParallelGCThreads; these steal CPU from the app, so more concurrent threads = shorter GC cycles but less CPU for your workload.

**Container gotcha:** before JDK 8u191/JDK 10, the JVM read the *host* CPU/memory count, not the cgroup limit — so a container limited to 2 CPUs might spin up GC threads sized for a 64-core host. Modern JDKs are container-aware (`-XX:+UseContainerSupport`, on by default) and respect cgroup limits. Always run a container-aware JDK; otherwise set `-XX:ActiveProcessorCount` manually.

---

## 8. GC pressure & allocation rate

### What "GC pressure" means

**GC pressure** is the rate at which your application forces the collector to do work. High GC pressure = GC runs frequently and/or does a lot of work per run, stealing CPU and causing pauses. It's driven primarily by two things:

1. **Allocation rate** — how many MB/s of new objects you create. Higher allocation rate → Eden fills faster → more frequent minor GCs.
2. **Promotion rate / live-set size** — how many bytes survive long enough to be promoted to old gen. Higher promotion → more frequent expensive old/mixed/full collections.

### Allocation rate — the number to know

Allocation rate = (Eden size) ÷ (time between minor GCs), or summed bytes allocated per second. You compute it from GC logs: look at young-gen occupancy before each minor GC and the timestamps.

- **Low (< ~1 GB/min):** usually fine.
- **Moderate (~1–2 GB/min):** watch it.
- **High (multiple GB/min):** likely a throughput and latency problem; expect frequent minor GCs.

**High allocation rate is usually an application problem, not a GC problem.** The fix is to allocate less, not to tune the collector. Common culprits: excessive autoboxing (`Integer`/`Long` in hot loops), string concatenation in loops, defensive copying, oversized collections, logging that builds big strings, per-request object graphs that could be pooled or streamed.

### Premature promotion & the "medium-lived" object problem

The generational design assumes objects are either short-lived (die in young gen) or long-lived (belong in old gen). **Medium-lived** objects — those that live just long enough to survive a few minor GCs but then die — are the worst case: they get promoted to old gen (premature promotion) and then die there, forcing old-gen collections. Symptoms: old gen grows steadily then drops sharply at each major GC; survivor spaces overflow. Fixes: enlarge young gen / survivor spaces, raise the tenuring threshold, or (best) restructure the code so these objects die in young gen. Watch tenuring distribution with `-XX:+PrintTenuringDistribution`.

### Allocation stalls

When allocation rate exceeds the rate at which the concurrent collector can reclaim memory, the JVM must **pause the allocating thread** until memory is freed. In G1 this can trigger a full GC; in ZGC/Shenandoah it shows up as **allocation stalls** in the logs. This is the low-latency collector's version of "falling behind" — the app slows even though pauses stay short, because threads block waiting for memory.

### The pressure feedback loop to recognize

High allocation → frequent minor GC → survivors overflow → premature promotion → old gen fills fast → frequent major/full GC → **more CPU in GC, less in app** → requests queue → even more allocation. This spiral is what a "GC death spiral" looks like in production, often ending in `OutOfMemoryError: GC overhead limit exceeded` (thrown when >98% of time is spent in GC while recovering <2% of heap).

---

## 9. Diagnosing GC CPU overhead in production

This is the section the interview question *"How would you check if there's CPU overhead in production due to GC?"* is really about. The answer is a **layered investigation**: confirm GC is the cause, quantify the overhead, then find the root cause. Walk it top-down.

### Step 0 — The one metric that answers the question: GC throughput %

**GC throughput** = fraction of wall-clock time the app spends running application code, i.e. `1 − (time in GC / total time)`. Equivalently, **GC overhead %** = time in GC ÷ total time.

- **> 95–99% throughput (< 1–5% in GC): healthy.**
- **90–95%: worth investigating.**
- **< 90% (i.e., >10% of CPU in GC): a real problem.** Below ~80% you're in a death spiral.

You get this from GC logs (tools like **GCeasy**, **GCViewer** compute it automatically) or by summing pause durations over a window. This single ratio is the crisp, quantitative answer interviewers want: *"I'd measure GC throughput from the GC logs; if more than X% of CPU time is spent in GC, that's the overhead, and here's how I'd confirm and fix it."*

### Step 1 — Confirm from the outside (is CPU high, and is it GC?)

Before touching GC, confirm the symptom and localize it to GC threads specifically.

- **`top` / `htop`:** is CPU actually high on the Java process?
- **Per-thread CPU:** `top -H -p <pid>` (Linux) shows CPU **per thread**. Take the hot thread IDs, convert to hex (`printf '%x\n' <tid>`), and match against a thread dump (`jstack <pid>`). If the hot threads are named `GC Thread#`, `G1 Young RemSet`, `G1 Conc#`, `ZWorker`, etc., GC is directly burning CPU. If the hot threads are your app's worker threads, the CPU cost may be *indirect* (allocation, barriers) rather than the collector itself.
- **`jstat -gcutil <pid> 1000`:** prints, every second, the % utilization of each region (S0, S1, E, O, M) plus **YGC/YGCT** (young GC count/time) and **FGC/FGCT** (full GC count/time), and **GCT** (total GC time). Rapidly climbing YGC/FGC counts and rising GCT relative to elapsed time = GC overhead, live, with zero setup. This is the fastest field diagnostic.

Example read: if over 60 seconds `GCT` rises by 12 seconds, ~20% of CPU-time went to GC — clearly a problem.

### Step 2 — Quantify from GC logs

Turn on GC logging (keep it on in production — it's cheap; see section 10 for flags) and analyze:

- **Unified logging (Java 9+):** `-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=20m`
- **Java 8:** `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20m`

Feed the log to **GCeasy.io** (web), **GCViewer**, or **gcplot**. Look at:
- **Throughput %** (the headline number).
- **Pause distribution** — p50/p99/max pause, and whether pauses cluster (back-to-back full GCs).
- **GC frequency** — minor GCs per minute, and any full GCs at all (with G1/ZGC, full GCs should be near-zero).
- **Allocation rate & promotion rate** — the tools chart these directly.
- **Heap after GC trend** — if live-set-after-full-GC keeps climbing, you have a leak (section 12), not just pressure.

### Step 3 — Low-overhead always-on profiling

- **JDK Flight Recorder (JFR)** — built in, ~1% overhead, safe for production. Start on the fly: `jcmd <pid> JFR.start duration=120s filename=rec.jfr settings=profile`. Open in **JDK Mission Control (JMC)**. JFR gives you GC pauses, **allocation profiling by call site** ("TLAB allocation" / "allocation outside TLAB" events tell you exactly which methods allocate the most), promotion, and time-to-safepoint — all correlated with CPU. This is the single best tool for "what is causing GC pressure?"
- **async-profiler** — sample the `alloc` event to get a **flame graph of allocation hot paths** with negligible overhead. The flame graph literally shows which stack traces allocate the most bytes; that's usually your fix target.

### Step 4 — Interpret: is it GC pressure, a leak, or misconfiguration?

Decision tree from the evidence above:

- **Frequent minor GCs, low pause each, high allocation rate, old gen stable →** allocation-rate/GC-pressure problem. Fix the app (allocate less) or enlarge young gen.
- **Frequent full GCs, old gen fills right back up after each →** either genuine leak (heap-after-GC trends up over hours) or under-sized heap for the working set.
- **Long pauses but low frequency, huge heap on Parallel/G1 →** wrong collector for the latency goal; move to G1/ZGC or tune `MaxGCPauseMillis`.
- **Pauses long but GC "real" time short in logs →** time-to-safepoint problem, not GC (section 7); check `-Xlog:safepoint`.
- **CPU high but `jstat` GCT flat →** not GC; profile the app normally.

### Step 5 — Common GC-overhead root causes & fixes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Very high allocation rate | Boxing, string building, per-request garbage | Reduce allocation; reuse buffers; primitive collections |
| Old gen fills fast, frequent full GC | Premature promotion / undersized heap / leak | Bigger young gen or heap; find the leak |
| Frequent humongous allocations (G1) | Large arrays ≥ half region size | Increase `-XX:G1HeapRegionSize`; avoid giant arrays |
| Long STW despite concurrent GC | Time-to-safepoint / weak-ref processing | Fix long counted loops; check `-Xlog:safepoint` |
| `GC overhead limit exceeded` | Heap too small / leak | Increase `-Xmx`; heap-dump analysis |
| High GC after deploy | New code allocates more or holds refs | Diff allocation flame graphs across versions |

### Step 6 — Production hygiene

Export GC metrics to your monitoring stack so you don't diagnose blind. Micrometer/Prometheus expose `jvm_gc_pause_seconds`, `jvm_gc_memory_allocated_bytes_total` (→ allocation rate), `jvm_memory_used_bytes{area="heap"}`, and GC collection counts. Alert on GC-time-percentage and on any full GC. This turns "check if there's GC CPU overhead" from a fire drill into a dashboard.

---

## 10. Reading GC logs

### Enabling logs

**Java 9+ (unified logging, `-Xlog`):**
```
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=10,filesize=50m
```
`gc*` = all GC tags; add `safepoint` to catch TTSP; use `gc+heap=debug` for detailed sizing. Log rotation is built in.

**Java 8:**
```
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime \
-Xloggc:gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=50m
```

GC logging overhead is negligible — **leave it on in production.** You cannot diagnose a GC incident after the fact without it.

### Anatomy of a G1 young pause line (Java 11+)

```
[2.734s][info][gc] GC(12) Pause Young (Normal) (G1 Evacuation Pause) 512M->48M(1024M) 8.393ms
```
Read it as: at 2.734s uptime, the 12th GC event, a normal young evacuation pause; heap went from **512M used before → 48M used after**, out of a **1024M** total heap; the pause lasted **8.393 ms**. The drop (512→48) is what was reclaimed; the "after" value (48M) is your live young set at that moment.

### The numbers you extract

- **Before→After(Total):** reclamation and current occupancy.
- **Pause duration:** the STW cost. Track its distribution, not just the average.
- **`Pause Young` vs `Pause Full` vs `Pause Remark`/`Cleanup`:** full pauses in G1/ZGC are alarms.
- **`User`/`Sys`/`Real` (Java 8 style `[Times: user=0.10 sys=0.00 real=0.03 secs]`):** if `real` ≫ `user/ParallelGCThreads`, GC threads were starved of CPU (noisy neighbor / oversubscription). If `sys` is high, suspect paging or transparent huge pages.

### Free tools

- **GCeasy.io** — upload a log, get throughput %, pause percentiles, allocation rate, and tuning recommendations. Fastest path to an answer.
- **GCViewer** — desktop, open-source, good charts.
- **JDK Mission Control** — for JFR recordings (richer than text logs).

---

## 11. Tuning playbook & JVM flags

### The golden rule

**Measure first, tune last, change one thing at a time.** Most GC problems are solved by allocating less or sizing the heap correctly — not by exotic flags. Modern collectors (G1, ZGC) are designed to work well with almost no tuning. Over-tuning is a common self-inflicted wound.

### Essential sizing flags

| Flag | Meaning |
|------|---------|
| `-Xms` / `-Xmx` | Initial / max heap. **Set them equal** in production to avoid resize pauses and give a stable footprint. |
| `-Xmn` / `-XX:NewSize`,`MaxNewSize` | Young gen size. Bigger young gen = fewer minor GCs but longer each. |
| `-XX:NewRatio` | Old:young ratio (default 2). |
| `-XX:SurvivorRatio` | Eden:survivor ratio (default 8). |
| `-XX:MaxTenuringThreshold` | Ages before promotion (default 15). Lower = promote sooner. |
| `-XX:MetaspaceSize` / `-XX:MaxMetaspaceSize` | Metaspace initial trigger / cap. |
| `-Xss` | Per-thread stack size. |

### Collector selection & goals

| Flag | Meaning |
|------|---------|
| `-XX:+UseG1GC` / `UseZGC` / `UseParallelGC` / `UseSerialGC` / `UseShenandoahGC` | Pick collector. |
| `-XX:MaxGCPauseMillis` | Soft pause target (G1 default 200). |
| `-XX:GCTimeRatio` | Throughput goal (Parallel): `1/(1+ratio)` max fraction in GC. |
| `-XX:InitiatingHeapOccupancyPercent` (IHOP) | When G1 starts a concurrent marking cycle (default 45). |
| `-XX:G1HeapRegionSize` | G1 region size (affects humongous threshold). |
| `-XX:ParallelGCThreads` / `-XX:ConcGCThreads` | STW / concurrent GC thread counts. |
| `-XX:+ZGenerational` | Enable generational ZGC (Java 21). |

### Diagnostics flags

| Flag | Meaning |
|------|---------|
| `-XX:+HeapDumpOnOutOfMemoryError` | **Always set in prod.** Dumps heap on OOM. |
| `-XX:HeapDumpPath=/path` | Where to write the dump. |
| `-XX:+ExitOnOutOfMemoryError` / `+CrashOnOutOfMemoryError` | Fail fast so the orchestrator restarts a healthy instance. |
| `-Xlog:gc*` (9+) / `-XX:+PrintGCDetails` (8) | GC logging. |
| `-XX:+PrintTenuringDistribution` | See how objects age (diagnose premature promotion). |
| `-XX:+DisableExplicitGC` | Turn `System.gc()` calls into no-ops. |
| `-XX:NativeMemoryTracking=summary` | Track native/off-heap memory. |

### Heap-sizing heuristics

A common starting point: after a full GC, note the **live set** (heap used right after full GC). Size `-Xmx` to roughly **2.5–4× the live set** so the collector has headroom to avoid constant collection. Too small → GC thrash; too large → longer pauses and wasted RAM (and, if it crosses ~32 GB, you lose compressed oops — a step change in per-object memory cost). In containers, use `-XX:MaxRAMPercentage` (e.g., 75) instead of hardcoding `-Xmx`, so the heap scales with the container's memory limit.

### `System.gc()`

Calling `System.gc()` **requests** (doesn't force) a full GC and is almost always a mistake in application code — it can trigger long STW pauses at bad times. Disable its effect with `-XX:+DisableExplicitGC`. Exceptions: some frameworks (NIO direct buffers, RMI DGC) rely on it; understand why before disabling.

---

## 12. Memory leaks & OutOfMemoryError

### GC doesn't prevent leaks

In Java a "leak" isn't unfreed memory — it's **unintentionally reachable** memory. The GC won't collect anything reachable from a root, so if you keep references you no longer need, the objects live forever. The heap grows until `OutOfMemoryError: Java heap space`.

### The usual suspects

- **Static collections that only grow** — a `static Map`/`List` cache with no eviction. The #1 Java leak.
- **Unbounded caches** — use `SoftReference`-based caches, or better, a real cache with size/TTL eviction (Caffeine, Guava). "Soft references as a cache" is fragile because the JVM clears them unpredictably under pressure.
- **Listeners/callbacks never unregistered** — observers held by a long-lived subject.
- **`ThreadLocal` in thread pools** — the thread outlives the request, so the ThreadLocal value is never cleared. Classic in app servers; call `remove()` in a `finally`.
- **ClassLoader leaks** — a single reference from outside a web app pins its entire ClassLoader (and all its classes/statics) → `OutOfMemoryError: Metaspace` on redeploy.
- **Growing `WeakHashMap` values referencing their keys** (see section 3).
- **Interned strings / large char arrays** held indefinitely.

### The flavors of OutOfMemoryError

`OutOfMemoryError` is not always "heap full" — the message tells you where:

- **`Java heap space`** — heap genuinely full (leak or undersized). Heap-dump it.
- **`GC overhead limit exceeded`** — >98% time in GC, <2% recovered. Nearly-full heap thrashing.
- **`Metaspace`** — too many classes (dynamic proxies, redeploys, code generation). Raise `MaxMetaspaceSize` or fix the ClassLoader leak.
- **`Unable to create new native thread`** — hit OS thread limit / native memory exhausted, *not* heap. Reduce thread count or `-Xss`.
- **`Requested array size exceeds VM limit`** — array near `Integer.MAX_VALUE`.
- **`Direct buffer memory`** — off-heap NIO buffers exhausted (`-XX:MaxDirectMemorySize`).

Interviewers love: *"You get an OOM — is it always the heap?"* Answer: no; read the message, it names the region.

### Diagnosing a leak

1. **Confirm the trend:** heap-used **after full GC** climbs monotonically over hours/days (not just sawtooth). Watch via `jstat -gcutil` or monitoring.
2. **Capture a heap dump:** `-XX:+HeapDumpOnOutOfMemoryError` (automatic on OOM) or on demand `jcmd <pid> GC.heap_dump /path/dump.hprof` (or `jmap -dump:live,format=b,file=dump.hprof <pid>`).
3. **Analyze in Eclipse MAT** (Memory Analyzer): run the **Leak Suspects** report; use **Dominator Tree** to find the objects retaining the most memory and **Path to GC Roots** to see *what* is keeping a suspect alive. VisualVM and JFR's old-object-sample event also work.
4. **Fix the retention:** break the reference, add eviction, call `ThreadLocal.remove()`, unregister the listener.

`-XX:+HeapDumpOnOutOfMemoryError` should be **on in every production JVM** — an OOM without a dump wastes the incident.

---

## 13. Off-heap & native memory

The Java heap is only part of a JVM process's memory. When "the container got OOM-killed but heap looked fine," the answer is usually **native memory**.

Total JVM process memory ≈ **heap** + **Metaspace** + **code cache** + **thread stacks** (`#threads × -Xss`) + **GC structures** (card tables, remembered sets, marking bitmaps — ZGC/Shenandoah in particular use extra) + **direct byte buffers** (`ByteBuffer.allocateDirect`, Netty, NIO) + **memory-mapped files** + native libraries.

- **Direct buffers** live off-heap and are reclaimed only when their (heap-side) wrapper object is GC'd and its `Cleaner` runs — so off-heap can balloon even with a small heap. Cap with `-XX:MaxDirectMemorySize`.
- **Native Memory Tracking (NMT):** start with `-XX:NativeMemoryTracking=summary`, then `jcmd <pid> VM.native_memory summary` to see the breakdown by category. Essential for "why is my container using 2 GB when `-Xmx` is 1 GB?"
- **Container sizing:** reserve headroom above `-Xmx` for all the above (often 25–50%), or the kernel OOM-killer terminates the process even though the Java heap never filled. Use `-XX:MaxRAMPercentage` so the JVM leaves room.

---

## 14. Interview question bank

Answers are written to be *spoken* — concise, then deepened if pressed. Grouped by level.

### A. Fundamentals

**Q1. Walk me through the JVM memory areas.**
Heap (GC-managed, holds all objects, split into young/old), Metaspace (class metadata, native memory, replaced PermGen in Java 8), per-thread JVM stacks (frames with locals and references), PC register, native method stack, plus the shared code cache and string pool. GC only touches the heap.

**Q2. Stack vs. heap — what goes where?**
Objects and arrays: always heap. Primitive locals and object *references*: on the stack if local, inside the object on the heap if a field. Stack frees automatically on method return (no GC); heap needs GC.

**Q3. How does Java decide an object is garbage?**
Reachability analysis (tracing), not reference counting. From GC roots (stack locals, static fields, active threads, JNI refs, monitors) it marks everything reachable; the rest is garbage. This collects cycles, which reference counting can't.

**Q4. What are GC roots?**
Entry points the trace starts from: local variables/operand stacks of live frames, static fields of loaded classes, active thread objects, JNI global refs, and synchronization monitors.

**Q5. Why doesn't Java use reference counting?**
It can't reclaim cyclic references, and maintaining counts on every assignment is costly. Tracing handles cycles and amortizes cost.

**Q6. Strong vs. soft vs. weak vs. phantom references?**
Strong: never collected while reachable. Soft: collected only under memory pressure (caches). Weak: collected at next GC if no strong ref (`WeakHashMap`, canonical maps). Phantom: `get()` is always null, used with a `ReferenceQueue` for precise cleanup, replacing `finalize()`.

**Q7. What replaced PermGen and why?**
Metaspace (Java 8). PermGen had a fixed max and caused frequent `OutOfMemoryError: PermGen space`, especially with class reloading. Metaspace lives in native memory and auto-grows, so it's bounded by native memory unless you cap it.

**Q8. Minor vs. major vs. full GC?**
Minor: young gen only — frequent, short, copying. Major: old gen. Full: entire heap (+ Metaspace) with compaction — rarest and most expensive. Frequent full GCs are a red flag.

**Q9. What's the generational hypothesis?**
Most objects die young, and few old→young references exist. So collect young often and cheaply (copying), old rarely (mark-compact). It's why the heap is generational.

### B. Collectors

**Q10. Which collector is the default, and since when?**
G1, default since Java 9. Java 8 and earlier defaulted to Parallel GC.

**Q11. Compare the collectors on the throughput/latency/footprint triangle.**
Serial: low footprint, long pauses, single-threaded. Parallel: max throughput, long STW pauses. G1: balanced, bounded pauses via a pause target. ZGC/Shenandoah: sub-ms/low-ms pauses independent of heap size, at the cost of CPU and footprint. You trade throughput and memory to buy low latency.

**Q12. Why was CMS removed?**
It didn't compact the old gen, so it fragmented and eventually fell back to a long single-threaded full GC ("concurrent mode failure"), plus it was complex and CPU-hungry. G1 does the same job better. Deprecated in Java 9, removed in Java 14.

**Q13. How does G1 achieve bounded pause times?**
It splits the heap into ~2048 equal regions, tracks garbage per region, and collects the most-garbage regions first, only as many as fit in the `MaxGCPauseMillis` budget (default 200 ms). Old-gen cleanup is spread across incremental "mixed" collections instead of one big pause.

**Q14. What are humongous objects in G1?**
Objects ≥ 50% of a region size. They're allocated directly into contiguous old-gen regions, can't use the normal fast path, and lots of them fragment region space and trigger early marking. Fix by increasing `G1HeapRegionSize` or avoiding giant arrays.

**Q15. How do ZGC and Shenandoah keep pauses sub-millisecond?**
They do marking *and compaction* concurrently with the app. ZGC uses colored pointers + load barriers (self-healing references when objects move); Shenandoah uses forwarding pointers / load-reference barriers. The only STW work is brief root scanning, so pauses don't grow with heap size.

**Q16. What is generational ZGC and why does it matter?**
Original ZGC was non-generational — it scanned the whole heap uniformly, wasting effort since most objects die young. Java 21 added young/old generations to ZGC, cutting CPU and allocation-stall overhead substantially. It's the recommended ZGC mode now.

**Q17. When would you actually choose Serial GC?**
Single-CPU environments or tiny containers/CLI tools where parallel/concurrent coordination overhead outweighs benefit and footprint matters most.

**Q18. What's the difference between parallel and concurrent GC?**
Parallel = multiple GC threads but app is paused. Concurrent = GC runs alongside the app. Parallel targets throughput; concurrent targets latency.

**Q19. How does concurrent marking stay correct while the app mutates the graph?**
Tri-color marking plus a write barrier. SATB (G1/Shenandoah) preserves the object graph "snapshot" at cycle start; incremental-update (CMS) re-scans modified references. A short STW remark reconciles what changed.

### C. Internals

**Q20. What is a TLAB and why does it exist?**
Thread-Local Allocation Buffer — a private slice of Eden per thread so allocation is a lock-free pointer bump. Avoids threads contending on the shared Eden allocation pointer.

**Q21. What is a safepoint? What's time-to-safepoint?**
A point where a thread's state is walkable by the GC; the JIT inserts polls at method returns and loop back-edges. TTSP is how long until *all* threads reach a safepoint after a pause is requested. A single thread slow to reach one stalls everyone — a pause that looks like GC but isn't.

**Q22. What are the card table and remembered set for?**
To find old→young references cheaply during minor GC without scanning all of old gen. A write barrier marks "cards" dirty on reference writes (card table, Serial/Parallel/CMS); G1 keeps per-region remembered sets so any region can be collected independently.

**Q23. What is a write barrier vs. a load barrier?**
Write barrier: code run on reference *stores*, to track cross-gen refs and concurrent-marking changes (G1). Load barrier: code run on reference *reads*, to fix up moved objects during concurrent relocation (ZGC). It's the mechanism that lets ZGC move objects without stopping the app.

**Q24. Explain object aging and the tenuring threshold.**
Each survival of a minor GC increments an object's age. When age exceeds `MaxTenuringThreshold` (default 15), it's promoted to old gen. Survivor overflow can promote earlier (premature promotion).

**Q25. What is escape analysis?**
A JIT optimization proving an object never leaves its creating method; the JIT can then scalar-replace it (keep fields in registers) so no heap allocation occurs at all. Reduces GC pressure invisibly.

**Q26. How big is an empty object, and why care?**
~16 bytes on 64-bit HotSpot (12-byte header + padding). Matters because millions of tiny objects (boxed primitives) carry massive header + allocation overhead — a top cause of GC pressure. Compressed oops (heap ≤ ~32 GB) shrink references to 32 bits.

### D. Production / diagnosis (the emphasis area)

**Q27. How do you check if there's CPU overhead in production due to GC?**
Compute **GC throughput** = 1 − (time in GC / total time). Get it live with `jstat -gcutil <pid> 1000` (watch GCT rise vs. elapsed) or from GC logs via GCeasy/GCViewer. Cross-check with `top -H -p <pid>` to see whether GC-named threads are actually hot. If more than ~5–10% of CPU-time is in GC, that's your overhead. Then confirm with JFR/async-profiler allocation profiling and find the root cause. (Full walkthrough in section 9.)

**Q28. What is GC pressure and what drives it?**
The rate the app forces GC to work. Driven by allocation rate (young-gen churn → frequent minor GCs) and promotion rate / live-set size (→ expensive old collections). Usually an app problem — fix by allocating less.

**Q29. How do you measure allocation rate?**
Eden size ÷ time between minor GCs, or `jvm_gc_memory_allocated_bytes_total` rate in monitoring, or read it directly from GCeasy. Multiple GB/min is high.

**Q30. Your service p99 latency spiked. How do you tell if GC is responsible?**
Correlate latency spikes with GC-pause timestamps from `-Xlog:gc`. If p99 lines up with pauses, it's GC — check pause distribution and collector fit. If pauses are short but "real" ≫ "user" time, suspect time-to-safepoint or CPU starvation, not the collector itself.

**Q31. You keep seeing frequent full GCs. What's happening and what do you do?**
Old gen fills faster than it's reclaimed: premature promotion, undersized heap, a leak, or (G1) humongous allocations / evacuation failure. Check whether heap-after-full-GC trends up (leak) vs. stable (pressure/sizing). Fixes: enlarge young gen/heap, fix allocation, tune IHOP/region size, or take a heap dump if it's a leak.

**Q32. What tools do you use to diagnose GC issues in production?**
`jstat` (live counters), `jcmd`/`jmap` (dumps, JFR control), GC logs + GCeasy/GCViewer, JDK Flight Recorder + Mission Control (~1% overhead, allocation-by-call-site), async-profiler (allocation flame graphs), Eclipse MAT (heap-dump/leak analysis), and monitoring (Micrometer/Prometheus/Grafana).

**Q33. Is `OutOfMemoryError` always about the heap?**
No — read the message. `Java heap space`, `GC overhead limit exceeded`, `Metaspace`, `unable to create new native thread`, `Direct buffer memory`, `array size exceeds VM limit` all point to different regions and different fixes.

**Q34. Container reports OOM-kill but the Java heap looks fine. Why?**
Process memory = heap + Metaspace + code cache + thread stacks + GC structures + direct buffers + mapped files. Off-heap/native growth (often direct byte buffers or too many threads) exceeded the container limit. Use NMT (`-XX:NativeMemoryTracking`) and leave headroom above `-Xmx` (`-XX:MaxRAMPercentage`).

**Q35. Should you call `System.gc()`?**
Almost never — it requests a full GC and can cause long pauses at bad times. Neutralize with `-XX:+DisableExplicitGC`. Some frameworks (NIO/RMI) legitimately depend on it.

**Q36. How do you find a memory leak?**
Confirm heap-after-full-GC climbs over time, capture a heap dump (`HeapDumpOnOutOfMemoryError` or `jcmd GC.heap_dump`), analyze in Eclipse MAT (Leak Suspects, Dominator Tree, Path to GC Roots) to find what retains the memory, then break that reference. Common causes: static collections, unbounded caches, un-removed `ThreadLocal`s in pools, unregistered listeners, ClassLoader leaks.

**Q37. How do you size the heap for a service?**
After a full GC, note the live set. Set `-Xmx` to ~2.5–4× that for headroom, set `-Xms = -Xmx` to avoid resize pauses, keep under ~32 GB to retain compressed oops if possible, and in containers use `-XX:MaxRAMPercentage` with headroom for native memory.

**Q38. What flags do you always set in production?**
`-Xms=-Xmx`, GC logging (`-Xlog:gc*` with rotation), `-XX:+HeapDumpOnOutOfMemoryError` + `HeapDumpPath`, a container-aware JDK with `MaxRAMPercentage`, and often `+ExitOnOutOfMemoryError` so the orchestrator restarts a clean instance.

### E. Scenario / whiteboard

**Q39. You migrated Java 8 (Parallel) → Java 17 and pauses changed character — why?**
Default collector changed to G1. Parallel had rarer but longer pauses (throughput-first); G1 has more frequent but bounded pauses. If you need Parallel's throughput for a batch job, set `-XX:+UseParallelGC` explicitly.

**Q40. A cache using a `HashMap` grows until OOM. How would you redesign it?**
Add bounded eviction (size/TTL) with a real cache (Caffeine), or use `SoftReference` values if you accept unpredictable clearing, or `WeakHashMap` if entries should die with their keys — being careful values don't strongly reference keys.

**Q41. Explain what happens, step by step, when `new` is called and Eden is full.**
Allocation tries the thread's TLAB; if it can't fit, it may take a new TLAB or allocate in shared Eden. If Eden is full, a minor GC runs: live objects in Eden + active survivor are copied to the other survivor, ages increment, dead objects are dropped, and objects past the tenuring threshold or overflowing survivor go to old gen. Then allocation proceeds.

**Q42. When would low-latency collectors (ZGC/Shenandoah) hurt you?**
Throughput-bound batch workloads: their concurrent work and barriers spend CPU and memory that a throughput job would rather give to computation. Parallel GC would finish faster overall despite longer pauses.

---

## 15. Rapid-fire flashcards

- **GC manages only the** heap.
- **Objects live on** the heap; **references** live on stack (locals) or heap (fields).
- **Default GC since Java 9:** G1. **Before:** Parallel.
- **CMS removed in** Java 14.
- **Reachability, not** reference counting → collects cycles.
- **Weak generational hypothesis:** most objects die young.
- **Young gen =** Eden + 2 survivors; collected by copying.
- **Minor GC** = young only; **Full GC** = whole heap + compaction.
- **Tenuring threshold default:** 15 survivals → promote to old gen.
- **TLAB:** per-thread lock-free bump-pointer allocation in Eden.
- **G1 pause target:** `-XX:MaxGCPauseMillis`, default 200 ms.
- **G1 IHOP default:** 45% old-gen occupancy triggers concurrent marking.
- **Humongous object:** ≥ 50% of a G1 region.
- **ZGC:** colored pointers + load barriers; pauses independent of heap size; generational since Java 21.
- **Shenandoah:** concurrent compaction via forwarding/load-reference barriers.
- **Epsilon:** allocates, never collects (testing).
- **Reference strength:** strong > soft (memory pressure) > weak (next GC) > phantom (post-mortem cleanup).
- **PermGen → Metaspace** in Java 8; Metaspace is native memory, auto-grows.
- **GC throughput** = 1 − (GC time / total time); < ~90% is a problem.
- **Live GC check:** `jstat -gcutil <pid> 1000` (watch YGC/FGC/GCT).
- **Hot GC threads:** `top -H -p <pid>` → hex TID → `jstack`.
- **Prod-safe profiling:** JFR (~1% overhead) + Mission Control.
- **Leak analysis:** heap dump → Eclipse MAT → Path to GC Roots.
- **#1 leak cause:** growing static collections.
- **Always in prod:** `-Xms=-Xmx`, GC logs, `+HeapDumpOnOutOfMemoryError`.
- **Compressed oops** lost above ~32 GB heap.
- **Write barrier** = on reference store (generational/SATB). **Load barrier** = on reference read (ZGC relocation).
- **Safepoint:** where a thread can be paused safely; **TTSP** stalls can masquerade as GC pauses.
- **`OutOfMemoryError` isn't always heap:** read the message (Metaspace, native thread, direct buffer…).
- **`System.gc()`** only requests a GC; disable with `-XX:+DisableExplicitGC`.

---

## 16. Glossary

- **Allocation rate** — bytes of new objects created per unit time; primary driver of minor-GC frequency.
- **Card table** — bitmap marking old-gen regions ("cards") that hold references into young gen, so minor GC needn't scan all of old gen.
- **Collection set (CSet)** — the set of regions G1 collects in a given pause.
- **Compaction** — sliding live objects together to remove fragmentation.
- **Compressed oops** — 32-bit encoded object pointers, used when heap ≤ ~32 GB, saving memory.
- **Concurrent GC** — collector threads run alongside application threads.
- **Eden** — young-gen space where new objects are allocated.
- **Escape analysis** — JIT analysis proving an object doesn't escape a method, enabling scalar replacement (no heap allocation).
- **Evacuation** — copying live objects out of a region into a fresh one (G1/ZGC/Shenandoah compaction mechanism).
- **Full GC** — collection of the entire heap (and usually Metaspace), typically compacting; most expensive event.
- **Generational hypothesis** — most objects die young; few old→young references exist.
- **GC roots** — starting points for reachability tracing (stack locals, statics, threads, JNI refs, monitors).
- **GC throughput** — fraction of time spent in application vs. GC.
- **Humongous object** — object ≥ 50% of a G1 region, allocated specially in old gen.
- **IHOP** — Initiating Heap Occupancy Percent; old-gen occupancy that triggers G1 concurrent marking.
- **Load barrier** — code on reference reads (ZGC) that fixes up relocated references.
- **Mark-sweep / mark-compact / copying** — the three primitive GC algorithms.
- **Metaspace** — native-memory region for class metadata (replaced PermGen in Java 8).
- **Minor / Major GC** — young-only / old-gen collection.
- **Premature promotion** — medium-lived objects promoted to old gen before dying, causing old-gen churn.
- **Promotion / tenuring** — moving surviving young objects to old gen after enough survivals.
- **Reachability** — whether an object is transitively referenced from a GC root; unreachable = garbage.
- **Remembered set (RSet)** — per-region record of incoming references, enabling independent region collection.
- **Safepoint** — an execution point where a thread's state is walkable, allowing it to be paused.
- **SATB** — Snapshot-At-The-Beginning; concurrent-marking correctness technique (G1/Shenandoah).
- **Scalar replacement** — JIT breaking an object into its fields to avoid heap allocation.
- **Stop-the-world (STW)** — a pause where all application threads are suspended for GC.
- **Survivor spaces (S0/S1)** — young-gen spaces objects are copied between while aging.
- **TLAB** — Thread-Local Allocation Buffer; per-thread lock-free allocation region in Eden.
- **Time-to-safepoint (TTSP)** — time for all threads to reach a safepoint after a pause request.
- **Tri-color marking** — white/gray/black scheme tracking marking progress during concurrent GC.
- **Write barrier** — code on reference stores tracking cross-gen refs and concurrent-marking changes.

---

*End of guide. Suggested revision path for interviews: sections 5 → 6 → 8 → 9, then the question bank (14) and flashcards (15).*