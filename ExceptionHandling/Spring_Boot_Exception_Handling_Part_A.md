<br>

# Spring Boot Exception Handling 

<br>
<br>

## Why Do We Need Exception Handling?

In a Spring Boot application, something can go wrong at many levels:

```text
Client
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Database
```

Examples:

- A user does not exist.
- An email is already registered.
- A request contains invalid data.
- A database operation fails.
- A user tries to access something they are not allowed to access.

We want the backend to return predictable HTTP responses instead of exposing Java exceptions or stack traces.

Example:

```json
{
  "status": 404,
  "message": "User not found"
}
```

<br>

## What Happens When an Exception Is Thrown?

Suppose the service contains:

```java
public UserResponse getUser(UUID id) {

    User user = userRepository.findById(id)
            .orElseThrow(() ->
                    new UserNotFoundException("User not found"));

    return mapToResponse(user);
}
```

If the user does not exist:

```text
Repository
    ↓
User not found
    ↓
UserNotFoundException
    ↓
Service
    ↓
Controller
    ↓
Global Exception Handler
    ↓
HTTP Response
```

The exception can travel upward until something handles it.

<br>

## Why Not Handle Every Exception in Every Controller?

You could write:

```java
@GetMapping("/{id}")
public ResponseEntity<?> getUser(@PathVariable UUID id) {

    try {
        return ResponseEntity.ok(userService.getUser(id));

    } catch (UserNotFoundException e) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body("User not found");
    }
}
```

This works, but imagine having 50 controllers.

You would repeat the same patterns everywhere.

Problems:

- duplicated code
- inconsistent responses
- difficult maintenance
- larger controllers
- harder debugging

Spring gives us a better approach: centralized exception handling.

<br>

## Global Exception Handling

Spring Boot allows us to create one central place for handling exceptions.

The main annotation is:

```java
@RestControllerAdvice
```

Think of it as:

> "When an exception comes out of a REST controller, check here to see whether I know how to handle it."

Example:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

}
```

<br>

## `@ExceptionHandler`

Inside the global handler, we use:

```java
@ExceptionHandler
```

Example:

```java
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<?> handleUserNotFound(
        UserNotFoundException ex) {

    return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ex.getMessage());
}
```

This means:

> Whenever a `UserNotFoundException` reaches the global handler, run this method.

<br>

## Complete Simple Example

<br>

### Custom Exception

```java
public class UserNotFoundException extends RuntimeException {

    public UserNotFoundException(String message) {
        super(message);
    }
}
```

### Service

```java
public UserResponse getUser(UUID id) {

    User user = userRepository.findById(id)
            .orElseThrow(() ->
                    new UserNotFoundException("User not found"));

    return mapToResponse(user);
}
```

### Controller

```java
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable UUID id) {

    return userService.getUser(id);
}
```

### Global Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<?> handleUserNotFound(
            UserNotFoundException ex) {

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ex.getMessage());
    }
}
```

Flow:

```text
GET /api/v1/users/123
        ↓
Controller
        ↓
Service
        ↓
Repository
        ↓
User does not exist
        ↓
UserNotFoundException
        ↓
GlobalExceptionHandler
        ↓
HTTP 404
```

<br>

## Why Use `ResponseEntity`?

`ResponseEntity` lets us control the HTTP response.

```java
return ResponseEntity
        .status(HttpStatus.NOT_FOUND)
        .body("User not found");
```

This controls:

- HTTP status
- response body
- headers

So the client receives:

```text
HTTP Status: 404 Not Found
Body: User not found
```

<br>

## Common HTTP Status Codes

| Status | Meaning | Typical Example |
|---|---|---|
| 200 | OK | Successful GET |
| 201 | Created | User successfully created |
| 204 | No Content | Successful delete |
| 400 | Bad Request | Invalid request data |
| 401 | Unauthorized | Authentication required/failed |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not Found | User does not exist |
| 409 | Conflict | Email already exists |
| 500 | Internal Server Error | Unexpected server problem |

Remember:

```text
400 → The request is invalid
401 → You are not authenticated
403 → You are authenticated but not allowed
404 → The resource does not exist
409 → The request conflicts with existing state
500 → Unexpected server-side problem
```

<br>

## Different Exceptions Can Have Different HTTP Statuses

Example:

```java
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<?> handleUserNotFound(
        UserNotFoundException ex) {

    return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ex.getMessage());
}
```

Another exception:

```java
@ExceptionHandler(EmailAlreadyExistsException.class)
public ResponseEntity<?> handleEmailAlreadyExists(
        EmailAlreadyExistsException ex) {

    return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(ex.getMessage());
}
```

Therefore:

```text
UserNotFoundException
        ↓
404

EmailAlreadyExistsException
        ↓
409
```

<br>

## Generic Exception Handler

We can also create a fallback handler:

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<?> handleGenericException(
        Exception ex) {

    return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body("Something went wrong");
}
```

This catches exceptions that do not have a more specific handler.

For example:

```text
UserNotFoundException
        ↓
Specific handler
        ↓
404
```

But:

```text
UnexpectedException
        ↓
No specific handler
        ↓
Generic Exception handler
        ↓
500
```

<br>

## Why Have a Generic Handler?

It gives the application a safety net.

```text
Unexpected internal problem
        ↓
Global handler
        ↓
500 Internal Server Error
```

However, do not expose internal technical details to the client.

Avoid returning sensitive exception messages or stack traces.

Prefer:

```java
.body("An unexpected error occurred");
```

Detailed information should normally be logged on the server.

<br>

## Never Expose Stack Traces to the Client

Bad:

```json
{
  "error": "NullPointerException",
  "message": "Cannot invoke ...",
  "stackTrace": "..."
}
```

A client normally does not need your Java stack trace.

It can reveal:

- internal class names
- package names
- database information
- implementation details
- server structure

Instead:

```json
{
  "status": 500,
  "message": "An unexpected error occurred"
}
```

The server can log the real exception for debugging.

<br>

## `@RestControllerAdvice` vs `@ControllerAdvice`

Spring provides:

```java
@ControllerAdvice
```

and:

```java
@RestControllerAdvice
```

For REST APIs, `@RestControllerAdvice` is usually convenient.

Conceptually:

```text
@ControllerAdvice
        +
@ResponseBody
        =
@RestControllerAdvice
```

For a REST backend like Woodland:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
}
```

is a natural choice.

<br>

## Basic Global Exception Handler Structure

A simple starting structure:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<?> handleUserNotFound(
            UserNotFoundException ex) {

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ex.getMessage());
    }

    @ExceptionHandler(EmailAlreadyExistsException.class)
    public ResponseEntity<?> handleEmailAlreadyExists(
            EmailAlreadyExistsException ex) {

        return ResponseEntity
                .status(HttpStatus.CONFLICT)
                .body(ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> handleGenericException(
            Exception ex) {

        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("An unexpected error occurred");
    }
}
```

This is only the foundation.

Later, we improve the response structure with an `ErrorResponse` DTO.

<br>

## Woodland Example

Imagine Woodland has:

```text
GET /api/v1/users/{id}
```

The request enters:

```text
Client
  ↓
Spring Security
  ↓
UserController
  ↓
UserService
  ↓
UserRepository
  ↓
PostgreSQL
```

Suppose the user does not exist.

The service does:

```java
throw new UserNotFoundException("User not found");
```

The exception travels upward.

Eventually:

```text
UserNotFoundException
        ↓
GlobalExceptionHandler
        ↓
HTTP 404
```

The frontend might receive:

```json
{
  "status": 404,
  "message": "User not found"
}
```

This is much cleaner than returning a Java exception.

<br>

## Important Mental Model

Think of the Global Exception Handler as an emergency reception desk.

Imagine a large office:

```text
Employee
   ↓
Department
   ↓
Manager
   ↓
Reception
```

When a problem reaches the appropriate central place, it can be translated into a consistent response.

In Spring:

```text
Service
   ↓
Controller
   ↓
@RestControllerAdvice
   ↓
HTTP Response
```

The global handler converts Java exceptions into API-friendly responses.

<br>

## Exception Handling Is Not Only About 500

A common beginner mistake is thinking:

> "Exception handling means catching errors and returning 500."

Not true.

Good exception handling translates different problems into meaningful HTTP responses.

```text
Invalid request
       ↓
400 Bad Request

Authentication failure
       ↓
401 Unauthorized

Permission failure
       ↓
403 Forbidden

User does not exist
       ↓
404 Not Found

Email already exists
       ↓
409 Conflict

Unexpected server problem
       ↓
500 Internal Server Error
```

<br>

## 18. Application Flow to Remember

For normal application exceptions:

```text
HTTP Request
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
Database
     ↓
Exception
     ↓
Global Exception Handler
     ↓
Error Response
     ↓
HTTP Response
```

This is one of the most important Spring Boot exception-handling mental models.

<br>

## 19. What We Will Improve Later

The examples in Part A use simple strings:

```java
.body("User not found");
```

For a production backend, we usually want a structured response such as:

```json
{
  "timestamp": "2026-09-27T11:15:30",
  "status": 404,
  "error": "Not Found",
  "message": "User not found",
  "path": "/api/v1/users/123"
}
```

We can also add:

```text
code
errors
field validation details
```

That is the next level of exception-handling design.

<br>
<br>



