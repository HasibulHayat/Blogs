<br>

# Java Concurrency - Phase 3: Synchronization

<br>
<br>

## 1. First, understand the problem

Imagine you have a bank account with a balance of 1,000.

Two people try to withdraw money at almost the same time:

-   Person A withdraws 700.
-   Person B withdraws 500.

Your application should never allow both withdrawals to succeed, because
the total requested is 1,200 and the account only contains 1,000.

But a computer can run multiple threads at the same time. If both
threads read the balance before either updates it, they may both believe
enough money is available.

This is a concurrency problem.

<br>

### What can happen without protection?

```
Suppose the balance is 1,000.

  Step   Thread A                Thread B
  ------ ----------------------- -----------------------
  1      Reads balance: 1,000    
  2                              Reads balance: 1,000
  3      Checks: 1,000 \>= 700   
  4                              Checks: 1,000 \>= 500
  5      Writes balance: 300     
  6                              Writes balance: 500
```

Both withdrawals may report success, even though only one should have
been allowed.

This is an example of a **race condition**. The final result depends on
how the threads' operations interleave.

The important problem is that the check and the update belong together.
We do not want another thread interfering in the middle.

<br>

## 2. What is `synchronized`?

Java provides the `synchronized` keyword to control access to code that
uses a particular lock.

In simple words:

> A synchronized method allows only one thread at a time to execute that
> method using the same object's lock.

Think of a room with one key:

-   Thread A takes the key and enters.
-   Thread B wants to enter, but must wait.
-   Thread A finishes and releases the key.
-   Thread B can then enter.

This is called **mutual exclusion**: only one thread at a time can hold
that lock.

<br>

### Important detail

The lock belongs to an object, not to the method itself.

For an ordinary instance synchronized method, Java uses the current
object (`this`) as the lock.

<br>

## 3. Your first runnable example

Create a file named `SynchronizedExample.java` and paste this code:

``` java
public class SynchronizedExample {

    static class Counter {
        private int count = 0;

        public synchronized void increment() {
            count++;
        }

        public int getCount() {
            return count;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();

        Runnable task = () -> {
            for (int i = 0; i < 100_000; i++) {
                counter.increment();
            }
        };

        Thread thread1 = new Thread(task);
        Thread thread2 = new Thread(task);

        thread1.start();
        thread2.start();

        thread1.join();
        thread2.join();

        System.out.println("Final count: " + counter.getCount());
    }
}
```

<br>

### Expected output

``` text
Final count: 200000
```

You should get `200000` because:

-   Thread 1 increments the counter 100,000 times.
-   Thread 2 increments the same counter 100,000 times.
-   Both threads share the same `Counter` object.
-   The synchronized method allows only one thread at a time to execute
    `increment()` on that object.

<br>

### Try changing one thing

Remove `synchronized` from this line:

``` java
public synchronized void increment() {
```

Change it to:

``` java
public void increment() {
```

Run the program several times.

You may see a result below `200000`. You might occasionally see `200000`
too. The exact result is not guaranteed because thread scheduling
varies.

Why can increments be lost?

The expression:

``` java
count++;
```

is not one indivisible operation. It involves reading the current value,
calculating a new value, and writing it back. Two threads can interfere
with those steps.

Put `synchronized` back before continuing.

<br>

## 4. How does a synchronized instance method work?

Look at this method:

``` java
public synchronized void increment() {
    count++;
}
```

It is conceptually similar to writing:

``` java
public void increment() {
    synchronized (this) {
        count++;
    }
}
```

For an instance synchronized method:

1.  A thread tries to enter the method.
2.  It must acquire the lock belonging to the current object.
3.  If another thread already holds that same lock, it waits.
4.  When the thread exits the method, it releases the lock.
5.  Another waiting thread can then acquire the lock.

The lock is automatically released when the method exits, including when
it exits because an exception is thrown.

<br>

### One object means one shared lock

Consider:

``` java
Counter counter = new Counter();

Thread thread1 = new Thread(() -> counter.increment());
Thread thread2 = new Thread(() -> counter.increment());
```

Both threads call the method on the same object. They therefore compete
for the same lock.

But consider:

``` java
Counter counter1 = new Counter();
Counter counter2 = new Counter();
```

Each object has its own lock.

A thread using `counter1` does not need to wait for a thread using
`counter2` merely because both call the same synchronized method.

This is a very important point:

> Synchronization works between threads that use the same lock. It does
> not automatically lock every instance of a class.

<br>

## 5. Experiment: two objects, two locks

Run this example:

``` java
public class DifferentObjectsExample {

    static class Worker {
        public synchronized void doWork(String name) {
            System.out.println(name + " entered");

            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            System.out.println(name + " finished");
        }
    }

    public static void main(String[] args) {
        Worker worker1 = new Worker();
        Worker worker2 = new Worker();

        Thread thread1 = new Thread(
                () -> worker1.doWork("Thread 1")
        );

        Thread thread2 = new Thread(
                () -> worker2.doWork("Thread 2")
        );

        thread1.start();
        thread2.start();
    }
}
```

<br>

### What should you observe?

Both threads can enter their methods without waiting for the other,
because they use different objects and therefore different locks.

Their output may look like this:

``` text
Thread 1 entered
Thread 2 entered
Thread 1 finished
Thread 2 finished
```

The order can vary. The important observation is that both can be inside
their methods during overlapping periods.

<br>

### Now change the experiment

Replace:

``` java
Thread thread2 = new Thread(
        () -> worker2.doWork("Thread 2")
);
```

with:

``` java
Thread thread2 = new Thread(
        () -> worker1.doWork("Thread 2")
);
```

Now both threads use `worker1`.

One thread must finish the synchronized method before the other can
enter it. You should see the messages grouped by thread, with roughly
two seconds between the first thread entering and finishing, and the
second thread entering and finishing.

<br>

## 6. What if a synchronized method calls another synchronized method?

Java's intrinsic locks are **reentrant**.

That means a thread that already owns an object's lock can acquire that
same lock again.

Example:

``` java
public class ReentrantExample {

    public synchronized void firstMethod() {
        System.out.println("Inside firstMethod");
        secondMethod();
    }

    public synchronized void secondMethod() {
        System.out.println("Inside secondMethod");
    }

    public static void main(String[] args) {
        ReentrantExample example = new ReentrantExample();
        example.firstMethod();
    }
}
```

Expected output:

``` text
Inside firstMethod
Inside secondMethod
```

This does not deadlock.

Why? The thread already owns the object's lock when it enters
`firstMethod()`. It is allowed to enter `secondMethod()` on the same
object.

------------------------------------------------------------------------

## 7. What does `synchronized` protect?

It protects code only when the threads that need coordination use the
same lock.

For example:

``` java
class Account {
    private int balance = 1000;

    public synchronized boolean withdraw(int amount) {
        if (balance >= amount) {
            balance -= amount;
            return true;
        }

        return false;
    }

    public synchronized int getBalance() {
        return balance;
    }
}
```

Both methods use the same account object's lock.

This means a withdrawal and a balance read cannot execute their
synchronized sections at the same time on that account object.

Also notice that the availability check and balance update are inside
one synchronized method. Protecting only the subtraction would not be
enough.

### What `synchronized` does not protect

It does not automatically protect:

-   Access through a different object with a different lock.
-   Code that reads or writes the same field without acquiring the
    required lock.
-   Data shared between separate JVM processes.
-   A database row from another application instance.
-   An external payment provider or another remote service.

All participating code must follow the same locking strategy.

------------------------------------------------------------------------

## 8. Synchronized instance methods vs static synchronized methods

So far, we have used instance methods:

``` java
public synchronized void doWork() {
    // ...
}
```

An instance synchronized method locks the current object (`this`).

Java also supports static synchronized methods:

``` java
public static synchronized void doWork() {
    // ...
}
```

A static synchronized method locks the `Class` object associated with
that class.

For example:

``` java
public class Example {

    public synchronized void instanceMethod() {
        // Locks this object
    }

    public static synchronized void staticMethod() {
        // Locks Example.class
    }
}
```

These are different locks.

-   `instanceMethod()` uses the individual object's lock.
-   `staticMethod()` uses `Example.class` as the lock.

Different instances can run their instance synchronized methods at the
same time. Static synchronized calls on the same class share the
class-level lock.

Do not use static synchronization just because you want stronger
protection. Choose the lock that represents the shared state you are
protecting.

------------------------------------------------------------------------

## 9. A practical bank account example

Create `BankAccountExample.java`:

``` java
public class BankAccountExample {

    static class BankAccount {
        private int balance = 1000;

        public synchronized boolean withdraw(
                String customer,
                int amount
        ) {
            System.out.println(
                    customer + " is checking the balance..."
            );

            if (balance >= amount) {
                System.out.println(
                        customer + " can withdraw " + amount
                );

                balance -= amount;

                System.out.println(
                        customer + " completed withdrawal."
                );

                return true;
            }

            System.out.println(
                    customer + " does not have enough money."
            );

            return false;
        }

        public synchronized int getBalance() {
            return balance;
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        BankAccount account = new BankAccount();

        Thread customerA = new Thread(
                () -> account.withdraw("Customer A", 700)
        );

        Thread customerB = new Thread(
                () -> account.withdraw("Customer B", 500)
        );

        customerA.start();
        customerB.start();

        customerA.join();
        customerB.join();

        System.out.println(
                "Final balance: " + account.getBalance()
        );
    }
}
```

### Run it

``` bash
javac BankAccountExample.java
java BankAccountExample
```

### What should happen?

Only one withdrawal should succeed:

-   If Customer A withdraws first, the balance becomes 300. Customer B
    is rejected.
-   If Customer B withdraws first, the balance becomes 500. Customer A
    is rejected.

The final balance should be either:

``` text
Final balance: 300
```

or:

``` text
Final balance: 500
```

The order is not guaranteed, but the account should not allow both
withdrawals.

Try removing `synchronized` from `withdraw()` and run the program
several times. Because the example is small and scheduling varies, you
may not see the bug every time. A missing bug in one run does not prove
the code is thread-safe.

------------------------------------------------------------------------

## 10. How does this relate to the Turf booking system?

Imagine two users try to book the same turf slot at nearly the same
time.

A simplified in-memory example might look like this:

``` java
class Slot {
    private boolean booked = false;

    public synchronized boolean book() {
        if (booked) {
            return false;
        }

        booked = true;
        return true;
    }
}
```

If both request threads call `book()` on the same `Slot` object, the
synchronized method ensures that only one thread at a time performs the
check and update.

But this is only a Java-process-level example.

### Why is this not enough for the real Turf backend?

Imagine your Spring Boot application runs on three servers:

``` text
             Users
               |
          Load Balancer
          /     |     \
         /      |      \
      App 1   App 2   App 3
```

Each application instance has its own JVM memory.

A synchronized method in App 1 does not lock the corresponding object in
App 2 or App 3.

So two requests handled by different instances could still try to book
the same database slot.

For a real booking system, you will need database-level protection, such
as:

-   A database constraint that prevents overlapping or duplicate
    bookings, depending on your data model.
-   Transactions.
-   Appropriate row locking or another concurrency-control strategy.
-   Idempotency for retried requests.

We will study these backend topics later in the concurrency roadmap.

**Remember:** `synchronized` is useful for coordinating threads inside
one JVM. It is not a distributed lock.

------------------------------------------------------------------------

## 11. Common mistakes to avoid

### Mistake 1: Thinking the method itself has one global lock

Incorrect. An instance synchronized method locks the current object.
Different objects have different locks.

### Mistake 2: Synchronizing only one side

If one thread writes a field inside a synchronized block but another
thread reads it without using the same lock or another valid visibility
mechanism, the second thread is not protected by that synchronization.

### Mistake 3: Thinking `synchronized` makes every operation in a class safe

Only the code coordinated through the relevant lock receives the
protection. Other methods and fields may still have concurrency
problems.

### Mistake 4: Using `sleep()` to solve race conditions

`Thread.sleep()` only pauses a thread. It does not provide mutual
exclusion or the required visibility guarantees.

### Mistake 5: Using synchronized for work that should be coordinated across servers

A Java monitor exists inside one JVM. It cannot coordinate separate
application instances.

------------------------------------------------------------------------

## 12. Interview questions

### Q1. What is a synchronized method?

A method declared with `synchronized` requires a thread to acquire a
particular monitor lock before entering it. For an instance method, that
lock belongs to the current object.

### Q2. What happens when two threads call the same synchronized instance method on the same object?

One thread acquires the object's lock and enters. The other must wait
until the lock is released.

### Q3. What happens when two threads call the same synchronized method on two different objects?

They use different object locks, so they do not have to wait for each
other because of that method's synchronization.

### Q4. What lock does a static synchronized method use?

It uses the `Class` object for the declaring class.

### Q5. Is `count++` atomic?

No. It is a read-modify-write operation and can be interrupted by
another thread's operations.

### Q6. Does synchronized provide visibility as well as mutual exclusion?

Yes. Releasing a monitor happens-before a subsequent acquisition of the
same monitor. This provides visibility for changes made before the
release.

### Q7. Is synchronized enough to prevent double booking in a multi-instance Spring Boot application?

No. It only coordinates threads that use the same lock inside one JVM.
Database-level coordination is required across application instances.

------------------------------------------------------------------------

## 13. Practice exercises

Try these yourself before looking for solutions.

### Exercise 1: Shared counter

1.  Run `SynchronizedExample`.
2.  Remove `synchronized`.
3.  Run it several times.
4.  Add `synchronized` again.
5.  Explain why the final count is reliable with synchronization.

### Exercise 2: Different objects

1.  Run `DifferentObjectsExample` with two different `Worker` objects.
2.  Change it so both threads use the same `Worker` object.
3.  Compare the output.
4.  Explain which lock each thread needs to acquire.

### Exercise 3: Bank withdrawals

1.  Run `BankAccountExample`.
2.  Confirm that only one withdrawal succeeds.
3.  Remove `synchronized`.
4.  Run it several times.
5.  Explain why the check and update should be protected together.

### Exercise 4: Reentrant locks

1.  Run `ReentrantExample`.
2.  Add another synchronized method.
3.  Call it from `firstMethod()`.
4.  Explain why the same thread can enter both methods without
    deadlocking.

------------------------------------------------------------------------

## 14. Final summary

The most important ideas from this lesson are:

1.  `synchronized` helps multiple threads coordinate access to shared
    data.
2.  A synchronized instance method locks the current object (`this`).
3.  Only one thread at a time can hold that object's monitor lock.
4.  Different objects have different instance locks.
5.  A static synchronized method locks the class's `Class` object.
6.  Java intrinsic locks are reentrant.
7.  Synchronization provides both mutual exclusion and visibility when
    threads use the same lock correctly.
8.  Synchronization inside one JVM does not protect data across multiple
    application instances.

**One sentence to remember:**

> A synchronized instance method lets only one thread at a time execute
> that method using the same object's lock.

## What's next?

**Phase 3 --- Lesson 2: `synchronized` Blocks**

In the next lesson, you will learn how to synchronize only a specific
part of a method instead of locking the entire method.
