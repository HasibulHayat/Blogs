# Java Concurrency --- Lesson 6

## `Thread.currentThread()`

In our previous lessons, we learned how to create threads, execute
tasks, pause threads, wait for them to finish, and understand their
lifecycle.

But how does a thread know which thread is currently executing its code?

Java provides a method for this purpose:

``` java
Thread.currentThread();
```

## 1. What is `Thread.currentThread()`?

`Thread.currentThread()` is a static method of the `Thread` class.

It returns a reference to the `Thread` object representing the thread
that is currently executing the code.

In simple words, it tells us which thread is currently running the code.

Example:

``` java
public class Main {

    public static void main(String[] args) {
        Thread current = Thread.currentThread();

        System.out.println(current);
    }
}
```

Possible output:

``` text
Thread[main,5,main]
```

We did not create a new thread. We simply asked Java which thread is
currently executing this code.

## 2. Why do we need `Thread.currentThread()`?

Imagine we create two threads:

``` java
public class Main {

    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            System.out.println("Hello from Thread A");
        });

        Thread threadB = new Thread(() -> {
            System.out.println("Hello from Thread B");
        });

        threadA.start();
        threadB.start();
    }
}
```

What if we want to know which thread is executing a particular piece of
code?

We could manually print thread names, but that is not practical for
reusable methods that can be called by any thread.

`Thread.currentThread()` lets a method identify the thread that is
currently executing it.

## 3. Getting the current thread's name

Every Java thread has a name.

We can retrieve it using:

``` java
Thread.currentThread().getName();
```

Example:

``` java
public class Main {

    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getName());
    }
}
```

Output:

``` text
main
```

The `main()` method executes on the main thread.

Now let's create another thread:

``` java
public class Main {

    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            System.out.println(
                Thread.currentThread().getName()
            );
        });

        threadA.start();
    }
}
```

Possible output:

``` text
Thread-0
```

Java automatically assigns a default name if we do not provide one.

Common default names include:

-   `main` for the main thread.
-   `Thread-0` for the first automatically named thread.
-   `Thread-1` for the next.
-   `Thread-2` after that.

These are default names, not special thread types.

## 4. Understanding `currentThread()` with multiple threads

Let's create three threads and print their names:

``` java
public class Main {

    public static void main(String[] args) {
        Runnable task = () -> {
            System.out.println(
                "Executing on: " +
                Thread.currentThread().getName()
            );
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

Possible output:

``` text
Executing on: Thread-0
Executing on: Thread-2
Executing on: Thread-1
```

The output order is not guaranteed. The JVM and operating-system
scheduler determine when each thread gets CPU time.

Each thread nevertheless identifies itself correctly.

## 5. Difference between `currentThread()` and a Thread reference

Consider:

``` java
Thread threadA = new Thread(task);
```

Here, `threadA` is a reference to a particular `Thread` object.

Now consider:

``` java
Thread.currentThread();
```

This returns a reference to whichever thread is currently executing the
code.

These are not necessarily the same thread.

Example:

``` java
public class Main {

    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            System.out.println(
                "Current thread: " +
                Thread.currentThread().getName()
            );
        });

        threadA.start();

        System.out.println(
            "Current thread: " +
            Thread.currentThread().getName()
        );
    }
}
```

Possible output:

``` text
Current thread: main
Current thread: Thread-0
```

When the main thread executes `currentThread()`, it returns the main
thread.

When Thread A executes `currentThread()`, it returns Thread A.

The result depends on which thread executes the method call.

## 6. Can we use `currentThread()` inside an ordinary method?

Yes. We can call it from any method.

``` java
public class Main {

    public static void printCurrentThread() {
        System.out.println(
            "Currently executing on: " +
            Thread.currentThread().getName()
        );
    }

    public static void main(String[] args) {
        printCurrentThread();

        Thread threadA = new Thread(() -> {
            printCurrentThread();
        });

        Thread threadB = new Thread(() -> {
            printCurrentThread();
        });

        threadA.start();
        threadB.start();
    }
}
```

Possible output:

``` text
Currently executing on: main
Currently executing on: Thread-0
Currently executing on: Thread-1
```

We did not pass thread information to `printCurrentThread()`. The method
identifies the executing thread automatically.

This is useful for reusable methods that might be called by different
threads.

## 7. Getting other information about the current thread

`currentThread()` returns a `Thread` object. We can call other `Thread`
methods on it.

### A. Get the thread name

``` java
String name = Thread.currentThread().getName();
```

### B. Get the thread ID

``` java
long id = Thread.currentThread().threadId();
```

`threadId()` is available since Java 19. The older `getId()` method was
deprecated in Java 19.

### C. Get the thread priority

``` java
int priority = Thread.currentThread().getPriority();
```

### D. Get the thread state

``` java
Thread.State state = Thread.currentThread().getState();
```

### E. Check whether it is a daemon thread

``` java
boolean daemon = Thread.currentThread().isDaemon();
```

We will study thread priorities and daemon threads in upcoming lessons.

## 8. Practical example --- Print current thread information

Let's combine what we have learned:

``` java
public class Main {

    public static void printThreadInfo() {
        Thread current = Thread.currentThread();

        System.out.println("----------------------");
        System.out.println("Thread Name: " + current.getName());
        System.out.println("Thread ID: " + current.threadId());
        System.out.println("Thread Priority: " + current.getPriority());
        System.out.println("Thread State: " + current.getState());
        System.out.println("----------------------");
    }

    public static void main(String[] args) {
        printThreadInfo();

        Thread threadA = new Thread(() -> {
            printThreadInfo();
        });

        Thread threadB = new Thread(() -> {
            printThreadInfo();
        });

        threadA.start();
        threadB.start();
    }
}
```

Possible output (the order of the thread sections may vary):

``` text
----------------------
Thread Name: main
Thread ID: 1
Thread Priority: 5
Thread State: RUNNABLE
----------------------

----------------------
Thread Name: Thread-0
Thread ID: 24
Thread Priority: 5
Thread State: RUNNABLE
----------------------

----------------------
Thread Name: Thread-1
Thread ID: 25
Thread Priority: 5
Thread State: RUNNABLE
----------------------
```

The actual thread IDs depend on the JVM and environment.

Notice that the same method provides different information depending on
which thread calls it.

## 9. Important things to remember

-   `Thread.currentThread()` is a static method of the `Thread` class.
-   It returns a reference to the currently executing `Thread` object.
-   It does not create a new thread.
-   It works inside `main()`, `Runnable` tasks, and ordinary methods.
-   `getName()` retrieves the current thread's name.
-   `threadId()` retrieves the current thread's ID (Java 19+).
-   `getPriority()` retrieves its priority.
-   `getState()` retrieves its observed state.
-   The returned `Thread` object can be used to access other `Thread`
    methods.
-   The result depends on which thread executes the method call.

## 10. Knowledge check

Try answering these questions before moving to the next lesson.

1.  What does `Thread.currentThread()` return?
2.  Is `currentThread()` a static method?
3.  If Thread A calls `Thread.currentThread()`, which thread does it
    return?
4.  Can we call `currentThread()` inside an ordinary Java method?
5.  Does `Thread.currentThread()` create or start a new thread?

------------------------------------------------------------------------

## What's next?

### Java Concurrency --- Lesson 7: Thread Names and IDs

We'll learn:

1.  Why do threads have names?
2.  How does Java assign default thread names?
3.  How do we create threads with custom names?
4.  Understanding `setName()` and `getName()`.
5.  Understanding thread IDs and `threadId()`.
6.  Are thread names and IDs unique?
7.  Practical examples.

**Goal:** Learn how to identify, name, and distinguish threads in a Java
application.
