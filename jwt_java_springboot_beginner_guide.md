# JWT with Java and Spring Boot: A Beginner-Friendly Guide

A practical, from-zero guide to understanding JSON Web Tokens (JWT) and
using them in a Java/Spring Boot application.

This guide assumes you are new to JWT. It explains the ideas first, then
builds toward a small Spring Boot API that protects an endpoint using a
JWT.

> **Version note:** The examples use modern Spring Boot 3 / Spring
> Security 6 style configuration and Java 17 or later. Dependency
> versions should normally be managed by your Spring Boot parent or BOM.
> Library APIs can change, so check the documentation for the versions
> in your project.

------------------------------------------------------------------------

## 1. First, what problem are we trying to solve?

Imagine you build a Spring Boot application with an endpoint:

``` http
GET /api/profile
```

You want only logged-in users to access it.

A user first sends their username and password to log in. Your server
checks those credentials. If they are correct, the user is
authenticated.

But what happens when the user later requests `/api/profile`?

The server needs a way to recognize that user without asking for the
password on every request.

There are two common approaches:

1.  **Session-based authentication:** The server stores login state and
    gives the browser a session identifier.
2.  **Token-based authentication:** The server gives the client a token.
    The client sends that token with later requests.

JWT is a popular token format used in the second approach.

**JWT stands for JSON Web Token.**

A JWT can carry information about a user or token, and a digital
signature lets the server detect whether the token has been changed.

Remember: JWT is not a complete login system. Spring Security still
needs to authenticate requests, validate tokens, and enforce
permissions.

## 2. A simple real-world analogy

Imagine a school gives you an identity card.

The card contains:

-   Your student ID
-   Your name
-   Your school
-   An expiration date

A guard checks the card before allowing you into a restricted area.

A JWT is a little like a digital identity card:

-   It carries claims (information).
-   It has a signature that can be checked.
-   It may have an expiration time.
-   Your application decides what access is allowed.

There is one important difference: a JWT is not automatically
trustworthy just because it contains a name or role. The server must
verify it.

## 3. What does a JWT look like?

A common signed JWT has three parts separated by dots:

``` text
HEADER.PAYLOAD.SIGNATURE
```

It looks like a long string:

``` text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiJ1c2VyLTEwMCIsInJvbGUiOiJVU0VSIn0
.
illustrative-signature-value
```

This is only an illustration, not a token to use.

The three parts are:

  Part        Meaning
  ----------- --------------------------------------------
  Header      Describes the token and signing algorithm
  Payload     Contains claims (information)
  Signature   Helps detect whether the token was changed

## 4. What is JSON?

JWT uses JSON data in its header and payload.

JSON is a simple format for representing information using keys and
values.

Example:

``` json
{
  "id": 100,
  "username": "rahim",
  "role": "USER"
}
```

Here:

-   `id` is a key, and `100` is its value.
-   `username` is a key, and `"rahim"` is its value.
-   `role` is a key, and `"USER"` is its value.

## 5. Understanding the JWT header

Example header:

``` json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

-   `alg` tells us which signing algorithm was used.
-   `typ` identifies the token type.

`HS256` is a signing algorithm that uses a shared secret key and
SHA-256.

The header is Base64URL-encoded to create the first part of the JWT.

## 6. Understanding the payload and claims

The payload contains **claims**. A claim is simply a statement about the
user or token.

Example:

``` json
{
  "sub": "user-100",
  "role": "USER",
  "iss": "my-spring-app",
  "aud": "my-api",
  "iat": 1793880000,
  "exp": 1793880900
}
```

Common claim names:

  Claim   Meaning
  ------- ----------------------------------
  `sub`   Subject, often the user's ID
  `iss`   Issuer, who created the token
  `aud`   Audience, the intended recipient
  `iat`   Issued-at time
  `exp`   Expiration time
  `nbf`   Not valid before this time
  `jti`   Unique token ID

The standard time claims use Unix time: seconds since 1 January 1970
UTC.

### Is the payload secret?

**No. A normal signed JWT is not encrypted.**

The header and payload are encoded, not hidden. Anyone who obtains the
token can generally decode and read them.

Never put these things in a normal JWT payload:

-   Passwords
-   Database credentials
-   API secrets
-   Private personal information
-   Signing keys

The signature helps detect changes. It does not hide the contents.

## 7. What is Base64URL encoding?

JWT uses Base64URL encoding to represent its header and payload as text.

Encoding is not encryption.

Think of it like changing the format of a sentence. It does not make the
sentence private.

You do not need to implement Base64URL yourself. A JWT library handles
it.

## 8. What is the signature?

Imagine a token contains:

``` json
{
  "sub": "user-100",
  "role": "USER"
}
```

What if somebody changes the role to `ADMIN`?

``` json
{
  "sub": "user-100",
  "role": "ADMIN"
}
```

The signature helps the server detect that change.

When a JWT is created, the signing process uses a key to sign the
encoded header and payload.

Conceptually:

``` text
signature = Sign(header + "." + payload, signingKey)
```

When the token arrives, the server verifies the signature. If the signed
content has been changed, verification should fail.

### Two common signing approaches

**HS256: shared secret**

-   The same secret is used to sign and verify.
-   The secret must be protected.
-   Any service that knows the secret can create valid tokens.

**RS256: public/private key pair**

-   A private key signs the token.
-   The matching public key verifies it.
-   A verification service can have the public key without having the
    private signing key.

Use a trusted library. Do not write your own cryptographic signing code.

## 9. How JWT authentication works

Let's follow a user named Rahim.

1.  Rahim sends his username and password to the Spring Boot backend.
2.  Spring Security checks the credentials.
3.  If they are correct, the application creates a signed JWT.
4.  The backend returns the token to the client.
5.  The client stores the token using an appropriate secure approach.
6.  When Rahim calls a protected API, the client sends the token.
7.  A Spring Security filter reads the token.
8.  The application verifies its signature and claims.
9.  Spring Security marks the request as authenticated.
10. The controller handles the request if the user has permission.

A simplified picture:

``` text
Client                         Spring Boot API
  |                                   |
  |---- username + password -------->|
  |                                   | Check credentials
  |                                   | Create JWT
  |<------------- JWT ----------------|
  |                                   |
  |---- GET /api/profile + JWT ------>|
  |                                   | Read JWT
  |                                   | Verify JWT
  |                                   | Set authentication
  |<------------- profile ------------|
```

## 10. Access token and refresh token

An **access token** is used to access protected APIs. It is usually
short-lived.

A **refresh token** is a separate credential used to obtain a new access
token after the old access token expires.

Example:

``` text
Login
  |
  v
Access token + refresh token
  |
  v
Call protected APIs with access token
  |
  v
Access token expires
  |
  v
Send refresh token to refresh endpoint
  |
  v
Receive a new access token
```

Refresh tokens need careful protection. A production application should
consider rotation, revocation, and detection of refresh-token reuse.

You do not need to implement refresh tokens in your first learning
project. First understand login, issuing an access token, and verifying
it.

## 11. JWT versus session authentication

  -----------------------------------------------------------------------
  Topic                   Session                 JWT
  ----------------------- ----------------------- -----------------------
  Login state             Stored on the server    Claims carried in token

  Client typically stores Session ID              Access token, often in
                                                  a cookie or held by a
                                                  client

  Server checks           Session store           Token signature and
                                                  claims

  Immediate logout        Usually straightforward Requires a revocation
                          by deleting the session strategy if the token
                                                  must stop working
                                                  immediately

  Common use              Traditional web         APIs and distributed
                          applications            services
  -----------------------------------------------------------------------

JWT is not automatically better than sessions. Many applications work
very well with server-side sessions.

------------------------------------------------------------------------

# Part 2: Build JWT authentication with Spring Boot

## 12. What are the main pieces?

For a basic Spring Boot JWT application, understand these parts:

  Component                   Responsibility
  --------------------------- -------------------------------------------------
  `User`                      Represents a user in your application
  `UserRepository`            Finds users in the database
  `UserDetailsService`        Loads user information for Spring Security
  `AuthenticationManager`     Checks login credentials
  `JwtService`                Creates and validates JWTs
  `JwtAuthenticationFilter`   Reads JWTs from incoming requests
  `SecurityConfig`            Configures which routes are public or protected
  `AuthController`            Provides login endpoint
  `ProfileController`         Example protected endpoint

The overall process is:

``` text
POST /auth/login
       |
       v
AuthenticationManager
       |
       v
Check username and password
       |
       v
JwtService creates token
       |
       v
Return token to client

GET /api/profile
       |
       v
JwtAuthenticationFilter
       |
       v
JwtService validates token
       |
       v
Spring Security Authentication
       |
       v
Controller
```

## 13. Create a Spring Boot project

Use Spring Initializr:

https://start.spring.io/

For this learning project, select:

-   Project: Maven
-   Language: Java
-   Spring Boot: a currently supported Spring Boot 3.x release
-   Java: 17 or later
-   Spring Web
-   Spring Security
-   Spring Data JPA (if you want database-backed users)
-   Your database driver, such as PostgreSQL or MySQL
-   Validation

You can also use an in-memory user while learning the JWT flow. A
database is not required to understand the token itself.

### Maven dependencies

Spring Initializr will create the Spring Boot dependencies. For a JWT
implementation using the JJWT library, add these dependencies to
`pom.xml`:

``` xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- JJWT: use the same version for all three modules.
         Check the library's current documentation for the latest version. -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.6</version>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

The version above is an example pinned version. Check the JJWT project
documentation before starting a new project, and keep all JJWT modules
on the same version.

If you use Spring Boot's dependency management, do not add unnecessary
explicit versions to Spring Boot-managed dependencies.

## 14. Understand the login request and response

A login request might look like this:

``` http
POST /auth/login
Content-Type: application/json
```

Body:

``` json
{
  "username": "rahim",
  "password": "myPassword"
}
```

If the credentials are correct, the server might return:

``` json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9.example.signature",
  "tokenType": "Bearer"
}
```

The token above is illustrative, not a real credential.

Let's create Java classes for this request and response.

### LoginRequest.java

``` java
public record LoginRequest(
    String username,
    String password
) {}
```

### AuthResponse.java

``` java
public record AuthResponse(
    String accessToken,
    String tokenType
) {}
```

A Java `record` is a compact way to define a class that mainly carries
data.

## 15. Create JwtService.java

This service creates and validates JWTs.

The example uses JJWT and an HS256 secret key.

``` java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;

import javax.crypto.SecretKey;
import java.util.Date;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

@Service
public class JwtService {

    private final SecretKey signingKey;
    private final long expirationMillis;

    public JwtService(
            @Value("${app.jwt.secret}") String secret,
            @Value("${app.jwt.expiration-ms}") long expirationMillis) {

        byte[] keyBytes = Decoders.BASE64.decode(secret);
        this.signingKey = Keys.hmacShaKeyFor(keyBytes);
        this.expirationMillis = expirationMillis;
    }

    public String generateToken(UserDetails userDetails) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + expirationMillis);

        return Jwts.builder()
                .subject(userDetails.getUsername())
                .issuedAt(now)
                .expiration(expiry)
                .issuer("my-spring-app")
                .audience()
                    .add("my-spring-api")
                    .and()
                .signWith(signingKey)
                .compact();
    }

    public String extractUsername(String token) {
        return extractAllClaims(token).getSubject();
    }

    public boolean isTokenValid(
            String token,
            UserDetails userDetails) {

        String username = extractUsername(token);

        return username.equals(userDetails.getUsername())
                && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractAllClaims(token)
                .getExpiration()
                .before(new Date());
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                .verifyWith(signingKey)
                .requireIssuer("my-spring-app")
                .requireAudience("my-spring-api")
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

### Understand this code slowly

**`@Service`**

Tells Spring that this class is a service bean. Spring can inject it
into other classes.

**`@Value`**

Reads values from application configuration.

**`SecretKey`**

Represents the cryptographic key used to sign and verify the JWT.

**`generateToken()`**

Creates a token with:

-   `sub`: the username
-   `iat`: when the token was created
-   `exp`: when the token expires
-   `iss`: who issued it
-   `aud`: the intended API

**`signWith(signingKey)`**

Signs the token.

**`parseSignedClaims(token)`**

Verifies and parses a signed JWT. It throws an exception when the token
is invalid, expired, or cannot be parsed.

**Important:** A real application should also validate the user is still
active and check current permissions. A valid token alone does not
guarantee that an account should still have access.

## 16. Configure the signing secret

In `application.properties`, configure:

``` properties
app.jwt.secret=${JWT_SECRET}
app.jwt.expiration-ms=900000
```

`900000` milliseconds is 15 minutes.

Do not put a real signing secret directly in your source code or commit
it to Git.

Generate a strong random secret and provide it through an environment
variable or a secrets manager.

For example, a development environment can set `JWT_SECRET` to a
Base64-encoded random key of at least 32 bytes for HS256.

The key must be Base64-encoded because the example uses:

``` java
Decoders.BASE64.decode(secret)
```

Do not use a short, memorable password as your JWT secret.

## 17. Create a UserDetailsService

Spring Security needs a way to load user information.

For learning, here is a simple in-memory implementation:

``` java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@Configuration
public class UserConfig {

    @Bean
    public UserDetailsService userDetailsService(
            PasswordEncoder passwordEncoder) {

        UserDetails rahim = User.builder()
                .username("rahim")
                .password(passwordEncoder.encode("password123"))
                .roles("USER")
                .build();

        UserDetails admin = User.builder()
                .username("admin")
                .password(passwordEncoder.encode("admin123"))
                .roles("ADMIN")
                .build();

        return new InMemoryUserDetailsManager(rahim, admin);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

This is only for learning. The usernames and passwords are hard-coded,
and the sample passwords are not suitable for a real application.

In a real application:

-   Store users in a database.
-   Store only password hashes, never plain-text passwords.
-   Use a strong password-hashing algorithm such as BCrypt.
-   Load users from your database using a `UserDetailsService`
    implementation.

## 18. Create the JWT authentication filter

The filter reads the incoming request's `Authorization` header.

A client normally sends:

``` http
Authorization: Bearer YOUR_ACCESS_TOKEN
```

The filter:

1.  Reads the header.
2.  Checks for the `Bearer` prefix.
3.  Extracts the token.
4.  Reads the username from the verified token.
5.  Loads the user.
6.  Validates the token.
7.  Creates an authenticated Spring Security context.

### JwtAuthenticationFilter.java

``` java
import java.io.IOException;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(
            JwtService jwtService,
            UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain)
            throws ServletException, IOException {

        final String authHeader =
                request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        final String token = authHeader.substring(7);

        try {
            String username = jwtService.extractUsername(token);

            if (username != null
                    && SecurityContextHolder.getContext()
                            .getAuthentication() == null) {

                UserDetails userDetails =
                        userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(token, userDetails)) {

                    UsernamePasswordAuthenticationToken authentication =
                            new UsernamePasswordAuthenticationToken(
                                    userDetails,
                                    null,
                                    userDetails.getAuthorities()
                            );

                    authentication.setDetails(
                            new WebAuthenticationDetailsSource()
                                    .buildDetails(request)
                    );

                    SecurityContextHolder.getContext()
                            .setAuthentication(authentication);
                }
            }
        } catch (io.jsonwebtoken.JwtException
                 | IllegalArgumentException ex) {
            // The token is invalid. Do not authenticate the request.
            SecurityContextHolder.clearContext();
        }

        filterChain.doFilter(request, response);
    }
}
```

### What is OncePerRequestFilter?

It is a Spring web filter base class designed to run once per request
within a single request thread.

### What is SecurityContextHolder?

Spring Security uses it to hold the authentication information for the
current request.

### What is UsernamePasswordAuthenticationToken?

Despite its name, it is also used to represent an authenticated user
after a JWT has been validated.

The three-argument constructor receives the user, credentials, and
granted authorities. In this example, the credentials are `null` because
the password is not needed after authentication.

### Important filter note

This is a teaching implementation. Production systems should have a
deliberate error-handling strategy for malformed tokens, inactive
accounts, and unavailable user stores. They should also ensure that the
filter never treats an unverified token as authenticated.

## 19. Configure Spring Security

Now tell Spring Security:

-   `/auth/**` is public.
-   `/api/**` requires authentication.
-   Use stateless request handling.
-   Run the JWT filter before the username/password authentication
    filter.

### SecurityConfig.java

``` java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    public SecurityConfig(
            JwtAuthenticationFilter jwtAuthenticationFilter) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http) throws Exception {

        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/api/**").authenticated()
                .anyRequest().permitAll()
            )
            .addFilterBefore(
                jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class
            );

        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration configuration)
            throws Exception {
        return configuration.getAuthenticationManager();
    }
}
```

### What does STATELESS mean?

It tells Spring Security not to create or use an HTTP session to store
the security context between requests.

Each protected request must provide its own valid authentication
credential.

### Why is CSRF disabled here?

This example sends a bearer token in the `Authorization` header and
assumes the browser is not automatically attaching the credential as a
cookie.

If you store authentication tokens in cookies, or use another
automatically attached credential, you must reconsider CSRF protection.
Do not copy `csrf.disable()` into every application without
understanding your authentication design.

For browser applications, CSRF protection and secure cookie
configuration are important.

## 20. Create the login controller

Now we need an endpoint that accepts a username and password,
authenticates them, and returns a JWT.

### AuthController.java

``` java
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;

    public AuthController(
            AuthenticationManager authenticationManager,
            JwtService jwtService) {
        this.authenticationManager = authenticationManager;
        this.jwtService = jwtService;
    }

    @PostMapping("/login")
    public AuthResponse login(@RequestBody LoginRequest request) {

        Authentication authentication =
                authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                        request.username(),
                        request.password()
                    )
                );

        UserDetails userDetails =
                (UserDetails) authentication.getPrincipal();

        String token = jwtService.generateToken(userDetails);

        return new AuthResponse(token, "Bearer");
    }
}
```

### Understand the important line

``` java
authenticationManager.authenticate(...)
```

This asks Spring Security to authenticate the supplied username and
password.

If the credentials are wrong, Spring Security throws an authentication
exception.

If they are correct, it returns an authenticated `Authentication`
object.

Then we get the authenticated user and generate a JWT.

The password is not placed inside the JWT.

## 21. Create a protected controller

Let's create an endpoint that only authenticated users can access.

### ProfileController.java

``` java
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class ProfileController {

    @GetMapping("/profile")
    public String profile(Authentication authentication) {
        return "Hello, " + authentication.getName()
                + "! You are authenticated.";
    }
}
```

The `Authentication` parameter is supplied by Spring Security for the
current request.

If the JWT filter successfully authenticated the request,
`authentication.getName()` returns the username.

## 22. Test the application using Postman

Let's test the complete flow.

### Step 1: Start Spring Boot

Run the application.

For Maven:

``` bash
./mvnw spring-boot:run
```

On Windows:

``` bash
mvnw.cmd spring-boot:run
```

### Step 2: Login

Send:

``` http
POST http://localhost:8080/auth/login
Content-Type: application/json
```

Body:

``` json
{
  "username": "rahim",
  "password": "password123"
}
```

You should receive a response similar to:

``` json
{
  "accessToken": "YOUR_GENERATED_JWT",
  "tokenType": "Bearer"
}
```

Copy the access token.

### Step 3: Access the protected endpoint

Send:

``` http
GET http://localhost:8080/api/profile
Authorization: Bearer YOUR_GENERATED_JWT
```

Expected response:

``` text
Hello, rahim! You are authenticated.
```

### Step 4: Try without a token

Send the same request without the `Authorization` header.

Spring Security should reject it with HTTP `401 Unauthorized`.

### Step 5: Try an invalid token

Change one or more characters in the token and send the request again.

The server should reject it because the signature is invalid.

### Step 6: Try an expired token

Wait until the access token expires, then send it again.

The server should reject it.

------------------------------------------------------------------------

# Part 3: Important Spring Security concepts

## 23. Authentication versus authorization

These are different concepts.

**Authentication:** Who are you?

Example: Rahim has successfully logged in.

**Authorization:** What are you allowed to do?

Example:

-   Rahim has the `USER` role.
-   An administrator has the `ADMIN` role.
-   Only administrators can delete users.

A valid JWT does not mean a user can access every endpoint.

### Example: Role-based endpoint protection

You can protect an endpoint using Spring Security:

``` java
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/admin")
public class AdminController {

    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/dashboard")
    public String dashboard() {
        return "Welcome to the admin dashboard";
    }
}
```

To use method-level security, enable it:

``` java
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}
```

Spring's `hasRole("ADMIN")` normally checks for the authority
`ROLE_ADMIN`.

If you add roles to JWT claims yourself, make sure your application maps
them to the correct Spring Security authorities. The earlier simple
example loads authorities from `UserDetailsService`, rather than
trusting a role claim in the token.

## 24. What happens when a JWT expires?

A token can contain an `exp` claim.

Example:

``` json
{
  "sub": "rahim",
  "exp": 2000000000
}
```

After the expiration time, the token must be rejected.

The JJWT parser used in the example checks expiration when parsing the
signed claims.

A common beginner mistake is to decode a token and check its `exp` value
without verifying its signature. Never use an unverified payload to
authenticate a user.

## 25. Can a JWT be logged out?

This is a very important detail.

A self-contained signed JWT may remain valid until it expires.

Deleting the token from the browser does not invalidate a copy that an
attacker may already have stolen.

Common strategies:

1.  Keep access tokens short-lived.
2.  Revoke or rotate refresh tokens.
3.  Use a denylist for tokens that must be rejected before expiry.
4.  Use server-side sessions when immediate centralized logout is
    important.

For a beginner project, short-lived access tokens are a reasonable
starting point. A real application needs a complete logout and
refresh-token design.

## 26. Where should the frontend store a JWT?

This depends on the type of client and your threat model.

### HttpOnly cookie

A server can set a cookie with:

-   `HttpOnly`: JavaScript cannot read the cookie.
-   `Secure`: The browser sends it only over HTTPS.
-   `SameSite`: Controls when the browser sends it in cross-site
    situations.

Cookies are often a good choice for browser authentication when
configured correctly.

However, cookies are automatically attached to matching requests, so you
must consider CSRF protection.

### localStorage

JavaScript can read localStorage.

If an attacker can run malicious JavaScript on your site through an XSS
vulnerability, that script may steal a token stored there.

Avoid treating localStorage as automatically safe, especially for
long-lived bearer tokens.

### A practical starting point

For a browser-based application, consider server-managed sessions or a
carefully configured HttpOnly-cookie approach.

For a mobile app or a non-browser client, use the platform's appropriate
secure credential storage.

Whatever you choose:

-   Use HTTPS.
-   Avoid putting tokens in URLs.
-   Never log tokens.
-   Protect against XSS.
-   Consider CSRF if credentials are sent automatically by the browser.

## 27. Common JWT mistakes

Avoid these mistakes:

1.  Putting passwords or secrets in the payload.
2.  Using a weak signing secret.
3.  Committing the signing secret to Git.
4.  Accepting unexpected signing algorithms.
5.  Not checking expiration.
6.  Not validating issuer and audience.
7.  Trusting a decoded payload without signature verification.
8.  Assuming logout automatically invalidates every copy of a token.
9.  Sending tokens over plain HTTP.
10. Logging access tokens.
11. Assuming a valid token grants access to every endpoint.
12. Disabling CSRF without understanding how the browser sends
    credentials.

## 28. Encoding, signing, and encryption

These words mean different things.

  Concept      Meaning                                          Hides the data?
  ------------ ------------------------------------------------ -----------------
  Encoding     Changes how data is represented                  No
  Signing      Helps detect changes and verify the signer       No
  Encryption   Makes data unreadable without a decryption key   Yes

A typical three-part signed JWT is encoded and signed, not encrypted.

An encrypted JWT uses a different format and process called JWE.

------------------------------------------------------------------------

# Part 4: Practice and review

## 29. Practice exercise

Imagine you decode this JWT payload:

``` json
{
  "sub": "rahim",
  "role": "USER",
  "iss": "my-spring-app",
  "aud": "my-spring-api",
  "iat": 2000000000,
  "exp": 2000000900
}
```

Answer these questions:

1.  Which user does the token refer to?
2.  What role is listed?
3.  Who issued the token?
4.  Which API is the intended audience?
5.  What does `exp` mean?
6.  Does this payload prove that the token is valid?
7.  Should the application allow this user to access an admin-only
    endpoint just because the token is valid?

### Answers

1.  `rahim`
2.  `USER`
3.  `my-spring-app`
4.  `my-spring-api`
5.  Expiration time
6.  No. The server must verify the signature and validate the claims.
7.  No. The application must check authorization.

## 30. What you should understand now

You should now be able to explain:

-   What JWT means.
-   Why applications use tokens.
-   What the header, payload, and signature do.
-   Why a signed JWT payload is not secret.
-   What claims such as `sub`, `iss`, `aud`, and `exp` mean.
-   How a user logs in and receives a token.
-   How a client sends a token using the `Authorization` header.
-   How Spring Security filters process incoming requests.
-   What `SecurityContextHolder` does.
-   What `AuthenticationManager` does.
-   How access tokens differ from refresh tokens.
-   Why authentication and authorization are different.
-   Why JWT needs a careful expiration, storage, and logout strategy.

## 31. A recommended learning path

Do not try to memorize all the code at once. Learn it in this order:

1.  Understand the three JWT parts: header, payload, signature.
2.  Understand signing and verification.
3.  Understand the login flow.
4.  Learn Spring Security's `AuthenticationManager`.
5.  Understand `UserDetails` and `UserDetailsService`.
6.  Understand filters and `SecurityContextHolder`.
7.  Build the login endpoint.
8.  Build the JWT filter.
9.  Protect one endpoint.
10. Add database-backed users.
11. Add role-based authorization.
12. Learn refresh-token rotation and revocation.

## 32. Official documentation and useful references

Use official documentation when checking details or adapting this guide
to a newer version:

-   Spring Security: https://docs.spring.io/spring-security/reference/
-   Spring Boot: https://docs.spring.io/spring-boot/reference/
-   JJWT project and documentation: https://github.com/jwtk/jjwt
-   JWT introduction and specification overview:
    https://jwt.io/introduction
-   JSON Web Token standard (RFC 7519):
    https://www.rfc-editor.org/rfc/rfc7519

------------------------------------------------------------------------

## Final mental picture

Think of a JWT as a digital note carrying claims about a user, with a
tamper-evident seal.

-   The **header** describes the token.
-   The **payload** contains claims.
-   The **signature** helps prove that the signed content has not been
    changed.
-   **Spring Security** checks the token and establishes authentication
    for the request.
-   Your application still decides what the authenticated user is
    allowed to do.

The most important idea is this:

**A JWT is not magic. It is a signed token that your backend must verify
and use carefully.**
