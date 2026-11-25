# Java Concurrency --- Phase 4, Lesson 1

## Executor Framework and ExecutorService

**Learning path:** Beginner → Practical\
**Project context:** Spring Boot Turf Booking System

------------------------------------------------------------------------

## 1. Why do we need the Executor Framework?

You already know how to create and start a thread directly:

``` java
Thread thread = new Thread(() -> {
    System.out.println("Processing booking...");
});
thread.start();
```

This works. But a real booking backend may need background tasks such
as:

-   Sending booking confirmation emails.
-   Sending SMS notifications.
-   Processing payment follow-ups.
-   Generating booking reports.
-   Processing audit logs.
-   Sending booking reminders.

One approach would be to create a new thread for every task:

``` java
for (int i = 0; i < 1000; i++) {
    Thread thread = new Thread(() -> processBookingNotification());
    thread.start();
}
```

### Problems with creating a thread for every task

1.  **Thread creation has a cost.** Threads consume memory and system
    resources.
2.  **Concurrency becomes uncontrolled.** A burst of requests can create
    a very large number of threads.
3.  **Resource usage can become excessive.** Too many threads increase
    memory use, context switching, and scheduling overhead.
4.  **Manual management is harder.** Application code must manage
    creation, reuse, shutdown, and failures.

The Executor Framework separates task definition from task execution.
Instead of manually creating a thread for every task, you submit work to
an executor.

## 2. What is the Executor Framework?

The Executor Framework is a Java concurrency API that separates:

1.  **Task definition:** what work should be done.
2.  **Task submission:** asking an executor to run the work.
3.  **Execution and thread management:** deciding which worker thread
    runs the task.

### The main interfaces

  -----------------------------------------------------------------------
  Interface                           Purpose
  ----------------------------------- -----------------------------------
  `Executor`                          Base interface. Defines
                                      `execute(Runnable)`.

  `ExecutorService`                   Extends `Executor`. Adds task
                                      submission, `Future` support, and
                                      lifecycle management.

  `ScheduledExecutorService`          Extends `ExecutorService`. Supports
                                      delayed and periodic task
                                      execution.
  -----------------------------------------------------------------------

Relationship:

``` text
Executor
   |
   v
ExecutorService
   |
   v
ScheduledExecutorService
```

`ScheduledExecutorService` extends `ExecutorService`. It is not the
parent of `ExecutorService`.

## 3. Understanding Executor

`Executor` is the simplest interface. Its key method is:

``` java
void execute(Runnable command);
```

Example:

``` java
Executor executor = Executors.newSingleThreadExecutor();

executor.execute(() -> {
    System.out.println("Processing booking notification");
});
```

The caller submits a `Runnable`. The executor arranges for it to
execute.

`execute()` returns `void`, so it does not give the caller a `Future` to
track the task or retrieve a result.

## 4. Understanding ExecutorService

`ExecutorService` extends `Executor`. It provides more control over
submitted tasks and the executor's lifecycle.

### Important methods

  -----------------------------------------------------------------------
  Method                              Purpose
  ----------------------------------- -----------------------------------
  `execute()`                         Submit a `Runnable` task; returns
                                      nothing.

  `submit()`                          Submit a `Runnable` or `Callable`
                                      and receive a `Future`.

  `shutdown()`                        Stop accepting new tasks; allow
                                      submitted tasks to finish.

  `shutdownNow()`                     Attempt to interrupt running tasks
                                      and return waiting tasks.

  `isShutdown()`                      Check whether shutdown has started.

  `isTerminated()`                    Check whether all tasks have
                                      completed after shutdown.

  `awaitTermination()`                Wait for termination for a
                                      specified time.
  -----------------------------------------------------------------------

### A. Single-thread executor

``` java
ExecutorService executor =
        Executors.newSingleThreadExecutor();

executor.submit(() -> System.out.println("Task 1"));
executor.submit(() -> System.out.println("Task 2"));
executor.submit(() -> System.out.println("Task 3"));
```

A single-thread executor uses one worker thread. Tasks are processed
sequentially. For this executor, tasks are taken in submission order,
although task completion can depend on the work each task performs.

### B. Fixed thread pool

``` java
ExecutorService executor =
        Executors.newFixedThreadPool(5);

for (int i = 1; i <= 10; i++) {
    int taskId = i;
    executor.submit(() -> {
        System.out.println("Task " + taskId
            + " on " + Thread.currentThread().getName());
    });
}
```

A fixed pool has five worker threads.

-   At most five tasks execute at the same time.
-   Additional tasks wait until a worker becomes available.
-   Worker threads are reused.
-   The exact execution and completion order is not guaranteed.

### C. Cached thread pool

``` java
ExecutorService executor =
        Executors.newCachedThreadPool();
```

A cached thread pool creates threads as needed and reuses idle threads.
It can grow substantially during bursts of work, so it should not be
selected blindly for a production backend.

**Production note:** `Executors.newFixedThreadPool()` uses an unbounded
task queue. If work arrives faster than it can be processed, the queue
can grow and consume excessive memory. Pool sizing, queue capacity, and
rejection policies are covered in later lessons.

## 5. `execute()` versus `submit()`

  -----------------------------------------------------------------------
  `execute()`                         `submit()`
  ----------------------------------- -----------------------------------
  Accepts a `Runnable`.               Accepts a `Runnable` or `Callable`.

  Returns `void`.                     Returns a `Future`.

  No `Future` for tracking or         `Future` can track completion,
  retrieving a result.                retrieve a result, or request
                                      cancellation.

  Task exceptions are not returned to Exceptions from submitted tasks can
  the caller.                         be observed through `Future.get()`.
  -----------------------------------------------------------------------

Example:

``` java
executor.execute(() -> {
    System.out.println("Sending email");
});

Future<?> future = executor.submit(() -> {
    System.out.println("Sending email");
});
```

Remember:

-   `execute()` submits a task.
-   `submit()` submits a task and returns a `Future` representing its
    execution.

We will study `Future` in detail in Lesson 3.

## 6. Practical example: booking confirmation email

Suppose a user successfully books a turf. Email sending should happen in
the background rather than unnecessarily delaying the booking response.

``` java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BookingService {

    private final ExecutorService executor =
            Executors.newFixedThreadPool(3);

    public void createBooking() {
        // Main booking operation
        System.out.println("Booking created successfully");

        // Background task
        executor.execute(() -> {
            sendBookingConfirmationEmail();
        });

        System.out.println("Returning booking response");
    }

    private void sendBookingConfirmationEmail() {
        System.out.println("Sending email on: "
                + Thread.currentThread().getName());
    }

    public void shutdown() {
        executor.shutdown();
    }
}
```

The email task can run independently of the request thread.

However, submitting a task does not guarantee it will finish before the
application stops. The executor lifecycle must be managed.

## 7. Executor lifecycle management

### `shutdown()`

``` java
executor.shutdown();
```

This:

-   Stops accepting new tasks.
-   Allows tasks already submitted to finish.
-   Does not immediately interrupt running tasks.

Submitting a new task after shutdown normally causes
`RejectedExecutionException`:

``` java
executor.shutdown();

executor.execute(() -> {
    System.out.println("This task is rejected");
});
```

### `shutdownNow()`

``` java
List<Runnable> waitingTasks = executor.shutdownNow();
```

This:

-   Stops accepting new tasks.
-   Attempts to interrupt currently executing tasks.
-   Returns tasks that were waiting in the executor's queue.
-   Does not guarantee that running tasks stop immediately.

A running task must cooperate with interruption. For example:

``` java
executor.execute(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // Perform a unit of work
    }

    System.out.println("Task stopping");
});
```

### `awaitTermination()`

``` java
executor.shutdown();

try {
    boolean terminated = executor.awaitTermination(
            10, TimeUnit.SECONDS);

    if (!terminated) {
        executor.shutdownNow();
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

This pattern:

1.  Requests graceful shutdown.
2.  Waits up to ten seconds.
3.  Requests interruption if tasks have not completed.
4.  Restores the waiting thread's interrupted status if the wait is
    interrupted.

## 8. Using an executor in Spring Boot

Do not create a new thread pool inside every service-method invocation.
That can create many independent pools and excessive resource usage.

### A shared Spring-managed executor bean

``` java
@Configuration
public class AsyncConfig {

    @Bean(destroyMethod = "shutdown")
    public ExecutorService bookingExecutor() {
        return Executors.newFixedThreadPool(5);
    }
}
```

Inject it into the service:

``` java
@Service
public class BookingNotificationService {

    private final ExecutorService bookingExecutor;

    public BookingNotificationService(
            ExecutorService bookingExecutor) {
        this.bookingExecutor = bookingExecutor;
    }

    public void sendBookingEmail() {
        bookingExecutor.execute(() -> {
            System.out.println("Sending booking email");
        });
    }
}
```

Spring creates and manages the shared bean. Services inject and use it.
Individual services should not shut down a shared executor; lifecycle
management belongs to the application/container.

### Production consideration

Spring applications commonly use a configured `ThreadPoolTaskExecutor`
or a carefully configured `ThreadPoolExecutor`.

`Executors.newFixedThreadPool()` uses an unbounded task queue. If tasks
arrive faster than they can be processed, the queue can grow and consume
excessive memory.

We will study:

-   Thread pool sizing.
-   Queue capacity.
-   Rejection policies.
-   `ThreadPoolExecutor`.

These are covered in later lessons.

## 9. Real-world considerations for Turf

### Scenario A: Booking confirmation email

A sensible initial flow is:

1.  Validate the booking request.
2.  Save the booking in a database transaction.
3.  Commit the transaction.
4.  Submit notification work.
5.  Return the API response.

**Important:** Do not send a success email before the booking
transaction commits. Otherwise, the email could claim the booking
succeeded even if the transaction rolls back.

For stronger delivery reliability, consider a **transactional outbox**:
save the booking and an outbox event in the same database transaction. A
background worker later processes pending events.

### Scenario B: Multiple Spring Boot instances

``` text
Load Balancer
   |
   +---- App 1 ---- Pool 1 (5 workers)
   |
   +---- App 2 ---- Pool 2 (5 workers)
   |
   +---- App 3 ---- Pool 3 (5 workers)

Total: 15 local worker threads
```

Each application instance has its own executor and worker threads.

`ExecutorService` is local to its JVM; it is not a distributed task
queue.

### Scenario C: Preventing double booking

`ExecutorService` cannot prevent two users from booking the same turf
slot. It manages task execution, not distributed booking consistency.

Use database-level protection such as:

-   Unique constraints.
-   Transactions.
-   Appropriate locking where needed.

  -----------------------------------------------------------------------
  Problem                             Appropriate mechanism
  ----------------------------------- -----------------------------------
  Execute background tasks            `ExecutorService`

  Limit concurrent tasks              Thread pool

  Prevent double booking              Database constraints and
                                      transactions

  Coordinate across instances         Shared database or distributed
                                      coordination

  Reliably deliver background events  Transactional outbox and messaging
  -----------------------------------------------------------------------

## 10. Common mistakes and failure scenarios

  -----------------------------------------------------------------------
  Mistake                             Why it is problematic
  ----------------------------------- -----------------------------------
  Creating a pool inside every        Can create too many independent
  service method                      pools.

  Never shutting down a manually      Worker threads may remain alive.
  created executor                    

  Using cached pools without limits   Can create too many threads during
                                      bursts.

  Assuming `shutdownNow()` guarantees Tasks may ignore interruption.
  termination                         

  Ignoring an unbounded queue         Queued tasks can consume excessive
                                      memory.

  Assuming a pool is shared across    Each JVM has its own executor.
  instances                           

  Assuming async work is guaranteed   The application can stop before the
  to finish                           task finishes.

  Using an executor to prevent double It does not enforce database
  booking                             consistency.
  -----------------------------------------------------------------------

## 11. Interview questions and answers

**Q1. What is the Executor Framework?**

A Java concurrency framework that separates task submission from task
execution and thread management.

**Q2. What is the difference between Executor and ExecutorService?**

`Executor` provides `execute()`. `ExecutorService` extends it and adds
task submission, `Future` support, and lifecycle-management methods.

**Q3. What is the difference between `execute()` and `submit()`?**

`execute()` accepts a `Runnable` and returns nothing. `submit()` accepts
`Runnable` or `Callable` and returns a `Future`.

**Q4. What is the difference between `shutdown()` and `shutdownNow()`?**

`shutdown()` allows submitted tasks to finish and rejects new tasks.
`shutdownNow()` attempts to interrupt running tasks and returns waiting
tasks.

**Q5. Does ExecutorService create a new thread for every task?**

Not necessarily. It depends on the executor implementation. A
single-thread executor uses one worker; a fixed pool reuses a fixed
number of workers.

**Q6. Is ExecutorService thread-safe?**

The standard `ExecutorService` implementations supplied by Java are
designed for concurrent use by multiple threads.

**Q7. Is ExecutorService shared across multiple Spring Boot instances?**

No. Each application instance has its own executor.

**Q8. Can ExecutorService prevent two users from booking the same
slot?**

No. Booking correctness requires database-level consistency mechanisms.

**Q9. What happens if a task is submitted after `shutdown()`?**

It is rejected, normally with `RejectedExecutionException`.

**Q10. Why avoid creating an executor inside every service method?**

Each invocation could create another pool, resulting in excessive
threads and resource consumption.

## 12. Practice questions with answers

**1. What happens when you submit 20 tasks to a single-thread
executor?**

All 20 tasks execute sequentially using one worker thread. They do not
execute simultaneously.

**2. A fixed thread pool has four workers. You submit ten tasks. How
many can execute simultaneously?**

At most four tasks can execute simultaneously. The remaining tasks wait
in the queue.

**3. What happens to already submitted tasks after `shutdown()`?**

They are allowed to complete. The executor stops accepting new tasks.

**4. Can `shutdownNow()` guarantee every running task stops
immediately?**

No. It requests interruption. Running tasks must cooperate with
interruption.

**5. You have three Spring Boot instances, each with a pool of ten
threads. How many worker threads can those pools provide in total?**

Thirty worker threads across the three instances, assuming all three
pools are active.

**6. Can a local ExecutorService guarantee that only one user books a
particular slot across three application instances?**

No. Each instance has an independent executor. Booking consistency must
be enforced through shared infrastructure, particularly the database.

## 13. System design: Background notification processing for Turf

### Question

Design background notification processing for Turf.

Requirements:

-   Users can book turfs simultaneously.
-   Every successful booking should trigger an email.
-   Email processing should not unnecessarily delay the booking API
    response.
-   Multiple Spring Boot instances will eventually run.

### Recommended architecture

``` text
Client
  |
  | Booking request
  v
Spring Boot Booking API
  |
  | Validate request and save booking
  v
Database transaction
  |
  | Booking + outbox event committed together
  v
Background event processing
  |
  | Worker retrieves pending notification events
  v
ExecutorService / Worker Pool
  |
  | Controlled concurrent processing
  v
Email Provider
  |
  v
Confirmation email
```

### How the design works

**1. Receive and validate the booking.**

The API validates the requested slot and applies booking consistency
rules.

**2. Commit booking and outbox event together.**

The booking record and notification event are written in one database
transaction.

**3. Process pending events in the background.**

A worker claims pending outbox records using a safe mechanism so
multiple instances do not process the same event concurrently.

**4. Use a bounded worker pool.**

`ExecutorService` or a Spring-managed worker pool controls how many
tasks are processed concurrently.

**5. Send the email and record the outcome.**

Mark the event as processed after successful delivery, with retry
handling for transient failures.

### Why this design?

1.  The booking transaction commits before the notification is
    processed.
2.  The outbox reduces the risk of losing a notification between
    database commit and task submission.
3.  A worker pool controls concurrent notification processing.
4.  Multiple application instances can process events if event claiming
    is implemented safely.

### What should the first version implement?

Start with a shared, Spring-managed executor for straightforward
background work.

Introduce the transactional outbox when reliable delivery becomes a
requirement.

An in-memory executor alone cannot guarantee that work survives a
process crash.

## 14. Key takeaways

1.  Executor Framework separates task submission from thread management.
2.  Executor provides `execute()`.
3.  ExecutorService adds `submit()`, `Future` support, and lifecycle
    management.
4.  Thread pools reuse worker threads.
5.  `shutdown()` allows submitted tasks to finish.
6.  `shutdownNow()` attempts to interrupt running tasks.
7.  Spring Boot should use shared, Spring-managed executor beans.
8.  Executors are local to their JVM, not distributed.
9.  ExecutorService does not solve database consistency or guarantee
    reliable task delivery.

## 15. Lesson completion checklist

-   [ ] I understand why manually creating threads for every task is
    problematic.
-   [ ] I understand Executor and ExecutorService.
-   [ ] I understand `execute()` versus `submit()`.
-   [ ] I understand single-thread, fixed, and cached executors.
-   [ ] I understand `shutdown()`, `shutdownNow()`, and
    `awaitTermination()`.
-   [ ] I understand how Spring Boot can manage a shared executor.
-   [ ] I understand why an ExecutorService cannot prevent double
    booking.
-   [ ] I understand how executor pools behave across multiple
    application instances.

------------------------------------------------------------------------

## Next lesson

**Phase 4 · Lesson 2: Thread Pools**

Topics:

-   Worker threads.
-   How a thread pool works internally.
-   Core pool size and maximum pool size.
-   Task queues.
-   How tasks are assigned to workers.
-   Why unbounded queues can be dangerous.
-   `ThreadPoolExecutor` internals.
-   Configuring a thread pool for your Spring Boot Turf backend.

Lesson 1 established how tasks are submitted and how an executor is
managed. Lesson 2 focuses on controlling the resources that execute
those tasks.
