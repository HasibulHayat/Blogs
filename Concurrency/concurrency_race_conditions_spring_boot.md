# Concurrency Fundamentals: Race Conditions and Protection in Spring Boot

## Learning objectives

After studying this document, you should be able to:

-   Explain what a race condition is and why it happens.
-   Identify shared mutable state and unsafe operations.
-   Trace how concurrent threads can interfere with one another.
-   Explain why application-level synchronization may not be enough in a
    multi-instance backend.
-   Distinguish Java synchronization, database transactions, locking,
    atomic updates, and constraints.
-   Explain why reliable booking systems usually enforce critical
    invariants at the database level.

------------------------------------------------------------------------

# 1. What is a race condition?

A **race condition** occurs when the correctness of a program depends on
the unpredictable timing or ordering of concurrent operations.

It commonly appears when multiple threads access and modify shared data
without sufficient coordination.

For example, imagine a bank account with a balance of 1,000:

``` java
public class BankAccount {
    private int balance = 1000;

    public void withdraw(int amount) {
        if (balance >= amount) {
            balance = balance - amount;
        }
    }

    public int getBalance() {
        return balance;
    }
}
```

Two threads attempt to withdraw money:

-   Thread A withdraws 700.
-   Thread B withdraws 500.

Only one withdrawal should succeed because the combined amount, 1,200,
is greater than the available balance of 1,000.

However, the withdrawal method consists of multiple logical steps:

1.  Read the balance.
2.  Check whether enough money is available.
3.  Calculate the new balance.
4.  Write the new balance.

These steps are not automatically one indivisible operation.

## 1.1 A possible unsafe execution

  Step   Thread A               Thread B                 Balance
  ------ ---------------------- ---------------------- ---------
  1      Reads 1,000                                       1,000
  2                             Reads 1,000                1,000
  3      Checks 1,000 \>= 700                              1,000
  4                             Checks 1,000 \>= 500       1,000
  5      Writes 300                                          300
  6                             Writes 500                   500

Both threads believed their withdrawal was valid. The final balance is
500, even though both withdrawals were supposedly successful.

This is a race condition. It is also an example of a **lost update**:
one thread's update overwrites another thread's result.

The key lesson is that a line of code that looks simple may involve
several lower-level operations.

------------------------------------------------------------------------

# 2. Shared mutable state

Three terms help us recognize common sources of race conditions:

  Term      Meaning
  --------- ---------------------------------------------------------
  Shared    Multiple threads can access the data.
  Mutable   The data can be changed.
  State     The data represents the current condition of something.

Examples include:

-   A bank account balance.
-   An inventory quantity.
-   A booking status.
-   A shared Java object.
-   A cache entry.
-   A counter modified by multiple threads.

Shared mutable state is a common source of race conditions, but it does
not automatically mean a race condition exists. Correct coordination can
make concurrent access safe.

A useful question when reviewing code is:

> Can two concurrent executions access this data, and can at least one
> of them modify it?

If so, inspect how those operations are coordinated.

------------------------------------------------------------------------

# 3. A simple Java race condition: `count++`

Consider this counter:

``` java
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

Suppose two threads each call `increment()` 10,000 times.

The expected result is:

``` text
20,000
```

But `count++` is not an atomic operation. Conceptually, it involves:

1.  Reading the current value.
2.  Adding one.
3.  Writing the result.

A possible interleaving:

  Step   Thread A        Thread B          Stored count
  ------ --------------- --------------- --------------
  1      Reads 10                                    10
  2                      Reads 10                    10
  3      Calculates 11                               10
  4                      Calculates 11               10
  5      Writes 11                                   11
  6                      Writes 11                   11

Two increments occurred, but the stored value increased only once.

This is why the final result can be less than 20,000.

------------------------------------------------------------------------

# 4. Race conditions in a Turf Booking System

Consider a football turf with one available slot:

``` text
Turf: Greenfield Football Turf
Date: November 15, 2026
Time: 10:00 AM
Status: AVAILABLE
```

Two customers attempt to book the same slot at nearly the same time.

A naive Spring Boot service might look like this:

``` java
public void bookSlot(Long slotId) {
    Slot slot = slotRepository.findById(slotId)
            .orElseThrow();

    if (slot.getStatus() == SlotStatus.AVAILABLE) {
        slot.setStatus(SlotStatus.BOOKED);
        slotRepository.save(slot);
    }
}
```

## 4.1 How double booking can happen

Both requests may read the slot while its status is `AVAILABLE`.

  Request A                    Request B
  ---------------------------- ----------------------------
  Reads `AVAILABLE`            
                               Reads `AVAILABLE`
  Checks availability          
                               Checks availability
  Changes status to `BOOKED`   
                               Changes status to `BOOKED`
  Saves                        
                               Saves

Both requests may proceed as if they successfully booked the slot.

The business consequence is a **double booking**: two customers believe
they reserved one slot.

This can cause refunds, customer complaints, operational confusion, and
loss of trust.

The booking system therefore needs a reliable way to ensure that only
one competing request can claim the slot.

------------------------------------------------------------------------

# 5. Preventing race conditions in Java

Java provides several concurrency tools. Each solves a particular kind
of problem.

## 5.1 `synchronized`

``` java
public synchronized void withdraw(int amount) {
    if (balance >= amount) {
        balance -= amount;
    }
}
```

For an instance `synchronized` method, a thread must acquire that
object's intrinsic monitor before entering the method. Other threads
trying to enter synchronized instance methods guarded by the same object
monitor must wait.

Important limitation: this coordinates threads that share the same
object in the same JVM. It does not create a lock shared across separate
application processes or servers.

## 5.2 `ReentrantLock`

``` java
private final Lock lock = new ReentrantLock();

public void withdraw(int amount) {
    lock.lock();
    try {
        if (balance >= amount) {
            balance -= amount;
        }
    } finally {
        lock.unlock();
    }
}
```

`ReentrantLock` provides explicit lock management and additional
features. Unlocking in a `finally` block is important so the lock is
released even if an exception occurs.

Like `synchronized`, an ordinary `ReentrantLock` instance coordinates
only threads using that same lock instance.

## 5.3 Atomic variables

For a simple counter, Java offers `AtomicInteger`:

``` java
private final AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet();
}
```

`incrementAndGet()` performs the increment atomically.

Atomic variables are useful for individual atomic operations. However,
several separate atomic operations do not automatically make a larger
multi-step business workflow atomic.

------------------------------------------------------------------------

# 6. Why Spring Boot synchronization alone may not be enough

Imagine the application grows and is deployed as two Spring Boot
instances behind a load balancer.

``` text
                 Customers
                     |
                Load Balancer
                 /         \
                /           \
       Spring Boot 1     Spring Boot 2
          JVM 1             JVM 2
             \               /
              \             /
               Shared Database
```

Each JVM has its own memory and its own lock objects.

If `synchronized` protects a method in JVM 1, it does not stop a thread
in JVM 2 from entering the corresponding method. The two JVMs do not
share that in-memory monitor.

Therefore, application-level synchronization by itself is not enough to
coordinate competing requests across multiple application instances.

This is a major reason that backend systems use database-level
concurrency control for important data invariants.

------------------------------------------------------------------------

# 7. The database as the final integrity boundary

For a booking system, an important business invariant is:

> A particular turf must not have two confirmed bookings for the same
> time slot.

The database is shared by all application instances. It can coordinate
competing transactions and enforce data integrity regardless of which
application instance receives a request.

There are several relevant database techniques.

## 7.1 Atomic conditional update

One possible approach is to claim a slot with a conditional SQL update:

``` sql
UPDATE booking_slots
SET status = 'BOOKED'
WHERE id = 101
  AND status = 'AVAILABLE';
```

Then inspect the number of affected rows:

-   **1 row affected:** this request changed the slot from available to
    booked.
-   **0 rows affected:** the slot was not available when the update was
    evaluated, or the requested row did not exist.

The database serializes conflicting writes to the same row. Once one
transaction successfully changes the status, a competing conditional
update cannot also change that row from `AVAILABLE` to `BOOKED`.

The application should treat the affected-row count as the result of the
claim attempt.

## 7.2 Transactions

A database transaction groups operations into a unit of work.

For example, confirming a booking might involve:

1.  Claiming the slot.
2.  Creating a booking record.
3.  Recording a payment or payment intent.
4.  Writing an audit record.

A transaction can ensure that related database changes commit together
or roll back together.

However, **a transaction alone does not automatically prevent every race
condition**. The result depends on the isolation level, queries, locks,
constraints, and data model.

## 7.3 Pessimistic locking

Pessimistic locking assumes conflicts may happen and locks data while a
transaction works with it.

In SQL, a common example is:

``` sql
SELECT *
FROM booking_slots
WHERE id = 101
FOR UPDATE;
```

The row is locked for the transaction, so another transaction attempting
a conflicting row lock must wait until the lock is released.

In Spring Data JPA, pessimistic locking can be expressed with a
repository query and `@Lock`, for example:

``` java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select s from Slot s where s.id = :id")
Optional<Slot> findByIdForUpdate(Long id);
```

The locking query must execute inside an appropriate transaction.

## 7.4 Optimistic locking

Optimistic locking assumes conflicts are relatively uncommon. It detects
conflicting updates rather than making every transaction wait up front.

With JPA, a version field can be declared:

``` java
@Entity
public class Slot {

    @Id
    private Long id;

    @Version
    private Long version;

    private SlotStatus status;
}
```

When updating the entity, the persistence provider checks that the
version has not changed since it was read. If another transaction has
already updated it, the stale update fails with an optimistic locking
conflict.

The application must handle that conflict appropriately, such as by
returning a conflict response or retrying a safe operation.

## 7.5 Database constraints

Constraints protect data rules at the database boundary.

For example, a unique constraint can prevent duplicate active
reservations for a slot, if the schema and reservation lifecycle are
designed to represent that rule correctly.

A constraint is valuable because it protects against mistakes from any
application instance, background job, script, or other database client.

The exact constraint depends on the data model. A simple unique
constraint on `slot_id` is appropriate only if a slot represents one
uniquely bookable occurrence and the row itself is the permanent
reservation record. Other designs may require a composite key or a
partial unique index.

------------------------------------------------------------------------

# 8. Which protection should we use?

There is no single concurrency mechanism that is best for every
situation.

  -----------------------------------------------------------------------
  Situation                           Commonly appropriate mechanism
  ----------------------------------- -----------------------------------
  Counter shared by threads in one    `AtomicInteger`
  JVM                                 

  Protecting a shared Java object's   `synchronized` or `Lock`
  multi-step operation                

  Claiming one database row           Conditional atomic update

  Coordinating transactions over a    Pessimistic locking
  row                                 

  Detecting concurrent entity updates Optimistic locking with `@Version`

  Keeping related database changes    Database transaction
  together                            

  Preventing invalid or duplicate     Database constraints
  data                                
  -----------------------------------------------------------------------

These mechanisms are not interchangeable. A transaction, for example,
groups database work but does not by itself guarantee that a business
rule is protected against every concurrent interleaving.

------------------------------------------------------------------------

# 9. Application-level and database-level protection: do we need both?

Sometimes one layer is enough for a narrowly scoped problem. In a real
multi-instance backend, however, it is common to use both layers for
different responsibilities.

A useful division of responsibility is:

### Application layer (Spring Boot)

-   Validates the request.
-   Applies business rules.
-   Coordinates the booking workflow.
-   Uses transactions and appropriate repository operations.
-   Handles conflicts and communicates meaningful outcomes to clients.

### Database layer

-   Coordinates conflicting writes.
-   Enforces constraints.
-   Preserves data integrity.
-   Acts as the shared consistency boundary across application
    instances.

For the Turf Booking System, we should not rely only on a Java
`synchronized` method to prevent double bookings. We should design the
database operation and schema so that competing requests cannot both
claim the same slot.

We should also avoid adding locks everywhere. Unnecessary locking can
reduce throughput and introduce deadlocks.

------------------------------------------------------------------------

# 10. Key takeaways

1.  A race condition occurs when concurrent operations interact with
    shared state and the order of execution affects correctness.
2.  Shared mutable state is a common source of race conditions.
3.  Operations such as `count++` and check-then-update logic are not
    automatically atomic.
4.  Race conditions can cause lost updates, incorrect balances,
    duplicate work, and double bookings.
5.  Java synchronization mechanisms coordinate threads that share the
    relevant in-memory lock.
6.  In-memory locks do not coordinate separate Spring Boot instances.
7.  Database transactions, locks, atomic updates, and constraints solve
    different parts of the problem.
8.  Critical business invariants should be enforced at the database
    boundary when multiple requests or application instances can
    compete.

------------------------------------------------------------------------

# 11. Knowledge check

Try answering these questions before moving to the next topic.

1.  What is a race condition?
2.  Why is `count++` not necessarily thread-safe?
3.  What is shared mutable state?
4.  How can two customers double-book one slot?
5.  Why does `synchronized` in one JVM not protect another JVM?
6.  Does adding `@Transactional` automatically prevent double booking?
    Why or why not?
7.  What does an affected-row count of zero mean after a conditional
    booking update?
8.  What is the difference between optimistic and pessimistic locking?
9.  Why are database constraints useful even when application validation
    exists?

------------------------------------------------------------------------

# 12. Next lesson: Critical Sections

The next topic is **critical sections**.

We will learn:

-   What a critical section is.
-   Why only certain parts of a method need protection.
-   How to identify critical sections in Java code.
-   Why locking an entire method can be inefficient.
-   How critical sections connect to race conditions.
-   How to identify the critical section in our Turf Booking System.

A useful question to bring into the next lesson:

> In a booking method, should we protect the entire method, or only the
> availability check and the operation that claims the slot?

Understanding critical sections will prepare us to study synchronization
and locks properly.
