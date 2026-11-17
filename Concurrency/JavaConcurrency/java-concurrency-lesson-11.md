# Java Concurrency --- Lesson 11

## Daemon Threads in Java

In Lesson 10, we learned how to request that a thread stop using
`interrupt()`. In this lesson, we'll learn about **daemon threads** and
how they affect a fundamental part of Java: when the JVM decides that an
application can exit.

## 1. What is a daemon thread?

A daemon thread is a thread that does not prevent the JVM from shutting
down.

Java threads are commonly described as either:

-   **User threads (non-daemon threads):** Keep the JVM alive while they
    remain alive.
-   **Daemon threads:** Do not keep the JVM alive.

For example:

  Thread                Type     Responsibility
  --------------------- -------- -----------------------------------------
  Main thread           User     Runs the application
  Booking worker        User     Processes bookings
  Notification worker   User     Sends notifications
  Monitoring thread     Daemon   Collects optional background statistics

When no live user threads remain, the JVM can begin its shutdown process
even if daemon threads are still running.

**The most important idea:** daemon status affects JVM shutdown
behavior. It does not make a thread less capable of executing Java code.

## 2. Why do daemon threads exist?

Imagine our Turf booking application has a background task that
periodically collects non-critical monitoring information:

-   Records diagnostic statistics.
-   Monitors application activity.
-   Performs best-effort housekeeping.

If the application is shutting down, we may not need to keep the JVM
alive just so this optional monitoring task can finish.

Daemon threads are useful for background activities that should not
independently keep the application alive.

However, **daemon threads are not suitable for important work that must
finish reliably**. If the JVM exits, daemon threads are not guaranteed
to finish their current operation or execute cleanup code.

A task being performed in the background does not automatically mean it
should be a daemon.

## 3. Creating a daemon thread: `setDaemon()`

Java provides:

``` java
thread.setDaemon(true);
```

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread backgroundThread = new Thread(() -> {
            System.out.println("Background task running...");
        });

        backgroundThread.setDaemon(true);
        backgroundThread.start();
    }
}
```

The order matters:

1.  Create the thread.
2.  Set its daemon status.
3.  Start it.

You must call `setDaemon()` **before** `start()`.

This is invalid:

``` java
Thread worker = new Thread(() -> System.out.println("Working..."));

worker.start();
worker.setDaemon(true); // Throws IllegalThreadStateException
```

Once a thread has started, its daemon status cannot be changed. Calling
`setDaemon()` after startup throws `IllegalThreadStateException`.

You can also explicitly create a user thread with:

``` java
thread.setDaemon(false);
```

## 4. Checking daemon status: `isDaemon()`

Use:

``` java
thread.isDaemon();
```

It returns `true` for a daemon thread and `false` for a user thread.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            System.out.println("Working...");
        });

        worker.setDaemon(true);

        System.out.println("Is daemon? " + worker.isDaemon());

        worker.start();
    }
}
```

Possible output:

``` text
Is daemon? true
Working...
```

You can inspect the currently executing thread too:

``` java
Thread.currentThread().isDaemon();
```

The main thread is normally a user thread, so this normally prints
`false`:

``` java
System.out.println(Thread.currentThread().isDaemon());
```

## 5. The most important concept: JVM shutdown behavior

### Example A --- A user thread keeps the JVM alive

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            for (int i = 1; i <= 5; i++) {
                System.out.println("Worker: " + i);

                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });

        worker.start();

        System.out.println("Main thread finished.");
    }
}
```

Possible output:

``` text
Main thread finished.
Worker: 1
Worker: 2
Worker: 3
Worker: 4
Worker: 5
```

The main thread can finish before the worker. The JVM normally remains
alive because the worker is a live user thread.

### Example B --- A daemon thread does not keep the JVM alive

Now mark the worker as a daemon:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            for (int i = 1; i <= 100; i++) {
                System.out.println("Worker: " + i);

                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });

        worker.setDaemon(true);
        worker.start();

        System.out.println("Main thread finished.");
    }
}
```

The output is nondeterministic. The worker might print once, several
times, or not at all before the JVM exits.

Why?

1.  The main thread is a user thread.
2.  The worker is a daemon thread.
3.  The main thread finishes.
4.  If no other live user threads remain, the JVM may exit.
5.  The daemon thread does not keep the JVM alive.

Do not rely on a particular output order or number of iterations.

## 6. Does a daemon thread stop as soon as `main()` finishes?

No. This is a common misunderstanding.

The JVM does not exit merely because the main thread finishes. It can
exit when no live user threads remain, subject to other JVM shutdown
mechanisms.

  Situation                                   Typical JVM behavior
  ------------------------------------------- -----------------------
  Main finishes, user worker remains alive    JVM stays alive
  Main finishes, only daemon workers remain   JVM may exit
  Main and all other threads finish           JVM can exit normally

The key condition is whether any live user threads remain---not simply
whether `main()` has returned.

## 7. Do daemon threads inherit daemon status?

Yes. A newly created thread normally inherits its daemon status from the
thread that creates it.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread daemonParent = new Thread(() -> {
            System.out.println(
                "Parent daemon: " + Thread.currentThread().isDaemon()
            );

            Thread child = new Thread(() -> {
                System.out.println(
                    "Child daemon: " + Thread.currentThread().isDaemon()
                );
            });

            child.start();
        });

        daemonParent.setDaemon(true);
        daemonParent.start();
    }
}
```

Possible output:

``` text
Parent daemon: true
Child daemon: true
```

The child inherits the daemon status of its creator. This is similar to
the priority inheritance we discussed in Lesson 8.

### Can a daemon thread create a user thread?

Yes. The child can explicitly be configured as a user thread:

``` java
Thread daemonParent = new Thread(() -> {
    Thread child = new Thread(() -> {
        System.out.println("Child is running");
    });

    child.setDaemon(false);
    child.start();
});

daemonParent.setDaemon(true);
daemonParent.start();
```

The child is a user thread. If it remains alive, it can keep the JVM
alive.

The daemon status of the parent does not force every descendant to
remain a daemon.

## 8. What happens to `finally` blocks in daemon threads?

Consider:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            try {
                while (true) {
                    System.out.println("Daemon working...");
                }
            } finally {
                System.out.println("Cleanup completed!");
            }
        });

        worker.setDaemon(true);
        worker.start();
    }
}
```

Can we guarantee that `Cleanup completed!` prints?

**No.**

When the JVM exits because no user threads remain, daemon threads may
not complete their normal execution. Their `finally` blocks are not
guaranteed to run.

Do not depend on daemon-thread cleanup to:

-   Save important data.
-   Close critical resources.
-   Commit transactions.
-   Complete payments.
-   Finish important booking operations.

If work must finish reliably, use a controlled shutdown process and wait
for the work to complete.

## 9. User threads vs daemon threads

  -----------------------------------------------------------------------
  Feature                 User thread             Daemon thread
  ----------------------- ----------------------- -----------------------
  Keeps JVM alive while   Yes                     No
  alive?                                          

  Can execute normal Java Yes                     Yes
  code?                                           

  Can be interrupted?     Yes                     Yes

  Can create child        Yes                     Yes
  threads?                                        

  Main thread's default   User                    No
  type?                                           

  Can finish normally?    Yes                     Yes

  Guaranteed to finish    JVM remains alive while No
  before JVM exits?       user threads remain,    
                          barring other shutdown  
                          mechanisms              

  Suitable for critical   Can be, with proper     No
  work by itself?         lifecycle management    
  -----------------------------------------------------------------------

Daemon threads are not weaker or slower. They simply have different JVM
shutdown behavior.

## 10. When should we use daemon threads?

Daemon threads can be appropriate for genuinely optional or best-effort
activities, such as:

-   Background monitoring.
-   Non-critical periodic statistics.
-   Diagnostic tasks.
-   Certain runtime support activities.

But a background task is not necessarily optional.

For example, sending a payment confirmation email may happen in the
background, but the application may need reliable delivery, persistence,
and retries. A daemon thread alone is not an appropriate reliability
strategy.

## 11. Should we use daemon threads in our Turf booking system?

  -----------------------------------------------------------------------
  Task                    Daemon-only?            Reason
  ----------------------- ----------------------- -----------------------
  Booking persistence     No                      Booking data must be
                                                  reliably saved

  Payment processing      No                      Financial operations
                                                  need controlled
                                                  completion

  Booking confirmation    No                      Needs reliable delivery
  delivery                                        and retries

  Daily analytics report  Depends                 Could be best-effort or
                                                  require guaranteed
                                                  completion

  Debug monitoring        Potentially             It may be acceptable to
                                                  lose during shutdown

  Critical database       No                      Must follow a
  cleanup                                         controlled lifecycle
  -----------------------------------------------------------------------

In a Spring Boot application, we will generally prefer managed executors
and application lifecycle management over manually creating every worker
thread.

This gives us control over graceful shutdown, in-flight work,
cancellation, rejected tasks, and resource cleanup. We'll study these
tools later.

## 12. Common mistakes to avoid

### Mistake 1 --- Thinking daemon means low priority

Daemon status and thread priority are unrelated. A daemon thread can
have high priority, and a user thread can have low priority.

### Mistake 2 --- Setting daemon status after starting

``` java
worker.start();
worker.setDaemon(true); // IllegalThreadStateException
```

Set daemon status before starting the thread.

### Mistake 3 --- Assuming main-thread completion means JVM shutdown

The JVM can remain alive while any user thread is still running.

### Mistake 4 --- Using daemon threads for payment or booking operations

Daemon threads may not finish before JVM shutdown. Critical operations
need controlled lifecycle and reliable processing.

### Mistake 5 --- Assuming daemon threads always stop immediately when main finishes

The JVM can exit when no live user threads remain. The main thread
finishing is not, by itself, the shutdown condition.

## 13. Practical exercises

Try these yourself.

### Exercise 1 --- Check the main thread

Print:

``` java
Thread.currentThread().isDaemon();
```

What does it return?

### Exercise 2 --- Create a daemon thread

Create a thread, mark it as daemon, and print its daemon status.

### Exercise 3 --- Test JVM shutdown

Create a user thread that sleeps for five seconds. Let the main method
finish immediately. Observe whether the JVM waits for the worker.

Then change the worker into a daemon thread and repeat. Compare the
behavior.

### Exercise 4 --- Test daemon inheritance

Create a daemon parent thread. Inside it, create another thread without
setting its daemon status. Check whether the child is a daemon.

### Exercise 5 --- Explicitly create a user child

Create a daemon parent. Inside it, create a child and call:

``` java
child.setDaemon(false);
```

Check the child's daemon status.

### Exercise 6 --- Test `setDaemon()` after start

Start a thread and then call `setDaemon(true)`. Observe the exception.

## 14. Knowledge check

1.  What makes a daemon thread different from a user thread?
2.  When must `setDaemon(true)` be called?
3.  What happens when main finishes but a user thread is still alive?
4.  What daemon status does a new thread normally inherit?
5.  Should a daemon thread be responsible for a payment operation that
    must complete reliably?
6.  Why should you not rely on a daemon thread's `finally` block for
    critical cleanup?

## 15. Final summary

1.  Java has user threads and daemon threads.
2.  User threads keep the JVM alive while they are running.
3.  Daemon threads do not keep the JVM alive.
4.  Use `setDaemon(true)` to mark a thread as daemon.
5.  Call `setDaemon()` before `start()`.
6.  Use `isDaemon()` to inspect daemon status.
7.  New threads normally inherit daemon status from their creator.
8.  The JVM can exit when no live user threads remain, even if daemon
    threads are still running.
9.  Daemon threads are not guaranteed to complete their work or cleanup
    during JVM shutdown.
10. Never rely on daemon threads for critical business operations.

------------------------------------------------------------------------

## 16. What's next?

### Lesson 12 --- Uncaught Exception Handling

We'll learn:

-   What happens when a thread throws an exception that nobody catches.
-   Why an exception in one thread does not normally crash all other
    threads.
-   How to use `Thread.UncaughtExceptionHandler`.
-   How to use `setUncaughtExceptionHandler()`.
-   How to configure a default uncaught exception handler.
-   How to build better error reporting for background workers.

This is another important step toward writing production-quality
multithreaded applications.
