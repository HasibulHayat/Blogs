# Java Concurrency --- Lesson 9

## `Thread.yield()` in Java

In Lesson 8, we learned about thread priorities and how they provide
hints to the scheduler about a thread's relative importance.

In this lesson, we'll learn about another interesting method:
`Thread.yield()`.

The idea behind `yield()` is simple:

**A thread can voluntarily signal that it is willing to give other
runnable threads an opportunity to execute.**

But there's an important catch: the scheduler is not required to honor
that suggestion.

------------------------------------------------------------------------

## 1. What is `Thread.yield()`?

`Thread.yield()` is a static method in Java's `Thread` class.

It provides a hint to the thread scheduler that the currently executing
thread is willing to yield its current opportunity to execute.

In simpler words, imagine Thread A is executing. When Thread A calls:

``` java
Thread.yield();
```

It is effectively saying:

"Scheduler, I'm willing to let another runnable thread execute if you
want."

The scheduler may choose another thread, but it is not required to.

### Syntax

``` java
Thread.yield();
```

Notice that we call it using `Thread`, not an individual thread object.

`yield()` affects the thread that is currently executing the method.

------------------------------------------------------------------------

## 2. Why does `yield()` exist?

Imagine we have two threads:

-   Thread A performs some background calculation.
-   Thread B performs another background calculation.

Thread A might reach a point where it has no immediate reason to
continue executing.

It can call:

``` java
Thread.yield();
```

This communicates that another runnable thread may use the CPU.

Historically, this was useful as a scheduling hint in certain
cooperative scheduling scenarios.

However, modern operating systems generally use preemptive scheduling,
where the operating system decides when threads receive CPU time.

Therefore, `yield()` is rarely necessary in modern application
development.

------------------------------------------------------------------------

## 3. A basic example

Let's create two threads.

``` java
public class Main {

    public static void main(String[] args) {

        Thread threadA = new Thread(() -> {

            for (int i = 1; i <= 5; i++) {
                System.out.println("Thread A: " + i);

                Thread.yield();
            }

        });

        Thread threadB = new Thread(() -> {

            for (int i = 1; i <= 5; i++) {
                System.out.println("Thread B: " + i);
            }

        });

        threadA.start();
        threadB.start();
    }
}
```

One possible output:

``` text
Thread A: 1
Thread B: 1
Thread B: 2
Thread A: 2
Thread B: 3
Thread A: 3
Thread B: 4
Thread B: 5
Thread A: 4
Thread A: 5
```

Another possible output:

``` text
Thread A: 1
Thread A: 2
Thread A: 3
Thread A: 4
Thread A: 5
Thread B: 1
Thread B: 2
Thread B: 3
Thread B: 4
Thread B: 5
```

Both outputs are possible.

The scheduler might:

-   Run another runnable thread.
-   Continue running the current thread.
-   Choose a different runnable thread.
-   Make a scheduling decision that produces no visible change.

**Never depend on `yield()` to control execution order.**

------------------------------------------------------------------------

## 4. Why is `yield()` a static method?

Look at the syntax again:

``` java
Thread.yield();
```

Why don't we write this?

``` java
threadA.yield();
```

Technically, Java allows calling a static method through an object
reference, but it is discouraged.

The reason `yield()` is static is that it applies to the currently
executing thread, not necessarily the object through which the method
was called.

Consider:

``` java
Thread threadA = new Thread(() -> {

    System.out.println("A is running");

    Thread.yield();

});

Thread threadB = new Thread(() -> {

    System.out.println("B is running");

});

threadA.start();
threadB.start();
```

When `Thread.yield()` executes inside Thread A, it is Thread A that
yields.

It does not cause Thread B to yield.

------------------------------------------------------------------------

## 5. Does `yield()` change the thread's state?

In Lesson 5, we learned about Java's thread lifecycle states:

  State           Meaning
  --------------- -----------------------------------
  NEW             Thread created but not started
  RUNNABLE        Thread is ready or executing
  BLOCKED         Waiting to acquire a monitor lock
  WAITING         Waiting indefinitely
  TIMED_WAITING   Waiting for a specified time
  TERMINATED      Execution completed

When a thread calls:

``` java
Thread.yield();
```

It does not enter `WAITING`, `BLOCKED`, or `TIMED_WAITING`.

It remains in the `RUNNABLE` state.

Why?

Because it has not started waiting for a specific event or resource. It
has simply given the scheduler a hint.

`getState()` is only a snapshot, so it should not be used as a
coordination mechanism.

------------------------------------------------------------------------

## 6. Difference between `sleep()` and `yield()`

This is one of the most important comparisons in this lesson.

  -----------------------------------------------------------------------
  Feature                 `Thread.sleep()`        `Thread.yield()`
  ----------------------- ----------------------- -----------------------
  Purpose                 Pause execution for a   Hint that another
                          time                    thread may execute

  Argument                Requires a duration     No arguments

  Static method           Yes                     Yes

  Current thread affected Yes                     Yes

  Thread state            TIMED_WAITING while     Remains RUNNABLE
                          sleeping                

  Guaranteed pause        Yes, for at least the   No
                          specified duration,     
                          subject to scheduling   
                          and timing precision    

  Guarantees another      No                      No
  thread runs                                     

  Releases monitor locks? No                      No
  -----------------------------------------------------------------------

### Example: sleep

``` java
Thread.sleep(1000);
```

The current thread enters `TIMED_WAITING` for approximately one second.

### Example: yield

``` java
Thread.yield();
```

The current thread remains `RUNNABLE` and simply gives the scheduler a
hint.

Remember:

-   `sleep()` says: "Pause me for a while."
-   `yield()` says: "I'm willing to let someone else run."

------------------------------------------------------------------------

## 7. Does `yield()` work with thread priorities?

In Lesson 8, we learned about thread priorities.

Let's combine those concepts.

``` java
public class Main {

    public static void main(String[] args) {

        Thread threadA = new Thread(() -> {

            for (int i = 1; i <= 5; i++) {
                System.out.println("A: " + i);
                Thread.yield();
            }

        });

        Thread threadB = new Thread(() -> {

            for (int i = 1; i <= 5; i++) {
                System.out.println("B: " + i);
            }

        });

        threadA.setPriority(Thread.MIN_PRIORITY);
        threadB.setPriority(Thread.MAX_PRIORITY);

        threadA.start();
        threadB.start();
    }
}
```

Does Thread B now have to execute first?

No.

Even with a higher priority, Thread B is not guaranteed to execute
first.

The combination of `yield()` and thread priority does not create
deterministic scheduling.

Both are scheduling hints, not execution-order controls.

------------------------------------------------------------------------

## 8. When should we use `yield()`?

In modern Java applications, you will rarely need it.

For example, in our Turf booking backend, imagine:

-   One thread processes booking requests.
-   Another thread generates reports.
-   A third thread sends notifications.

We should not use `yield()` to make booking requests execute before
reports.

Instead, we should use appropriate executors, task queues, and
resource-management strategies.

`yield()` is useful to understand because:

1.  It teaches us about cooperative scheduling hints.
2.  It helps us understand that thread scheduling is controlled by the
    runtime and operating system.
3.  It appears in older Java code and concurrency examples.

But it is not a reliable synchronization or coordination mechanism.

------------------------------------------------------------------------

## 9. Important things to remember

1.  `Thread.yield()` is a static method.
2.  It affects the currently executing thread.
3.  It hints that the scheduler may run another runnable thread.
4.  The scheduler is free to ignore the hint.
5.  It does not guarantee that another thread executes.
6.  It does not guarantee a particular execution order.
7.  The calling thread remains in the `RUNNABLE` state.
8.  It does not release monitor locks.
9.  It is not a replacement for `sleep()`, `wait()`, or proper
    synchronization.
10. Modern application code rarely needs it.

------------------------------------------------------------------------

## 10. Practical exercises

Try these yourself before looking for help.

### Exercise 1 --- Basic yield

Create two threads.

Each thread should print numbers from 1 to 10.

Call `Thread.yield()` after every print in Thread A.

Observe the output.

### Exercise 2 --- Compare with sleep

Create two threads.

-   Thread A calls `Thread.sleep(500)` after every iteration.
-   Thread B calls `Thread.yield()` after every iteration.

Observe how their output differs.

Remember to handle `InterruptedException` when using `sleep()`.

### Exercise 3 --- Check the thread state

Inside a thread, print:

``` java
System.out.println(Thread.currentThread().getState());

Thread.yield();

System.out.println(Thread.currentThread().getState());
```

What do you observe?

Remember that `getState()` is only a snapshot, so it is not a
coordination mechanism.

### Exercise 4 --- Combine priority and yield

Create two threads with different priorities.

Call `yield()` in the lower-priority thread.

Run the program several times.

Does the higher-priority thread always execute first?

------------------------------------------------------------------------

## 11. Knowledge check

1.  What does `Thread.yield()` do?
2.  Which thread does `Thread.yield()` affect?
3.  What is the thread state after calling `yield()`?
4.  What is the key difference between `sleep()` and `yield()`?
5.  Why is `yield()` not a reliable synchronization or coordination
    mechanism?

------------------------------------------------------------------------

## 12. What's next?

### Lesson 10 --- `Thread.interrupt()`

We'll explore `interrupt()`, why it exists, how to request that a thread
stop what it is doing, and how Java handles interrupted threads.

This is an especially important lesson because interruption is one of
Java's main mechanisms for cooperative thread cancellation.
