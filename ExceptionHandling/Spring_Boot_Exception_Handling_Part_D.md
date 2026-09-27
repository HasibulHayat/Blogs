# Spring Boot Exception Handling — Part D
## Spring Security Exception Handling

Part D is the final part of this exception-handling series.

Parts A–C focused mainly on application-level exceptions:

```text
Part A → Global Exception Handling
Part B → ErrorResponse
Part C → Validation
```

Part D focuses on **Spring Security**.

The biggest idea to understand is:

> Security errors are handled differently because Spring Security usually runs before the request reaches your controller.

---

# 1. The Two Worlds of Exception Handling

A Spring Boot REST API has two important areas.

## Application World

This includes:

```text
Controller
Service
Repository
Database
```

Typical problems:

```text
UserNotFoundException
ClientNotFoundException
EmailAlreadyExistsException
Validation failure
Database problems
```

These are commonly handled by:

```java
@RestControllerAdvice
```

---

## Security World

This includes:

```text
Authentication
Authorization
JWT
SecurityContext
Security Filters
```

Typical problems:

```text
Missing authentication
Invalid authentication
Expired JWT
Insufficient permissions
```

These are commonly handled through Spring Security mechanisms such as:

```text
AuthenticationEntryPoint
AccessDeniedHandler
```

---

# 2. The Most Important Difference

Normal application flow:

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
```

Spring Security changes the flow:

```text
HTTP Request
     ↓
Spring Security Filter Chain
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
Database
```

Security runs before the controller.

This is extremely important.

---

# 3. Authentication vs Authorization

These two terms are often confused.

## Authentication

Authentication asks:

> "Who are you?"

Examples:

```text
Login with email/password
Validate JWT
Check session
```

If authentication fails, the user is not properly identified.

Typical response:

```text
401 Unauthorized
```

---

## Authorization

Authorization asks:

> "Are you allowed to do this?"

The user is already authenticated, but their permissions are checked.

Example:

```text
User is logged in
        ↓
Role = EMPLOYEE
        ↓
Attempts SUPER_ADMIN operation
        ↓
Access denied
```

Typical response:

```text
403 Forbidden
```

---

# 4. 401 vs 403

This is one of the most important concepts in Spring Security.

## 401 Unauthorized

Means:

> Authentication is missing or unsuccessful.

Examples:

```text
No access token
Invalid authentication
Expired authentication
Authentication cannot be established
```

Typical response:

```http
401 Unauthorized
```

---

## 403 Forbidden

Means:

> The user is authenticated, but does not have permission.

Example:

```text
User is authenticated
        ↓
Role = EMPLOYEE
        ↓
Attempts SUPER_ADMIN endpoint
        ↓
403 Forbidden
```

Mental shortcut:

```text
401 → "Who are you?"
403 → "I know who you are, but you cannot do this."
```

---

# 5. Why `@RestControllerAdvice` Is Not Enough

Suppose the request is:

```http
GET /api/v1/users
```

with no JWT.

The request may be stopped by Spring Security before it reaches:

```java
UserController
```

So the flow can be:

```text
HTTP Request
     ↓
Spring Security
     ↓
Authentication failure
     ↓
Controller never runs
```

Because the controller never runs, the normal controller exception-handling path is not necessarily involved.

That is why Spring Security has its own exception-handling mechanisms.

---

# 6. Spring Security Filter Chain

A simplified flow:

```text
HTTP Request
       ↓
┌─────────────────────────┐
│ Spring Security Filters │
└────────────┬────────────┘
             ↓
       Authentication
             ↓
       Authorization
             ↓
        Controller
             ↓
          Service
             ↓
        Repository
```

For a JWT application, one of these filters may be your:

```text
JwtAuthFilter
```

---

# 7. JWT Flow

A simplified JWT flow looks like:

```text
HTTP Request
     ↓
Authorization Header
     ↓
Bearer JWT
     ↓
JwtAuthFilter
     ↓
Read Token
     ↓
Validate Token
     ↓
Create Authentication
     ↓
SecurityContext
     ↓
Authorization
     ↓
Controller
```

For example:

```http
Authorization: Bearer eyJhbGciOi...
```

The JWT filter examines the token.

---

# 8. What Is `AuthenticationEntryPoint`?

`AuthenticationEntryPoint` handles authentication failures for protected requests.

Think of it as:

> "The user has not successfully authenticated, so I need to tell the client that authentication is required."

Conceptually:

```java
public interface AuthenticationEntryPoint {

    void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException
    );
}
```

The important part is not memorizing the interface.

Understand its responsibility:

```text
Authentication failure
        ↓
AuthenticationEntryPoint
        ↓
401 Unauthorized
```

---

# 9. Simple Authentication Entry Point

A custom implementation might look like:

```java
@Component
public class CustomAuthenticationEntryPoint
        implements AuthenticationEntryPoint {

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException)
            throws IOException {

        response.setStatus(
                HttpServletResponse.SC_UNAUTHORIZED
        );

        response.setContentType("application/json");

        response.getWriter().write(
                "{"status":401,"message":"Authentication required"}"
        );
    }
}
```

This is a basic example.

In a real application, you may want to serialize a proper `ErrorResponse` object rather than manually constructing JSON strings.

---

# 10. What Is `AccessDeniedHandler`?

`AccessDeniedHandler` handles authorization failures.

Think:

> "The user is authenticated, but they are not allowed to access this resource."

Conceptually:

```java
public interface AccessDeniedHandler {

    void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException
    );
}
```

The flow is:

```text
Authenticated user
        ↓
Authorization check
        ↓
Permission denied
        ↓
AccessDeniedHandler
        ↓
403 Forbidden
```

---

# 11. Simple Access Denied Handler

Example:

```java
@Component
public class CustomAccessDeniedHandler
        implements AccessDeniedHandler {

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException)
            throws IOException {

        response.setStatus(
                HttpServletResponse.SC_FORBIDDEN
        );

        response.setContentType("application/json");

        response.getWriter().write(
                "{"status":403,"message":"Access denied"}"
        );
    }
}
```

Again, a production implementation should generally use a structured response rather than manually building JSON.

---

# 12. Connecting Them in `SecurityConfig`

Spring Security can be configured to use both.

Conceptually:

```java
http.exceptionHandling(exception -> exception

        .authenticationEntryPoint(
                authenticationEntryPoint
        )

        .accessDeniedHandler(
                accessDeniedHandler
        )
);
```

This tells Spring Security:

```text
Authentication failure
        ↓
AuthenticationEntryPoint

Authorization failure
        ↓
AccessDeniedHandler
```

---

# 13. Woodland Example

Imagine Woodland has:

```text
SUPER_ADMIN
ADMIN
DIRECTOR
SHAREHOLDER
EMPLOYEE
```

Suppose:

```text
GET /api/v1/users
```

requires:

```text
SUPER_ADMIN or ADMIN
```

---

## Case 1 — No JWT

Request:

```http
GET /api/v1/users
```

No authentication.

Flow:

```text
Request
  ↓
Security Filter Chain
  ↓
No authentication
  ↓
401
```

The `AuthenticationEntryPoint` handles it.

---

## Case 2 — Valid JWT, Wrong Role

Request contains a valid JWT.

The user is:

```text
EMPLOYEE
```

But the endpoint requires:

```text
ADMIN
```

Flow:

```text
Request
  ↓
JWT validated
  ↓
User authenticated
  ↓
Authorization check
  ↓
Access denied
  ↓
403
```

The `AccessDeniedHandler` handles it.

---

# 14. The Difference Visually

```text
                    HTTP Request
                         │
                         ▼
              Spring Security Filter
                         │
                  ┌──────┴──────┐
                  │             │
             Not Authenticated  Authenticated
                  │             │
                  ▼             ▼
              401 Path      Authorization
                                │
                         ┌──────┴──────┐
                         │             │
                      Allowed       Denied
                         │             │
                         ▼             ▼
                    Controller        403
```

This diagram is worth remembering.

---

# 15. Where Does `JwtAuthFilter` Fit?

In a JWT-based Woodland backend, you may have:

```text
JwtAuthFilter
```

Its job can include:

```text
Read Authorization header
        ↓
Extract Bearer token
        ↓
Validate JWT
        ↓
Identify user
        ↓
Create Authentication
        ↓
Put Authentication into SecurityContext
        ↓
Continue filter chain
```

For example:

```java
SecurityContextHolder
        .getContext()
        .setAuthentication(authentication);
```

After that, Spring Security knows who the user is.

---

# 16. What If the Token Is Missing?

Suppose:

```http
GET /api/v1/users
```

has no:

```http
Authorization: Bearer ...
```

The protected endpoint requires authentication.

The request may eventually result in:

```text
AuthenticationEntryPoint
        ↓
401 Unauthorized
```

The important point is that the exact path depends on your security configuration and filter behavior.

---

# 17. What If the Token Is Invalid?

Suppose the client sends:

```http
Authorization: Bearer invalid-token
```

Your JWT filter may attempt to parse or validate it.

If token validation fails, the security layer must prevent the request from being treated as authenticated.

Depending on the implementation, the failure may need to be translated into an appropriate 401 response.

This is one reason your custom JWT filter and security exception handling need to be designed together.

---

# 18. Why `JwtAuthFilter` Matters

Suppose your filter contains:

```java
try {
    // parse JWT
    // validate JWT
    // create Authentication

} catch (Exception ex) {

    // security error handling
}
```

This is different from an exception thrown in:

```java
UserService
```

because the JWT filter runs inside the security filter chain.

Therefore, your normal:

```java
@RestControllerAdvice
```

should not be assumed to handle every JWT-filter exception.

---

# 19. `SecurityErrorHandler`

A project may also have a custom class such as:

```text
SecurityErrorHandler
```

This is not a standard Spring class name with one fixed responsibility.

The exact behavior depends on how the project implements it.

It may centralize things such as:

```text
JWT authentication errors
401 responses
403 responses
security-related JSON responses
```

In a Woodland backend, the actual code should be inspected before deciding exactly what your `SecurityErrorHandler` is responsible for.

---

# 20. Security Errors vs Application Errors

This distinction is extremely important.

## Security layer

Examples:

```text
No authentication
Invalid authentication
Insufficient permission
```

Common handlers:

```text
AuthenticationEntryPoint
AccessDeniedHandler
JWT filter/security handlers
```

Common statuses:

```text
401
403
```

---

## Application layer

Examples:

```text
User not found
Client not found
Email already exists
Validation failed
Unexpected service error
```

Common handler:

```text
@RestControllerAdvice
```

Common statuses:

```text
400
404
409
500
```

---

# 21. Complete Woodland Architecture

A useful mental model:

```text
                         HTTP REQUEST
                              │
                              ▼
                   ┌────────────────────┐
                   │ Spring Security    │
                   │ Filter Chain       │
                   └─────────┬──────────┘
                             │
                  ┌──────────┴──────────┐
                  │                     │
             Security Error          Success
                  │                     │
             ┌────┴────┐                ▼
             │         │           Controller
            401       403              │
             │         │               ▼
             ▼         ▼             Service
       Authentication  Access           │
          EntryPoint   DeniedHandler     ▼
                                  Repository
                                         │
                                         ▼
                                      Database
                                         │
                                      Exception
                                         │
                                         ▼
                               Global Exception
                                   Handler
                                         │
                                         ▼
                                  ErrorResponse
                                         │
                                         ▼
                                   HTTP Response
```

This is the complete picture tying Parts A–D together.

---

# 22. Can Security Errors Use the Same `ErrorResponse`?

Yes.

It can be useful for the frontend to receive a consistent structure.

For example, 401:

```json
{
  "timestamp": "2026-09-27T11:20:00Z",
  "status": 401,
  "error": "Unauthorized",
  "code": "AUTHENTICATION_REQUIRED",
  "message": "Authentication required",
  "path": "/api/v1/users"
}
```

And 403:

```json
{
  "timestamp": "2026-09-27T11:21:00Z",
  "status": 403,
  "error": "Forbidden",
  "code": "ACCESS_DENIED",
  "message": "You do not have permission to perform this action",
  "path": "/api/v1/users"
}
```

This gives the frontend one predictable error structure.

---

# 23. Security Error Flow

For authentication:

```text
Request
   ↓
Security Filter Chain
   ↓
Authentication failure
   ↓
AuthenticationEntryPoint
   ↓
ErrorResponse
   ↓
401
```

For authorization:

```text
Request
   ↓
Security Filter Chain
   ↓
Authenticated
   ↓
Authorization check
   ↓
AccessDeniedHandler
   ↓
ErrorResponse
   ↓
403
```

For application errors:

```text
Request
   ↓
Security
   ↓
Controller
   ↓
Service
   ↓
Exception
   ↓
@RestControllerAdvice
   ↓
ErrorResponse
   ↓
HTTP response
```

---

# 24. Why Security Is Separate

Imagine a building.

Before entering the building:

```text
Security guard
```

checks your identity.

Once inside:

```text
Office departments
```

handle business operations.

The security guard is outside the normal business workflow.

Similarly:

```text
Spring Security
```

runs around/before the controller layer.

Therefore security failures need security-specific handling.

---

# 25. Authentication and Authorization in Woodland

Woodland has roles such as:

```text
SUPER_ADMIN
ADMIN
DIRECTOR
SHAREHOLDER
EMPLOYEE
```

Suppose an endpoint requires:

```text
SUPER_ADMIN
```

An `EMPLOYEE` may have a perfectly valid JWT.

Therefore:

```text
JWT valid
   ↓
Authentication successful
   ↓
Role = EMPLOYEE
   ↓
Required role = SUPER_ADMIN
   ↓
Authorization fails
   ↓
403 Forbidden
```

This is NOT a 401.

The user is already authenticated.

---

# 26. A Common Beginner Mistake

Incorrect thinking:

> "The user cannot access this endpoint, so return 401."

Not always.

If the user is authenticated but lacks permission:

```text
403
```

Use:

```text
401 → authentication problem
403 → authorization problem
```

---

# 27. Another Common Mistake

Another mistake is assuming:

> "Every exception will be handled by `@RestControllerAdvice`."

Not necessarily.

Consider:

```text
JWT parsing
```

inside:

```text
JwtAuthFilter
```

The controller may never be reached.

Therefore, security-layer failures need security-layer handling.

---

# 28. Avoid Catching Everything in the JWT Filter

Be careful with:

```java
catch (Exception e) {
    // ignore
}
```

This is dangerous.

It can hide:

- programming bugs
- token problems
- unexpected security failures
- configuration errors

A filter should handle expected authentication-related failures deliberately and allow unexpected problems to be handled appropriately.

---

# 29. Do Not Expose JWT Internals

Avoid returning detailed information such as:

```text
JWT parser failed at character 42
```

or:

```text
Signing key validation failed because ...
```

to the client.

A safer response might be:

```json
{
  "status": 401,
  "code": "INVALID_AUTHENTICATION",
  "message": "Authentication is invalid or has expired"
}
```

The server can keep detailed information in logs where appropriate.

---

# 30. Refresh Tokens and Exception Handling

Woodland's authentication design includes access tokens and refresh tokens.

The same general distinction applies.

For example:

```text
Access token expired
        ↓
Authentication failure
        ↓
401
```

The frontend may then use the refresh-token flow to obtain a new access token, depending on the application's design.

If the refresh token itself is invalid or revoked, the refresh operation should also return an appropriate authentication error.

The exact behavior depends on the refresh-token implementation.

---

# 31. Exception Handling Layers in Woodland

A useful final architecture is:

```text
                    HTTP REQUEST
                         │
                         ▼
               ┌──────────────────┐
               │ Spring Security  │
               └────────┬─────────┘
                        │
              ┌─────────┴─────────┐
              │                   │
             401                 403
              │                   │
              ▼                   ▼
      AuthenticationEntryPoint  AccessDeniedHandler
              │                   │
              └─────────┬─────────┘
                        │
                    ErrorResponse
                        │
                        ▼
                   HTTP Response


If security succeeds:

HTTP Request
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
Exception
     ↓
@RestControllerAdvice
     ↓
ErrorResponse
     ↓
HTTP Response
```

---

# 32. The Three Main Handlers to Remember

## 1. `AuthenticationEntryPoint`

Handles:

```text
Authentication failure
```

Usually:

```text
401
```

Mental model:

> "You are not successfully authenticated."

---

## 2. `AccessDeniedHandler`

Handles:

```text
Authorization failure
```

Usually:

```text
403
```

Mental model:

> "You are authenticated, but you do not have permission."

---

## 3. `@RestControllerAdvice`

Handles normal application exceptions.

Examples:

```text
Validation failure
UserNotFoundException
EmailAlreadyExistsException
ClientNotFoundException
Unexpected application exception
```

Common statuses:

```text
400
404
409
500
```

Mental model:

> "The application encountered a problem after the request entered the normal application layer."

---

# 33. Final Mental Model for the Entire Series

This is the most important diagram from Parts A–D:

```text
                           HTTP REQUEST
                                │
                                ▼
                    ┌──────────────────────┐
                    │ Spring Security      │
                    │ Filter Chain         │
                    └──────────┬───────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
              Authentication          Authentication
                 failure                  success
                    │                     │
                    ▼                     ▼
          AuthenticationEntryPoint   Authorization
                    │                     │
                   401              ┌─────┴─────┐
                                    │           │
                                  Denied      Allowed
                                    │           │
                                    ▼           ▼
                                   403      Controller
                                               │
                                               ▼
                                            Service
                                               │
                                               ▼
                                          Repository
                                               │
                                               ▼
                                            Database
                                               │
                                            Exception
                                               │
                                               ▼
                                      Global Exception
                                          Handler
                                               │
                                               ▼
                                         ErrorResponse
                                               │
                                               ▼
                                         HTTP Response
```

---

# 34. Parts A–D Summary

## Part A — Foundation

Learned:

```text
@RestControllerAdvice
@ExceptionHandler
ResponseEntity
Global exception handling
HTTP status codes
```

Main idea:

```text
Exception → Global Handler → HTTP Response
```

---

## Part B — ErrorResponse

Learned:

```text
ErrorResponse DTO
timestamp
status
error
message
path
code
```

Main idea:

```text
Exception → ErrorResponse → JSON
```

---

## Part C — Validation

Learned:

```text
@NotNull
@NotEmpty
@NotBlank
@Email
@Size
@Min
@Max
@Positive
@PositiveOrZero
@Pattern
@Valid
MethodArgumentNotValidException
```

Main idea:

```text
Invalid input → 400
```

---

## Part D — Security

Learned:

```text
Authentication
Authorization
401
403
AuthenticationEntryPoint
AccessDeniedHandler
JwtAuthFilter
SecurityContext
Security Filter Chain
```

Main idea:

```text
Authentication failure → 401
Authorization failure  → 403
```

---

# 35. Final Exception-Handling Cheat Sheet

| Problem | Typical Handler | Status |
|---|---|---:|
| Invalid request body | Global Exception Handler | 400 |
| Validation failure | Global Exception Handler | 400 |
| Missing authentication | AuthenticationEntryPoint | 401 |
| Invalid authentication | Security handling | 401 |
| Insufficient permission | AccessDeniedHandler | 403 |
| Resource not found | Global Exception Handler | 404 |
| Duplicate/conflicting resource | Global Exception Handler | 409 |
| Unexpected application error | Global Exception Handler | 500 |

---

# 36. Final Mental Model

When debugging an exception in Woodland, first ask:

### Question 1

> Did the request reach the controller?

If **no**, investigate:

```text
Spring Security
JwtAuthFilter
AuthenticationEntryPoint
AccessDeniedHandler
SecurityConfig
```

If **yes**, investigate:

```text
Controller
Service
Repository
@RestControllerAdvice
```

### Question 2

> Is this authentication, authorization, validation, business logic, or an unexpected server problem?

That classification usually tells you where the problem belongs.

---

# 37. The Most Important Rules

Remember these:

```text
1. Authentication ≠ Authorization

2. 401 ≠ 403

3. Security filters run before controllers.

4. @RestControllerAdvice handles normal application exceptions.

5. AuthenticationEntryPoint handles authentication failures.

6. AccessDeniedHandler handles authorization failures.

7. Validation catches bad input early.

8. Business rules belong primarily in the service layer.

9. Do not expose stack traces or sensitive internal details.

10. Keep API error responses consistent.
```

---

# 38. What This Means for Your Woodland Backend

Your architecture can conceptually become:

```text
                 CLIENT
                    │
                    ▼
          Spring Security Layer
                    │
          ┌─────────┴─────────┐
          │                   │
       401/403              SUCCESS
          │                   │
          ▼                   ▼
    Security Handlers      Controllers
                              │
                              ▼
                           Services
                              │
                              ▼
                         Repositories
                              │
                              ▼
                          PostgreSQL
                              │
                         Application
                           Exception
                              │
                              ▼
                    Global Exception
                         Handler
                              │
                              ▼
                        ErrorResponse
                              │
                              ▼
                           CLIENT
```

This is the architecture you should keep in your head when you start reviewing your actual Woodland exception-handling code.

The next practical step is not to write more exception classes blindly.

Instead, inspect the actual:

```text
SecurityConfig
JwtAuthFilter
SecurityErrorHandler
GlobalExceptionHandler
ErrorResponse
```

and map each class to the concepts from Parts A–D.

That is how you move from **understanding exception handling** to **understanding how exception handling is implemented in your own Woodland backend**.
