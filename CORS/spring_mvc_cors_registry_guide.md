# Spring MVC CORS Configuration: A Complete Guide to `CorsRegistry`

This guide explains Spring MVC's `CorsRegistry` and every public
configuration method you commonly use after calling `addMapping()`. It
uses Java examples and relates the concepts to a Spring Boot + Spring
Security + PostgreSQL Apartment Management System.

> **Version note:** The examples use the Spring MVC API available in
> modern Spring Framework 6.x / Spring Boot 3.x. Some methods,
> especially `allowPrivateNetwork`, depend on the exact Spring Framework
> version. Check your project's Spring Framework version before using
> newer methods.

------------------------------------------------------------------------

## 1. First, what are we configuring?

CORS means **Cross-Origin Resource Sharing**. It is a browser security
mechanism that controls whether JavaScript running on one origin may
access a response from another origin.

An **origin** consists of:

-   Scheme/protocol: `http` or `https`
-   Hostname: `localhost`, `example.com`
-   Port: `3000`, `8080`

These are different origins:

``` text
http://localhost:3000
http://localhost:8080
https://localhost:3000
http://127.0.0.1:3000
```

For example, your frontend may run at:

``` text
http://localhost:5500
```

while your Spring Boot API runs at:

``` text
http://localhost:8080
```

The browser considers these different origins because their ports
differ. A request from the frontend to the API is therefore
cross-origin.

CORS does **not** replace authentication, authorization, or tenant
isolation. It is not a firewall and does not stop requests made by tools
such as curl or Postman. It controls browser access to cross-origin
responses.

------------------------------------------------------------------------

## 2. Where `CorsRegistry` fits

A typical configuration class looks like this:

``` java
package com.example.backend.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:5500");
    }
}
```

The pieces have different responsibilities:

-   `@Configuration`: tells Spring that this class contributes
    configuration.
-   `WebMvcConfigurer`: lets you customize Spring MVC without replacing
    its normal configuration.
-   `addCorsMappings(...)`: a callback Spring MVC gives you to register
    URL-based CORS rules.
-   `CorsRegistry`: the registry to which you add CORS mappings.
-   `addMapping(...)`: creates a CORS configuration for a URL path
    pattern.
-   The returned `CorsRegistration`: lets you configure the rules for
    that path.

### Does `CorsRegistry` have many methods?

The main method you call on `CorsRegistry` is:

``` java
registry.addMapping("/api/**");
```

Most of the methods you will use---such as `allowedOrigins`,
`allowedMethods`, and `maxAge`---belong to the **`CorsRegistration`**
object returned by `addMapping()`.

That is why this fluent chain works:

``` java
registry.addMapping("/api/**")
        .allowedOrigins("http://localhost:5500")
        .allowedMethods("GET", "POST");
```

------------------------------------------------------------------------

# Part I --- The mapping

## 3. `addMapping(String pathPattern)`

``` java
registry.addMapping("/api/**");
```

This creates a CORS rule for requests whose URL paths match the supplied
pattern.

It does not define which websites are allowed. It defines **which
backend paths this CORS rule applies to**.

### Common path patterns

  Pattern             Matches
  ------------------- ------------------------------------------
  `/**`               All paths
  `/api/**`           All paths under `/api/`
  `/api/v1/**`        All paths under `/api/v1/`
  `/api/v1/auth/**`   Authentication endpoints under that path
  `/public/**`        Paths under `/public/`

Suppose your API has these endpoints:

``` text
/api/v1/auth/login
/api/v1/auth/refresh
/api/v1/users/me
/api/v1/apartments
/api/v1/bills
/api/v1/expenses
```

This mapping:

``` java
registry.addMapping("/api/**");
```

applies the same CORS policy to all of them.

This mapping:

``` java
registry.addMapping("/api/v1/auth/**");
```

applies only to the authentication endpoints.

### Can I add multiple mappings?

Yes. You can register different CORS policies for different groups of
endpoints:

``` java
@Override
public void addCorsMappings(CorsRegistry registry) {

    registry.addMapping("/api/v1/public/**")
            .allowedOrigins("*")
            .allowedMethods("GET");

    registry.addMapping("/api/v1/**")
            .allowedOrigins("http://localhost:5500")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS");
}
```

Be deliberate about overlapping patterns. Prefer a small number of
clear, non-conflicting policies unless you have a real reason to
separate them.

**For your application:** `/**` is reasonable while the API is small and
all endpoints share the same frontend origins and CORS policy. You can
narrow it to `/api/**` if all browser-facing endpoints use that prefix.

------------------------------------------------------------------------

# Part II --- Origin rules

## 4. `allowedOrigins(String... origins)`

``` java
.allowedOrigins(
        "http://localhost:5500",
        "http://localhost:3000"
)
```

Specifies the exact origins permitted to make browser cross-origin
requests.

An origin includes the scheme, hostname, and port where applicable. It
does not include a URL path.

Correct:

``` text
http://localhost:5500
https://example.com
https://app.example.com
```

Incorrect as an origin:

``` text
localhost:5500
https://example.com/dashboard
https://example.com/api
```

### Exact matching matters

If you allow:

``` java
.allowedOrigins("http://localhost:5500")
```

that does not automatically allow:

``` text
http://localhost:3000
https://localhost:5500
http://127.0.0.1:5500
```

Those are separate origins.

### Can I use `*`?

You can configure:

``` java
.allowedOrigins("*")
```

This means any origin is allowed. However, the wildcard origin cannot be
combined with:

``` java
.allowCredentials(true)
```

The CORS specification does not permit the server to return
`Access-Control-Allow-Origin: *` for a credentialed request.

Because your system uses HTTP-only refresh-token cookies, you will
likely need credentialed cross-origin requests. Use explicit origins
rather than `*`.

------------------------------------------------------------------------

## 5. `allowedOriginPatterns(String... patterns)`

``` java
.allowedOriginPatterns("https://*.example.com")
```

This is an alternative to `allowedOrigins`. It supports patterns,
including a wildcard in the hostname.

For example:

``` java
.allowedOriginPatterns(
        "https://*.example.com",
        "http://localhost:*"
)
```

This can match subdomains of `example.com` and localhost origins on
different ports, subject to Spring's pattern matching rules.

### `allowedOrigins` versus `allowedOriginPatterns`

  Method                    What it supports
  ------------------------- --------------------------------------------
  `allowedOrigins`          Exact origins, or the special wildcard `*`
  `allowedOriginPatterns`   Patterns such as `https://*.example.com`

A key distinction: when an origin pattern matches, Spring returns the
**actual matched origin** in the CORS response. It does not return the
literal pattern.

Patterns can be used with credentials:

``` java
.allowedOriginPatterns("https://*.example.com")
.allowCredentials(true)
```

That does not mean you should automatically trust every subdomain. Only
use a pattern if you control and trust all matching subdomains. A
forgotten, abandoned, or user-controlled subdomain can turn a broad
pattern into a security problem.

**For your multi-tenant system:** explicit tenant origins are easier to
reason about initially. Later, if you adopt a controlled subdomain
structure, patterns may be useful---but only if every matching subdomain
is trusted.

------------------------------------------------------------------------

# Part III --- HTTP methods

## 6. `allowedMethods(String... methods)`

``` java
.allowedMethods(
        "GET",
        "POST",
        "PUT",
        "PATCH",
        "DELETE",
        "OPTIONS"
)
```

Specifies which HTTP methods are allowed for cross-origin requests.

  Method      Typical meaning                   Apartment-system example
  ----------- --------------------------------- ----------------------------
  `GET`       Read/retrieve data                Get apartment details
  `POST`      Create or submit                  Create a bill
  `PUT`       Replace/update a resource         Update an apartment record
  `PATCH`     Partially update a resource       Change an apartment status
  `DELETE`    Delete a resource                 Delete a draft bill
  `OPTIONS`   Ask about communication options   Browser CORS preflight

### Why is `OPTIONS` important?

Some cross-origin requests cause the browser to send a **preflight
request** before the actual request.

For example, the frontend wants to send:

``` http
DELETE /api/v1/apartments/123 HTTP/1.1
Origin: http://localhost:5500
Authorization: Bearer <access-token>
```

Before sending the `DELETE`, the browser may send:

``` http
OPTIONS /api/v1/apartments/123 HTTP/1.1
Origin: http://localhost:5500
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: authorization
```

The browser is asking whether this origin may use the `DELETE` method
and the `Authorization` header.

Spring's CORS processing handles the preflight when configured
correctly. Do not confuse the browser's preflight request with an
endpoint you need to implement in every controller.

### Should I use `allowedMethods("*")`?

It is possible, but an explicit list documents what your frontend is
expected to use and avoids granting unnecessary methods.

For your API, the six methods above are a sensible starting point. Add
`HEAD` or another method only if your application needs it.

------------------------------------------------------------------------

# Part IV --- Request headers

## 7. `allowedHeaders(String... headers)`

``` java
.allowedHeaders(
        "Authorization",
        "Content-Type",
        "Accept"
)
```

Specifies which request headers the browser may send in a cross-origin
request.

Headers are metadata sent with an HTTP request. They are not the same
thing as the JSON body.

### Common headers

#### `Authorization`

Your frontend sends the access token:

``` http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

This is important for your JWT-based authentication.

#### `Content-Type`

Describes the format of the request body:

``` http
Content-Type: application/json
```

For example, when creating an apartment:

``` http
POST /api/v1/apartments
Content-Type: application/json

{
    "apartmentName": "A-101",
    "floorNumber": 1
}
```

#### `Accept`

Tells the server which response media type the client prefers:

``` http
Accept: application/json
```

#### `X-Requested-With`

A header historically used by some AJAX clients:

``` http
X-Requested-With: XMLHttpRequest
```

You probably do not need it unless your frontend actually sends it.

### What if I use a custom header?

Suppose you later add a request ID:

``` http
X-Request-ID: 8a5d...
```

Then add it to the allowed headers:

``` java
.allowedHeaders(
        "Authorization",
        "Content-Type",
        "Accept",
        "X-Request-ID"
)
```

If a browser asks to use a header that is not allowed, the preflight
check can fail and the browser will not send the actual request.

### Can I use `*`?

Yes:

``` java
.allowedHeaders("*")
```

This allows requested headers in the preflight process. An explicit list
is often easier to audit. Spring also has special handling for wildcard
headers when credentials are enabled, so do not confuse the wildcard
rules for headers with the stricter wildcard rule for origins.

------------------------------------------------------------------------

# Part V --- Response headers

## 8. `exposedHeaders(String... headers)`

``` java
.exposedHeaders("Content-Disposition")
```

This method is easy to overlook.

`allowedHeaders` controls what the browser may **send**.

`exposedHeaders` controls which response headers browser JavaScript may
**read**.

By default, browser JavaScript can read only a limited set of response
headers in a cross-origin response. If your backend sends a custom
response header, the frontend may not be able to access it unless you
expose it.

### Example: downloading an invoice

Imagine the backend responds:

``` http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="invoice-2026-001.pdf"
```

Your frontend wants to read `Content-Disposition` to determine the
filename.

Configure:

``` java
.exposedHeaders("Content-Disposition")
```

Now browser JavaScript can read that response header.

### Other possible examples

``` java
.exposedHeaders(
        "Content-Disposition",
        "X-Request-ID"
)
```

This could be useful for report downloads or request tracing.

You do not need to expose every response header. Add only those that
your frontend actually needs to read.

------------------------------------------------------------------------

# Part VI --- Credentials and cookies

## 9. `allowCredentials(boolean allowCredentials)`

``` java
.allowCredentials(true)
```

Controls whether the browser is permitted to make credentialed
cross-origin requests, such as requests that include cookies.

This matters for your authentication design because you use an HTTP-only
refresh-token cookie.

### Two sides must agree

Frontend:

``` javascript
fetch("http://localhost:8080/api/v1/auth/refresh", {
    method: "POST",
    credentials: "include"
});
```

Backend:

``` java
.allowCredentials(true)
```

Both settings are needed for a cross-origin cookie request.

### What does `credentials: "include"` do?

It tells the browser to include credentials, such as eligible cookies,
with the cross-origin request.

The browser---not JavaScript---attaches an HTTP-only cookie. JavaScript
cannot read the cookie's value, which is one of the benefits of
`HttpOnly`.

### Why can't I combine credentials with `allowedOrigins("*")`?

This is invalid:

``` java
.allowedOrigins("*")
.allowCredentials(true)
```

A credentialed CORS response must identify a specific allowed origin. It
cannot say that every origin is allowed to access credentialed data.

Use:

``` java
.allowedOrigins(
        "http://localhost:5500",
        "http://localhost:3000"
)
.allowCredentials(true)
```

Or use a carefully controlled `allowedOriginPatterns` configuration.

### Important: CORS credentials are not cookie configuration

`allowCredentials(true)` does not set the cookie's:

-   `HttpOnly`
-   `Secure`
-   `SameSite`
-   `Path`
-   `Domain`
-   `Max-Age`

Those are separate cookie attributes configured when the backend creates
the cookie.

It also does not protect your application against CSRF.
Cookie-authenticated state-changing endpoints need an appropriate CSRF
strategy.

------------------------------------------------------------------------

# Part VII --- Preflight caching

## 10. `maxAge(long maxAge)`

``` java
.maxAge(3600)
```

Sets how long, in **seconds**, the browser may cache the result of a
successful CORS preflight request.

For example:

``` java
.maxAge(3600)
```

means up to one hour.

### Why cache preflight results?

Suppose the browser makes a cross-origin `DELETE` request. It first
sends an `OPTIONS` preflight request to ask whether the method and
headers are allowed.

If the browser had to repeat that preflight before every applicable
request, it would create extra network traffic.

With a cached preflight result, the browser may skip another preflight
until the cached permission expires.

      Value Duration
  --------- ------------
       `60` 1 minute
      `600` 10 minutes
     `1800` 30 minutes
     `3600` 1 hour
    `86400` 24 hours

This does **not** cache API response data, user sessions, JWTs, or
refresh tokens.

A browser may impose its own maximum cache duration, so the configured
value is not a guarantee that every browser will cache for exactly that
long.

Spring's documented default for this setting is 1800 seconds when using
its standard MVC global CORS configuration defaults.

------------------------------------------------------------------------

# Part VIII --- Advanced options

## 11. `allowPrivateNetwork(boolean allowPrivateNetwork)`

``` java
.allowPrivateNetwork(true)
```

This is an advanced option related to **Private Network Access** in
browsers.

It concerns requests from a less-private address space to a more-private
one---for example, a public website trying to reach a service on a
user's local network.

This is not the same as allowing `localhost` as a normal development
origin.

For a typical hosted Apartment Management System where the frontend and
backend are deployed on public infrastructure, you generally do not need
this option.

It is version-dependent. Spring Framework added support in the 5.3.32
line; verify that your exact dependency version exposes the method
before using it.

Do not enable it just because your application is multi-tenant.

------------------------------------------------------------------------

## 12. `combine(CorsConfiguration other)`

``` java
.combine(otherCorsConfiguration)
```

This is an advanced method for combining another `CorsConfiguration`
with the configuration being built.

It is useful when you already have a `CorsConfiguration` object and want
to merge its settings into a registration.

Most applications using a straightforward `WebMvcConfigurer` do not need
to call this method.

You can safely ignore it while learning the normal CORS configuration
flow.

------------------------------------------------------------------------

# Part IX --- Putting it all together

## 13. A complete development configuration

For your current local development setup:

``` java
package com.example.backend.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/**")
                .allowedOrigins(
                        "http://localhost:5500",
                        "http://localhost:3000",
                        "http://127.0.0.1:5500",
                        "http://127.0.0.1:3000"
                )
                .allowedMethods(
                        "GET",
                        "POST",
                        "PUT",
                        "PATCH",
                        "DELETE",
                        "OPTIONS"
                )
                .allowedHeaders(
                        "Authorization",
                        "Content-Type",
                        "Accept"
                )
                .exposedHeaders(
                        "Content-Disposition"
                )
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

Change the frontend origins and ports to match your actual development
server.

If you are not downloading files yet, you can omit
`exposedHeaders("Content-Disposition")` until you need it.

### Spring Security integration

Because your project uses Spring Security, make sure CORS processing is
enabled in your security filter chain:

``` java
http.cors(Customizer.withDefaults());
```

With Spring MVC on the classpath and CORS configured through
`WebMvcConfigurer`, Spring Security can use that MVC CORS configuration.

Avoid creating a second, conflicting CORS policy unless you have a
specific reason to do so.

------------------------------------------------------------------------

## 14. How a preflight request looks in practice

Suppose your frontend is running at `http://localhost:5500` and wants to
delete an apartment.

### Step 1: Browser sends a preflight request

``` http
OPTIONS /api/v1/apartments/123 HTTP/1.1
Host: localhost:8080
Origin: http://localhost:5500
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: authorization
```

### Step 2: Server responds with CORS permissions

A successful response may contain headers similar to:

``` http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:5500
Access-Control-Allow-Methods: GET,POST,PUT,PATCH,DELETE,OPTIONS
Access-Control-Allow-Headers: Authorization,Content-Type,Accept
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 3600
```

The exact response and status code can vary with the request and
framework configuration.

### Step 3: Browser sends the actual request

If the preflight succeeds, the browser sends:

``` http
DELETE /api/v1/apartments/123 HTTP/1.1
Host: localhost:8080
Origin: http://localhost:5500
Authorization: Bearer <access-token>
```

Your normal backend security and authorization logic still decides
whether the user is authenticated and allowed to delete that apartment.

------------------------------------------------------------------------

# Part X --- Common mistakes and important distinctions

## 15. Mistake: using `*` with credentials

Avoid:

``` java
.allowedOrigins("*")
.allowCredentials(true)
```

Use explicit origins or carefully selected origin patterns.

## 16. Mistake: assuming CORS protects the API

CORS is enforced by browsers. It does not stop Postman, curl, mobile
apps, or another backend from sending requests.

Your application still needs:

-   Authentication: who is making the request?
-   Authorization: what may that user do?
-   Tenant isolation: which tenant's data may that user access?
-   Object-level authorization: may the user access this particular
    apartment, bill, or expense?
-   CSRF protection appropriate to cookie-based authentication.

## 17. Mistake: confusing allowed and exposed headers

Remember:

  -----------------------------------------------------------------------
  Configuration           Direction               Meaning
  ----------------------- ----------------------- -----------------------
  `allowedHeaders`        Frontend → backend      Headers the browser may
                                                  send

  `exposedHeaders`        Backend → frontend      Response headers
                                                  JavaScript may read
  -----------------------------------------------------------------------

## 18. Mistake: assuming localhost origins are interchangeable

These are different:

``` text
http://localhost:5500
http://localhost:3000
http://127.0.0.1:5500
```

Allow the exact origin your frontend uses.

## 19. Mistake: forgetting cookie settings

CORS permission does not guarantee that a cookie will be sent. Cookie
`SameSite`, `Secure`, domain, path, browser context, and frontend
`credentials` settings also matter.

If the frontend and API are on different sites, cookie policy may
require `SameSite=None; Secure`. Test the actual deployed topology
rather than assuming that CORS alone is sufficient.

## 20. Mistake: treating `OPTIONS` as a business endpoint

A preflight request is a browser protocol check. Spring's CORS support
handles it. You generally should not create a controller method for
every `OPTIONS` request.

------------------------------------------------------------------------

# Part XI --- How this fits your multi-tenant architecture

Right now, your system has one backend serving multiple tenant
frontends.

For development, you can keep a small list of local origins in
`WebConfig`.

When you deploy, you can replace those local origins with the actual
frontend origins, for example:

``` java
.allowedOrigins(
        "https://elitepark.example",
        "https://greenpark.example",
        "https://dreamcity.example"
)
```

Those are illustrative domains; use your real deployed origins.

Later, if you want to resolve allowed origins dynamically from your
existing tenant data, you can build that without introducing a new table
solely for CORS. The important part is to validate the incoming `Origin`
against trusted, active tenant domains. Do not blindly reflect whatever
`Origin` the browser sends.

Also remember that a valid CORS origin does not establish which tenant's
data a user may access. Tenant identification and tenant authorization
must remain separate server-side responsibilities.

------------------------------------------------------------------------

# Quick reference

  -----------------------------------------------------------------------------------------------------
  Method                    Purpose                 Example
  ------------------------- ----------------------- ---------------------------------------------------
  `addMapping`              Choose backend URL      `addMapping("/api/**")`
                            paths                   

  `allowedOrigins`          Allow exact frontend    `allowedOrigins("http://localhost:5500")`
                            origins                 

  `allowedOriginPatterns`   Allow matching origin   `allowedOriginPatterns("https://*.example.com")`
                            patterns                

  `allowedMethods`          Allow HTTP methods      `allowedMethods("GET", "POST")`

  `allowedHeaders`          Allow request headers   `allowedHeaders("Authorization", "Content-Type")`

  `exposedHeaders`          Let JavaScript read     `exposedHeaders("Content-Disposition")`
                            response headers        

  `allowCredentials`        Allow credentialed      `allowCredentials(true)`
                            browser requests        

  `maxAge`                  Cache preflight result  `maxAge(3600)`
                            in seconds              

  `allowPrivateNetwork`     Support certain         `allowPrivateNetwork(true)`
                            private-network         
                            requests                

  `combine`                 Merge another CORS      `combine(other)`
                            configuration           
  -----------------------------------------------------------------------------------------------------

## What I recommend you remember

For day-to-day work, the most important methods are:

1.  `addMapping`
2.  `allowedOrigins`
3.  `allowedMethods`
4.  `allowedHeaders`
5.  `exposedHeaders`
6.  `allowCredentials`
7.  `maxAge`

The advanced methods can wait until you encounter a real requirement for
them.

------------------------------------------------------------------------

## Official references

-   [Spring Framework 6.2 --- CORS support in Spring
    MVC](https://springframework.org/spring-framework/reference/6.2/web/webmvc-cors.html)
-   [Spring Framework 6.2.1 --- CorsRegistration
    API](https://docs.spring.io/spring-framework/docs/6.2.1/javadoc-api/org/springframework/web/servlet/config/annotation/CorsRegistration.html)
-   [Spring Framework --- CorsConfiguration
    source](https://github.com/spring-projects/spring-framework/blob/main/spring-web/src/main/java/org/springframework/web/cors/CorsConfiguration.java)
