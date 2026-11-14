# Java Concurrency --- Lesson 8

## Thread Priorities in Java

In Lesson 7, we learned how to identify threads using their names and
IDs. In this lesson, we'll learn another property of a thread:
**priority**.

Java allows us to assign a priority to a thread. This gives the JVM and
underlying operating system a hint about the thread's relative
scheduling importance.

The important point is that priority is a hint---not a guarantee that a
thread will run first or finish faster.

------------------------------------------------------------------------

## 1. What is thread priority?

Thread priority is a numeric value associated with a thread. It
indicates its relative importance as a scheduling hint.

Java provides three constants:

  Constant                   Value Meaning
  ------------------------ ------- ------------------
  `Thread.MIN_PRIORITY`          1 Lowest priority
  `Thread.NORM_PRIORITY`         5 Normal priority
  `Thread.MAX_PRIORITY`         10 Highest priority

The default priority of the main thread is normally `5`.

A newly created thread normally inherits the priority of the thread that
creates it.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            System.out.println("Hello from thread");
        });

        System.out.println("Main priority: "
                + Thread.currentThread().getPriority());

        System.out.println("New thread priority: "
                + thread.getPriority());
    }
}
```

Possible output:

``` text
Main priority: 5
New thread priority: 5
```

The new thread inherits the priority of the main thread.

------------------------------------------------------------------------

## 2. Why do thread priorities exist?

Imagine our Turf booking system eventually needs to perform several
background tasks:

-   Send booking confirmation notifications.
-   Generate daily booking reports.
-   Process analytics.
-   Clean up expired temporary data.

You might consider booking-related work more important than generating
analytics reports.

Thread priorities provide a way to express relative importance to the
scheduler.

However, **higher priority does not guarantee earlier execution or
faster completion**. Priority is only a scheduling hint. The JVM and
operating system decide how threads actually receive CPU time.

------------------------------------------------------------------------

## 3. Getting a thread's priority

Java provides the `getPriority()` method.

### Syntax

``` java
thread.getPriority();
```

It returns an integer between `1` and `10`.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            System.out.println("Working...");
        });

        System.out.println(worker.getPriority());
    }
}
```

Output:

``` text
5
```

You can also get the priority of the currently executing thread:

``` java
Thread.currentThread().getPriority();
```

Remember from Lesson 6 that `currentThread()` returns the thread
currently executing the code.

------------------------------------------------------------------------

## 4. Setting a thread's priority

Java provides the `setPriority()` method.

### Syntax

``` java
thread.setPriority(priority);
```

The priority must be between `1` and `10`.

Example:

``` java
public class Main {
    public static void main(String[] args) {
        Thread notificationThread = new Thread(() -> {
            System.out.println("Sending notification...");
        });

        notificationThread.setPriority(Thread.MAX_PRIORITY);

        System.out.println(
                "Priority: " + notificationThread.getPriority()
        );

        notificationThread.start();
    }
}
```

Output:

``` text
Priority: 10
Sending notification...
```

Here, we assigned the highest Java thread priority to the notification
thread.

You can also use numeric values directly:

``` java
thread.setPriority(8);
```

But using the constants is more readable:

``` java
thread.setPriority(Thread.MAX_PRIORITY);
```

------------------------------------------------------------------------

## 5. A complete example with multiple priorities

Let's create three threads with different priorities.

``` java
public class Main {
    public static void main(String[] args) {

        Thread lowPriorityThread = new Thread(() -> {
            System.out.println(
                    "Low priority: "
                    + Thread.currentThread().getPriority()
            );
        });

        Thread normalPriorityThread = new Thread(() -> {
            System.out.println(
                    "Normal priority: "
                    + Thread.currentThread().getPriority()
            );
        });

        Thread highPriorityThread = new Thread(() -> {
            System.out.println(
                    "High priority: "
                    + Thread.currentThread().getPriority()
            );
        });

        lowPriorityThread.setPriority(Thread.MIN_PRIORITY);
        normalPriorityThread.setPriority(Thread.NORM_PRIORITY);
        highPriorityThread.setPriority(Thread.MAX_PRIORITY);

        lowPriorityThread.start();
        normalPriorityThread.start();
        highPriorityThread.start();
    }
}
```

One possible output:

``` text
High priority: 10
Low priority: 1
Normal priority: 5
```

Another execution might produce a different order:

``` text
Normal priority: 5
High priority: 10
Low priority: 1
```

Both are valid. The order in which threads print is not guaranteed by
their priorities.

------------------------------------------------------------------------

## 6. What happens if you provide an invalid priority?

Java accepts priority values from `1` through `10`.

For example:

``` java
thread.setPriority(15);
```

This throws an `IllegalArgumentException`.

Likewise:

``` java
thread.setPriority(0);
```

Also throws an `IllegalArgumentException`.

The valid range is:

``` java
Thread.MIN_PRIORITY <= priority
        && priority <= Thread.MAX_PRIORITY
```

One additional detail: a thread's priority may be restricted by its
`ThreadGroup`'s maximum priority. This is a legacy API detail that is
unlikely to matter in most everyday application development.

------------------------------------------------------------------------

## 7. Why set priority if it does not guarantee faster work?

This is an important question.

Imagine a computer with one CPU core and three runnable threads:

  Thread     Priority Task
  -------- ---------- --------------------------------
  A                10 Important background operation
  B                 5 Normal operation
  C                 1 Low-importance operation

All three threads want CPU time. The scheduler has to decide which
thread gets to execute.

Priority gives the scheduler information that A is more important than
B, and B is more important than C. Depending on the JVM and operating
system, the scheduler may take that information into account.

But Java does not promise a strict priority order.

### Why can't Java guarantee it?

There are several layers involved:

1.  Your Java application creates and configures threads.
2.  The JVM manages Java threads and interacts with the operating
    system.
3.  The operating system scheduler makes actual CPU scheduling
    decisions.
4.  The CPU executes instructions.

Different operating systems have different scheduling mechanisms. The
effect of Java thread priorities can therefore vary by platform. Some
systems may take priority into account; others may have a limited effect
or effectively ignore it.

### When might it be useful?

Historically, priorities could influence which threads received CPU time
more readily. For example, an application might try to give a
user-interface thread more scheduling importance than background work.

But even in such cases, priority is not a reliable guarantee. Modern
applications generally use higher-level scheduling mechanisms for
application tasks.

------------------------------------------------------------------------

## 8. Thread priority versus application task scheduling

Let's use our Turf booking system as an example.

Imagine we have these tasks:

1.  Process booking requests.
2.  Send booking confirmation notifications.
3.  Generate analytics reports.

Suppose we want booking requests to be handled ahead of analytics jobs.

We should not depend on this:

``` java
bookingThread.setPriority(10);
analyticsThread.setPriority(1);
```

Instead, we can design separate executors with appropriate worker
capacities and task queues.

Conceptually:

``` text
Booking executor
    Booking requests
        -> Booking processing workers

Analytics executor
    Analytics jobs
        -> Analytics workers
```

This gives us more control over how application tasks are submitted and
executed.

We'll learn about `ExecutorService`, thread pools, and task queues later
in our concurrency lessons.

One important distinction: separate executors help isolate workloads,
but they do not automatically guarantee that booking requests always
finish first. That requires deliberate queueing, capacity, and
resource-management policies.

------------------------------------------------------------------------

## 9. Important things to remember

1.  Java thread priorities range from `1` to `10`.
2.  `Thread.MIN_PRIORITY` is `1`.
3.  `Thread.NORM_PRIORITY` is `5`.
4.  `Thread.MAX_PRIORITY` is `10`.
5.  The main thread's default priority is normally `5`.
6.  A newly created thread normally inherits its creator's priority.
7.  Use `getPriority()` to read a thread's priority.
8.  Use `setPriority()` to change a thread's priority.
9.  An invalid priority throws `IllegalArgumentException`.
10. A higher priority does not guarantee earlier execution or faster
    completion.
11. Priority does not change a thread's lifecycle state or create
    additional CPU capacity.

### Engineering recommendation

Learn `Thread.setPriority()` because it is part of Java's threading
fundamentals, but do not build critical application logic around it.

For application-level scheduling, use appropriate executors, queues, and
resource-management policies.

**Key principle:** Do not use CPU scheduling priority to solve
business-level task scheduling problems.

------------------------------------------------------------------------

## 10. Practical exercises

Try these yourself before looking for help.

### Exercise 1 --- Read the default priority

Create a thread and print its priority without setting one manually.

### Exercise 2 --- Change the priority

Create a thread, assign it `Thread.MAX_PRIORITY`, and print its
priority.

### Exercise 3 --- Priority inheritance

Set the main thread's priority to `8`. Then create a child thread and
print the child's priority.

Hint:

``` java
Thread.currentThread().setPriority(8);
```

What priority does the child inherit?

### Exercise 4 --- Invalid priority

Try assigning priority `11`. Observe the exception.

### Exercise 5 --- Output order

Create three threads with priorities `1`, `5`, and `10`. Start them in
that order.

Run the program multiple times. Does the output always follow priority
order?

------------------------------------------------------------------------

## 11. Knowledge check

1.  What is the maximum Java thread priority?
2.  What priority does a newly created thread normally inherit?
3.  Does priority `10` guarantee that a thread executes before priority
    `5`?
4.  What happens when you call `setPriority(11)`?
5.  Why might an executor and task queue be a better solution than
    thread priority for scheduling business tasks?

------------------------------------------------------------------------

## 12. What's next?

### Lesson 9 --- `Thread.yield()`

We'll explore `yield()`, why it exists, what happens when we call it,
and why it also does not guarantee a particular execution order.
