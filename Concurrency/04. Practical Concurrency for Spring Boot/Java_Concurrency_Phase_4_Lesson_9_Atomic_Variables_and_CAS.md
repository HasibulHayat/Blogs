# Practical Concurrency for Spring Boot - Phase 4, Lesson 9 : Atomic Variables and CAS

<br>

<br>

## 1. What You Will Learn

In this lesson, you will learn:

- Why a normal variable can cause problems when multiple threads update it.
- What an atomic variable is.
- How to use `AtomicInteger`, `AtomicLong`, and `AtomicBoolean`.
- What the `incrementAndGet()`, `getAndIncrement()`, `compareAndSet()`, and `updateAndGet()` methods do.
- What CAS (Compare-And-Set) means.
- Why atomic variables are useful in concurrent applications.
- When atomic variables are appropriate in your Turf Booking System.
- Why atomic variables do not replace database transactions or distributed locking.

By the end, you should be able to recognize when a shared counter or flag needs an atomic operation.

<br>

## 2. The Problem: Updating a Shared Variable

Imagine that two threads are updating the same counter.

```java
public class Counter {

    private int count = 0;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```

The important line is:

```java
count++;
```

It looks like one operation, but it is not guaranteed to happen as one indivisible operation between threads.

Conceptually, Java needs to:

1. Read the current value of `count`.
2. Add `1` to that value.
3. Write the new value back to `count`.

Suppose the current value is `0`.

Two threads can interleave their work like this:

| Step | Thread A | Thread B |
|---|---|---|
| 1 | Reads `count = 0` | |
| 2 | | Reads `count = 0` |
| 3 | Calculates `1` | |
| 4 | | Calculates `1` |
| 5 | Writes `count = 1` | |
| 6 | | Writes `count = 1` |

Both threads tried to increment the counter. You would expect the final value to be `2`, but it is `1`.

One increment was lost. This is called a **lost update**.

The issue is not that Java cannot add numbers correctly. The issue is that the complete read-modify-write operation is not protected from concurrent access.

<br>

## 3. What Is an Atomic Variable?

Java provides atomic classes in the `java.util.concurrent.atomic` package.

An atomic variable provides operations that are performed atomically. For example, an atomic increment updates the value without allowing another thread to interrupt that particular operation halfway through.

The most commonly used classes are:

| Class | Stores |
|---|---|
| `AtomicInteger` | An `int` value |
| `AtomicLong` | A `long` value |
| `AtomicBoolean` | A `boolean` value |
| `AtomicReference<T>` | A reference to an object |

For now, focus on `AtomicInteger`, `AtomicLong`, and `AtomicBoolean`.

An atomic class is designed for safe concurrent access to its supported operations. It does not automatically make every group of operations involving that variable safe.

<br>

## 4. Using AtomicInteger

Here is the previous counter rewritten with `AtomicInteger`.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class Counter {

    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}
```

Let's understand each important line.

<br>

### `AtomicInteger count = new AtomicInteger(0);`

```java
private final AtomicInteger count = new AtomicInteger(0);
```

- `AtomicInteger` is the class that stores the integer.
- `new AtomicInteger(0)` creates it with an initial value of `0`.
- `final` means the `count` variable cannot be assigned to a different `AtomicInteger` object later.
- The value inside the atomic object can still change.

<br>

### `count.incrementAndGet();`

```java
count.incrementAndGet();
```

This atomically:

1. Increments the stored value by `1`.
2. Returns the updated value.

If the value was `5`, it becomes `6`, and the method returns `6`.

<br>

### `count.get();`

```java
count.get();
```

Returns the current value stored inside the atomic variable.

If 10 threads each call `increment()` 1,000 times, the final count will be 10,000, assuming all calls complete.

<br>

## 5. Important AtomicInteger Methods

You should understand these methods because they are commonly used in concurrent Java code.

<br>

### 5.1 `get()`

```java
int current = count.get();
```

Reads the current value.

<br>

### 5.2 `set()`

```java
count.set(10);
```

Sets the value to `10`.

The write itself is atomic. However, using `set()` does not make a larger sequence of operations safe.

<br>

### 5.3 `incrementAndGet()`

```java
int result = count.incrementAndGet();
```

Increments first, then returns the new value.

Example:

```java
AtomicInteger count = new AtomicInteger(5);

int result = count.incrementAndGet();

System.out.println(result); // 6
System.out.println(count.get()); // 6
```

<br>

### 5.4 `getAndIncrement()`

```java
int result = count.getAndIncrement();
```

Returns the old value, then increments the stored value.

Example:

```java
AtomicInteger count = new AtomicInteger(5);

int result = count.getAndIncrement();

System.out.println(result); // 5
System.out.println(count.get()); // 6
```

The difference is:

| Method | Returned value | Stored value after call |
|---|---:|---:|
| `incrementAndGet()` | New value | New value |
| `getAndIncrement()` | Old value | New value |

<br>

### 5.5 `decrementAndGet()` and `getAndDecrement()`

These methods work like the increment methods, but subtract `1`.

```java
count.decrementAndGet();
count.getAndDecrement();
```

<br>

### 5.6 `addAndGet()`

Adds a given number and returns the updated value.

```java
int result = count.addAndGet(5);
```

If the current value is `10`, the new value is `15`, and the method returns `15`.

<br>

### 5.7 `getAndAdd()`

Returns the old value, then adds the supplied amount.

```java
int result = count.getAndAdd(5);
```

If the current value is `10`, the method returns `10`, and the stored value becomes `15`.

<br>

## 6. What Is CAS?

CAS means **Compare-And-Set** (also commonly called Compare-And-Swap).

It is an atomic operation that checks whether a value is what you expect. If it is, it changes the value. If it is not, it leaves the value unchanged.

The method looks like this:

```java
boolean changed = count.compareAndSet(expectedValue, newValue);
```

It performs two things as one atomic operation:

1. Checks whether the current value equals `expectedValue`.
2. If they are equal, changes the value to `newValue` and returns `true`.
3. If they are not equal, does not change the value and returns `false`.

<br>

### Example

```java
AtomicInteger count = new AtomicInteger(10);

boolean first = count.compareAndSet(10, 20);

System.out.println(first);       // true
System.out.println(count.get()); // 20

boolean second = count.compareAndSet(10, 30);

System.out.println(second);      // false
System.out.println(count.get()); // 20
```

Why did the first call succeed?

- The current value was `10`.
- The expected value was `10`.
- They matched, so the value changed to `20`.

Why did the second call fail?

- The current value was now `20`.
- The expected value was `10`.
- They did not match, so the value stayed `20`.

<br>

### Why is CAS useful?

CAS lets a thread say:

"Change this value only if nobody has changed it since the value I observed."

This is useful when multiple threads may try to update the same state.

<br>

## 7. A CAS Example: Safely Claiming a Shared Flag

Suppose an operation should be started only once inside one application instance.

```java
import java.util.concurrent.atomic.AtomicBoolean;

public class OneTimeTask {

    private final AtomicBoolean started = new AtomicBoolean(false);

    public void start() {

        if (started.compareAndSet(false, true)) {
            System.out.println("Starting task...");
            // Perform the one-time action.
        } else {
            System.out.println("Task has already been started.");
        }
    }
}
```

What happens when several threads call `start()` at the same time?

- Initially, `started` is `false`.
- Only one thread can successfully change it from `false` to `true`.
- That thread enters the `if` block.
- Other threads see that the value is already `true`, so their CAS operation returns `false`.

This prevents multiple threads in this JVM from successfully claiming the same one-time flag.

Important: this is only an in-memory, single-JVM guarantee. If your Spring Boot application runs on three server instances, each instance has its own `AtomicBoolean`. All three instances could start the task once. This is not a distributed lock.

<br>

## 8. `updateAndGet()` and `getAndUpdate()`

Sometimes you want to update a value using a calculation.

For example, increase a counter by `10`.

```java
int updated = count.updateAndGet(current -> current + 10);
```

The function receives the current value and returns the new value.

If the current value is `5`:

- `current` is `5`.
- `current + 10` is `15`.
- The atomic variable is updated to `15`.
- `updateAndGet()` returns `15`.

The expression:

```java
current -> current + 10
```

is a lambda expression. It describes how to calculate the new value from the current value.

You may also see:

```java
int oldValue = count.getAndUpdate(current -> current + 10);
```

This updates the value but returns the old value.

**Important:** the update function may be evaluated more than once when competing threads update the same variable. Keep the function free of side effects. Do not send emails, charge payments, write database records, or perform other external actions inside it.

<br>

## 9. AtomicLong

`AtomicLong` works like `AtomicInteger`, but stores a `long`.

Use it when the counter may exceed the range of an `int`, or when the value is naturally represented as a `long`.

```java
import java.util.concurrent.atomic.AtomicLong;

public class RequestCounter {

    private final AtomicLong totalRequests = new AtomicLong(0);

    public void recordRequest() {
        totalRequests.incrementAndGet();
    }

    public long getTotalRequests() {
        return totalRequests.get();
    }
}
```

Possible uses include:

- Counting requests handled by this application instance.
- Counting local background-task attempts.
- Tracking local metrics.

Remember that an in-memory counter resets when the application restarts unless you separately persist it.

<br>

## 10. AtomicBoolean

`AtomicBoolean` stores a boolean value and supports atomic updates.

```java
import java.util.concurrent.atomic.AtomicBoolean;

public class WorkerStatus {

    private final AtomicBoolean running = new AtomicBoolean(false);

    public void start() {
        if (running.compareAndSet(false, true)) {
            System.out.println("Worker started");
        }
    }

    public void stop() {
        running.set(false);
    }

    public boolean isRunning() {
        return running.get();
    }
}
```

Useful methods include:

- `get()` — read the current value.
- `set(value)` — set the value.
- `compareAndSet(expected, newValue)` — update only if the current value matches the expected value.
- `getAndSet(newValue)` — replace the value and return the previous value.

For example:

```java
boolean wasRunning = running.getAndSet(false);
```

This sets `running` to `false` and returns its previous value.

<br>

## 11. Atomic Variables vs `synchronized`

Both atomic variables and `synchronized` can help protect shared state, but they solve the problem in different ways.

<br>

### Using `synchronized`

```java
public class Counter {

    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

Only one thread at a time can execute a synchronized method on the same object monitor.

<br>

### Using `AtomicInteger`

```java
public class Counter {

    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}
```

The increment operation is atomic without using a `synchronized` method.

<br>

### Which should you use?

| Situation | Suitable choice |
|---|---|
| A simple shared counter | `AtomicInteger` or `AtomicLong` |
| A simple shared true/false flag | `AtomicBoolean` |
| Several related variables must change together | Often `synchronized` or a lock |
| A sequence of multiple steps must be protected as one operation | Often `synchronized` or a lock |
| A database booking must be unique across application instances | Database constraints and transaction design |

Do not choose atomic variables just because they sound faster. Choose the simplest mechanism that correctly protects the state you are changing.

<br>

## 12. Important Limitation: Atomic Does Not Mean Every Sequence Is Atomic

Consider this code:

```java
if (count.get() < 10) {
    count.incrementAndGet();
}
```

It contains two separate operations:

1. Read the value and check whether it is less than `10`.
2. Increment the value.

Those two operations are not one atomic operation.

Suppose the current value is `9`.

- Thread A reads `9`. The condition is true.
- Thread B reads `9`. The condition is also true.
- Thread A increments to `10`.
- Thread B increments to `11`.

The final value becomes `11`, even though the intention was to stop at `10`.

Use a CAS loop to make this particular check-and-update operation atomic:

```java
public void incrementUntilTen(AtomicInteger count) {

    while (true) {

        int current = count.get();

        if (current >= 10) {
            return;
        }

        int next = current + 1;

        if (count.compareAndSet(current, next)) {
            return;
        }
    }
}
```

How it works:

1. Read the current value.
2. If it is already `10` or more, stop.
3. Calculate the next value.
4. Try to change the value only if it is still equal to the value you read.
5. If another thread changed it first, CAS fails and the loop tries again.

This is a simple CAS retry loop. In real code, prefer a clear, well-tested approach and avoid writing custom concurrency algorithms unless you need them.

<br>

## 13. What Is the ABA Problem?

CAS compares the current value with the expected value. It does not necessarily tell you whether the value changed in between and later returned to the same value.

For example:

1. Thread A reads the value `10`.
2. Thread B changes it from `10` to `20`.
3. Thread B later changes it from `20` back to `10`.
4. Thread A performs `compareAndSet(10, 30)`.
5. The CAS operation succeeds because the current value is `10`.

Thread A cannot tell from the integer value alone that it changed in between.

This is known as the **ABA problem**.

You do not need to implement a solution for this in your Turf application's ordinary counters. Just know that it can matter in more advanced lock-free algorithms.

Java provides `AtomicStampedReference` for cases where you need to track a reference together with a version stamp. This is an advanced topic and is not required for the current Turf version.

<br>

## 14. Where Atomic Variables Fit in Your Turf Booking System

Atomic variables are useful for small pieces of shared, in-memory state inside one running application instance.

<br>

### Example 1: Local request counter

```java
@Component
public class RequestMetrics {

    private final AtomicLong requestCount = new AtomicLong(0);

    public void recordRequest() {
        requestCount.incrementAndGet();
    }

    public long getRequestCount() {
        return requestCount.get();
    }
}
```

This can count requests handled by this particular application instance.

It is not a reliable, permanent business metric unless you export or persist the measurements.

<br>

### Example 2: Preventing duplicate execution inside one JVM

An `AtomicBoolean` can ensure that only one thread in the same application instance claims a local one-time operation.

This can be useful for local initialization or a process-local task.

It does not coordinate multiple Spring Boot instances.

<br>

### Example 3: Local in-memory limit

A CAS loop can safely enforce a simple limit on a value stored in one JVM.

But if the limit is a business rule that must hold across all application instances, use a shared system such as the database or a distributed coordination mechanism.

<br>

## 15. Very Important: AtomicInteger Cannot Prevent Double Booking

Suppose your Turf Booking System has three Spring Boot instances:

- Instance A
- Instance B
- Instance C

Each instance has its own memory.

If each instance creates this field:

```java
private final AtomicInteger bookingCount = new AtomicInteger(0);
```

then each instance has a separate counter.

The values might look like this:

| Application instance | Local counter |
|---|---:|
| Instance A | 5 |
| Instance B | 5 |
| Instance C | 5 |

They are not sharing one atomic variable.

More importantly, an atomic counter cannot guarantee that two users do not book the same turf slot.

For booking correctness, use database-level protection, for example:

- A unique constraint that prevents duplicate active bookings for the same slot, where the data model supports it.
- Transactions.
- Appropriate locking or conditional updates when required.
- Idempotency for retryable booking and payment requests.

The database is shared by all application instances, so it can enforce rules across them. An `AtomicInteger` only coordinates threads using that particular object in that particular JVM.

**Remember:** atomic variables protect in-memory state. They do not replace database consistency.

<br>

## 16. Common Mistakes

<br>

### Mistake 1: Assuming `count++` is thread-safe

```java
count++;
```

A normal increment is a read-modify-write operation. Concurrent threads can lose updates.

<br>

### Mistake 2: Assuming `get()` followed by `set()` is one atomic operation

```java
int current = count.get();
count.set(current + 1);
```

Another thread can update the value between the read and the write. Use `incrementAndGet()` for a simple increment.

<br>

### Mistake 3: Assuming multiple atomic operations form one transaction

```java
if (count.get() < 10) {
    count.incrementAndGet();
}
```

The check and increment are separate operations. Use a CAS loop or a lock when the combined condition and update must be atomic.

<br>

### Mistake 4: Using an atomic variable for distributed coordination

An atomic object belongs to one JVM. It cannot coordinate separate application instances.

<br>

### Mistake 5: Performing side effects inside `updateAndGet()`

The update function may be retried. Keep it focused on calculating the new value.

<br>

### Mistake 6: Treating an in-memory counter as permanent data

The value is lost when the process stops unless you persist it elsewhere.

<br>

### Mistake 7: Using atomic variables for complicated shared state

If several fields must remain consistent together, a single atomic variable may not be enough. Use an appropriate lock or redesign the state.

<br>

## 17. Interview Questions and Answers

<br>

### Q1. What is an atomic variable in Java?

An atomic variable is a variable provided by classes in `java.util.concurrent.atomic` that supports thread-safe atomic operations, such as incrementing or compare-and-set.

<br>

### Q2. What is CAS?

CAS stands for Compare-And-Set or Compare-And-Swap. It checks whether a value matches an expected value and updates it only if the values match.

<br>

### Q3. What is the difference between `incrementAndGet()` and `getAndIncrement()`?

`incrementAndGet()` increments first and returns the new value. `getAndIncrement()` returns the old value and then increments.

<br>

### Q4. Is `count++` thread-safe?

No. It is a read-modify-write operation and can lose updates when multiple threads execute it concurrently.

<br>

### Q5. Does `AtomicInteger` use `synchronized` for every operation?

Its atomic operations are generally implemented using low-level atomic CPU and JVM mechanisms, commonly involving CAS. You should rely on the documented atomic behavior rather than assume a particular internal implementation.

<br>

### Q6. Does an `AtomicInteger` make a whole method thread-safe?

No. It makes its individual supported operations atomic. Multiple operations still need coordination if they must behave as one indivisible operation.

<br>

### Q7. Can `AtomicInteger` prevent two users from booking the same turf slot?

No. It only coordinates access to one atomic object inside one JVM. Use database constraints and transaction design for booking correctness across application instances.

<br>

### Q8. When would you choose `synchronized` instead of an atomic variable?

Use `synchronized` when multiple related operations or variables must be protected together. Use an atomic variable for a simple shared value with supported atomic operations.

<br>

### Q9. What is the ABA problem?

It is a situation where a value changes from A to B and then back to A. A CAS operation that checks only the current value may not detect that the intermediate change happened.

<br>

## 18. Practice Questions

Try answering these before looking at the answers.

<br>

### Question 1

What will this code print?

```java
AtomicInteger count = new AtomicInteger(5);

int result = count.incrementAndGet();

System.out.println(result);
System.out.println(count.get());
```

**Answer:**

```text
6
6
```

`incrementAndGet()` increments the value and returns the updated value.

<br>

### Question 2

What will this code print?

```java
AtomicInteger count = new AtomicInteger(5);

int result = count.getAndIncrement();

System.out.println(result);
System.out.println(count.get());
```

**Answer:**

```text
5
6
```

`getAndIncrement()` returns the old value, then increments.

<br>

### Question 3

What will happen here?

```java
AtomicInteger count = new AtomicInteger(10);

boolean result = count.compareAndSet(5, 20);

System.out.println(result);
System.out.println(count.get());
```

**Answer:**

```text
false
10
```

The current value is `10`, not the expected value `5`. The update does not happen.

<br>

### Question 4

Is this code safe if multiple threads execute it?

```java
if (count.get() < 10) {
    count.incrementAndGet();
}
```

**Answer:**

No. The check and increment are separate operations. Multiple threads can pass the check before any of them increments the value.

<br>

### Question 5

Three Spring Boot instances each have their own `AtomicInteger`. Do they share one counter?

**Answer:**

No. Each application instance has its own JVM memory and its own atomic object.

<br>

### Question 6

Which would you choose for a simple concurrent request counter inside one application instance: `int` or `AtomicLong`?

**Answer:**

`AtomicLong` is a suitable choice if multiple threads update the counter concurrently. A normal `int` with `count++` is not safe for concurrent increments.

<br>

## 19. System Design Questions for Turf

<br>

### Question 1: You want to count how many HTTP requests one application instance handles. What could you use?

You could use an `AtomicLong` for a simple in-memory counter. For production monitoring, prefer an established metrics library and export metrics to your monitoring system.

<br>

### Question 2: You want to ensure a scheduled task starts only once per application instance. What could you use?

An `AtomicBoolean` with `compareAndSet(false, true)` can ensure only one thread in that JVM claims the task.

If the application runs on multiple instances and the task must run only once across the entire deployment, use a distributed coordination solution or a database-backed job-claiming design.

<br>

### Question 3: Two users try to book the same turf slot at almost the same time. Should you use `AtomicBoolean`?

No. The booking rule must be enforced by the shared database and transaction design. An in-memory flag cannot coordinate all application instances or survive restarts.

<br>

### Question 4: You need to update a booking count and a remaining-capacity value together. Is one `AtomicInteger` enough?

Not necessarily. If both values must always remain consistent, use a transaction or a synchronization mechanism that protects the complete state change. In a persistent booking system, the database is usually the right place to enforce this business rule.

<br>

## 20. Lesson Summary

You learned:

- A normal `count++` can lose updates when multiple threads execute it.
- Atomic classes provide thread-safe operations on individual values.
- `AtomicInteger` stores an integer; `AtomicLong` stores a long; `AtomicBoolean` stores a boolean.
- `incrementAndGet()` returns the updated value.
- `getAndIncrement()` returns the previous value.
- CAS changes a value only when it matches an expected value.
- A sequence of atomic operations is not automatically atomic as a group.
- Atomic variables coordinate threads inside one JVM, not separate application instances.
- Atomic variables do not replace database constraints, transactions, or distributed coordination.

**The key lesson for Turf:** use atomic variables for simple, local, in-memory concurrent state. Use the database to protect important booking and payment business rules across the whole system.

<br>

## 21. Completion Checklist

Before moving to Lesson 10, make sure you can:

- [ ] Explain why `count++` is not thread-safe.
- [ ] Explain what an atomic variable does.
- [ ] Use `AtomicInteger` for a shared counter.
- [ ] Explain the difference between `incrementAndGet()` and `getAndIncrement()`.
- [ ] Use `compareAndSet()` to update a value conditionally.
- [ ] Explain why a check followed by an increment may still have a race condition.
- [ ] Explain why atomic variables do not prevent double booking across multiple application instances.
