# Backend / Java Interview Prep — Complete Answer Guide

A consolidated set of interview-ready answers covering Java, Spring Boot, JPA/Hibernate, Kafka, SQL, RAG/GenAI, DSA, System Design, and resilience. Each answer is written to be spoken confidently in an interview and to hold up to follow-up questions.

---

## Table of Contents

1. [Java Core](#1-java-core)
2. [Java Concurrency & Threads](#2-java-concurrency--threads)
3. [Spring Boot & Backend](#3-spring-boot--backend)
4. [JPA / Hibernate](#4-jpa--hibernate)
5. [Resilience & API Gateway](#5-resilience--api-gateway)
6. [Kafka](#6-kafka)
7. [SQL & Databases](#7-sql--databases)
8. [RAG & Generative AI](#8-rag--generative-ai)
9. [DSA / Coding](#9-dsa--coding)
10. [System Design](#10-system-design)
11. [Miscellaneous / Production Topics](#11-miscellaneous--production-topics)

---

## 1. Java Core

### What is a Functional Interface?

A functional interface is an interface with **exactly one abstract method** (a Single Abstract Method, or SAM). It can have any number of `default` and `static` methods, but only one unimplemented method. Because it has a single abstract method, it can be the target type of a **lambda expression** or **method reference**.

Examples from the JDK: `Runnable` (`run()`), `Callable` (`call()`), `Comparator` (`compare()`), `Supplier`, `Consumer`, `Function`, `Predicate`. The `@FunctionalInterface` annotation is optional but recommended — it makes the compiler enforce the "exactly one abstract method" rule so you get a compile error if someone accidentally adds a second abstract method.

```java
@FunctionalInterface
interface Calculator {
    int operate(int a, int b);
}

Calculator add = (a, b) -> a + b;      // lambda
Calculator mul = (a, b) -> a * b;
System.out.println(add.operate(2, 3)); // 5
```

Key point: `equals`, `hashCode`, `toString` (public methods of `Object`) don't count toward the abstract-method count, so an interface redeclaring `Object` methods can still be functional.

### What is a default method?

A default method (Java 8+) is a method in an interface with a body, declared with the `default` keyword. It lets you **add new methods to an interface without breaking existing implementors** — the classic motivating example was adding `stream()`, `forEach()` etc. to the `Collection`/`Iterable` hierarchy without forcing every implementation to change.

```java
interface Vehicle {
    void start();
    default void honk() { System.out.println("Beep!"); } // has a body
}
```

Implementors inherit `honk()` for free but can override it. This is Java's answer to the "interface evolution" problem and enables a limited, controlled form of multiple inheritance of *behavior* (not state).

### What is the output when a class implements two interfaces with the same default method?

If a class implements two interfaces that both define a default method with the same signature (e.g. `newMethod()`), the code **won't compile** — this is the "diamond problem" for default methods. Java forces you to resolve the ambiguity explicitly by overriding the method, and you can call a specific parent's version using `InterfaceName.super.method()`.

```java
interface A { default String newMethod() { return "A"; } }
interface B { default String newMethod() { return "B"; } }

class C implements A, B {
    @Override
    public String newMethod() {
        return A.super.newMethod(); // must resolve; returns "A"
    }
}
// Without the override, compiler error:
// "class C inherits unrelated defaults for newMethod() from types A and B"
```

So the "output" answer: it does not compile unless you override; once you override and delegate to `A.super.newMethod()`, the output is `A`.

### What is the use of the Optional class?

`Optional<T>` is a container that may or may not hold a non-null value. Its purpose is to make the **possibility of absence explicit in the type system** and to reduce `NullPointerException`s by encouraging you to handle the empty case rather than silently dereferencing null.

```java
Optional<User> user = repository.findById(id);
String name = user.map(User::getName).orElse("Unknown");
user.ifPresent(u -> log.info("Found {}", u));
User must = user.orElseThrow(() -> new NotFoundException(id));
```

Best-practice notes an interviewer likes to hear: use `Optional` primarily as a **return type** for methods that may not find a result; avoid using it as a field or method parameter, and never call `.get()` without checking `isPresent()` (prefer `orElse`, `orElseGet`, `orElseThrow`, `map`, `flatMap`). Don't use `Optional` for collections — return an empty list instead.

### What is a Marker Interface? How do you create one?

A marker interface is an **empty interface** (no methods, no fields) used to *tag* or *mark* a class so that code — often the JVM or a framework — can check `instanceof` and change behavior. Classic examples: `Serializable`, `Cloneable`, `RandomAccess`.

```java
public interface Auditable { }   // marker: no members

public class Order implements Auditable { }

// somewhere in a framework:
if (obj instanceof Auditable) {
    audit(obj);
}
```

The marker carries no behavior itself — it conveys **metadata** ("this class is allowed / intended to do X"). Since Java 5, **annotations** are usually the preferred, more flexible mechanism (they can carry parameters and be retained at runtime), which leads to the next question.

### Can we create our own Marker Annotation?

Yes. A marker annotation is an annotation with **no elements** — it just tags. You declare it with `@interface` and typically set retention to `RUNTIME` so it's visible via reflection.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Auditable { }

@Auditable
public class Order { }

// reflection check:
if (Order.class.isAnnotationPresent(Auditable.class)) {
    audit();
}
```

Annotations are generally preferred over marker interfaces because they can target methods/fields, carry attributes, and be processed at compile time or runtime — while a marker interface only tags a *type* and pollutes the type hierarchy.

### How does HashMap handle collision attacks in Java 8?

In a `HashMap`, entries with the same bucket index form a chain. Before Java 8 the chain was always a **linked list**, giving O(n) lookup in the worst case — an attacker who deliberately submits many keys with colliding hashes (a **hash-flooding / hash DoS** attack) could degrade an operation to O(n²).

Java 8 mitigates this with **treeification**: when a single bucket's chain length exceeds `TREEIFY_THRESHOLD` (8) *and* the table capacity is at least `MIN_TREEIFY_CAPACITY` (64), the bucket is converted from a linked list into a **balanced red-black tree**, dropping worst-case lookup within that bucket from O(n) to **O(log n)**. If the bucket shrinks below `UNTREEIFY_THRESHOLD` (6), it reverts to a list. Java 8 also improved the hash spreading function (`h ^ (h >>> 16)`) to mix high bits into the low bits used for indexing.

### Is HashMap a good or bad practice in a DoS scenario? And what if the client intentionally forces identical hash codes?

`HashMap` is fine for normal application code, but it is **not designed to be safe against adversarial input**. Treeification makes worst-case degraded lookups O(log n) instead of O(n), which greatly reduces the impact — but it only kicks in for keys that are `Comparable` (like `String`, `Integer`). If the keys are not `Comparable`, the tree can't order them and it degrades back toward list behavior.

So if a system is already released and a malicious client intentionally crafts keys with identical hash codes:
- With `String`/`Integer` keys you still get O(log n) per bucket thanks to red-black trees — the attack is largely neutralized.
- The correct defensive strategy for untrusted, user-supplied keys is **not to rely on `HashMap` semantics** alone: validate/limit input size, cap the number of distinct keys, use a keyed/randomized hash (SipHash-style seed) so the attacker can't predict buckets, or store untrusted data in a structure with guaranteed bounds. Frameworks that parse untrusted request parameters into maps typically cap the number of parameters for exactly this reason.

The honest interview answer: `HashMap` is a good general-purpose structure; for adversarial input you add input limits and don't let untrusted clients control the key space unbounded.

### What happens memory-wise when an ArrayList reaches its capacity/threshold?

An `ArrayList` is backed by an array. When you `add` and the backing array is full, it **grows**: it allocates a new, larger array — in the OpenJDK implementation the new capacity is roughly **oldCapacity + (oldCapacity >> 1)**, i.e. about **1.5×** — then **copies** all existing elements into it (`Arrays.copyOf`, effectively `System.arraycopy`) and the old array becomes garbage.

Implications:
- The resize is an **O(n) copy**, but because it happens geometrically (1.5×), the *amortized* cost of `add` stays **O(1)**.
- There is a transient moment where **both** old and new arrays exist, so peak memory is ~2.5× the old array during the copy.
- After growth the list may hold spare capacity; `trimToSize()` releases it.
- If you know the size in advance, pre-size with `new ArrayList<>(expectedSize)` to avoid repeated resizes and copies.

### What is GC pressure?

GC (garbage collection) pressure is the rate at which your application creates garbage — short-lived objects that must be reclaimed. High allocation rates mean the GC has to run **more frequently** and do **more work**, which consumes CPU and causes **pause times / latency spikes** ("stop-the-world" pauses), reducing throughput.

Symptoms: frequent young-generation (minor) GCs, rising CPU spent in GC, latency jitter, and — if objects are promoted to old gen faster than they die — more expensive full GCs and eventually `OutOfMemoryError`. Common causes: creating lots of temporary objects in hot loops, boxing/unboxing, string concatenation in loops, large intermediate collections. You reduce GC pressure by reusing objects, using primitives wisely, pre-sizing collections, avoiding unnecessary allocations, and pooling only where truly justified. GC logs, JFR (Java Flight Recorder), and APM tools help you spot it.

### Difference between SynchronousQueue and TransferQueue

Both are `BlockingQueue` implementations focused on **handoff** between threads.

- **`SynchronousQueue`** has **zero capacity** — it holds no elements at all. Each `put()` must wait for a corresponding `take()` and vice versa; it's a pure rendezvous/handoff point. It's what `Executors.newCachedThreadPool()` uses internally. You can't peek; `size()` is always 0.

- **`TransferQueue`** (implemented by `LinkedTransferQueue`) is a **superset**: it *is* an unbounded queue, but it adds a `transfer(e)` method that **blocks until another thread receives the element** (like a synchronous handoff), plus `tryTransfer(...)`. So it gives you the choice per operation: `put`/`add` behave like a normal queue (don't wait for a consumer), while `transfer` behaves like `SynchronousQueue` (wait for a consumer). It's more flexible: producers can hand off directly when a consumer is waiting, or enqueue otherwise.

Summary: `SynchronousQueue` = always-handoff, no storage; `TransferQueue` = can store *and* can force-handoff on demand.

### Compare ArrayBlockingQueue and LinkedBlockingQueue

Both are thread-safe blocking FIFO queues used in producer-consumer designs (e.g., as the work queue of a `ThreadPoolExecutor`), but they differ structurally:

| Aspect | ArrayBlockingQueue | LinkedBlockingQueue |
|---|---|---|
| Backing structure | Fixed-size array (ring buffer) | Linked nodes |
| Capacity | **Bounded, mandatory** at construction | Optional bound; **defaults to Integer.MAX_VALUE** (effectively unbounded) |
| Locking | **Single lock** for put and take | **Two separate locks** (putLock, takeLock) → higher throughput |
| Memory | Allocated up front; no per-element node objects | Allocates a node per element (more GC, more memory per element) |
| Throughput | Lower under heavy concurrency (one lock) | Higher — producers and consumers don't block each other |

Practical guidance: use `ArrayBlockingQueue` when you want a **strict, predictable capacity** and predictable memory, and `LinkedBlockingQueue` when you want **higher throughput** and can tolerate dynamic sizing — but be careful with its unbounded default, which can hide backpressure and cause `OutOfMemoryError` if producers outrun consumers. In a thread pool, always give `LinkedBlockingQueue` an explicit bound.

---

## 2. Java Concurrency & Threads

### What is the difference between CompletableFuture and Virtual Threads? When should each be used?

They solve overlapping problems (doing many things concurrently, especially I/O) but at different layers.

**`CompletableFuture`** (Java 8) is an **asynchronous, non-blocking composition API**. You describe a pipeline of stages — `supplyAsync`, `thenApply`, `thenCompose`, `thenCombine`, `exceptionally` — that run on a thread pool (default `ForkJoinPool.commonPool()`). The value is that a small number of platform threads are never *blocked* waiting; callbacks fire when results are ready. The cost is **programming model complexity**: you have to write callback/chained code, error handling is awkward, and it's easy to accidentally block a pool thread.

**Virtual Threads** (Project Loom, stable in Java 21) are **lightweight threads managed by the JVM**, not the OS. You write ordinary, **blocking, synchronous** code (`thread.join()`, blocking I/O), but a virtual thread that blocks on I/O is **unmounted** from its carrier (platform) thread so the OS thread is freed to run other virtual threads. You can have **millions** of them. The value is you get the scalability of async with the **simplicity of blocking code** — no callback hell.

When to use each:
- **Virtual threads**: I/O-bound workloads with high concurrency (thousands of concurrent requests each doing DB/HTTP calls) where you want simple, readable, thread-per-request code. This is the modern default for such workloads on Java 21+.
- **CompletableFuture**: when you need to **compose and orchestrate** multiple async operations with dependencies (fan-out/fan-in, combining results, timeouts, pipelines) — its combinator API is expressive for that. Also still relevant on JVMs before 21.
- They **combine well**: run blocking calls on virtual threads and still use `CompletableFuture`/structured concurrency to coordinate results.
- For **CPU-bound** work, neither magically helps — you're bounded by cores; use a sized pool of platform threads.

### Which Garbage Collector for a high-throughput, low-latency application, and why?

First, be precise: "high throughput" and "low latency" pull in slightly different directions, so name the trade-off, then pick.

- **G1 GC** (default since Java 9) is the balanced choice — it does mostly-concurrent work, divides the heap into regions, and lets you set a pause-time target (`-XX:MaxGCPauseMillis`). Good all-rounder for large heaps with predictable, moderate pauses (tens of ms).
- **ZGC** (production-ready from Java 15+) is the answer when **low latency is the priority**: it's a concurrent, region-based collector with **sub-millisecond, pause times that stay roughly constant regardless of heap size** (works well from small heaps up to terabytes). Choose ZGC for latency-sensitive services (payments, trading, real-time APIs) where you cannot tolerate long stop-the-world pauses.
- **Shenandoah** (Red Hat, OpenJDK) is a comparable concurrent low-pause collector — another valid low-latency choice.
- **Parallel GC** maximizes raw **throughput** (batch jobs, analytics) at the cost of longer stop-the-world pauses — good when total work done matters more than per-request latency.

So the crisp answer: for a service that needs **both** high throughput and low latency on a modern JDK, I'd start with **G1** as the safe default and move to **ZGC** (or Shenandoah) if pause times are the binding constraint — because ZGC keeps pauses in the sub-millisecond range even with very large heaps, whereas if it were a pure throughput batch job I'd consider **Parallel GC**. Always validate with GC logs and load tests rather than guessing.

### How would you implement a high-performance, thread-safe payment worker pool?

The core idea: **bounded queue + fixed thread pool + idempotency + backpressure + graceful shutdown**, because payments are money and must be exactly-once-effective and never lost.

Design:
- Use a **`ThreadPoolExecutor`** with a fixed core/max thread count sized to your throughput and downstream limits, and a **bounded `ArrayBlockingQueue`** so you get backpressure instead of unbounded memory growth.
- Use a **`CallerRunsPolicy`** (or a custom rejection handler that persists to a durable store) so that when the queue is full you slow producers down rather than dropping payments.
- Each task must be **idempotent** — key each payment by a unique `paymentId`/idempotency key and check a store before charging, so a retry never double-charges.
- Make shared state thread-safe with immutable task objects and concurrent structures; avoid shared mutable state, or guard it with `ReentrantLock`/atomic types.
- **Durability**: enqueue from a persistent source (DB outbox or Kafka) so a crash doesn't lose in-flight work; commit offset / mark done only after success.
- **Graceful shutdown**: `shutdown()` then `awaitTermination(...)` so in-flight payments finish; on timeout, `shutdownNow()` and let the durable store re-drive.

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    16, 16,                       // fixed size, tuned to downstream capacity
    0L, TimeUnit.MILLISECONDS,
    new ArrayBlockingQueue<>(1000),                  // bounded -> backpressure
    new ThreadFactoryBuilder().setNameFormat("pay-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy());      // slow producers when full

void submit(Payment p) {
    pool.execute(() -> {
        if (ledger.alreadyProcessed(p.idempotencyKey())) return; // idempotent
        try {
            gateway.charge(p);            // with retry + circuit breaker
            ledger.markProcessed(p.idempotencyKey());
        } catch (RetryableException e) {
            outbox.reschedule(p);         // durable retry
        }
    });
}
```

Also add per-task **timeouts**, a **circuit breaker** around the gateway call, and metrics (queue depth, latency, success/failure) so you can see saturation. Modern alternative: with **virtual threads** (`Executors.newVirtualThreadPerTaskExecutor()`) plus a **semaphore** to cap concurrency toward the payment gateway — this keeps the code blocking-simple while limiting downstream load.

### Advanced synchronization mechanisms in Java

Beyond `synchronized`, the `java.util.concurrent.locks` and atomic packages provide:

- **`ReentrantLock`** — like `synchronized` but with `tryLock()`, timed lock, interruptible lock, and optional **fairness**.
- **`ReentrantReadWriteLock`** — separate read and write locks; many concurrent readers, exclusive writer. Good for read-heavy shared state.
- **`StampedLock`** — adds **optimistic reads** (a cheap validation-based read that doesn't block writers) on top of read/write modes; higher throughput but not reentrant.
- **`Semaphore`** — permits to limit concurrent access to a resource (e.g., cap DB or gateway concurrency).
- **`CountDownLatch`** — one-shot "wait until N events happen" gate.
- **`CyclicBarrier`** — reusable barrier where N threads wait for each other, then all proceed.
- **`Phaser`** — flexible, reusable, multi-phase barrier with dynamic party registration.
- **`Exchanger`** — two threads swap objects at a rendezvous point.
- **Atomics** (`AtomicInteger`, `AtomicReference`, `LongAdder`) — lock-free CAS-based updates; `LongAdder` scales better than `AtomicLong` under high contention.
- **`Condition`** — await/signal on a `Lock`, replacing `wait/notify`.

### Lock scenarios (real-world use cases)

- **`ReentrantReadWriteLock` / `StampedLock`**: an in-memory cache or config that's read constantly but updated rarely — let readers run concurrently, block only on writes.
- **`Semaphore`**: limiting concurrent calls to a rate-limited third-party API or a fixed-size connection pool.
- **`ReentrantLock` with `tryLock(timeout)`**: acquiring a resource without risking a permanent hang — back off if you can't get it, which also helps avoid deadlock.
- **`CountDownLatch`**: a service that must wait for several async initializations (DB warmup, cache load) before it starts serving.
- **`CyclicBarrier`**: parallel computation split across threads that must all finish a phase before the next phase.
- **Distributed locks (Redisson/ZooKeeper)**: coordinating across *multiple JVMs/pods*, e.g. ensuring only one instance runs a scheduled job.

### What are the different states of a thread?

Per `Thread.State`:
- **NEW** — created but `start()` not yet called.
- **RUNNABLE** — eligible to run; either running on a CPU or ready and waiting for CPU time (Java doesn't distinguish "ready" vs "running").
- **BLOCKED** — waiting to acquire a monitor lock to enter a `synchronized` block/method.
- **WAITING** — waiting indefinitely for another thread's action: `Object.wait()`, `Thread.join()`, `LockSupport.park()` with no timeout.
- **TIMED_WAITING** — waiting for a bounded time: `sleep(ms)`, `wait(ms)`, `join(ms)`, `parkNanos`.
- **TERMINATED** — finished execution (run method returned or threw).

Lifecycle: NEW → RUNNABLE → (BLOCKED/WAITING/TIMED_WAITING ↔ RUNNABLE) → TERMINATED.

### What is a race condition? How do you avoid it?

A **race condition** occurs when the correctness of a program depends on the **relative timing/interleaving** of multiple threads accessing shared mutable state, and at least one of them writes. The classic example is a non-atomic **read-modify-write** like `count++` (read, increment, write): two threads can both read the same value and one update is lost.

How to avoid:
- **Don't share mutable state** — prefer immutable objects, local variables, and confining state to a single thread (thread-per-request, actors).
- **Synchronize** access with `synchronized`, `ReentrantLock`, so read-modify-write is atomic.
- Use **atomic classes** (`AtomicInteger.incrementAndGet()`) or `LongAdder` for counters — lock-free CAS.
- Use **concurrent collections** (`ConcurrentHashMap`, `CopyOnWriteArrayList`) instead of external locking.
- Ensure **visibility** with `volatile` for flags (visibility only, not compound atomicity).
- At the data layer, use **DB transactions and optimistic/pessimistic locking** for cross-request races.

---

## 3. Spring Boot & Backend

### How does @Transactional work internally?

`@Transactional` is **declarative transaction management** implemented with **Spring AOP proxies**. When Spring sees the annotation, it wraps the bean in a **proxy** (JDK dynamic proxy if it implements an interface, otherwise a CGLIB subclass). Calls to the annotated method go through the proxy, which runs a `TransactionInterceptor` **around** your method:

1. Before the method: the interceptor asks a `PlatformTransactionManager` (e.g. `DataSourceTransactionManager`, `JpaTransactionManager`) to **get a transaction** according to the **propagation** setting. Typically it opens a connection, sets `autoCommit=false`, and binds the connection/EntityManager to the current thread via a `ThreadLocal` (`TransactionSynchronizationManager`).
2. It invokes your actual method.
3. On **normal return**, it **commits**.
4. On a **RuntimeException / Error** (by default), it **rolls back**; checked exceptions do **not** roll back unless you set `rollbackFor`.
5. It restores the previous transaction state and cleans up the thread binding.

Key consequences you should mention:
- Because it's proxy-based, **self-invocation doesn't work**: calling one `@Transactional` method from another method *in the same class* bypasses the proxy, so the annotation is ignored. Call through the proxy (inject the bean into itself, or split into another bean).
- Only **public** methods are advised by default.
- **Propagation** (`REQUIRED` default, `REQUIRES_NEW`, `NESTED`, `SUPPORTS`, `MANDATORY`, `NEVER`, `NOT_SUPPORTED`) controls whether it joins an existing transaction or starts a new one.
- **Isolation** maps to the DB isolation level; **readOnly** is a hint/optimization.

### How do begin(), commit(), and rollback() work internally?

Underneath the abstraction these map to JDBC/driver operations on a **`Connection`**:
- **begin**: there's no literal `begin()` in JDBC — a transaction starts implicitly when you set `connection.setAutoCommit(false)`. Spring's transaction manager does this and records a savepoint context for the current thread.
- **commit**: `connection.commit()` — the DB makes all changes in the transaction **durable and visible** to others, releasing locks. In JPA, the persistence context is **flushed** (SQL sent) first, then the DB commits.
- **rollback**: `connection.rollback()` — the DB **discards** all uncommitted changes since the transaction began (or since a savepoint), undoing them and releasing locks.

For nested transactions, Spring uses **JDBC savepoints** (`Savepoint sp = conn.setSavepoint()` / `conn.rollback(sp)`) to implement `PROPAGATION_NESTED`. `REQUIRES_NEW` suspends the current transaction (unbinds its connection) and starts a genuinely separate one.

### How do you create a custom Spring Boot Starter?

A starter is a convenience dependency that brings in libraries **and** auto-configures beans so consumers get functionality by just adding the dependency.

Steps (Spring Boot 3 / 2.7+ convention):
1. **Two modules** (recommended): `xxx-spring-boot-autoconfigure` (the auto-config + `@ConfigurationProperties`) and `xxx-spring-boot-starter` (a thin pom that just depends on the autoconfigure module and the required libs). Naming convention: never prefix with `spring-boot` (that's reserved); use `acme-spring-boot-starter`.
2. Write an **auto-configuration class** with conditional beans:

```java
@AutoConfiguration
@ConditionalOnClass(GreetingService.class)
@EnableConfigurationProperties(GreetingProperties.class)
public class GreetingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean            // let users override
    public GreetingService greetingService(GreetingProperties props) {
        return new GreetingService(props.getMessage());
    }
}

@ConfigurationProperties(prefix = "acme.greeting")
public class GreetingProperties {
    private String message = "Hello";
    // getters/setters
}
```

3. **Register** the auto-config so Boot discovers it. In Boot 3, list it in
   `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
   (in Boot 2 it was `spring.factories` under `EnableAutoConfiguration`).
4. Use `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty` so the config only activates when appropriate and users can override defaults.
5. Consumers add the starter dependency and configure via `application.yml` (`acme.greeting.message: Hi`).

### While building a REST Controller, what annotations would you use?

- **`@RestController`** — combines `@Controller` + `@ResponseBody`; return values are serialized (usually to JSON) directly.
- **`@RequestMapping`** — base path / shared config at class level.
- **`@GetMapping` / `@PostMapping` / `@PutMapping` / `@PatchMapping` / `@DeleteMapping`** — HTTP-verb-specific handlers.
- **`@PathVariable`** — bind URI template variables (`/users/{id}`).
- **`@RequestParam`** — bind query params (`?page=2`).
- **`@RequestBody`** — deserialize request body into an object.
- **`@ResponseStatus`** — set the HTTP status for a handler or exception.
- **`@RequestHeader`, `@CookieValue`** — bind headers/cookies.
- **`@Valid` / `@Validated`** — trigger Bean Validation on the body/params.
- **`@ResponseBody` / `ResponseEntity<T>`** — the latter gives full control over status, headers, body.
- **`@ExceptionHandler` / `@RestControllerAdvice`** — centralized error handling.

### What are the HTTP response codes in REST APIs?

- **2xx Success**: `200 OK`, `201 Created` (with `Location` header, for POST that creates), `202 Accepted` (async), `204 No Content` (successful, no body — e.g. DELETE).
- **3xx Redirection**: `301 Moved Permanently`, `304 Not Modified` (caching).
- **4xx Client errors**: `400 Bad Request` (validation), `401 Unauthorized` (not authenticated), `403 Forbidden` (authenticated but not allowed), `404 Not Found`, `405 Method Not Allowed`, `409 Conflict` (e.g. optimistic-lock/duplicate), `422 Unprocessable Entity`, `429 Too Many Requests` (rate limiting).
- **5xx Server errors**: `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`.

### What are the different ways of exception handling in Spring Boot?

1. **`@ExceptionHandler` methods** inside a controller — handle exceptions for that controller only.
2. **`@RestControllerAdvice` / `@ControllerAdvice`** — global, centralized handlers across all controllers; the standard approach for consistent error responses.
3. **`ResponseStatusException`** — throw it directly with a status and reason (no custom class needed).
4. **`@ResponseStatus` on a custom exception** — maps an exception type to an HTTP status.
5. **`ResponseEntityExceptionHandler`** — extend it in your advice to customize Spring MVC's built-in exceptions (validation, message-not-readable, etc.).
6. **`ErrorController` / problem-details** — Spring Boot 3 supports RFC 7807 `ProblemDetail` for standardized error bodies.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(EntityNotFoundException.class)
    public ProblemDetail handleNotFound(EntityNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<?> handleValidation(MethodArgumentNotValidException ex) {
        // collect field errors...
        return ResponseEntity.badRequest().body(/* details */);
    }
}
```

### How does OAuth2 authentication work?

OAuth2 is an **authorization** framework (delegated access) — often paired with **OIDC** for authentication. The key roles: **Resource Owner** (the user), **Client** (your app), **Authorization Server** (issues tokens, e.g. Keycloak/Okta/Google), **Resource Server** (the API holding protected data).

The most common flow — **Authorization Code (with PKCE for public clients)**:
1. The client redirects the user to the **authorization server's** login/consent page.
2. The user authenticates and consents; the auth server redirects back to the client with a short-lived **authorization code**.
3. The client exchanges the code (plus its client secret / PKCE verifier) at the **token endpoint** for an **access token** (and usually a **refresh token** and, with OIDC, an **ID token**).
4. The client calls the resource server with `Authorization: Bearer <access_token>`.
5. The resource server **validates** the token — either by introspection or, for **JWTs**, by verifying the signature (via the auth server's JWKS public keys) and checking claims (`exp`, `aud`, `scope`).
6. When the access token expires, the client uses the **refresh token** to get a new one without re-prompting the user.

Other grant types: **Client Credentials** (machine-to-machine, no user), **Device Code** (TVs/CLI). In Spring, `spring-boot-starter-oauth2-resource-server` validates bearer JWTs and `spring-boot-starter-oauth2-client` handles the login/redirect flow.

### How do you version REST APIs?

Common strategies, with trade-offs:
- **URI path versioning** — `/api/v1/users`, `/api/v2/users`. Most explicit, easy to route and cache; the most widely used. Downside: version leaks into the URL and duplicating controllers.
- **Query parameter** — `/users?version=2`. Simple but easy to miss and weaker for caching.
- **Custom header** — `X-API-Version: 2`. Keeps URLs clean; harder to test/browse.
- **Content negotiation / media type** — `Accept: application/vnd.company.v2+json`. RESTful/"purest", but more complex for clients.

Practical guidance: **URI versioning** is the pragmatic default for public APIs. Whatever you choose, version only on **breaking changes**, keep old versions running through a deprecation window, and prefer **backward-compatible additive changes** (new optional fields) so you rarely need a new version at all.

---

## 4. JPA / Hibernate

### What is the difference between flush() and commit() in Hibernate?

- **`flush()`** synchronizes the **persistence context (first-level cache)** with the database: Hibernate generates and sends the pending SQL (INSERT/UPDATE/DELETE) to the DB **within the current transaction**. The changes are now *visible inside* the same transaction and to the DB session, but they are **not yet permanent** — they can still be **rolled back**. Flush does **not** end the transaction.
- **`commit()`** ends the transaction: it triggers a flush (if needed) and then issues a **database COMMIT**, making all changes **permanent, durable, and visible to other transactions**, and releasing locks. After commit you cannot roll back.

Analogy: `flush()` = "send the SQL now"; `commit()` = "make it permanent." Hibernate auto-flushes at certain points (before query execution that could be affected, and at commit) based on the `FlushMode`.

### What is LazyInitializationException and how do you fix it?

It happens when you access a **lazily-loaded** association (e.g. `@OneToMany(fetch = LAZY)`) **after the Hibernate Session / persistence context is already closed**. Lazy fields are backed by a proxy that fetches data on first access; if the transaction/session has ended (common when you touch the collection in the view/controller layer after the service method returned), Hibernate has no open session to run the query and throws `LazyInitializationException`.

Fixes (in rough order of preference):
- **Fetch what you need inside the transaction**: use a **JPQL `JOIN FETCH`** or an **`@EntityGraph`** to eagerly load the needed associations in the same query.
- **DTO projection**: select exactly the fields you need into a DTO within the transactional boundary — avoids lazy access entirely and is the cleanest for APIs.
- **Initialize explicitly** inside the transaction: `Hibernate.initialize(entity.getItems())`.
- **Keep the transaction open longer** only where appropriate (e.g. `@Transactional` on the service method that also does the mapping).
- **Avoid** the anti-patterns: `FetchType.EAGER` everywhere (causes N+1 and over-fetching) and Open-Session-In-View (`spring.jpa.open-in-view=true`) which hides the problem and holds connections too long.

### What is optimistic locking and pessimistic locking in JPA/Hibernate?

Both prevent **lost updates** from concurrent modification, with opposite assumptions:

**Optimistic locking** assumes conflicts are **rare**. It uses a **version column** (`@Version` on an int/long/timestamp). On update, Hibernate issues `UPDATE ... WHERE id=? AND version=?` and increments the version; if the row's version changed meanwhile, **zero rows update** and Hibernate throws `OptimisticLockException` (HTTP 409 to the client), who then retries. No DB locks are held between read and write, so it **scales well** and avoids blocking — ideal for **read-heavy, low-contention** web apps.

```java
@Entity
class Account {
    @Id Long id;
    @Version Long version;   // enables optimistic locking
    BigDecimal balance;
}
```

**Pessimistic locking** assumes conflicts are **likely**. It takes an actual **database lock** at read time so no one else can modify (or even read, depending on mode) the row until you commit. Use `LockModeType.PESSIMISTIC_WRITE` (`SELECT ... FOR UPDATE`) or `PESSIMISTIC_READ`. It **guarantees** no conflict but **serializes** access, reduces concurrency, holds locks, and risks **deadlocks/timeouts** — use it for **high-contention, critical** operations (e.g. decrementing limited inventory, moving money) where a retry loop is unacceptable.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select a from Account a where a.id = :id")
Account findForUpdate(@Param("id") Long id);
```

Use-case summary: **optimistic** for the common web case (better throughput, retry on conflict); **pessimistic** when a conflict would be costly and you must block rather than retry.

### How do you define an entity class with a composite primary key?

Two standard approaches:

**1. `@EmbeddedId`** — a single embeddable key class:

```java
@Embeddable
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;
    // equals() and hashCode() are REQUIRED
}

@Entity
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;
    private int quantity;
}
```

**2. `@IdClass`** — separate id class, individual `@Id` fields on the entity:

```java
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;   // field names must match; equals/hashCode required
}

@Entity
@IdClass(OrderItemId.class)
public class OrderItem {
    @Id private Long orderId;
    @Id private Long productId;
    private int quantity;
}
```

Requirements to stress: the key class must be `Serializable` and **must implement `equals()` and `hashCode()`** (Hibernate uses them to identify entities). `@EmbeddedId` is generally cleaner and preferred; `@IdClass` reads more like plain columns.

### How would you define entity classes for a Customer–Address one-to-many relationship?

One customer has many addresses; each address belongs to one customer. Model the **owning side** on the `Address` (the side with the foreign key).

```java
@Entity
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "customer",
               cascade = CascadeType.ALL,
               orphanRemoval = true,
               fetch = FetchType.LAZY)
    private List<Address> addresses = new ArrayList<>();

    // helper keeps both sides in sync
    public void addAddress(Address a) {
        addresses.add(a);
        a.setCustomer(this);
    }
}

@Entity
public class Address {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String street;
    private String city;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")   // FK column, owning side
    private Customer customer;
}
```

Points to mention: `mappedBy` marks `Customer` as the **inverse** side (no extra join table), the FK lives in the `address.customer_id` column, `cascade = ALL` + `orphanRemoval` propagate persistence/removal, and `LAZY` fetch avoids loading all addresses unless needed. Always keep both sides synchronized via a helper method.

---

## 5. Resilience & API Gateway

### How would you implement Retry and Circuit Breaker in an API Gateway?

Use a resilience library — **Resilience4j** (the modern standard, replacing Netflix Hystrix) — or the gateway's built-in filters (Spring Cloud Gateway supports Resilience4j filters).

**Retry**: automatically re-attempt a failed call a bounded number of times, ideally only for **transient/idempotent** failures (timeouts, 503), with **exponential backoff + jitter** to avoid thundering herds. Never blindly retry non-idempotent writes.

**Circuit Breaker**: wraps a downstream dependency and tracks its failure rate. It has three states:
- **CLOSED** — calls pass through; failures are counted.
- **OPEN** — once the failure rate exceeds a threshold, the breaker "trips" and **fast-fails** all calls for a cooldown period (returning a fallback immediately instead of hammering a sick service).
- **HALF-OPEN** — after the cooldown, it lets a few trial calls through; if they succeed it closes, if they fail it re-opens.

This prevents **cascading failures** and gives the downstream time to recover.

```java
@CircuitBreaker(name = "paymentSvc", fallbackMethod = "fallback")
@Retry(name = "paymentSvc")
public PaymentResult charge(Payment p) {
    return paymentClient.charge(p);
}
public PaymentResult fallback(Payment p, Throwable t) {
    return PaymentResult.queuedForLater(p);   // graceful degradation
}
```

Order matters: typically **Retry wraps CircuitBreaker** (or configure so the breaker sees final outcomes). Combine with **timeouts** so a slow call doesn't hang, and **bulkheads** to isolate resources.

### How do Retry, Circuit Breaker, Timeout, Bulkhead, and Fallback work together?

They're complementary resilience patterns; together they keep one failing dependency from taking down the whole system:

- **Timeout** — cap how long you wait for a response so a slow dependency doesn't tie up threads indefinitely. The first line of defense.
- **Retry** — recover from *transient* blips by re-attempting (with backoff), for idempotent operations.
- **Circuit Breaker** — stop calling a dependency that's *consistently* failing, fast-failing instead to let it recover and to protect your own resources.
- **Bulkhead** — isolate resources (separate thread pools / concurrency limits per dependency) so that saturation of one downstream can't exhaust all threads and starve unrelated calls — like watertight compartments in a ship.
- **Fallback** — provide a graceful degraded response when the call ultimately fails (cached value, default, "try again later", queue for async processing) instead of surfacing an error.

Typical composition for a single outbound call: **Bulkhead → CircuitBreaker → Retry → Timeout → the call**, with a **Fallback** wrapping the whole thing. In Resilience4j you literally decorate a supplier with these in order. The mental model: Timeout bounds latency, Retry handles transient faults, Circuit Breaker handles sustained faults, Bulkhead contains blast radius, Fallback preserves user experience.

---

## 6. Kafka

### What is a partition in Kafka? What does 3 partitions mean?

A **topic** is a logical stream of messages; Kafka splits each topic into **partitions**, which are the unit of **parallelism, ordering, and storage**. Each partition is an **ordered, append-only, immutable log**; messages within a partition have monotonically increasing **offsets**. A topic with **3 partitions** means the topic's data is spread across 3 such logs (often on different brokers), so up to 3 consumers in a group can read in parallel and you get 3× the throughput. **Ordering is guaranteed only within a partition**, not across the whole topic.

### A publisher publishes a message to a 3-partition topic — does it go to all three or only one?

**Only one** partition. Each message lands in exactly one partition, chosen by the producer's **partitioner**:
- If a **key** is provided → `hash(key) % numPartitions` → all messages with the same key go to the same partition (preserving per-key ordering).
- If **no key** → distributed across partitions (round-robin / sticky partitioning).
- You can also specify the partition explicitly.

A message is never duplicated across partitions. (Duplication across *consumers* is a separate matter — see consumer groups below.)

### How does Kafka consumer partition assignment work? / How are partitions assigned among consumers?

Consumers that share a **`group.id`** form a **consumer group**, and Kafka guarantees each partition is consumed by **exactly one consumer within that group**. When consumers join/leave, a **rebalance** occurs and the group coordinator (a broker) assigns partitions using a **partition assignor strategy**: `RangeAssignor`, `RoundRobinAssignor`, `StickyAssignor`, or the modern **`CooperativeStickyAssignor`** (which minimizes "stop-the-world" rebalances by only moving the partitions that need to move).

So with 3 partitions and:
- **1 consumer** in the group → it gets **all 3** partitions.
- **3 consumers** → each gets **1** partition (ideal, full parallelism).
- **More consumers than partitions** → the extras sit **idle** with no partition. Partitions are the ceiling on group parallelism.

### If one consumer (one pod) in a group listens to a 3-partition topic, will it read one or all three?

**All three.** Since it's the only member of the group, the coordinator assigns every partition to it, and it reads from all three (sequentially/interleaved). Add more consumers to the same group to distribute the partitions.

### If two different applications consume the same topic and one throws exceptions, will it impact the other?

**No.** Two *different applications* use **different `group.id`s**, so Kafka maintains **separate offsets** for each group and delivers a **full independent copy** of the stream to each. One application crashing, lagging, or throwing exceptions does not affect the other's consumption or offsets — this is exactly Kafka's pub/sub fan-out strength. (If they were in the *same* group, they'd split the partitions and a failure would trigger a rebalance.)

### How are offsets managed / stored in Kafka?

An **offset** is the position of a message within a partition. Consumers track "how far I've read" by **committing** offsets. Committed offsets are stored in an internal compacted Kafka topic called **`__consumer_offsets`**, keyed by `(group.id, topic, partition)` — not in ZooKeeper (that's the old pre-0.9 behavior). On restart or rebalance, a consumer resumes from its last committed offset.

Commit modes:
- **Auto-commit** (`enable.auto.commit=true`) — periodic, convenient, but can cause message loss or duplicates around failures.
- **Manual commit** — `commitSync()` / `commitAsync()`, committing **after** successful processing for at-least-once semantics. This is the recommended pattern for reliability.

### If two applications listen to the same topic, do they have the same or different offsets? What about within the same group? And across partitions?

- **Two different applications (different groups)**: **independent offsets** — each group tracks its own position, and both receive all messages.
- **Within the same consumer group**: there's **one committed offset per partition** for the group, and each partition is owned by one consumer — so members don't "share" an offset for the same partition; each partition has its own group-level offset. Different members simply own different partitions.
- **Across partitions**: offsets are **per-partition** and completely independent — offset 100 in partition 0 has no relationship to offset 100 in partition 1. There is no global offset across a topic.

---

## 7. SQL & Databases

### What is a composite index and how does it work?

A composite (multi-column) index is an index built on **two or more columns together**, e.g. `CREATE INDEX idx ON orders(customer_id, status, created_at)`. Internally it's usually a **B-tree** whose keys are the **concatenation** of the indexed columns in the declared order. Entries are sorted first by the first column, then by the second within equal first values, and so on — like a phone book sorted by (last name, first name).

### How do composite indexes follow the leftmost-prefix rule?

Because the index is sorted by columns **left to right**, it can be used efficiently only when the query constrains a **contiguous prefix starting from the leftmost column**. For an index on `(a, b, c)`:
- `WHERE a = ?` ✅ uses index
- `WHERE a = ? AND b = ?` ✅
- `WHERE a = ? AND b = ? AND c = ?` ✅ (full)
- `WHERE a = ? AND c = ?` ⚠️ uses only the `a` part (can't skip `b` to seek on `c`)
- `WHERE b = ?` ❌ can't use the index for seeking (no leftmost `a`)
- `WHERE b = ? AND c = ?` ❌

This is the **leftmost-prefix rule**: you must include the leading column(s) contiguously. It's why **column order matters** — put the most selective / most-frequently-filtered columns, and equality-filtered columns before range-filtered ones.

### How do composite indexes improve query performance?

- **Fewer rows scanned**: the DB seeks directly to matching entries instead of a full table scan.
- **Multi-column filtering in one structure**: one index satisfies `WHERE a=? AND b=?` without intersecting two single-column indexes.
- **Covering index**: if the index contains *all* columns the query needs (SELECT + WHERE + ORDER BY), the DB answers entirely from the index — an **index-only scan**, skipping the table (heap) lookup entirely.
- **Sorting/grouping for free**: because entries are pre-sorted, `ORDER BY a, b` or range scans on the trailing range column avoid a separate sort.

### When will the optimizer use a composite index, and when won't it?

**Will use it** when:
- The query filters on a **leftmost prefix** of the index columns.
- Equality on leading columns, optionally a range on the next column.
- It's **selective enough** that the index seek + heap fetch is cheaper than a scan.
- It can be a **covering** index for the query.
- Used for `ORDER BY`/`GROUP BY` matching the index order.

**Won't use it** when:
- The `WHERE` doesn't include the leftmost column (leftmost-prefix violated).
- A **function/expression wraps the column** (`WHERE lower(a) = ?`, `WHERE a + 1 = ?`) — unless there's a matching functional index.
- **Implicit type mismatch** on the column (e.g. comparing a string column to a number).
- **Leading wildcard** `LIKE '%abc'` (a trailing wildcard `'abc%'` can use it).
- The predicate is **not selective** — if it matches a large fraction of rows, a **sequential scan** is cheaper, so the optimizer skips the index.
- Small table where a full scan is trivially cheap.
- Stale statistics causing a bad estimate (fix with `ANALYZE`).

### What is the difference between the WHERE clause and the HAVING clause? Can they be used together?

- **`WHERE`** filters **individual rows before grouping/aggregation**. It cannot reference aggregate functions.
- **`HAVING`** filters **groups after `GROUP BY` and aggregation**. It's used to filter on aggregate results (`HAVING COUNT(*) > 5`).

**Yes, they're commonly used together**: `WHERE` first narrows the rows, then `GROUP BY` groups them, then `HAVING` filters the groups. Doing as much filtering as possible in `WHERE` (before aggregation) is more efficient because fewer rows are grouped.

```sql
SELECT dept, AVG(salary) AS avg_sal
FROM   employees
WHERE  active = true          -- row filter, before grouping
GROUP BY dept
HAVING AVG(salary) > 50000;   -- group filter, after aggregation
```

Logical order of evaluation: `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`.

### What are the ACID properties? / How do we manage ACID properties of a transaction?

ACID describes the guarantees of a reliable transaction:
- **Atomicity** — all statements in a transaction succeed or none do; on failure everything rolls back. There's no partial application.
- **Consistency** — a transaction moves the database from one **valid state to another**, respecting all constraints, cascades, and triggers.
- **Isolation** — concurrent transactions don't interfere; the result is as if they ran in some serial order (tunable via isolation levels: Read Uncommitted, Read Committed, Repeatable Read, Serializable — trading off phenomena like dirty/non-repeatable/phantom reads against concurrency).
- **Durability** — once committed, changes survive crashes/power loss (persisted to disk, typically via a write-ahead log).

How they're managed internally: databases use a **Write-Ahead Log (WAL/redo log)** for durability and crash recovery, **undo logs** for rollback (atomicity), **locking or MVCC** for isolation, and **constraint enforcement** for consistency. In application code you manage ACID via **transaction boundaries** — `@Transactional` in Spring, `BEGIN/COMMIT/ROLLBACK` — and choose an appropriate **isolation level** for the concurrency/correctness trade-off. Across microservices where a single DB transaction isn't possible, you approximate atomicity with the **Saga pattern** and **outbox pattern** (eventual consistency + compensating actions).

### PostgreSQL query optimization (for a senior/7+ years developer)

A structured approach an interviewer expects:

1. **Measure first with `EXPLAIN (ANALYZE, BUFFERS)`** — read the actual plan, look for Seq Scans on big tables, bad row estimates (planned vs actual rows), expensive sorts, nested-loop joins over large sets, and high buffer reads.
2. **Indexing** — add appropriate B-tree indexes on filter/join/sort columns; use **composite indexes** honoring the leftmost-prefix rule; use **covering indexes** (`INCLUDE` columns) for index-only scans; **partial indexes** (`WHERE status='ACTIVE'`) for skewed data; **GIN** indexes for JSONB/full-text/array columns; **expression indexes** to match functional predicates.
3. **Keep statistics fresh** — `ANALYZE` / autovacuum so the planner estimates correctly; raise `default_statistics_target` for skewed columns; use **extended statistics** (`CREATE STATISTICS`) for correlated columns.
4. **Write sargable queries** — avoid wrapping indexed columns in functions, avoid implicit type casts, avoid leading-wildcard `LIKE`, avoid `SELECT *`.
5. **Joins & set ops** — ensure join keys are indexed; prefer `EXISTS`/`JOIN` over correlated subqueries where appropriate; watch for accidental cartesian products.
6. **Pagination** — replace deep `OFFSET` with **keyset (seek) pagination** (`WHERE id > :last ORDER BY id LIMIT n`).
7. **Partitioning** — range/list **partition** very large tables so queries prune to relevant partitions; helps vacuum and archival.
8. **VACUUM / bloat** — control dead-tuple bloat from updates/deletes; tune autovacuum; use `pg_stat_user_tables`/`pgstattuple`.
9. **Connection & config** — use a pooler (**PgBouncer**), size `work_mem` for big sorts/hashes, `shared_buffers`, and `effective_cache_size` appropriately.
10. **Reduce round-trips / caching** — batch queries, use materialized views for expensive aggregates, cache hot read paths (e.g. Redis).
11. **Observability** — `pg_stat_statements` to find the top time-consuming queries and optimize by impact, not guesswork.

---

## 8. RAG & Generative AI

### Explain the architecture / flow of RAG (Retrieval-Augmented Generation)

RAG grounds an LLM's answers in **your own data** so it can answer domain/private questions and reduce hallucination, without retraining the model. There are two phases:

**A. Indexing (offline / ingestion):**
1. **Load** source documents (PDFs, wikis, DB records, tickets).
2. **Chunk** them into passages (e.g. 200–1000 tokens) with some overlap, respecting structure (paragraphs, headings).
3. **Embed** each chunk with an embedding model → a dense vector capturing semantic meaning.
4. **Store** the vectors (plus the text and metadata) in a **vector database**.

**B. Retrieval + Generation (online / query time):**
1. **Embed the user's question** with the same embedding model.
2. **Similarity search** in the vector DB (e.g. top-k nearest neighbors by cosine similarity) to fetch the most relevant chunks — optionally **hybrid search** (vector + keyword/BM25) and a **re-ranker** for precision.
3. **Augment the prompt**: build a prompt that inserts the retrieved chunks as **context**, plus the question and instructions ("answer only from the context; cite sources").
4. **Generate**: the LLM produces an answer grounded in that context.
5. Optionally return **citations** to the source chunks.

The mental model: *retrieval brings the relevant facts to the model at query time; the model just reasons over and phrases them.*

### Which Java library did you use for RAG? / Which Java libraries are common?

The two dominant options in the Java ecosystem are:
- **Spring AI** — Spring's abstraction over chat models, embedding models, vector stores, and RAG plumbing (document readers, splitters, `VectorStore`, advisors/retrievers). Best fit if you're already in the Spring Boot ecosystem — consistent config, DI, autoconfiguration.
- **LangChain4j** — a Java port of the LangChain idea: chains, embeddings stores, retrievers, tools/agents, memory. Framework-agnostic and very feature-rich for LLM orchestration.

### Why would you choose Spring AI instead of LangChain4j?

Choose **Spring AI** when your application is **already Spring Boot-based** and you value **native integration**: autoconfiguration, `application.yml` config, Spring-style beans/dependency injection, portable abstractions over providers (OpenAI, Azure, Bedrock, Ollama) and vector stores, observability via Micrometer, and a consistent programming model with the rest of your stack. It keeps the LLM layer idiomatic to Spring and reduces glue code.

Choose **LangChain4j** when you want a **richer, provider-agnostic orchestration toolkit** (more built-in chains, agents, tools, memory patterns) or you're **not** on Spring. Both are valid; the honest answer is "Spring AI for tight Spring integration and simplicity; LangChain4j for breadth of LLM-orchestration features." (Spring AI can also be layered on LangChain4j-style needs.)

### What are vector databases? How do they work?

A **vector database** stores and searches **high-dimensional embedding vectors** and finds the ones **most similar** to a query vector — i.e. **nearest-neighbor search** by semantic distance (cosine similarity / dot product / Euclidean), rather than exact keyword matching. This is what powers semantic search and RAG retrieval.

How it works: exact nearest-neighbor over millions of vectors is too slow, so vector DBs use **Approximate Nearest Neighbor (ANN)** indexes that trade a tiny bit of recall for huge speed. Common index types:
- **HNSW** (Hierarchical Navigable Small World) — a layered graph you navigate greedily; excellent recall/latency, the most popular.
- **IVF** (inverted file) — cluster vectors, search only the nearest clusters.
- **PQ** (product quantization) — compress vectors to save memory.

They also store **metadata** alongside each vector so you can **filter** (e.g. by tenant, date, document type) during search, and support upserts/deletes. Examples: **pgvector** (Postgres extension), **Pinecone, Weaviate, Milvus, Qdrant, Chroma**, and Elastic/OpenSearch kNN. Query flow: embed query → ANN search top-k → return the chunks + metadata + similarity scores.

### How do you programmatically construct system prompts in Java for LLM APIs?

Build the messages list with a **system message** (role/instructions/guardrails), then user (and prior assistant) messages. Keep the prompt as a **template** with placeholders you fill at runtime — never string-concatenate untrusted input carelessly.

With **Spring AI**:
```java
var system = new SystemPromptTemplate("""
        You are a support assistant for {product}.
        Answer ONLY using the provided context. If unsure, say you don't know.
        Context:
        {context}
        """).createMessage(Map.of("product", product, "context", retrieved));

var user = new UserMessage(question);
Prompt prompt = new Prompt(List.of(system, user));
String answer = chatModel.call(prompt).getResult().getOutput().getContent();
```

Best practices to mention: keep the **system prompt** for stable instructions/persona/guardrails and put dynamic data in it via templating; inject retrieved RAG context clearly delimited; instruct the model to answer only from context and to cite; externalize templates (files/resources) so prompts are versioned and testable; and **sanitize** user input to mitigate prompt injection.

### How do you check if RAG answers are correct? And how at 50 million documents scale?

Evaluate along two axes — **retrieval quality** and **generation quality**:

Retrieval metrics: **recall@k / precision@k / MRR / nDCG** — did the right chunks get retrieved? Build a labeled set of question→relevant-doc pairs and measure.

Generation metrics: **faithfulness/groundedness** (is every claim supported by the retrieved context?), **answer relevance**, and **correctness** vs a reference answer. Frameworks like **RAGAS**, **TruLens**, or **DeepEval** automate these, often using an **LLM-as-a-judge** to score faithfulness and relevance. Add **citations** so answers are traceable back to sources, and track **hallucination rate**.

At **50 million documents** you cannot human-label everything, so you scale evaluation:
- **Sample intelligently** — evaluate on a **representative, stratified sample** and on the **high-traffic / high-risk** queries, not the whole corpus.
- **Automated LLM-as-judge** faithfulness scoring on production traffic (offline, on sampled logs).
- **Golden/regression test set** — a curated set of Q&A pairs run in CI so quality doesn't regress after changes.
- **Groundedness checks** — verify each answer's claims are entailed by the retrieved chunks; flag unsupported claims.
- **Human-in-the-loop / feedback signals** — thumbs up/down, correction capture, expert review of a sample.
- **Guardrails at answer time** — require citations, refuse when retrieval confidence/similarity is low, and detect "context doesn't contain the answer."
- **Monitoring** — track retrieval scores, answer-length anomalies, refusal rate, and user feedback over time. The point: correctness at scale is a **continuous, sampled, automated** evaluation process, not a one-time manual check.

### Is RAG compliant to use?

It can be, but compliance depends on **how you handle the data**, not RAG itself. Key considerations: **data residency & privacy** (does customer/PII data leave your boundary? use a provider/region and contracts — e.g. enterprise agreements with no-training clauses — or self-host models); **access control** (retrieval must respect per-user/tenant permissions so RAG never surfaces documents a user isn't allowed to see — enforce row/document-level security in the vector store filters); **PII handling** (redact/mask sensitive fields before indexing where possible); **auditability** (log sources and answers, provide citations); **retention & right-to-erasure** (be able to delete a user's documents and their embeddings — GDPR); and **model provider terms** (ensure prompts/data aren't used for training). So the answer: RAG is compliant when you enforce access control on retrieval, keep data in an approved boundary, redact PII, honor retention/erasure, and use providers under appropriate data-protection terms.

### Where would you use RAG and how?

Typical enterprise use cases: an **internal knowledge assistant** over wikis/Confluence/policies; **customer support** answering from product docs and past tickets; **developer assistant** over API docs and runbooks; **document Q&A** over contracts/reports; **compliance/policy lookup**. The "how" is the pipeline above: ingest and chunk the domain docs, embed and store in a vector DB with per-tenant metadata, retrieve top-k relevant chunks at query time filtered by the user's permissions, and have the LLM answer grounded in those chunks with citations.

### Agentic AI vs Gen AI tools; using Codex; and "won't AI-generated code create noise?"

**Gen AI tools** generate content (text/code/images) in response to a prompt — a single-shot assistant. **Agentic AI** goes further: it **plans, uses tools, takes multi-step actions, and iterates toward a goal** (calling APIs, running code, reading files, self-correcting) with some autonomy. Coding assistants (Codex-style / Copilot / Claude Code) sit on this spectrum — from autocomplete to agents that edit and test across a repo.

**Using such tools for coding**: they accelerate boilerplate, tests, refactors, and exploration. The valid concern that "*this will create noise / generate lots of low-quality output*" is real, and the professional answer is about **guardrails**: treat AI output as a **draft, not a commit** — every change goes through **code review, CI, tests, linting, and static analysis**; scope prompts tightly; keep a human accountable for correctness and security; and use it where it demonstrably saves time rather than blanket-generating code. In other words, AI raises productivity but doesn't remove the engineering discipline (review, testing, ownership) that keeps quality high.

---

## 9. DSA / Coding

### Climbing Stairs — N stairs, climb 1 or 2 at a time, count the ways

**Intuition**: to reach step `n`, your last move was either a **1-step from `n-1`** or a **2-step from `n-2`**. So the number of ways to reach `n` is `ways(n-1) + ways(n-2)` — it's the **Fibonacci** recurrence. Base cases: `ways(1) = 1`, `ways(2) = 2` (either 1+1 or 2).

```java
// O(n) time, O(1) space
int climbStairs(int n) {
    if (n <= 2) return n;
    int prev2 = 1, prev1 = 2;      // ways(1)=1, ways(2)=2
    for (int i = 3; i <= n; i++) {
        int cur = prev1 + prev2;
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}
```

**Why O(n) even though we iterate?** The loop runs `n-2` times and each iteration is O(1) work, so total is O(n). It's linear, not exponential, because we **reuse** already-computed results (bottom-up DP) instead of the naive recursion `f(n-1)+f(n-2)` which recomputes overlapping subproblems and is O(2ⁿ). Space is O(1) because we only keep the last two values.

**Dry run for n = 4:**
- start: prev2 = 1 (ways to reach step 1), prev1 = 2 (ways to reach step 2)
- i=3: cur = 2 + 1 = 3 → prev2 = 2, prev1 = 3   (3 ways to reach step 3)
- i=4: cur = 3 + 2 = 5 → prev2 = 3, prev1 = 5   (5 ways to reach step 4)
- return 5.

The 5 ways for n=4: `1+1+1+1`, `1+1+2`, `1+2+1`, `2+1+1`, `2+2`.

### Coin denominations {1,2,5,10,50,100} — minimum coins for an amount

**Two answers depending on the coin system.**

For this particular set, the coins are **canonical** (each larger coin is a multiple/covers combinations of smaller ones), so a **greedy** approach works and is optimal: repeatedly take the largest coin ≤ remaining amount.

```java
// Greedy — works for canonical systems like {1,2,5,10,50,100}
int minCoinsGreedy(int amount, int[] coins) {   // coins sorted descending
    int count = 0;
    for (int c : coins) {
        count += amount / c;   // take as many of this coin as possible
        amount %= c;
    }
    return count;              // amount hits 0 because a 1-coin exists
}
// amount=187 -> 100(1) +50(1) +10(3) +5(1) +2(1) = 1+1+3+1+1 = 7 coins
```

**Important caveat to state in the interview**: greedy is **not** correct for arbitrary coin sets (e.g. `{1,3,4}` for amount 6: greedy gives 4+1+1=3 coins, optimal is 3+3=2). The **general, always-correct** solution is **Dynamic Programming** (unbounded knapsack):

```java
// DP — correct for ANY coin set. O(amount * coins) time, O(amount) space
int minCoinsDP(int amount, int[] coins) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);           // "infinity"
    dp[0] = 0;
    for (int a = 1; a <= amount; a++)
        for (int c : coins)
            if (c <= a) dp[a] = Math.min(dp[a], dp[a - c] + 1);
    return dp[amount] > amount ? -1 : dp[amount];   // -1 if unreachable
}
```

`dp[a]` = min coins to make amount `a`; for each amount try every coin and take the best. Mention both: greedy for the given canonical set (O(number of coins)), DP as the robust general answer.

### Combination Sum — all combinations in an array summing to a target

**Intuition**: this is **backtracking**. At each index you decide to *include* the current number (staying on the same index if reuse is allowed, or moving on if not) or *skip* it, building up a running combination; when the running sum hits the target you record the combination, and if it overshoots you prune.

Version where each number may be reused unlimited times (classic Combination Sum):

```java
List<List<Integer>> combinationSum(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(candidates);                 // enables pruning
    backtrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}

void backtrack(int[] c, int remain, int start,
               List<Integer> path, List<List<Integer>> result) {
    if (remain == 0) {                       // exact match -> record
        result.add(new ArrayList<>(path));
        return;
    }
    for (int i = start; i < c.length; i++) {
        if (c[i] > remain) break;            // sorted -> no point continuing
        path.add(c[i]);
        backtrack(c, remain - c[i], i, path, result);  // i (not i+1) => reuse allowed
        path.remove(path.size() - 1);        // undo choice (backtrack)
    }
}
```

Key details to explain: passing `start = i` lets a number be reused; use `i + 1` if each element can be used at most once. Sorting + the `break` prunes branches that can't succeed. The `path.remove(...)` at the end is the **backtracking step** — undo the last choice before trying the next. Complexity is exponential in the worst case (it's an enumeration problem): roughly O(N^(target/min)) states, O(target) to copy each solution.

**Dry run** for `candidates = {2,3,6,7}`, `target = 7`: paths explored yield `[2,2,3]` (2+2+3=7) and `[7]`. Result: `[[2,2,3], [7]]`.

### Find the employee with the maximum salary in each department using Java Streams

Given `Employee(empId, employeeName, salary, dept)`:

```java
Map<String, Optional<Employee>> topByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))));

// Cleaner: unwrap Optional and get just the max Employee per dept
Map<String, Employee> result = employees.stream()
    .collect(Collectors.toMap(
        Employee::getDept,
        Function.identity(),
        (a, b) -> a.getSalary() >= b.getSalary() ? a : b));  // merge: keep higher
```

Explanation: `groupingBy(dept, maxBy(bySalary))` groups employees by department and reduces each group to the max-salary employee (wrapped in `Optional` because a group *could* be empty in general). The `toMap` variant with a **merge function** is neat: it keeps, for each department key, whichever employee has the higher salary. To also print highest salary per department simply:

```java
result.forEach((dept, e) ->
    System.out.println(dept + " -> " + e.getEmployeeName() + " : " + e.getSalary()));
```

### DTO / Mapper design — transform a flat students_marksheet into nested JSON

Suppose the table has rows like `(year, studentName, subject, marks)` and you need nested output grouped by **year → students → subjects/marks**. The idea: define DTOs that mirror the **output** shape, then group the flat rows.

```java
// Output DTOs (nested shape)
record SubjectMark(String subject, int marks) {}
record StudentDTO(String studentName, List<SubjectMark> subjects) {}
record YearDTO(String year, List<StudentDTO> students) {}

// Mapper: flat rows -> nested structure
List<YearDTO> toNested(List<MarksRow> rows) {
    // group by year, then by student, collecting subject/marks
    Map<String, Map<String, List<SubjectMark>>> grouped = rows.stream()
        .collect(Collectors.groupingBy(
            MarksRow::year,
            Collectors.groupingBy(
                MarksRow::studentName,
                Collectors.mapping(
                    r -> new SubjectMark(r.subject(), r.marks()),
                    Collectors.toList()))));

    return grouped.entrySet().stream()
        .map(yearEntry -> new YearDTO(
            yearEntry.getKey(),
            yearEntry.getValue().entrySet().stream()
                .map(stuEntry -> new StudentDTO(stuEntry.getKey(), stuEntry.getValue()))
                .toList()))
        .toList();
}
```

How to explain the "multiple students per year" nesting: the **outer `groupingBy(year)`** produces one entry per year; the **nested `groupingBy(studentName)`** produces multiple students within each year; the **`mapping(... toList())`** collects each student's list of subject/marks. Then you convert the nested maps into your DTO records. Because Year 1 and Year 2 each contain many students and each student many subjects, the two levels of `groupingBy` naturally build the two levels of nesting.

### Generate the required JSON output with a SQL query (Postgres)

Postgres can build nested JSON directly with `json_agg` / `json_build_object`:

```sql
SELECT json_agg(
         json_build_object(
           'year', y.year,
           'students', y.students
         )
       ) AS result
FROM (
  SELECT year,
         json_agg(
           json_build_object('studentName', studentName, 'subjects', subjects)
         ) AS students
  FROM (
    SELECT year, studentName,
           json_agg(json_build_object('subject', subject, 'marks', marks)) AS subjects
    FROM students_marksheet
    GROUP BY year, studentName
  ) s
  GROUP BY year
) y;
```

The inner query groups by `(year, student)` to build each student's subjects array; the middle groups by `year` to build the students array; the outer aggregates years into the final JSON array. In an interview, mention you *can* do it in SQL (as above) but often it's cleaner and more testable to do the nesting in the Java mapper layer as shown previously.

---

## 10. System Design

### Monolith where Task1, Task2, Task3 are independent and Task4 aggregates — how would you improve it, and how many microservices?

The problem: Tasks 1–3 are **independent** but run **sequentially** in the monolith, and Task 4 needs all three results. The improvement is to run 1–3 **in parallel** and then aggregate.

**Within a single service (simplest win):** run Task1/2/3 concurrently and join:
```java
CompletableFuture<R1> f1 = CompletableFuture.supplyAsync(this::task1, pool);
CompletableFuture<R2> f2 = CompletableFuture.supplyAsync(this::task2, pool);
CompletableFuture<R3> f3 = CompletableFuture.supplyAsync(this::task3, pool);
CompletableFuture.allOf(f1, f2, f3).join();
Result r4 = task4(f1.join(), f2.join(), f3.join());   // aggregate
```
Total latency drops from `t1+t2+t3+t4` to `max(t1,t2,t3)+t4`. On Java 21 you could use **virtual threads / structured concurrency** for the same effect with simpler code.

**As microservices:** a reasonable decomposition is **4 services** — one each for Task1, Task2, Task3 (independently deployable/scalable) and an **aggregator** (Task4) that fans out to the three in parallel and combines results (scatter-gather pattern). But the honest senior answer is: **don't split just because you can.** Only break into separate services if the tasks have **different scaling profiles, ownership, or deployment cadence**; otherwise parallelizing inside one service avoids network latency, distributed failures, and operational overhead. If you do split, use async messaging or parallel calls, add timeouts/circuit breakers on the aggregator, and consider that Task4 becomes a **backend-for-frontend / API composition** layer. So: "4 microservices if the domains genuinely warrant independence; otherwise 1 service with parallel execution."

### Explain a real-time deadlock scenario

A **deadlock** is when two (or more) threads each hold a resource the other needs and both wait forever. Real example — a **money transfer** between two accounts that locks both rows:

- Thread A: transfer from Account 1 → Account 2, locks **row 1**, then tries to lock **row 2**.
- Thread B: transfer from Account 2 → Account 1, locks **row 2**, then tries to lock **row 1**.
- A waits for B to release row 2; B waits for A to release row 1 → **deadlock**.

The four Coffman conditions must all hold: mutual exclusion, hold-and-wait, no preemption, and circular wait. **Fixes**: impose a **global lock ordering** (always lock the lower account id first — breaks circular wait); use **`tryLock` with timeout** and back off/retry; keep transactions short and lock fewer rows; use **optimistic locking** to avoid holding locks; and rely on the DB's deadlock detector (Postgres/MySQL detect and kill one transaction with a deadlock error, which the app retries).

### How do you ensure data consistency across microservices?

You can't use a single ACID transaction across services/databases, so you use patterns for **eventual consistency**:

- **Saga pattern** — model a business transaction as a sequence of local transactions, each publishing an event that triggers the next; if a step fails, run **compensating transactions** to undo prior steps. Two styles: **choreography** (services react to each other's events) and **orchestration** (a central coordinator drives the steps).
- **Transactional Outbox** — write the business change and an "event to publish" row in the **same local DB transaction**, then a relay/CDC (e.g. Debezium) publishes the event to Kafka. This guarantees the event is sent **if and only if** the data was committed (solves dual-write inconsistency).
- **Idempotency** — consumers must handle duplicate events safely (idempotency keys) since delivery is at-least-once.
- **Event-driven architecture** — services own their data and sync via events rather than shared DBs.
- **Distributed transactions (2PC)** — technically possible but avoided in practice due to poor scalability and availability.

The mental model: prefer **eventual consistency + sagas + outbox + idempotency** over trying to force strong global transactions.

### How do you implement logging in a microservices architecture?

Use **centralized, structured, correlated** logging:
- **Structured logs** (JSON) with consistent fields (timestamp, level, service, traceId) so they're machine-parseable.
- **Correlation / trace IDs** — propagate a `traceId`/`spanId` across service calls (via headers) so you can follow one request across services. **Spring Cloud Sleuth / Micrometer Tracing + OpenTelemetry** do this automatically and inject IDs into the MDC.
- **Centralized aggregation** — ship logs to a central store: **ELK/EFK (Elasticsearch + Logstash/Fluentd + Kibana)**, **Loki + Grafana**, or a cloud service (CloudWatch, Datadog). Don't rely on per-container files.
- **Distributed tracing** — **OpenTelemetry → Jaeger/Zipkin/Tempo** to visualize the full call path and latency per hop.
- **Consistent levels & no sensitive data** — standard log levels, and never log secrets/PII.
- **Metrics + logs + traces** together (the "three pillars of observability").

### How do you implement authentication and authorization in microservices?

- **Centralized authentication** via an **Identity Provider / Auth Server** (Keycloak, Okta, Auth0) using **OAuth2 + OIDC**. The user authenticates once and gets a signed **JWT** access token.
- **Stateless token validation** — each service (resource server) validates the JWT **locally** by verifying the signature against the IdP's public keys (JWKS) and checking claims (`exp`, `aud`, `scope`, roles). No DB round-trip per request.
- **API Gateway** — often does coarse-grained auth (token presence/validity, rate limiting) at the edge and forwards the token/identity downstream.
- **Authorization** — role/scope/claim-based checks in each service (`@PreAuthorize("hasRole('ADMIN')")`), and for fine-grained needs, a policy engine (OPA) or per-resource ACLs.
- **Service-to-service auth** — **mTLS** and/or the **client-credentials** OAuth grant for machine-to-machine calls; a service mesh (Istio) can enforce mTLS.
- **Token propagation** — pass the user's token (or a downscoped token) along the call chain so authorization context is preserved.

### How do you identify a CPU spike?

- **Metrics/APM first** — dashboards (Prometheus/Grafana, Datadog, New Relic) show per-pod CPU; identify *which* service/instance and *when* it spiked and correlate with deploys or traffic.
- **On the host/pod** — `top`/`htop`, `pidstat`, container CPU throttling metrics.
- **Find the hot thread in the JVM** — `top -H -p <pid>` to get the OS thread eating CPU, convert its TID to hex, and match it in a **thread dump** (`jstack`) to see the exact stack. Repeated thread dumps reveal what's spinning.
- **Profilers** — **async-profiler**, **Java Flight Recorder (JFR)** + Mission Control give flame graphs pinpointing the hot methods with low overhead.
- **Common culprits** — busy-wait/spin loops, inefficient algorithms in a hot path, excessive **GC** (check GC logs — high CPU in GC threads means allocation pressure), regex catastrophic backtracking, serialization, or a sudden traffic surge.

The method: **observe (APM) → localize (which pod/thread) → profile (JFR/async-profiler) → fix the hot path or GC cause.**

### Rate limiting use cases

Rate limiting caps how many requests a client can make in a time window to **protect resources, ensure fairness, prevent abuse, and control cost**. Use cases: protecting an API from **DDoS/brute-force** (e.g. login attempts), enforcing **per-tenant/plan quotas** (free vs paid tiers), shielding a **downstream dependency** from overload, and **cost control** for expensive operations (LLM calls, exports). Algorithms: **Token Bucket** (allows bursts up to bucket size, most common), **Leaky Bucket** (smooths to a constant rate), **Fixed Window** (simple but boundary bursts), **Sliding Window log/counter** (accurate, more memory). Implement at the **API Gateway** (edge) and/or per-service, typically backed by **Redis** for a shared counter across instances (e.g. Bucket4j + Redis, or the gateway's built-in `RequestRateLimiter`). Respond with **HTTP 429** and a `Retry-After` header.

### Design patterns used in practice / use case of Factory pattern

Common ones in real backend code: **Singleton** (Spring beans are effectively singletons), **Factory / Factory Method** and **Abstract Factory** (object creation), **Builder** (constructing complex objects, e.g. `Lombok @Builder`, immutable DTOs), **Strategy** (interchangeable algorithms — e.g. different payment/discount strategies chosen at runtime), **Template Method** (Spring's `JdbcTemplate`/`RestTemplate`), **Observer** (event listeners / `ApplicationEvent`), **Proxy** (Spring AOP, `@Transactional`), **Decorator** (wrapping behavior), **Adapter** (bridging incompatible APIs).

**Factory use case**: when the exact class to instantiate is decided at **runtime** based on input, and you want to **decouple** the caller from concrete classes. Example — a `PaymentProcessorFactory` returns a `CreditCardProcessor`, `UpiProcessor`, or `PaypalProcessor` based on the payment type, so the calling code just asks the factory and codes against the `PaymentProcessor` interface:

```java
PaymentProcessor p = PaymentProcessorFactory.create(paymentType); // hides concrete class
p.process(payment);
```

This centralizes creation logic, makes adding a new type a localized change, and keeps client code open/closed.

### Monitoring tools

- **Metrics**: **Prometheus + Grafana** (with **Micrometer** in Spring Boot exposing `/actuator/prometheus`), or Datadog/New Relic/AppDynamics/Dynatrace.
- **Logs**: **ELK/EFK stack** (Elasticsearch, Logstash/Fluentd, Kibana), **Grafana Loki**, Splunk, CloudWatch.
- **Tracing**: **OpenTelemetry** with **Jaeger / Zipkin / Grafana Tempo**.
- **APM**: New Relic, Datadog APM, Dynatrace, AppDynamics, Elastic APM.
- **Alerting/uptime**: Alertmanager, PagerDuty/Opsgenie, Grafana alerts.
- **Health**: Spring Boot Actuator health/readiness/liveness probes for Kubernetes.

---

## 11. Miscellaneous / Production Topics

### AES encryption

**AES (Advanced Encryption Standard)** is a **symmetric-key block cipher** — the same secret key encrypts and decrypts. It operates on **128-bit blocks** with key sizes of **128, 192, or 256 bits**, using multiple rounds of substitution-permutation (SubBytes, ShiftRows, MixColumns, AddRoundKey). It's fast, hardware-accelerated (AES-NI), and the current standard for data-at-rest and in-transit symmetric encryption.

Practical points an interviewer wants: never use **ECB mode** (identical plaintext blocks produce identical ciphertext — leaks patterns). Use an **authenticated mode** like **GCM** (provides confidentiality *and* integrity/authentication) or CBC with a separate MAC. Always use a **random IV/nonce per encryption** (never reuse a nonce with GCM), and manage keys securely (a KMS/HSM, key rotation), never hardcoded. Since AES is symmetric, key distribution is the hard part — often you exchange the AES key using asymmetric crypto (RSA/ECDH), i.e. **envelope encryption**.

```java
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
byte[] iv = new byte[12]; new SecureRandom().nextBytes(iv);   // unique per message
cipher.init(Cipher.ENCRYPT_MODE, secretKey, new GCMParameterSpec(128, iv));
byte[] ciphertext = cipher.doFinal(plaintext);
```

### If Redis goes down, what happens?

Depends on **how you use Redis**:
- **As a cache (most common)**: cache reads miss → requests fall back to the **database**, so you get **higher latency and a load spike** on the DB (risk of a **thundering herd / cache stampede**), but the system still functions if the DB can absorb it. Mitigate with graceful fallback, request coalescing, circuit breakers, and DB capacity headroom.
- **As the source of truth / session store / rate limiter / distributed lock**: an outage is more serious — sessions/locks/limits become unavailable, so those features degrade or fail. Design these to **fail safe** (e.g. fail-open or fail-closed deliberately).

**Resilience measures**: run Redis in **HA** — **Redis Sentinel** (automatic failover) or **Redis Cluster** (sharding + replicas), enable **persistence (RDB/AOF)** for recovery, use **replicas** for read failover, add **client-side timeouts and circuit breakers** so a Redis hang doesn't block app threads, and make the app **degrade gracefully** (serve from DB, use local caches). The key message: never let a cache being down take the whole system down — treat it as an optimization with a fallback path.

### Explain the architecture of a rules engine

A **rules engine** externalizes business logic as **declarative rules** (`when <conditions> then <actions>`) so business logic can change **without redeploying code**. Core components:
- **Rule Repository** — where rules are authored/stored (DB, DRL files, decision tables/spreadsheets, a UI).
- **Working Memory / Facts** — the input data objects the rules evaluate against.
- **Inference Engine** — matches facts against rule conditions and decides which rules fire. Production engines like **Drools** use the **Rete algorithm** to efficiently match many rules against many facts without re-evaluating everything.
- **Agenda / Conflict Resolution** — when multiple rules match, ordering/priority (salience) decides firing order.
- **Action/Execution** — fired rules execute their consequences (mutate facts, produce decisions, trigger side effects).

Flow: facts are inserted into working memory → the engine matches them against the rule set → matched rules are placed on the agenda → they fire (possibly asserting new facts, which can trigger further matching) → results are collected. Use cases: **eligibility, pricing/discounts, fraud detection, validation, underwriting**. Benefits: business rules are **decoupled** from application code, editable by analysts, and centralized/auditable. Java implementations: **Drools** (the standard), or lightweight ones like Easy Rules; simpler needs can use a strategy/spec pattern instead. Trade-off: for a handful of rules a rules engine is overkill — introduce one when rules are **numerous, volatile, and business-owned**.

### How do we version REST APIs? (production perspective)

Covered in the Spring section — URI path versioning (`/v1`) is the pragmatic default; version only on breaking changes, keep old versions during a deprecation window, prefer additive backward-compatible changes, and communicate deprecation via headers/docs.

### Have you encountered production issues? (how to answer)

Structure the answer with a concrete example and the **STAR / incident narrative**: *Situation* (what broke, impact), *Task* (your role), *Action* (how you diagnosed — logs/metrics/traces/thread dumps — and fixed), *Result* (resolution + prevention). Strong examples to draw from: a **memory leak / `OutOfMemoryError`** found via heap dump; a **connection pool exhaustion** causing timeouts (fixed pool sizing/leak); an **N+1 query** slowing an endpoint (fixed with `JOIN FETCH`); a **Kafka consumer lag** from slow processing (scaled partitions/consumers); a **deadlock** fixed with lock ordering; a **cache stampede** when Redis restarted. Emphasize **detection (monitoring/alerting), root-cause analysis, the fix, and the preventive follow-up** (tests, alerts, runbooks, post-mortem). Interviewers care most that you diagnose methodically and prevent recurrence.

### What is APM (Application Performance Monitoring)?

APM is the practice and tooling for **monitoring the performance, availability, and health of applications in production** — end-to-end. It goes beyond infra metrics to show **application-level** behavior: request **throughput, latency (p50/p95/p99), error rates**, **distributed traces** across services, **slow database queries and external calls**, **JVM/GC and memory** metrics, and **code-level hotspots**. It ties together the three pillars — **metrics, logs, and traces** — often with alerting and dashboards.

Why it matters: it lets you **detect, localize, and diagnose** performance problems quickly (which service, which endpoint, which query), understand user-impacting latency, and do capacity planning. Tools: **New Relic, Datadog APM, Dynatrace, AppDynamics, Elastic APM**, and the open-source stack **OpenTelemetry + Prometheus/Grafana + Jaeger**. In Spring Boot you instrument with **Micrometer + Actuator** (and OpenTelemetry agents) to feed these systems. The goal: **observability** — being able to answer "why is it slow/failing right now?" from telemetry rather than guesswork.

---

*End of guide. Good luck with your interviews — for each answer, practice stating the one-line summary first, then the detail, and be ready for the follow-up "why" behind each choice.*