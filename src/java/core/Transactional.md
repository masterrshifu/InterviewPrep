# How @Transactional Actually Works Under the Hood in Spring Boot

*Thirteen characters, a runtime proxy, and a JDBC flag called `autoCommit`. Here's the full machinery — and the four places it quietly breaks.*

---

## The Comfortable Lie We All Believe

Most of us learn `@Transactional` the same way: "Put it on a method, everything inside runs in one transaction, and if something throws, it all rolls back." We nod, it works, we move on.

That model is *mostly* true — which is exactly why it's dangerous. It hides the machinery, and the machinery is where the production bugs live. So let's replace the magic with mechanics.

The first uncomfortable truth: **`@Transactional` is not a database feature, and it's barely a Spring feature. It's a proxy trick wrapped around one JDBC flag.**

---

## 1. Spring Never Touches Your Method — It Wraps It

An annotation is just metadata. It can't *do* anything on its own. So how does behavior get injected?

At startup, a `BeanPostProcessor` scans your beans. When it finds `@Transactional` on a class or method, it does not put your raw object into the application context. It puts a **proxy** there — a generated stand-in that holds a reference to your real object.

Everyone who "autowires your service" is actually wired to this proxy. When you call `orderService.placeOrder()`, you're calling the proxy, which opens a transaction, *then* calls your real method, *then* commits or rolls back.

Spring builds that proxy in one of two ways:

| Proxy type | When it's used | How it's built |
|---|---|---|
| **JDK dynamic proxy** | Your bean implements an interface | Runtime class implementing the same interface |
| **CGLIB proxy** | No interface present | Runtime **subclass** that overrides your methods |

> **Why this matters:** the transactional behavior lives in the *wrapper*, not your object. Only calls that pass *through* the wrapper get a transaction. Remember this — it explains bug #1 later.

---

## 2. The Real Magic Is `setAutoCommit(false)`

When a call hits the proxy, it flows through a chain of interceptors. The important one is `TransactionInterceptor`, and here's the logic it runs *around* your method:

```java
// Simplified TransactionInterceptor flow
TransactionAttribute attr = readAnnotationMetadata(); // propagation, isolation, rollbackFor, readOnly

TransactionStatus tx = transactionManager.getTransaction(attr);
// Under the hood, getTransaction():
//   1. Grabs a Connection from the DataSource (pool)
//   2. connection.setAutoCommit(false);      <-- THE ENTIRE TRICK
//   3. Binds that Connection to the current thread (ThreadLocal)

try {
    Object result = yourActualMethod();       // your code finally runs
    transactionManager.commit(tx);            // -> connection.commit()
    return result;
} catch (Throwable ex) {
    if (shouldRollback(ex, attr))
        transactionManager.rollback(tx);      // -> connection.rollback()
    else
        transactionManager.commit(tx);        // yes, it can still commit on exception
    throw ex;
}
```

That one line — `setAutoCommit(false)` — is the whole story.

By default, JDBC commits after *every single statement*. Spring flips auto-commit off, lets all your statements pile up on the connection, and commits **once** at the end. The transaction isn't Spring's and it isn't the annotation's — it belongs to the **database**. Spring is just the choreographer who tells the DB when to start, commit, and roll back.

---

## 3. Why Three Repositories Land in One Transaction (Thread-Binding)

Notice step 3 above: the connection gets **bound to the current thread** via `TransactionSynchronizationManager`, which stores it in a `ThreadLocal`.

Now when any repository runs a query, Spring's `DataSourceUtils.getConnection()` asks one question: *is there already a connection bound to this thread?*

- **Yes** → reuse it. That's how three different repository calls inside one method all join the *same* transaction.
- **No** → hand out a fresh, auto-committing connection.

```java
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);       // ┐
    paymentRepo.record(order);   // ├─ same thread → same bound connection → one transaction
    inventoryRepo.deduct(order); // ┘
}
```

The consequence is subtle and important: **transactions are thread-bound.** Cross a thread boundary — a `@Async` method, a `new Thread()`, a parallel stream — and the new thread has *no* bound connection. Your transaction does not follow it. The work on that thread runs in its own auto-committing world and commits independently. This is one of the most common "half my data saved" production surprises.

---

## The Four Places @Transactional Quietly Breaks

Everything above is the happy path. Here's where the same mechanics turn against you. Every one of these looks correct and does nothing.

### Trap #1 — Self-Invocation (the silent killer)

```java
@Service
public class OrderService {

    public void process() {
        placeOrder();   // internal call — NO transaction starts
    }

    @Transactional
    public void placeOrder() { ... }
}
```

The transaction lives in the **proxy**. But `placeOrder()` here is really `this.placeOrder()` — a direct call on the *raw* object. The proxy never sees it, so it never intercepts it. The annotation is right there, it looks perfect, and it does absolutely nothing.

**Fix:** move the method to a separate bean, inject a self-reference to the proxy, or use `AopContext.currentProxy()`.

### Trap #2 — Checked Exceptions Don't Roll Back

```java
@Transactional
public void placeOrder() throws IOException {
    orderRepo.save(order);
    throw new IOException("gateway down");   // saved order COMMITS anyway
}
```

**By default, Spring rolls back only on `RuntimeException` and `Error` — never on checked exceptions.** This is an inheritance from old EJB conventions and it matches almost nobody's intuition.

**Fix:** be explicit.

```java
@Transactional(rollbackFor = Exception.class)
```

### Trap #3 — Swallowing the Exception

If you `try/catch` inside the method and don't rethrow, the exception never reaches the proxy — so the proxy sees a clean return and **commits**. Rollback is driven by what escapes your method, not by what happens inside it.

### Trap #4 — Rolling Back External Side Effects

```java
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    paymentGateway.charge(order);  // external HTTP call
    inventoryRepo.deduct(order);
}
```

A database transaction can un-save a row. It **cannot** un-charge a credit card. Rollback governs the database only — never external HTTP calls, message sends, or file writes. That's an architecture problem (outbox pattern, sagas, compensating actions), not something an annotation can solve.

---

## The Mental Model That Sticks

Forget "it makes a transaction." Hold these five facts instead:

1. **It's a proxy.** Spring wraps your bean; only calls that go *through* the wrapper get a transaction.
2. **The real mechanism is `setAutoCommit(false)`.** Spring borrows a JDBC connection, turns off auto-commit, runs your method, then commits/rolls back once. The database does the real work.
3. **It's thread-bound.** The active connection lives in a `ThreadLocal`. Cross a thread and you leave the transaction behind.
4. **Self-invocation is invisible.** A method calling another `@Transactional` method on the same object bypasses the proxy entirely.
5. **Default rollback is unchecked-only.** Checked exceptions commit unless you say `rollbackFor = Exception.class`.

---

## One Line to Remember

`@Transactional` is a proxy whispering `setAutoCommit(false)` to a connection pinned to your thread. The moment you cross a thread, call yourself internally, or throw the wrong flavor of exception, it quietly stops protecting you.

Master those five facts and the annotation stops being a spell you cast and hope. It becomes a tool whose sharp edges you can see coming — which is exactly what separates *"I use `@Transactional`"* from *"I understand `@Transactional`"* in every interview and every incident review.

---

*Found this useful? The five facts above are worth a screenshot. 👏 Follow for more deep-dives into the Spring internals nobody explains until they break in production.*