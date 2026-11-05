# Beginner's Guide to Your `JwtService`

This guide explains your Java/Spring Boot class in simple language, with
examples.

## 1. What does `JwtService` do?

A JWT (JSON Web Token) is a compact string that carries information
called **claims**. Your service has three main responsibilities:

1.  Create and sign an access token after a user logs in.
2.  Read claims from a token and convert them into Java types.
3.  Check whether a token passes signature and time-validity checks.

A signed JWT is **not encrypted**. People who obtain it can generally
decode and read its payload. The signature helps detect changes; it does
not hide the data.

A compact JWT looks like:

``` text
header.payload.signature
```

## 2. The package and imports

``` java
package com.example.backend.service;
```

This places the class in your application's `service` package.

The important imports are:

  Import                Purpose
  --------------------- ----------------------------------------------------
  `AuthenticatedUser`   Supplies the logged-in user's information.
  `AccessContext`       Enum describing the user's current access context.
  `Claims`              Represents the data inside a parsed JWT payload.
  `JwtException`        Represents JWT parsing/validation errors.
  `Jwts`                JJWT API for building and parsing tokens.
  `Decoders`            Decodes the Base64 secret.
  `Keys`                Creates a cryptographic key from bytes.
  `@Value`              Reads values from Spring configuration.
  `@Service`            Registers this class as a Spring service bean.
  `SecretKey`           Holds the key used to sign and verify tokens.
  `Instant`, `Date`     Represent times.
  `UUID`                Represents unique identifiers.

## 3. `@Service`

``` java
@Service
public class JwtService {
```

`@Service` tells Spring to create and manage this class so other
Spring-managed classes can use it.

For example, another service can receive it through constructor
injection:

``` java
@Service
public class AuthService {
    private final JwtService jwtService;

    public AuthService(JwtService jwtService) {
        this.jwtService = jwtService;
    }
}
```

Spring creates the `JwtService` object for you.

## 4. The two fields

``` java
private final SecretKey signingKey;
private final long accessTokenExpiration;
```

-   `signingKey`: the cryptographic secret used to sign new JWTs and
    verify existing ones.
-   `accessTokenExpiration`: how long an access token lasts, in
    **milliseconds**.

Examples:

-   `900000` milliseconds = 15 minutes.
-   `3600000` milliseconds = 1 hour.

Think of the signing key as a special stamp only your backend should
possess. The backend uses it to create a signature and later checks that
signature.

## 5. The constructor: reading configuration

``` java
public JwtService(
        @Value("${jwt.secret}") String secret,
        @Value("${jwt.access-token-expiration}") long accessTokenExpiration
) {
    byte[] keyBytes = Decoders.BASE64.decode(secret);

    this.signingKey = Keys.hmacShaKeyFor(keyBytes);
    this.accessTokenExpiration = accessTokenExpiration;
}
```

Spring calls this constructor when it creates the service.

### Step 1: Read the settings

`@Value("${jwt.secret}")` asks Spring for the `jwt.secret` setting.

`@Value("${jwt.access-token-expiration}")` asks for the token lifetime.

For example, `application.yml` could contain:

``` yaml
jwt:
  secret: ${JWT_SECRET}
  access-token-expiration: 900000
```

`${JWT_SECRET}` means the actual secret is supplied through an
environment variable. This is safer than committing a real secret into
Git.

### Step 2: Decode Base64

``` java
byte[] keyBytes = Decoders.BASE64.decode(secret);
```

The configuration value is expected to be Base64 text. This line decodes
it into bytes.

Base64 is an encoding, **not encryption**. It does not make a weak
secret strong.

### Step 3: Create the HMAC key

``` java
this.signingKey = Keys.hmacShaKeyFor(keyBytes);
```

This creates a key for HMAC-based signing. Your service uses the same
secret key to sign and verify tokens.

Use a strong, randomly generated key. For HS256, the key must contain at
least 256 bits (32 bytes) of key material. Do not use a short,
human-readable password.

### Step 4: Store the expiration setting

``` java
this.accessTokenExpiration = accessTokenExpiration;
```

Saves the configured lifetime for use when generating tokens.

## 6. `generateAccessToken()`: creating a JWT

``` java
public String generateAccessToken(AuthenticatedUser user) {
```

This method receives an `AuthenticatedUser` and returns a JWT string.

Imagine the user has successfully logged in. Your application has an
`AuthenticatedUser` object containing the user's IDs, role, tenant
information, and other details.

### Step 1: Get the current time

``` java
Instant now = Instant.now();
```

Gets the current moment. Imagine it is 10:00 AM.

### Step 2: Calculate when the token expires

``` java
Instant expiration = now.plusMillis(accessTokenExpiration);
```

If the configured duration is 15 minutes, the expiration time is 10:15
AM.

`plusMillis()` is used because the configuration is measured in
milliseconds.

### Step 3: Start building the token

``` java
return Jwts.builder()
```

This starts JJWT's token builder. The following calls add information to
it.

### Step 4: Add the subject

``` java
.subject(user.userAuthId().toString())
```

The JWT standard calls `sub` the **subject**. In your application it
stores the user's authentication ID.

For example:

``` json
"sub": "11111111-1111-1111-1111-111111111111"
```

`.toString()` converts the Java UUID into text.

### Step 5: Add custom claims

A **claim** is one piece of information stored in the JWT payload.

#### Session ID

``` java
.claim("sid", user.sessionId().toString())
```

Stores the user's session ID. It can help identify a particular login
session. Merely putting it in the token does not check whether that
session is still active; some other code must perform that check.

#### Role

``` java
.claim("role", user.role())
```

Stores a role such as `"ADMIN"` or `"USER"`. Your authorization logic
can use it to decide which actions the user may perform.

Only trust this claim after the backend has successfully verified the
token.

#### User tenant ID

``` java
.claim("tenant_id", user.userTenantId() != null
        ? user.userTenantId().toString() : null)
```

Stores the tenant associated with the user's account, if one exists.

The `? :` syntax is called the **ternary operator**. It means:

-   If `user.userTenantId()` is not null, convert it to a string.
-   Otherwise, use null.

It is a shorter form of an `if/else`:

``` java
String tenantId = null;
if (user.userTenantId() != null) {
    tenantId = user.userTenantId().toString();
}
```

A tenant is often an organization or customer account in a multi-tenant
application.

#### Token version

``` java
.claim("token_version", user.tokenVersion())
```

Stores the user's token version.

For example, a token may contain version `1`. If your application later
changes the user's current token version to `2`, it can reject old
tokens containing version `1`.

Important: **this class only writes and reads the version**. It does not
compare it with the latest database value. That check must happen
elsewhere.

#### Access context

``` java
.claim("access_context", user.accessContext().name())
```

Stores the name of the access-context enum.

For example, if the enum is:

``` java
public enum AccessContext {
    PERSONAL,
    TENANT
}
```

then `AccessContext.TENANT.name()` returns the string `"TENANT"`.

#### Context tenant ID

``` java
.claim("context_tenant_id", user.contextTenantId() != null
        ? user.contextTenantId().toString() : null)
```

Stores the tenant for the user's current access context, if there is
one.

This may have a different meaning from `tenant_id`:

-   `tenant_id`: tenant associated with the user's account.
-   `context_tenant_id`: tenant the user is currently acting within.

The exact meaning depends on your application's design.

### Step 6: Add the issue time

``` java
.issuedAt(Date.from(now))
```

Adds the standard `iat` claim, meaning "issued at." `Date.from()`
converts an `Instant` into a `Date` for the JJWT API.

### Step 7: Add the expiration time

``` java
.expiration(Date.from(expiration))
```

Adds the standard `exp` claim. It tells the parser when the token
expires.

### Step 8: Sign the token

``` java
.signWith(signingKey)
```

Creates the cryptographic signature using your secret key.

If someone changes the payload after the token is signed, the signature
will no longer match. Without the secret key, they cannot create the
correct replacement signature.

### Step 9: Make the compact token string

``` java
.compact();
```

Finishes building the JWT and returns it as a string, typically three
sections separated by dots.

### Generation flow

``` text
AuthenticatedUser
       |
       v
Read IDs, role, tenant and context
       |
       v
Add claims to JWT
       |
       v
Add issue and expiration times
       |
       v
Sign with secret key
       |
       v
Return JWT string
```

## 7. `extractAllClaims()`: parsing and verifying a token

``` java
public Claims extractAllClaims(String token) {
    return Jwts.parser()
            .verifyWith(signingKey)
            .build()
            .parseSignedClaims(token)
            .getPayload();
}
```

This method takes a JWT string and returns its payload claims.

Line by line:

1.  `Jwts.parser()` starts creating a parser.
2.  `.verifyWith(signingKey)` tells it which key to use to verify the
    signature.
3.  `.build()` creates the configured parser.
4.  `.parseSignedClaims(token)` parses a signed JWT and verifies its
    signature.
5.  `.getPayload()` returns the claims inside the payload.

This is more than simply decoding the token. It verifies the signature
first.

The parser also checks standard time constraints such as expiration when
an `exp` claim is present. An expired token normally causes a JWT
exception.

## 8. The extraction methods: getting individual values

Once the token has been parsed, you can retrieve individual claims.

### User authentication ID

``` java
public UUID extractUserAuthId(Claims claims) {
    return UUID.fromString(claims.getSubject());
}
```

-   `claims.getSubject()` reads the `sub` value as text.
-   `UUID.fromString(...)` converts that text into a Java UUID.

### Session ID

``` java
public UUID extractSessionId(Claims claims) {
    return UUID.fromString(claims.get("sid", String.class));
}
```

Reads the `sid` claim as a string and converts it into a UUID.

`String.class` tells JJWT which Java type you expect.

### Role

``` java
public String extractRole(Claims claims) {
    return claims.get("role", String.class);
}
```

Reads the role as a string, for example `"ADMIN"`.

### User tenant ID

``` java
public UUID extractUserTenantId(Claims claims) {
    return extractNullableUuid(claims, "tenant_id");
}
```

Uses the helper method explained in the next section. It returns a UUID
or null.

### Token version

``` java
public Long extractTokenVersion(Claims claims) {
    return claims.get("token_version", Long.class);
}
```

Reads the token version as a Java `Long`, for example `1L`.

### Access context

``` java
public AccessContext extractAccessContext(Claims claims) {
    return AccessContext.valueOf(
            claims.get("access_context", String.class)
    );
}
```

Reads the string and converts it back into an enum value.

For example, `AccessContext.valueOf("TENANT")` returns
`AccessContext.TENANT`.

The string must match an enum constant exactly. Otherwise, `valueOf()`
throws `IllegalArgumentException`.

### Context tenant ID

``` java
public UUID extractContextTenantId(Claims claims) {
    return extractNullableUuid(claims, "context_tenant_id");
}
```

Reads the context tenant ID as a UUID, or returns null when it is
absent/null.

## 9. The helper: `extractNullableUuid()`

``` java
private UUID extractNullableUuid(Claims claims, String claimName) {
    String value = claims.get(claimName, String.class);

    return value != null ? UUID.fromString(value) : null;
}
```

This is a reusable helper method. `private` means it can be called
inside `JwtService`, but not directly from other classes.

It receives the claims and the name of the claim to read.

For example:

``` java
extractNullableUuid(claims, "tenant_id");
```

First it reads the claim as a string. Then:

-   If the value exists, it converts it to a UUID.
-   If the value is null, it returns null.

This avoids repeating the same conversion logic for both tenant ID
fields.

## 10. `isTokenValid()`: checking a token

``` java
public boolean isTokenValid(String token) {
    try {
        extractAllClaims(token);
        return true;
    } catch (JwtException | IllegalArgumentException exception) {
        return false;
    }
}
```

A `boolean` can be either `true` or `false`.

The method tries to parse the token:

``` java
extractAllClaims(token);
```

If parsing succeeds, it returns `true`.

If parsing throws a `JwtException` (for example, an invalid signature or
expired token) or an `IllegalArgumentException`, the `catch` block runs
and returns `false`.

The `|` in the catch statement is Java's **multi-catch** syntax. It
means catch either exception type.

### Example: someone changes the role

Suppose the original payload contains:

``` json
{
  "sub": "11111111-1111-1111-1111-111111111111",
  "role": "USER"
}
```

Someone edits it to:

``` json
{
  "sub": "11111111-1111-1111-1111-111111111111",
  "role": "ADMIN"
}
```

They do not know your signing key, so they cannot generate the correct
signature for the changed payload. When your backend parses the changed
token, signature verification fails and `isTokenValid()` returns
`false`.

### What this method does NOT check

This is an important distinction. A token can have a valid signature and
still be unsuitable for access.

This class does not itself:

-   Check whether the user still exists in the database.
-   Check whether the session ID is still active.
-   Compare `token_version` with the latest database value.
-   Check whether the account has been disabled.
-   Decide whether the user is allowed to call a particular endpoint.

Your authentication filter or another service must perform those checks
if your application requires them.

Also, this method catches the two listed exception types. Unexpected
errors of other types are not automatically converted to `false`.

## 11. How it fits into the application

### When a user logs in

``` text
User sends email and password
          |
          v
Backend checks credentials
          |
          v
Backend creates AuthenticatedUser
          |
          v
jwtService.generateAccessToken(user)
          |
          v
JWT string is returned to the client
```

### When the client makes another request

The client commonly sends the token in this HTTP header:

``` http
Authorization: Bearer eyJ...
```

A simplified example of how a filter might use your service:

``` java
String token = authorizationHeader.substring(7);

if (jwtService.isTokenValid(token)) {
    Claims claims = jwtService.extractAllClaims(token);

    UUID userId = jwtService.extractUserAuthId(claims);
    String role = jwtService.extractRole(claims);

    // Continue authentication.
}
```

This is only an illustration, not a complete production filter. Notice
that it parses the token twice: once inside `isTokenValid()` and again
in `extractAllClaims()`. A real filter can parse once, catch parsing
exceptions, and reuse the returned `Claims`.

A simplified request flow:

``` text
HTTP request arrives
        |
        v
Read Authorization header
        |
        v
Get Bearer token
        |
        v
Verify signature and expiration
        |
     +--+--+
     |     |
   Valid Invalid
     |     |
     v     v
Read claims  Reject / leave unauthenticated
     |
     v
Check user/session if required
     |
     v
Set Spring Security Authentication
     |
     v
Continue to controller
```

## 12. Quick glossary

  -----------------------------------------------------------------------
  Term                                Simple meaning
  ----------------------------------- -----------------------------------
  JWT                                 A compact token that carries
                                      claims.

  Header                              JWT section containing metadata
                                      such as the algorithm.

  Payload                             JWT section containing the
                                      claims/data.

  Signature                           Cryptographic value used to detect
                                      tampering.

  Claim                               One piece of information in the
                                      payload.

  `sub`                               Subject; here, the user's
                                      authentication ID.

  `iat`                               Time the token was issued.

  `exp`                               Time the token expires.

  Signing key                         Secret key used by this service to
                                      sign and verify tokens.

  UUID                                A standard unique identifier
                                      format.

  Claims                              JJWT's Java representation of the
                                      payload.

  Tenant                              Often an organization/customer
                                      boundary in a multi-tenant app.

  Enum                                A Java type containing a fixed set
                                      of named values.
  -----------------------------------------------------------------------

## 13. Security points to remember

1.  **Protect `jwt.secret`.** Store it in a secure environment variable
    or secret manager. Never commit the real key to Git.
2.  **Use a strong random key.** For HS256, use at least 32 bytes of
    random key material.
3.  **JWT payloads are readable.** Never put passwords or secrets in
    claims.
4.  **Use HTTPS.** It protects the token while travelling between client
    and server.
5.  **Choose a sensible expiration.** Short-lived access tokens reduce
    the time a stolen token can be used.
6.  **Consider issuer and audience checks.** This class currently does
    not set or require `iss` (issuer) or `aud` (audience).
7.  **Implement revocation checks separately.** A valid signature does
    not prove a session has not been logged out or revoked.
8.  **Validate claim values.** A correctly signed token may still
    contain values your current application should no longer accept.

## 14. Check your understanding

Try these questions before reading the answers.

1.  What does `Instant now = Instant.now();` do?
2.  If `accessTokenExpiration` is `900000`, how long does the token
    last?
3.  What does `.subject(user.userAuthId().toString())` store?
4.  Why does the parser call `.verifyWith(signingKey)`?
5.  What should happen if someone changes `role` from `USER` to `ADMIN`
    without the signing key?
6.  Does `isTokenValid()` check the session status in your database?

### Answers

1.  It gets the current time.
2.  900,000 milliseconds is 15 minutes.
3.  It stores the user's authentication ID as the JWT subject (`sub`).
4.  It tells the parser which key to use to verify the signature.
5.  Signature verification should fail, so the token should be rejected.
6.  No. This class does not query the database or check whether the
    session is active.

## Final mental model

Remember these three jobs:

-   **`generateAccessToken()`** --- creates and signs a token.
-   **`extractAllClaims()` and the `extract...()` methods** ---
    verify/read the token and retrieve its information.
-   **`isTokenValid()`** --- returns true when parsing and the parser's
    checks succeed; returns false for the exception types it catches.

Your `JwtService` is the JWT utility for your backend. It creates tokens
and helps other parts of the application verify and read them. Your
authentication filter and other services use that information to
authenticate requests and apply session and authorization rules.
