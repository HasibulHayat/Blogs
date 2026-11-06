<br>

# Understanding `AuthenticatedUser` and `AuthenticationContext`

<br>

## 1. What is `AuthenticatedUser`?

Your code:

``` java
package com.example.backend.dto.auth;

import com.example.backend.model.AccessContext;
import java.util.UUID;

public record AuthenticatedUser(
        UUID userAuthId,
        UUID userTenantId,
        UUID sessionId,
        String role,
        Long tokenVersion,
        AccessContext accessContext,
        UUID contextTenantId
) {
}
```

In simple words:

**`AuthenticatedUser` is an object that carries important information
about the user who has logged in.**

After a user logs in successfully, your backend needs to keep track of
some details about that user while processing requests.

For example:

-   Who is the user?
-   Which tenant is the user's account associated with?
-   Which login session is being used?
-   What is the user's role?
-   Which token version does the user have?
-   What access context is active?
-   Which tenant is active in that context?

This record keeps those details together in one object.

It is **not a database entity**. It is a Java data object used to carry
authentication-related information inside your application.

<br>

## 2. What is a Java `record`?

``` java
public record AuthenticatedUser(
```

A `record` is a special kind of Java class designed mainly to hold data.

With a normal Java class, you might have to write:

-   Fields
-   A constructor
-   Getter methods
-   `equals()`
-   `hashCode()`
-   `toString()`

A Java record generates much of that code for you.

For example:

``` java
public record User(String name, int age) {
}
```

You can create an object:

``` java
User user = new User("Rahim", 25);
```

And read its values:

``` java
String name = user.name();
int age = user.age();
```

Notice that record accessors use the component names, such as `name()`
and `age()`, rather than traditional getter names like `getName()`.

Your `AuthenticatedUser` works in the same way.

<br>

## 3. Why keep everything in one object?

Imagine a service needs the current user's ID, role, session ID, and
tenant ID.

You could pass several separate values:

``` java
someMethod(userId, role, sessionId, tenantId);
```

Or you can pass one object:

``` java
someMethod(authenticatedUser);
```

This keeps related authentication information together and makes code
easier to organize.

<br>

## 4. What is `AuthenticationContext`?

Your class starts like this:

``` java
@Component
public class AuthenticationContext {
    // methods
}
```

In simple words:

**`AuthenticationContext` is a helper component that lets other parts of
your application easily get information about the currently logged-in
user.**

Spring Security keeps the current request's authentication information
in something called the `SecurityContextHolder`.

Your `AuthenticationContext` reads that information and provides
convenient methods such as:

``` java
authenticationContext.getUserAuthId();
authenticationContext.getRole();
authenticationContext.getSessionId();
```

Other classes do not need to work directly with `SecurityContextHolder`
every time.

<br>

## 5. The complete method

``` java
public AuthenticatedUser getCurrentUser() {

    Authentication authentication =
            SecurityContextHolder.getContext().getAuthentication();

    if (authentication == null ||
            !authentication.isAuthenticated() ||
            authentication instanceof AnonymousAuthenticationToken) {

        throw new IllegalStateException("User is not authenticated");
    }

    Object principal = authentication.getPrincipal();

    if (!(principal instanceof AuthenticatedUser authenticatedUser)) {
        throw new IllegalStateException("Invalid authentication principal");
    }

    return authenticatedUser;
}
```

This is the most important method in `AuthenticationContext`.

It does four things:

1.  Gets the current Spring Security authentication.
2.  Checks that the request is authenticated.
3.  Checks that the principal is an `AuthenticatedUser`.
4.  Returns that `AuthenticatedUser` object.

Let's go through it slowly.

<br>

## 6. What is `SecurityContextHolder`?

``` java
SecurityContextHolder.getContext().getAuthentication();
```

Spring Security needs somewhere to keep information about the current
request's authentication.

It uses the `SecurityContextHolder`.

Think of it as a temporary storage area for the current request's
security information.

A simplified flow:

``` text
HTTP request arrives
        |
        v
JwtAuthFilter reads the JWT
        |
        v
JwtService verifies the token
        |
        v
JwtAuthFilter identifies the user
        |
        v
JwtAuthFilter creates Authentication
        |
        v
Authentication is stored in SecurityContextHolder
        |
        v
Other classes can access the current authentication
```

Your `AuthenticationContext` reads from that storage.

### Break down the line

``` java
SecurityContextHolder.getContext().getAuthentication();
```

It has three parts:

1.  `SecurityContextHolder` --- accesses Spring Security's holder.
2.  `.getContext()` --- gets the current `SecurityContext`.
3.  `.getAuthentication()` --- gets the authentication object stored
    inside it.

The result is saved in:

``` java
Authentication authentication
```

<br>

## 7. What is `Authentication`?

`Authentication` is a Spring Security interface. It represents the
authentication information for the current request.

It can contain:

-   The principal (who the user is).
-   The user's authorities or roles.
-   Whether the authentication is considered authenticated.

In your application, the principal is expected to be an
`AuthenticatedUser`.

<br>

## 8. Why check these three conditions?

``` java
if (authentication == null ||
        !authentication.isAuthenticated() ||
        authentication instanceof AnonymousAuthenticationToken) {

    throw new IllegalStateException("User is not authenticated");
}
```

This checks whether there is a proper authenticated user.

### Condition 1: `authentication == null`

Checks whether an authentication object exists.

If it is `null`, there is no authentication object available.

### Condition 2: `!authentication.isAuthenticated()`

The exclamation mark (`!`) means NOT.

So this means:

> The authentication is not authenticated.

### Condition 3: `authentication instanceof AnonymousAuthenticationToken`

Spring Security can represent an unauthenticated visitor using an
`AnonymousAuthenticationToken`.

This can be confusing because an anonymous authentication object may
exist and may report itself as authenticated.

That is why your code also checks its actual type:

``` java
authentication instanceof AnonymousAuthenticationToken
```

It specifically excludes anonymous visitors.

### What do `instanceof` and `||` mean?

`instanceof` checks whether an object belongs to a particular type.

`||` means OR.

So your full condition means:

``` text
If authentication is null
OR authentication is not authenticated
OR authentication is anonymous
THEN throw an exception.
```

This line:

``` java
throw new IllegalStateException("User is not authenticated");
```

stops the method and reports that it cannot provide a logged-in user.

<br>

## 9. What is `getPrincipal()`?

``` java
Object principal = authentication.getPrincipal();
```

The principal represents the identity of the person or account
associated with the authentication.

In your application, you expect it to be an `AuthenticatedUser`.

For example, your JWT filter might create authentication like this:

``` java
new UsernamePasswordAuthenticationToken(
        authenticatedUser,
        null,
        authorities
);
```

The first argument, `authenticatedUser`, becomes the principal.

Later:

``` java
authentication.getPrincipal();
```

retrieves that same object.

Why is the variable type `Object`?

Because Spring Security's `getPrincipal()` can return different types
depending on the application's authentication setup. It might be a
username string, a `UserDetails` object, or your custom
`AuthenticatedUser`.

So your code checks its type before using it.

<br>

## 10. Understanding the principal type check

``` java
if (!(principal instanceof AuthenticatedUser authenticatedUser)) {
    throw new IllegalStateException("Invalid authentication principal");
}
```

This uses Java's pattern matching for `instanceof`.

It checks whether `principal` is an `AuthenticatedUser`.

If it is, Java creates a variable named `authenticatedUser` containing
that object.

The `!` means NOT.

So the condition means:

> If the principal is not an `AuthenticatedUser`, throw an exception.

For example, if the principal is simply:

``` java
"rahim"
```

then it is not an `AuthenticatedUser`, so the exception is thrown.

But if the principal is an `AuthenticatedUser`, the condition is false
and the method continues.

<br>

## 11. Return the user

``` java
return authenticatedUser;
```

At this point, the method has confirmed that:

-   An authentication object exists.
-   It is authenticated.
-   It is not anonymous.
-   Its principal is an `AuthenticatedUser`.

So it returns that object.

<br>

## 12. Why do these methods exist?

Your class contains:

``` java
public UUID getUserAuthId() {
    return getCurrentUser().userAuthId();
}

public UUID getUserTenantId() {
    return getCurrentUser().userTenantId();
}

public UUID getSessionId() {
    return getCurrentUser().sessionId();
}

public String getRole() {
    return getCurrentUser().role();
}

public Long getTokenVersion() {
    return getCurrentUser().tokenVersion();
}

public AccessContext getAccessContext() {
    return getCurrentUser().accessContext();
}

public UUID getContextTenantId() {
    return getCurrentUser().contextTenantId();
}
```

These are **convenience methods**.

Each one calls `getCurrentUser()` and retrieves one field from the
returned record.

For example:

``` java
public UUID getUserAuthId() {
    return getCurrentUser().userAuthId();
}
```

This means:

1.  `getCurrentUser()` gets the authenticated user's record.
2.  `.userAuthId()` gets the user's authentication ID from that record.
3.  The method returns that UUID.

Instead of writing:

``` java
AuthenticatedUser user = authenticationContext.getCurrentUser();
UUID userId = user.userAuthId();
```

You can simply write:

``` java
UUID userId = authenticationContext.getUserAuthId();
```

### Example: using it in a service

Imagine an expense service needs to know who created an expense:

``` java
@Service
public class ExpenseService {

    private final AuthenticationContext authenticationContext;

    public ExpenseService(AuthenticationContext authenticationContext) {
        this.authenticationContext = authenticationContext;
    }

    public void createExpense() {

        UUID createdBy = authenticationContext.getUserAuthId();

        // Use createdBy when creating the expense.
    }
}
```

The service can retrieve the current user ID from Spring Security
instead of receiving it as a parameter from the frontend.

This is useful for audit fields such as `createdBy` and `updatedBy`.

<br>

## 13. The complete method

``` java
public Optional<AuthenticatedUser> getCurrentUserOptional() {

    Authentication authentication =
            SecurityContextHolder.getContext().getAuthentication();

    if (authentication == null ||
            !authentication.isAuthenticated() ||
            authentication instanceof AnonymousAuthenticationToken) {
        return Optional.empty();
    }

    Object principal = authentication.getPrincipal();

    if (!(principal instanceof AuthenticatedUser authenticatedUser)) {
        return Optional.empty();
    }

    return Optional.of(authenticatedUser);
}
```

This method is very similar to `getCurrentUser()`.

The biggest difference is what happens when no valid user is available:

-   `getCurrentUser()` throws an exception.
-   `getCurrentUserOptional()` returns an empty `Optional`.

<br>

## 14. What is `Optional`?

`Optional` is a Java type that represents a value that may or may not
exist.

Imagine you ask:

> Is there a logged-in user?

There are two possible answers:

1.  Yes, here is the user.
2.  No, there is no user.

`Optional` helps represent both possibilities.

### `Optional.of()`

``` java
Optional.of(authenticatedUser)
```

Means:

> A value exists. Here is the user.

### `Optional.empty()`

``` java
Optional.empty()
```

Means:

> There is no value.

<br>

## 15. How does this method work?

First, it retrieves the current authentication, just like
`getCurrentUser()`.

If there is no proper authenticated user:

``` java
return Optional.empty();
```

Then it gets the principal:

``` java
Object principal = authentication.getPrincipal();
```

If the principal is not an `AuthenticatedUser`, it again returns:

``` java
return Optional.empty();
```

If everything is valid:

``` java
return Optional.of(authenticatedUser);
```

It returns the user wrapped inside an Optional.

<br>

## 16. How can you use it?

``` java
Optional<AuthenticatedUser> optionalUser =
        authenticationContext.getCurrentUserOptional();
```

You can check whether a user exists:

``` java
if (optionalUser.isPresent()) {
    AuthenticatedUser user = optionalUser.get();

    System.out.println(user.userAuthId());
} else {
    System.out.println("No authenticated user");
}
```

A more modern style is:

``` java
authenticationContext.getCurrentUserOptional()
        .ifPresent(user -> {
            System.out.println(user.userAuthId());
        });
```

This means:

> If a user exists, run this code with that user.

### When should you use each method?

  ----------------------------------------------------------------------------
  Method                       If no authenticated     Typical use
                               user exists             
  ---------------------------- ----------------------- -----------------------
  `getCurrentUser()`           Throws an exception.    Places where a
                                                       logged-in user is
                                                       required.

  `getCurrentUserOptional()`   Returns                 Places where a user may
                               `Optional.empty()`.     or may not exist.
  ----------------------------------------------------------------------------

For example, creating an expense normally requires an authenticated
user, so `getCurrentUser()` makes sense.

A feature that can work for both logged-in and anonymous visitors may
use `getCurrentUserOptional()`.

<br>

## 17. The complete relationship

You have learned that `JwtService` creates and verifies JWTs, and
`JwtAuthFilter` processes incoming requests.

Now let's connect everything:

``` text
                 USER LOGS IN
                      |
                      v
             Backend verifies password
                      |
                      v
             JwtService creates JWT
                      |
                      v
             Frontend receives JWT
                      |
                      |
               USER MAKES REQUEST
                      |
                      v
             Request reaches security filter
                      |
                      v
                 JwtAuthFilter
                      |
                      v
             Extracts JWT from header
                      |
                      v
             Calls JwtService to verify
                      |
                      v
             Reads user information
                      |
                      v
             Creates AuthenticatedUser
                      |
                      v
             Creates Authentication object
                      |
                      v
             Stores it in SecurityContextHolder
                      |
                      |
                CONTROLLER / SERVICE
                      |
                      v
             AuthenticationContext reads
             the current Authentication
                      |
                      v
             Returns AuthenticatedUser
                      |
                      v
             Service can get user ID,
             role, tenant ID, session ID, etc.
```

The key point:

-   `AuthenticatedUser` is the **data object**.
-   `AuthenticationContext` is the **helper that retrieves that data
    from Spring Security**.

## 18. How does the JWT filter put `AuthenticatedUser` into the SecurityContext?

A simplified example:

``` java
AuthenticatedUser authenticatedUser = new AuthenticatedUser(
        userAuthId,
        userTenantId,
        sessionId,
        role,
        tokenVersion,
        accessContext,
        contextTenantId
);

Authentication authentication =
        new UsernamePasswordAuthenticationToken(
                authenticatedUser,
                null,
                authorities
        );

SecurityContextHolder.getContext()
        .setAuthentication(authentication);
```

Let's focus on this part:

``` java
new UsernamePasswordAuthenticationToken(
        authenticatedUser,
        null,
        authorities
);
```

The first argument is the principal.

You are giving Spring Security your custom `AuthenticatedUser` object.

Then:

``` java
SecurityContextHolder.getContext()
        .setAuthentication(authentication);
```

You store that authentication object in the current security context.

Later, `AuthenticationContext` retrieves it:

``` java
Authentication authentication =
        SecurityContextHolder.getContext().getAuthentication();
```

And retrieves the principal:

``` java
Object principal = authentication.getPrincipal();
```

Because you stored an `AuthenticatedUser` as the principal, your helper
can retrieve that object.

That is how the two classes connect.

<br>
