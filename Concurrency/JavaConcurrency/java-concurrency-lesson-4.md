# Java Concurrency --- Lesson 4: `Thread.join()`

## 1. What problem does `join()` solve?

Sometimes one thread needs to wait until another thread finishes its
work. For example, the main thread may need to wait for a download
thread to finish before processing the downloaded file.

Java provides `join()` for this.

## 2. What does `join()` do?

When a thread calls `anotherThread.join()`, the **calling thread** waits
until `anotherThread` terminates.

``` java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(() -> {
            System.out.println("Thread A started");

            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }

            System.out.println("Thread A finished");
        });

        threadA.start();
        threadA.join();

        System.out.println("All tasks completed");
    }
}
```

Typical output:

``` text
Thread A started

(approximately 3 seconds pass)

Thread A finished
All tasks completed
```

The main thread calls `threadA.join()`, so the main thread waits until
Thread A terminates. Thread A continues doing its work.

## 3. What happens without `join()`?

If we remove `threadA.join()`, the main thread does not wait for Thread
A. It can print `All tasks completed` while Thread A is still working.

``` java
threadA.start();
System.out.println("All tasks completed");
```

The output could be:

``` text
Thread A started
All tasks completed
(approximately 3 seconds pass)
Thread A finished
```

The exact interleaving depends on scheduling.

## 4. Which thread waits?

This is a key point:

``` java
threadA.join();
```

It does **not** make Thread A wait for the main thread. It makes the
thread that executes this line wait for Thread A.

In our example:

1.  Main starts Thread A.
2.  Main calls `threadA.join()`.
3.  Main waits.
4.  Thread A continues executing.
5.  Thread A terminates.
6.  Main resumes after `join()`.

`join()` does not stop the target thread.

## 5. Using `join()` with multiple threads

``` java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(() -> {
            System.out.println("Thread A started");
            sleep(2000);
            System.out.println("Thread A finished");
        });

        Thread threadB = new Thread(() -> {
            System.out.println("Thread B started");
            sleep(3000);
            System.out.println("Thread B finished");
        });

        Thread threadC = new Thread(() -> {
            System.out.println("Thread C started");
            sleep(1000);
            System.out.println("Thread C finished");
        });

        threadA.start();
        threadB.start();
        threadC.start();

        threadA.join();
        threadB.join();
        threadC.join();

        System.out.println("Everything finished!");
    }

    private static void sleep(long milliseconds) {
        try {
            Thread.sleep(milliseconds);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

All three threads are started before the main thread joins them.
Therefore, they can work concurrently.

The main thread waits until each target thread has terminated. Total
elapsed time is approximately the duration of the slowest task (about
three seconds here), plus scheduling and other overhead---not the sum of
all three sleep durations.

The order of start and finish messages is not guaranteed.

## 6. Why might Thread C print first?

``` java
threadA.start();
threadB.start();
threadC.start();
```

This requests that the JVM start the three threads in that order. It
does **not** guarantee that they execute or print in that order.

The JVM and operating-system scheduler determine when each thread gets
CPU time. Thread C may get scheduled first, even though its `start()`
call came third.

Concurrency means their execution can overlap and interleave.

## 7. `join()` does not control the target threads' execution order

``` java
threadA.start();
threadB.start();
threadC.start();

threadA.join();
threadB.join();
threadC.join();
```

All three threads can run concurrently. The joins make the main thread
wait for them to finish.

The order of the join calls does not force the target threads to execute
in that order. Thread C may finish before Thread A, while the main
thread is still waiting in `threadA.join()`.

## 8. Concurrent versus sequential execution

### Concurrent approach

``` java
threadA.start();
threadB.start();
threadC.start();

threadA.join();
threadB.join();
threadC.join();
```

All three threads are started before the main thread waits. They can run
concurrently.

### Sequential approach

``` java
threadA.start();
threadA.join();

threadB.start();
threadB.join();

threadC.start();
threadC.join();
```

Thread B is not started until Thread A terminates. Thread C is not
started until Thread B terminates. The tasks run one after another.

## 9. `sleep()` versus `join()`

  -----------------------------------------------------------------------
  `Thread.sleep()`                    `threadA.join()`
  ----------------------------------- -----------------------------------
  Pauses the currently executing      Makes the calling thread wait for
  thread for a duration               Thread A to terminate

  Takes a time duration               Called on the target thread

  Does not wait for another thread to Waits for a particular thread to
  finish                              finish
  -----------------------------------------------------------------------

``` java
Thread.sleep(3000); // Current thread pauses for approximately 3 seconds.
threadA.join();     // Current thread waits until Thread A terminates.
```

Both methods can throw `InterruptedException`.

## 10. Important details to remember

-   `join()` makes the calling thread wait for the target thread to
    terminate.
-   It does not pause or stop the target thread.
-   If the target thread has already terminated, `join()` returns
    immediately.
-   Starting several threads before joining them allows their work to
    overlap.
-   Calling `start()` in a particular order does not guarantee execution
    or output order.
-   Joining threads one by one after starting them does not make their
    work sequential.
-   Starting and joining each thread before starting the next one makes
    the tasks sequential.
-   `join()` can throw `InterruptedException`.
-   A successful `join()` also establishes a memory-visibility
    guarantee: actions performed by the target thread happen-before the
    calling thread returns from `join()`.

## 11. Practice

### Exercise 1 --- Basic join

Create a thread that: 1. Prints `Downloading started`. 2. Sleeps for
three seconds. 3. Prints `Downloading finished`.

The main thread should print `Processing downloaded file` only after the
download thread finishes. Use `join()`.

### Exercise 2 --- Multiple threads

Create three threads: - Thread A sleeps for two seconds. - Thread B
sleeps for four seconds. - Thread C sleeps for one second.

Start all three, then make the main thread wait for all three using
`join()`. Observe the total execution time.

### Exercise 3 --- Compare execution styles

First, start all three threads and then join them. Next, start and join
each thread before starting the next one. Compare the total elapsed time
and output.

### Check your understanding

**Question 1:** What does `threadA.join()` do when called from the main
thread?

**Question 2:** If three threads are started before the main thread
joins all three, can they run concurrently?

**Question 3:** What happens if the target thread has already terminated
before `join()` is called?

**Question 4:** If Thread A takes five seconds and Thread B takes two
seconds, and both are started before either is joined, approximately how
long until the main thread has joined both?

------------------------------------------------------------------------

**Lesson 4 complete.** In Lesson 5, we will learn the Java thread
lifecycle and its six states: `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`,
`TIMED_WAITING`, and `TERMINATED`.
