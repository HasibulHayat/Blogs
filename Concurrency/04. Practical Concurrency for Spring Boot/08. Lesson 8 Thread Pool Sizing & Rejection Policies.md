<br>

<br>

# Practical Concurrency for Spring Boot - Phase 4, Lesson 8 : Thread Pool Sizing and Rejection Policies

<br>

<br>

## 1. What You Will Learn

In Lesson 2, you learned what a thread pool is and how `ThreadPoolExecutor` manages worker threads and a task queue.

In this lesson, we will go deeper into two practical questions:

1. How many worker threads should your application use?
2. What should happen when the executor can no longer accept more tasks?

You will learn:

- What CPU-bound and I/O-bound tasks are.
- What `corePoolSize` and `maximumPoolSize` mean.
- How the work queue affects thread pool behavior.
- What happens when the queue becomes full.
- What a rejection policy is.
- The four built-in rejection policies.
- How to configure a Spring `ThreadPoolTaskExecutor`.
- How to think about executor sizing for your Turf Booking System.
- Common mistakes, interview questions, practice questions, and system design scenarios.

The goal is not to memorize one magic thread count. The goal is to understand how to choose a reasonable configuration, measure it, and protect your application when demand exceeds capacity.

<br>

<br>

## 2. Why Thread Pool Sizing Matters

Imagine your Turf Booking System receives requests that trigger background tasks:

- Sending booking confirmation emails.
- Sending SMS notifications.
- Calling an external payment provider.
- Generating reports.
- Processing uploaded files.
- Running scheduled maintenance tasks.

Each task needs resources to run.

A worker thread consumes memory and CPU scheduling resources. Some tasks also wait for databases, network responses, or external services.

If you create too few worker threads, tasks may wait in a queue for a long time.

If you create too many worker threads, the application may waste memory, spend too much time switching between threads, overload the database, or overload external services.

If you allow the queue to grow without a limit, a large backlog can consume memory and make the application slow or unstable.

So thread pool sizing is about balancing:

- Number of worker threads.
- Queue capacity.
- Task duration.
- CPU and memory available.
- Database connection capacity.
- External service limits.
- Expected traffic and bursts.
- How the application should behave when overloaded.

There is no universally correct thread count. You choose an initial configuration, load-test it, monitor it, and adjust it.

<br>

<br>

## 3. Understand CPU-Bound and I/O-Bound Tasks

Before sizing a pool, identify what kind of work its tasks perform.

<br>

<br>

### 3.1 CPU-Bound Tasks

A CPU-bound task spends most of its time performing calculations.

Examples:

- Calculating a large report.
- Compressing a large file.
- Processing an image.
- Performing complex pricing calculations over a large dataset.
- Generating a computationally expensive export.

The task is actively using the CPU for much of its execution.

For CPU-bound work, creating far more worker threads than available CPU capacity often does not improve throughput. It can make performance worse because threads compete for CPU time and the operating system spends time switching between them.

A reasonable starting point is often around the number of available processor cores, followed by measurement. The best value depends on the workload and the rest of the application.

You can inspect the number of processors visible to the JVM:

```java
int processors = Runtime.getRuntime().availableProcessors();

System.out.println("Available processors: " + processors);
```

This reports the processors available to the JVM environment. In containers, the value can depend on the JVM version and container CPU configuration.

<br>

<br>

### 3.2 I/O-Bound Tasks

An I/O-bound task spends a significant part of its time waiting for another resource.

Examples:

- Calling a payment provider over HTTP.
- Sending an email through an external provider.
- Calling an SMS API.
- Reading from or writing to a database.
- Downloading a file from another service.

While a task waits for a network or database response, its worker thread may not be actively using the CPU.

An I/O-bound workload may benefit from having more concurrent tasks than the number of CPU cores. But that does not mean you should create unlimited threads.

The database, remote service, connection pool, and memory still have limits.

<br>

<br>

### 3.3 Compare the Two

| Workload | Main resource being used | Initial sizing consideration |
|---|---|---|
| CPU-bound | CPU time | Start near available CPU capacity and measure |
| I/O-bound | Network, database, or other waiting resources | More concurrent workers may help, but respect downstream limits |

Do not use one shared executor for every kind of work without considering the consequences.

For example, a long-running report task should not be able to occupy every worker needed for time-sensitive booking notifications.

Separate executors can help isolate different workloads.

<br>

<br>

## 4. Understand the Main ThreadPoolExecutor Settings

The Java `ThreadPoolExecutor` constructor accepts several important settings.

A simplified constructor looks like this:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4,
        8,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(100)
);
```

This constructor uses default thread factory and rejection handler settings.

Let's understand the important parameters.

<br>

<br>

### 4.1 corePoolSize

```java
4
```

This is the core pool size.

The executor generally tries to maintain this number of worker threads, even when they are idle, unless core thread timeout is enabled.

When a task arrives and fewer than `corePoolSize` workers are running, the executor normally creates a worker to handle the task rather than putting the task into the queue first.

In this example:

```java
corePoolSize = 4
```

The executor can create up to four core workers as tasks arrive.

<br>

<br>

### 4.2 maximumPoolSize

```java
8
```

This is the maximum number of worker threads the executor can use.

The executor does not immediately create eight threads.

The usual task-submission sequence is:

1. If the number of workers is below `corePoolSize`, create a worker for the task.
2. Otherwise, try to put the task into the work queue.
3. If the queue cannot accept the task, try to create another worker, up to `maximumPoolSize`.
4. If the executor has reached its maximum size and the queue cannot accept the task, apply the rejection policy.

For this example:

```java
corePoolSize = 4
maximumPoolSize = 8
```

The executor can use between four and eight workers as demand and queue capacity require, subject to the executor's lifecycle and configuration.

<br>

<br>

### 4.3 keepAliveTime

```java
60
```

This is the amount of time that excess idle workers may remain alive before being terminated.

By default, this mainly applies to workers above the core pool size.

For example:

- Core size: 4.
- Maximum size: 8.
- Current workers: 8.
- Workload decreases.
- Four excess workers become idle.
- After the keep-alive period, those excess workers can be removed.

Core workers are not normally removed by this setting unless core thread timeout is enabled.

<br>

<br>

### 4.4 workQueue

```java
new ArrayBlockingQueue<>(100)
```

This is a bounded queue with a capacity of 100 tasks.

When all core workers are busy, newly submitted tasks are normally placed in this queue.

If the queue becomes full, the executor can create additional workers beyond the core size, up to the maximum size.

If the queue is full and the executor has reached the maximum number of workers, the rejection policy is used.

The queue is a very important part of thread pool sizing.

<br>

<br>

## 5. Understand the Task Submission Sequence

Let's use this configuration:

```java
corePoolSize = 2
maximumPoolSize = 4
queueCapacity = 3
```

Assume each task takes long enough that workers remain busy while more tasks are submitted.

The sequence is:

<box border radius="lg" padding={3} gap={3}>
  **Task submission behavior**

  <list connector="solid" marker="number" gap={2}>
    <list-item>
      **Task 1 arrives**

      Worker 1 is created and starts Task 1.
    </list-item>
    <list-item>
      **Task 2 arrives**

      Worker 2 is created and starts Task 2.
    </list-item>
    <list-item>
      **Tasks 3, 4, and 5 arrive**

      Both core workers are busy. These three tasks enter the queue.
    </list-item>
    <list-item>
      **Task 6 arrives**

      The queue is full. The executor creates Worker 3, which starts Task 6.
    </list-item>
    <list-item>
      **Task 7 arrives**

      The queue is still full. The executor creates Worker 4, which starts Task 7.
    </list-item>
    <list-item>
      **Task 8 arrives**

      The queue is full and all four workers are busy. The executor applies its rejection policy.
    </list-item>
  </list>
</box>

This sequence is one of the most important things to understand about `ThreadPoolExecutor`.

With this configuration, the executor can have:

- 4 active workers.
- 3 queued tasks.

That is a maximum of 7 submitted tasks being actively handled or waiting in the queue at that moment. Additional submissions can be rejected.

The exact moment at which a task starts depends on scheduling and task completion, but the submission rules above describe the executor's normal decision process.

<br>

<br>

## 6. Why Queue Capacity Matters

A queue stores tasks that have been accepted but have not started running yet.

Consider these two configurations:

```java
new LinkedBlockingQueue<>();
```

and:

```java
new ArrayBlockingQueue<>(100);
```

The first is effectively unbounded for practical application purposes.

The second has a fixed capacity of 100.

<br>

<br>

### 6.1 Unbounded Queue

With an unbounded queue, tasks can continue accumulating while the core workers are busy.

Because the queue does not become full under normal memory conditions, the executor generally has no reason to grow beyond its core pool size.

This has two consequences:

1. The executor may keep only the core number of workers.
2. The backlog can grow significantly when tasks arrive faster than they are processed.

A large backlog can increase memory usage and make tasks wait for a long time.

For example, `Executors.newFixedThreadPool(10)` uses a fixed number of workers with an unbounded work queue.

That is why a fixed thread pool can still have an overload problem.

<br>

<br>

### 6.2 Bounded Queue

A bounded queue limits how many waiting tasks can be stored.

Example:

```java
new ArrayBlockingQueue<>(100)
```

When the queue fills up, the executor can grow beyond its core size, up to its maximum size.

If the queue remains full and the executor reaches its maximum size, new tasks are rejected according to the configured policy.

A bounded queue gives you a defined limit on queued work.

It does not automatically make the application safe. You still need to choose a sensible capacity and a suitable rejection policy.

<br>

<br>

### 6.3 How Should You Choose Queue Capacity?

Do not choose a queue size only because a tutorial uses 100 or 1,000.

Consider:

- How much memory each queued task may retain.
- How long tasks normally take.
- How much waiting time is acceptable.
- How quickly tasks arrive during traffic spikes.
- Whether tasks are time-sensitive.
- What should happen if the application cannot keep up.

For a user-facing operation, a task waiting several minutes may be worse than rejecting it quickly and returning a controlled response.

For some background tasks, a durable external queue may be more appropriate than keeping a large backlog in application memory.

<br>

<br>

## 7. What Is a Rejection Policy?

A rejection policy defines what happens when an executor cannot accept a new task.

For a `ThreadPoolExecutor`, this normally happens when:

- The executor has been shut down; or
- The worker count has reached `maximumPoolSize` and the work queue is full.

Java provides four built-in rejection policies:

1. `AbortPolicy`
2. `CallerRunsPolicy`
3. `DiscardPolicy`
4. `DiscardOldestPolicy`

Let's understand each one.

<br>

<br>

## 8. AbortPolicy

This is the default rejection policy for a normal `ThreadPoolExecutor` if no other handler is specified.

Example:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(3),
        new ThreadPoolExecutor.AbortPolicy()
);
```

When the executor cannot accept a task, it throws:

```java
RejectedExecutionException
```

Example:

```java
try {
    executor.execute(() -> {
        System.out.println("Task running");
    });
} catch (RejectedExecutionException exception) {
    System.out.println("Task was rejected");
}
```

Why can this be useful?

- The caller knows that the task was not accepted.
- The application can log the rejection.
- The application can return a controlled error or take another action.
- It avoids silently pretending that the task was accepted.

For important work, this is often easier to reason about than silently discarding tasks.

But the application must actually handle the exception at an appropriate boundary.

<br>

<br>

## 9. CallerRunsPolicy

With `CallerRunsPolicy`, the task is executed by the thread that submitted it when the executor rejects the task, provided the executor has not been shut down.

Example:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(3),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

Suppose a Spring MVC request thread submits a background task.

If the executor is saturated:

- The executor cannot accept the task.
- `CallerRunsPolicy` runs the task on the submitting request thread.
- The request thread is now busy performing that work.

This creates a form of backpressure: the producer slows down because it has to perform the work itself.

However, this can have important consequences.

If a task takes five seconds, the request thread may spend five seconds running it.

That can increase HTTP response times and tie up request-handling threads.

Also, if the executor has been shut down, `CallerRunsPolicy` does not run the task; it discards it.

Use this policy only when running the task on the submitting thread is safe and acceptable.

<br>

<br>

## 10. DiscardPolicy

With `DiscardPolicy`, the rejected task is silently discarded.

Example:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(3),
        new ThreadPoolExecutor.DiscardPolicy()
);
```

If the executor cannot accept a task:

- No exception is thrown by this policy.
- The rejected task is dropped.
- The caller may not know that the task was lost.

This is dangerous for important work.

Do not casually use it for:

- Payment processing.
- Booking confirmation.
- Required customer notifications.
- Financial records.
- Tasks that must eventually complete.

It may be appropriate only for work that is explicitly allowed to be dropped, and even then you should consider metrics or other observability.

<br>

<br>

## 11. DiscardOldestPolicy

With `DiscardOldestPolicy`, when a task is rejected and the executor is still running:

1. The executor removes the task at the head of the queue.
2. It tries to submit the new task again.

The task removed from the queue is the oldest waiting task.

Example:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(3),
        new ThreadPoolExecutor.DiscardOldestPolicy()
);
```

This policy prioritizes newer queued work over the oldest queued work.

But it can silently lose an older task.

It is not suitable when every task matters.

Also, the retry of submission can happen repeatedly under sustained saturation, so this policy should be selected only when its behavior is appropriate for the workload.

<br>

<br>

## 12. Compare the Four Rejection Policies

| Policy | What happens when saturated? | Main concern |
|---|---|---|
| `AbortPolicy` | Throws `RejectedExecutionException` | Caller must handle rejection |
| `CallerRunsPolicy` | Submitting thread runs the task | Can slow down request threads |
| `DiscardPolicy` | Drops the new task | Task may be silently lost |
| `DiscardOldestPolicy` | Removes oldest queued task and retries submission | An older task may be lost |

There is no universally best policy.

Choose based on what the task means to your application.

For important business operations, silently losing work is usually unacceptable. Consider a durable queue or outbox rather than relying on an in-memory executor to retain critical work.

<br>

<br>

## 13. Configure a ThreadPoolExecutor Directly

Here is a complete Java example:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4,                              // corePoolSize
        8,                              // maximumPoolSize
        60,                             // keepAliveTime
        TimeUnit.SECONDS,               // time unit
        new ArrayBlockingQueue<>(100),  // queue capacity
        new ThreadPoolExecutor.AbortPolicy()
);
```

What each value means:

| Setting | Value | Meaning |
|---|---:|---|
| `corePoolSize` | 4 | Normally maintained core worker count |
| `maximumPoolSize` | 8 | Maximum worker count |
| `keepAliveTime` | 60 seconds | Idle time allowed for excess workers |
| Queue capacity | 100 | Maximum waiting tasks |
| Rejection policy | Abort | Throw when a task cannot be accepted |

You can submit a task:

```java
executor.execute(() -> {
    System.out.println("Background task running");
});
```

And shut down the executor when its owner is finished with it:

```java
executor.shutdown();
```

In a Spring Boot application, you normally define the executor as a Spring-managed bean and let the application lifecycle manage its shutdown. Do not shut down a shared application executor after each request.

<br>

<br>

## 14. Configure ThreadPoolTaskExecutor in Spring Boot

In Spring Boot, you will often configure a `ThreadPoolTaskExecutor` bean.

Example:

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "notificationExecutor")
    public ThreadPoolTaskExecutor notificationExecutor() {

        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setKeepAliveSeconds(60);

        executor.setThreadNamePrefix("notification-");

        executor.setRejectedExecutionHandler(
                new ThreadPoolExecutor.AbortPolicy()
        );

        executor.initialize();

        return executor;
    }
}
```

Let's understand the important parts.

**`@Configuration`**

Marks this class as a Spring configuration class.

**`@EnableAsync`**

Enables Spring's asynchronous method execution support, including `@Async`.

**`@Bean`**

Registers the executor as a Spring-managed bean.

**`setCorePoolSize(4)`**

Sets the core worker count to four.

**`setMaxPoolSize(8)`**

Sets the maximum worker count to eight.

**`setQueueCapacity(100)`**

Sets the queue capacity to 100.

**`setKeepAliveSeconds(60)`**

Sets the keep-alive time for excess idle workers.

**`setThreadNamePrefix("notification-")`**

Gives worker threads recognizable names, which helps when reading logs and thread dumps.

**`setRejectedExecutionHandler(...)`**

Configures what happens when the executor cannot accept more work.

**`initialize()`**

Initializes the Spring executor.

Spring manages the bean lifecycle. Do not create and shut down this executor inside each service method.

<br>

<br>

## 15. Use the Named Executor With @Async

You can select the executor by name:

```java
@Service
public class NotificationService {

    @Async("notificationExecutor")
    public void sendBookingConfirmation(Long bookingId) {

        // Send notification
        System.out.println("Sending notification for " + bookingId);
    }
}
```

When Spring invokes this method through its async proxy, the method is submitted to the configured executor.

Important Spring details:

- The call must go through the Spring proxy for `@Async` to apply.
- Calling an `@Async` method from another method in the same class (self-invocation) normally bypasses the proxy, so it will not run asynchronously as expected.
- The caller does not automatically wait for a `void` async method to finish.
- Exceptions from a `void` `@Async` method need appropriate async exception handling and logging.
- If you need a result, use a supported return type such as `CompletableFuture<T>`.

<br>

<br>

## 16. Use an Explicit Executor With CompletableFuture

You can also pass the configured executor directly:

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(
                () -> "Booking notification prepared",
                notificationExecutor
        );
```

This tells `CompletableFuture` to use the specified executor for the asynchronous task.

Without the executor argument:

```java
CompletableFuture.supplyAsync(() -> "Booking notification prepared");
```

the task normally uses `ForkJoinPool.commonPool()`.

For application-specific blocking work, using an appropriately configured Spring-managed executor is often easier to control than placing everything on the common pool.

<br>

<br>

## 17. How Should You Size Executors for the Turf Project?

Do not start by creating one enormous executor.

First, identify different categories of work.

For example:

| Workload | Nature of work | Design consideration |
|---|---|---|
| Booking request processing | Mostly request and database work | Managed by the web server and database connection pool; do not move everything to background threads |
| Email notifications | Usually I/O-bound | Separate executor or durable messaging |
| SMS notifications | I/O-bound | Respect provider rate limits and timeouts |
| Payment provider calls | I/O-bound and business-critical | Explicit timeouts, idempotency, provider limits, and careful state handling |
| Report generation | May be CPU- and memory-intensive | Isolate from latency-sensitive tasks |
| File processing | Depends on file size and processing type | Limit concurrency and memory usage |

A useful starting design could use separate executors for distinct workloads:

- `notificationExecutor`
- `reportExecutor`
- `integrationExecutor`

These are example names, not a requirement to create all three immediately.

The important reason for separation is isolation.

For example, if report generation occupies all workers in a shared pool, booking notification tasks may be delayed. A separate notification executor can prevent report tasks from consuming its worker capacity.

But separate pools also consume resources. Do not create many pools without a clear workload reason.

<br>

<br>

## 18. Understand Downstream Limits

Your executor is not the only component with a capacity limit.

Imagine you configure:

```java
executor.setMaxPoolSize(50);
```

But your database connection pool allows only 10 connections.

If all 50 tasks need a database connection at the same time:

- Some tasks may wait for a connection.
- Database connection acquisition time may increase.
- The executor's worker threads may remain occupied while waiting.
- Other tasks in the same executor may be delayed.

Similarly, an external SMS provider may allow only a limited number of requests per second.

Increasing your thread pool size does not increase the provider's allowed request rate.

Your concurrency should respect downstream capacity:

- Database connection pool size.
- HTTP connection pool size.
- Payment provider limits.
- SMS and email provider limits.
- CPU and memory limits.

Thread pool sizing must be considered together with these resources.

<br>

<br>

## 19. Monitoring: How Do You Know the Pool Is Too Small or Too Large?

Do not rely only on assumptions. Monitor the executor.

Useful `ThreadPoolExecutor` metrics include:

| Metric | What it tells you |
|---|---|
| `getPoolSize()` | Current number of worker threads |
| `getActiveCount()` | Approximate number of active workers |
| `getCorePoolSize()` | Configured core size |
| `getMaximumPoolSize()` | Configured maximum size |
| `getQueue().size()` | Number of tasks waiting in the queue |
| `getCompletedTaskCount()` | Approximate number of completed tasks |
| `getTaskCount()` | Approximate total number of tasks handled or scheduled |

You should also measure:

- Task execution duration.
- Queue waiting time.
- Rejected task count.
- Task failure count.
- Timeout count.
- Downstream connection wait time.

Possible warning signs:

**Queue size keeps growing**

Tasks are arriving faster than they are being completed, or workers are blocked for too long.

**Rejections happen frequently**

The executor is reaching its configured capacity. You need to investigate whether to increase capacity, reduce task demand, improve task duration, or use a durable queue.

**Workers are often idle**

The pool may have more capacity than needed, although bursts and latency requirements also matter.

**Task duration increases**

The bottleneck may be a slow database, slow external service, CPU saturation, or lock contention rather than simply too few threads.

In Spring Boot, Micrometer and Spring Boot Actuator can help expose executor and application metrics when configured appropriately.

<br>

<br>

## 20. Common Mistakes

<br>

### Mistake 1: Increasing Maximum Pool Size Without Understanding the Bottleneck

More threads do not automatically mean more throughput.

If the database or external provider is already overloaded, more concurrent requests can make the problem worse.

<br>

### Mistake 2: Using an Unbounded Queue for Every Executor

An unbounded queue can accumulate a large backlog and consume memory.

It can also prevent a `ThreadPoolExecutor` from growing beyond its core size under normal queueing behavior.

<br>

### Mistake 3: Assuming corePoolSize Is the Maximum Number of Threads

It is not.

`maximumPoolSize` controls the upper limit. The executor can grow beyond the core size when the queue is full and additional workers can be created.

<br>

### Mistake 4: Thinking the Executor Immediately Creates maximumPoolSize Threads

It does not.

The executor normally creates core workers first, queues tasks, and only grows beyond the core size when the queue cannot accept more tasks.

<br>

### Mistake 5: Using DiscardPolicy for Important Work

The task can be silently lost.

Do not use it for business-critical work unless losing the task is explicitly acceptable.

<br>

### Mistake 6: Using CallerRunsPolicy Without Considering Request Latency

A request thread may end up performing a slow background task.

That can increase response times and consume web request capacity.

<br>

### Mistake 7: Sharing One Pool Between Unrelated Workloads

Long-running reports can delay short notification tasks.

Separate pools can provide isolation when there is a real need.

<br>

### Mistake 8: Creating a New Executor for Every Request

This defeats the purpose of a managed pool and can create too many threads.

Create application-managed executor beans instead.

<br>

### Mistake 9: Forgetting That Every Application Instance Has Its Own Pool

If your Spring Boot application runs on four instances and each instance has a maximum of eight workers for an executor, the total possible workers across those instances can be up to 32.

This is still not a distributed queue or a global concurrency limit.

<br>

### Mistake 10: Treating an In-Memory Executor as Durable Storage

A process restart can lose queued in-memory work.

Use an outbox, message broker, or durable job system when work must survive application failure.

<br>

<br>

## 21. Interview Questions and Answers

<br>

### Q1. What is the difference between corePoolSize and maximumPoolSize?

`corePoolSize` is the number of core workers the executor generally maintains.

`maximumPoolSize` is the maximum number of workers the executor can create.

<br>

### Q2. How does ThreadPoolExecutor decide whether to create a thread or queue a task?

It normally creates workers until the core size is reached. After that, it tries to queue tasks. If the queue is full, it creates workers beyond the core size up to the maximum. If it still cannot accept the task, it invokes the rejection handler.

<br>

### Q3. Why is an unbounded queue risky?

Tasks can accumulate without a practical queue limit, increasing memory usage and queue waiting time. It can also prevent the executor from growing beyond its core size during normal queueing behavior.

<br>

### Q4. What is a rejection policy?

It defines what the executor does when it cannot accept a submitted task.

<br>

### Q5. What is the default rejection policy of ThreadPoolExecutor?

`AbortPolicy`, which throws `RejectedExecutionException`.

<br>

### Q6. What does CallerRunsPolicy do?

When the executor rejects a task while it is not shut down, the thread submitting the task executes it.

<br>

### Q7. What is the difference between DiscardPolicy and DiscardOldestPolicy?

`DiscardPolicy` drops the newly submitted rejected task.

`DiscardOldestPolicy` removes the oldest queued task and retries submitting the new task.

<br>

### Q8. Why might CPU-bound tasks need a different pool size from I/O-bound tasks?

CPU-bound tasks actively consume CPU time, so too many workers can cause unnecessary contention.

I/O-bound tasks spend time waiting, so more concurrent workers may improve utilization, but downstream resource limits still apply.

<br>

### Q9. Why should notification tasks and report generation use separate executors in some systems?

Separate executors can isolate workloads so that long-running report tasks do not occupy all workers needed for time-sensitive notifications.

<br>

### Q10. Does a bounded queue guarantee that no tasks will be lost?

No. When the queue and worker capacity are exhausted, the rejection policy determines what happens. Some policies reject or discard tasks.

<br>

### Q11. If an application has four instances and each has a pool of eight workers, how many workers can exist in total?

Up to 32 workers for that pool across the four instances, assuming each instance reaches its configured maximum. This is local capacity multiplied across instances, not a global coordination mechanism.

<br>

<br>

## 22. Practice Questions With Answers

<br>

### Practice 1: Identify the Core and Maximum Sizes

**Question:** In this configuration, what are the core and maximum worker counts?

```java
new ThreadPoolExecutor(
        3,
        7,
        30,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(50)
);
```

**Answer:**

- Core pool size: 3.
- Maximum pool size: 7.
- Queue capacity: 50.

<br>

### Practice 2: Understand the Submission Sequence

**Question:** A pool has:

```text
corePoolSize = 2
maximumPoolSize = 4
queueCapacity = 2
```

Assume all tasks are long-running and workers remain busy. What happens to the first six submitted tasks?

**Answer:**

1. Task 1 starts on Worker 1.
2. Task 2 starts on Worker 2.
3. Task 3 enters the queue.
4. Task 4 enters the queue.
5. Task 5 starts on Worker 3 because the queue is full.
6. Task 6 starts on Worker 4 because the queue is still full.

A seventh task would be rejected if all four workers remained busy and the queue remained full.

<br>

### Practice 3: Choose a Rejection Policy

**Question:** You want the submitting code to know when the executor cannot accept a task. Which built-in policy is the direct choice?

**Answer:**

`AbortPolicy`, because it throws `RejectedExecutionException`.

The application should catch or otherwise handle that rejection at the appropriate boundary.

<br>

### Practice 4: Identify the Risk of CallerRunsPolicy

**Question:** A web request submits a five-second background task. The executor is full and uses `CallerRunsPolicy`. What can happen?

**Answer:**

The request thread may execute the task itself and remain occupied for around five seconds. This can increase response latency and reduce request-handling capacity.

<br>

### Practice 5: Identify the Risk of an Unbounded Queue

**Question:** Why can an unbounded queue be dangerous when tasks arrive faster than workers can process them?

**Answer:**

The queue can keep growing, increasing memory usage and task waiting time. A large backlog can make the application unstable.

<br>

### Practice 6: Calculate Total Worker Capacity Across Instances

**Question:** Your application runs on three instances. Each instance has an executor with `maximumPoolSize = 6`. What is the maximum number of workers across all instances for this executor?

**Answer:**

```text
3 instances × 6 workers = 18 workers
```

This is only the combined local worker capacity. It does not mean the instances share one executor.

<br>

### Practice 7: Select an Executor for CPU-Heavy Work

**Question:** A report task performs expensive calculations for several seconds. Should it share the same executor as short notification tasks?

**Answer:**

Not necessarily. A separate executor can isolate report work from notifications. The report executor should be sized based on CPU and memory measurements.

<br>

### Practice 8: Explain Why More Threads May Make Things Worse

**Question:** Your tasks all call a database, and the connection pool has only 10 connections. You increase the executor maximum size from 10 to 100. Why might performance get worse?

**Answer:**

Many more workers may compete for a limited number of database connections. Threads may wait for connections, increasing resource usage and latency without increasing database throughput.

<br>

<br>

## 23. System Design Questions

<br>

### Question 1: Your notification queue grows continuously during peak traffic. What would you investigate?

**Answer:**

Investigate:

- Incoming task rate.
- Task processing rate.
- Average and percentile task duration.
- Number of active workers.
- Queue size and queue waiting time.
- Rejection count.
- Email or SMS provider response times and rate limits.
- Database connection usage if tasks access the database.

Then decide whether to optimize task execution, adjust pool capacity, apply rate limiting, or move the work to a durable messaging system.

Do not simply increase the queue capacity without understanding why the backlog is growing.

<br>

### Question 2: How would you prevent report generation from delaying booking notifications?

**Answer:**

Use separate executors for report generation and notifications when their workload and priority requirements justify it.

Give each executor a deliberate worker and queue capacity. Monitor both independently.

For important notifications, consider a transactional outbox and durable messaging so that the work is not lost if the application restarts.

<br>

### Question 3: Your payment provider allows only a limited number of concurrent requests. How should you configure your integration executor?

**Answer:**

The integration concurrency should respect the provider's documented limits.

Consider:

- A bounded executor queue.
- A suitable maximum number of concurrent calls.
- HTTP connection pool limits.
- Request timeouts.
- Rate limiting if the provider limits requests per time period.
- Idempotency and retry behavior.
- Metrics for active requests, queue waiting time, and failures.

The executor maximum should not exceed what the downstream service and application can safely handle.

<br>

### Question 4: What should happen when all workers and the queue are full for a booking-related background task?

**Answer:**

First, decide whether the task is business-critical.

For critical work, do not silently discard it. A durable outbox or message broker is usually a better foundation than relying only on an in-memory executor.

If using an in-memory executor, handle rejection explicitly and make sure the booking transaction and task-submission design cannot leave the system in an inconsistent state.

<br>

### Question 5: How would you choose initial thread pool settings for a new Spring Boot service?

**Answer:**

1. Identify the task type: CPU-bound, I/O-bound, or mixed.
2. Estimate normal and peak task arrival rates.
3. Measure task duration.
4. Identify downstream limits such as database connections and provider quotas.
5. Choose an initial core size, maximum size, and bounded queue.
6. Choose a rejection policy based on the task's importance.
7. Load-test realistic traffic.
8. Monitor queue size, task duration, rejections, CPU, memory, and downstream latency.
9. Adjust based on measured bottlenecks.

Do not treat an example configuration as a universal production setting.

<br>

<br>

## 24. Lesson 8 Summary

The most important ideas from this lesson:

1. CPU-bound tasks and I/O-bound tasks have different resource behavior.
2. `corePoolSize` is the core worker count.
3. `maximumPoolSize` is the maximum worker count.
4. The executor normally creates core workers first, then queues tasks, then grows beyond core size if the queue is full.
5. A bounded queue limits waiting tasks.
6. An unbounded queue can accumulate a large backlog and consume memory.
7. A rejection policy controls what happens when the executor cannot accept a task.
8. `AbortPolicy` throws an exception.
9. `CallerRunsPolicy` runs the task on the submitting thread when the executor is not shut down.
10. `DiscardPolicy` drops the new task.
11. `DiscardOldestPolicy` removes the oldest queued task and retries.
12. Spring Boot applications should use managed executor beans for application-level background work.
13. Pool sizes must respect database, HTTP connection, provider, CPU, and memory limits.
14. Measure and load-test instead of guessing.
15. In-memory executors are not durable job queues.

<br>

<br>

## 25. What Comes Next?

In Lesson 9, we will study **Atomic Variables and CAS (Compare-And-Set)**.

We will learn:

- Why ordinary shared variables can cause race conditions.
- What atomic variables are.
- How `AtomicInteger` works.
- What compare-and-set means.
- How atomic operations differ from using `synchronized`.
- Where atomic variables are useful in a Spring Boot application.
- Why atomic variables do not replace database constraints or distributed coordination.

These concepts will help you understand thread-safe counters and simple shared state inside a single JVM.
