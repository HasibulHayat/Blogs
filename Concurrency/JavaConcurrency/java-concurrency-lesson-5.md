# Java Concurrency --- Lesson 5

## Thread Lifecycle and Thread States

In the previous four lessons, we learned how to create threads, assign
tasks to them, pause them using `sleep()`, and wait for them to finish
using `join()`.

Now we'll explore what happens to a thread from the moment it is created
until it finishes executing.

## 1. What is the lifecycle of a thread?

A thread has a lifecycle that describes the different states it can go
through during execution.

For example:

1.  We create a `Thread` object.
2.  The thread has not started yet.
3.  We call `start()`.
4.  The thread becomes eligible to execute.
5.  The thread executes its task.
6.  The task finishes.
7.  The thread terminates.

Java represents these stages using **thread states**.

## 2. The six Java thread states

Java defines six official thread states through the `Thread.State` enum:

1.  `NEW`
2.  `RUNNABLE`
3.  `BLOCKED`
4.  `WAITING`
5.  `TIMED_WAITING`
6.  `TERMINATED`

### A. NEW

A thread is in the `NEW` state when its `Thread` object has been created
but `start()` has not yet been called.

``` java
Thread myThread = new Thread(() -> {
    System.out.println("Hello");
});

System.out.println(myThread.getState()); // NEW
```

Remember:

-   The `Thread` object has been created.
-   Its execution has not started.
-   Calling `start()` moves it out of the `NEW` state.

### B. RUNNABLE

A thread enters the `RUNNABLE` state after its `start()` method is
called.

``` java
Thread myThread = new Thread(() -> {
    while (true) {
        // Keep running
    }
});

myThread.start();
```

In Java, `RUNNABLE` represents both:

-   A thread that is ready to run and waiting for CPU time.
-   A thread that is currently executing on the CPU.

Java does not have a separate `RUNNING` state in its official
`Thread.State` enum. The JVM and operating-system scheduler decide when
a runnable thread gets CPU time.

Remember:

-   A thread enters `RUNNABLE` after `start()`.
-   It may be waiting for CPU time or actively executing.
-   `RUNNABLE` does not guarantee that the thread is currently using the
    CPU.

### C. BLOCKED

A thread enters the `BLOCKED` state when it is waiting to acquire a
monitor lock.

The following example uses `synchronized`, which we'll study in detail
in a later lesson.

``` java
public class Main {

    static synchronized void printMessage() {
        System.out.println("Inside synchronized method");

        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return;
        }
    }

    public static void main(String[] args) {
        Thread threadA = new Thread(Main::printMessage);
        Thread threadB = new Thread(Main::printMessage);

        threadA.start();
        threadB.start();
    }
}
```

One possible sequence:

1.  Thread A enters `printMessage()` and acquires the class monitor
    lock.
2.  Thread A sleeps for five seconds while holding the lock.
3.  Thread B attempts to enter `printMessage()`.
4.  Thread B cannot acquire the lock because Thread A owns it.
5.  Thread B enters the `BLOCKED` state.
6.  When Thread A exits the synchronized method, Thread B can acquire
    the lock.

Remember:

-   `BLOCKED` means waiting to acquire a monitor lock.
-   It commonly happens when another thread owns a required
    `synchronized` lock.
-   It is different from sleeping or waiting for another thread to
    finish.

### D. WAITING

A thread enters the `WAITING` state when it waits indefinitely for
another thread to perform a particular action.

Operations that can cause this state include:

-   `Object.wait()`
-   `Thread.join()` without a timeout
-   `LockSupport.park()`

Example using `join()`:

``` java
public class Main {

    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        threadA.start();

        try {
            threadA.join();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Here:

1.  Main starts Thread A.
2.  Thread A begins sleeping.
3.  Main calls `threadA.join()`.
4.  Main waits for Thread A to terminate.
5.  Main may enter the `WAITING` state.

Thread A is still doing its own work, although it is temporarily in
`TIMED_WAITING` because of `sleep()`.

Remember:

-   `WAITING` means waiting indefinitely for another action.
-   `join()` without a timeout can cause this state.
-   `Object.wait()` and `LockSupport.park()` can also cause this state.
-   A waiting thread does not consume CPU time while it is waiting.

### E. TIMED_WAITING

A thread enters the `TIMED_WAITING` state when it waits for a specified
amount of time.

Operations that can cause this state include:

-   `Thread.sleep(milliseconds)`
-   `Thread.join(milliseconds)`
-   `Object.wait(milliseconds)`
-   `LockSupport.parkNanos()`
-   `LockSupport.parkUntil()`

Example:

``` java
Thread myThread = new Thread(() -> {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

myThread.start();
```

While the thread is sleeping, its state is `TIMED_WAITING`.

After the requested waiting period ends, it can become runnable again.

Remember:

-   `TIMED_WAITING` means waiting for a specified duration.
-   `sleep()` causes this state.
-   Timed versions of `join()` and `wait()` can also cause this state.
-   The thread is not guaranteed to resume at the exact instant the
    duration expires.

### F. TERMINATED

A thread enters the `TERMINATED` state when its execution has finished.

``` java
public class Main {

    public static void main(String[] args) throws InterruptedException {
        Thread myThread = new Thread(() -> {
            System.out.println("Thread is executing");
        });

        myThread.start();
        myThread.join();

        System.out.println(myThread.getState());
    }
}
```

Possible output:

``` text
Thread is executing
TERMINATED
```

Remember:

-   `TERMINATED` means the thread's execution has finished.
-   A terminated thread cannot be started again.
-   Calling `start()` on the same `Thread` object a second time throws
    `IllegalThreadStateException`.

## 3. Understanding the complete lifecycle

Here is a summary of the six states:

  State             Meaning
  ----------------- -----------------------------------------
  `NEW`             Thread object created, not started
  `RUNNABLE`        Thread started and eligible to execute
  `BLOCKED`         Waiting to acquire a monitor lock
  `WAITING`         Waiting indefinitely for another action
  `TIMED_WAITING`   Waiting for a specified duration
  `TERMINATED`      Thread execution finished

These states do not follow one fixed sequence.

For example, a simple thread might go through:

``` text
NEW → RUNNABLE → TERMINATED
```

A thread that sleeps might go through:

``` text
NEW → RUNNABLE → TIMED_WAITING → RUNNABLE → TERMINATED
```

A thread that needs a lock might go through:

``` text
NEW → RUNNABLE → BLOCKED → RUNNABLE → TERMINATED
```

The transitions depend on what the thread is doing.

## 4. What is `Thread.State`?

Java represents thread states using an enum inside the `Thread` class:

``` java
Thread.State
```

The enum contains these six constants:

``` java
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED
}
```

An enum is a Java type containing a fixed set of constants.

We can declare a variable of this type:

``` java
Thread.State currentState;
```

We can assign a state:

``` java
currentState = Thread.State.NEW;
```

And compare states:

``` java
if (currentState == Thread.State.RUNNABLE) {
    System.out.println("Thread is runnable");
}
```

Enum constants are commonly compared using `==`.

## 5. What is `getState()`?

Java provides the `getState()` method to inspect a thread's state.

``` java
myThread.getState();
```

It returns a value of type `Thread.State`.

Example:

``` java
public class Main {

    public static void main(String[] args) {
        Thread myThread = new Thread(() -> {
            System.out.println("Hello from thread");
        });

        System.out.println(myThread.getState());

        myThread.start();

        System.out.println(myThread.getState());
    }
}
```

The first output is guaranteed to be:

``` text
NEW
```

The second output is not guaranteed. It might be `RUNNABLE`, or the
thread might already have finished and be `TERMINATED`.

**Important:** `getState()` returns a snapshot of the thread's state at
the moment of observation. The state can change immediately afterward.

It is useful for monitoring and debugging, but it should not be used to
coordinate thread execution.

## 6. Checking a thread's state while it runs

``` java
public class Main {

    public static void main(String[] args) throws InterruptedException {
        Thread myThread = new Thread(() -> {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
        });

        myThread.start();

        System.out.println(myThread.getState());

        Thread.sleep(1000);

        System.out.println(myThread.getState());

        myThread.join();

        System.out.println(myThread.getState());
    }
}
```

A possible output is:

``` text
RUNNABLE
TIMED_WAITING
TERMINATED
```

The first output can also be `TIMED_WAITING`, depending on scheduling.

The second output should be `TIMED_WAITING` in this example because the
worker has a three-second sleep and the main thread waits one second
before checking.

After `join()` returns, the worker has terminated.

## 7. Important rules about thread states

  Operation or situation              Relevant state
  ----------------------------------- ------------------------------------------
  `new Thread()`                      `NEW`
  After `start()`                     `RUNNABLE`
  `Thread.sleep(...)`                 `TIMED_WAITING`
  `join()` without timeout            Calling thread may enter `WAITING`
  `join(timeout)`                     Calling thread may enter `TIMED_WAITING`
  Waiting to acquire a monitor lock   `BLOCKED`
  `Object.wait()` without timeout     `WAITING`
  `Object.wait(timeout)`              `TIMED_WAITING`
  Task finishes                       `TERMINATED`

Additional details:

-   Calling `sleep()` does not release a monitor lock already held by
    the sleeping thread.
-   Calling `Object.wait()` releases the monitor lock associated with
    that object while waiting.
-   A thread in `BLOCKED`, `WAITING`, or `TIMED_WAITING` is not
    considered runnable.
-   `RUNNABLE` does not guarantee that a thread is currently executing
    on a CPU.
-   A thread cannot transition from `TERMINATED` back to `RUNNABLE`.

## 8. Practical exercise --- Observe thread states

Run this program and observe the output:

``` java
public class Main {

    public static void main(String[] args) throws InterruptedException {
        Thread myThread = new Thread(() -> {
            System.out.println("Worker started");

            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }

            System.out.println("Worker finished");
        });

        System.out.println("State 1: " + myThread.getState());

        myThread.start();

        System.out.println("State 2: " + myThread.getState());

        Thread.sleep(1000);

        System.out.println("State 3: " + myThread.getState());

        myThread.join();

        System.out.println("State 4: " + myThread.getState());
    }
}
```

Expected observations:

  -----------------------------------------------------------------------
  Point                               Expected state
  ----------------------------------- -----------------------------------
  Before `start()`                    `NEW`

  Immediately after `start()`         Usually `RUNNABLE`, but it may have
                                      progressed already

  After main sleeps for one second    `TIMED_WAITING`

  After `join()` returns              `TERMINATED`
  -----------------------------------------------------------------------

Run this program several times. Notice that thread scheduling can affect
the exact output.

## 9. Knowledge check

Try answering these questions before moving to the next lesson.

1.  What is the state of a `Thread` object immediately after creation?
2.  Does Java have a separate `RUNNING` state in `Thread.State`?
3.  What state does a thread enter when it calls `Thread.sleep(3000)`?
4.  Thread B wants to enter a synchronized method, but Thread A owns the
    monitor lock. What state can Thread B enter?
5.  Can a `TERMINATED` thread be started again?
6.  What is the difference between `WAITING` and `TIMED_WAITING`?
7.  Why should `getState()` not be used to coordinate thread execution?

## 10. Important things to remember

-   Java defines six official thread states.
-   `Thread.State` is an enum containing these six states.
-   `getState()` returns the observed state of a thread.
-   `NEW` means the thread has been created but not started.
-   `RUNNABLE` includes both ready-to-run and currently-running threads.
-   `BLOCKED` means waiting to acquire a monitor lock.
-   `WAITING` means waiting indefinitely.
-   `TIMED_WAITING` means waiting for a specified duration.
-   `TERMINATED` means the thread has finished execution.
-   Thread states can change as the thread progresses.
-   `getState()` is primarily useful for monitoring and debugging.

------------------------------------------------------------------------

## What's next?

### Java Concurrency --- Lesson 6: `Thread.currentThread()`

We'll learn:

1.  What is `Thread.currentThread()`?
2.  Why do we need it?
3.  How can a thread identify itself?
4.  How does it behave inside the main thread?
5.  How does it behave inside newly created threads?
6.  Practical examples using multiple threads.

**Goal:** Understand how Java allows code to identify and interact with
the thread that is currently executing it.
