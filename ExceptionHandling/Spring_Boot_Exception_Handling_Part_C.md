# Spring Boot Exception Handling — Part C
## Validation and Validation Errors

Part A taught us **global exception handling**.

Part B taught us how to create a consistent **`ErrorResponse`**.

Part C focuses on **validation**.

Validation answers a simple question:

> "Is the data the client is sending acceptable before we process it?"

---

# 1. What Is Validation?

Suppose Woodland has an endpoint:

```http
POST /api/v1/users
```

The client sends:

```json
{
  "firstName": "",
  "lastName": "Hayat",
  "email": "hello",
  "password": "123"
}
```

There are obvious problems:

```text
firstName → empty
email     → invalid format
password  → too short
```

We should detect these problems before reaching business logic.

That is validation.

---

# 2. Why Do We Need Validation?

Without validation, invalid data can travel through the application:

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

This can cause:

- confusing errors
- unnecessary database queries
- database constraint failures
- inconsistent data
- difficult debugging

With validation:

```text
Client
  ↓
Controller
  ↓
Validation
  ↓
Invalid? → 400 response
  ↓
Valid
  ↓
Service
```

Validation acts as an early gate.

---

# 3. Validation vs Business Logic

This distinction is extremely important.

### Validation

Checks whether the input has an acceptable basic structure.

Examples:

```text
Email must have a valid format
Password must have at least 8 characters
Name cannot be blank
Age cannot be negative
Amount must be positive
```

### Business Logic

Checks whether the operation is allowed according to application rules.

Examples:

```text
Email must not already exist
User must belong to this tenant
A deactivated user cannot perform this operation
Only SUPER_ADMIN can create certain users
A payment cannot be approved twice
```

Think:

```text
Validation
    ↓
"Is this input structurally acceptable?"

Business Logic
    ↓
"Is this operation allowed?"
```

---

# 4. Jakarta Bean Validation

Spring Boot commonly uses Jakarta Bean Validation.

Typical annotations include:

```java
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
```

These annotations describe validation rules directly on DTO fields.

---

# 5. `@NotNull`

`@NotNull` means:

> The value cannot be `null`.

Example:

```java
@NotNull
private String name;
```

Valid:

```json
{
  "name": "Hasibul"
}
```

Invalid:

```json
{
  "name": null
}
```

Important:

`@NotNull` does NOT reject an empty string.

This can still pass:

```json
{
  "name": ""
}
```

---

# 6. `@NotEmpty`

`@NotEmpty` means:

> The value cannot be `null` or empty.

Example:

```java
@NotEmpty
private String name;
```

These fail:

```text
null
""
```

But whitespace can still be an issue.

For example:

```text
"   "
```

---

# 7. `@NotBlank`

`@NotBlank` is especially useful for strings.

It means the string:

- cannot be `null`
- cannot be empty
- cannot contain only whitespace

Example:

```java
@NotBlank
private String firstName;
```

These fail:

```text
null
""
"   "
```

This is why `@NotBlank` is often appropriate for names, emails, addresses, etc.

---

# 8. Quick Comparison

| Annotation | `null` | `""` | `"   "` |
|---|---:|---:|---:|
| `@NotNull` | ❌ | ✅ | ✅ |
| `@NotEmpty` | ❌ | ❌ | ✅ |
| `@NotBlank` | ❌ | ❌ | ❌ |

Mental shortcut:

```text
@NotNull
→ must exist

@NotEmpty
→ must contain something

@NotBlank
→ must contain meaningful text
```

---

# 9. `@Email`

`@Email` checks whether a string has an email-like format.

Example:

```java
@Email
private String email;
```

Valid example:

```text
hasibul@example.com
```

Invalid example:

```text
hello
```

Usually combine it with `@NotBlank`:

```java
@NotBlank(message = "Email is required")
@Email(message = "Invalid email format")
private String email;
```

Why both?

Because `@Email` is about format.

`@NotBlank` is about presence.

---

# 10. `@Size`

`@Size` checks length or size.

Example:

```java
@Size(min = 8, max = 100)
private String password;
```

This means:

```text
minimum = 8
maximum = 100
```

Example:

```text
"1234567"   → invalid
"12345678"  → valid
```

---

# 11. `@Min` and `@Max`

These are useful for numeric values.

Example:

```java
@Min(18)
private int age;
```

This means:

```text
age >= 18
```

Another example:

```java
@Max(100)
private int percentage;
```

This means:

```text
percentage <= 100
```

---

# 12. `@Positive`

`@Positive` means the value must be greater than zero.

Example:

```java
@Positive
private BigDecimal amount;
```

Valid:

```text
100
50.25
```

Invalid:

```text
0
-10
```

This is useful for things such as:

```text
payment amount
quantity
price
```

---

# 13. `@PositiveOrZero`

This allows zero but not negative numbers.

```java
@PositiveOrZero
private BigDecimal balance;
```

Valid:

```text
0
100
500.50
```

Invalid:

```text
-10
```

---

# 14. `@Pattern`

`@Pattern` allows us to define a regular-expression rule.

Example:

```java
@Pattern(
    regexp = "^[0-9]{11}$",
    message = "Phone number must contain 11 digits"
)
private String phone;
```

This is useful when you need a specific format.

Examples:

```text
01712345678 → valid
12345       → invalid
abc         → invalid
```

Be careful with regex validation. Keep the rule understandable and only as strict as your actual business requirement.

---

# 15. Complete Create User DTO

For Woodland, a DTO could look like:

```java
public class CreateUserRequest {

    @NotBlank(message = "First name is required")
    private String firstName;

    @NotBlank(message = "Last name is required")
    private String lastName;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(
        min = 8,
        max = 100,
        message = "Password must be between 8 and 100 characters"
    )
    private String password;
}
```

Now the DTO describes the basic rules for creating a user.

---

# 16. `@Valid`

Defining validation annotations is not enough.

Spring needs to know:

> "Validate this request body."

We do that with:

```java
@Valid
```

Example:

```java
@PostMapping
public UserResponse createUser(
        @Valid @RequestBody CreateUserRequest request) {

    return userService.createUser(request);
}
```

The important part is:

```java
@Valid @RequestBody
```

---

# 17. What Happens When Validation Fails?

Suppose the client sends:

```json
{
  "firstName": "",
  "lastName": "",
  "email": "hello",
  "password": "123"
}
```

Spring validates the DTO before the controller method proceeds normally.

Conceptually:

```text
HTTP Request
     ↓
@RequestBody
     ↓
CreateUserRequest
     ↓
@Valid
     ↓
Validation
     ↓
FAIL
     ↓
MethodArgumentNotValidException
     ↓
Global Exception Handler
     ↓
400 Bad Request
```

The service does not need to process the invalid request.

---

# 18. `MethodArgumentNotValidException`

For a typical:

```java
@Valid @RequestBody
```

validation failure, Spring can throw:

```java
MethodArgumentNotValidException
```

This exception contains information about the fields that failed validation.

For example:

```text
firstName → First name is required
email     → Invalid email format
password  → Password must be between 8 and 100 characters
```

Our global exception handler can extract these errors.

---

# 19. Extracting Field Errors

Spring provides:

```java
e.getBindingResult().getFieldErrors()
```

Example:

```java
for (FieldError error : e.getBindingResult().getFieldErrors()) {

    String field = error.getField();

    String message = error.getDefaultMessage();
}
```

This gives us:

```text
field
message
```

For example:

```text
email
Invalid email format
```

---

# 20. Building an Error Map

We can create:

```java
Map<String, String> errors = new HashMap<>();
```

Then:

```java
for (FieldError error : e.getBindingResult().getFieldErrors()) {

    errors.put(
        error.getField(),
        error.getDefaultMessage()
    );
}
```

Result:

```text
{
    "firstName": "First name is required",
    "email": "Invalid email format",
    "password": "Password must be between 8 and 100 characters"
}
```

---

# 21. Validation Error Response

Now our response can look like:

```json
{
  "status": 400,
  "error": "Bad Request",
  "code": "VALIDATION_FAILED",
  "message": "Validation failed",
  "errors": {
    "firstName": "First name is required",
    "email": "Invalid email format",
    "password": "Password must be between 8 and 100 characters"
  }
}
```

This is much more useful to the frontend.

The frontend can associate each message with the correct form field.

---

# 22. Example Validation Handler

A simplified handler:

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationException(
        MethodArgumentNotValidException ex,
        HttpServletRequest request) {

    Map<String, String> errors = new HashMap<>();

    for (FieldError error : ex.getBindingResult().getFieldErrors()) {

        errors.put(
                error.getField(),
                error.getDefaultMessage()
        );
    }

    ErrorResponse response = new ErrorResponse(
            HttpStatus.BAD_REQUEST,
            "VALIDATION_FAILED",
            "Validation failed",
            request.getRequestURI(),
            errors
    );

    return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(response);
}
```

The exact `ErrorResponse` constructor depends on how we design the DTO.

---

# 23. Updating `ErrorResponse`

If we want to support validation errors, we can add:

```java
private Map<String, String> errors;
```

For example:

```java
public class ErrorResponse {

    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String code;
    private String message;
    private String path;
    private Map<String, String> errors;
}
```

For normal errors:

```text
errors = null
```

For validation errors:

```text
errors = {
    "email": "Invalid email format",
    "password": "Password is too short"
}
```

---

# 24. Validation vs Database Constraints

This is another important distinction.

Suppose we have:

```java
@NotBlank
@Email
private String email;
```

This validates the basic input format.

But it does NOT guarantee that the email is unique.

Suppose the database contains:

```text
hasibul@example.com
```

The client sends:

```text
hasibul@example.com
```

The email format is valid.

But the business rule may say:

```text
Email must be unique
```

That requires a database/business check.

For example:

```java
if (userRepository.existsByEmail(request.getEmail())) {
    throw new EmailAlreadyExistsException(
        "Email already exists"
    );
}
```

So:

```text
@Email
    ↓
"Is this formatted like an email?"

existsByEmail()
    ↓
"Is this email already registered?"
```

These are different concerns.

---

# 25. Validation vs Business Rule Example

Suppose the client sends:

```json
{
  "email": "hasibul@example.com"
}
```

### Validation

```java
@Email
```

passes.

### Business rule

Database already contains:

```text
hasibul@example.com
```

Therefore:

```text
EmailAlreadyExistsException
        ↓
409 Conflict
```

The request was structurally valid but violated an application rule.

---

# 26. Validation vs Security

Validation is also different from security.

Example:

```text
Request:
POST /api/v1/users
```

The JSON is perfectly valid.

But the current user may not have permission to create another user.

Then:

```text
Validation
    ↓
PASS

Authorization
    ↓
FAIL

403 Forbidden
```

So remember:

```text
Validation
→ Is the data acceptable?

Authentication
→ Who are you?

Authorization
→ Are you allowed?

Business logic
→ Is this operation allowed by the application's rules?
```

---

# 27. Validation of Path Variables and Request Parameters

Validation is not limited to request bodies.

For example:

```http
GET /api/v1/users/123
```

You may want to validate a path variable.

Modern Spring applications can use method validation.

Example:

```java
@Validated
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @GetMapping("/{id}")
    public UserResponse getUser(
            @PathVariable
            @Positive
            Long id) {

        return userService.getUser(id);
    }
}
```

The exact exception produced depends on the validation scenario and Spring version/configuration, so the global handler should be designed around the exceptions actually used by the application.

---

# 28. Nested DTO Validation

Suppose:

```java
public class CreateClientRequest {

    @NotBlank
    private String name;

    @Valid
    private AddressRequest address;
}
```

And:

```java
public class AddressRequest {

    @NotBlank
    private String city;

    @NotBlank
    private String country;
}
```

The `@Valid` on `address` tells Bean Validation to validate the nested object too.

Conceptually:

```text
CreateClientRequest
       ↓
AddressRequest
       ↓
Validate city
Validate country
```

---

# 29. Validation Messages

You can define messages:

```java
@NotBlank(message = "First name is required")
```

instead of relying on default messages.

This makes the API response clearer.

For example:

```json
{
  "firstName": "First name is required"
}
```

Messages should be useful to the API consumer but should not expose internal implementation details.

---

# 30. Do Not Put Everything Into Validation

A common mistake is trying to make annotations handle every business rule.

For example:

```text
"Only SUPER_ADMIN can create this type of user."
```

That is not normal field validation.

It belongs to authorization/business logic.

Similarly:

```text
"Email must not already exist."
```

is usually a business/database rule.

Use validation for basic input constraints and service logic for application rules.

---

# 31. Woodland Example: User Creation

Imagine this request:

```json
{
  "firstName": "Hasibul",
  "lastName": "Hayat",
  "email": "hasibul@example.com",
  "password": "StrongPassword123"
}
```

Flow:

```text
HTTP Request
     ↓
Controller
     ↓
@Valid
     ↓
Basic validation
     ↓
PASS
     ↓
UserService
     ↓
Check email uniqueness
     ↓
Check business rules
     ↓
Create user
```

If email is missing:

```text
@Valid
   ↓
FAIL
   ↓
400
```

If email already exists:

```text
Validation
   ↓
PASS
   ↓
Service
   ↓
EmailAlreadyExistsException
   ↓
409
```

If the current user is not allowed:

```text
Spring Security
   ↓
403
```

This is a very important complete picture.

---

# 32. The Three Common Failure Layers

When creating an API endpoint, think about three different layers.

### Layer 1 — Input Validation

Example:

```text
Email is invalid
```

Usually:

```text
400 Bad Request
```

### Layer 2 — Business Rules

Example:

```text
Email already exists
```

Usually:

```text
409 Conflict
```

### Layer 3 — Security

Example:

```text
User does not have permission
```

Usually:

```text
403 Forbidden
```

So:

```text
Request
  ↓
Validation
  ↓
Security / Authorization
  ↓
Business Logic
  ↓
Database
```

The exact order can vary internally because Spring Security's filter chain runs before controller-level validation, but conceptually these are separate responsibilities.

---

# 33. Validation Error Flow

Remember this flow:

```text
Client sends invalid JSON data
            ↓
@RequestBody
            ↓
DTO
            ↓
@Valid
            ↓
Bean Validation
            ↓
Validation fails
            ↓
MethodArgumentNotValidException
            ↓
GlobalExceptionHandler
            ↓
ErrorResponse
            ↓
HTTP 400
```

---

# 34. Example From Start to Finish

### DTO

```java
public class CreateUserRequest {

    @NotBlank(message = "First name is required")
    private String firstName;

    @NotBlank(message = "Last name is required")
    private String lastName;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
}
```

### Controller

```java
@PostMapping
public UserResponse createUser(
        @Valid @RequestBody CreateUserRequest request) {

    return userService.createUser(request);
}
```

### Invalid request

```json
{
  "firstName": "",
  "lastName": "Hayat",
  "email": "hello",
  "password": "123"
}
```

### Handler

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidation(
        MethodArgumentNotValidException ex,
        HttpServletRequest request) {

    Map<String, String> errors = new HashMap<>();

    for (FieldError error : ex.getBindingResult().getFieldErrors()) {
        errors.put(
                error.getField(),
                error.getDefaultMessage()
        );
    }

    ErrorResponse response = new ErrorResponse(
            HttpStatus.BAD_REQUEST,
            "VALIDATION_FAILED",
            "Validation failed",
            request.getRequestURI(),
            errors
    );

    return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(response);
}
```

### Response

```json
{
  "status": 400,
  "error": "Bad Request",
  "code": "VALIDATION_FAILED",
  "message": "Validation failed",
  "path": "/api/v1/users",
  "errors": {
    "firstName": "First name is required",
    "email": "Invalid email format",
    "password": "Password must be at least 8 characters"
  }
}
```

---

# 35. Important Mental Model

Think of validation as a gate.

```text
                  ┌───────────────┐
Request ─────────→│  Validation   │
                  └───────┬───────┘
                          │
                 ┌────────┴────────┐
                 │                 │
               FAIL               PASS
                 │                 │
                 ↓                 ↓
               400              Service
                                   ↓
                              Business Rules
```

The validation gate prevents obviously invalid input from going deeper into the application.

---

# 36. Part C Summary

### Validation

Checks whether incoming data satisfies basic constraints.

### Common Annotations

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
```

### `@Valid`

Triggers validation for a request body.

Example:

```java
@Valid @RequestBody CreateUserRequest request
```

### Validation Failure

Often results in:

```text
MethodArgumentNotValidException
```

for `@Valid @RequestBody` validation.

### Global Handler

Extracts field errors and creates a structured response.

### Typical Response

```json
{
  "status": 400,
  "code": "VALIDATION_FAILED",
  "message": "Validation failed",
  "errors": {
    "email": "Invalid email format"
  }
}
```

### Most Important Distinction

```text
Validation
→ Is the input structurally valid?

Business Rule
→ Is this operation allowed according to application rules?

Authorization
→ Is this user allowed to perform it?
```

---

# Quick Reference

DTO:

```java
public class CreateUserRequest {

    @NotBlank(message = "First name is required")
    private String firstName;

    @NotBlank(message = "Last name is required")
    private String lastName;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100)
    private String password;
}
```

Controller:

```java
@PostMapping
public UserResponse createUser(
        @Valid @RequestBody CreateUserRequest request) {

    return userService.createUser(request);
}
```

Validation flow:

```text
Request
   ↓
@Valid
   ↓
Validation
   ↓
FAIL → MethodArgumentNotValidException
   ↓
Global Exception Handler
   ↓
400 ErrorResponse
```

The key idea:

> Validate basic input early, keep business rules in the service layer, and return validation errors in a consistent format.
