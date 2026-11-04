<br>

# Spring Security



This guide focuses on understanding **what Spring Security does, how a
request travels through it, and why each piece exists**. The goal is not
to memorize configuration code.

<br>
<br>

## 1. First, what problem does Spring Security solve?

Imagine your apartment-management application has these users:

-   An administrator who can manage apartments and residents.
-   An apartment owner who can see their own apartment information.
-   A resident who can see information about the household they belong
    to.

Your backend exposes URLs such as:

``` text
GET    /api/apartments
GET    /api/apartments/123
POST   /api/apartments
DELETE /api/apartments/123
```

Without security, anyone who can reach your server might be able to call
these endpoints.

You need to answer two different questions:

1.  **Authentication:** Who is making this request?
2.  **Authorization:** Is this person allowed to do what they are
    asking?

Spring Security helps you answer both questions and protects requests
before they reach your controllers.

### A simple analogy

Think of an apartment building:

-   **Authentication** is the security guard checking your identity
    card.
-   **Authorization** is checking whether your card allows you into the
    office, parking area, or a particular apartment.

Knowing who you are does not automatically mean you can enter every
area.

<br>

## 2. The most important concepts


```
  -----------------------------------------------------------------------
  Concept                             Easy explanation
  ----------------------------------- -----------------------------------

  Authentication                      Verifying who the user is

  Authorization                       Checking what the user is allowed
                                      to do

  Principal                           The identity of the authenticated
                                      user

  Authority                           A permission or role granted to the
                                      user

  SecurityContext                     Where Spring Security keeps the
                                      current request's authentication
                                      information

  SecurityContextHolder               The usual way to access the current
                                      SecurityContext

  Filter                              Code that can inspect or process an
                                      HTTP request before it reaches a
                                      controller

  SecurityFilterChain                 The ordered security filters that
                                      process a request

  AuthenticationManager               Coordinates authentication

  AuthenticationProvider              Performs a particular kind of
                                      authentication

  UserDetailsService                  Loads user information for
                                      username/password authentication

  PasswordEncoder                     Hashes passwords and verifies
                                      submitted passwords


```

You do not need to master every class immediately. Start with the
request flow.

<br>

## 3. How a request travels through Spring Security

This is the most important mental model in this guide.

When a request reaches a Spring Boot application, it normally passes
through the servlet filter system before reaching a controller.

Spring Security registers a special filter called `FilterChainProxy`. It
selects the appropriate `SecurityFilterChain`, whose filters perform
security-related work.

A simplified request flow looks like this:

``` text
Browser / Frontend
       |
       | HTTP request
       v
Servlet filter system
       |
       v
Spring Security FilterChainProxy
       |
       v
Selected SecurityFilterChain
       |
       +--> Read or check authentication
       |
       +--> Apply authorization rules
       |
       v
DispatcherServlet
       |
       v
Controller
       |
       v
Service
       |
       v
Repository / Database
```

The exact filters and their order depend on your configuration. The
diagram is deliberately simplified.

### Example: a request with a JWT

Suppose the frontend sends:

``` http
GET /api/apartments/123
Authorization: Bearer eyJhbGciOi...
```

A JWT authentication filter can:

1.  Read the `Authorization` header.
2.  Check whether it contains a Bearer token.
3.  Extract the token.
4.  Validate the token's signature and expiry.
5.  Read the user's identity and relevant claims.
6.  Load or construct the authenticated user's authorities.
7.  Put an `Authentication` object into the `SecurityContext`.
8.  Continue the filter chain.

Later, Spring Security checks whether that authenticated user is allowed
to access the requested endpoint.

If the request passes security, it can reach the controller.

**Important:** A filter does not normally call your controller directly.
It processes the request and then continues the chain with
`filterChain.doFilter(request, response)`.

<br>

## 4. Authentication: how does Spring know who the user is?

Spring Security represents the result of authentication using an
`Authentication` object.

An `Authentication` object can contain:

-   The principal --- who the user is.
-   Credentials --- information used to authenticate, such as a password
    during login.
-   Authorities --- the roles or permissions granted to the user.
-   Whether authentication has been completed.

For example, after successful authentication, the object might
conceptually contain:

``` text
Principal:
    userAuthId = 8b...
    tenantId   = 2a...
    username   = hasib

Authorities:
    ROLE_ADMIN
    apartment:read
    apartment:create

Authenticated:
    true
```

The actual principal can be a `UserDetails` implementation, a custom
user object, or another object chosen by your application.

### The Authentication interface

You will often see:

``` java
Authentication authentication =
        SecurityContextHolder.getContext().getAuthentication();
```

This means:

1.  Get the current security context.
2.  Get the authentication information stored in it.

If the request has been authenticated, this object normally describes
the authenticated user.

### Anonymous requests

An unauthenticated request may still have an `Authentication` object
representing an anonymous user, depending on the configuration.
Therefore, do not assume that `authentication != null` always means a
real user has logged in.

Use Spring Security's authorization rules, or check the authenticated
state and anonymous status appropriately.

------------------------------------------------------------------------

## 5. Authorization: what is the user allowed to do?

Authentication answers:

> Who are you?

Authorization answers:

> Are you allowed to do this?

Spring Security commonly uses **GrantedAuthority** values to represent
what an authenticated user can do.

For example:

``` text
ROLE_ADMIN
ROLE_STAFF
apartment:read
apartment:create
payment:read
payment:approve
```

A role is a broad category. A permission is usually a more specific
capability.

For example:

-   `ROLE_ADMIN` may be a role.
-   `apartment:create` may be a permission.

### URL-level authorization

You can protect URL patterns in your `SecurityFilterChain`
configuration.

``` java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/v1/auth/**").permitAll()
    .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.GET, "/api/v1/apartments/**")
        .hasAuthority("apartment:read")
    .requestMatchers(HttpMethod.POST, "/api/v1/apartments/**")
        .hasAuthority("apartment:create")
    .anyRequest().authenticated()
)
```

Read this from top to bottom:

-   Authentication endpoints are public.
-   Admin URLs require the `ADMIN` role.
-   Reading apartments requires `apartment:read`.
-   Creating apartments requires `apartment:create`.
-   Everything else requires an authenticated user.

**Order matters.** More specific matchers should generally appear before
broader matchers that might also match the same request.

### `hasRole` versus `hasAuthority`

This difference is important:

``` java
hasRole("ADMIN")
```

is a convenient role check. Spring Security normally looks for the
authority:

``` text
ROLE_ADMIN
```

But:

``` java
hasAuthority("apartment:read")
```

checks for that exact authority string.

So, if you use `hasRole("ADMIN")`, your granted authority should
normally be `ROLE_ADMIN`, not just `ADMIN`.

------------------------------------------------------------------------

## 6. What is SecurityContext?

The `SecurityContext` stores the security information for the current
execution context. Its most familiar property is the current
`Authentication`.

You can access it like this:

``` java
Authentication authentication =
        SecurityContextHolder.getContext().getAuthentication();
```

A simplified picture:

``` text
SecurityContextHolder
        |
        v
SecurityContext
        |
        v
Authentication
        |
        +--> Principal
        +--> Authorities
        +--> Authenticated state
```

### Why does this matter?

Suppose a controller or service needs the ID of the currently logged-in
user.

Instead of passing the user ID from the frontend, your backend can
obtain the authenticated identity from the security context.

That is safer because a client must not be trusted to tell the backend
which user it is.

For example:

``` java
@GetMapping("/me")
public UserProfileDto getCurrentUser() {
    Authentication authentication =
            SecurityContextHolder.getContext().getAuthentication();

    // Read the authenticated principal and return the current user's profile.
    return userService.getCurrentUserProfile();
}
```

In a real application, it is usually cleaner to put this logic in a
dedicated component, such as your `AuthenticationContext`, rather than
repeating it in controllers and services.

### Thread and request considerations

Spring Security normally associates the security context with the
current thread using `SecurityContextHolder`'s configured strategy. In a
typical servlet request, the context is made available while the request
is being processed and is cleaned up afterward.

Do not assume that the context automatically follows work into arbitrary
background threads or asynchronous tasks. If you introduce asynchronous
processing, learn how Spring Security propagates security context for
that execution model.

------------------------------------------------------------------------

## 7. What is SecurityFilterChain?

This is one of the most important parts of modern Spring Security
configuration.

A `SecurityFilterChain` defines the security filters and rules that
apply to matching HTTP requests.

In a Spring Boot application, you normally create it as a bean:

``` java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    // Configure security for matching HTTP requests.
    return http.build();
}
```

The `HttpSecurity` object is a configuration builder. You use it to
configure things such as:

-   Which requests are public.
-   Which requests require authentication.
-   Which roles or authorities are required.
-   Whether CSRF protection is enabled.
-   Whether the application uses sessions or stateless authentication.
-   Which custom filters should run and where they should run.
-   How authentication and access-denied errors should be handled.

### A basic REST API example

This is an illustrative configuration for a stateless API that
authenticates requests using Bearer tokens. It assumes that the
application has a working authentication mechanism configured.

``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http
            // This example assumes a stateless API using bearer tokens
            // sent in the Authorization header.
            .csrf(csrf -> csrf.disable())

            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/login").permitAll()
                .requestMatchers("/api/v1/auth/refresh").permitAll()
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

This example does not itself validate JWTs. It only configures request
authorization and session behavior. A JWT resource-server configuration
or a correctly implemented custom JWT filter is still needed to
authenticate Bearer tokens.

### Understand each part

**`csrf(csrf -> csrf.disable())`**

CSRF protection helps prevent a browser from being tricked into sending
unwanted authenticated requests.

Disabling it can be appropriate for an API that authenticates using a
Bearer token explicitly added to the `Authorization` header and is not
relying on automatically attached browser cookies for authentication.

Do not blindly disable CSRF if your authentication relies on cookies.
Cookie-based authentication needs a deliberate CSRF strategy.

**`sessionCreationPolicy(STATELESS)`**

This tells Spring Security not to use an HTTP session to store the
security context between requests.

A typical stateless JWT API expects the client to send a token with each
request.

**`permitAll()`**

The matched endpoint does not require the caller to be authenticated.

**`authenticated()`**

The caller must be authenticated.

**`http.build()`**

Finishes the configuration and creates the `SecurityFilterChain` bean.

------------------------------------------------------------------------

## 8. What is a filter?

A servlet filter is code that runs around an HTTP request before the
request reaches the servlet/controller layer.

A filter can:

-   Inspect request headers.
-   Read cookies.
-   Check authentication tokens.
-   Add information to the request context.
-   Reject a request.
-   Continue processing by calling the next filter.

A simplified filter looks like this:

``` java
@Component
public class ExampleFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        System.out.println("Request path: " + request.getRequestURI());

        filterChain.doFilter(request, response);
    }
}
```

`OncePerRequestFilter` is a Spring helper class commonly used when you
want a filter to run once per request dispatch, subject to its dispatch
behavior.

### Why does a JWT filter exist?

A JWT is usually sent with every protected API request. Your backend
needs to inspect and validate it before authorization can make a
meaningful decision.

A JWT filter is one way to do that.

However, Spring Security also supports built-in OAuth2 Resource Server
JWT support. For many applications, that is preferable to writing a
custom JWT filter because Spring handles much of the token-processing
integration for you.

------------------------------------------------------------------------

## 9. What happens during login?

Let's follow a username/password login.

The frontend sends:

``` http
POST /api/v1/auth/login
Content-Type: application/json

{
  "username": "hasib",
  "password": "user-entered-password"
}
```

A typical flow is:

``` text
Frontend
   |
   | username + password
   v
AuthController
   |
   v
AuthService
   |
   v
AuthenticationManager
   |
   v
AuthenticationProvider
   |
   +--> UserDetailsService loads the user
   |
   +--> PasswordEncoder checks the password
   |
   v
Authentication succeeds or fails
   |
   v
AuthService creates tokens (if successful)
   |
   v
Frontend receives the response / refresh cookie
```

The exact implementation can differ, but this is a common Spring
Security username/password flow.

### AuthenticationManager

`AuthenticationManager` is the main interface responsible for
authenticating an `Authentication` request.

A common implementation is `ProviderManager`, which delegates to one or
more `AuthenticationProvider` implementations.

You can think of it as a coordinator:

> I have an authentication request. Which provider can verify it?

### AuthenticationProvider

An `AuthenticationProvider` knows how to authenticate a particular kind
of credential.

For username/password authentication, a common provider is
`DaoAuthenticationProvider`.

It uses a `UserDetailsService` to load the user and a `PasswordEncoder`
to verify the password.

### UserDetailsService

This interface loads user information by username.

A simplified example:

``` java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserAuthRepository userAuthRepository;

    public CustomUserDetailsService(UserAuthRepository userAuthRepository) {
        this.userAuthRepository = userAuthRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) {
        UserAuth user = userAuthRepository.findByUsername(username)
            .orElseThrow(() ->
                new UsernameNotFoundException("User not found")
            );

        return User.withUsername(user.getUsername())
            .password(user.getPasswordHash())
            .authorities("ROLE_" + user.getRole().getName())
            .build();
    }
}
```

This is only an example. A multi-tenant application must make sure the
lookup is scoped to the correct tenant or uses a globally unique login
identity. Looking up a username without tenant context can be ambiguous
if two tenants can have the same username.

Also, adapt the authority mapping to your actual role and permission
model. Do not grant permissions merely because a user supplied a role
name.

### PasswordEncoder

Never store plain-text passwords.

Use a password hashing encoder such as BCrypt:

``` java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

When a user registers or changes a password:

``` java
String passwordHash = passwordEncoder.encode(rawPassword);
```

When a user logs in, Spring Security uses the encoder to compare the
submitted password with the stored hash. You should not decrypt a stored
password; secure password hashes are not meant to be decrypted.

------------------------------------------------------------------------

## 10. JWT authentication and Spring Security

JWT stands for JSON Web Token.

A JWT is a signed token that can carry claims, such as:

``` json
{
  "sub": "user-auth-id",
  "tenant_id": "tenant-id",
  "role": "ADMIN",
  "exp": 1790000000
}
```

This is an illustrative payload, not a recommended complete token
design. Do not put passwords, secrets, or sensitive personal information
in a JWT. A signed JWT is generally readable by its holder; signing does
not encrypt its contents.

### Typical request flow

1.  The user logs in with credentials.
2.  The backend verifies the credentials.
3.  The backend issues an access token.
4.  The frontend sends the access token with protected requests:

``` http
Authorization: Bearer <access-token>
```

5.  The backend validates the token on each request.
6.  The backend establishes the authenticated identity for that request.
7.  Spring Security applies authorization rules.

### JWT filter --- conceptual example

The following is deliberately simplified. It shows the responsibilities
of a custom filter, not a complete production-ready implementation.

``` java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        if (header == null || !header.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);

        if (jwtService.isValid(token)) {
            // In a real implementation:
            // 1. Extract the trusted subject/claims.
            // 2. Load or construct the user's authorities safely.
            // 3. Create an authenticated Authentication object.
            // 4. Store it in the SecurityContext.
        }

        filterChain.doFilter(request, response);
    }
}
```

A production implementation must also consider malformed tokens, expired
tokens, invalid signatures, revoked or disabled users, token-version
checks if used, tenant context, and safe error handling.

### Where should the JWT filter go?

A custom JWT filter is commonly placed before
`UsernamePasswordAuthenticationFilter`:

``` java
.addFilterBefore(
    jwtAuthenticationFilter,
    UsernamePasswordAuthenticationFilter.class
)
```

This is a common placement for a custom Bearer-token filter in a
stateless API. The correct placement depends on the other filters and
authentication mechanisms in your application.

### Do you need to write your own JWT filter?

Not necessarily.

Spring Security's OAuth2 Resource Server support can validate JWT Bearer
tokens and establish the security context. If you use it, configure a
`JwtDecoder` and map claims to authorities as needed.

A custom filter may still make sense for a specialized token design, but
it means you are responsible for more security-sensitive details.

------------------------------------------------------------------------

## 11. HTTP 401 versus HTTP 403

These two responses are easy to confuse.

### 401 Unauthorized

Despite the word "Unauthorized," HTTP 401 usually means:

> You have not successfully authenticated.

Examples:

-   No access token was provided for a protected endpoint.
-   The token is expired.
-   The token is invalid.

### 403 Forbidden

This usually means:

> We know who you are, but you are not allowed to do this.

Example:

-   A logged-in resident tries to access an admin-only endpoint.

A useful simplified distinction:

``` text
Not authenticated  -> 401
Authenticated but not permitted -> 403
```

There are edge cases and application-specific behaviors, but this is the
right starting mental model.

### AuthenticationEntryPoint

An `AuthenticationEntryPoint` handles the case where authentication is
required but the request is not properly authenticated. For a REST API,
it commonly returns a 401 response.

### AccessDeniedHandler

An `AccessDeniedHandler` handles an authenticated user who does not have
sufficient permission. For a REST API, it commonly returns a 403
response.

These are different from your application's general exception handler.
Security can reject a request before it reaches a controller, so
controller-level exception handling alone is not enough.

------------------------------------------------------------------------

## 12. CORS and CSRF are not the same thing

These terms often appear beside each other in `SecurityConfig`, but they
solve different problems.

### CORS --- Cross-Origin Resource Sharing

CORS controls which browser origins are allowed to make cross-origin
requests to your backend.

For example:

``` text
Frontend: https://app.example.com
Backend:  https://api.example.com
```

These are different origins. The backend must have an appropriate CORS
policy if the browser frontend needs to call it.

CORS is primarily a browser-enforced cross-origin access mechanism. It
is not authentication and does not stop non-browser clients from calling
your API.

### CSRF --- Cross-Site Request Forgery

CSRF is an attack in which a victim's browser is tricked into sending an
unwanted request using credentials the browser automatically attaches,
such as cookies.

If your refresh token is stored in an HttpOnly cookie, that is useful
for protecting the token from direct JavaScript access, but cookies are
automatically attached in certain browser requests. You must design
cookie attributes and your CSRF strategy carefully.

Do not assume that a stateless backend automatically has no CSRF risk.
The relevant question is how the browser sends authentication
credentials.

------------------------------------------------------------------------

## 13. Method-level security

URL rules are useful, but sometimes you want to protect a service method
as well.

Enable method security:

``` java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}
```

Then:

``` java
@PreAuthorize("hasAuthority('apartment:create')")
public Apartment createApartment(CreateApartmentRequest request) {
    // ...
}
```

Or:

``` java
@PreAuthorize("hasRole('ADMIN')")
public void suspendTenant(UUID tenantId) {
    // ...
}
```

Method-level security is useful when a business operation should be
protected regardless of which controller calls it.

However, roles and permissions are not the whole authorization story.

### Object-level authorization

Suppose a resident has `apartment:read`.

That does not mean the resident should be able to read every apartment.

For example:

``` text
GET /api/apartments/123
```

The backend must verify that the current user is entitled to access
apartment 123.

This is object-level authorization. It usually belongs in your
service/business logic, often using the authenticated user's ID and
tenant context.

A permission such as `apartment:read` answers:

> Can this kind of user read apartment records?

Object-level authorization answers:

> Can this user read this particular apartment?

You need both.

Never rely only on a hidden button or a frontend check. The backend must
enforce access.

------------------------------------------------------------------------

## 14. Multi-tenancy: an important extra security boundary

In a multi-tenant application, checking a user's role is not enough.

Imagine:

``` text
Tenant A
  - Admin A
  - Apartment A1

Tenant B
  - Admin B
  - Apartment B1
```

Admin A must not be able to access Apartment B1 just by changing an ID
in the URL.

Your backend should derive the authenticated user's tenant identity from
trusted authentication data and enforce tenant scoping in the service
and data-access layers.

A safe conceptual flow:

``` text
JWT validated
     |
     v
Authenticated principal
     |
     +--> userAuthId
     +--> tenantId
     +--> authorities
     |
     v
Service method
     |
     v
Query scoped to tenant AND requested resource
```

For example, prefer a query conceptually like:

``` sql
SELECT *
FROM apartment
WHERE id = :apartmentId
  AND tenant_id = :currentTenantId;
```

rather than retrieving by apartment ID alone and trusting the frontend
to send the correct tenant.

The exact implementation depends on your entity relationships and
repository design. Tenant isolation should be enforced consistently, not
only in one controller.

Also remember:

-   Validate that the user is active and allowed to sign in.
-   Consider what happens when a tenant is suspended.
-   Do not trust a tenant ID supplied freely in a request body or URL.
-   If tenant identity is included in a JWT, validate the token and
    ensure the account/session is still valid according to your
    revocation policy.
-   Super-admin access should be an explicit, carefully controlled
    exception---not an accidental consequence of a null tenant ID.

------------------------------------------------------------------------

## 15. A practical example of a SecurityConfig for a REST API

This example brings several concepts together. It is a starting point,
not a complete copy-and-paste production configuration.

``` java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(
            HttpSecurity http,
            JwtAuthenticationFilter jwtAuthenticationFilter,
            AuthenticationEntryPoint authenticationEntryPoint,
            AccessDeniedHandler accessDeniedHandler
    ) throws Exception {

        http
            .csrf(csrf -> csrf.disable())

            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(authenticationEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            )

            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/api/v1/auth/login",
                    "/api/v1/auth/refresh"
                ).permitAll()

                .requestMatchers("/api/v1/admin/**")
                    .hasRole("ADMIN")

                .anyRequest().authenticated()
            )

            .addFilterBefore(
                jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class
            );

        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

Before using this example, make sure:

1.  Your JWT filter is correctly implemented and registered only once.
2.  Your token validation is cryptographically correct.
3.  Your authentication object contains the correct authorities.
4.  Your public endpoints are intentionally public.
5.  Your CSRF decision matches your actual cookie/token design.
6.  Your CORS configuration is explicit and appropriate for your
    frontend origins.
7.  Your services enforce tenant and object-level access.

If you choose Spring Security's OAuth2 Resource Server support for JWTs,
you would generally configure that mechanism instead of registering your
own JWT filter. Avoid configuring two competing mechanisms for the same
Bearer token flow without a clear reason.

------------------------------------------------------------------------

## 16. Common mistakes to avoid

1.  **Thinking authentication means authorization.** A logged-in user is
    not automatically allowed to access every record.
2.  **Trusting IDs from the frontend.** A user ID, tenant ID, or
    apartment ID in a request is not proof of access.
3.  **Storing plain-text passwords.** Store password hashes produced by
    a suitable `PasswordEncoder`.
4.  **Putting sensitive data in JWT claims.** Signed JWT payloads are
    not automatically encrypted.
5.  **Disabling CSRF without understanding cookies.** Cookie-based
    authentication needs a deliberate CSRF strategy.
6.  **Using `permitAll()` too broadly.** Public endpoints should be
    intentionally selected.
7.  **Forgetting matcher order.** A broad matcher can capture requests
    before a more specific rule.
8.  **Returning the wrong status code.** Authentication failures are
    generally 401; authorization failures are generally 403.
9.  **Assuming controller exception handling catches everything.**
    Security filters can reject requests before controllers run.
10. **Assuming a role check protects individual records.** You still
    need object-level and tenant-level authorization.
11. **Trusting a JWT forever.** Consider account status,
    logout/revocation requirements, refresh-token rotation, and token
    expiry.
12. **Registering the same filter twice.** Be careful when a filter is
    both a Spring bean and manually added to the security chain; ensure
    it is not also registered as a servlet-container filter
    unintentionally.

------------------------------------------------------------------------

## 17. What should you learn first?

Do not try to understand every Spring Security class at once. Learn in
this order:

1.  **HTTP request and servlet filters** --- understand what happens
    before a controller runs.
2.  **Authentication and Authorization** --- understand the difference
    clearly.
3.  **SecurityContext and Authentication** --- learn how Spring
    represents the current user.
4.  **SecurityFilterChain** --- learn how requests and security rules
    are configured.
5.  **PasswordEncoder and username/password authentication** ---
    understand login.
6.  **AuthenticationManager, AuthenticationProvider, and
    UserDetailsService** --- understand how Spring verifies credentials.
7.  **JWT authentication** --- understand how each API request
    establishes identity.
8.  **401, 403, AuthenticationEntryPoint, and AccessDeniedHandler** ---
    understand security errors.
9.  **Method security and object-level authorization** --- protect
    business operations and individual records.
10. **Tenant isolation** --- ensure one tenant cannot access another
    tenant's data.

### A small learning exercise

Imagine these three requests:

``` text
1. GET /api/v1/apartments
2. POST /api/v1/apartments
3. GET /api/v1/apartments/123
```

Ask yourself:

-   Does the request need authentication?
-   Which role or permission is required?
-   Does the current user belong to the tenant that owns the apartment?
-   Is the current user allowed to access this particular apartment?

If you can answer those questions, you are beginning to think about
security correctly.

------------------------------------------------------------------------

## 18. Quick glossary

**Filter:** A component that processes an HTTP request before it reaches
the controller.

**Filter chain:** An ordered sequence of filters through which a request
passes.

**SecurityFilterChain:** Spring Security's configured set of filters and
security behavior for matching requests.

**Authentication:** The representation of the current authentication
result.

**AuthenticationManager:** Coordinates authentication attempts.

**AuthenticationProvider:** Performs a particular authentication method.

**UserDetailsService:** Loads user information for username/password
authentication.

**UserDetails:** A Spring Security representation of a user's login and
authority information.

**GrantedAuthority:** A role or permission granted to an authenticated
user.

**SecurityContext:** Holds the current authentication information.

**SecurityContextHolder:** Provides access to the current
SecurityContext.

**PasswordEncoder:** Hashes passwords and verifies submitted passwords
against stored hashes.

**AuthenticationEntryPoint:** Starts the authentication response when
authentication is required but missing or invalid.

**AccessDeniedHandler:** Handles an authenticated user who lacks
permission.

**CSRF:** A browser-based attack involving unwanted requests sent with
automatically attached credentials.

**CORS:** A browser mechanism controlling cross-origin requests.

**JWT:** A signed token format often used to carry authentication claims
between a client and API.

------------------------------------------------------------------------

## Final advice

Spring Security can look complicated because it has many classes, but
the central idea is straightforward:

1.  A request enters the security filter chain.
2.  Spring Security establishes or checks the user's identity.
3.  Spring Security checks whether the user has permission.
4.  Your application performs additional business and data-access
    checks.
5.  Only then should the requested operation proceed.

Learn the **request flow** first. Once that makes sense,
`SecurityFilterChain`, `Authentication`, `SecurityContext`, JWT filters,
and authorization rules become much easier to understand.
