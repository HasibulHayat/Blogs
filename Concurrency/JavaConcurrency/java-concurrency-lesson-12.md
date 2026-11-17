# Java Concurrency — Lesson 12: Uncaught Exception Handling

## Introduction

In Lesson 10, we learned how to interrupt a thread. In Lesson 11, we learned about daemon threads.

This lesson explains what happens when a thread encounters an exception that it does not handle, and how Java lets us observe and report that failure.

This matters in applications such as a Turf booking system, where background workers may process bookings, send notifications, or perform cleanup independently of the thread that started them.

---

## 1. What happens when a thread throws an uncaught exception?

Consider:

```java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            System.out.println("Worker started");

            int result = 10 / 0;

            System.out.println(result);
        });

        worker.start();

        System.out.println("Main thread continues");
    }
}
```

The worker throws an `ArithmeticException` because it divides by zero. Since no `try-catch` handles it, the worker's execution ends abruptly and the worker thread terminates.

The main thread is independent. It does not automatically receive the worker's exception.

**Important:** An uncaught exception in one thread does not normally terminate every other thread in the JVM. The JVM can continue running while other non-daemon threads are alive.

The JVM's uncaught-exception handling mechanism is invoked when a thread is about to terminate because of an uncaught exception.

## 2. Why do we need uncaught exception handling?

Suppose a background worker processes pending booking notifications:

```java
Thread bookingWorker = new Thread(() -> {
    processBooking();
});

bookingWorker.start();
```

If `processBooking()` unexpectedly throws an exception, useful questions include:

- Which thread failed?
- What exception occurred?
- What was the cause?
- Which background operation was affected?

Uncaught exception handling provides a last-resort place to report failures that escape a thread's execution.

## 3. The `Thread.UncaughtExceptionHandler` interface

Java provides this nested interface:

```java
Thread.UncaughtExceptionHandler
```

It declares one method:

```java
void uncaughtException(Thread t, Throwable e);
```

The parameters are:

| Parameter | Meaning |
|---|---|
| `t` | The thread that encountered the uncaught throwable |
| `e` | The uncaught throwable |

The parameter type is `Throwable`, not merely `Exception`, because the uncaught throwable can be an `Exception` or an `Error`.

The interface is functional, so it can be implemented with a lambda.

## 4. Setting a handler for one thread

```java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            System.out.println("Worker started");

            int result = 10 / 0;

            System.out.println(result);
        }, "booking-worker");

        worker.setUncaughtExceptionHandler((thread, error) -> {
            System.err.println("Thread failed: " + thread.getName());
            System.err.println("Exception: " + error);
        });

        worker.start();
    }
}
```

Possible output:

```text
Worker started
Thread failed: booking-worker
Exception: java.lang.ArithmeticException: / by zero
```

The key API is:

```java
worker.setUncaughtExceptionHandler((thread, error) -> {
    // Report the failure
});
```

When this thread is about to terminate because of an uncaught throwable, Java invokes the registered handler.

It is best practice to configure the handler before calling `start()`, so it is installed before the thread can execute and fail.

### Related method

```java
worker.getUncaughtExceptionHandler();
```

This retrieves the thread's effective uncaught-exception handler. If no handler was explicitly set on the thread, the effective handler is its `ThreadGroup`.

## 5. Setting a default handler

If an application creates many threads, configuring a handler individually on each one can be repetitive.

Java provides a static method:

```java
Thread.setDefaultUncaughtExceptionHandler(handler);
```

Example:

```java
public class Main {
    public static void main(String[] args) {
        Thread.setDefaultUncaughtExceptionHandler((thread, error) -> {
            System.err.println("Thread failed: " + thread.getName());
            System.err.println("Cause: " + error);
        });

        Thread worker1 = new Thread(() -> {
            throw new RuntimeException("Worker 1 failed");
        }, "worker-1");

        Thread worker2 = new Thread(() -> {
            throw new RuntimeException("Worker 2 failed");
        }, "worker-2");

        worker1.start();
        worker2.start();
    }
}
```

You can retrieve the configured default handler using:

```java
Thread.getDefaultUncaughtExceptionHandler();
```

### Per-thread handler versus default handler

| Feature | Per-thread handler | Default handler |
|---|---|---|
| Method | `setUncaughtExceptionHandler()` | `setDefaultUncaughtExceptionHandler()` |
| Scope | One thread | JVM-wide default |
| Role | Specific handling for one thread | Fallback handling |
| Precedence | Used first for that thread | Used only if fallback handling reaches it |

### The `ThreadGroup` detail

There is an intermediate step worth knowing.

If a thread has no explicitly configured handler, its `ThreadGroup` acts as its uncaught-exception handler. A group may delegate to its parent group. At the root, the default group can invoke the JVM's default uncaught-exception handler if one is configured; otherwise, the usual default behavior prints the stack trace to `System.err`.

`ThreadGroup` is a legacy API. You do not need to use it in modern application design, but understanding its role makes the fallback behavior clearer.

## 6. What if we catch the exception ourselves?

An uncaught-exception handler is not a replacement for ordinary `try-catch`.

```java
Thread worker = new Thread(() -> {
    try {
        int result = 10 / 0;
    } catch (ArithmeticException e) {
        System.out.println("Handled inside worker");
    }
});

worker.setUncaughtExceptionHandler((thread, error) -> {
    System.out.println("Uncaught exception handler called");
});

worker.start();
```

Output:

```text
Handled inside worker
```

The uncaught-exception handler is not invoked because the exception was caught inside the thread.

Think of the distinction this way:

- **`try-catch`:** Handle failures that the task knows how to handle.
- **`UncaughtExceptionHandler`:** Report unexpected throwables that escape the thread's execution.

## 7. Does a worker's exception reach the main thread?

No—not automatically.

```java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            throw new RuntimeException("Booking failed");
        });

        worker.start();

        System.out.println("Main continues");
    }
}
```

The main thread does not receive the worker's exception just because it called `start()`.

Calling `join()` does not change that:

```java
worker.start();
worker.join();
```

`join()` waits for the worker to terminate. It does not rethrow the worker's exception in the calling thread.

Later, with `ExecutorService` and `Future`, we will see a way to retrieve task failures through `Future.get()`, which can throw an `ExecutionException`.

## 8. Can a failed thread be restarted?

No. A Java `Thread` instance can be started only once.

```java
Thread worker = new Thread(() -> {
    throw new RuntimeException("Failed");
});

worker.start();
worker.join();

worker.start(); // IllegalThreadStateException
```

If an operation should be retried, create a new thread or submit a new task. An uncaught-exception handler is for observing and reporting the failure; it does not revive or restart the terminated thread.

## 9. Practical Turf booking example

Imagine a background worker processes pending booking notifications:

```java
public class BookingNotificationWorker {
    public static void main(String[] args) {
        Thread notificationWorker = new Thread(() -> {
            System.out.println("Processing notifications...");

            throw new RuntimeException(
                "Notification service unavailable"
            );
        }, "booking-notification-worker");

        notificationWorker.setUncaughtExceptionHandler((thread, error) -> {
            System.err.println("Background worker crashed!");
            System.err.println("Worker: " + thread.getName());
            System.err.println("Failure: " + error.getMessage());
            error.printStackTrace();
        });

        notificationWorker.start();
    }
}
```

In a real application, an exception handler can contribute to:

- Structured logging.
- Error monitoring.
- Metrics and alerts.
- Recording a failure for investigation.

Keep the handler simple and reliable. Avoid slow or complicated recovery work inside it; the handler itself should not become another source of failure.

### Spring Boot note

In a Spring Boot application, prefer framework-managed executors and their task error-handling and lifecycle mechanisms over manually creating every thread and assigning handlers. This lesson focuses on the underlying Java thread API.

## 10. Important things to remember

1. Each Java thread executes independently.
2. An uncaught throwable normally terminates the affected thread.
3. An uncaught exception in a worker does not automatically propagate to the thread that called `start()`.
4. `UncaughtExceptionHandler` receives the failed thread and its `Throwable`.
5. A per-thread handler takes precedence over the default fallback.
6. The thread group participates in the fallback path when no per-thread handler is set.
7. Use `try-catch` for failures the task can handle.
8. Use uncaught-exception handling for last-resort reporting of failures that escape the thread.
9. A terminated thread cannot be restarted.
10. `join()` waits for termination; it does not propagate the worker's exception.

---

## 11. Knowledge check

Try answering these without looking back:

1. A worker thread throws an uncaught `RuntimeException`. What normally happens?
2. Which interface handles uncaught exceptions from Java threads?
3. What does `worker.join()` do if the worker throws an exception?
4. When can the JVM-wide default uncaught-exception handler be invoked?
5. Can you restart a `Thread` after it has terminated?

### Answers

1. The worker thread terminates; other threads normally continue.
2. `Thread.UncaughtExceptionHandler`.
3. It waits for the worker to finish; it does not rethrow the worker's exception.
4. When no per-thread handler is set and the thread-group fallback delegates to the default handler.
5. No. Create a new thread or submit a new task.

---

## Next lesson

**Lesson 13 — ThreadGroup**

We will learn what `ThreadGroup` represents, how Java groups threads, and the basic APIs for inspecting and managing a group. It is a legacy API, so we will also discuss why modern applications generally prefer executors for task management.
