# Java Concurrency --- Lesson 1: Your First Thread

## 1. What is a thread?

A **thread** is a path of execution inside a running program. It is the
sequence of instructions that the program's CPU work follows.

A Java program starts with a thread called the **main thread**. The
`main()` method runs on this thread.

A common misconception is that every Java class is a thread. **That is
not true.** A class is a blueprint for objects. Most classes do not
create or represent threads. The `Thread` class is a special Java class
that represents a thread of execution.

You can create a `Thread` object from code inside any ordinary Java
class.

## 2. A normal Java program

``` java
public class Main {

    public static void main(String[] args) {
        System.out.println("Hello");
        System.out.println("How are you?");
        System.out.println("Goodbye");
    }
}
```

These statements run in order on the main thread:

``` text
Hello
How are you?
Goodbye
```

## 3. Creating a Thread object

``` java
Thread myThread = new Thread();
```

This creates a `Thread` object, but it does **not** start the new thread
running.

To start it, call:

``` java
myThread.start();
```

Calling `start()` asks the JVM to start a new path of execution. The new
thread will execute the task assigned to it.

Remember:

-   `new Thread()` creates a Thread object.
-   `start()` starts the new thread.
-   `run()` is the method containing the task that the thread executes.
    Calling `run()` directly is not the same as calling `start()`; we
    will study this in a later lesson.

## 4. Starting a thread with a task

Here is the complete example from Lesson 1:

``` java
public class Main {

    public static void main(String[] args) {

        System.out.println("Main thread started");

        Thread myThread = new Thread(() -> {
            System.out.println("My first thread is running!");
        });

        myThread.start();

        System.out.println("Main thread finished");
    }
}
```

The lambda expression supplies the task for the new thread. For now,
think of it as the instructions that the new thread should perform. We
will study `Runnable` and `run()` properly in Lesson 2.

## 5. Why can the output order change?

There are now two threads:

1.  **Main thread** --- continues executing the statements in `main()`.
2.  **myThread** --- executes the task supplied to the `Thread`
    constructor.

The JVM scheduler decides when each thread gets time to run. Their
instructions can therefore be interleaved in different ways.

One possible output:

``` text
Main thread started
Main thread finished
My first thread is running!
```

Another possible output:

``` text
Main thread started
My first thread is running!
Main thread finished
```

Both outputs are valid. The program does not guarantee which of the two
threads prints its message first after `myThread.start()`.

## 6. Important ideas to remember

-   A Java class is **not automatically a thread**.
-   The `main()` method runs on the main thread.
-   You can create a new thread from code inside an ordinary Java class.
-   Creating a `Thread` object does not start it.
-   Calling `start()` starts a new thread of execution.
-   The main thread does not automatically wait for a newly started
    thread to finish.
-   When multiple threads run, the order of their output may vary.

## 7. Practice

1.  Create a Java project named `JavaConcurrencyPractice`.
2.  Create a class named `Main`.
3.  Type and run the example in Section 4.
4.  Run it at least five times.
5.  Notice whether the output order changes.

### Check your understanding

**Question 1:** What is the difference between `new Thread()` and
`myThread.start()`?

**Question 2:** Why can the output order change between executions?

**Question 3:** Is every Java class a thread? Explain the difference
between a class and a thread.

------------------------------------------------------------------------

**Lesson 1 complete.** In Lesson 2, we will learn what `run()` and
`Runnable` do, and how to give a thread a task in a clearer way.
