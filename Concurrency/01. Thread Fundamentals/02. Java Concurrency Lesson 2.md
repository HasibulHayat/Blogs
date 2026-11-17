<br>

# Java Concurrency - Lesson 2

<br>
<br>

## 1. What did we learn in Lesson 1?

A Java program starts with a thread called the **main thread**. We can
create another thread using the `Thread` class and start it by calling
`start()`.

But a thread needs a task to execute. In this lesson, we learn how to
define that task.

<br>

## 2. The `run()` method

The `Thread` class has a `run()` method. A thread executes its `run()`
method when it is started.

One way to define a task is to create an anonymous subclass of `Thread`
and override `run()`:

``` java
Thread myThread = new Thread() {

    @Override
    public void run() {
        System.out.println("Hello World");
    }
};

myThread.start();
```

What happens:

1.  `new Thread() { ... }` creates an anonymous subclass of `Thread`.
2.  `@Override` replaces the inherited `run()` implementation with our
    task.
3.  `myThread.start()` starts a new thread of execution.
4.  The new thread executes the overridden `run()` method.

<br>

## 3. What is `Runnable`?

`Runnable` is a Java interface that represents a task that can be
executed.

Its key method is:

``` java
public interface Runnable {
    void run();
}
```

We can create a `Runnable` task separately from a `Thread`:

``` java
Runnable myTask = new Runnable() {

    @Override
    public void run() {
        System.out.println("Hello from my Runnable task!");
    }
};

Thread myThread = new Thread(myTask);

myThread.start();
```

Here:

-   `myTask` describes the work to perform.
-   `new Thread(myTask)` gives that task to a thread.
-   `myThread.start()` starts the new thread, which executes the task.

This separates the **task** from the **thread that executes it**.

<br>

## 4. Using a lambda with `Thread`

This common Java code:

``` java
Thread myThread = new Thread(() -> {
    System.out.println("Hello World");
});

myThread.start();
```

does **not** override the `Thread` class's `run()` method.

The lambda is an implementation of the `Runnable` interface. The
`Thread` constructor accepts a `Runnable` task.

The code is effectively equivalent to:

``` java
Runnable task = () -> {
    System.out.println("Hello World");
};

Thread myThread = new Thread(task);

myThread.start();
```

Because `Runnable` has just one abstract method, `run()`, Java allows a
lambda expression to implement it.

### Remember the difference

**Overriding `Thread.run()`:**

``` java
Thread myThread = new Thread() {

    @Override
    public void run() {
        System.out.println("Hello World");
    }
};

myThread.start();
```

This creates an anonymous subclass of `Thread` and overrides its `run()`
method.

**Passing a `Runnable` task:**

``` java
Thread myThread = new Thread(() -> {
    System.out.println("Hello World");
});

myThread.start();
```

This passes a `Runnable` implementation to the `Thread` constructor. It
does not subclass `Thread` or override its method.

<br>

## 5. `start()` versus `run()`

These calls are different:

``` java
myThread.start();
```

-   Starts a new thread of execution.
-   The JVM schedules the new thread.
-   The new thread executes its task.

``` java
myThread.run();
```

-   Calls `run()` like a normal method.
-   Does not start a new thread.
-   The code executes on the thread that made the call.

Example:

``` java
public class Main {

    public static void main(String[] args) {

        Thread myThread = new Thread(() -> {
            System.out.println(
                "Executing on: " +
                Thread.currentThread().getName()
            );
        });

        myThread.run();
        myThread.start();
    }
}
```

Possible output:

``` text
Executing on: main
Executing on: Thread-0
```

The first call runs on the main thread. The second call starts a new
thread.

<br>

## 6. Important ideas to remember

-   `run()` contains the task's instructions.
-   `Runnable` represents a task that can be executed.
-   A `Thread` can receive a `Runnable` task through its constructor.
-   A lambda passed to `new Thread(...)` is a `Runnable` implementation.
-   `new Thread() { ... }` creates an anonymous subclass and overrides
    `run()`.
-   `start()` starts a new thread; calling `run()` directly does not.
-   Separating a task (`Runnable`) from its execution (`Thread`) is a
    useful design practice.

<br>
