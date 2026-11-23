<br>

# Java Concurrency — Phase 2, Lesson 2: Race Conditions

<br>
<br>

## 1. A real-world example: Turf bookings

Imagine your Turf booking system has one available slot:

```java
int availableSlots = 1;
```

Two users attempt to book it at nearly the same time.

1. User A checks availability and sees one slot.
2. User B checks availability and also sees one slot.
3. User A attempts to book it.
4. User B also attempts to book it.

If the application does not coordinate these operations correctly, it may accept two bookings for one slot. This is a simplified race-condition example.

<br>

## 2. A shared counter example

```java
public class Counter {
    int count = 0;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```

Two threads access the same `Counter` object:

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();

        Thread threadA = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        });

        Thread threadB = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        });

        threadA.start();
        threadB.start();

        threadA.join();
        threadB.join();

        System.out.println("Final count: " + counter.getCount());
    }
}
```

We expect 2,000 because each thread attempts 1,000 increments. But the result is **not guaranteed** to be 2,000. It may be smaller because both threads update the same variable without coordination.

A run may still produce 2,000 by chance. A race condition does not have to produce an incorrect result every time.

<br>

## 3. Why can `count++` lose an update?

This statement looks like one operation:

```java
count++;
```

Conceptually, it involves multiple steps:

1. Read the current value.
2. Add one.
3. Write the new value back.

Suppose `count` starts at 10:

```
| Step | Thread A | Thread B |
|---:|---|---|
| 1 | Reads 10 | |
| 2 | | Reads 10 |
| 3 | Calculates 11 | |
| 4 | | Calculates 11 |
| 5 | Writes 11 | |
| 6 | | Writes 11 |
```

The final value is 11, although two increments were attempted. One update was overwritten. This is called a **lost update**.

A single line of Java code is not necessarily one indivisible operation.

<br>

## 4. Thread interleaving

Thread interleaving means operations from different threads can be mixed together.

One possible order:

```text
Thread A: Read
Thread B: Read
Thread A: Calculate
Thread B: Calculate
Thread A: Write
Thread B: Write
```

Another possible order:

```text
Thread A: Read
Thread A: Calculate
Thread A: Write
Thread B: Read
Thread B: Calculate
Thread B: Write
```

The second order produces the expected result for these two increments. We cannot rely on the scheduler always choosing a favorable order.

<br>

## 5. Reproducing a race condition

To make the problem easier to observe, insert a small delay between reading and writing:

```java
public class Counter {
    int count = 0;

    public void increment() {
        int current = count;

        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return;
        }

        count = current + 1;
    }
}
```

Then run two threads:

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();

        Thread threadA = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                counter.increment();
            }
        });

        Thread threadB = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                counter.increment();
            }
        });

        threadA.start();
        threadB.start();

        threadA.join();
        threadB.join();

        System.out.println("Expected: 200");
        System.out.println("Actual: " + counter.count);
    }
}
```

A possible result:

```text
Expected: 200
Actual: 100
```

The sleep creates an opportunity for another thread to run after the current thread has read the old value but before it writes the new value.

**Caveats:**
- `Thread.sleep(1)` does not guarantee an exact one-millisecond delay.
- It does not guarantee a particular thread ordering.
- It is a demonstration technique, not a production fix.
- Sleep does not make shared data safe.

The catch block restores the interrupt status and exits the increment operation if the thread is interrupted, rather than silently swallowing the interruption.

<br>

## 6. Race conditions are not limited to counters

Race conditions can occur whenever concurrent operations depend on shared mutable state.

<br>

### A. Bank account

Suppose:

```java
int balance = 1000;
```

Two threads attempt withdrawals at the same time.

- Thread A reads 1,000 and plans to withdraw 700.
- Thread B reads 1,000 and plans to withdraw 500.
- Both may believe sufficient funds are available.

Without coordination, the resulting balance or transaction outcome can be incorrect.

<br>

### B. Inventory or slot availability

Suppose:

```java
int availableSlots = 5;
```

Several users try to reserve slots. Without proper concurrency control, the system may accept more reservations than capacity allows.

<br>

### C. Payment processing

Two requests may attempt to process the same payment. Duplicate charges can involve concurrency, retries, distributed systems, and missing idempotency controls. It is not only a Java-thread problem.

<br>

## 7. Database check-then-act race

A database does not automatically solve every race condition.

Consider:

```java
if (bookingRepository.existsByTurfIdAndTimeSlot(turfId, timeSlot)) {
    throw new RuntimeException("Slot already booked");
}

bookingRepository.save(booking);
```

Two requests may interleave:

```
| Time | Request A | Request B |
|---|---|---|
| 1 | Checks the slot | |
| 2 | Slot appears available | |
| 3 | | Checks the slot |
| 4 | | Slot appears available |
| 5 | Saves booking | |
| 6 | | Saves booking |
```

Both requests passed the availability check.

This is a **check-then-act race condition**: checking a condition and acting on it are separate operations, so another request can change the state between them.

A transaction alone does not automatically prevent every such race. Correct protection depends on the database, isolation level, schema, and transaction design.

Later, we will study:
- Unique constraints.
- Transaction isolation levels.
- Optimistic locking.
- Pessimistic locking.
- Atomic database updates.

A production booking system needs data-integrity protection at the database level as well as appropriate application-level coordination.

<br>

## 8. How can race conditions be prevented?

We will study these mechanisms in later lessons:

```
| Technique | What it helps with |
|---|---|
| `synchronized` | Coordinates access to critical sections |
| `volatile` | Provides visibility guarantees; it does not make increments atomic |
| `AtomicInteger` | Provides atomic operations such as increment |
| `ReentrantLock` | Provides explicit lock-based coordination |
| Database constraints | Enforces data integrity in the database |
| Optimistic/pessimistic locking | Coordinates conflicting database updates |
```

For now, focus on understanding the problem before memorizing solutions.

<br>

## 9. Important things to remember

1. A race condition occurs when concurrent operations interact with shared state and timing affects the outcome.
2. Multiple threads can read the same value before either writes an update.
3. `count++` is not an atomic operation.
4. A lost update occurs when one update overwrites another.
5. Thread interleaving can produce different execution orders.
6. A race condition may not appear in every run.
7. `Thread.sleep()` can help demonstrate timing problems, but it does not fix them.
8. Race conditions can affect counters, balances, inventory, bookings, and payments.
9. Database code can also have check-then-act races.
10. Correct solutions require appropriate synchronization and data-integrity mechanisms.

<br>
