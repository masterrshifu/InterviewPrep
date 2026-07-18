
# Java Multithreading & Concurrency — Complete Interview Guide

A deep, interview-ready reference covering 200 questions across the full concurrency stack: from thread basics and the Java Memory Model to locks, executors, `CompletableFuture`, concurrent collections, Spring Boot, Kafka, JVM internals, and production debugging.

**How to use this guide:** Each answer is written to be understood, not memorized. Read the "why" behind each concept. Interviewers reward candidates who can explain trade-offs, not just definitions. Code snippets are runnable Java 8+ unless noted.
 
---

## Table of Contents

1. [Core Multithreading (Q1–10)](#1-core-multithreading)
2. [Synchronization Basics (Q11–20)](#2-synchronization-basics)
3. [Java Memory Model (Q21–30)](#3-java-memory-model-jmm)
4. [volatile (Q31–40)](#4-volatile)
5. [Atomic Classes & CAS (Q41–50)](#5-atomic-classes--cas)
6. [Locks (Q51–60)](#6-locks)
7. [wait(), notify() (Q61–70)](#7-wait-notify)
8. [Deadlock, Starvation & Livelock (Q71–80)](#8-deadlock-starvation--livelock)
9. [Executor Framework (Q81–90)](#9-executor-framework)
10. [CompletableFuture (Q91–100)](#10-completablefuture)
11. [Concurrent Collections (Q101–110)](#11-concurrent-collections)
12. [Synchronizers (Q111–120)](#12-synchronizers)
13. [ThreadLocal (Q121–130)](#13-threadlocal)
14. [Parallelism (Q131–140)](#14-parallelism)
15. [Spring Boot Concurrency (Q141–150)](#15-spring-boot-concurrency)
16. [Production Scenarios (Q151–160)](#16-production-scenarios)
17. [Kafka & Concurrency (Q161–170)](#17-kafka--concurrency)
18. [Coding Questions (Q171–180)](#18-coding-questions)
19. [JVM & Internals (Q181–190)](#19-jvm--internals)
20. [Debugging & Performance (Q191–200)](#20-debugging--performance)
---

## 1. Core Multithreading

### 1. What is a thread? How is it different from a process?

A **process** is an independent program in execution with its own memory space (heap, code, data, open files, OS resources). A **thread** is the smallest unit of execution *within* a process. A process can contain many threads.

The key differences:

- **Memory:** Threads of the same process **share** the heap, static variables, and code segment, but each thread has its **own stack** and program counter. Processes have completely separate address spaces.
- **Communication:** Threads communicate cheaply via shared memory. Processes need IPC (pipes, sockets, shared memory segments), which is heavier.
- **Cost:** Creating a thread is far cheaper than a process. Context switching between threads is cheaper than between processes (no address-space switch, TLB flush is smaller).
- **Isolation:** A crash in one process doesn't kill another. A crash (or corruption) in one thread can bring down the whole process because they share memory.
  In Java, each `Thread` maps (typically) to a native OS thread. The shared heap is exactly why concurrency is hard — visibility and race conditions arise from shared mutable state.

### 2. Why do we need multithreading?

Three main motivations:

- **Responsiveness:** Keep a UI or server responsive while long work happens in the background (e.g., a web server handling one request per thread while others continue).
- **Throughput / parallelism:** On multi-core CPUs, run independent work simultaneously to finish faster (CPU-bound tasks like image processing, matrix math).
- **Resource utilization:** For I/O-bound work (DB calls, network, disk), a thread blocked on I/O lets other threads use the CPU instead of the core sitting idle.
  Without threads, a program does one thing at a time. Modern hardware has many cores; a single-threaded program leaves them idle. The trade-off is complexity: race conditions, deadlocks, visibility bugs, and harder debugging.

### 3. What are the different ways to create a thread in Java?

1. **Extend `Thread`** and override `run()`:
```java
   class MyThread extends Thread {
       public void run() { System.out.println("running"); }
   }
   new MyThread().start();
```
Downside: uses up your single inheritance slot.

2. **Implement `Runnable`** and pass to a `Thread` (preferred over extending):
```java
   Runnable r = () -> System.out.println("running");
   new Thread(r).start();
```

3. **Implement `Callable<V>`** — returns a value and can throw checked exceptions; submitted to an `ExecutorService`, returns a `Future`.
4. **`ExecutorService` / thread pools** — the recommended production approach; you submit tasks and let the pool manage threads.
5. **`CompletableFuture`** — for async composition.
6. **Virtual threads (Java 21+, Project Loom)** — `Thread.ofVirtual().start(...)` or `Executors.newVirtualThreadPerTaskExecutor()`; lightweight threads managed by the JVM for massive I/O concurrency.
   **Preferred:** `Runnable`/`Callable` with an executor, because it decouples the task from the thread and lets you manage the thread lifecycle centrally.

### 4. Runnable vs Callable

| Aspect | `Runnable` | `Callable<V>` |
|---|---|---|
| Method | `void run()` | `V call()` |
| Return value | None | Returns a result of type `V` |
| Checked exceptions | Cannot throw | Can throw checked exceptions |
| Introduced | Java 1.0 | Java 5 (`java.util.concurrent`) |
| Used with | `Thread`, executors | `ExecutorService.submit()` → `Future<V>` |

Use `Callable` when the task produces a result or may fail with a checked exception. Both can be submitted to an executor; `submit(Runnable)` returns a `Future<?>` whose `get()` yields `null`.

### 5. start() vs run()

- **`start()`** creates a new thread (native thread), and the JVM schedules it; `run()` executes **on that new thread**. You can call `start()` only once per `Thread` object.
- **`run()`** is just an ordinary method call. Calling `run()` directly executes the code **on the current thread** — no new thread is created, no concurrency.
```java
Thread t = new Thread(() -> System.out.println(Thread.currentThread().getName()));
t.run();    // prints "main"  — runs on current thread
t.start();  // prints "Thread-0" — runs on a new thread
```

Calling `run()` when you meant `start()` is a classic bug: the code works but is entirely sequential.

### 6. What happens if start() is called twice?

It throws `IllegalThreadStateException`. A `Thread` object represents a single thread of execution and cannot be restarted once it has been started (even after it has terminated). To run the same task again, create a new `Thread`, or better, submit the `Runnable` to an executor which reuses pooled threads.

### 7. What are the lifecycle states of a thread?

`Thread.State` enum:

- **NEW** — created but `start()` not yet called.
- **RUNNABLE** — eligible to run; either running or waiting for CPU. (Java doesn't distinguish "ready" from "running"; both are RUNNABLE. A thread blocked on I/O is also RUNNABLE from the JVM's view.)
- **BLOCKED** — waiting to acquire a monitor lock (to enter a `synchronized` block/method).
- **WAITING** — waiting indefinitely for another thread's signal: `wait()`, `join()`, `LockSupport.park()` without timeout.
- **TIMED_WAITING** — waiting for a bounded time: `sleep(ms)`, `wait(ms)`, `join(ms)`, `parkNanos`.
- **TERMINATED** — `run()` has completed (normally or via exception).
  Transitions: NEW → RUNNABLE (start) → (BLOCKED/WAITING/TIMED_WAITING ↔ RUNNABLE) → TERMINATED.

### 8. What is thread scheduling?

Thread scheduling is how the OS/JVM decides which RUNNABLE thread gets CPU time and for how long. The JVM delegates to the **OS scheduler**, which on modern systems is **preemptive** and typically priority-plus-time-slice based. A running thread can be preempted when its time slice expires or a higher-priority thread becomes runnable.

Java gives you almost no direct control — you can set priorities and yield hints, but the actual scheduling policy is OS-specific and not guaranteed. Never write correctness-dependent code that assumes a particular scheduling order.

### 9. What is thread priority? Does Java guarantee priority execution?

Each thread has a priority from `Thread.MIN_PRIORITY` (1) to `Thread.MAX_PRIORITY` (10), default `NORM_PRIORITY` (5). Priority is a **hint** to the scheduler that higher-priority threads should be preferred.

**Java does not guarantee priority-based execution.** Priority handling depends entirely on the OS. Some OSes ignore Java priorities or map many Java levels to a few native levels. Relying on priority for correctness is a bug; at best it's a performance nudge. Priority can also cause **starvation** of low-priority threads.

### 10. What is a daemon thread? Give real-world examples.

A **daemon thread** is a background thread that does not prevent the JVM from exiting. The JVM terminates when only daemon threads remain (all user/non-daemon threads have finished). Set with `thread.setDaemon(true)` **before** `start()`.

Key points:
- User (non-daemon) threads keep the JVM alive; daemon threads don't.
- When the JVM exits, daemon threads are abruptly stopped — `finally` blocks may not run — so don't do critical I/O or resource cleanup in daemons.
  Real-world examples: the **garbage collector**, JIT compiler threads, `finalizer` thread, timer/heartbeat threads, background log flushers, and monitoring/metrics threads. Thread pools created via `Executors` use non-daemon threads by default (which is why forgetting `shutdown()` can hang the JVM).

---

## 2. Synchronization Basics

### 11. What is synchronization?

Synchronization is coordinating access to shared mutable state so that concurrent threads don't corrupt data or observe inconsistent state. In Java, the `synchronized` keyword provides **mutual exclusion** (only one thread in the critical section at a time) *and* **visibility** (changes made by one thread become visible to others via happens-before edges on lock release/acquire).

### 12. Why is synchronization needed?

Because of **race conditions** on shared mutable state. Consider `count++` — it's actually read-modify-write (three steps). Two threads can both read `5`, both increment to `6`, and one update is lost. Synchronization also addresses **visibility**: without it, one thread's write may sit in a CPU register/cache and never be seen by another thread. Synchronization guarantees atomicity of the critical section and memory visibility across threads.

### 13. What is intrinsic locking (monitor lock)?

Every Java object has an associated **monitor** (intrinsic lock). `synchronized` acquires this monitor. Entering a `synchronized` block/method acquires the object's monitor; exiting releases it. Only one thread can hold a given object's monitor at a time. It is **reentrant** — a thread already holding the monitor can re-enter other synchronized code guarded by the same monitor without deadlocking itself (a hold-count is tracked). This is "intrinsic" because it's built into every object, versus explicit `Lock` objects.

### 14. Synchronized method vs synchronized block

- **Synchronized method:** locks on `this` (instance method) or the `Class` object (static method). The entire method body is the critical section.
```java
  public synchronized void m() { ... }   // locks on this
  public static synchronized void s() { ... } // locks on ClassName.class
```
- **Synchronized block:** you choose the lock object and scope. Finer-grained → better performance, and you can lock on a private dedicated object to avoid exposing your lock.
```java
  private final Object lock = new Object();
  void m() {
      // non-critical work...
      synchronized (lock) { /* critical section only */ }
  }
```

Blocks are preferred in practice: smaller critical sections reduce contention, and a private lock object prevents outside code from acquiring your lock (which can happen if you lock on `this`).

### 15. Class-level lock vs object-level lock

- **Object-level lock:** acquired via a `synchronized` instance method or `synchronized(this)`. One lock **per instance**. Threads working on *different* instances don't block each other.
- **Class-level lock:** acquired via a `synchronized` static method or `synchronized(MyClass.class)`. One lock **per class** (the `Class` object). Guards static/shared state across all instances.
  They are independent locks. A thread holding the class lock does **not** block a thread wanting an instance lock, and vice versa. To protect static data, use the class lock; to protect per-instance data, use the object lock.

### 16. Can two synchronized methods execute simultaneously?

It depends on the lock they use:
- Two synchronized **instance** methods on the **same object**: **No** — both need `this`'s monitor, so they're mutually exclusive.
- On **different objects**: **Yes** — different monitors.
- One synchronized instance method and one synchronized **static** method: **Yes**, they can run concurrently — one locks `this`, the other locks the `Class` object (different locks).
### 17. Can synchronized methods on different objects execute simultaneously?

Yes. Instance-level `synchronized` locks on `this`, so different instances have different monitors. Two threads each operating on their own object run concurrently with no mutual exclusion. Mutual exclusion only applies when threads contend for the **same** lock object.

### 18. What happens if an exception occurs inside a synchronized block?

The monitor is **automatically released**. The JVM guarantees the lock is released when control leaves the synchronized block — whether normally or via an exception (the compiler emits a `monitorexit` in an implicit exception handler). This is a nice property versus explicit `Lock`, where you must manually `unlock()` in a `finally` block. Note: the shared state may be left in an inconsistent/partially-updated state, so you still need to reason about invariants — but the lock itself won't leak.

### 19. Can constructors be synchronized?

No — `synchronized` is not allowed on constructors; it's a compile error. Rationale: only the constructing thread has a reference to the object until construction completes, so there's nothing to synchronize against at that point. If you need synchronization during construction, use a `synchronized` block inside the constructor on some lock object. (Beware publishing `this` before the constructor finishes — that can leak a partially-constructed object.)

### 20. What objects can be used as locks?

**Any non-null Java object reference** can serve as a monitor lock. Common practice:
- A `private final Object lock = new Object();` — best practice for a dedicated lock.
- `this` — convenient but exposes your lock to external code.
- `ClassName.class` — for static/class-level locking.
  **Avoid** locking on:
- **Boxed primitives / String literals** (`Integer`, `Boolean`, interned `String`) — they may be cached/shared across unrelated code, causing accidental contention or deadlock.
- **Mutable references you reassign** — the lock identity changes, breaking mutual exclusion.
  Use a `final` field so the lock reference can't change.

---

## 3. Java Memory Model (JMM)

### 21. What is the Java Memory Model?

The JMM (specified in JLS Chapter 17, defined by JSR-133) is the abstract contract that defines **how and when** changes made by one thread to memory become visible to other threads, and what reorderings the compiler/JVM/CPU are allowed to perform. It defines the **happens-before** relationship. It exists because compilers and CPUs aggressively optimize (reorder, cache in registers) for single-threaded correctness, but those optimizations break naive multithreaded assumptions. The JMM tells you exactly which guarantees you get (via `volatile`, `synchronized`, `final`, atomics) so you can write correct concurrent code without knowing the underlying hardware.

### 22. What is memory visibility?

Visibility is the guarantee that a write performed by one thread is observable by another thread. Without synchronization, there is **no guarantee** a reader ever sees a writer's update — the value may live in a CPU register or per-core cache and never be flushed, or the reader may cache a stale copy. `volatile`, `synchronized`, and atomics establish happens-before edges that force the write to be visible. The classic symptom of a visibility bug is a loop `while (!flag) {}` that never terminates because `flag`'s update is never seen.

### 23. What is atomicity?

Atomicity means an operation happens entirely or not at all — no other thread can observe a partial/intermediate state. In Java:
- Reads/writes of most primitives and references are atomic.
- **`long` and `double` are NOT guaranteed atomic** for reads/writes on 32-bit VMs (they may be split into two 32-bit operations) — unless declared `volatile`.
- Compound operations like `count++`, `check-then-act`, and `x = x + 1` are **not** atomic (multiple steps). Make them atomic with `synchronized`, `Atomic*` classes, or locks.
### 24. What is ordering?

Ordering concerns the sequence in which memory operations *appear* to execute. Within a single thread, the program **appears** to run in program order (as-if-serial). But across threads, the compiler and CPU may **reorder** independent instructions, and different threads may observe operations in different orders. The JMM constrains ordering via happens-before: e.g., a `volatile` write cannot be reordered after a subsequent `volatile` read/write, and everything before the write is visible after a matching read.

### 25. What is the happens-before relationship?

Happens-before is the JMM's core ordering guarantee: if action A **happens-before** action B, then A's memory effects are visible to and ordered before B. Key happens-before rules:

- **Program order:** each action in a thread happens-before later actions in that same thread.
- **Monitor lock:** releasing a lock happens-before every subsequent acquisition of the *same* lock.
- **Volatile:** a write to a `volatile` field happens-before every subsequent read of that field.
- **Thread start:** `Thread.start()` happens-before any action in the started thread.
- **Thread join:** all actions in a thread happen-before another thread returns from `join()` on it.
- **Transitivity:** if A hb B and B hb C, then A hb C.
  If two accesses to shared data aren't ordered by happens-before and at least one is a write, you have a **data race** and behavior is undefined regarding visibility/ordering.

### 26. What are memory barriers?

Memory barriers (fences) are low-level CPU/compiler instructions that enforce ordering and visibility by preventing certain reorderings and flushing/invalidating caches. The JMM's guarantees are implemented via barriers. Conceptual types:
- **LoadLoad, LoadStore, StoreStore, StoreLoad** barriers.
- A `volatile` write emits a **StoreStore** before and a **StoreLoad** after (the expensive one); a `volatile` read emits **LoadLoad/LoadStore** after.
  You rarely use barriers directly (except via `VarHandle.fullFence()` etc.); `volatile`/`synchronized`/atomics insert them for you. The takeaway: barriers are how "happens-before" becomes real on hardware.

### 27. Why does the CPU cache create concurrency issues?

Each core has its own L1/L2 cache. When a thread writes a variable, the new value may sit in that core's cache and not be immediately visible to other cores reading their own cached copies — leading to **stale reads** (a visibility problem). Cache coherence protocols (MESI) eventually reconcile, but "eventually" isn't a guarantee your code can rely on without memory barriers. `volatile`/synchronization insert barriers that force the write to be flushed and other caches to be invalidated, making the value visible. This core-local caching is the hardware root of visibility bugs.

### 28. Stack memory vs heap memory in multithreading

- **Stack:** each thread has its **own** stack holding local variables, method frames, and primitive locals / object references. Because it's thread-private, local variables are inherently thread-safe (no sharing).
- **Heap:** shared across all threads; holds all objects and instance/static fields. This is where concurrency problems arise — shared objects on the heap need synchronization.
  A local *reference* is on the stack, but the *object* it points to lives on the heap. If two threads get references to the same heap object, they share it. Immutability and thread-confinement (keeping objects on one thread's stack) are strategies to avoid heap sharing hazards.

### 29. What are instruction reordering problems?

Compilers, JIT, and CPUs reorder independent instructions for performance. This is invisible in single-threaded code (as-if-serial) but can break multithreaded code. Classic example — an object publication bug:

```java
// Thread A
config = new Config();   // (1) allocate, (2) init fields, (3) assign reference
// steps 2 and 3 can be reordered
```

Another thread might see a non-null `config` (step 3 done) before its fields are initialized (step 2). This is precisely why **double-checked locking needs `volatile`** (Q38). Reordering problems are fixed by establishing happens-before (volatile/synchronized/final).

### 30. Explain visibility problems with an example.

```java
class Task {
    private boolean running = true;      // not volatile
    void stop() { running = false; }
    void run() {
        while (running) { /* work */ }   // may loop forever
    }
}
```

Thread A runs `run()`, thread B calls `stop()`. Without `volatile`, the JIT may hoist `running` into a register (since it doesn't see it change within the loop), so thread A never observes B's write and **loops forever**. Fix: declare `running` as `volatile` (or use an `AtomicBoolean`), which establishes a happens-before edge so B's write is visible to A.
 
---

## 4. volatile

### 31. What is volatile?

`volatile` is a field modifier that tells the JVM a variable may be accessed by multiple threads and must not be cached in a thread-local way. Every read goes to main memory (up-to-date value) and every write is flushed to main memory, with memory barriers preventing reordering around it. It provides **visibility** and **ordering** guarantees but **not** atomicity for compound operations.

### 32. What guarantees does volatile provide?

1. **Visibility:** a write to a volatile is immediately visible to subsequent reads by other threads (happens-before edge between write and later read).
2. **Ordering / no reordering:** reads/writes of a volatile can't be reordered with each other, and (since JSR-133) writes before a volatile write and reads after a volatile read can't cross the barrier — so a volatile write "publishes" everything written before it.
3. **Atomic access for `long`/`double`:** a volatile `long`/`double` read/write is atomic even on 32-bit VMs.
   It does **not** provide mutual exclusion or make compound operations atomic.

### 33. Can volatile make operations atomic?

Only for **single reads and single writes** of the field itself (including `long`/`double`). It does **not** make compound operations (`x++`, `x = x + 1`, check-then-act) atomic, because those involve read → modify → write, and another thread can interleave between the steps. For atomic compound updates use `AtomicInteger`/`AtomicLong` (CAS) or `synchronized`.

### 34. volatile vs synchronized

| | `volatile` | `synchronized` |
|---|---|---|
| Guarantees | Visibility + ordering | Visibility + ordering + **mutual exclusion (atomicity)** |
| Blocking | Never blocks | Can block (only one thread in critical section) |
| Scope | A single field | A block/method (any compound logic) |
| Compound ops | Not safe | Safe |
| Performance | Cheaper (no locking) | More expensive (lock acquire/release, possible contention) |

Use `volatile` for a simple flag or a safely-published reference where only one thread writes. Use `synchronized` (or locks/atomics) when you need atomic multi-step updates.

### 35. When should volatile be preferred?

- A **status/stop flag** written by one thread and read by others (`volatile boolean shutdown`).
- **Safe publication** of an immutable object reference (write the fully-built object to a volatile field; readers see it fully initialized).
- The **`volatile` in double-checked locking** for a singleton.
- One-writer / many-reader scenarios where the value doesn't depend on its previous value.
  Prefer volatile when you need cheap visibility but not atomic compound updates and not mutual exclusion.

### 36. Can volatile variables participate in compound operations?

They can be *used* in compound operations, but volatile does not make those operations atomic/thread-safe. `volatile int count; count++;` still has the lost-update race because `count++` is read-modify-write. If multiple threads mutate based on the current value, volatile is insufficient — use atomics or locks.

### 37. Can multiple threads safely increment a volatile integer?

No. `volatileInt++` is not atomic — two threads can read the same value and both write back the same incremented value, losing an update. Volatile guarantees each thread sees the latest value at the moment of reading, but not that the read-modify-write is indivisible. Use `AtomicInteger.incrementAndGet()`, `LongAdder`, or a `synchronized` block instead.

### 38. Why is double-checked locking broken without volatile?

Classic DCL singleton:
```java
class Singleton {
    private static volatile Singleton instance;  // volatile is essential
    static Singleton get() {
        if (instance == null) {                  // 1st check (no lock)
            synchronized (Singleton.class) {
                if (instance == null)            // 2nd check (locked)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```
`instance = new Singleton()` is (1) allocate memory, (2) run constructor, (3) publish reference. Steps 2 and 3 can be **reordered**. Without `volatile`, another thread doing the first (unlocked) check could see a **non-null but not-yet-constructed** object and use it — returning a partially initialized singleton. `volatile` prevents that reordering (and provides visibility), making DCL correct on Java 5+.

### 39. How does volatile prevent instruction reordering?

The JMM requires memory barriers around volatile accesses. A volatile **write** has a `StoreStore` barrier before it (all prior writes complete first) and a `StoreLoad` barrier after it. A volatile **read** has `LoadLoad`/`LoadStore` barriers after it. These barriers forbid the compiler/CPU from moving normal reads/writes across the volatile access. Concretely: everything written before a volatile write is guaranteed visible to a thread that reads that volatile afterward — enabling safe publication.

### 40. Real-world use cases of volatile

- **Shutdown/stop flags:** `volatile boolean running`.
- **Double-checked locking** singleton reference.
- **Configuration hot-swap:** publish a new immutable config object via a volatile reference.
- **Status publishing:** a background thread writes progress/heartbeat, monitors read it.
- **`AtomicX` internals:** the underlying `value` fields are volatile.
- **One-time initialization guards** and lazy-loaded flags where only visibility (not atomic mutation) is needed.
---

## 5. Atomic Classes & CAS

### 41. What are Atomic classes?

Classes in `java.util.concurrent.atomic` that provide **lock-free, thread-safe** operations on single variables using CAS (compare-and-swap). Examples: `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference<V>`, `AtomicIntegerArray`, `AtomicStampedReference`, and adders/accumulators like `LongAdder`, `DoubleAdder`, `LongAccumulator`. They give you atomic read-modify-write (`incrementAndGet`, `compareAndSet`, `getAndAdd`, `updateAndGet`) without `synchronized`, avoiding lock overhead and blocking.

### 42. How does AtomicInteger work internally?

It holds a `private volatile int value` (volatile → visibility). Mutating methods use CAS via `Unsafe`/`VarHandle`:
```java
public final int incrementAndGet() {
    int prev, next;
    do {
        prev = get();            // read current volatile value
        next = prev + 1;
    } while (!compareAndSet(prev, next));  // retry until CAS succeeds
    return next;
}
```
`compareAndSet(expected, new)` maps to a single hardware CAS instruction (e.g., `LOCK CMPXCHG` on x86). If another thread changed the value in between, CAS fails and the loop retries with the fresh value — a **spin/retry** (optimistic) loop instead of a lock.

### 43. What is Compare-And-Swap (CAS)?

CAS is an atomic CPU instruction with three operands: a memory location V, an expected old value A, and a new value B. Atomically: **if V == A, set V = B and return success; else do nothing and return failure** (usually returning the current value). It lets you do read-modify-write atomically without locks: read the current value, compute the new value, and CAS — if it fails, someone else changed it, so retry. It's the foundation of lock-free algorithms and the `Atomic*` classes.

### 44. Why is CAS faster than synchronized?

- **No blocking / no context switch:** a failed CAS just retries in a spin loop; a failed lock acquisition may park the thread (OS context switch — expensive).
- **No lock bookkeeping:** no monitor acquire/release, no wait queues under low contention.
- **Optimistic:** assumes conflicts are rare; under low-to-moderate contention it's much cheaper.
  Caveat: under **very high contention**, CAS retry loops can waste CPU (many failed spins), and a lock (which parks threads) can sometimes be better. That's the motivation for `LongAdder`.

### 45. What is the ABA problem?

CAS only checks that the value *equals* the expected value, not that it never changed. If a value goes A → B → A between your read and your CAS, the CAS still succeeds even though state changed and back — you miss the intervening modification. This matters for lock-free structures (e.g., a popped-and-repushed node in a stack) where "same pointer value" doesn't mean "unchanged state." Simple counters don't care; pointer-based structures do.

### 46. How does AtomicStampedReference solve ABA?

`AtomicStampedReference<V>` pairs the reference with an **integer stamp (version)**. Every update increments the stamp. CAS checks **both** the reference and the stamp:
```java
asr.compareAndSet(expectedRef, newRef, expectedStamp, newStamp);
```
So A → B → A changes the stamp (e.g., 0 → 1 → 2), and a CAS expecting stamp 0 fails even though the reference is A again. This detects the intervening change. (`AtomicMarkableReference` is a lighter variant with a boolean mark instead of a full version.)

### 47. AtomicInteger vs LongAdder

- **`AtomicInteger`/`AtomicLong`:** single volatile value updated by CAS. Under **high contention**, many threads CAS the same cell → lots of failed retries → cache-line ping-pong → poor scalability. Great when you also need the exact current value cheaply and contention is low.
- **`LongAdder`:** maintains an array of **cells**; threads hash to different cells and update them independently, spreading contention. `sum()` adds all cells. Far better **write throughput under high contention**, at the cost of a slightly more expensive/approximate `sum()` (not a single atomic snapshot). Prefer `LongAdder` for hot counters/metrics; prefer `AtomicLong` when you frequently need an exact instantaneous value or contention is low.
### 48. AtomicReference use cases

`AtomicReference<V>` gives atomic CAS on an object reference. Use cases:
- **Lock-free data structures** (Treiber stack, Michael-Scott queue) using `compareAndSet` on node pointers.
- **Atomic swap of an immutable state object** — build a new immutable snapshot and CAS it in (copy-on-write style for a small object).
- **Atomic lazy initialization** / caching with `updateAndGet`/`accumulateAndGet`.
- **Non-blocking algorithms** where you need optimistic update of a whole object rather than a number.
### 49. Can CAS fail?

Yes — CAS fails when the current value doesn't match the expected value, i.e., another thread modified it in between. Failure is normal and expected; the standard pattern is a retry loop (re-read, recompute, re-CAS). "Failure" here means "someone else won the race," not an error. There's also **spurious**-free semantics for strong CAS, but `weakCompareAndSet` may fail spuriously (allowed to fail even when values match) in exchange for cheaper implementation on some platforms — used only inside retry loops.

### 50. What are lock-free algorithms?

Lock-free algorithms guarantee that **at least one** thread makes progress in a bounded number of steps, without using locks — typically built on CAS retry loops. Properties:
- **Non-blocking:** a stalled/descheduled thread can't block others (no held lock).
- **No deadlock** (no locks to deadlock on); immune to priority inversion.
- Progress guarantees hierarchy: **wait-free** (every thread makes progress in bounded steps) ⊃ **lock-free** (some thread progresses) ⊃ **obstruction-free**.
  Examples: `ConcurrentLinkedQueue`, `AtomicInteger` counters, Treiber stack. Trade-off: harder to design correctly (ABA, memory reclamation) and can burn CPU under heavy contention.

---

## 6. Locks

### 51. ReentrantLock vs synchronized

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Acquire/release | Implicit (block scope) | Explicit `lock()`/`unlock()` |
| Reentrant | Yes | Yes |
| Try-acquire | No | `tryLock()` (optionally with timeout) |
| Interruptible acquire | No | `lockInterruptibly()` |
| Fairness | No (always unfair) | Optional (`new ReentrantLock(true)`) |
| Multiple conditions | One implicit wait-set | Multiple `Condition` objects |
| Auto-release on exception | Yes (JVM) | No — must `unlock()` in `finally` |
| Lock status/metrics | No | `isLocked()`, `getHoldCount()`, `hasQueuedThreads()` |

`synchronized` is simpler and safer (auto-release). `ReentrantLock` gives advanced control at the cost of manual management.

### 52. Why use ReentrantLock?

When you need capabilities `synchronized` lacks: **timed** lock attempts (`tryLock(timeout)`) to avoid indefinite blocking/deadlock, **interruptible** acquisition, **fairness** to prevent starvation, **multiple condition variables** (e.g., "not full" and "not empty" for a bounded buffer), non-block-structured locking (acquire in one method, release in another), and lock instrumentation/metrics. If you don't need any of these, prefer `synchronized`.

### 53. What is fairness in ReentrantLock?

A **fair** lock (`new ReentrantLock(true)`) grants the lock in roughly **FIFO** order — the longest-waiting thread acquires next. This prevents starvation but reduces throughput (more context switching, can't do "barging"). The default **unfair** lock allows **barging**: a thread can grab a just-released lock even if others have been waiting, which yields higher throughput but risks starving some threads. Use fair only when starvation is a real concern; otherwise unfair is faster.

### 54. What is tryLock()?

`tryLock()` attempts to acquire the lock and returns **immediately** with `true` (acquired) or `false` (not acquired) instead of blocking. `tryLock(timeout, unit)` waits up to the timeout, and is responsive to interruption. Uses: avoid deadlock (acquire multiple locks with tryLock and back off on failure), implement best-effort/optional locking, and keep responsiveness. Always pair with `finally`:
```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { /* work */ } finally { lock.unlock(); }
} else { /* couldn't get lock — do something else */ }
```

### 55. What is lockInterruptibly()?

`lockInterruptibly()` acquires the lock but allows the waiting thread to be **interrupted** while blocked — it throws `InterruptedException` if another thread interrupts it before it acquires. Plain `synchronized` and `lock()` are **not** interruptible (a thread can wait forever). This is essential for cancellable tasks and avoiding permanent hangs (e.g., letting a shutdown interrupt threads blocked on lock acquisition).

### 56. What is ReadWriteLock?

`ReadWriteLock` (impl: `ReentrantReadWriteLock`) provides two locks: a **read (shared) lock** and a **write (exclusive) lock**.
- Multiple threads can hold the **read** lock simultaneously (as long as no writer holds it).
- The **write** lock is exclusive — only one writer, and no readers concurrently.
  This improves throughput when reads vastly outnumber writes, because readers don't block each other. It supports lock downgrading (write → read) but not upgrading (read → write causes deadlock). Fairness and reentrancy are configurable.

### 57. When should ReadWriteLock be used?

When you have a **read-heavy** shared structure with infrequent writes — e.g., a cache, configuration store, or lookup table read by many threads and updated rarely. Concurrent reads then proceed in parallel instead of serializing. If writes are frequent, the overhead and writer-starvation risk can make it **slower** than a plain lock, and readers still block during writes. For very read-heavy cases, also consider `StampedLock` (optimistic reads) or copy-on-write structures.

### 58. What is StampedLock?

`StampedLock` (Java 8) is a capability-based lock returning a **stamp** (long) from acquisition, used to release/validate. It supports three modes: **writing** (exclusive), **reading** (shared), and **optimistic reading**. It's **not reentrant** and doesn't implement the `Lock` interface. Its headline feature is **optimistic read** (`tryOptimisticRead()`), which acquires no actual lock — you read, then `validate(stamp)` to check no write intervened; if it did, fall back to a real read lock. This makes read-mostly workloads very fast. Caveats: no reentrancy, no condition support, and easy to misuse.

### 59. Optimistic read vs pessimistic read

- **Pessimistic read** (`readLock()`): actually acquires a shared lock; blocks writers while held. Guarantees a consistent read but has locking overhead and blocks writers.
- **Optimistic read** (`tryOptimisticRead()` on `StampedLock`): acquires **no lock**; you read the fields then call `validate(stamp)`. If no writer intervened, your read was consistent and essentially free. If validation fails, you retry or upgrade to a pessimistic read lock.
  Optimistic read wins when writes are rare (validation rarely fails), giving lock-free reads. Pessimistic is safer when writes are common.
```java
long stamp = sl.tryOptimisticRead();
int x = this.x, y = this.y;         // read fields
if (!sl.validate(stamp)) {          // a write happened
    stamp = sl.readLock();
    try { x = this.x; y = this.y; } finally { sl.unlockRead(stamp); }
}
```

### 60. Why is unlocking inside finally important?

With explicit `Lock`s, if the critical section throws (or returns early) and you don't `unlock()`, the lock is **never released** → every other thread blocks forever (a permanent deadlock/hang). Putting `unlock()` in `finally` guarantees release on every exit path:
```java
lock.lock();
try { /* critical section */ }
finally { lock.unlock(); }
```
Also acquire the lock **outside** the `try` (or as the first statement) so you don't call `unlock()` when `lock()` itself failed. This is the manual equivalent of the automatic release `synchronized` gives you.
 
---

## 7. wait(), notify()

### 61. wait() vs sleep()

| | `wait()` | `sleep()` |
|---|---|---|
| Class | `Object` | `Thread` |
| Releases lock? | **Yes** — releases the monitor it's called on | **No** — keeps all held locks |
| Must hold monitor? | Yes — must be inside `synchronized` on that object | No |
| Wakes on | `notify()`/`notifyAll()`, timeout, interrupt, spurious | Timeout elapses (or interrupt) |
| Purpose | Inter-thread coordination (wait for a condition) | Pause execution for a duration |

The critical difference: `wait()` **releases the lock** so another thread can change the condition and notify; `sleep()` holds all locks (sleeping while holding a lock is a common cause of contention).

### 62. wait() vs notify() vs notifyAll()

- **`wait()`** — the current thread releases the monitor and suspends until signaled/timed-out/interrupted.
- **`notify()`** — wakes **one** arbitrary thread waiting on that monitor (you don't control which). Risk: if the woken thread's condition isn't the one satisfied, it may go back to waiting while another eligible thread stays asleep → possible missed-signal/liveness bug.
- **`notifyAll()`** — wakes **all** waiting threads; they re-contend for the lock and each re-checks its condition. Safer in general.
  All three must be called while holding the object's monitor.

### 63. Why must wait() be inside synchronized?

Because `wait()` operates on the object's monitor — it atomically releases the monitor and suspends the thread; on wakeup it re-acquires the monitor before returning. If you weren't holding the monitor, there'd be nothing to release, and the JVM throws `IllegalMonitorStateException`. Deeper reason: it avoids the **lost-wakeup** race. Checking the condition and calling `wait()` must be atomic relative to another thread changing the condition and calling `notify()`; holding the monitor for both sides guarantees that atomicity.

### 64. Why should wait() always be inside a while loop?

```java
synchronized (lock) {
    while (!condition) {   // NOT if
        lock.wait();
    }
    // proceed — condition is guaranteed true here
}
```
Because when `wait()` returns, the condition **may not actually hold**:
- **Spurious wakeups** (Q65) can wake the thread with no notify.
- With `notifyAll()`, multiple threads wake but only one may find the condition true; the others must re-check and re-wait.
- Between wakeup and re-acquiring the lock, another thread may have changed the condition again.
  An `if` checks once and proceeds on a false assumption; a `while` re-checks after waking. Always use `while`.

### 65. What are spurious wakeups?

A **spurious wakeup** is when a waiting thread wakes up **without** any thread calling `notify()`/`notifyAll()` and without timeout/interrupt — an artifact permitted by the JVM/OS threading implementation. The JLS explicitly allows them. This is precisely why you must re-check the condition in a `while` loop rather than assuming that returning from `wait()` means the condition is satisfied.

### 66. Why is notifyAll() usually preferred?

`notify()` wakes only one arbitrary waiter. If multiple threads wait on the same monitor for **different conditions**, `notify()` might wake a thread whose condition isn't satisfied while leaving the right thread asleep — a **lost/missed signal** and potential deadlock/stall. `notifyAll()` wakes everyone so each re-checks its own condition; exactly one (or the right ones) proceeds and the rest re-wait. It's safer at the cost of some overhead (thundering herd). You can safely use `notify()` only when all waiters are interchangeable and wait on the same condition, and each notify enables exactly one. When in doubt, `notifyAll()`.

### 67. What happens if notify() is called before wait()?

The notification is **lost** — `notify()` only wakes threads *currently* waiting; it isn't remembered. If a thread later calls `wait()`, it will block despite the earlier notify (a **missed signal**), possibly forever. The fix is a **condition/state flag** checked in a `while` loop under the same lock: the notifying thread sets the flag before notifying, and the waiter checks the flag before/while waiting, so it won't wait if the condition already holds. This is why raw `wait/notify` is error-prone and higher-level tools (`BlockingQueue`, `CountDownLatch`, `Condition`) are preferred.

### 68. Can wait() be called outside synchronized?

No. Calling `wait()` (or `notify()`/`notifyAll()`) without holding the object's monitor throws `IllegalMonitorStateException` at runtime. You must be inside a `synchronized` block/method on the same object whose `wait()` you call.

### 69. Can sleep() release locks?

No. `Thread.sleep()` pauses the current thread for the specified time but **retains all monitors/locks** it holds. Other threads waiting on those locks stay blocked for the whole sleep. This is a key contrast with `wait()`, which releases the monitor. Sleeping while holding a lock is usually a design smell.

### 70. What exceptions are thrown by wait()?

- **`InterruptedException`** (checked) — if another thread interrupts the waiting thread; you must handle or propagate it.
- **`IllegalMonitorStateException`** (unchecked) — if the current thread doesn't own the object's monitor (called outside `synchronized`).
- `wait(long timeout)` also throws `IllegalArgumentException` if the timeout is negative.
  The same monitor-ownership rule (and `IllegalMonitorStateException`) applies to `notify()`/`notifyAll()`.

---

## 8. Deadlock, Starvation & Livelock

### 71. What is deadlock?

Deadlock is a state where two or more threads are each **blocked forever**, each waiting for a resource (lock) held by another. Classic example: thread 1 holds lock A and waits for lock B; thread 2 holds lock B and waits for lock A. Neither can proceed, neither releases, and the system is stuck. No thread makes progress.

### 72. What are Coffman conditions?

The four conditions that must **all** hold simultaneously for deadlock:
1. **Mutual exclusion** — resources are non-shareable (only one thread holds a lock at a time).
2. **Hold and wait** — a thread holds at least one resource while waiting to acquire others.
3. **No preemption** — resources can't be forcibly taken; only released voluntarily.
4. **Circular wait** — a cycle of threads each waiting on a resource held by the next.
   Breaking **any one** prevents deadlock — e.g., eliminating circular wait via global lock ordering, or eliminating hold-and-wait by acquiring all locks at once.

### 73. How can deadlocks be prevented?

- **Lock ordering:** always acquire locks in a consistent global order → breaks circular wait (most common fix).
- **Lock timeout / tryLock:** use `tryLock(timeout)` and back off/retry on failure → breaks hold-and-wait.
- **Avoid nested locks:** hold only one lock at a time when possible.
- **Acquire all locks atomically** (all-or-nothing) → breaks hold-and-wait.
- **Use higher-level constructs** (concurrent collections, `java.util.concurrent`) that avoid manual multi-lock logic.
- **Open calls:** don't call foreign/alien methods while holding a lock.
- **Reduce lock scope / lock granularity** thoughtfully.
### 74. How do you detect deadlocks in production?

- **Thread dumps** via `jstack <pid>`, `kill -3 <pid>`, or JVM tools — the JVM detects and prints "Found one Java-level deadlock" with the involved threads and locks.
- **JMX / `ThreadMXBean.findDeadlockedThreads()`** — programmatically detect deadlocked threads and alert/log.
- **Monitoring tools:** VisualVM, JConsole, Java Mission Control, JProfiler show thread states and deadlocks.
- **Symptoms:** threads stuck in BLOCKED state, hung requests, thread pool exhaustion, no CPU usage but no progress. Take a couple of dumps a few seconds apart; if the same threads are BLOCKED on the same monitors, you likely have a deadlock (or severe contention).
### 75. What is livelock?

Livelock is when threads are **not blocked** but still make **no progress** because they keep reacting to each other and repeatedly changing state. Analogy: two people in a hallway each stepping aside in the same direction, forever. Example: two threads each detect contention, release their lock, back off, retry, and collide again in lockstep. Unlike deadlock (threads are stuck/idle), livelock threads are busy (often burning CPU) but accomplishing nothing. Fix: add randomized backoff so they don't stay in sync.

### 76. What is starvation?

Starvation is when a thread is **perpetually denied** the resources/CPU it needs to progress because other threads keep taking them. Causes: unfair locks (barging threads keep winning), thread priorities (high-priority threads monopolize CPU), or a few threads holding a lock for long durations. The starved thread is runnable but never scheduled/granted the lock. Fix: fair locks, avoiding priority abuse, bounding critical-section length.

### 77. Fair lock vs unfair lock

- **Fair lock:** grants access in FIFO order of request; the longest waiter goes next. Prevents starvation; lower throughput (more context switches, no barging).
- **Unfair lock (default):** allows barging — a thread may acquire a just-freed lock ahead of longer-waiting threads. Higher throughput; risk of starving some threads.
  `ReentrantLock` and `ReentrantReadWriteLock` support both (`new ReentrantLock(true)` = fair). `synchronized` is always unfair. Choose fair only when starvation is a demonstrated problem.

### 78. Give a real-world deadlock example.

**Bank transfer:** transferring money between two accounts, locking both:
```java
void transfer(Account from, Account to, int amt) {
    synchronized (from) {
        synchronized (to) {
            from.debit(amt); to.credit(amt);
        }
    }
}
```
Thread 1 does `transfer(A, B)` (locks A, wants B); thread 2 does `transfer(B, A)` (locks B, wants A) → deadlock. **Fix:** impose a global order (e.g., lock the account with the smaller ID first), or use `tryLock` with backoff. Other real cases: nested `synchronized` in ORM/JDBC connection pools, UI thread waiting on a background thread that waits on the UI thread.

### 79. What is lock ordering?

Lock ordering (lock hierarchy) is a deadlock-prevention technique: define a **global, consistent order** in which locks must be acquired, and require all threads to acquire multiple locks in that order. This eliminates **circular wait** (a Coffman condition), so deadlock can't form. Example: order locks by a unique id/hashcode and always acquire the lower one first. If a natural order isn't obvious, use `System.identityHashCode` as a tie-breaker (with a third "tie lock" for equal hashes).

### 80. How does tryLock() help avoid deadlocks?

`tryLock()` (especially with a timeout) breaks the **hold-and-wait**/no-preemption conditions: instead of blocking forever waiting for a second lock, a thread tries to acquire it and, on failure, **releases the locks it already holds** and retries later (often with randomized backoff). Since no thread waits indefinitely while holding locks, the circular wait can't persist:
```java
if (lockA.tryLock(1, SECONDS)) {
  try {
    if (lockB.tryLock(1, SECONDS)) {
      try { /* work */ } finally { lockB.unlock(); }
    } else { /* back off, retry */ }
  } finally { lockA.unlock(); }
}
```
 
---

## 9. Executor Framework

### 81. Why should we avoid creating threads manually?

- **Cost:** each `new Thread()` maps to an OS thread (~1MB stack); creating them per task is expensive and doesn't scale. Unbounded creation → `OutOfMemoryError: unable to create native thread`.
- **No reuse:** manual threads are created and destroyed per task; pools reuse threads, amortizing creation cost.
- **No management:** no built-in queuing, throttling, lifecycle, rejection policy, or result handling.
- **Hard to tune/monitor:** you can't easily bound concurrency or observe the pool.
  The **Executor framework** decouples task submission from execution, manages a pool of reusable threads, provides queuing, back-pressure (bounded queues), rejection policies, and `Future`s. Prefer executors (or virtual threads for massive I/O).

### 82. What is ExecutorService?

`ExecutorService` is the primary interface (extends `Executor`) for managing a pool of threads and asynchronous task execution. It lets you `submit(Callable/Runnable)` → get a `Future`, `invokeAll`/`invokeAny` for batches, and manage lifecycle via `shutdown()`/`shutdownNow()`/`awaitTermination()`. Implementations: `ThreadPoolExecutor`, `ScheduledThreadPoolExecutor`, `ForkJoinPool`. You typically obtain one via the `Executors` factory (or construct `ThreadPoolExecutor` directly for full control over pool size, queue, and rejection policy).

### 83. execute() vs submit()

- **`execute(Runnable)`** (from `Executor`): fire-and-forget. Returns `void`. An uncaught exception propagates to the thread's `UncaughtExceptionHandler`.
- **`submit(Runnable/Callable)`** (from `ExecutorService`): returns a **`Future`**. You can get the result and, importantly, any exception thrown by the task is **captured** in the `Future` and rethrown (wrapped in `ExecutionException`) when you call `future.get()`. If you never call `get()`, a `submit`ted task's exception is **silently swallowed** — a common bug.
  Use `submit` when you need a result/exception; `execute` for pure fire-and-forget.

### 84. Future vs CompletableFuture

- **`Future<V>`** (Java 5): a handle to a pending result. You can `get()` (blocking), `isDone()`, `cancel()`. Limitations: **blocking `get()`**, no chaining/composition, no callback on completion, no manual completion, and clumsy exception handling.
- **`CompletableFuture<V>`** (Java 8): implements `Future` **and** `CompletionStage`. Adds non-blocking **composition/chaining** (`thenApply`, `thenCompose`, `thenCombine`), **callbacks** (`thenAccept`, `whenComplete`), **exception handling** (`exceptionally`, `handle`), manual completion (`complete()`), and combining many futures (`allOf`, `anyOf`). It enables reactive, non-blocking async pipelines that `Future` cannot.
### 85. Callable vs Future

They work together, not either/or. **`Callable<V>`** is the **task** (`V call()`) you submit — it produces a result and may throw. **`Future<V>`** is the **handle** returned by `executor.submit(callable)` — it represents the eventual result, letting you `get()` it, check `isDone()`, or `cancel()`. In short: `Callable` is what runs; `Future` is how you retrieve what it produced.

### 86. Different types of thread pools

Via `Executors` factory:
- **`newFixedThreadPool(n)`** — fixed n threads, unbounded `LinkedBlockingQueue`. Good for steady, bounded concurrency.
- **`newCachedThreadPool()`** — 0..`Integer.MAX_VALUE` threads, `SynchronousQueue`, idle threads reaped after 60s. Good for many short-lived tasks; dangerous under sustained load (unbounded thread creation).
- **`newSingleThreadExecutor()`** — one thread, sequential execution, tasks queued in order.
- **`newScheduledThreadPool(n)`** — for delayed/periodic tasks.
- **`newWorkStealingPool()` / `ForkJoinPool`** — work-stealing, for parallelism.
- **`newVirtualThreadPerTaskExecutor()`** (Java 21) — a new virtual thread per task, for massive I/O concurrency.
  Best practice: for production, construct `ThreadPoolExecutor` directly with a **bounded** queue and explicit rejection policy rather than the unbounded factory methods.

### 87. FixedThreadPool vs CachedThreadPool

| | FixedThreadPool | CachedThreadPool |
|---|---|---|
| Core/max threads | Fixed `n` | 0 to `Integer.MAX_VALUE` |
| Queue | Unbounded `LinkedBlockingQueue` | `SynchronousQueue` (no capacity) |
| Idle threads | Kept alive | Terminated after 60s |
| Best for | Steady load, cap concurrency | Bursts of short async tasks |
| Risk | Unbounded **queue** growth → OOM if tasks pile up | Unbounded **thread** creation → OOM/thrash under sustained load |

Fixed bounds threads but not the queue; Cached bounds neither. Both can OOM under the wrong workload — hence prefer a custom bounded pool.

### 88. ScheduledThreadPool use cases

`ScheduledThreadPoolExecutor` runs tasks after a delay or periodically:
- `schedule(task, delay, unit)` — one-shot after a delay.
- `scheduleAtFixedRate(task, initialDelay, period, unit)` — runs every `period` (fixed **rate**; if a run overruns, the next may start immediately/back-to-back).
- `scheduleWithFixedDelay(task, initialDelay, delay, unit)` — waits `delay` **between** the end of one run and the start of the next.
  Use cases: periodic health checks/heartbeats, cache eviction/refresh, metrics reporting, polling, cleanup jobs, retry scheduling, session timeouts. It's the modern replacement for `java.util.Timer` (which uses a single thread and dies on an uncaught exception).

### 89. How should ExecutorService be shut down?

Graceful two-phase shutdown:
```java
executor.shutdown();                    // stop accepting new tasks; finish queued ones
try {
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow();         // interrupt running tasks, drop queued
        if (!executor.awaitTermination(60, TimeUnit.SECONDS))
            log.error("Pool did not terminate");
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt(); // restore interrupt status
}
```
- `shutdown()` — orderly: no new tasks, existing ones complete.
- `shutdownNow()` — attempts to interrupt running tasks and returns queued tasks not started.
- Forgetting to shut down keeps non-daemon threads alive → the JVM won't exit. (In Spring, a `@Bean` `ThreadPoolTaskExecutor` handles this with `setWaitForTasksToCompleteOnShutdown`.)
### 90. What happens if a task throws an exception?

- **Via `execute(Runnable)`:** the exception propagates up, the worker thread terminates, and the pool's `UncaughtExceptionHandler` (or default) handles it — often just printing the stack trace. The pool replaces the dead worker. `ThreadPoolExecutor.afterExecute(Runnable, Throwable)` can be overridden to observe it.
- **Via `submit(...)`:** the exception is **captured** in the returned `Future`. The worker thread is **not** killed. The exception surfaces (wrapped in `ExecutionException`) only when you call `future.get()`. If you never call `get()`, the exception is **silently swallowed** — a frequent source of "why did my task silently fail?" bugs.
  Best practice: handle exceptions inside the task, or always check `Future` results (e.g., in `afterExecute` or via `whenComplete` with `CompletableFuture`).

---

## 10. CompletableFuture

### 91. Why was CompletableFuture introduced?

To fix `Future`'s limitations and enable **non-blocking, composable async** programming. `Future.get()` blocks; you can't chain steps, register completion callbacks, combine multiple futures, or handle errors fluently. `CompletableFuture` (Java 8, implements `CompletionStage`) provides declarative pipelines: transform results (`thenApply`), chain dependent async calls (`thenCompose`), combine independent results (`thenCombine`), react on completion (`whenComplete`), handle errors (`exceptionally`/`handle`), and fan-in/out (`allOf`/`anyOf`) — all without blocking threads. It's Java's answer to promises/futures in reactive programming.

### 92. thenApply() vs thenCompose()

Both chain a function onto a result, but differ in what the function returns:
- **`thenApply(Function<T,U>)`** — the function returns a **plain value** `U`; result is `CompletableFuture<U>`. Use for a synchronous transformation. (Analogous to `map`.)
- **`thenCompose(Function<T,CompletableFuture<U>>)`** — the function returns **another `CompletableFuture<U>`**; result is a flattened `CompletableFuture<U>`, not `CompletableFuture<CompletableFuture<U>>`. Use to chain a **dependent async call**. (Analogous to `flatMap`.)
```java
cf.thenApply(x -> x + 1);                 // T -> U
cf.thenCompose(x -> callServiceAsync(x)); // T -> CompletableFuture<U>, flattened
```
Rule: if your mapping function is itself async (returns a future), use `thenCompose` to avoid nesting.

### 93. thenApply() vs thenAccept()

- **`thenApply(Function)`** — transforms the result and **returns a new value** → `CompletableFuture<U>`. Use when you want to keep processing the transformed result.
- **`thenAccept(Consumer)`** — **consumes** the result and returns **nothing** (`CompletableFuture<Void>`). Terminal-style side effect (e.g., logging, sending a response). No further value to chain.
### 94. thenRun() vs thenAccept()

Both return `CompletableFuture<Void>` and are used for side effects after completion, differing in whether they receive the result:
- **`thenAccept(Consumer<T>)`** — receives the previous stage's **result** and consumes it.
- **`thenRun(Runnable)`** — receives **nothing**; just runs an action after completion (doesn't care about the result). Use for cleanup/notification that doesn't need the value.
### 95. supplyAsync() vs runAsync()

Both start an async computation (optionally on a supplied `Executor`):
- **`supplyAsync(Supplier<T>)`** — runs a task that **returns a value** → `CompletableFuture<T>`.
- **`runAsync(Runnable)`** — runs a task that **returns nothing** → `CompletableFuture<Void>`.
  Use `supplyAsync` when you need a result, `runAsync` for fire-and-forget side effects. Without an explicit executor, both default to the **common `ForkJoinPool`** (see Q100 caveat).

### 96. Exception handling in CompletableFuture

- **`exceptionally(Function<Throwable,T>)`** — invoked only on failure; provides a fallback value. Doesn't see the normal result.
- **`handle(BiFunction<T,Throwable,U>)`** — invoked on **both** success and failure; you get `(result, exception)` (one is null) and return a value. Good for recovering + transforming.
- **`whenComplete(BiConsumer<T,Throwable>)`** — invoked on both outcomes as a **side effect**; does **not** alter the result/exception (it passes through). Good for logging/cleanup.
  Exceptions propagate down the chain until handled. Note the exception is wrapped in `CompletionException`. Example:
```java
cf.thenApply(this::parse)
  .exceptionally(ex -> defaultValue)
  .thenAccept(this::send);
```

### 97. allOf() vs anyOf()

- **`allOf(cf1, cf2, ...)`** — returns `CompletableFuture<Void>` that completes when **all** given futures complete (fan-in / wait-for-all). To collect results, join each after `allOf` completes. If any fails, `allOf` completes exceptionally.
- **`anyOf(cf1, cf2, ...)`** — returns `CompletableFuture<Object>` that completes when the **first** of them completes (with that one's result). Useful for "fastest wins"/timeout races/redundant calls.
```java
CompletableFuture.allOf(a, b, c).join();          // wait for all
Object first = CompletableFuture.anyOf(a, b).join(); // first to finish
```

### 98. thenCombine() vs thenCompose()

- **`thenCombine(other, BiFunction)`** — combines **two independent** futures that run **in parallel**; when both complete, applies a function to both results. Use when the two async calls don't depend on each other.
- **`thenCompose(Function)`** — chains a **dependent** future: the second call needs the first's result, so they run **sequentially**.
```java
priceCF.thenCombine(taxCF, (price, tax) -> price + tax);   // parallel, independent
userCF.thenCompose(user -> loadOrdersAsync(user.id));      // sequential, dependent
```

### 99. How do you combine multiple async calls?

- **Two independent, need both:** `thenCombine`.
- **Many independent, need all:** `allOf(...)` then join each (often collecting via a stream).
```java
  List<CompletableFuture<T>> futures = ids.stream()
      .map(id -> CompletableFuture.supplyAsync(() -> fetch(id), pool))
      .collect(toList());
  CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
      .thenApply(v -> futures.stream().map(CompletableFuture::join).collect(toList()));
```
- **Dependent (chain):** `thenCompose`.
- **First to finish:** `anyOf`.
  Always supply a dedicated `Executor` for I/O-bound work rather than relying on the common pool.

### 100. When is CompletableFuture not suitable?

- **CPU-bound heavy parallel compute:** the common `ForkJoinPool` is sized to cores and shared JVM-wide; blocking or long tasks starve it. Use a dedicated pool or parallel streams / ForkJoin carefully.
- **Blocking I/O on the default pool:** blocking the common pool starves other users (parallel streams, other CFs). Always pass a dedicated executor.
- **Complex reactive streams / backpressure:** for streaming, backpressure, and rich operators, use Reactor/RxJava (`Flux`/`Mono`) instead.
- **Simple synchronous code:** don't add async complexity where a plain call suffices.
- **Structured cancellation/timeouts:** cancellation semantics are limited (cancel doesn't interrupt the running task). For structured concurrency, Java 21's `StructuredTaskScope` or reactive libs are better.
---

## 11. Concurrent Collections

### 101. ConcurrentHashMap internals (Java 8+)

Java 8 redesigned CHM away from Java 7's segment locking:
- Backed by a `Node<K,V>[] table` (like `HashMap`). Concurrency is at the **bucket (bin) level**, not a fixed number of segments.
- **Reads are lock-free** — nodes' `val` and `next` are `volatile`, so `get()` never locks.
- **Writes** lock only the **head node of the specific bin** using `synchronized` (CAS is used to install the first node in an empty bin). So concurrency scales with the number of buckets, not a fixed segment count.
- **Treeification:** a bin with > 8 nodes (and table ≥ 64) converts from a linked list to a **red-black tree** (`TreeNode`), improving worst-case lookup from O(n) to O(log n).
- **Size** is tracked with a `LongAdder`-like striped counter (`baseCount` + `CounterCell[]`) to avoid contention.
- **Resizing** is concurrent — multiple threads help transfer bins (`ForwardingNode` marks migrated bins).
- Does **not** allow `null` keys or values (to disambiguate "absent" from "mapped to null" in concurrent reads).
### 102. Hashtable vs ConcurrentHashMap

| | `Hashtable` | `ConcurrentHashMap` |
|---|---|---|
| Locking | Single lock on **every** method (whole map) | Fine-grained per-bin; lock-free reads |
| Throughput | Poor (all ops serialize) | High (concurrent reads + segmented writes) |
| Nulls | No null key/value | No null key/value |
| Iterator | Fail-fast (`ConcurrentModificationException`) | **Weakly consistent** (no CME; reflects some concurrent changes) |
| Era | Legacy (Java 1.0) | Modern (`java.util.concurrent`) |

`Hashtable` locks the entire table for every operation; CHM allows concurrent access. Always prefer CHM (or `Collections.synchronizedMap`, which is also whole-map locked) over `Hashtable`.

### 103. Why is HashMap not thread-safe?

Concurrent modification corrupts its internal state:
- **Lost updates / overwritten entries** when two threads `put` into the same bucket simultaneously.
- **Resize race (Java 7):** concurrent `put` during rehash could create a **circular linked list** in a bucket, causing an infinite loop (100% CPU) on a later `get`. (Java 8 changed rehash to preserve order and avoid the cycle, but it's still not thread-safe — data can still be lost/corrupted.)
- **Visibility:** without synchronization, one thread may not see another's writes.
- Iterators are **fail-fast** and throw `ConcurrentModificationException`.
  For concurrency use `ConcurrentHashMap`.

### 104. CopyOnWriteArrayList internals

A thread-safe `List` where **every mutation** (`add`, `set`, `remove`) creates a **fresh copy** of the entire backing array under a lock, then atomically swaps the `volatile` array reference:
- **Reads are lock-free** and never block — they read the current immutable array snapshot.
- **Iterators** operate over the snapshot taken at creation time → **never throw `ConcurrentModificationException`**, but don't reflect later mutations and don't support `remove`/`set` on the iterator.
- Writes are **expensive** (O(n) array copy) and memory-heavy.
  Ideal for **read-mostly, write-rarely** lists (e.g., event listener lists).

### 105. When should CopyOnWriteArrayList be used?

When reads vastly outnumber writes and you want lock-free, consistent iteration:
- **Listener/observer registries** (subscribers rarely change, events fire often).
- **Configuration or routing tables** read constantly, updated occasionally.
- Cases where you iterate frequently and must avoid `ConcurrentModificationException` without locking.
  Avoid it for write-heavy or large lists — each write copies the whole array (O(n)), which is slow and GC-heavy.

### 106. ConcurrentLinkedQueue vs LinkedBlockingQueue

| | `ConcurrentLinkedQueue` | `LinkedBlockingQueue` |
|---|---|---|
| Blocking | **Non-blocking** (returns null if empty) | **Blocking** (`take()` waits if empty, `put()` waits if full) |
| Algorithm | Lock-free (CAS, Michael-Scott queue) | Lock-based (two locks: put/take) |
| Bounded? | Unbounded | Optionally bounded (capacity) |
| Use with | High-throughput non-blocking producers/consumers | Producer-consumer with backpressure / thread pools |

Use `ConcurrentLinkedQueue` when you never need to block/wait and want max throughput. Use `LinkedBlockingQueue` for classic producer-consumer where consumers should **wait** for items and producers should **wait** when full (backpressure) — it's the default queue for `ThreadPoolExecutor`.

### 107. BlockingQueue implementations

`BlockingQueue` (blocks on empty `take()` / full `put()`) implementations:
- **`ArrayBlockingQueue`** — bounded, array-backed, single lock, optional fairness. Predictable memory.
- **`LinkedBlockingQueue`** — optionally bounded, linked nodes, separate put/take locks (higher throughput).
- **`PriorityBlockingQueue`** — unbounded, orders by priority/comparator (no FIFO).
- **`DelayQueue`** — elements available only after their delay expires.
- **`SynchronousQueue`** — zero capacity; each `put` waits for a matching `take` (direct handoff). Used by `CachedThreadPool`.
- **`LinkedTransferQueue`** — unbounded; `transfer()` waits until an element is consumed.
### 108. PriorityBlockingQueue use cases

An unbounded blocking queue that dequeues elements in **priority order** (natural ordering or a `Comparator`) rather than FIFO. Consumers `take()` the highest-priority element and block when empty. Use cases: **priority task scheduling** (urgent jobs first), **job/work queues** with SLAs, event processing where some events preempt others, **Dijkstra/A*** algorithms, and load shedding. Note: it's unbounded (no backpressure — watch memory), and elements with equal priority have no guaranteed order.

### 109. DelayQueue use cases

`DelayQueue` holds elements implementing `Delayed`; an element can only be **taken after its delay has expired**. `take()` blocks until the head element's delay elapses. Use cases: **scheduled/deferred task execution**, **cache expiration/TTL eviction**, **retry with backoff** (schedule a retry N seconds later), **session/connection timeout** handling, and rate limiting. It's the backbone of `ScheduledThreadPoolExecutor`'s delayed work queue.

### 110. Why does ConcurrentHashMap not lock the whole map?

For **scalability**. Locking the whole map (like `Hashtable`) serializes every operation, so throughput doesn't improve with more cores. CHM locks only the **individual bin** being written (Java 8) while keeping reads lock-free (volatile nodes). This lets many threads read and write **different** buckets concurrently, so throughput scales with core count and bucket count. Whole-map locking would negate the point of a concurrent map. The trade-off is weaker cross-map guarantees: `size()` is an estimate at any instant, and iterators are weakly consistent rather than snapshot-atomic.
 
---

## 12. Synchronizers

### 111. What is CountDownLatch?

A one-shot synchronizer initialized with a count. Threads call `await()` to block until the count reaches zero; other threads call `countDown()` to decrement. Once it hits zero, all waiters proceed and the latch **cannot be reset**. Two classic patterns:
- **Wait for N tasks to finish:** main thread `await()`s while N workers each `countDown()` when done.
- **Start gate:** N workers `await()` on a latch of 1; a controller `countDown()`s to release them all at once.
```java
CountDownLatch latch = new CountDownLatch(3);
// each worker: ... ; latch.countDown();
latch.await();  // main resumes after all 3 finish
```

### 112. CountDownLatch vs CyclicBarrier

| | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Reusable | **No** (one-shot) | **Yes** (resets automatically) |
| Who waits | One/many threads wait for events counted by others | A fixed set of threads wait for **each other** |
| Trigger | Count reaches 0 via `countDown()` | All N parties call `await()` |
| Barrier action | None | Optional `Runnable` runs when barrier trips |
| Counting | Any thread can count down (even the same one N times) | Each party arrives once per cycle |

Use a latch for "wait until these events happen"; use a barrier for "all threads meet at a common point, repeatedly" (e.g., iterative parallel algorithms).

### 113. CyclicBarrier vs Phaser

- **`CyclicBarrier`** — fixed number of parties set at construction; all must `await()` each cycle; reusable but the party count can't change.
- **`Phaser`** — more flexible, **dynamic** synchronizer: parties can **register/deregister** at runtime (`register`/`arriveAndDeregister`), supports **multiple phases** with `arriveAndAwaitAdvance()`, tracks a phase number, and can be tiered for scalability. Think of it as a resizable, multi-phase CyclicBarrier + CountDownLatch hybrid.
  Use `Phaser` when the number of participating threads varies dynamically or you need multi-phase coordination; `CyclicBarrier` when parties are fixed.

### 114. Semaphore use cases

A `Semaphore` maintains a set of **permits**; `acquire()` takes one (blocking if none), `release()` returns one. It limits how many threads can access a resource concurrently. Use cases:
- **Resource pools / connection limiting** (e.g., allow at most N concurrent DB connections).
- **Rate limiting / throttling** concurrent access to an API or expensive resource.
- **Bounded concurrency** — cap parallel tasks even within a larger pool.
- A **binary semaphore** (1 permit) can act as a simple non-reentrant mutex/signal.
```java
Semaphore sem = new Semaphore(5); // max 5 concurrent
sem.acquire(); try { /* limited resource */ } finally { sem.release(); }
```

### 115. Exchanger class

`Exchanger<V>` is a synchronization point where **two threads swap objects**. Each calls `exchange(myObject)`; each blocks until the other arrives, then they swap and both continue with the other's object. Use cases: **pipeline/buffer swapping** (a producer fills a buffer while a consumer empties another, then they swap — double buffering), and genetic-algorithm/simulation data exchange. It's for exactly **two** threads pairing up.

### 116. Phaser use cases

`Phaser` suits scenarios needing **dynamic party registration** and **multi-phase** coordination:
- **Multi-stage parallel computation** where threads must all finish phase 1 before any starts phase 2, repeated across phases.
- **Dynamic worker pools** where threads join/leave between phases (unlike CyclicBarrier's fixed count).
- **Fork/join-style iterative algorithms** (simulations, graph BFS levels) with a variable number of participants.
- Replacing a combination of `CountDownLatch` + `CyclicBarrier` when both dynamic membership and reuse are needed.
### 117. Which synchronizer would you choose for a batch job?

Depends on the coordination pattern:
- **"Wait for all N sub-tasks to complete, then aggregate":** `CountDownLatch` (count = N), or better, an `ExecutorService` with `invokeAll()`/`CompletableFuture.allOf()`.
- **"Process in synchronized phases/rounds, all workers advance together":** `CyclicBarrier` (fixed workers) or `Phaser` (dynamic workers / multiple phases).
- **"Limit concurrency of the batch to N at a time":** `Semaphore`.
  For a typical "fan out work, wait for everything, then combine," `CountDownLatch` or `CompletableFuture.allOf` is the idiomatic choice; use `Phaser`/`CyclicBarrier` when the batch runs in coordinated iterative phases.

### 118. Implement producer-consumer using BlockingQueue

`BlockingQueue` handles all the waiting/signaling internally — no manual `wait/notify`:
```java
BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10); // bounded → backpressure
 
Runnable producer = () -> {
    try {
        for (int i = 0; i < 100; i++) queue.put(i);  // blocks if full
        queue.put(-1); // poison pill to stop consumer
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
};
 
Runnable consumer = () -> {
    try {
        int item;
        while ((item = queue.take()) != -1) {        // blocks if empty
            process(item);
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
};
 
new Thread(producer).start();
new Thread(consumer).start();
```
The bounded queue gives natural backpressure: producers block when full, consumers block when empty.

### 119. Bounded queue vs unbounded queue

- **Bounded queue** (fixed capacity): provides **backpressure** — producers block/are rejected when full, protecting memory and signaling overload. Preferred in production. Examples: `ArrayBlockingQueue`, bounded `LinkedBlockingQueue`.
- **Unbounded queue** (no capacity limit): producers never block on full, but under sustained overload the queue grows without limit → **`OutOfMemoryError`** and hidden latency (tasks pile up). Examples: unbounded `LinkedBlockingQueue` (default in `newFixedThreadPool`), `ConcurrentLinkedQueue`.
  The danger of unbounded queues is why `Executors.newFixedThreadPool` can OOM under load — always prefer bounded queues with an explicit rejection policy.

### 120. How does BlockingQueue internally work?

Take `ArrayBlockingQueue`: a `ReentrantLock` plus two `Condition`s — `notEmpty` and `notFull`:
- **`put()`:** acquires the lock; `while (count == capacity) notFull.await();` then enqueues, increments count, and `notEmpty.signal()`.
- **`take()`:** acquires the lock; `while (count == 0) notEmpty.await();` then dequeues, decrements count, and `notFull.signal()`.
  The `while` loop guards against spurious wakeups (Q64). `LinkedBlockingQueue` uses **two separate locks** (putLock/takeLock) so producers and consumers don't contend on the same lock, improving throughput. `SynchronousQueue` uses a direct handoff (no storage). In all cases the queue encapsulates the condition-waiting logic so callers don't write raw `wait/notify`.

---

## 13. ThreadLocal

### 121. What is ThreadLocal?

`ThreadLocal<T>` provides a variable where **each thread has its own independent copy**. Reads/writes via `get()`/`set()` are scoped to the current thread — one thread can't see another's value. It achieves thread-safety through **confinement** (no sharing) rather than synchronization. Use `ThreadLocal.withInitial(supplier)` for a default. Common for per-thread context (user/session, transaction, `SimpleDateFormat` which isn't thread-safe).

```java
private static final ThreadLocal<SimpleDateFormat> FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
String s = FMT.get().format(new Date()); // each thread gets its own formatter
```

### 122. How does ThreadLocal work internally?

Each `Thread` object has a field `ThreadLocal.ThreadLocalMap threadLocals`. The map's **keys are the `ThreadLocal` instances** (held via **weak references**) and the values are the per-thread stored objects.
- `threadLocal.get()` → looks up the current thread's `ThreadLocalMap` using `this` (the ThreadLocal) as key.
- `set(v)` → stores `v` in the current thread's map.
  So the data lives on the **thread**, not in the `ThreadLocal` object. Because there's no sharing across threads, no synchronization is needed. The weak-reference keys are what enable (but also complicate) garbage collection — see the leak question.

### 123. ThreadLocal vs static variable

- **Static variable:** **one** value shared by **all** threads → needs synchronization; changes are visible across threads.
- **ThreadLocal:** **one value per thread** → isolated, no sharing, no synchronization; changes are invisible to other threads.
  A `static ThreadLocal` is common: the *field* is static (one ThreadLocal object), but each thread still gets its own value through it. Use a static variable for genuinely shared state; use ThreadLocal for per-thread state you don't want to pass through every method call.

### 124. ThreadLocal memory leak

The classic leak occurs in **thread pools**:
- The `ThreadLocalMap` **key** is a **weak** reference to the ThreadLocal, but the **value** is a **strong** reference.
- If the ThreadLocal object is GC'd, the key becomes null but the **value remains** (stale entry) as long as the thread is alive.
- Pooled threads live for the application's lifetime, so their `ThreadLocalMap` retains these values indefinitely → memory leak (and the value's whole object graph, e.g., a big user object or classloader → potential `ClassLoader` leak / `Metaspace` leak in app servers).
  Fix: always call `remove()` when done (Q125). `ThreadLocalMap` does opportunistically clean some stale entries on access, but you can't rely on it.

### 125. Why should ThreadLocal.remove() be called?

Because in **long-lived / pooled threads**, values set on a ThreadLocal persist after the task finishes and the thread is **reused** for the next task — causing:
- **Memory leaks** (stale values never GC'd; Q124).
- **Data bleed / correctness bugs** — the next task on the same pooled thread sees a *previous* request's leftover value (e.g., wrong user/tenant context) → security and correctness issues.
  So the safe pattern is:
```java
try {
    context.set(value);
    // ... handle request ...
} finally {
    context.remove();   // clean up before the thread returns to the pool
}
```
Frameworks (Spring, servlet filters) do this cleanup for their own ThreadLocals.

### 126. Real-world use cases of ThreadLocal

- **Per-request/user context:** security principal, tenant id, request id / correlation id (MDC in SLF4J logging uses ThreadLocal).
- **Non-thread-safe objects reused per thread:** `SimpleDateFormat`, `Random` (though `ThreadLocalRandom` is better), Jackson `ObjectMapper` scratch, `SAXParser`.
- **Transaction/connection context:** Spring's transaction manager binds the current `Connection`/`EntityManager` to the thread.
- **Avoiding parameter passing:** propagating context implicitly through a call stack without threading it through every method signature.
### 127. How does Spring Security use ThreadLocal?

Spring Security stores the current `Authentication` in a `SecurityContext` held by `SecurityContextHolder`, whose default strategy is `ThreadLocalSecurityContextHolderStrategy` — a `ThreadLocal<SecurityContext>`. So `SecurityContextHolder.getContext().getAuthentication()` returns the **current thread's** authenticated user anywhere in the request without passing it around. A filter (`SecurityContextPersistenceFilter`/`SecurityContextHolderFilter`) sets it at the start of the request and **clears it in a finally** at the end so the pooled thread doesn't leak the principal to the next request. For async work, `MODE_INHERITABLETHREADLOCAL` or `DelegatingSecurityContextExecutor` propagates it to child threads.

### 128. How do transaction managers use ThreadLocal?

Spring's `TransactionSynchronizationManager` keeps a set of `ThreadLocal` maps binding **resources to the current thread** — e.g., the JDBC `Connection` or JPA `EntityManager` associated with the active transaction. When a `@Transactional` method starts, the transaction manager opens a connection and **binds it to the thread**. Any DAO/`JdbcTemplate` call on that thread then retrieves the **same** connection from the ThreadLocal, so all operations share one transaction. On commit/rollback, the resource is **unbound and cleaned up**. This is why a transaction is thread-bound and doesn't automatically span across threads (async calls start a new, separate transaction context).

### 129. What happens in thread pools if ThreadLocal isn't cleaned?

Because pool threads are **reused** across tasks:
- **Stale data leaks between tasks** — task B running on the same thread sees task A's leftover ThreadLocal value (e.g., wrong user context, security leak, incorrect tenant).
- **Memory leaks** — values (and their entire object graphs) are retained for the life of the pool thread (which is the app's lifetime).
- In app servers, this can pin **classloaders** → `Metaspace`/`PermGen` leaks and redeploy failures.
  Hence always `remove()` in a `finally`, or use framework mechanisms that manage cleanup.

### 130. InheritableThreadLocal vs ThreadLocal

- **`ThreadLocal`** — a child thread does **not** inherit the parent's value; it starts empty.
- **`InheritableThreadLocal`** — when a thread **creates a child thread**, the child **inherits a copy** of the parent's value (via `childValue()`, shallow copy by default). Useful for propagating context (like a trace id) to spawned threads.
  Caveat: it only works at **thread creation** time — it does **not** work with **thread pools**, because pooled threads are created once and reused, so they won't pick up the submitting thread's current value. For pools/async, use explicit context propagation (e.g., `TaskDecorator` in Spring, or libraries like `TransmittableThreadLocal`).

---

## 14. Parallelism

### 131. Parallel Stream vs Stream

- **Sequential `stream()`** — processes elements one at a time on the calling thread.
- **`parallelStream()`** — splits the source into chunks (via a `Spliterator`), processes them concurrently across multiple threads (the common `ForkJoinPool`), and combines results.
  Parallel streams help only for **CPU-bound**, large, easily-splittable datasets with **stateless, non-interfering, associative** operations. They add overhead (splitting, thread coordination, merging), so for small collections or I/O-bound work they're often **slower**. Never mutate shared state in a parallel stream (use proper reduction/collectors).

### 132. ForkJoinPool internals

`ForkJoinPool` is a specialized `ExecutorService` for **divide-and-conquer** parallelism:
- Each worker thread has its **own double-ended queue (deque)** of tasks.
- A worker pushes/pops subtasks from the **head** of its own deque (LIFO — good cache locality).
- Idle workers **steal** tasks from the **tail** of other workers' deques (**work stealing**), balancing load automatically.
- `fork()` submits a subtask to the current worker's deque; `join()` waits for and retrieves its result (and may help execute other tasks meanwhile).
- The **common pool** (`ForkJoinPool.commonPool()`) is shared JVM-wide, sized to `Runtime.availableProcessors() - 1` by default, and backs parallel streams and `CompletableFuture` async methods without an explicit executor.
### 133. What is work stealing?

Work stealing is the load-balancing strategy where **idle worker threads "steal" tasks from busy workers' queues**. Each worker primarily processes its own deque (from the head); when it runs out, it takes a task from the **tail** of another worker's deque. Benefits: keeps all cores busy, self-balances uneven workloads, minimizes contention (workers mostly touch their own deque; stealing happens at the opposite end). It's the core mechanism behind `ForkJoinPool` and parallel streams.

### 134. RecursiveTask vs RecursiveAction

Both are `ForkJoinTask`s for fork/join:
- **`RecursiveTask<V>`** — `compute()` **returns a result** of type `V`. Use when subtasks produce values to combine (e.g., parallel sum).
- **`RecursiveAction`** — `compute()` returns **void**. Use for side-effecting work with no return (e.g., parallel array sort/transform in place).
```java
class SumTask extends RecursiveTask<Long> {
    protected Long compute() {
        if (small) return computeDirectly();
        SumTask left = new SumTask(...), right = new SumTask(...);
        left.fork();                       // async
        long r = right.compute();          // this thread
        return left.join() + r;            // combine
    }
}
```

### 135. Why are parallel streams sometimes slower?

- **Small data / cheap operations:** splitting + thread coordination + merging overhead exceeds the gain.
- **Poorly splittable sources:** `LinkedList`, `Iterator`-based streams, or `IO` sources don't split evenly (arrays and `ArrayList`/ranges split best).
- **I/O-bound or blocking tasks:** they block common-pool threads without using CPU; more threads don't help and can starve other users of the common pool.
- **Boxing overhead, stateful/ordered ops** (`sorted`, `limit`, `findFirst` with encounter order) reduce parallel efficiency.
- **Merge cost:** combining partial results (e.g., into a `Map`) can be expensive.
- **Shared common pool contention** with other parallel work.
  Rule of thumb: parallelize only large, CPU-bound, stateless, associative pipelines over splittable sources — and **measure**.

### 136. How do parallel streams choose the thread pool?

By default, parallel streams run on the **common `ForkJoinPool`** (`ForkJoinPool.commonPool()`), shared across the whole JVM, sized to `availableProcessors() - 1`. You **cannot** pass an executor directly. The known workaround is to submit the terminal operation from **within your own `ForkJoinPool`**:
```java
ForkJoinPool custom = new ForkJoinPool(8);
long result = custom.submit(() ->
    list.parallelStream().mapToLong(this::heavy).sum()
).get();
```
A parallel stream started inside a ForkJoinPool worker uses **that** pool. This isolates it from the common pool (important so one workload can't starve another).

### 137. Can parallel streams cause deadlocks?

Yes. Because parallel streams share the **common ForkJoinPool**, blocking operations inside a parallel stream (e.g., waiting on a lock, calling `.get()` on another future that also needs the common pool, nested parallel streams, or blocking I/O) can **exhaust the pool** and deadlock — all worker threads block waiting for work that can't be scheduled because no threads are free. Nested parallelism and blocking calls in stream operations are the usual culprits. Mitigations: don't block inside parallel streams, use a dedicated pool, or use `ManagedBlocker` to let the pool compensate.

### 138. CPU-bound vs IO-bound tasks

- **CPU-bound:** limited by processor speed (computation, encoding, math). Optimal thread count ≈ **number of cores** (more threads just add context-switching overhead). Parallel streams/ForkJoin shine here.
- **IO-bound:** limited by waiting on external resources (DB, network, disk). Threads spend most time **blocked**, so you want **many more threads than cores** to keep the CPU busy while others wait (or better, use async/non-blocking I/O or virtual threads).
  Recognizing which kind of workload you have drives thread-pool sizing (Q139–140).

### 139. How many threads should a thread pool have?

Depends on workload type:
- **CPU-bound:** `threads ≈ number of cores` (or cores + 1 to cover occasional stalls).
- **IO-bound:** more than cores, scaled by how much time tasks spend waiting.
  Brian Goetz's formula:
```
N_threads = N_cores × U_cpu × (1 + W/C)
```
where `U_cpu` = target CPU utilization (0–1), `W/C` = ratio of wait time to compute time. For purely CPU-bound (W/C = 0): threads = cores × U. For heavily IO-bound (W ≫ C), the multiplier grows large. Always **measure** with realistic load; also bound the queue and set a rejection policy.

### 140. How do you size thread pools?

Steps:
1. **Classify the workload** (CPU vs IO-bound; mixed → consider separate pools).
2. **Apply the formula** `N = cores × targetUtil × (1 + wait/compute)` as a starting point.
3. **Bound the queue** to provide backpressure and avoid OOM; choose a **rejection policy** (`CallerRunsPolicy` for backpressure, `AbortPolicy` to fail fast, etc.).
4. **Separate pools** for different concerns (don't let slow I/O tasks starve fast ones; isolate the common pool).
5. **Load-test and monitor** — track queue length, task latency, active threads, rejections; tune iteratively.
6. Consider **virtual threads** (Java 21) for high-concurrency blocking I/O instead of huge platform-thread pools.
---

## 15. Spring Boot Concurrency

### 141. Is a Spring Bean thread-safe?

Not inherently — thread safety depends on the bean's **state**, not on Spring. Spring's default scope is **singleton**: one shared instance injected everywhere, accessed by many request threads concurrently. If that singleton holds **mutable instance state**, it's **not** thread-safe. If it's **stateless** (only local variables and immutable/injected collaborators), it is effectively thread-safe. Spring doesn't add synchronization for you.

### 142. Why are singleton beans safe most of the time?

Because well-designed Spring beans (controllers, services, repositories) are typically **stateless** — they hold only references to other (also stateless) beans and use **method-local variables**, which live on each thread's own stack and aren't shared. Since there's no shared mutable instance state, concurrent calls don't interfere. The moment you add a mutable field (a counter, a cached value, a non-thread-safe helper stored as a field), that safety disappears.

### 143. What happens if singleton beans have mutable state?

You get **race conditions**: because all request threads share the single instance, concurrent reads/writes to a mutable field can corrupt data, produce lost updates, or leak one request's data into another. Example: storing per-request data in an instance field, or a shared `SimpleDateFormat` field (not thread-safe). Fixes: make the bean stateless (pass state as parameters/return values), make fields immutable, use thread-safe types (`AtomicInteger`, `ConcurrentHashMap`), use `ThreadLocal`, use proper synchronization, or use a narrower scope (`prototype`, `request`).

### 144. How does @Async work internally?

`@Async` makes a method execute on a **separate thread** and return immediately (`void`, `Future`, or `CompletableFuture`). Mechanics:
- Enabled by `@EnableAsync`. Spring creates a **proxy** (CGLIB/JDK dynamic) around the bean.
- When you call the `@Async` method, the proxy **submits the invocation to a `TaskExecutor`** instead of running it inline, returning control to the caller immediately.
- Caveats (proxy-based, like `@Transactional`): **self-invocation doesn't work** (calling an `@Async` method from within the same class bypasses the proxy); the method should be **public**; return types are limited to `void`/`Future`/`CompletableFuture`; and exceptions from `void` async methods go to an `AsyncUncaughtExceptionHandler`.
### 145. How do you configure Spring async executors?

Define a `TaskExecutor` bean so `@Async` doesn't use the default `SimpleAsyncTaskExecutor` (which creates a new thread per task — no pooling, dangerous):
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(8);
        ex.setMaxPoolSize(16);
        ex.setQueueCapacity(100);          // bounded queue
        ex.setThreadNamePrefix("async-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        ex.setWaitForTasksToCompleteOnShutdown(true);
        ex.initialize();
        return ex;
    }
}
```
Reference a specific executor via `@Async("taskExecutor")`. Use a `TaskDecorator` to propagate context (MDC, security) into async threads. (Boot also supports properties like `spring.task.execution.pool.*`.)

### 146. @Scheduled thread safety

By default, `@Scheduled` tasks run on a **single-threaded** scheduler (`ThreadPoolTaskScheduler` with pool size 1) — so different scheduled methods run **serially**, and a long-running one **delays** others. Thread-safety considerations:
- If tasks share mutable state, protect it (they may overlap if you increase pool size or use `@Async` on scheduled methods).
- For **fixed-rate** tasks, a slow run can cause overlap; ensure the task is idempotent/guarded or use `fixedDelay`.
- In a **clustered/multi-instance** deployment, the same `@Scheduled` job runs on **every** instance → use a distributed lock (ShedLock, Quartz clustering) to ensure single execution.
- Increase concurrency via a custom `TaskScheduler` with a larger pool if independent tasks shouldn't block each other.
### 147. Thread safety in a RestController

Controllers are **singletons** and handle many concurrent requests, so the same rules apply: keep them **stateless**. Don't store per-request data in instance fields — use method parameters, local variables, and return values. Injected services should also be thread-safe (stateless or properly synchronized). Request-scoped data belongs in method scope, `ThreadLocal`, or `@RequestScope` beans. A common bug is caching request-specific values in controller fields, which leak across concurrent requests.

### 148. Can @Transactional methods execute concurrently?

Yes — multiple threads can execute a `@Transactional` method **simultaneously**, each in its **own transaction** bound to its own thread (the transaction context is thread-local; each gets its own DB connection). `@Transactional` provides **database** transaction semantics (atomicity, isolation), **not** mutual exclusion between threads. So two threads updating the same row concurrently still race at the DB level — you need DB isolation levels, optimistic/pessimistic locking, or `SELECT ... FOR UPDATE` to coordinate. `@Transactional` alone does not serialize threads.

### 149. Transaction isolation and concurrency

Isolation levels control what one transaction sees of others' uncommitted/committed changes, trading consistency for concurrency:
- **READ_UNCOMMITTED** — can see uncommitted changes (dirty reads). Fastest, least safe.
- **READ_COMMITTED** — sees only committed data (no dirty reads); allows non-repeatable reads. (Default in Postgres/Oracle/SQL Server.)
- **REPEATABLE_READ** — same rows read twice return the same values (no non-repeatable reads); may allow phantoms. (Default in MySQL/InnoDB.)
- **SERIALIZABLE** — full isolation, as if transactions ran one at a time; prevents phantoms; lowest concurrency (more locking/aborts).
  Anomalies to know: **dirty read, non-repeatable read, phantom read**. Higher isolation = more consistency but more contention/blocking/deadlocks. Set via `@Transactional(isolation = Isolation.REPEATABLE_READ)`.

### 150. Thread safety of EntityManager

**`EntityManager` (JPA) is NOT thread-safe** and must not be shared across threads. It represents a **persistence context** (a unit of work / first-level cache) tied to a single transaction/thread. Spring handles this correctly: the injected `EntityManager` is a **thread-safe proxy** that delegates to the **actual `EntityManager` bound to the current thread/transaction** (via `TransactionSynchronizationManager` ThreadLocal). So each thread effectively gets its own real `EntityManager`. If you manually create/share one across threads, you get corruption and errors. (`EntityManagerFactory`, by contrast, **is** thread-safe and is meant to be shared.)
 
---

## 16. Production Scenarios

### 151. How do you avoid double charging in payment systems?

Layered defenses:
- **Idempotency keys:** the client sends a unique key per payment attempt; the server records it and returns the original result on retries instead of charging again (Q153). This is the primary defense against retries/network timeouts.
- **Database uniqueness constraint** on (order_id / idempotency_key) so a duplicate insert fails atomically.
- **Optimistic/pessimistic locking** on the order/wallet row to serialize concurrent charge attempts.
- **State machine:** move the order from `PENDING → CHARGED` in a single atomic transaction; reject charging an already-`CHARGED` order.
- **Exactly-once via the provider:** pass the idempotency key to the payment gateway (Stripe et al. support idempotency keys).
- **Distributed lock** (Redis/DB) on the order id if the flow spans services.
  The combination of an idempotency key + unique constraint + state check is the robust pattern.

### 152. How do you make an API thread-safe?

- **Statelessness:** keep handlers/services stateless; store per-request data in local scope, not shared fields.
- **Thread-safe shared state:** use `Atomic*`, `ConcurrentHashMap`, immutable objects, or proper locking for any shared mutable data.
- **Idempotency** for retriable/mutating operations (Q153).
- **Database-level concurrency control:** optimistic (`@Version`) or pessimistic locking; appropriate isolation.
- **Guard critical sections** minimally; prefer atomics/CAS over coarse locks.
- **Distributed coordination** (Redis/DB locks) when running multiple instances.
- **Rate limiting** to bound concurrency.
  Design so correctness doesn't depend on request timing.

### 153. How do you implement idempotency?

An operation is idempotent if performing it multiple times has the same effect as once. Implementation:
1. Client generates a unique **idempotency key** (UUID) and sends it (e.g., `Idempotency-Key` header) with the request.
2. Server, in a transaction, checks a store (DB table/Redis) for that key:
    - **Seen before** → return the **stored previous response** (don't re-execute).
    - **New** → execute the operation, **persist the key + result atomically** (unique constraint prevents races), then return.
3. Use a **DB unique constraint** on the key so two concurrent requests with the same key can't both proceed — one insert wins, the other detects the conflict and returns the existing result.
4. Set a **TTL** on keys as appropriate.
   This makes retries (from timeouts, at-least-once messaging) safe.

### 154. Optimistic locking vs pessimistic locking

- **Optimistic locking:** assume conflicts are rare; don't lock on read. Use a **version column** (`@Version` in JPA). On update, check the version matches; if another transaction changed it, the version mismatches and you get an `OptimisticLockException` → retry. Great for **low-contention, read-heavy** workloads; no DB locks held, better throughput.
- **Pessimistic locking:** assume conflicts are likely; **lock the row on read** (`SELECT ... FOR UPDATE`, `@Lock(PESSIMISTIC_WRITE)`) so others block until you commit. Prevents conflicts up front but reduces concurrency and risks deadlocks/lock waits. Good for **high-contention** or when retrying is expensive.
  Choose optimistic for scalability with rare conflicts; pessimistic when conflicts are frequent or must be prevented.

### 155. How does database locking affect concurrency?

Locks serialize access to protect consistency but reduce concurrency:
- **Row locks** block only conflicting transactions on the same rows — fine-grained, good concurrency.
- **Table locks / lock escalation** block far more, hurting throughput.
- **Shared (read) vs exclusive (write) locks:** readers can share; writers exclude.
- **Longer transactions hold locks longer** → more blocking, more lock waits, higher deadlock risk.
- **Higher isolation levels** take more/longer locks (Q149).
  Best practices: keep transactions short, access rows in a consistent order (avoid deadlocks), use the lowest sufficient isolation, prefer optimistic locking under low contention, and add proper indexes (an unindexed `WHERE` can lock more rows/gaps than expected).

### 156. What is optimistic concurrency control?

Optimistic Concurrency Control (OCC) is a strategy that lets transactions proceed **without locking**, then **validates at commit** that no conflicting change occurred; if it did, the transaction **aborts and retries**. In practice it's implemented with a **version number or timestamp**: read the version, do work, and on write assert the version is unchanged (`UPDATE ... WHERE id=? AND version=?`); zero rows updated means a conflict. It maximizes concurrency when conflicts are rare (read-heavy systems, REST APIs, JPA `@Version`), at the cost of retry logic when conflicts do happen.

### 157. How would you design a rate limiter?

Pick an algorithm and a scope:
- **Token bucket** (most common): tokens refill at a fixed rate up to a capacity; each request consumes a token; allows bursts up to capacity.
- **Leaky bucket:** requests drain at a constant rate; smooths bursts.
- **Fixed window counter:** count per time window; simple but allows 2× bursts at window edges.
- **Sliding window log/counter:** smoother, more accurate, more memory.
  **Single instance:** an in-memory counter/`Semaphore`/Guava `RateLimiter`. **Distributed (multi-instance):** centralize state in **Redis** (atomic `INCR`+`EXPIRE` or a token-bucket Lua script) so the limit is global. Key by user/IP/API-key. Return `429 Too Many Requests` with `Retry-After`. Consider per-user and global limits, and fail-open vs fail-closed on limiter outage.

### 158. How do you implement distributed locking?

A lock that coordinates across multiple JVMs/instances, since in-process locks don't span machines:
- **Redis (`SET key value NX PX ttl`):** set the lock only if absent, with a **TTL** (auto-expiry to avoid permanent locks if a holder dies) and a **unique token** so only the owner can release it (release via a Lua compare-and-delete). The **Redlock** algorithm extends this across multiple Redis nodes.
- **Database:** a row/advisory lock (`SELECT ... FOR UPDATE`, or an insert into a `locks` table with a unique key).
- **ZooKeeper/etcd:** ephemeral sequential znodes; the lowest sequence holds the lock; ephemeral nodes disappear on client death (robust).
  Key concerns: **TTL/lease** to survive crashes, **fencing tokens** to prevent a stalled old holder from acting after its lease expired, ownership checks on release, and reentrancy if needed. Libraries: Redisson, ShedLock, Curator.

### 159. Redis lock vs database lock

- **Redis lock:** very fast, low latency, TTL-based auto-expiry, easy to scale. Downsides: relies on Redis availability; single-node Redis lock can be lost on failover (Redlock mitigates but is debated); needs careful TTL/fencing to be safe. Best for high-throughput, short-lived locks where occasional rare lock loss is tolerable.
- **Database lock:** strongly consistent (uses the DB you already trust), transactional, no extra infra. Downsides: slower, adds DB load/contention, can cause lock waits/deadlocks, and holding app-level locks in the DB doesn't scale as well. Best when you need strong consistency and already have the DB, and locks are infrequent.
  Trade-off: Redis = speed/scale with weaker guarantees; DB = strong consistency with more overhead. For strict correctness, ZooKeeper/etcd is often preferred.

### 160. ZooKeeper lock vs Redis lock

- **ZooKeeper (etcd) lock:** built on a **consensus** (ZAB/Raft) quorum, so it's **strongly consistent** and reliable under failover. **Ephemeral nodes** auto-release the lock when a client session dies (no reliance on TTL guesswork), and **sequential** nodes give fair FIFO ordering and easy "wait for the lock ahead of me." Downsides: heavier to operate, higher latency, more infrastructure.
- **Redis lock:** faster and simpler, but a single-node lock can be lost during failover, and correctness leans on TTLs and fencing tokens (Redlock is debated for strict correctness).
  Rule of thumb: choose **ZooKeeper/etcd** when correctness under failure is paramount (leader election, strict mutual exclusion); choose **Redis** when you want speed/simplicity and can tolerate rare lock loss with idempotent operations as a safety net.

---

## 17. Kafka & Concurrency

### 161. Is KafkaProducer thread-safe?

**Yes.** `KafkaProducer` is thread-safe and **designed to be shared** across multiple threads. Sharing a **single producer instance** among threads is actually the **recommended** pattern — it's generally faster than one producer per thread because it batches records across threads and maintains a shared buffer/connection pool. Internally it has a thread-safe record accumulator and a background **I/O (sender) thread** that batches and sends. So create one producer and reuse it.

### 162. Is KafkaConsumer thread-safe?

**No.** `KafkaConsumer` is **not** thread-safe (except for `wakeup()`, which is the one method safe to call from another thread to interrupt `poll()`). You must **not** share a consumer across threads. The standard patterns are:
- **One consumer per thread** (each with its own consumer instance), typically one thread per partition subset.
- **Single consumer thread + a worker pool:** one thread does `poll()` and hands records to a thread pool for processing (but then you must manage offset commits carefully to preserve at-least-once/ordering).
  The consumer enforces single-threaded access and throws `ConcurrentModificationException` if accessed by multiple threads.

### 163. How many consumers should be in a consumer group?

At most the **number of partitions** of the topic — because each partition is assigned to **exactly one** consumer within a group at a time. So:
- Consumers **≤ partitions** for full utilization.
- If consumers **> partitions**, the extra consumers are **idle** (no partition to read).
- To scale consumption, increase partitions (and consumers together).
  Rule: **partition count is the unit and upper bound of parallelism** within a consumer group.

### 164. What happens when multiple threads consume the same partition?

Within a single consumer group, a partition is assigned to only **one** consumer, so this doesn't happen at the group level. If you deliberately have **multiple threads process records from the same partition** (e.g., handing polled records to a thread pool), you **lose Kafka's per-partition ordering guarantee**, and offset management becomes error-prone (you might commit an offset before an earlier record finished → data loss on failure). If you use **separate consumer groups**, each group independently reads the whole partition (that's fine — pub/sub). The key point: ordering is guaranteed only when a single thread processes a partition sequentially.

### 165. Kafka ordering guarantees

Kafka guarantees ordering **only within a partition**, not across partitions of a topic. Records in one partition are strictly ordered by offset and delivered in order to a consumer. To preserve order for related events, **route them to the same partition** using a consistent **key** (Kafka hashes the key to pick the partition) — e.g., key by `userId` or `orderId`. Consequences: cross-partition order is undefined; using multiple partitions (for throughput) means only per-key ordering. Also, retries with `max.in.flight.requests > 1` can reorder unless the **idempotent producer** is enabled (which preserves order even with retries).

### 166. Exactly-once processing

Kafka supports **exactly-once semantics (EOS)** so each record affects the output **once**, even across failures/retries:
- **Idempotent producer** (`enable.idempotence=true`): dedups producer retries within a partition (no duplicates from resends).
- **Transactions** (`transactional.id`): atomically write to multiple partitions **and** commit consumer offsets in the same transaction — the "consume-transform-produce" loop becomes atomic. Consumers set `isolation.level=read_committed` to see only committed records.
- In **Kafka Streams**, set `processing.guarantee=exactly_once_v2`.
  EOS applies to Kafka-to-Kafka pipelines. For external sinks (DBs), you still need **idempotent writes** or transactional outbox patterns, since Kafka can't make an arbitrary external system transactional.

### 167. Idempotent producer

Enabled via `enable.idempotence=true` (default in recent Kafka). The producer is assigned a **producer id (PID)** and attaches a **monotonic sequence number** per partition to each record. The broker tracks the last sequence per (PID, partition) and **rejects duplicates** from retries, so a record sent multiple times (due to network retries) is written **once** and **in order**. This gives **exactly-once *delivery to a partition*** and preserves ordering even with `retries > 0` and `max.in.flight.requests.per.connection` up to 5. It requires `acks=all`. It's the foundation for Kafka transactions/EOS.

### 168. Thread safety in Kafka listeners

In Spring Kafka, a `@KafkaListener` method is invoked by the **container's consumer thread**. Considerations:
- The **listener bean** (like any Spring singleton) must be **stateless** or thread-safe, especially if concurrency > 1 (multiple containers/threads invoke it).
- Each concurrent container has its **own** `KafkaConsumer` (which is not shared) — that part is safe.
- If your listener offloads work to another thread pool, you break ordering and must handle **manual offset commits** carefully to avoid data loss.
- Shared resources the listener touches (caches, counters) need thread-safe access.
### 169. Consumer concurrency in Spring Kafka

Spring Kafka's `concurrency` setting (`@KafkaListener(concurrency = "3")` or on the `ConcurrentKafkaListenerContainerFactory`) creates that many **child listener containers**, each with its **own consumer thread and `KafkaConsumer`**. Kafka then distributes the topic's **partitions** among these threads (each partition to one thread). So `concurrency` should be **≤ number of partitions** (extra threads stay idle). This gives parallel, per-partition-ordered consumption within one application instance. Across multiple app instances, the total consumer count across all instances still must be ≤ partitions.

### 170. How does partitioning affect concurrency?

Partitions are Kafka's **unit of parallelism**:
- More partitions → more consumers/threads can process in parallel → higher throughput.
- Each partition is consumed by exactly one consumer per group, so **max consumer parallelism = partition count**.
- Partitioning by **key** preserves per-key ordering while spreading load; a poor key choice causes **hot partitions** (skew) that bottleneck one consumer.
- Trade-offs of many partitions: more open files/memory on brokers, longer leader-election/rebalance times, and more overhead.
  So you size partitions to your target parallelism/throughput, balancing ordering needs and operational cost.

---

## 18. Coding Questions

> These are the canonical concurrency coding problems. Each shows a clean, correct, idiomatic solution with the key idea called out.

### 171. Print odd-even numbers using two threads

Two threads alternate printing; use a shared lock + a turn flag with `wait/notify`.
```java
class OddEvenPrinter {
    private final Object lock = new Object();
    private int number = 1;
    private final int max;
    OddEvenPrinter(int max) { this.max = max; }
 
    void printOdd() {
        synchronized (lock) {
            while (number <= max) {
                while (number % 2 == 0) waitQuietly();       // wait for odd turn
                if (number <= max) System.out.println("Odd : " + number++);
                lock.notifyAll();
            }
        }
    }
    void printEven() {
        synchronized (lock) {
            while (number <= max) {
                while (number % 2 == 1) waitQuietly();       // wait for even turn
                if (number <= max) System.out.println("Even: " + number++);
                lock.notifyAll();
            }
        }
    }
    private void waitQuietly() {
        try { lock.wait(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
    public static void main(String[] a) {
        OddEvenPrinter p = new OddEvenPrinter(10);
        new Thread(p::printOdd).start();
        new Thread(p::printEven).start();
    }
}
```
Key idea: shared state (`number`) + condition check in a `while` loop + `notifyAll` to wake the other thread.

### 172. Implement producer-consumer

Use a `BlockingQueue` — it handles waiting/signaling for you (see Q118 for the full version). Manual `wait/notify` version:
```java
class BoundedBuffer<T> {
    private final Queue<T> q = new LinkedList<>();
    private final int capacity;
    BoundedBuffer(int capacity) { this.capacity = capacity; }
 
    public synchronized void put(T item) throws InterruptedException {
        while (q.size() == capacity) wait();   // full → wait
        q.add(item);
        notifyAll();                            // signal consumers
    }
    public synchronized T take() throws InterruptedException {
        while (q.isEmpty()) wait();             // empty → wait
        T item = q.remove();
        notifyAll();                            // signal producers
        return item;
    }
}
```
Key idea: `while` guards for full/empty conditions, `notifyAll` after each state change. In production, just use `LinkedBlockingQueue`.

### 173. Implement a thread-safe singleton

Best options:

**Bill Pugh (lazy holder idiom)** — lazy, no synchronization overhead, thread-safe via classloader guarantees:
```java
class Singleton {
    private Singleton() {}
    private static class Holder { static final Singleton INSTANCE = new Singleton(); }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

**Enum singleton** — simplest, serialization- and reflection-safe (Joshua Bloch's recommendation):
```java
enum Singleton { INSTANCE; public void doWork() { /* ... */ } }
```

**Double-checked locking** (if you need lazy + a class, and can't use the holder):
```java
class Singleton {
    private static volatile Singleton instance;   // volatile is essential
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}
```
Prefer the holder idiom or enum; use DCL only when required.

### 174. Build a thread-safe LRU cache

Simplest correct approach: wrap `LinkedHashMap` (access-order) with synchronization.
```java
class LRUCache<K, V> {
    private final LinkedHashMap<K, V> map;
    LRUCache(int capacity) {
        map = new LinkedHashMap<>(capacity, 0.75f, true) {  // access-order = true
            protected boolean removeEldestEntry(Map.Entry<K, V> e) {
                return size() > capacity;                    // evict LRU
            }
        };
    }
    public synchronized V get(K k) { return map.get(k); }
    public synchronized void put(K k, V v) { map.put(k, v); }
}
```
Key idea: `LinkedHashMap` with `accessOrder=true` moves accessed entries to the tail; `removeEldestEntry` evicts the head (least recently used). Synchronize because `LinkedHashMap` isn't thread-safe. For higher concurrency, use **Caffeine** or **Guava** cache, which give near-lock-free LRU/LFU eviction.

### 175. Implement a custom blocking queue

```java
class SimpleBlockingQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
 
    SimpleBlockingQueue(int capacity) { this.capacity = capacity; }
 
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) notFull.await();
            queue.add(item);
            notEmpty.signal();
        } finally { lock.unlock(); }
    }
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) notEmpty.await();
            T item = queue.remove();
            notFull.signal();
            return item;
        } finally { lock.unlock(); }
    }
}
```
Key idea: a `ReentrantLock` with **two `Condition`s** (`notFull`, `notEmpty`) so producers and consumers wait on the right condition; always `unlock()` in `finally`; `while` guards for spurious wakeups.

### 176. Print A-B-C repeatedly using three threads

Generalize the turn-based approach with a shared turn counter.
```java
class ABCPrinter {
    private final Object lock = new Object();
    private int turn = 0;                 // 0=A, 1=B, 2=C
    private final int rounds;
    ABCPrinter(int rounds) { this.rounds = rounds; }
 
    void print(char c, int id) {
        synchronized (lock) {
            for (int i = 0; i < rounds; i++) {
                while (turn % 3 != id) {
                    try { lock.wait(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
                }
                System.out.print(c);
                turn++;
                lock.notifyAll();
            }
        }
    }
    public static void main(String[] a) {
        ABCPrinter p = new ABCPrinter(5);
        new Thread(() -> p.print('A', 0)).start();
        new Thread(() -> p.print('B', 1)).start();
        new Thread(() -> p.print('C', 2)).start();
    }
}
```
Key idea: each thread has an `id`; it acts only when `turn % 3 == id`, then increments `turn` and `notifyAll`. (A `Semaphore[]` per thread, or `ReentrantLock` + 3 `Condition`s, are cleaner alternatives.)

### 177. Print 1-2-3 repeatedly using three threads

Identical mechanism to A-B-C — each thread prints its number when it's its turn:
```java
class NumberPrinter {
    private final Object lock = new Object();
    private int turn = 1;                 // threads for 1,2,3
    private final int max;
    NumberPrinter(int max) { this.max = max; }
 
    void print(int id) {  // id in {1,2,3}
        synchronized (lock) {
            while (true) {
                while (turn != id) {
                    try { lock.wait(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
                }
                if (currentNumber > max) { lock.notifyAll(); return; }
                System.out.println("T" + id + ": " + currentNumber++);
                turn = id == 3 ? 1 : id + 1;   // next thread's turn
                lock.notifyAll();
            }
        }
    }
    private int currentNumber = 1;
    public static void main(String[] a) {
        NumberPrinter p = new NumberPrinter(9);
        for (int i = 1; i <= 3; i++) { int id = i; new Thread(() -> p.print(id)).start(); }
    }
}
```
Key idea: a rotating `turn` (1→2→3→1…) ensures ordered, interleaved printing; a shared `currentNumber` is the value being printed. Use `Semaphore`s for a cleaner version.

### 178. Build a thread-safe counter

From cheapest/most scalable to most general:
```java
// 1) Atomic — lock-free, best for a single counter
AtomicLong atomic = new AtomicLong();
atomic.incrementAndGet();
 
// 2) LongAdder — best under high contention
LongAdder adder = new LongAdder();
adder.increment();
long total = adder.sum();
 
// 3) synchronized — general but slower
class Counter {
    private long count;
    public synchronized void inc() { count++; }
    public synchronized long get() { return count; }
}
```
Key idea: **don't** use `volatile long count; count++` — that's not atomic (Q37). Prefer `AtomicLong` for exact values at low/moderate contention, `LongAdder` for hot counters under high contention.

### 179. Dining Philosophers problem

Five philosophers, five forks; each needs two adjacent forks to eat. Naive "pick left then right" deadlocks. Fix with **resource ordering** (always pick the lower-numbered fork first) — breaks circular wait.
```java
class DiningPhilosophers {
    private final ReentrantLock[] forks;
    DiningPhilosophers(int n) {
        forks = new ReentrantLock[n];
        for (int i = 0; i < n; i++) forks[i] = new ReentrantLock();
    }
    void dine(int id) throws InterruptedException {
        int left = id, right = (id + 1) % forks.length;
        int first = Math.min(left, right), second = Math.max(left, right); // ordered
        forks[first].lock();
        try {
            forks[second].lock();
            try {
                System.out.println("Philosopher " + id + " eating");
                // eat...
            } finally { forks[second].unlock(); }
        } finally { forks[first].unlock(); }
    }
}
```
Alternatives: use `tryLock` with backoff, allow at most n-1 philosophers at the table (a `Semaphore(n-1)`), or make one philosopher pick forks in the opposite order. All break a Coffman condition (Q72).

### 180. Readers-Writers problem

Many readers may read concurrently, but a writer needs exclusive access. Use `ReadWriteLock`.
```java
class SharedResource {
    private final ReadWriteLock rw = new ReentrantReadWriteLock(true); // fair → avoid writer starvation
    private final Lock readLock = rw.readLock();
    private final Lock writeLock = rw.writeLock();
    private int data;
 
    public int read() {
        readLock.lock();
        try { return data; }                 // concurrent reads allowed
        finally { readLock.unlock(); }
    }
    public void write(int value) {
        writeLock.lock();
        try { data = value; }                // exclusive write
        finally { writeLock.unlock(); }
    }
}
```
Key idea: shared read lock + exclusive write lock. Use a **fair** lock (or a writer-preference policy) to prevent **writer starvation** under heavy read load. For read-mostly data, `StampedLock`'s optimistic read is even faster (Q59).
 
---

## 19. JVM & Internals

### 181. What happens inside Thread.start()?

`start()` does far more than call `run()`:
1. Checks the thread state — throws `IllegalThreadStateException` if already started.
2. Adds the thread to its thread group.
3. Calls the **native `start0()`**, which asks the OS to **create a new native (OS) thread** with its own stack.
4. The OS schedules the new thread; when it runs, the JVM invokes the thread's `run()` **on that new native thread**.
5. The original thread returns from `start()` immediately and continues in parallel.
   So `start()` establishes a happens-before edge (`start()` happens-before the new thread's actions) and creates real OS-level concurrency, whereas `run()` is just a method call on the current thread.

### 182. Native thread vs Java thread

A **Java `Thread`** is an object/abstraction in the JVM. Under the classic (platform-thread) model, each Java thread is backed **1:1** by a **native OS thread** (kernel-scheduled). The JVM delegates scheduling, context switching, and CPU assignment to the OS. So a Java thread = a thin wrapper (stack size, priority, name, state) over a native thread. This 1:1 mapping is why platform threads are relatively heavyweight (each costs a native thread + ~1MB stack) and why you can't have millions of them — which motivated **virtual threads** (Java 21), where many virtual threads are multiplexed onto a small pool of native carrier threads (M:N).

### 183. Green threads vs OS threads

- **Green threads:** threads scheduled entirely by the **JVM/runtime in user space**, not the OS. Many green threads map onto one (or few) OS threads (M:1 or M:N). Cheap to create, but classic green threads can't use multiple cores well and block all of them when one does a blocking syscall. Early Java (1.1) used green threads; they were dropped for native threads.
- **OS (native) threads:** scheduled by the kernel, can run on multiple cores truly in parallel, but are heavier.
  Modern relevance: **virtual threads** (Project Loom, Java 21) are a modern take on green threads — user-mode-scheduled, extremely cheap, but designed to correctly yield the carrier OS thread on blocking so they scale to millions while still using all cores.

### 184. Context switching cost

A context switch is when the CPU saves one thread's state (registers, program counter, stack pointer) and loads another's. Costs:
- **Direct cost:** saving/restoring registers, kernel scheduler work (microseconds each).
- **Indirect cost (often bigger):** **cache and TLB pollution** — the new thread's working set isn't in cache, causing cache misses until it warms up.
  Too many threads → excessive context switching → CPU spends time switching instead of doing work (thrashing). This is why oversized thread pools **hurt** throughput, and why CPU-bound pools should be sized near the core count.

### 185. Thread stack size

Each thread gets its **own stack** for method frames, local variables, and return addresses. Default size is platform-dependent (often ~512KB–1MB on 64-bit HotSpot); tunable with `-Xss`. Implications:
- **Deep recursion** or large local arrays can overflow it → `StackOverflowError`.
- **Many threads × stack size = large memory**: e.g., 1000 threads × 1MB ≈ 1GB reserved — a reason you can't create unlimited threads (`OutOfMemoryError: unable to create native thread`).
- Reducing `-Xss` lets you create more threads but risks `StackOverflowError`.
  Virtual threads sidestep this — their stacks are small and grow/shrink on the heap.

### 186. Why are too many threads bad?

- **Memory:** each platform thread reserves a stack (~1MB) → thousands of threads consume gigabytes; can hit `OutOfMemoryError: unable to create native thread`.
- **Context-switching overhead:** the scheduler thrashes; CPU time is wasted switching instead of computing; cache locality degrades.
- **Contention:** more threads fighting over the same locks → more blocking, more lock waits, diminishing returns (Amdahl's law).
- **Scheduling latency:** more runnable threads than cores means each waits longer for CPU.
  Beyond the core count (for CPU work) or the I/O concurrency you actually need, adding threads reduces throughput. Bound your pools and match size to workload (Q139).

### 187. OutOfMemoryError: unable to create native thread

This error means the JVM asked the OS for a new native thread and the OS refused — it's **not** heap exhaustion. Common causes:
- **Too many threads** (thread leak — creating threads/pools without shutting them down; unbounded `CachedThreadPool` under load).
- **OS limits reached:** `ulimit -u` (max user processes/threads), `/proc/sys/kernel/threads-max`, or `pid_max`.
- **Memory for thread stacks exhausted** (each thread needs stack space; large `-Xss` × many threads).
  Fixes: find and fix thread leaks (bound pools, always `shutdown()`), reduce `-Xss`, raise OS `ulimit`, or move to virtual threads / async I/O for high concurrency. Diagnose with a thread dump to count/categorize threads.

### 188. What is false sharing?

False sharing happens when two threads modify **different** variables that happen to reside on the **same CPU cache line** (typically 64 bytes). Even though the variables are logically independent, the cache-coherence protocol invalidates the whole line on each write, forcing the other core to reload it — so the threads unintentionally contend on the cache line, killing performance. Classic case: adjacent fields/array elements updated by different threads. It's "false" because there's no logical sharing, only physical co-location.

### 189. Cache line padding

Cache line padding is the fix for false sharing: **pad hot fields so each sits alone on its own cache line**, preventing two threads' variables from colliding on one line. Techniques:
- Manually add padding fields (e.g., extra `long`s) around the hot variable.
- Use **`@sun.misc.Contended`** (JDK 8+, requires `-XX:-RestrictContended`), which the JVM honors by inserting padding — used internally by `LongAdder`'s `Cell` and `ForkJoinPool`.
  Trade-off: padding wastes memory, so apply only to genuinely hot, contended fields identified by profiling.

### 190. Biased locking, lightweight locking, heavyweight locking

HotSpot optimizes `synchronized` through escalating tiers based on contention:
- **Biased locking:** optimizes the common case of a lock **repeatedly acquired by the same thread**. The lock is "biased" toward that thread; re-entry needs no atomic CAS — just a check. Cheapest. (Deprecated/disabled by default since JDK 15 because modern hardware made it less worthwhile.)
- **Lightweight locking (thin lock):** when a second thread contends but there's **no actual simultaneous** contention, the JVM uses **CAS** on the object header to acquire/release — spin-based, no OS involvement. Cheap.
- **Heavyweight locking (fat lock):** under **real contention**, the lock inflates to an **OS-level monitor** (mutex); waiting threads are **blocked/parked** by the OS. Most expensive (context switches).
  The JVM starts optimistic and escalates only when contention forces it, so uncontended synchronization is nearly free.

---

## 20. Debugging & Performance

### 191. How do you analyze a thread dump?

A thread dump is a snapshot of every thread's stack and state. Capture with `jstack <pid>`, `jcmd <pid> Thread.print`, `kill -3 <pid>` (to stdout), or JMC/VisualVM. When reading it:
- **Thread states:** many `BLOCKED` on the same monitor → contention/deadlock; many `WAITING`/`TIMED_WAITING` → threads idle (maybe a stuck downstream); many `RUNNABLE` in the same method → CPU hotspot.
- **"waiting to lock" vs "locked":** find which thread **holds** a monitor and which are **waiting** for it.
- **Deadlock section:** the JVM prints "Found one Java-level deadlock" with the cycle.
- **Take 2–3 dumps a few seconds apart:** threads stuck in the same place across dumps indicate a hang; threads that move indicate progress.
- **Group by stack** to find where most threads are (thread-pool saturation, a slow external call).
### 192. How do you identify deadlocks using jstack?

Run `jstack <pid>` and look for the explicit **"Found one Java-level deadlock:"** section — the JVM's built-in deadlock detector lists the involved threads, the locks each **holds**, and the locks each is **waiting to acquire**, showing the cycle (Thread-1 waiting for lock held by Thread-2 and vice versa). You'll also see the threads in **BLOCKED** state with "waiting to lock <address>" pointing at a monitor "locked" by another thread, forming a loop. For programmatic detection, `ThreadMXBean.findDeadlockedThreads()`. Take the dump while the app is hung.

### 193. High CPU due to multithreading — how do you debug?

Correlate OS-level CPU with Java threads:
1. **Find the hot thread(s):** on Linux, `top -H -p <pid>` shows per-thread CPU; note the high-CPU thread's **native id (TID)**.
2. **Convert TID to hex:** `printf '%x\n' <tid>`.
3. **Take a thread dump** (`jstack <pid>`) and find the thread whose **`nid=0x<hex>`** matches — its stack shows exactly what code is burning CPU.
4. **Common culprits:** infinite/spin loops (e.g., busy-waiting, `HashMap` corruption from concurrent use → Java 7 loop), excessive GC (check GC logs), tight retry/CAS loops under contention, or a runaway regex/serialization.
5. **Profile** (async-profiler, JFR flame graphs) to confirm the hotspot over time.
### 194. Thread contention analysis

Contention is when threads compete for the same lock and spend time **blocked** instead of working. To analyze:
- **Thread dumps:** many threads `BLOCKED` "waiting to lock" the same monitor identifies the hot lock.
- **JFR (Java Flight Recorder):** `jdk.JavaMonitorEnter` / monitor-blocked events show which locks cause the most/longest waits.
- **Profilers:** async-profiler `-e lock`, JProfiler/YourKit monitor views quantify lock wait time.
- **Fixes:** reduce critical-section size, use finer-grained or striped locks, replace locks with atomics/concurrent collections, use read-write/optimistic locks, or reduce shared state.
### 195. Lock contention vs CPU contention

- **Lock contention:** threads block **waiting for a lock**; CPU may be **underutilized** (threads parked, not running) yet throughput is poor. Symptom: many BLOCKED threads, low CPU, high latency. Fix by reducing locking/serialization.
- **CPU contention:** more runnable threads than cores, so threads compete for **CPU time**; CPU is **saturated** (near 100%) and context switching is high. Symptom: high CPU, high run-queue length. Fix by reducing thread count/work or scaling cores.
  Distinguishing them tells you whether to optimize **synchronization** (lock contention) or **workload/thread-count** (CPU contention). Low CPU + slow = locks; high CPU + slow = compute/too many threads.

### 196. What tools do you use for thread analysis?

- **`jstack` / `jcmd Thread.print`** — thread dumps, deadlock detection.
- **`top -H`, `ps -T`** — per-thread OS CPU.
- **VisualVM / JConsole** — live thread states, CPU, deadlock detection, basic profiling.
- **Java Mission Control + Java Flight Recorder (JFR)** — low-overhead production profiling of lock contention, thread parks, allocations.
- **async-profiler** — CPU/lock/allocation flame graphs with very low overhead.
- **Commercial profilers** — JProfiler, YourKit (rich thread/lock/monitor views).
- **Thread-dump analyzers** — fastThread.io, Samurai, IBM TMDA for parsing dumps.
- **`ThreadMXBean`** (JMX) — programmatic deadlock detection, CPU time, contention stats.
### 197. How do you profile multithreaded applications?

- Use a **low-overhead sampling profiler** (JFR, async-profiler) in production/realistic load rather than instrumenting profilers that distort timing.
- Capture **CPU flame graphs** to find hot methods, and **lock/monitor events** to find contention and blocked time.
- Track **thread states over time** (running vs blocked vs waiting) to see whether you're CPU-bound or lock-bound.
- Measure **wall-clock latency and throughput**, not just CPU — parallel code can be CPU-cheap but latency-bound on locks/I/O.
- **Load-test** with realistic concurrency; watch queue depths, pool saturation, GC pauses.
- Iterate: profile → fix the top bottleneck → re-profile (bottlenecks shift).
### 198. Common concurrency bugs you've fixed

Framed as talking points from experience:
- **Race conditions** on shared mutable state (e.g., `count++`, check-then-act) → fixed with atomics/locks/idempotency.
- **Visibility bugs** — a non-`volatile` flag causing an infinite loop or stale reads → added `volatile`/proper publication.
- **Deadlocks** from inconsistent lock ordering → imposed a global lock order or used `tryLock`.
- **ThreadLocal leaks/bleed** in pooled threads → added `remove()` in `finally`.
- **Non-thread-safe objects shared** (`SimpleDateFormat`, `HashMap`) across threads → `ThreadLocal` or concurrent types.
- **Silently swallowed exceptions** from `submit()` without checking `Future` → added result handling / `afterExecute`.
- **Double-processing** from message retries → idempotency keys.
- **Thread pool exhaustion** from blocking calls inside the common ForkJoinPool → dedicated executors.
### 199. How do you stress-test concurrent code?

- **High-concurrency load tests:** many threads hammering the code (JMeter, Gatling, `k6`, or custom `ExecutorService` harness) to surface races under contention.
- **Repeat + randomize:** run the scenario thousands of times with randomized timing/interleavings; concurrency bugs are non-deterministic, so a single pass isn't enough.
- **jcstress** — the JDK's specialized harness for testing concurrency invariants and memory-model behavior; it explores interleavings systematically.
- **Assertions/invariant checks** under load to catch corruption (e.g., final count must equal expected).
- **Chaos/latency injection:** add artificial delays to widen race windows; use thread-affinity/CPU pinning to expose visibility issues.
- **Run on multiple cores / different hardware** and enable flags (`-XX:+StressLCM`, etc.) to change scheduling.
- **Static/dynamic tools:** FindBugs/SpotBugs, ThreadSanitizer-style analysis, and code review for happens-before reasoning.
### 200. What multithreading challenges have you solved in production?

Strong closing narrative examples:
- **Deadlock in a payment/transfer flow** — diagnosed via `jstack`, root-caused to inconsistent lock ordering across accounts, fixed with ordered locking + `tryLock` timeouts.
- **Double charging under retries** — introduced idempotency keys + a DB unique constraint so at-least-once delivery became effectively exactly-once.
- **CPU spiking to 100%** — traced with `top -H` + thread dump to a `HashMap` used concurrently (Java 7 infinite loop); replaced with `ConcurrentHashMap`.
- **Memory leak / `OOM: unable to create native thread`** — an unbounded executor created per request; switched to a shared bounded pool with proper shutdown.
- **Lock contention bottleneck** — a coarse `synchronized` on a hot path serialized everything; replaced with `LongAdder`/striped locks/`ConcurrentHashMap`, multiplying throughput.
- **ThreadLocal context bleeding across pooled requests** — user/tenant context leaked to the next request; enforced `remove()` in a `finally`/filter.
- **Scheduled job running on every instance** in a cluster → added a distributed lock (ShedLock) for single execution.
  When answering, pick one or two, and structure as: **symptom → diagnosis (tools) → root cause → fix → outcome/metric.**

---

## Quick Revision Cheat Sheet

- **Visibility** → `volatile` / happens-before. **Atomicity of compound ops** → `synchronized` / `Atomic*` / locks. `volatile` gives visibility, **not** atomic `x++`.
- **CAS** underlies lock-free atomics; watch the **ABA** problem (`AtomicStampedReference`). **`LongAdder` > `AtomicLong`** under high contention.
- **`synchronized`** auto-releases on exception; **`Lock`** needs `unlock()` in `finally`. `ReentrantLock` adds tryLock, interruptible, fairness, multiple conditions.
- **`wait()` releases the lock; `sleep()` doesn't.** Always `wait()` in a `while` loop (spurious wakeups). Prefer `notifyAll()`.
- **Deadlock** = 4 Coffman conditions; break **circular wait** with **lock ordering** or `tryLock`.
- **Prefer executors** over manual threads; **bounded queues** + rejection policy; shut down properly. `submit()` swallows exceptions until `get()`.
- **`CompletableFuture`**: `thenApply` (map) vs `thenCompose` (flatMap); `thenCombine` (parallel) vs `thenCompose` (dependent); always pass a dedicated executor for I/O.
- **`ConcurrentHashMap`**: per-bin locking, lock-free reads, treeified bins, no nulls. **`CopyOnWriteArrayList`** for read-mostly.
- **ThreadLocal**: per-thread state; **always `remove()`** in pooled threads (leak + data bleed).
- **Sizing**: CPU-bound ≈ cores; IO-bound = cores × (1 + wait/compute). **Measure.**
- **Debugging**: `jstack` for deadlocks/contention; `top -H` + hex nid for high-CPU threads; JFR/async-profiler for lock contention.
---

*End of guide — 200 questions covered. Good luck with your interviews.*