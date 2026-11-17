# Java Concurrency — Phase 2, Lesson 1: Understanding Shared Data

## Introduction

In Phase 1, we learned how to create, start, interrupt, and manage individual Java threads.

Now we are moving into a central challenge of concurrency:

**What happens when multiple threads access the same data?**

This lesson introduces shared data using simple Java examples and connects the concept to a Spring Boot booking application.

---

## 1. What is shared data?

Shared data is data that can be accessed by multiple threads.

For example:

```java
int bookingCount = 0;
```

If Thread A and Thread B can both access this same variable, it is shared data.

The threads are separate, but they can access the same piece of data.

```text
             Shared variable
            bookingCount = 0
                 /   \
                /     \
          Thread A   Thread B
```

## 2. A first shared-data example

Create a class with a counter:

```java
public class Counter {

    int count = 0;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```

Now access the same `Counter` object from two threads:

```java
public class Main {

    public static void main(String[] args) throws InterruptedException {

        Counter counter = new Counter();

        Thread threadA = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        });

        Thread threadB = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        });

        threadA.start();
        threadB.start();

        threadA.join();
        threadB.join();

        System.out.println(counter.getCount());
    }
}
```

We might expect the result to be:

```text
2000
```

Each thread performs 1,000 increments.

However, the result is **not guaranteed** to be 2,000. It may be smaller because both threads update the same variable without coordination.

This is a concurrency problem called a **race condition**, which we will study in the next lesson.

## 3. Why can `count++` be dangerous?

This line looks like one operation:

```java
count++;
```

But conceptually it involves three steps:

1. Read the current value of `count`.
2. Add one to that value.
3. Write the new value back to `count`.

Suppose `count` is initially 10:

| Step | Thread A | Thread B |
|---:|---|---|
| 1 | Reads 10 | |
| 2 | | Reads 10 |
| 3 | Calculates 11 | |
| 4 | | Calculates 11 |
| 5 | Writes 11 | |
| 6 | | Writes 11 |

The final value is 11, even though two increments were attempted. One update was lost.

This is called a **lost update**.

The important lesson is that a line of Java code is not necessarily one indivisible operation.

## 4. Are all variables shared?

No. Whether data is shared depends on where it lives and whether multiple threads can access the same data.

### A. Local variables

```java
public void process() {
    int amount = 500;
}
```

Each method invocation has its own local `amount`.

If two threads independently call `process()`, they each have their own local variable.

```text
Thread A                 Thread B
amount = 500              amount = 500
(its own variable)        (its own variable)
```

These local variables are not shared with each other.

**Important nuance:** A local variable can refer to an object that is shared. The reference variable is local, but the referenced object may be accessible to multiple threads.

### B. Instance variables

```java
public class BookingService {
    private int bookingCount = 0;
}
```

An instance variable belongs to an object.

If multiple threads access the same `BookingService` object, they can access the same `bookingCount` field.

An instance variable is not automatically shared just because it is an instance variable. Multiple threads must be able to reach the same object.

### C. Static variables

```java
public class BookingService {
    static int totalBookings = 0;
}
```

A static variable belongs to a class rather than an individual instance.

Threads accessing that static variable through the same class definition and class loader access the same variable.

Static data is therefore often shared, but it still depends on which class definition is involved.

## 5. Objects can be shared too

Shared data is not limited to primitive variables such as `int`, `long`, or `boolean`.

Objects can also be shared:

```java
public class Booking {

    private String status;

    public Booking(String status) {
        this.status = status;
    }

    public void confirm() {
        status = "CONFIRMED";
    }
}
```

Suppose two threads receive the same `Booking` object:

```java
Booking booking = new Booking("PENDING");

Thread threadA = new Thread(() -> {
    booking.confirm();
});

Thread threadB = new Thread(() -> {
    System.out.println(booking);
});
```

Both threads access the same object. One modifies it while another accesses it.

In a real booking application, shared objects may include:

- Booking objects.
- Payment status objects.
- User session state.
- In-memory availability caches.
- Shared counters.
- Shared configuration objects.

## 6. How this relates to Spring Boot

This concept is particularly important in a Spring Boot application.

Consider:

```java
@Service
public class BookingService {

    private int totalBookings = 0;

    public void createBooking() {
        totalBookings++;
    }
}
```

Spring beans are singleton-scoped by default when declared with `@Service`, `@Component`, or similar stereotypes.

That means Spring normally creates one `BookingService` instance and makes it available to multiple request-handling threads.

Two users may send booking requests at nearly the same time. Both requests can access the same service instance and its `totalBookings` field.

### A dangerous example

```java
@Service
public class BookingService {

    private Long currentUserId;

    public void createBooking(Long userId) {
        currentUserId = userId;

        // Process booking...
    }
}
```

Imagine:

1. User A calls `createBooking(101)`.
2. The service stores `101` in `currentUserId`.
3. Before User A finishes, User B calls `createBooking(202)`.
4. The service overwrites `currentUserId` with `202`.
5. User A's operation may now read the wrong user ID.

This is dangerous shared mutable state.

### A safer starting point

```java
@Service
public class BookingService {

    public void createBooking(Long userId) {
        Long currentUserId = userId;

        // Process booking...
    }
}
```

The local variable belongs to that method invocation, so another request does not overwrite that local variable.

However, remember that objects referenced by local variables may still be shared. Local variables do not magically make every object they reference thread-safe.

## 7. Is shared data always bad?

No. Shared data is normal and often necessary.

Examples include:

- Multiple requests reading application configuration.
- Multiple threads accessing a cache.
- Workers using a shared task queue.
- Services using a shared database connection pool.

The problem is not sharing data by itself.

The real concern is **unsafely accessing mutable shared data**.

Later lessons will teach us how to coordinate access and make shared state safe.

## 8. Important things to remember

1. Shared data is data accessible by multiple threads.
2. Multiple threads can share the same object.
3. Instance variables can be shared when multiple threads access the same object.
4. Static variables can be shared by threads accessing the same class definition.
5. Local variables belong to individual method invocations.
6. A local variable can still reference a shared object.
7. `count++` is not an atomic operation.
8. Spring singleton beans can be accessed concurrently by multiple request threads.
9. Avoid mutable request-specific fields in singleton services.
10. Sharing mutable data requires careful concurrency control.

---

## 9. Knowledge check

Try answering these questions before checking the answers.

1. Two threads call a method that declares `int amount = 500` locally. Do they share that local variable?
2. Two threads access the same `BookingService` object. Is its instance field potentially shared?
3. Why can `count++` lose an update?
4. Can a Spring singleton service bean be accessed by multiple request threads concurrently?
5. Which statement is correct?
   - All shared data is unsafe.
   - Only static variables can be shared.
   - Mutable shared data needs proper concurrency control.
   - Local variables can never reference shared objects.

### Answers

1. No. Each method invocation has its own local variable, although referenced objects may still be shared.
2. Yes. Both threads can access the same object's instance field.
3. Because its read, increment, and write steps can interleave between threads.
4. Yes. Singleton scope means one shared bean instance, not one request at a time.
5. Mutable shared data needs proper concurrency control.

---

## Next lesson — Phase 2, Lesson 2: Race Conditions

Next, we will investigate the counter example more deeply.

We will learn:

- What exactly is a race condition?
- How thread interleaving causes unexpected results.
- What a lost update is.
- Why the same program can produce different results on different runs.
- How to reproduce and observe a race condition.

**Goal:** Understand the problem thoroughly before learning how to fix it with synchronization and atomic operations.
