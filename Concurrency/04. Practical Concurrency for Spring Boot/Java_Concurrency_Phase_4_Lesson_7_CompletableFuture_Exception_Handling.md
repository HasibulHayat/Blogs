<br>

<br>

# Practical Concurrency for Spring Boot - Phase 4, Lesson 7 : CompletableFuture Exception Handling

<br>

<br>

## 1. What You Will Learn

In Lesson 5, you learned the fundamentals of `CompletableFuture`.

In Lesson 6, you learned how to connect operations using methods such as `thenApply()`, `thenCompose()`, `thenCombine()`, and `allOf()`.

Now we need to understand an important part of real application development:

**What happens when an asynchronous operation fails?**

In this lesson, you will learn:

1. What happens when a task inside a CompletableFuture throws an exception.
2. How an exception moves through a chain.
3. How to use `exceptionally()`.
4. How to use `handle()`.
5. How to use `whenComplete()`.
6. The difference between these three methods.
7. How to recover from failures and provide fallback values.
8. How to handle errors in combined futures.
9. How these concepts apply to your Spring Boot Turf Booking System.

We will use simple examples first, then connect them to practical application scenarios.

<br>

<br>

## 2. First, Understand What Happens When a Task Fails

Consider this code:

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> {
            int result = 10 / 0;
            return result;
        });
```

The task tries to divide `10` by `0`.

Java throws an `ArithmeticException`.

Normally, if an exception is thrown and nobody handles it, the current method may fail.

With `CompletableFuture`, the exception is captured and stored as the future's completion outcome.

That means:

- The future does not complete with a normal Integer result.
- The future completes exceptionally.
- The failure can be observed when you call `get()` or `join()`, or handled with a completion-stage method.

For example:

```java
try {
    Integer result = future.join();
} catch (CompletionException exception) {
    System.out.println("The asynchronous task failed");
}
```

`join()` throws a `CompletionException` when the future completed exceptionally.

The original exception is available as the cause:

```java
Throwable originalError = exception.getCause();
```

For `get()`, the usual wrapper is `ExecutionException`, while `get()` can also throw `InterruptedException`.

You do not need to memorize every wrapper immediately. The important point is that an exception inside the asynchronous task is not silently converted into a normal result.

<br>

<br>

## 3. What Happens to the Next Stage When an Earlier Stage Fails?

Consider this chain:

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> {
            int result = 10 / 0;
            return result;
        }).thenApply(result -> {
            return result * 2;
        });
```

The first stage fails because of division by zero.

Therefore, the `thenApply()` operation does not run with a normal result.

Why?

Because `thenApply()` needs a successful result from the previous stage. There is no Integer result to pass to its lambda.

The final future also completes exceptionally.

This is an important rule:

**If a stage in a CompletableFuture chain fails, later normal success stages are generally skipped until an error-handling stage handles the failure.**

For example, the following `thenApply()` will not run if the previous stage failed:

```java
.thenApply(result -> result * 2)
```

We can add error-handling methods to decide what should happen next.

<br>

<br>

## 4. exceptionally() — Recover From an Exception

`exceptionally()` is used when you want to handle a failure and provide a fallback result.

Its general form is:

```java
future.exceptionally(exception -> {
    // handle the error
    return fallbackValue;
});
```

The lambda receives the exception.

It must return a value of the future's result type.

<br>

<br>

### 4.1 A Simple Example

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> {
            int result = 10 / 0;
            return result;
        }).exceptionally(exception -> {
            System.out.println("Something went wrong");
            return 0;
        });
```

Step by step:

1. `supplyAsync()` starts the task.
2. The task tries to divide by zero.
3. The task fails with an exception.
4. `exceptionally()` receives the failure.
5. It prints a message.
6. It returns `0`.
7. The future can now complete normally with the fallback value `0`.

You can retrieve the result:

```java
Integer result = future.join();

System.out.println(result); // 0
```

The exception was handled by `exceptionally()`, and the chain recovered with a normal result.

<br>

<br>

### 4.2 Why Must exceptionally() Return a Value?

Suppose your future has this type:

```java
CompletableFuture<Integer>
```

That means the future represents an Integer result.

If the original operation fails, `exceptionally()` must provide an Integer fallback:

```java
.exceptionally(exception -> 0)
```

This would not be correct:

```java
.exceptionally(exception -> "Failed")
```

Because `"Failed"` is a String, not an Integer.

The fallback must match the result type expected by the chain.

<br>

<br>

### 4.3 A Turf Example

Imagine that your application loads a venue's review count asynchronously.

For a non-critical display feature, you may decide that if the review service fails, the page can still show zero reviews temporarily.

```java
CompletableFuture<Integer> reviewCountFuture =
        CompletableFuture.supplyAsync(() -> {
            return reviewService.getReviewCount(venueId);
        }, turfExecutor).exceptionally(exception -> {
            log.error("Could not load review count for venue {}", venueId, exception);
            return 0;
        });
```

What happens?

- If the review count loads successfully, the future contains the actual count.
- If the operation fails, the exception is logged and the future completes with `0`.

Important: this fallback is only appropriate if showing zero is acceptable for that particular feature. A failed review lookup does not necessarily mean the venue has zero reviews. If that difference matters, return a response that clearly indicates the data is unavailable instead of presenting a misleading zero.

<br>

<br>

### 4.4 exceptionally() Does Not Catch Every Error in Your Entire Method

It handles an exceptional completion from the stage before it in that chain.

For example:

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10 / 0)
                .exceptionally(exception -> 0)
                .thenApply(result -> result + 5);
```

The first stage fails.

`exceptionally()` recovers by returning `0`.

Then the `thenApply()` stage receives `0` and returns `5`.

The final future completes normally with `5`.

<br>

<br>

## 5. handle() — Process Both Success and Failure

`exceptionally()` is focused on failures.

But sometimes you want one method that can inspect both possible outcomes:

- The operation succeeded and produced a result.
- The operation failed and produced an exception.

Use `handle()` for this.

Its general form is:

```java
future.handle((result, exception) -> {
    // inspect result and exception
    return someValue;
});
```

The lambda receives two parameters:

- `result`: the successful result, or `null` if the stage failed.
- `exception`: the failure, or `null` if the stage succeeded.

The lambda must return a value.

<br>

<br>

### 5.1 A Simple Successful Example

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "Booking confirmed")
                .handle((result, exception) -> {
                    if (exception != null) {
                        return "Booking status unavailable";
                    }

                    return result.toUpperCase();
                });
```

If the first operation succeeds:

- `result` contains `"Booking confirmed"`.
- `exception` is `null`.
- The method returns `"BOOKING CONFIRMED"`.

The final future completes with that String.

<br>

<br>

### 5.2 A Simple Failure Example

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Service unavailable");
        }).handle((result, exception) -> {
            if (exception != null) {
                return "Fallback response";
            }

            return result;
        });
```

Here:

- The original task fails.
- `result` is `null`.
- `exception` contains the failure.
- The handler returns `"Fallback response"`.

The final future completes normally with the fallback String.

<br>

<br>

### 5.3 When Should You Use handle()?

Use `handle()` when the next step needs to make a decision based on whether the previous operation succeeded or failed.

Examples:

- Convert either success or failure into a response DTO.
- Return a fallback result when a non-critical lookup fails.
- Add a status field to a result.
- Transform a successful result while also providing a failure result.

Be careful: returning a fallback from `handle()` means the future completes normally. Make sure the caller can distinguish a real result from a fallback if that distinction matters.

<br>

<br>

## 6. whenComplete() — Observe the Result Without Replacing It

Sometimes you want to log or observe the result, but you do not want to change it.

Use `whenComplete()` for this.

Its general form is:

```java
future.whenComplete((result, exception) -> {
    // observe success or failure
});
```

It receives:

- The successful result, or `null` if the stage failed.
- The exception, or `null` if the stage succeeded.

Unlike `handle()`, the callback does not return a replacement result.

<br>

<br>

### 6.1 A Successful Example

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "Booking confirmed")
                .whenComplete((result, exception) -> {
                    System.out.println("Task finished");
                    System.out.println(result);
                });
```

If the task succeeds:

- `result` contains `"Booking confirmed"`.
- `exception` is `null`.
- The callback prints the result.
- The original successful result is preserved by the returned stage.

<br>

<br>

### 6.2 A Failure Example

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Payment provider unavailable");
        }).whenComplete((result, exception) -> {
            if (exception != null) {
                System.out.println("Operation failed");
            }
        });
```

The callback observes the failure.

Important:

**whenComplete() does not recover from the original failure.**

If the original future failed, the future returned by `whenComplete()` also remains exceptional, unless the callback itself changes the outcome in a way that takes precedence under the completion rules.

In normal use, think of `whenComplete()` as an observation or side-effect callback, not as a recovery method.

<br>

<br>

### 6.3 A Turf Example: Logging Completion

```java
CompletableFuture<ReviewSummary> reviewFuture =
        reviewService.getSummaryAsync(venueId)
                .whenComplete((summary, exception) -> {
                    if (exception != null) {
                        log.error(
                                "Review summary request failed for venue {}",
                                venueId,
                                exception
                        );
                    } else {
                        log.info(
                                "Review summary loaded for venue {}",
                                venueId
                        );
                    }
                });
```

This logs whether the operation succeeded or failed.

It does not turn a failed review request into a successful result.

<br>

<br>

## 7. exceptionally() vs handle() vs whenComplete()

These three methods are easy to confuse.

| Method | Runs on success? | Runs on failure? | Can return a replacement result? | Main purpose |
|---|---|---|---|---|
| `exceptionally()` | No, only for exceptional completion | Yes | Yes, fallback result | Recover from failure |
| `handle()` | Yes | Yes | Yes | Transform success or failure into a result |
| `whenComplete()` | Yes | Yes | No | Observe outcome, such as logging |

Remember:

- `exceptionally()` handles a failure and can recover with a fallback.
- `handle()` receives either the result or the exception and creates a new result.
- `whenComplete()` observes the outcome while normally preserving it.

<br>

<br>

## 8. Where Should You Put Error-Handling Methods in a Chain?

The position of an error-handling method matters.

Consider:

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10 / 0)
                .exceptionally(exception -> 0)
                .thenApply(result -> result + 5);
```

The order is:

1. The first stage fails.
2. `exceptionally()` recovers with `0`.
3. `thenApply()` receives `0`.
4. It returns `5`.

Now consider this:

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10 / 0)
                .thenApply(result -> result + 5)
                .exceptionally(exception -> 0);
```

Here:

1. The first stage fails.
2. `thenApply()` does not run.
3. `exceptionally()` handles the failure and returns `0`.

Both examples recover, but the first example has a later success operation after recovery.

This matters when you decide whether later processing should continue using a fallback value.

<br>

<br>

## 9. What If the Error-Handling Method Itself Throws an Exception?

An error handler can also fail.

Example:

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10 / 0)
                .exceptionally(exception -> {
                    throw new IllegalStateException("Recovery also failed");
                });
```

The original operation fails.

Then the `exceptionally()` handler runs, but it throws another exception.

The resulting future completes exceptionally with the new failure.

This means:

- Error-handling code must also be reliable.
- Do not assume that adding `exceptionally()` guarantees success.
- The caller should still have a final error-handling boundary.

<br>

<br>

## 10. Handling Errors in thenCombine() and allOf()

In Lesson 6, you learned how to combine independent futures.

Now consider what happens if one of them fails.

<br>

<br>

### 10.1 Error in thenCombine()

```java
CompletableFuture<String> venueFuture =
        CompletableFuture.supplyAsync(() -> "Green Field");

CompletableFuture<Integer> facilityFuture =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Facility service failed");
        });

CompletableFuture<String> summaryFuture =
        venueFuture.thenCombine(
                facilityFuture,
                (venue, count) -> venue + " has " + count + " facilities"
        );
```

The combining function needs both successful results.

If one future completes exceptionally, the combined future also completes exceptionally. The combining function cannot build the normal summary from two successful results.

You can handle the failure after combining:

```java
CompletableFuture<String> safeSummaryFuture =
        summaryFuture.exceptionally(exception -> {
            return "Venue information is temporarily unavailable";
        });
```

Or handle each input separately if the application can safely use a fallback for one of them:

```java
CompletableFuture<Integer> safeFacilityFuture =
        facilityFuture.exceptionally(exception -> 0);
```

Then combine `venueFuture` with `safeFacilityFuture`.

Only use `0` if it is a valid fallback for the feature. Otherwise represent the data as unavailable.

<br>

<br>

### 10.2 Error in allOf()

```java
CompletableFuture<String> venueFuture =
        CompletableFuture.supplyAsync(() -> "Green Field");

CompletableFuture<Integer> facilityFuture =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Facility lookup failed");
        });

CompletableFuture<Void> allCompleted =
        CompletableFuture.allOf(venueFuture, facilityFuture);
```

`allOf()` completes exceptionally if one or more of its supplied futures complete exceptionally.

You can attach an error handler:

```java
CompletableFuture<Void> handledFuture =
        allCompleted.exceptionally(exception -> {
            log.error("At least one page-data task failed", exception);
            return null;
        });
```

Remember:

- `allOf()` does not return the individual results.
- Handling the `allOf()` future does not change the original individual futures.
- If you later call `join()` on a failed individual future, that individual future can still throw a `CompletionException`.

<br>

<br>

## 11. A Practical Turf Example: Optional Page Data

Imagine your venue page needs:

1. Main venue details, which are required.
2. Review summary, which is optional.
3. Facility information, which is also required for this page.

A possible design is:

```java
CompletableFuture<VenueDto> venueFuture =
        venueService.getVenueAsync(venueId);

CompletableFuture<ReviewSummary> reviewFuture =
        reviewService.getSummaryAsync(venueId)
                .exceptionally(exception -> {
                    log.warn(
                            "Could not load reviews for venue {}",
                            venueId,
                            exception
                    );

                    return ReviewSummary.unavailable();
                });

CompletableFuture<List<FacilityDto>> facilitiesFuture =
        facilityService.getFacilitiesAsync(venueId);
```

Then combine the results:

```java
CompletableFuture<VenuePageDto> pageFuture =
        venueFuture.thenCombine(
                reviewFuture,
                (venue, reviews) -> new VenuePageDto(
                        venue,
                        reviews,
                        List.of()
                )
        ).thenCombine(
                facilitiesFuture,
                (page, facilities) -> page.withFacilities(facilities)
        );
```

This example demonstrates a design decision:

- If review data fails, the page can still be built with an explicit unavailable review state.
- If required venue or facility data fails, the overall page future can fail.

The DTO methods shown here (`unavailable()` and `withFacilities()`) are illustrative. You would implement them according to your actual DTO design.

This is often better than returning fake data that looks like a successful result.

<br>

<br>

## 12. Important Payment and Booking Rules

Exception handling is especially important in payment and booking workflows.

But a CompletableFuture exception handler does not automatically make a payment workflow safe.

<br>

<br>

### 12.1 A Payment Timeout Does Not Prove the Payment Failed

Imagine your application sends a payment request to a payment provider.

Your application times out before receiving the response.

Possible situations include:

- The provider never received the request.
- The provider received it but did not process it.
- The provider processed the payment, but the response was delayed.
- The provider processed the payment, but your application lost the response.

Therefore, do not automatically assume that a timeout means the payment failed.

Use the payment provider's documented status-check or reconciliation process.

<br>

<br>

### 12.2 Use Idempotency for Payment Operations

A retry may send the same logical payment request more than once.

Use an idempotency key or equivalent provider-supported mechanism so that retries do not accidentally create duplicate charges.

Your own database should also track payment attempts and their states.

<br>

<br>

### 12.3 Do Not Turn Every Payment Failure Into a Success-Looking Fallback

This is dangerous:

```java
paymentFuture.exceptionally(exception -> PaymentResult.success());
```

It would convert a failure into a success-looking result.

Never do this.

A payment failure, timeout, or unknown outcome must remain distinguishable from a confirmed successful payment.

<br>

<br>

### 12.4 Keep Booking State and Payment State Clear

A booking may be pending while payment is being processed.

Your system should define explicit states and allowed transitions, for example:

- `PENDING_PAYMENT`
- `CONFIRMED`
- `PAYMENT_FAILED`
- `PAYMENT_UNKNOWN`
- `CANCELLED`

The exact states depend on your payment and reservation design.

A failed asynchronous callback must not silently mark a booking as paid or confirmed.

<br>

<br>

### 12.5 Use Durable Processing for Important Follow-Up Work

If payment confirmation, booking confirmation, or a notification must survive application restarts, do not rely only on an in-memory CompletableFuture.

Consider:

- A database transaction.
- A transactional outbox.
- A message broker or durable job mechanism.
- Idempotent consumers.
- Retry policies and dead-letter handling where appropriate.

<br>

<br>

## 13. Common Mistakes

<br>

### Mistake 1: Thinking exceptionally() Runs on Success

It does not. It runs when the preceding stage completes exceptionally.

<br>

### Mistake 2: Thinking whenComplete() Recovers From an Error

It normally observes the result or exception and preserves the original outcome. Use `exceptionally()` or `handle()` when you need recovery or a replacement result.

<br>

### Mistake 3: Returning the Wrong Type From exceptionally()

If the future represents an Integer, the fallback must be an Integer.

<br>

### Mistake 4: Returning Fake Success After a Payment Error

Never convert an unknown or failed payment result into a confirmed success.

<br>

### Mistake 5: Logging and Swallowing Every Exception

Logging is useful, but the application must still decide whether the operation can continue, should return an error, or should be retried.

<br>

### Mistake 6: Assuming One Handler Covers Every Later Failure

A handler only handles failures that reach that point in the chain. A later stage can fail after an earlier handler has already completed.

Example:

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10)
                .exceptionally(exception -> 0)
                .thenApply(value -> {
                    throw new RuntimeException("Later stage failed");
                });
```

The `exceptionally()` stage does not catch the exception thrown by the later `thenApply()` stage.

You would need another handler after that stage if you want to handle that later failure.

<br>

<br>

## 14. Interview Questions and Answers

<br>

### Q1. What happens when a task inside supplyAsync() throws an exception?

The CompletableFuture completes exceptionally. The exception can be observed through `get()` or `join()`, or handled through methods such as `exceptionally()`, `handle()`, and `whenComplete()`.

<br>

### Q2. What does exceptionally() do?

It handles an exceptional completion and can return a fallback value of the future's result type. That allows the chain to recover and complete normally.

<br>

### Q3. What does handle() do?

It receives both the result and the exception. It runs for success or failure and returns a new result.

<br>

### Q4. What does whenComplete() do?

It observes the result or exception, commonly for logging or metrics. It does not normally replace the original result or recover from the original failure.

<br>

### Q5. What is the difference between exceptionally() and handle()?

`exceptionally()` is for handling failure.

`handle()` runs for both success and failure and can transform either outcome into a new result.

<br>

### Q6. What is the difference between handle() and whenComplete()?

Both can observe success or failure.

`handle()` returns a new result and can change the outcome.

`whenComplete()` is intended for observation and normally preserves the original outcome.

<br>

### Q7. Does exceptionally() catch exceptions thrown by later stages?

No. It handles exceptional completion from stages before it in the chain. A later stage can fail independently.

<br>

### Q8. What happens if one future used by thenCombine() fails?

The combined future completes exceptionally, and the combining function cannot produce its normal result from two successful inputs.

<br>

### Q9. Does allOf() succeed if one of its futures fails?

No. The `allOf()` future completes exceptionally if one or more supplied futures complete exceptionally.

<br>

### Q10. Should a payment timeout be treated as a confirmed payment failure?

Not automatically. The payment provider may have processed the payment even if your application did not receive the response. Check the provider's status or reconciliation mechanism and use idempotency for retries.

<br>

<br>

## 15. Practice Questions With Answers

<br>

### Practice 1: Recover From Division by Zero

**Question:** Create a future that divides by zero and returns `-1` if the operation fails.

**Answer:**

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10 / 0)
                .exceptionally(exception -> -1);
```

The future completes normally with `-1` after recovery.

<br>

### Practice 2: Convert Success or Failure Into a String

**Question:** Use `handle()` to return the result as text, or `"Could not calculate"` if an exception occurred.

**Answer:**

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> 100 / 5)
                .handle((result, exception) -> {
                    if (exception != null) {
                        return "Could not calculate";
                    }

                    return "Result: " + result;
                });
```

<br>

### Practice 3: Log Failure Without Recovering

**Question:** Use `whenComplete()` to log an exception but preserve the original failure.

**Answer:**

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Something failed");
        }).whenComplete((result, exception) -> {
            if (exception != null) {
                log.error("Task failed", exception);
            }
        });
```

The returned future remains exceptional.

<br>

### Practice 4: Recover and Continue the Chain

**Question:** A task fails. Recover with `10`, then multiply the value by `2`.

**Answer:**

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10 / 0)
                .exceptionally(exception -> 10)
                .thenApply(value -> value * 2);
```

The final result is `20`.

<br>

### Practice 5: Handle a Later Stage's Failure

**Question:** Why does this code still fail?

```java
CompletableFuture<Integer> future =
        CompletableFuture.supplyAsync(() -> 10)
                .exceptionally(exception -> 0)
                .thenApply(value -> {
                    throw new RuntimeException("Failed later");
                });
```

**Answer:**

The first stage succeeds, so `exceptionally()` does not need to run.

The later `thenApply()` throws an exception after the `exceptionally()` stage. That earlier handler cannot catch a failure that occurs later.

Add a handler after the failing stage if you want to recover from it.

<br>

### Practice 6: Choose the Correct Method

**Question:** You want to log both successful and failed outcomes without intentionally changing the result. Which method should you use?

**Answer:**

Use `whenComplete()`.

<br>

### Practice 7: Choose Between exceptionally() and handle()

**Question:** You only want to provide a fallback when an operation fails. Which method is the most direct choice?

**Answer:**

Use `exceptionally()`.

If you need to inspect both the successful result and the exception, use `handle()`.

<br>

<br>

## 16. System Design Questions

<br>

### Question 1: The venue review service fails, but the venue details page should still load. How could you design this?

**Answer:**

Treat review data as optional if the product requirements allow it.

- Load venue details.
- Load review information independently.
- Handle review failure with an explicit unavailable state.
- Combine the results into the page response.

Do not report zero reviews if the actual state is unknown, unless the UI clearly distinguishes unavailable data from a real count of zero.

<br>

### Question 2: A payment request times out. Should you mark the payment as failed and let the user pay again?

**Answer:**

Not immediately.

A timeout does not prove the provider did not process the payment.

- Keep the payment attempt in an appropriate pending or unknown state.
- Check the provider's payment status using its documented mechanism.
- Use idempotency keys for retries.
- Reconcile provider records with your database.
- Only transition the booking and payment to states supported by confirmed information.

<br>

### Question 3: A future fails while other independent futures are still running. Does exceptionally() cancel the other tasks?

**Answer:**

No. Exception handling changes how the relevant future's failure is processed. It does not automatically cancel unrelated futures.

If cancellation is required, design it explicitly and understand that cancellation is generally best-effort.

<br>

### Question 4: How would you make asynchronous booking notifications reliable?

**Answer:**

Do not rely only on an in-memory CompletableFuture.

A more reliable approach is:

1. Save the booking change and an outbox event in the same database transaction.
2. A background publisher reads pending outbox events.
3. Publish the event to a durable message broker or processing system.
4. A consumer sends the notification.
5. Make the consumer idempotent so retries do not create duplicate effects.
6. Retry transient failures and monitor events that cannot be processed.

<br>

<br>

## 17. Lesson 7 Summary

| Method | Runs when | Main purpose |
|---|---|---|
| `exceptionally()` | Previous stage failed | Recover with a fallback result |
| `handle()` | Previous stage succeeded or failed | Inspect either outcome and return a new result |
| `whenComplete()` | Previous stage succeeded or failed | Observe the outcome, usually for logging or metrics |

The most important points:

1. An exception thrown inside an asynchronous task makes its future complete exceptionally.
2. Normal success stages after a failed stage do not receive a successful result.
3. `exceptionally()` can recover from a failure with a fallback value.
4. `handle()` can process success and failure and return a new result.
5. `whenComplete()` is mainly for observing completion; it does not normally recover.
6. A handler does not automatically catch failures in later stages.
7. Never turn an unknown payment outcome into a confirmed success.
8. CompletableFuture error handling does not replace database correctness, idempotency, retries, or durable background processing.

<br>

<br>

## 18. What Comes Next?

In Lesson 8, we will study **Thread Pool Sizing and Rejection Policies**.

We will learn:

- How to think about the number of worker threads.
- Why CPU-bound and I/O-bound tasks need different considerations.
- What happens when an executor's queue fills up.
- What rejection policies do.
- How to configure executor capacity for a Spring Boot application.
- How to avoid unlimited queues and uncontrolled resource usage.

These topics are important when your Turf Booking System starts receiving more simultaneous requests and background tasks.
