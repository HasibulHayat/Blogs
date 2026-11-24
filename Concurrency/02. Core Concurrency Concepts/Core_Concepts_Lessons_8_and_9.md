# Java Concurrency --- Core Concepts

# Lesson 8: Happens-Before Relationship

## 1. What is Happens-Before?

The **happens-before relationship** is a rule in Java that tells us when
one thread's actions are guaranteed to be visible to another thread.

In simple words:

> If action A happens-before action B, then the effects of A are
> guaranteed to be visible to B.

It helps Java developers reason about how threads share information.

## 2. Why do we need it?

Imagine Thread A changes a shared variable:

``` java
class SharedData {
    boolean ready = false;
}
```

Thread A:

``` java
data.ready = true;
```

Thread B:

``` java
if (data.ready) {
    System.out.println("Ready!");
}
```

Without a proper visibility guarantee, Thread B is not guaranteed to see
the updated value.

Happens-before rules help establish that guarantee.

## 3. Common happens-before rules

You do not need to memorize every detail immediately. Start with these
important rules.

### Rule 1: Program order

Within one thread, earlier actions happen-before later actions in that
same thread.

``` java
int number = 10;
System.out.println(number);
```

The assignment happens-before the print statement.

### Rule 2: synchronized

Unlocking a monitor happens-before a later lock of the same monitor.

``` java
class SharedData {
    private int number = 0;

    public synchronized void update() {
        number = 10;
    }

    public synchronized int read() {
        return number;
    }
}
```

When one thread exits the synchronized method, its changes become
visible to a thread that later enters a synchronized method using the
same object lock.

### Rule 3: volatile

A write to a `volatile` variable happens-before a later read of that
same variable.

``` java
class SharedData {
    volatile boolean ready = false;
}
```

If Thread A writes:

``` java
data.ready = true;
```

and Thread B later reads:

``` java
if (data.ready) {
    System.out.println("Ready!");
}
```

the volatile rule provides a visibility guarantee for that variable.

### Rule 4: Thread start

Calling `start()` on a thread happens-before the actions inside that
started thread.

``` java
int number = 10;

Thread thread = new Thread(() -> {
    System.out.println(number);
});

thread.start();
```

The actions before `start()` are ordered before the actions in the new
thread.

### Rule 5: Thread join

All actions in a thread happen-before another thread successfully
returns from `join()` on it.

``` java
Thread thread = new Thread(() -> {
    // Do some work
});

thread.start();
thread.join();

// The thread has finished its work here.
```

After `join()` returns, the joining thread is guaranteed to see the
completed thread's actions.

## 4. What does NOT create happens-before?

Calling `Thread.sleep()` does not, by itself, guarantee that another
thread will see your changes.

``` java
data.ready = true;
Thread.sleep(1000);
```

Sleeping is not a replacement for `volatile`, `synchronized`, or another
proper coordination mechanism.

## 5. Why does this matter in the Turf project?

Imagine an administrator enables bookings:

``` java
bookingEnabled = true;
```

Many request-handling threads may read this setting.

If the value is shared between threads, you need a valid visibility
mechanism. For a simple shared flag, `volatile` may be suitable. For
more complex state changes, synchronization or database-backed
coordination may be needed.

Remember: a Java happens-before relationship works within the Java
memory model. It does not coordinate separate Spring Boot application
instances by itself.

## 6. Quick recap

-   Happens-before defines ordering and visibility guarantees between
    actions.
-   Program order applies within a single thread.
-   `synchronized` provides a relationship through the same monitor
    lock.
-   `volatile` provides a relationship between a write and a later read
    of the same variable.
-   `start()` and `join()` also establish happens-before relationships.
-   `sleep()` does not establish one.

## 7. Check your understanding

1.  What does happens-before guarantee?
2.  Does `Thread.sleep()` guarantee visibility of shared data?
3.  What relationship is established by writing to a volatile variable
    and later reading it?
4.  What does `join()` guarantee after it returns?

------------------------------------------------------------------------

# Lesson 9: Instruction Reordering

## 1. What is instruction reordering?

Instruction reordering means the Java compiler, JVM, or CPU may change
the order in which operations are executed to improve performance.

The goal is to make programs run efficiently.

For a single thread, these optimizations should preserve the behavior
that the program is allowed to rely on.

However, when multiple threads share data, reordering can sometimes
produce results that surprise developers.

## 2. A simple example

Consider:

``` java
int number = 10;
boolean ready = true;
```

A developer may expect another thread to always observe `number = 10`
after it sees `ready = true`.

But without proper coordination, Java does not guarantee that another
thread will observe those changes in the expected order.

The problem is not necessarily that the source code lines were written
in the wrong order. The issue is that unsynchronized threads do not have
the visibility and ordering guarantees the developer expects.

## 3. How can we prevent problems?

Use Java's concurrency tools to establish the guarantees your program
needs.

### Using volatile

``` java
class SharedData {
    int number = 0;
    volatile boolean ready = false;

    void publish() {
        number = 10;
        ready = true;
    }

    void read() {
        if (ready) {
            System.out.println(number);
        }
    }
}
```

When one thread calls `publish()` and another thread observes
`ready == true`, the volatile happens-before rule ensures that the
earlier write to `number` is visible to that reader.

### Using synchronized

You can also protect related reads and writes with the same lock:

``` java
class SharedData {
    private int number = 0;

    public synchronized void update() {
        number = 10;
    }

    public synchronized int read() {
        return number;
    }
}
```

Both methods use the same object's monitor, so the synchronization
provides visibility and ordering guarantees.

## 4. Connection to the Java Memory Model

The Java Memory Model (JMM) defines which reorderings are allowed and
what guarantees a program can rely on.

You do not need to control the compiler or CPU yourself.

Your responsibility is to use tools such as:

-   `synchronized`
-   `volatile`
-   Atomic classes
-   Thread-safe concurrent utilities

These tools establish the required ordering and visibility guarantees.

## 5. Why does this matter in the Turf project?

Suppose a booking service updates a shared in-memory flag and then other
request threads read it.

If the reads and writes are not coordinated, another thread may not
observe the state as expected.

For actual booking availability, do not rely only on Java fields or
synchronization inside one application instance. Use database
transactions, constraints, and appropriate locking to protect bookings
across multiple application instances.

## 6. Quick recap

-   Instruction reordering is an optimization performed by the compiler,
    JVM, or CPU.
-   Reordering is generally safe for single-threaded behavior.
-   Unsynchronized shared data can lead to unexpected observations
    between threads.
-   The JMM defines the guarantees Java programs can rely on.
-   `volatile` and `synchronized` establish important visibility and
    ordering guarantees.

## 7. Check your understanding

1.  Why do compilers and CPUs reorder instructions?
2.  Why can instruction reordering matter when multiple threads share
    data?
3.  Which Java keyword can establish visibility and ordering for a
    volatile variable?
4.  Why should a Turf booking system use database-level protection as
    well as Java concurrency tools?

------------------------------------------------------------------------

## Final connection between Lessons 8 and 9

These two lessons are closely related:

-   **Happens-before** describes the ordering and visibility guarantees
    your program can rely on.
-   **Instruction reordering** is one reason those guarantees matter.

You generally do not need to prevent every optimization. You need to
write concurrent code using the correct Java mechanisms so that the Java
Memory Model guarantees the behavior your application requires.
