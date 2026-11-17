# Java Concurrency — Lesson 13: ThreadGroup

## Introduction

So far, we have mostly worked with individual `Thread` objects. Java also provides `ThreadGroup`, a class that represents a group of threads.

A `ThreadGroup` can contain threads and other thread groups. It offers some group-level operations, such as listing active threads, interrupting threads in the group, and managing the group's maximum priority.

However, `ThreadGroup` is a **legacy API**. It is useful to understand because it appears in Java's thread APIs and participates in uncaught-exception handling, but modern applications generally use executors and other concurrency utilities to manage tasks.

---

## 1. What is ThreadGroup?

`java.lang.ThreadGroup` represents a group of threads.

A group can contain:
- Individual threads.
- Other thread groups, called child groups.

This creates a hierarchy:

```text
system thread group
└── main thread group
    ├── main thread
    ├── worker-1
    └── background group
        ├── worker-2
        └── worker-3
```

The exact groups and threads in a running JVM depend on the application and runtime.

A `ThreadGroup` is not a thread pool. It does not provide a task queue, automatic worker reuse, task scheduling, or result retrieval.

## 2. Creating a ThreadGroup

Use the constructor that accepts a group name:

```java
ThreadGroup bookingGroup = new ThreadGroup("booking-workers");
```

You can then create threads belonging to that group:

```java
Thread worker1 = new Thread(
    bookingGroup,
    () -> System.out.println("Worker 1 running"),
    "booking-worker-1"
);

Thread worker2 = new Thread(
    bookingGroup,
    () -> System.out.println("Worker 2 running"),
    "booking-worker-2"
);

worker1.start();
worker2.start();
```

The relevant constructor shape is:

```java
new Thread(ThreadGroup group, Runnable task, String name)
```

The group is assigned when the thread is constructed. You cannot move an already-created thread into a different group.

### A complete example

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        ThreadGroup bookingGroup =
            new ThreadGroup("booking-workers");

        Thread worker1 = new Thread(
            bookingGroup,
            () -> {
                System.out.println(
                    Thread.currentThread().getName()
                    + " is running"
                );
            },
            "booking-worker-1"
        );

        Thread worker2 = new Thread(
            bookingGroup,
            () -> {
                System.out.println(
                    Thread.currentThread().getName()
                    + " is running"
                );
            },
            "booking-worker-2"
        );

        worker1.start();
        worker2.start();

        worker1.join();
        worker2.join();

        System.out.println("Group name: " + bookingGroup.getName());
    }
}
```

The order of the two worker messages is not guaranteed. Thread scheduling is nondeterministic.

## 3. Getting a thread's group

A thread can tell you which group it belongs to:

```java
ThreadGroup group = worker1.getThreadGroup();
```

You can inspect the group:

```java
System.out.println(group.getName());
```

If the thread has terminated, `getThreadGroup()` returns `null`.

This is useful to know when inspecting thread ownership, but do not build application task management around thread groups.

## 4. Useful ThreadGroup methods

Here are some methods you may encounter.

| Method | Purpose |
|---|---|
| `getName()` | Returns the group's name |
| `getParent()` | Returns the parent group, or `null` if there is no parent |
| `activeCount()` | Estimates the number of active threads in the group and its subgroups |
| `activeGroupCount()` | Estimates the number of active groups in the group and its subgroups |
| `list()` | Prints information about the group and its active threads/subgroups |
| `interrupt()` | Interrupts the threads in the group and its subgroups |
| `isDaemon()` | Tests whether the group is a daemon group |
| `setDaemon(boolean)` | Sets the daemon status of the group, subject to its restrictions |
| `getMaxPriority()` | Gets the group's maximum priority |
| `setMaxPriority(int)` | Sets the group's maximum priority limit |
| `isDestroyed()` | Tests whether the group has been destroyed; deprecated API |

### Important: `activeCount()` is an estimate

Example:

```java
int count = bookingGroup.activeCount();
System.out.println("Estimated active threads: " + count);
```

The result is an estimate, not a reliable snapshot or synchronization mechanism. Threads can start or finish while the count is being obtained.

The same caution applies to `activeGroupCount()`.

Do not use these counts to make correctness-critical decisions, such as deciding whether all booking work has finished.

## 5. Listing a group's threads

You can call:

```java
bookingGroup.list();
```

This prints information about the group, its active threads, and its subgroups to standard output.

It is mainly a diagnostic facility. For production observability, structured logging, thread dumps, monitoring, and executor metrics are usually more useful.

## 6. Interrupting all threads in a group

One of the more notable group-level operations is:

```java
bookingGroup.interrupt();
```

This requests interruption of the threads in that group and its subgroups.

Remember what `interrupt()` means:
- It does not forcibly kill a thread.
- It sets the thread's interrupt status or causes certain interruptible blocking operations to throw `InterruptedException`.
- The task must cooperate with the interruption request.

For example:

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        ThreadGroup group = new ThreadGroup("workers");

        Runnable task = () -> {
            while (!Thread.currentThread().isInterrupted()) {
                System.out.println(
                    Thread.currentThread().getName() + " working"
                );

                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }

            System.out.println(
                Thread.currentThread().getName() + " stopping"
            );
        };

        Thread worker1 = new Thread(group, task, "worker-1");
        Thread worker2 = new Thread(group, task, "worker-2");

        worker1.start();
        worker2.start();

        Thread.sleep(1200);

        group.interrupt();

        worker1.join();
        worker2.join();

        System.out.println("Workers finished");
    }
}
```

The workers cooperate with interruption and exit their loops.

In modern application code, it is generally better to manage task cancellation through an `ExecutorService` or another explicit lifecycle mechanism rather than using `ThreadGroup.interrupt()`.

## 7. ThreadGroup hierarchy

A group can have a parent:

```java
ThreadGroup parent = new ThreadGroup("application-workers");
ThreadGroup child = new ThreadGroup(parent, "booking-workers");
```

You can retrieve the parent:

```java
ThreadGroup actualParent = child.getParent();
```

A child group is part of its parent's hierarchy. Some group operations, such as `interrupt()`, include subgroups.

A thread group hierarchy is not the same thing as an application architecture hierarchy. It does not represent booking ownership, user permissions, or business relationships.

## 8. ThreadGroup and priority

A group has a maximum priority:

```java
int maxPriority = bookingGroup.getMaxPriority();
```

You can change the maximum:

```java
bookingGroup.setMaxPriority(Thread.NORM_PRIORITY);
```

The group's maximum priority acts as a limit on the priorities of threads in that group and its subgroups. It does not guarantee scheduling order or CPU time.

As with thread priorities, do not use this mechanism to implement business-level task prioritization.

## 9. ThreadGroup and uncaught exception handling

This connects directly to Lesson 12.

`ThreadGroup` implements:

```java
Thread.UncaughtExceptionHandler
```

That means a thread group can participate in handling an uncaught exception from one of its threads.

If a thread does not have its own uncaught exception handler, its group is used as the effective handler. The group may delegate to its parent group. The root group's default behavior can invoke the JVM-wide default uncaught exception handler if one is configured; otherwise, it prints the stack trace to `System.err`.

This is why the default uncaught-exception handling path can involve `ThreadGroup`.

You can override `uncaughtException` in a custom subclass, but this is rarely the right approach for modern application-level error handling.

## 10. Why ThreadGroup is considered legacy

`ThreadGroup` was introduced when Java's thread-management facilities were much more limited.

It has important limitations:

- It is not a thread pool.
- It does not schedule or queue tasks.
- It does not provide `Future` results.
- Its active-thread counts are estimates.
- Group-wide operations are relatively coarse.
- It does not provide the lifecycle and resource-management features expected from modern task execution frameworks.

For modern Java applications, prefer:
- `ExecutorService` for task execution and lifecycle management.
- `ScheduledExecutorService` for scheduled tasks.
- `Future` and `CompletableFuture` for asynchronous results.
- Structured concurrency APIs where available and appropriate.

You should understand `ThreadGroup`, but you generally should not design new application architecture around it.

## 11. Turf booking system perspective

Imagine the Turf application has:
- Booking confirmation tasks.
- Notification tasks.
- Expired booking cleanup tasks.

It may be tempting to create one `ThreadGroup` for each category and use the groups to manage those tasks.

For example:

```java
ThreadGroup notificationGroup =
    new ThreadGroup("notification-workers");
```

That can group raw threads, but it does not give you:
- A bounded number of reusable workers.
- A queue for pending notification tasks.
- Backpressure when the system is overloaded.
- Task result retrieval.
- A standard graceful shutdown workflow.

An executor is a better fit for those requirements. We will study it later in this roadmap.

## 12. Important things to remember

1. `ThreadGroup` represents a group of threads and can contain child groups.
2. A thread's group is assigned when the thread is constructed.
3. `Thread.getThreadGroup()` returns the thread's group, or `null` after termination.
4. `activeCount()` and `activeGroupCount()` return estimates.
5. `ThreadGroup.interrupt()` requests interruption; it does not forcibly stop threads.
6. A group's maximum priority is a limit, not a scheduling guarantee.
7. `ThreadGroup` participates in uncaught-exception handling.
8. `ThreadGroup` is a legacy API, not a thread pool.
9. Prefer executors and modern concurrency utilities for new application task management.

---

## 13. Knowledge check

Try answering these questions before checking the answers.

1. Is `ThreadGroup` a thread pool?
2. How do you create a thread that belongs to a particular group?
3. Does `activeCount()` provide an exact, stable count?
4. What does `ThreadGroup.interrupt()` do?
5. Why should modern applications generally prefer executors?

### Answers

1. No. It groups threads but does not provide task queues, worker reuse, or scheduling.
2. Pass the group to a `Thread` constructor, such as `new Thread(group, task, name)`.
3. No. It is an estimate and can change as threads start and finish.
4. It requests interruption of threads in the group and its subgroups. Threads must cooperate.
5. Executors provide task-oriented facilities such as worker pools, queues, lifecycle management, and result handling.

---

## Next lesson

**Lesson 14 — What Is Shared Data?**

We will move from learning individual thread APIs to understanding the central challenge of concurrency: multiple threads accessing the same data.

We will start with simple examples before introducing race conditions, critical sections, atomicity, thread safety, visibility, and the Java Memory Model.
