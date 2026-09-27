# Spring Boot Exception Handling — Part B
## Designing `ErrorResponse`

Part A taught us how exceptions reach a global exception handler.

In Part B, we improve the response that the client receives.

Instead of returning a plain string like:

```text
User not found
```

we want a consistent JSON structure such as:

```json
{
  "timestamp": "2026-09-27T11:15:30",
  "status": 404,
  "error": "Not Found",
  "message": "User not found",
  "path": "/api/v1/users/123"
}
```

---

# 1. Why Do We Need an `ErrorResponse`?

Imagine your frontend calls several endpoints.

One endpoint returns:

```json
{
  "message": "User not found"
}
```

Another returns:

```json
{
  "error": "Email already exists"
}
```

Another returns:

```text
Invalid request
```

Another returns:

```json
{
  "statusCode": 403,
  "reason": "Forbidden"
}
```

This makes frontend development harder.

The frontend has to understand many different response formats.

Instead, we want one standard structure.

For example:

```json
{
  "timestamp": "...",
  "status": 404,
  "error": "Not Found",
  "message": "User not found",
  "path": "/api/v1/users/123"
}
```

Now the frontend knows what to expect.

---

# 2. Think of `ErrorResponse` as a Standard Error Package

A useful mental model:

```text
Java Exception
       ↓
Global Exception Handler
       ↓
ErrorResponse
       ↓
JSON
       ↓
HTTP Response
```

The exception is an internal Java object.

`ErrorResponse` is the external API representation of the problem.

So:

```text
Exception ≠ ErrorResponse
```

The handler acts as the bridge.

---

# 3. Basic `ErrorResponse` Class

A simple DTO can look like this:

```java
public class ErrorResponse {

    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;

    public ErrorResponse(
            HttpStatus status,
            String message,
            String path) {

        this.timestamp = LocalDateTime.now();
        this.status = status.value();
        this.error = status.getReasonPhrase();
        this.message = message;
        this.path = path;
    }

    // getters and setters
}
```

Let's understand every field.

---

# 4. `timestamp`

Example:

```json
"timestamp": "2026-09-27T11:15:30"
```

This tells us when the error response was created.

Useful for:

- debugging
- logs
- tracing problems
- comparing client/server events

For a production system, `Instant` or `OffsetDateTime` can be preferable when you want an unambiguous timezone/offset representation.

For learning and a simple implementation, `LocalDateTime` is easy to understand.

---

# 5. `status`

Example:

```json
"status": 404
```

This is the HTTP status code.

Examples:

```text
400
401
403
404
409
500
```

In Java:

```java
status.value()
```

converts:

```java
HttpStatus.NOT_FOUND
```

into:

```text
404
```

---

# 6. `error`

Example:

```json
"error": "Not Found"
```

This is the standard reason phrase associated with the HTTP status.

For example:

```java
HttpStatus.NOT_FOUND.getReasonPhrase()
```

returns:

```text
Not Found
```

Another example:

```java
HttpStatus.CONFLICT.getReasonPhrase()
```

returns:

```text
Conflict
```

This gives the client a general category for the error.

---

# 7. `message`

Example:

```json
"message": "User not found"
```

This describes what happened in a human-readable way.

For example:

```text
User not found
Email already exists
Client not found
Invalid request
You do not have permission to perform this action
```

However, be careful with messages for unexpected internal errors.

Do not expose sensitive technical details.

Instead of:

```text
NullPointerException in UserService.java line 87
```

return something safe:

```text
An unexpected error occurred
```

---

# 8. `path`

Example:

```json
"path": "/api/v1/users/123"
```

This tells us which endpoint produced the error.

In Spring, we can get the request URI from:

```java
HttpServletRequest request
```

and:

```java
request.getRequestURI()
```

Example:

```java
String path = request.getRequestURI();
```

---

# 9. Complete Example

Suppose the client requests:

```http
GET /api/v1/users/123
```

But user `123` does not exist.

The service throws:

```java
throw new UserNotFoundException("User not found");
```

The global handler can create:

```java
ErrorResponse response = new ErrorResponse(
        HttpStatus.NOT_FOUND,
        ex.getMessage(),
        request.getRequestURI()
);
```

Then return:

```java
return ResponseEntity
        .status(HttpStatus.NOT_FOUND)
        .body(response);
```

The client receives:

```json
{
  "timestamp": "2026-09-27T11:15:30",
  "status": 404,
  "error": "Not Found",
  "message": "User not found",
  "path": "/api/v1/users/123"
}
```

---

# 10. Improved Global Exception Handler

Now we can combine Part A and Part B:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex,
            HttpServletRequest request) {

        ErrorResponse response = new ErrorResponse(
                HttpStatus.NOT_FOUND,
                ex.getMessage(),
                request.getRequestURI()
        );

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(response);
    }
}
```

Notice something important:

```java
HttpServletRequest request
```

Spring can provide the current request to the exception handler.

This allows us to get:

```java
request.getRequestURI()
```

---

# 11. Why Use `HttpStatus` Instead of Magic Numbers?

Avoid:

```java
.status(404)
```

Prefer:

```java
.status(HttpStatus.NOT_FOUND)
```

Why?

Because this:

```java
HttpStatus.NOT_FOUND
```

is much easier to understand than:

```java
404
```

Similarly:

```java
HttpStatus.BAD_REQUEST
HttpStatus.UNAUTHORIZED
HttpStatus.FORBIDDEN
HttpStatus.CONFLICT
HttpStatus.INTERNAL_SERVER_ERROR
```

This improves readability.

---

# 12. Handling Multiple Custom Exceptions

Suppose Woodland has:

```text
UserNotFoundException
ClientNotFoundException
RoleNotFoundException
EmailAlreadyExistsException
```

Each can have its own handler.

Example:

```java
@ExceptionHandler(ClientNotFoundException.class)
public ResponseEntity<ErrorResponse> handleClientNotFound(
        ClientNotFoundException ex,
        HttpServletRequest request) {

    ErrorResponse response = new ErrorResponse(
            HttpStatus.NOT_FOUND,
            ex.getMessage(),
            request.getRequestURI()
    );

    return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(response);
}
```

And:

```java
@ExceptionHandler(EmailAlreadyExistsException.class)
public ResponseEntity<ErrorResponse> handleEmailAlreadyExists(
        EmailAlreadyExistsException ex,
        HttpServletRequest request) {

    ErrorResponse response = new ErrorResponse(
            HttpStatus.CONFLICT,
            ex.getMessage(),
            request.getRequestURI()
    );

    return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(response);
}
```

Now the API remains consistent.

---

# 13. Example Responses

## User Not Found

```json
{
  "timestamp": "2026-09-27T11:15:30",
  "status": 404,
  "error": "Not Found",
  "message": "User not found",
  "path": "/api/v1/users/123"
}
```

## Email Already Exists

```json
{
  "timestamp": "2026-09-27T11:17:10",
  "status": 409,
  "error": "Conflict",
  "message": "Email already exists",
  "path": "/api/v1/users"
}
```

## Permission Problem

```json
{
  "timestamp": "2026-09-27T11:18:40",
  "status": 403,
  "error": "Forbidden",
  "message": "You do not have permission to perform this action",
  "path": "/api/v1/users"
}
```

---

# 14. Add an Error Code

A useful improvement is adding a stable machine-readable error code.

For example:

```json
{
  "timestamp": "2026-09-27T11:15:30",
  "status": 404,
  "error": "Not Found",
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "path": "/api/v1/users/123"
}
```

Why is this useful?

Because frontend code should not depend on English messages.

Bad approach:

```javascript
if (response.message === "User not found") {
    // ...
}
```

What happens if the backend changes the message to:

```text
The requested user could not be found
```

The frontend breaks.

Instead:

```javascript
if (response.code === "USER_NOT_FOUND") {
    // ...
}
```

The message can change while the code remains stable.

---

# 15. Example `ErrorResponse` With `code`

```java
public class ErrorResponse {

    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String code;
    private String message;
    private String path;

    public ErrorResponse(
            HttpStatus status,
            String code,
            String message,
            String path) {

        this.timestamp = LocalDateTime.now();
        this.status = status.value();
        this.error = status.getReasonPhrase();
        this.code = code;
        this.message = message;
        this.path = path;
    }

    // getters and setters
}
```

Example:

```java
ErrorResponse response = new ErrorResponse(
        HttpStatus.NOT_FOUND,
        "USER_NOT_FOUND",
        "User not found",
        request.getRequestURI()
);
```

---

# 16. Should Every Error Have a Code?

For a growing production application, stable error codes can be very useful.

Examples:

```text
USER_NOT_FOUND
EMAIL_ALREADY_EXISTS
CLIENT_NOT_FOUND
ROLE_NOT_FOUND
PERMISSION_DENIED
INVALID_REQUEST
AUTHENTICATION_REQUIRED
ACCOUNT_LOCKED
```

But you do not need to over-engineer this immediately.

The important thing is to keep the API response structure consistent.

---

# 17. Validation Errors Are Different

Suppose a user sends:

```json
{
  "firstName": "",
  "email": "hello",
  "password": "123"
}
```

Several fields may be invalid at the same time.

Returning only:

```json
{
  "message": "Validation failed"
}
```

is not very useful.

The frontend needs to know which fields failed.

A better response is:

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "errors": {
    "firstName": "First name is required",
    "email": "Invalid email format",
    "password": "Password must be at least 8 characters"
  }
}
```

This will be covered more deeply in Part C.

---

# 18. Optional `errors` Field

The `ErrorResponse` can eventually contain:

```java
private Map<String, String> errors;
```

For example:

```java
{
    "email": "Invalid email format",
    "password": "Password must be at least 8 characters"
}
```

This allows one error response class to support both:

```text
Normal application errors
```

and:

```text
Validation errors
```

---

# 19. A Production-Friendly Shape

For Woodland, a possible general structure is:

```json
{
  "timestamp": "2026-09-27T11:15:30Z",
  "status": 404,
  "error": "Not Found",
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "path": "/api/v1/users/123"
}
```

Validation:

```json
{
  "timestamp": "2026-09-27T11:16:30Z",
  "status": 400,
  "error": "Bad Request",
  "code": "VALIDATION_FAILED",
  "message": "Validation failed",
  "path": "/api/v1/users",
  "errors": {
    "email": "Invalid email format",
    "password": "Password must be at least 8 characters"
  }
}
```

This gives the frontend a predictable contract.

---

# 20. Important Distinction: Exception vs ErrorResponse

This is extremely important.

An exception is an internal Java object:

```java
UserNotFoundException
```

The client does not need to know about the Java class.

The client needs an HTTP response:

```json
{
  "status": 404,
  "code": "USER_NOT_FOUND",
  "message": "User not found"
}
```

Therefore:

```text
Internal world
────────────────────────
UserNotFoundException
        ↓
GlobalExceptionHandler

External world
────────────────────────
ErrorResponse
        ↓
JSON
        ↓
HTTP Response
```

The global handler is the bridge between these two worlds.

---

# 21. ErrorResponse and HTTP Status Are Related but Different

Do not confuse:

```text
HTTP status
```

with:

```text
ErrorResponse
```

For example:

```text
HTTP status:
404
```

while the body might be:

```json
{
  "status": 404,
  "error": "Not Found",
  "code": "USER_NOT_FOUND",
  "message": "User not found"
}
```

The HTTP status tells the HTTP client what category of response it received.

The JSON body gives the application more information.

---

# 22. Woodland Mental Model

Imagine Woodland's backend has:

```text
UserController
ClientController
RoleController
PermissionController
ProductController
```

Any of them can produce application exceptions.

Instead of each controller creating its own error format:

```text
UserController → format A
ClientController → format B
RoleController → format C
```

we want:

```text
UserController ─┐
ClientController ├──→ GlobalExceptionHandler
RoleController ──┤              ↓
ProductController┘         ErrorResponse
                                 ↓
                              JSON API
```

One central format.

That is the real value of `ErrorResponse`.

---

# 23. Do Not Put Business Logic in the Handler

The global exception handler should mainly translate exceptions into responses.

Avoid putting business logic such as:

```java
if (userIsSpecial) {
    ...
}
```

inside the handler.

Business logic belongs in:

```text
Service layer
```

The handler should mainly do:

```text
Exception
   ↓
Determine response
   ↓
Create ErrorResponse
   ↓
Return HTTP response
```

---

# 24. Do Not Log Everything Blindly

The handler can be a useful place to log unexpected errors.

For example:

```java
log.error("Unexpected exception", ex);
```

But do not automatically log every expected business exception as a severe error.

For example:

```text
UserNotFoundException
```

may be a normal API outcome.

Whereas:

```text
Unexpected database failure
```

may deserve stronger logging.

Logging strategy will depend on the application's needs.

---

# 25. Timestamps and Time Zones

A timestamp such as:

```text
2026-09-27T11:15:30
```

does not contain a timezone.

For distributed systems, an unambiguous representation is often preferable:

```text
2026-09-27T05:15:30Z
```

or an offset-aware value.

Java options include:

```java
Instant
OffsetDateTime
ZonedDateTime
```

For Woodland, using UTC-based timestamps can make logs and distributed systems easier to reason about.

You do not need to over-engineer this while learning the fundamentals.

---

# 26. Part B Mental Model

Remember this chain:

```text
Exception
    ↓
Global Exception Handler
    ↓
Choose HTTP Status
    ↓
Create ErrorResponse
    ↓
Serialize to JSON
    ↓
Send to Client
```

For example:

```text
UserNotFoundException
    ↓
404 NOT_FOUND
    ↓
ErrorResponse
    ↓
JSON
```

---

# 27. Part B Summary

### `ErrorResponse`

A DTO representing an API error.

### Common Fields

```text
timestamp
status
error
message
path
```

Optional:

```text
code
errors
```

### Why It Matters

It gives the frontend a consistent API contract.

### Important Rule

Do not expose internal technical details or stack traces to clients.

### Stable Error Codes

Prefer:

```text
USER_NOT_FOUND
```

over making frontend logic depend on:

```text
"User not found"
```

### Main Mental Model

```text
Java Exception
      ↓
Global Handler
      ↓
ErrorResponse
      ↓
JSON
      ↓
Frontend
```

---

# Quick Reference

A basic `ErrorResponse`:

```java
public class ErrorResponse {

    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;

    public ErrorResponse(
            HttpStatus status,
            String message,
            String path) {

        this.timestamp = LocalDateTime.now();
        this.status = status.value();
        this.error = status.getReasonPhrase();
        this.message = message;
        this.path = path;
    }
}
```

A handler:

```java
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<ErrorResponse> handleUserNotFound(
        UserNotFoundException ex,
        HttpServletRequest request) {

    ErrorResponse response = new ErrorResponse(
            HttpStatus.NOT_FOUND,
            ex.getMessage(),
            request.getRequestURI()
    );

    return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(response);
}
```

The key idea:

> The exception is for the backend.  
> The `ErrorResponse` is for the API client.

That distinction is fundamental to designing clean Spring Boot REST APIs.
