# Java Concurrency --- Lesson 10

# `Thread.interrupt()` in Java: A Detailed Guide

`Thread.interrupt()` is one of the most important APIs in Java
concurrency. It is also one of the easiest to misunderstand.

The method's name can make it sound as if Java immediately stops a
thread. That is **not** what it does.

Java uses interruption as a cooperative cancellation mechanism: one
thread requests that another thread stop or change what it is doing, and
the target thread responds at an appropriate point.

This lesson goes deeper than the earlier lessons. We'll cover the
interrupt flag, the two methods for checking it, what happens during
blocking operations, how to handle `InterruptedException`, common
mistakes, and how to design tasks that can shut down safely.

------------------------------------------------------------------------

## 1. The problem: how do we ask a thread to stop?

Suppose our Turf booking backend has a background worker that
periodically checks for expired temporary booking holds.

Conceptually:

``` java
while (true) {
    checkForExpiredBookingHolds();
    waitBeforeCheckingAgain();
}
```

The worker is useful while the application is running. But what happens
when the application is shutting down?

We need a way to ask the worker to stop.

Java's old `Thread.stop()` method attempted to forcefully stop a thread,
but it is unsafe and deprecated because stopping a thread at an
arbitrary point can leave shared data and resources in an inconsistent
state.

Java's preferred approach is cooperative cancellation:

1.  A thread requests cancellation.
2.  The worker becomes aware of the request.
3.  The worker stops at a safe point.
4.  The worker performs any necessary cleanup.
5.  The worker exits normally.

`interrupt()` is a standard way to send that request.

------------------------------------------------------------------------

## 2. What does `Thread.interrupt()` actually do?

The `interrupt()` method is an instance method of `Thread`.

### Syntax

``` java
thread.interrupt();
```

It requests interruption of the thread represented by `thread`.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            System.out.println("Worker is running");
        });

        worker.start();

        worker.interrupt();
    }
}
```

This example demonstrates how to make the request, but it does not
demonstrate a useful cancellation response. The worker does not inspect
its interrupt status or perform an interruptible blocking operation.

The central rule is:

> `interrupt()` requests interruption. It does not forcibly terminate
> the target thread.

If the target thread is doing ordinary computation and never checks its
interrupt status, it may continue running as if nothing happened.

### Who calls it?

Usually, one thread calls `interrupt()` on another thread:

``` java
worker.interrupt();
```

For example:

-   The main thread requests that a worker stop.
-   A shutdown controller requests that background work stop.
-   A task coordinator requests cancellation of a task.

The calling thread does not need to be the main thread.

------------------------------------------------------------------------

## 3. The interrupt status: a flag associated with a thread

Every thread has an interrupt status. You can think of it as a flag that
records whether an interruption request is pending.

At a high level:

``` text
interrupt status = false  -> no pending interrupt request
interrupt status = true   -> an interrupt request is pending
```

Calling `interrupt()` on a live thread normally sets its interrupt
status to `true`, unless the thread is currently in a special
interruptible blocking operation that responds by throwing an exception
or otherwise handling the interruption.

The target thread can inspect this status and decide what to do.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                System.out.println("Working...");
            }

            System.out.println("Worker noticed the request and is exiting.");
        });

        worker.start();

        worker.interrupt();
    }
}
```

The loop condition means:

-   Continue while the current thread has not been interrupted.
-   Exit the loop when the interrupt status becomes `true`.

The exact number of `Working...` lines is unpredictable. The worker
might notice the request before printing, or it might print several
times first.

### Important: the flag does not stop the thread by itself

An interrupt status is information. Java does not automatically
terminate a thread just because the flag is `true`.

Your code must respond to it, or the thread must be in an operation that
has defined interruption behavior.

------------------------------------------------------------------------

## 4. The two methods for checking interruption

Java provides two commonly used methods:

-   `isInterrupted()`
-   `Thread.interrupted()`

Their names are similar, but their behavior is different.

### 4.1 `isInterrupted()`

This is an instance method.

``` java
thread.isInterrupted();
```

It checks the interrupt status of the particular thread object.

It **does not clear** the interrupt status.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            // Work
        });

        worker.start();
        worker.interrupt();

        System.out.println(worker.isInterrupted());
    }
}
```

This will commonly print `true` if the worker is still alive and its
interrupt status has not been cleared. However, a thread can terminate
or clear its status before another thread checks it, so don't use this
example as a timing guarantee.

Inside the worker itself, this is common:

``` java
while (!Thread.currentThread().isInterrupted()) {
    doOneUnitOfWork();
}
```

### 4.2 `Thread.interrupted()`

This is a static method.

``` java
Thread.interrupted();
```

It checks the interrupt status of the **currently executing thread** and
clears that status.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread.currentThread().interrupt();

        System.out.println(Thread.interrupted());
        System.out.println(Thread.interrupted());
    }
}
```

Output:

``` text
true
false
```

Why?

1.  `main` interrupts itself, setting its interrupt status.
2.  The first `Thread.interrupted()` returns `true` and clears the
    status.
3.  The second call returns `false` because the status has already been
    cleared.

### Side-by-side comparison

  Method                     Which thread does it inspect?   Clears the status?
  -------------------------- ------------------------------- --------------------
  `thread.isInterrupted()`   The specified thread            No
  `Thread.interrupted()`     The current thread              Yes

**Practical rule:** Use `isInterrupted()` when you want to check without
consuming the request. Use `Thread.interrupted()` when you intentionally
want to check and clear the current thread's status.

Be especially careful with `Thread.interrupted()` inside loops: calling
it can clear the flag, so a later check may no longer see the original
request.

------------------------------------------------------------------------

## 5. What happens when an interrupted thread is sleeping?

In Lesson 3, we learned about:

``` java
Thread.sleep(10_000);
```

The current thread pauses for the requested duration, unless it is
interrupted or the sleep ends normally.

If another thread interrupts a thread while it is sleeping, `sleep()`
throws `InterruptedException`.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            try {
                System.out.println("Worker is going to sleep.");

                Thread.sleep(10_000);

                System.out.println("Worker woke up normally.");

            } catch (InterruptedException e) {
                System.out.println("Worker was interrupted while sleeping.");
            }
        });

        worker.start();

        try {
            Thread.sleep(1_000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        worker.interrupt();
    }
}
```

A typical output is:

``` text
Worker is going to sleep.
Worker was interrupted while sleeping.
```

The worker does not need to wait for the full ten seconds. The
interruption wakes it by causing `sleep()` to throw.

### What happens to the interrupt status?

When `sleep()` throws `InterruptedException`, the interrupt status is
cleared as part of that behavior.

So, inside this catch block:

``` java
catch (InterruptedException e) {
    System.out.println(Thread.currentThread().isInterrupted());
}
```

the result is normally:

``` text
false
```

This detail is essential when writing correct interruption-handling
code.

------------------------------------------------------------------------

## 6. Which operations respond to interruption?

Not every Java operation reacts to interruption in the same way.

Some blocking methods have a defined `InterruptedException` response.
Common examples include:

-   `Thread.sleep(...)`
-   `Object.wait(...)`
-   `Thread.join(...)`

If a thread is interrupted while blocked in one of these methods, the
method throws `InterruptedException`, and the interrupt status is
cleared.

We have already studied `sleep()` and `join()`. We'll study `wait()`
later when we learn about object monitors and coordination.

Other APIs have their own interruption behavior. For example,
interrupting a thread blocked on an interruptible NIO channel can close
the channel and cause a `ClosedByInterruptException`. A thread blocked
in a `Selector` can be made ready to return from selection.

For ordinary computation, such as this:

``` java
while (true) {
    calculateSomething();
}
```

there is no automatic exception merely because another thread calls
`interrupt()`.

The computation must check its interrupt status, or use an interruptible
operation.

### A useful mental model

  -----------------------------------------------------------------------
  What the target thread is doing     Typical response to interruption
  ----------------------------------- -----------------------------------
  Ordinary computation                Interrupt status is set; code must
                                      check it

  `Thread.sleep(...)`                 Throws `InterruptedException`;
                                      status is cleared

  `Object.wait(...)`                  Throws `InterruptedException`;
                                      status is cleared

  `Thread.join(...)`                  Throws `InterruptedException`;
                                      status is cleared

  Certain interruptible I/O           API-specific response
  operations                          
  -----------------------------------------------------------------------

Do not assume every blocking API responds in exactly the same way. Check
the documentation for the operation you are using.

------------------------------------------------------------------------

## 7. Why does `InterruptedException` clear the interrupt status?

This behavior can initially feel surprising.

Suppose a thread is sleeping and another thread interrupts it.

The sleep operation throws `InterruptedException`. The exception is the
method's way of telling the thread:

"Your wait was interrupted."

Java clears the interrupt status when throwing this exception so that
the code handling the exception can decide what to do next.

That creates a responsibility for the application:

-   Should the current method propagate the interruption?
-   Should it finish handling cancellation and exit?
-   Should it restore the interrupt status so higher-level code can see
    the request?

This is why simply catching the exception and ignoring it can be a bug.

------------------------------------------------------------------------

## 8. Handling `InterruptedException` correctly

There are two common approaches.

### Approach A: propagate the exception

If your method can declare `throws InterruptedException`, often the
cleanest choice is to let the exception propagate.

``` java
public static void performTask() throws InterruptedException {
    System.out.println("Task started");

    Thread.sleep(5_000);

    System.out.println("Task completed");
}
```

The caller decides how to handle the interruption:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            try {
                performTask();
            } catch (InterruptedException e) {
                System.out.println("Task cancelled.");
            }
        });

        worker.start();
    }

    static void performTask() throws InterruptedException {
        System.out.println("Performing task...");
        Thread.sleep(5_000);
        System.out.println("Task completed.");
    }
}
```

This keeps the lower-level method simple and lets a higher-level method
make the cancellation decision.

### Approach B: restore the interrupt status

Sometimes your method cannot propagate `InterruptedException`, or its
responsibility is to handle cancellation locally.

If you cannot propagate the exception and you are not fully handling the
cancellation, restore the interrupt status:

``` java
try {
    Thread.sleep(5_000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

Why restore it?

Because the exception cleared the status. Calling `interrupt()` on the
current thread sets it again, allowing higher-level code to detect the
request.

A more complete example:

``` java
public class Worker {
    public void runTask() {
        try {
            doWork();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        if (Thread.currentThread().isInterrupted()) {
            System.out.println("Interruption is still signalled.");
        }
    }

    private void doWork() throws InterruptedException {
        Thread.sleep(5_000);
    }
}
```

### When might you not restore it?

If the current method deliberately handles cancellation completely and
is about to exit, it may not need to restore the status. The key is not
to accidentally consume an interruption that a caller or higher-level
component still needs to know about.

### A practical rule

When you catch `InterruptedException`, do not automatically ignore it.

Choose deliberately:

1.  Propagate it if your method can declare it.
2.  If you cannot propagate it, restore the interrupt status when
    higher-level code still needs to see the request.
3.  If you fully handle cancellation and exit, make that behavior
    explicit.

------------------------------------------------------------------------

## 9. Designing a computation loop that responds to interruption

A thread that performs ordinary computation should check its interrupt
status at suitable points.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            long counter = 0;

            while (!Thread.currentThread().isInterrupted()) {
                counter++;

                if (counter % 1_000_000 == 0) {
                    System.out.println("Processed: " + counter);
                }
            }

            System.out.println("Worker is stopping.");
        });

        worker.start();

        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        worker.interrupt();
    }
}
```

The worker periodically checks whether cancellation has been requested.

In a real application, don't necessarily check on every single CPU
instruction. Choose safe and sensible cancellation checkpoints---for
example, between records, batches, or independent units of work.

The important thing is that the task must not spend an unlimited amount
of time doing work without checking for cancellation if prompt
cancellation is required.

------------------------------------------------------------------------

## 10. A complete example: a stoppable booking-cleanup worker

Let's build a slightly more realistic example for our Turf booking
system.

The worker checks for expired temporary booking holds, then sleeps
before checking again.

``` java
public class BookingCleanupWorker {

    public static void main(String[] args) {
        Thread cleanupThread = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    System.out.println("Checking expired booking holds...");

                    // Simulate one cleanup operation.
                    Thread.sleep(2_000);

                } catch (InterruptedException e) {
                    System.out.println("Cleanup worker received a stop request.");

                    // Restore the status so the loop condition can observe it.
                    Thread.currentThread().interrupt();
                }
            }

            System.out.println("Cleanup worker stopped safely.");
        }, "booking-cleanup");

        cleanupThread.start();

        // Simulate the application running for a short period.
        try {
            Thread.sleep(7_000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // Request that the cleanup worker stop.
        cleanupThread.interrupt();
    }
}
```

### Walk through the sequence

1.  The main thread creates and starts `cleanupThread`.
2.  The worker checks for expired booking holds.
3.  The worker sleeps for two seconds.
4.  The main thread eventually calls `cleanupThread.interrupt()`.
5.  If the worker is sleeping, `Thread.sleep()` throws
    `InterruptedException`.
6.  The catch block restores the worker's interrupt status.
7.  The loop condition checks `isInterrupted()` and becomes false.
8.  The worker exits the loop and prints its final message.

This demonstrates cooperative cancellation: the worker responds to a
request and exits through its normal control flow.

In a real Spring Boot service, we would normally use managed executors
and application lifecycle hooks rather than manually managing every
worker thread. We'll get to those tools later.

------------------------------------------------------------------------

## 11. Important correction: interrupting a thread before it starts

There is a subtle but important lifecycle detail.

A thread is not considered alive before `start()` has been called. The
`Thread.interrupt()` API specifies that interrupting a thread that is
not alive has no effect.

Therefore, this is **not** a reliable way to pre-cancel a newly
constructed thread:

``` java
Thread worker = new Thread(() -> {
    System.out.println(
        Thread.currentThread().isInterrupted()
    );
});

worker.interrupt(); // Thread has not started; request has no effect.
worker.start();
```

Do not expect the worker to start with its interrupt status set because
of that call.

If you need a task to be cancellable before it begins, use an explicit
cancellation flag or a higher-level task abstraction designed for
cancellation. We'll study those approaches later.

Also remember:

-   Interrupting a live thread requests interruption.
-   Interrupting a thread that has already terminated has no effect.
-   Interruption does not restart a terminated thread.

------------------------------------------------------------------------

## 12. Common mistakes to avoid

### Mistake 1: assuming `interrupt()` kills a thread

``` java
worker.interrupt();
```

This sends a request. It does not force the thread to stop.

**Better:** Make the worker check its interrupt status or use
interruptible blocking operations.

### Mistake 2: swallowing `InterruptedException`

``` java
try {
    Thread.sleep(5_000);
} catch (InterruptedException e) {
    // Do nothing
}
```

This discards the cancellation signal.

**Better:** Propagate the exception, restore the status, or deliberately
complete cancellation and exit.

### Mistake 3: using `Thread.interrupted()` without realizing it clears the flag

``` java
if (Thread.interrupted()) {
    // The status has already been cleared.
}
```

**Better:** Use `isInterrupted()` when you only want to inspect the
status.

### Mistake 4: expecting normal computation to stop automatically

``` java
while (true) {
    performCalculation();
}
```

Calling `interrupt()` from another thread does not automatically break
this loop.

**Better:** Add interruption checks at sensible cancellation points.

### Mistake 5: using `sleep()` as a cancellation mechanism

Sleeping for a long time just to make a thread periodically check a flag
is often unnecessary. Interruption can wake a thread from `sleep()`, but
for more complex coordination, use the appropriate concurrency
utilities.

------------------------------------------------------------------------

## 13. `interrupt()` compared with the methods we've learned

  -----------------------------------------------------------------------
  Method                  Main purpose            What it does to the
                                                  current/target thread
  ----------------------- ----------------------- -----------------------
  `start()`               Begin a new thread      Starts execution

  `sleep()`               Pause for a duration    Enters `TIMED_WAITING`

  `join()`                Wait for another thread Calling thread waits
                          to finish               

  `yield()`               Offer a scheduling hint Remains `RUNNABLE`

  `interrupt()`           Request interruption    Sets interrupt status
                                                  or triggers an
                                                  interruption response
  -----------------------------------------------------------------------

One distinction is especially important:

-   `join()` makes the calling thread wait for another thread.
-   `interrupt()` asks a target thread to respond to cancellation.

They solve different problems.

------------------------------------------------------------------------

## 14. Practical exercises

Try these exercises in order. The goal is to understand the behavior,
not just reproduce the output.

### Exercise 1 --- Interrupt a computation loop

Create a worker thread that prints `"Working..."` continuously.

1.  Start the worker.
2.  Wait briefly in the main thread.
3.  Call `worker.interrupt()`.
4.  First, run the worker without checking interruption.
5.  Then add `isInterrupted()` and make it exit when the request is
    detected.

Observe the difference.

### Exercise 2 --- Interrupt a sleeping thread

Create a worker that calls `Thread.sleep(10_000)`.

Interrupt it after approximately one second.

Questions:

-   Does it sleep for the full ten seconds?
-   Which exception is thrown?
-   What is its interrupt status inside the catch block?

### Exercise 3 --- Compare the two status-checking methods

In the main thread, run:

``` java
Thread.currentThread().interrupt();

System.out.println(Thread.currentThread().isInterrupted());
System.out.println(Thread.interrupted());
System.out.println(Thread.currentThread().isInterrupted());
```

Predict the output before running it.

### Exercise 4 --- Restore the interrupt status

Write a method that calls `Thread.sleep(5_000)` but cannot declare
`throws InterruptedException`.

Catch the exception and restore the current thread's interrupt status.

Then check the status after the method returns.

### Exercise 5 --- Make the cleanup worker stoppable

Adapt the booking-cleanup example:

-   Give the thread the name `"booking-cleanup"`.
-   Let it perform three cleanup cycles if it is not interrupted.
-   Make it stop promptly if interrupted while sleeping.
-   Print a final message when it exits.

### Exercise 6 --- Test interruption before start

Create a thread, call `interrupt()` before `start()`, and then start it.

Print its interrupt status from inside the thread.

Compare that result with interrupting the thread after it has started
and is still alive.

------------------------------------------------------------------------

## 15. Knowledge check

1.  Does `interrupt()` forcibly terminate a thread?
2.  What does `isInterrupted()` return, and does it clear the status?
3.  What is the difference between `isInterrupted()` and
    `Thread.interrupted()`?
4.  What happens when a thread is interrupted while sleeping?
5.  Does catching `InterruptedException` preserve the interrupt status?
6.  Why might a method restore the interrupt status inside a catch
    block?
7.  How can a computation-heavy thread respond to interruption?
8.  Does calling `interrupt()` on a thread before it starts reliably
    pre-cancel it?
9.  Why is cooperative cancellation safer than forcibly stopping a
    thread?

------------------------------------------------------------------------

## 16. Final summary

The most important idea in this lesson is:

**Interruption is a cancellation request, not a forced shutdown
command.**

Remember these rules:

-   `interrupt()` requests interruption of a target thread.
-   `isInterrupted()` checks a specific thread's status without clearing
    it.
-   `Thread.interrupted()` checks the current thread's status and clears
    it.
-   `sleep()`, `wait()`, and `join()` respond to interruption by
    throwing `InterruptedException` and clearing the interrupt status.
-   Ordinary computation must check for interruption if it should
    support cancellation.
-   When handling `InterruptedException`, propagate it or deliberately
    handle cancellation; restore the status when higher-level code still
    needs to see the request.
-   Interrupting a thread that is not alive has no effect.
-   Never treat interruption as a guarantee that a thread has already
    stopped.

------------------------------------------------------------------------

## 17. What's next?

### Lesson 11 --- Daemon Threads

We'll learn:

-   What daemon threads are.
-   How daemon threads differ from normal user threads.
-   How to call `setDaemon(true)` and `isDaemon()`.
-   Why the JVM can exit when only daemon threads remain.
-   Why daemon threads are not a safe choice for work that must
    complete.
-   How daemon threads relate to background services and application
    shutdown.

This will complete another important part of our Java `Thread` API
fundamentals.
