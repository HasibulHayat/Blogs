# Java Concurrency - Phase 3 Lesson 7 : Deadlock, Starvation, and Livelock

## 1. Why do these problems happen?

Concurrency allows multiple threads to make progress at the same time.
But when threads compete for resources, coordination mistakes can cause
serious problems.

Three important problems are:

1.  **Deadlock:** threads are stuck waiting for one another forever.
2.  **Starvation:** a thread keeps getting denied the resource or time
    it needs.
3.  **Livelock:** threads keep reacting to one another, but no useful
    work gets completed.

They can look similar from the outside, but their causes are different.

## 2. Deadlock --- two threads waiting forever

### Simple analogy

Imagine:

-   Person A holds a pen and waits for a notebook.
-   Person B holds the notebook and waits for the pen.
-   Neither person gives up what they already hold.

Both wait forever.

This is deadlock.

### Java example

``` java
public class DeadlockDemo {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread thread1 = new Thread(() -> {
            synchronized (lockA) {
                System.out.println("Thread 1 acquired lockA");

                sleep();

                synchronized (lockB) {
                    System.out.println("Thread 1 acquired lockB");
                }
            }
        });

        Thread thread2 = new Thread(() -> {
            synchronized (lockB) {
                System.out.println("Thread 2 acquired lockB");

                sleep();

                synchronized (lockA) {
                    System.out.println("Thread 2 acquired lockA");
                }
            }
        });

        thread1.start();
        thread2.start();
    }

    private static void sleep() {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### How deadlock occurs

One possible execution:

1.  Thread 1 acquires `lockA`.
2.  Thread 2 acquires `lockB`.
3.  Thread 1 tries to acquire `lockB`, but Thread 2 owns it.
4.  Thread 2 tries to acquire `lockA`, but Thread 1 owns it.
5.  Thread 1 waits for Thread 2.
6.  Thread 2 waits for Thread 1.

Neither can proceed.

The sleep makes the timing overlap more likely, but thread scheduling is
nondeterministic. The program may not deadlock on every run.

## 3. The four conditions for deadlock

The classic four necessary conditions are:

1.  **Mutual exclusion:** a resource can be held by only one thread at a
    time.
2.  **Hold and wait:** a thread holds one resource while waiting for
    another.
3.  **No preemption:** a resource cannot simply be taken away from its
    owner.
4.  **Circular wait:** there is a cycle of threads, each waiting for a
    resource held by the next.

If you can reliably break at least one of these conditions, you can
prevent this form of deadlock.

## 4. How to prevent deadlocks

### Technique 1: Always acquire locks in the same order

If every thread acquires `lockA` before `lockB`, the opposite-order
cycle cannot form.

``` java
synchronized (lockA) {
    synchronized (lockB) {
        // Work requiring both locks.
    }
}
```

Every code path that needs both locks must follow the same order.

### Technique 2: Avoid nested locks when possible

If an operation can be redesigned to hold only one lock at a time, the
chance of lock-order deadlock is reduced.

### Technique 3: Keep critical sections small

Do not hold locks while doing slow network calls, waiting for user
input, or performing unrelated work.

### Technique 4: Use timed lock acquisition where appropriate

`ReentrantLock` supports `tryLock()` and timed acquisition. A thread can
stop waiting and take a recovery path.

``` java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class TryLockExample {
    private final ReentrantLock lockA = new ReentrantLock();
    private final ReentrantLock lockB = new ReentrantLock();

    public void doWork() throws InterruptedException {
        if (lockA.tryLock(1, TimeUnit.SECONDS)) {
            try {
                if (lockB.tryLock(1, TimeUnit.SECONDS)) {
                    try {
                        System.out.println("Acquired both locks");
                    } finally {
                        lockB.unlock();
                    }
                } else {
                    System.out.println("Could not acquire lockB; retry later");
                }
            } finally {
                lockA.unlock();
            }
        }
    }
}
```

Timed acquisition does not automatically solve every coordination
problem. Design the retry and cleanup behavior carefully, and always
release acquired locks in `finally`.

### Technique 5: Diagnose with thread dumps

A thread dump can show threads blocked on monitors or waiting for locks.
Tools such as `jstack`, JVM monitoring tools, and application
observability can help investigate.

## 5. Starvation --- a thread never gets a fair chance

### Simple analogy

Imagine a queue where new people keep being allowed to move ahead of
you. You are always waiting, even though the system is active.

That is starvation.

A thread is starved when it repeatedly fails to get CPU time or a
resource it needs, while other threads continue making progress.

### Causes

-   Unfair lock acquisition.
-   A thread holding a lock for too long.
-   High-priority threads repeatedly taking resources from
    lower-priority threads.
-   A workload that continually favors some threads.

Java's intrinsic `synchronized` monitor does not promise fair lock
acquisition.

### Fair `ReentrantLock` example

``` java
import java.util.concurrent.locks.ReentrantLock;

public class FairLockExample {
    private final ReentrantLock lock = new ReentrantLock(true);

    public void doWork() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName()
                    + " acquired the lock");
        } finally {
            lock.unlock();
        }
    }
}
```

The `true` argument requests a fair lock policy, which generally favors
the longest-waiting thread.

Important: - Fairness can reduce throughput. - Fair locks do not
guarantee fairness from the operating system scheduler. - A fair lock is
not a universal fix for starvation.

## 6. Livelock --- active threads, but no progress

### Simple analogy

Two people meet in a narrow hallway:

-   Person A moves left to let B pass.
-   Person B also moves left.
-   Both move right.
-   Both move left again.

They are active and reacting, but neither gets past the other.

That is livelock.

### What makes it different from deadlock?

-   Deadlock: threads are blocked and make no progress.
-   Livelock: threads are running and changing state, but keep
    preventing useful progress.

### A conceptual example

Imagine two booking workers that both detect a conflict and immediately
back off, then retry at exactly the same time. They collide again, back
off again, and repeat.

### Ways to reduce livelock

-   Add randomized backoff so retries do not synchronize.
-   Use bounded retries.
-   Introduce a clear priority or coordinator.
-   Make retry policies include jitter.
-   Ensure retries are safe and idempotent.

Random backoff is useful, but it is not a substitute for a correct
concurrency design.

## 7. Compare the three problems

  -----------------------------------------------------------------------
  Problem           What threads do   Is the system     Main idea
                                      active?           
  ----------------- ----------------- ----------------- -----------------
  Deadlock          Wait for one      Usually no        Circular waiting
                    another           progress by the   
                    indefinitely      involved threads  

  Starvation        Some threads keep Other threads     Unfair or
                    losing access to  continue          unbalanced access
                    a resource                          

  Livelock          Keep reacting and Yes, but no       Activity without
                    retrying          useful progress   progress
  -----------------------------------------------------------------------

## 8. How can these problems appear in the Turf backend?

Imagine two operations:

-   Booking operation: locks a slot, then tries to update a payment
    record.
-   Payment operation: locks a payment record, then tries to update the
    slot.

If the two operations acquire database rows in opposite orders, they may
cause a **database deadlock**.

That is related to Java lock deadlock, but it is not the same mechanism:

-   Java monitor deadlock happens inside a JVM.
-   Database deadlock happens inside the database's transaction and lock
    manager.

Practical protections:

1.  Keep database transactions short.
2.  Acquire rows in a consistent order where possible.
3.  Avoid network calls while holding database locks or an open
    transaction.
4.  Use appropriate constraints and transaction isolation.
5.  Make retryable operations idempotent.
6.  Retry transient deadlock failures only in a bounded, deliberate way.
7.  Log and monitor repeated lock contention and transaction failures.

A Java `synchronized` block in one Spring Boot instance cannot protect a
booking from another application instance. Use database-level
correctness mechanisms for shared booking data.

## 9. Interview questions and answers

### Q1. What is deadlock?

A condition where two or more threads wait indefinitely for resources
held by one another, so none can proceed.

### Q2. What are the four necessary conditions for deadlock?

Mutual exclusion, hold and wait, no preemption, and circular wait.

### Q3. How can you prevent deadlock?

Use consistent lock ordering, avoid unnecessary nested locks, keep
critical sections small, and consider timed lock acquisition with a
recovery strategy.

### Q4. What is starvation?

A thread repeatedly fails to obtain the CPU or a resource it needs while
other threads continue to make progress.

### Q5. Does `synchronized` guarantee fairness?

No. Intrinsic monitor acquisition does not promise that waiting threads
acquire the monitor in FIFO order.

### Q6. What is livelock?

Threads remain active and respond to one another, but their repeated
actions prevent useful progress.

### Q7. What is the difference between deadlock and livelock?

In deadlock, threads are stuck waiting. In livelock, threads keep
running and changing state but do not make meaningful progress.

### Q8. Can `ReentrantLock` help with starvation?

A fair `ReentrantLock` can favor longer-waiting threads, but it does not
guarantee overall system fairness and may reduce throughput.

### Q9. Does `tryLock()` automatically prevent deadlock?

No. It can help avoid indefinite waiting if used with timeouts and a
sound retry/release strategy. Poor retry logic can still cause livelock.

### Q10. How do you investigate a Java deadlock?

Capture a thread dump and inspect which threads own and wait for which
monitors or locks. JVM tools can often identify monitor deadlocks.

### Q11. Can a Java monitor protect a Turf booking across multiple Spring Boot instances?

No. A Java monitor coordinates threads only within its JVM. Shared
booking correctness needs database-level or distributed coordination
mechanisms.

### Q12. Are database deadlocks and Java deadlocks identical?

No. The general idea of circular waiting is similar, but the resources
and lock managers are different. Database deadlocks involve database
transactions and database locks.

## 10. Practice questions

1.  Thread A owns `lockA` and waits for `lockB`. Thread B owns `lockB`
    and waits for `lockA`. What problem is this?
2.  Name the four necessary conditions for deadlock.
3.  If a thread waits for a lock while other threads repeatedly acquire
    it first, what problem might occur?
4.  Two threads repeatedly detect a conflict, back off, and retry at the
    same time without completing their work. What problem might this be?
5.  Why should all code paths acquire two shared locks in a consistent
    order?
6.  Does `new ReentrantLock(true)` guarantee that every thread in the
    application will be scheduled fairly?
7.  Why is it dangerous to make an HTTP request while holding a database
    transaction open?
8.  A Java `synchronized` block protects a slot in one Spring Boot
    instance. Is that enough if the application has three instances?
9.  How would you investigate a suspected Java monitor deadlock?
10. What safeguards should surround retries after a database deadlock?

### Practice answers

1.  Deadlock.
2.  Mutual exclusion, hold and wait, no preemption, circular wait.
3.  Starvation.
4.  Livelock.
5.  A consistent order prevents the circular-wait pattern caused by
    opposite lock acquisition.
6.  No. It requests fair lock acquisition for that lock, but does not
    control every scheduling decision or guarantee system-wide fairness.
7.  The request may take a long time or fail, extending lock duration
    and increasing contention and deadlock risk.
8.  No. The other instances have separate JVMs and separate monitors.
9.  Capture and inspect a thread dump, identify lock owners and waiters,
    and examine the lock acquisition paths.
10. Use bounded retries, backoff, idempotency, and monitoring; retry
    only failures known to be transient and safe to retry.

## 11. Key takeaways

-   Deadlock is circular waiting.
-   Starvation is repeatedly being denied a resource or CPU time.
-   Livelock is activity without useful progress.
-   Consistent lock ordering is one of the strongest deadlock-prevention
    techniques.
-   Fair locks can help with some starvation scenarios, with possible
    throughput costs.
-   Backoff and jitter can reduce livelock in retry loops.
-   Database deadlocks need database-aware handling.
-   JVM-local locks do not coordinate multiple Spring Boot instances.
