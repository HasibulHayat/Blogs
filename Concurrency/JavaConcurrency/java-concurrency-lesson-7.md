# Java Concurrency --- Lesson 7

## Thread Names and IDs

In our previous lesson, we learned about `Thread.currentThread()` and
how a thread can identify itself while executing code.

We also learned that every Java thread has a name and an ID.

In this lesson, we'll explore:

-   Why threads have names.
-   How Java assigns names automatically.
-   How to give threads custom names.
-   What a thread ID is.
-   Whether thread names and IDs are unique.
-   Whether a thread's name can be changed after creation.

## 1. Why do threads have names?

Imagine an application that creates multiple threads:

``` java
public class Main {

    public static void main(String[] args) {
        Runnable task = () -> {
            System.out.println("Task is executing");
        };

        Thread threadA = new Thread(task);
        Thread threadB = new Thread(task);
        Thread threadC = new Thread(task);

        threadA.start();
        threadB.start();
        threadC.start();
    }
}
```

If we inspect logs that only say `Task is executing`, it can be
difficult to tell which thread printed each message.

Thread names make threads easier to identify in application logs,
debugging tools, and monitoring.

For example:

``` text
Booking-Thread
Payment-Thread
Notification-Thread
```

## 2. How does Java assign default thread names?

When we create a thread without specifying a name, Java automatically
assigns one.

``` java
public class Main {

    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            System.out.println("Thread A");
        });

        Thread threadB = new Thread(() -> {
            System.out.println("Thread B");
        });

        Thread threadC = new Thread(() -> {
            System.out.println("Thread C");
        });

        System.out.println(threadA.getName());
        System.out.println(threadB.getName());
        System.out.println(threadC.getName());
    }
}
```

Possible output:

``` text
Thread-0
Thread-1
Thread-2
```

The main thread is normally named:

``` text
main
```

Remember:

-   The main thread is normally named `main`.
-   Automatically named threads commonly follow the `Thread-0`,
    `Thread-1`, `Thread-2` pattern.
-   These are default names, not special thread types.
-   We are not required to use the default names.

## 3. What is `getName()`?

Java provides `getName()` to retrieve the name of a particular thread.

``` java
thread.getName();
```

Example:

``` java
public class Main {

    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            System.out.println("Hello");
        });

        System.out.println(threadA.getName());

        threadA.start();
    }
}
```

Possible output:

``` text
Thread-0
```

Compare these expressions:

  ------------------------------------------------------------------------
  Expression                           Meaning
  ------------------------------------ -----------------------------------
  `threadA.getName()`                  Gets the name of the thread
                                       referenced by `threadA`

  `Thread.currentThread().getName()`   Gets the name of the currently
                                       executing thread
  ------------------------------------------------------------------------

They can return different names depending on where they are called.

## 4. How do we create threads with custom names?

The `Thread` class provides a constructor that accepts both a `Runnable`
task and a thread name.

``` java
new Thread(Runnable task, String name);
```

Example:

``` java
public class Main {

    public static void main(String[] args) {
        Runnable task = () -> {
            System.out.println(
                "Executing on: " +
                Thread.currentThread().getName()
            );
        };

        Thread threadA = new Thread(task, "Booking-Thread");
        Thread threadB = new Thread(task, "Payment-Thread");

        threadA.start();
        threadB.start();
    }
}
```

Possible output:

``` text
Executing on: Booking-Thread
Executing on: Payment-Thread
```

Here:

-   `task` is the work the thread will execute.
-   `"Booking-Thread"` is the name assigned to the thread.

We can also write the constructor inline:

``` java
Thread threadA = new Thread(
    () -> {
        System.out.println("Booking task executing");
    },
    "Booking-Thread"
);
```

## 5. What is `setName()`?

If we have already created a thread and want to change its name, Java
provides `setName()`.

``` java
thread.setName("New-Thread-Name");
```

Example:

``` java
public class Main {

    public static void main(String[] args) {
        Thread myThread = new Thread(() -> {
            System.out.println(
                Thread.currentThread().getName()
            );
        });

        myThread.setName("My-Custom-Thread");

        myThread.start();
    }
}
```

Output:

``` text
My-Custom-Thread
```

Execution:

1.  We create a `Thread` object.
2.  Java assigns its default name.
3.  We call `setName()`.
4.  Java changes the thread's name.
5.  We start the thread.
6.  The thread prints its updated name.

We can also change a thread's name after it has started, provided we
still have its reference and the security environment permits it.

Example:

``` java
public class Main {

    public static void main(String[] args) throws InterruptedException {
        Thread myThread = new Thread(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }

            System.out.println(
                Thread.currentThread().getName()
            );
        });

        myThread.start();

        myThread.setName("Updated-Thread");

        myThread.join();
    }
}
```

Possible output:

``` text
Updated-Thread
```

A thread's name is not required to remain unchanged throughout its
lifecycle.

## 6. What is a Thread ID?

Java also gives each thread a numerical identifier called a **thread
ID**.

For example:

``` text
Thread name: Booking-Thread
Thread ID: 24
```

We can retrieve a thread's ID using:

``` java
thread.threadId();
```

Example:

``` java
public class Main {

    public static void main(String[] args) {
        Thread myThread = new Thread(
            () -> System.out.println("Hello"),
            "Booking-Thread"
        );

        System.out.println("Thread Name: " + myThread.getName());
        System.out.println("Thread ID: " + myThread.threadId());

        myThread.start();
    }
}
```

The actual ID depends on the JVM and environment.

### `getId()` versus `threadId()`

Older Java code commonly uses:

``` java
thread.getId();
```

However, `getId()` was deprecated in Java 19.

For modern Java applications, use:

``` java
thread.threadId();
```

The `threadId()` method was introduced in Java 19.

## 7. Are thread names and IDs unique?

### Thread names

Thread names are not guaranteed to be unique.

For example:

``` java
Thread threadA = new Thread(task, "Worker");
Thread threadB = new Thread(task, "Worker");
```

Both threads can have the same name:

``` text
Worker
Worker
```

Java allows this.

Therefore, we should not rely on thread names as unique identifiers.

### Thread IDs

Thread IDs are unique among threads created within the same JVM, as
specified by the Java API.

A thread ID remains constant during that thread's lifetime.

After a thread terminates, its ID may be reused by a subsequently
created thread.

Therefore:

-   Thread names can be duplicated.
-   Thread IDs distinguish threads within a JVM.
-   Thread IDs are not permanent identifiers across application
    restarts.

## 8. Difference between Thread Names and Thread IDs

  -----------------------------------------------------------------------
  Feature                 Thread Name             Thread ID
  ----------------------- ----------------------- -----------------------
  Data type               `String`                `long`

  Default value           Automatically generated Automatically assigned
                          name                    ID

  Can we change it?       Yes, using `setName()`  No

  Can duplicates exist?   Yes                     Not among threads
                                                  within the same JVM

  Main purpose            Human-readable          Programmatic
                          identification          identification

  Method                  `getName()`             `threadId()`
  -----------------------------------------------------------------------

A useful way to remember this:

-   Names help humans identify threads.
-   IDs help distinguish threads programmatically.

## 9. Practical example --- Managing multiple named threads

Imagine we're developing an application that handles different types of
tasks.

We'll create three threads:

1.  Booking Thread
2.  Payment Thread
3.  Notification Thread

``` java
public class Main {

    public static void main(String[] args) {
        Runnable bookingTask = () -> {
            printThreadInfo();
        };

        Runnable paymentTask = () -> {
            printThreadInfo();
        };

        Runnable notificationTask = () -> {
            printThreadInfo();
        };

        Thread bookingThread = new Thread(
            bookingTask,
            "Booking-Thread"
        );

        Thread paymentThread = new Thread(
            paymentTask,
            "Payment-Thread"
        );

        Thread notificationThread = new Thread(
            notificationTask,
            "Notification-Thread"
        );

        bookingThread.start();
        paymentThread.start();
        notificationThread.start();
    }

    public static void printThreadInfo() {
        Thread current = Thread.currentThread();

        System.out.println("Thread Name: " + current.getName());
        System.out.println("Thread ID: " + current.threadId());
        System.out.println("--------------------");
    }
}
```

Possible output (the order may vary):

``` text
Thread Name: Booking-Thread
Thread ID: 24
--------------------
Thread Name: Notification-Thread
Thread ID: 26
--------------------
Thread Name: Payment-Thread
Thread ID: 25
--------------------
```

Notice:

-   We gave each thread a meaningful name.
-   Each thread has its own ID.
-   We used `Thread.currentThread()` to identify the currently executing
    thread.
-   We used `getName()` and `threadId()` to retrieve its information.
-   The execution order is not guaranteed.

This is a common pattern when debugging multithreaded applications.

## 10. Important things to remember

-   Every Java thread has a name.
-   Java automatically assigns default names when none are provided.
-   The main thread is normally named `main`.
-   `getName()` retrieves a thread's name.
-   `setName()` changes a thread's name.
-   We can provide a custom name through the `Thread` constructor.
-   Every Java thread has a numerical thread ID.
-   Use `threadId()` to retrieve the ID in modern Java.
-   `getId()` has been deprecated since Java 19.
-   Thread names are not guaranteed to be unique.
-   Thread IDs are unique among threads within the same JVM, and may be
    reused after termination.

## 11. Knowledge check

Try answering these questions before moving to the next lesson.

1.  What is the default name of the main thread?
2.  Which method retrieves a thread's name?
3.  How can we create a thread with a custom name?
4.  What is the difference between `getName()` and `setName()`?
5.  Can two threads have the same name?
6.  Which method should we use to retrieve a thread ID in modern Java?
7.  Can we change a thread's name after it has started?
8.  Are thread IDs guaranteed to remain unique across different JVM
    restarts?

------------------------------------------------------------------------

## What's next?

### Java Concurrency --- Lesson 8: Thread Priorities

We'll learn:

1.  What is thread priority?
2.  Why does Java provide thread priorities?
3.  Understanding the priority levels.
4.  Understanding `MIN_PRIORITY`, `NORM_PRIORITY`, and `MAX_PRIORITY`.
5.  What is `setPriority()`?
6.  What is `getPriority()`?
7.  Does higher priority guarantee earlier execution?
8.  Practical examples using multiple threads.

**Goal:** Understand how Java allows us to assign priority values to
threads and how those priorities relate to thread scheduling.
