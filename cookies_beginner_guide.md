<br>

# Web Cookies

<br>
<br>

## 1. What is a cookie?

A **web cookie** is a small piece of data that a website asks a browser
to store. The browser can later send that data back to the website with
matching requests.

Think of a cookie like a small claim ticket:

1.  You visit a website.
2.  The website gives your browser a cookie.
3.  Your browser stores it.
4.  On later requests that match the cookie's rules, the browser sends
    it back.
5.  The website can use the value to recognize a session or remember a
    preference.

A cookie is not automatically secure, encrypted, or trustworthy. It is
simply data stored and managed by the browser.

### A simple example

The server sends this HTTP response:

``` http
HTTP/1.1 200 OK
Set-Cookie: theme=dark
```

The browser stores a cookie named `theme` with the value `dark`.

Later, when the browser makes an eligible request, it may send:

``` http
GET /profile HTTP/1.1
Host: example.com
Cookie: theme=dark
```

The browser, not your frontend JavaScript, normally adds matching
cookies automatically.

<br>

## 2. The two important HTTP headers

There are two headers to remember.

```
  -----------------------------------------------------------------------
  Header                  Direction               Purpose
  ----------------------- ----------------------- -----------------------
  `Set-Cookie`            Server → Browser        Instructs the browser
                                                  to store, update, or
                                                  delete a cookie

  `Cookie`                Browser → Server        Carries applicable
                                                  cookie name/value pairs
                                                  to the server
  -----------------------------------------------------------------------

```

### `Set-Cookie`

A server can send a response like:

``` http
HTTP/1.1 200 OK
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
```

This tells the browser to store the cookie and apply the listed
attributes.

### `Cookie`

When a later request matches the cookie's rules, the browser may send:

``` http
GET /account HTTP/1.1
Host: example.com
Cookie: session_id=abc123
```

Notice that the request's `Cookie` header contains cookie name/value
pairs. It does not send the attributes such as `HttpOnly`, `Secure`, or
`SameSite` back to the server.

<br>

## 3. Where are cookies stored?

Cookies are stored in the browser's cookie store, separately for
relevant hosts and cookie scopes.

In Chrome or Chromium-based browsers, you can inspect them:

1.  Open a website.
2.  Open Developer Tools (`F12` or `Ctrl+Shift+I`).
3.  Select **Application**.
4.  Under **Storage**, select **Cookies**.
5.  Choose the website.

You may see columns such as:

-   Name
-   Value
-   Domain
-   Path
-   Expires / Max-Age
-   Size
-   HttpOnly
-   Secure
-   SameSite
-   Priority (browser-dependent)

Browser developer tools differ slightly, but the basic information is
similar.

<br>

## 4. Cookie attributes

A cookie consists of a name and value, and can have attributes that
control its lifetime, scope, and handling.

For example:

``` http
Set-Cookie: refresh_token=some-random-value; Path=/api/auth/refresh; Max-Age=604800; HttpOnly; Secure; SameSite=Lax
```

Let's understand the important attributes one by one.

### 4.1 Expires

`Expires` gives the cookie an absolute expiration date and time.

``` http
Set-Cookie: preference=dark; Expires=Tue, 10 Nov 2026 12:00:00 GMT
```

After the expiration time, the browser should no longer send the cookie.

The date must use the HTTP cookie date format (usually expressed in
GMT/UTC).

### 4.2 Max-Age

`Max-Age` gives the cookie a lifetime in seconds.

``` http
Set-Cookie: preference=dark; Max-Age=3600
```

This asks the browser to keep the cookie for 3,600 seconds---one
hour---from when it is set.

If both `Max-Age` and `Expires` are present, `Max-Age` takes precedence
in modern browsers.

#### What if neither is specified?

The cookie is generally a **session cookie**. It is intended to last for
the browser session, but browser session restoration may preserve
session cookies after a restart. Do not treat session-cookie lifetime as
a security guarantee.

### 4.3 Domain

`Domain` controls which hosts can receive a cookie.

Suppose the response comes from:

``` text
https://api.example.com
```

If it sets:

``` http
Set-Cookie: session_id=abc123; Domain=example.com
```

the cookie can be sent to `example.com` and its subdomains, subject to
the other cookie rules.

If the server omits `Domain`:

``` http
Set-Cookie: session_id=abc123
```

the browser creates a **host-only cookie**. It is sent only to the host
that set it.

For example, a host-only cookie set by `api.example.com` is not sent to
`admin.example.com`.

#### Security advice

Prefer omitting `Domain` unless sharing a cookie across subdomains is
genuinely necessary. A broader domain scope makes the cookie available
to more hosts.

A cookie cannot normally set a `Domain` that is an unrelated domain or a
public suffix such as `com`.

### 4.4 Path

`Path` limits the URL paths for which the browser sends a cookie.

``` http
Set-Cookie: refresh_token=abc123; Path=/api/auth/refresh
```

The cookie is sent to matching paths, such as `/api/auth/refresh`, but
not to unrelated paths such as `/api/users`.

If `Path` is omitted, the browser derives a default path from the URL
that set the cookie.

#### Important limitation

`Path` is a sending-scope rule, not a strong security boundary. It
should not be relied on to protect a cookie from other applications or
scripts on the same host.

### 4.5 Secure

``` http
Set-Cookie: session_id=abc123; Secure
```

`Secure` tells the browser to send the cookie only over secure
connections, normally HTTPS.

This helps prevent the cookie from being exposed over an unencrypted
HTTP connection.

For production authentication cookies, use `Secure`. Local development
over plain HTTP may need a separate development configuration.

### 4.6 HttpOnly

``` http
Set-Cookie: refresh_token=abc123; HttpOnly
```

`HttpOnly` prevents browser JavaScript from reading the cookie through
APIs such as `document.cookie`.

For example:

``` javascript
console.log(document.cookie);
```

An HttpOnly cookie will not appear in that JavaScript-readable cookie
list.

The browser can still attach the cookie to eligible HTTP requests. The
server can still receive it.

#### What HttpOnly protects against

It helps prevent an attacker who can run JavaScript in the page from
directly reading and exfiltrating that cookie.

#### What HttpOnly does not protect against

-   It does not stop CSRF.
-   It does not stop malicious JavaScript from making requests as the
    user from within the compromised site.
-   It does not encrypt the cookie value.
-   It does not make an XSS vulnerability harmless.

HttpOnly is valuable, but it is only one layer of defense.

### 4.7 SameSite

`SameSite` controls whether a cookie is sent with requests in cross-site
contexts. It is an important defense against many CSRF attacks.

There are three commonly used values.

#### `SameSite=Strict`

``` http
Set-Cookie: session_id=abc123; SameSite=Strict
```

The browser generally withholds the cookie from cross-site requests,
including many navigations initiated from another site.

This is restrictive and can affect user experience when someone follows
a link to your site from elsewhere.

#### `SameSite=Lax`

``` http
Set-Cookie: session_id=abc123; SameSite=Lax
```

This permits the cookie in same-site requests and certain cross-site
top-level navigations using safe methods, typically GET. It generally
withholds the cookie from cross-site POST form submissions and
cross-site subresource requests.

This is a common practical default.

#### `SameSite=None`

``` http
Set-Cookie: session_id=abc123; SameSite=None; Secure
```

This allows the cookie to be sent in cross-site contexts, subject to
browser privacy settings and other restrictions.

Modern browsers require `Secure` when `SameSite=None` is used.

#### Quick comparison

  Attribute   General behavior
  ----------- -------------------------------------------------
  `Strict`    Most restrictive for cross-site sending
  `Lax`       Allows certain cross-site top-level navigations
  `None`      Allows cross-site contexts; requires `Secure`

The exact behavior is governed by browser cookie rules. Do not assume
every browser or privacy configuration behaves identically.

### 4.8 Priority

Some browsers support a `Priority` cookie attribute, such as `Low`,
`Medium`, or `High`, to indicate which cookies are more important when
the browser needs to evict cookies.

``` http
Set-Cookie: preference=dark; Priority=High
```

This is not as widely central to application design as `Secure`,
`HttpOnly`, or `SameSite`, and support can vary.

### 4.9 Partitioned

The `Partitioned` attribute is used with partitioned cookies, associated
with the CHIPS (Cookies Having Independent Partitioned State) mechanism.

``` http
Set-Cookie: widget_state=abc; Secure; SameSite=None; Partitioned
```

A partitioned cookie is stored separately for different top-level sites.
This can allow certain embedded third-party services to maintain state
without using one shared, unpartitioned third-party cookie across every
site.

Browser support and detailed behavior can vary. Use it only when your
application has a real need for partitioned third-party state.

<br>

## 5. Cookie scope: origin, site, domain, and path

These concepts are easy to mix up.

### 5.1 What is an origin?

An origin is the combination of:

1.  Scheme (such as `http` or `https`)
2.  Host
3.  Port

For example:

``` text
https://app.example.com
https://api.example.com
```

These are different origins because their hosts differ.

These are also different origins:

``` text
https://example.com
http://example.com
```

The schemes differ.

And these are different origins:

``` text
https://example.com:443
https://example.com:8443
```

The ports differ.

### 5.2 What is a site?

For modern cookie SameSite behavior, a site is broadly based on the
scheme and registrable domain (the domain boundary determined using the
Public Suffix List).

For example:

``` text
https://app.example.com
https://api.example.com
```

These are different origins, but generally the same site.

By contrast:

``` text
https://app.example.com
https://mybackend.net
```

These are different sites.

Scheme matters too: `http://example.com` and `https://example.com` are
considered different sites for schemeful SameSite behavior.

### 5.3 Origin versus site

```
  -------------------------------------------------------------------------------
  URLs                                     Same origin?                Same site?
  --------------------------- ------------------------- -------------------------
  `https://app.example.com`                         Yes                       Yes
  and                                                   
  `https://app.example.com`                             

  `https://app.example.com`                          No             Generally yes
  and                                                   
  `https://api.example.com`                             

  `https://app.example.com`                          No                        No
  and `https://mybackend.net`                           

  `http://example.com` and                           No                        No
  `https://example.com`                                 
  -------------------------------------------------------------------------------

```

Remember:

-   **Same-Origin Policy (SOP)** is a browser security policy primarily
    concerned with origins.
-   **CORS** controls when browser JavaScript from one origin may access
    responses from another origin.
-   **SameSite** is a cookie policy concerned with sites, not origins.

They are related, but they are not interchangeable.

<br>

## 6. How does the browser decide whether to send a cookie?

The browser evaluates the cookie's rules against the request.

Important factors include:

-   Is the cookie expired?
-   Does the request's host match the cookie's host/domain scope?
-   Does the request path match the cookie's path?
-   Is the request using HTTPS when the cookie is `Secure`?
-   Does the SameSite context permit sending it?
-   Are browser privacy settings or third-party-cookie restrictions
    blocking it?

For example, suppose the browser has:

``` text
Name: refresh_token
Domain: api.example.com (host-only)
Path: /api/auth/refresh
HttpOnly: true
Secure: true
SameSite: Lax
```

A request to:

``` text
https://api.example.com/api/auth/refresh
```

matches the host and path. The cookie can be sent if the other
conditions are satisfied.

A request to:

``` text
https://api.example.com/api/users
```

does not match the cookie path, so the cookie is not sent.

A request to:

``` text
http://api.example.com/api/auth/refresh
```

does not meet the Secure requirement, so the cookie is not sent over
that insecure connection.

<br>

## 7. Cookies versus LocalStorage versus SessionStorage

These are different browser storage mechanisms.


```
  -----------------------------------------------------------------------
  Feature           Cookies           LocalStorage      SessionStorage
  ----------------- ----------------- ----------------- -----------------
  Stores data in    Yes               Yes               Yes
  browser                                               

  Automatically     Yes               No                No
  sent with                                             
  matching HTTP                                         
  requests                                              

  Accessible to     Depends on        Yes               Yes
  JavaScript        `HttpOnly`                          

  Has built-in      Yes               No built-in       Usually tied to
  expiration                          expiry            the tab's page
  controls                                              session

  Typical capacity  Around 4 KB per   Usually several   Usually several
                    cookie (browser   MB                MB
                    limits vary)                        
  -----------------------------------------------------------------------

```

### Cookies

``` http
Set-Cookie: refresh_token=abc123; HttpOnly; Secure
```

The browser automatically attaches the cookie when its rules match.

### LocalStorage

``` javascript
localStorage.setItem("theme", "dark");
const theme = localStorage.getItem("theme");
```

The browser stores the value, but does not automatically send it with
HTTP requests.

### SessionStorage

``` javascript
sessionStorage.setItem("currentTab", "payments");
const tab = sessionStorage.getItem("currentTab");
```

It is scoped to a browsing context, usually a tab, and is cleared when
that page session ends.

### Which should store authentication tokens?

There is no single answer for every architecture.

Common patterns include:

-   Store a refresh token in an `HttpOnly`, `Secure` cookie.
-   Keep a short-lived access token in memory and attach it using an
    `Authorization: Bearer ...` header.
-   Avoid assuming LocalStorage is safe for long-lived secrets:
    JavaScript running through an XSS vulnerability can read it.

The correct design depends on the application's architecture and threat
model.

<br>

## 8. Cookies and authentication

Cookies are often used to maintain a login session.

Imagine a user logs in.

1.  The browser sends the username and password over HTTPS.
2.  The server verifies the credentials.
3.  The server creates a session or issues a token.
4.  The server sends a cookie containing a session identifier or token.
5.  The browser stores the cookie.
6.  The browser attaches it to future matching requests.
7.  The server validates it and identifies the user.

A cookie does not itself prove that a user is legitimate. The server
must validate the session identifier or token it receives.

### 8.1 Session cookie versus persistent cookie

A session cookie generally has no explicit `Expires` or `Max-Age`.

A persistent cookie has an expiration time, for example:

``` http
Set-Cookie: remember_me=abc123; Max-Age=2592000
```

That asks the browser to keep it for 30 days.

The cookie's lifetime and the server-side session/token lifetime should
be designed together. A cookie that remains in the browser does not mean
the server must continue accepting its value.

### 8.2 Refresh token in an HttpOnly cookie

A common architecture is to store a refresh token in an HttpOnly cookie.

Example response:

``` http
HTTP/1.1 200 OK
Set-Cookie: refresh_token=RANDOM_SECRET;
            HttpOnly;
            Secure;
            SameSite=Lax;
            Path=/api/v1/auth/refresh;
            Max-Age=604800
```

In ordinary production headers, this is one `Set-Cookie` header line;
the line breaks above are only for readability.

The frontend can request a new access token without reading the refresh
token:

``` javascript
fetch("/api/v1/auth/refresh", {
    method: "POST",
    credentials: "include"
});
```

The browser attaches the cookie if the request matches its cookie rules.

The backend then validates the refresh token and, if valid, issues a new
access token.

#### What does `credentials: "include"` do?

For Fetch requests, `credentials: "include"` tells the browser to
include credentials such as cookies on cross-origin requests where
cookie rules permit them, and to respect credentialed responses.

For same-origin Fetch requests, the default credentials mode is
`same-origin`. When frontend and backend are on different origins,
`include` is generally needed if the request should carry cookies.

It does not override `SameSite`, `Secure`, domain, path, or browser
privacy restrictions.

<br>

## 9. Cookies and CORS

Suppose your frontend is:

``` text
https://app.example.com
```

And your backend is:

``` text
https://api.example.com
```

They are different origins.

Your frontend sends:

``` javascript
fetch("https://api.example.com/api/auth/refresh", {
    method: "POST",
    credentials: "include"
});
```

The backend must allow the frontend origin and credentials through CORS.

A credentialed CORS response typically includes:

``` http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

Important rules:

-   The allowed origin must be explicit;
    `Access-Control-Allow-Origin: *` cannot be used with credentialed
    CORS responses.
-   The frontend must use the appropriate credentials mode.
-   CORS does not itself prevent all CSRF attacks. It primarily controls
    whether browser JavaScript can access cross-origin responses.
-   A browser may send some cross-origin requests even when JavaScript
    is not allowed to read the response.

If the frontend and backend are on different sites, cookie SameSite
rules also matter. A cross-site cookie generally requires
`SameSite=None; Secure`, and browser third-party-cookie policies may
still restrict it.

<br>

## 10. Cookies and CSRF

CSRF stands for **Cross-Site Request Forgery**.

It is an attack in which a malicious site tricks a user's browser into
sending an unwanted request to another site where the user is
authenticated.

Why can this happen?

Because the browser may automatically attach authentication cookies to
requests.

Imagine:

1.  You log into `bank.example`.
2.  The bank stores a session cookie in your browser.
3.  You visit `evil.example`.
4.  The malicious site causes your browser to submit a request to the
    bank.
5.  If the cookie is sent and the bank does not adequately verify the
    request, the bank may mistake it for an action you intended.

The malicious website does not necessarily need to read or steal the
cookie. It may only need to make the browser send it.

### 10.1 CSRF tokens

A CSRF token is an unpredictable value that the server validates on
requests that need CSRF protection.

A legitimate form might contain:

``` html
<form action="/transfer" method="POST">
    <input type="hidden" name="amount" value="1000">
    <input type="hidden" name="_csrf" value="UNPREDICTABLE_TOKEN">
    <button type="submit">Transfer</button>
</form>
```

The server checks the token before processing the request.

A malicious site generally cannot read a token from another origin
because of browser same-origin restrictions. It therefore cannot
normally include the correct token in a forged request.

CSRF tokens, SameSite cookies, and Origin/Referer validation can be used
as complementary defenses.

### 10.2 Which requests need protection?

Protect operations that change server-side state, such as:

-   Transferring money
-   Changing a password
-   Updating an account
-   Creating or deleting records
-   Refreshing or rotating a token when that endpoint uses cookie
    authentication

Use GET for safe retrieval operations. Do not design GET endpoints that
change important server state.

<br>

## 11. Cookies and XSS

XSS means **Cross-Site Scripting**. It happens when an attacker can
cause malicious JavaScript to execute in a trusted website's context.

Consider:

``` javascript
console.log(document.cookie);
```

If a sensitive cookie is not HttpOnly, malicious JavaScript may be able
to read it.

An attacker may then try to send the value to another server.

HttpOnly helps protect the cookie from direct JavaScript reading.

But remember:

-   HttpOnly does not prevent the malicious script from making
    authenticated requests from the compromised page.
-   SameSite is not a general XSS defense.
-   CSRF tokens are not a general XSS defense.
-   Preventing XSS requires safe output handling, input handling,
    appropriate Content Security Policy, and other secure coding
    practices.

Different defenses address different threats.

<br>

## 12. Deleting a cookie

A server can ask the browser to delete a cookie by setting it with an
expiration in the past or a zero/negative `Max-Age`.

Example:

``` http
Set-Cookie: refresh_token=; Max-Age=0; Path=/api/auth/refresh; HttpOnly; Secure; SameSite=Lax
```

The cookie's scope should match the cookie being deleted, particularly
its Path and Domain settings.

But deleting a browser cookie is not the same as revoking a server-side
token.

For secure logout:

1.  The server should invalidate or revoke the refresh token/session.
2.  The server should send a cookie-clearing response.
3.  The browser removes the cookie.

If the server does not invalidate the token, a copied token might remain
usable until it expires or is otherwise rejected.

<br>

## 13. Cookie security best practices

For authentication cookies, a sensible starting point is:

-   Use HTTPS in production.
-   Set `Secure`.
-   Set `HttpOnly` for secrets that frontend JavaScript does not need to
    read.
-   Choose an appropriate `SameSite` policy.
-   Omit `Domain` unless cross-subdomain sharing is necessary.
-   Use the narrowest practical `Path`, while remembering that Path is
    not a strong security boundary.
-   Set an intentional expiration.
-   Keep server-side token/session expiration and revocation in sync.
-   Protect cookie-authenticated state-changing endpoints against CSRF.
-   Avoid putting passwords or unnecessary personal data in cookies.
-   Do not assume cookie values are encrypted or trustworthy.

### 13.1 The `__Host-` cookie prefix

Browsers support special cookie name prefixes that impose additional
restrictions.

For example:

``` http
Set-Cookie: __Host-session=abc123; Path=/; Secure; HttpOnly; SameSite=Lax
```

A cookie using the `__Host-` prefix must:

-   Be set from a secure origin.
-   Include `Secure`.
-   Use `Path=/`.
-   Omit the `Domain` attribute.

This helps prevent a subdomain from setting a competing domain-scoped
cookie with the same prefixed name.

It is a useful option for security-sensitive cookies when its
requirements fit the application.

<br>

## 14. Common misconceptions

### Misconception 1: Cookies are encrypted.

Not necessarily. A cookie value is not automatically encrypted. HTTPS
protects it in transit, but the value itself may still be readable by
the browser, the server, and anyone with access to it.

### Misconception 2: HttpOnly makes a cookie completely safe.

No. It prevents JavaScript from reading the cookie, but does not stop
CSRF or all consequences of XSS.

### Misconception 3: SameSite=None is always better.

No. It permits cross-site cookie sending. Use it only when the
architecture requires it and apply appropriate CSRF defenses.

### Misconception 4: Deleting a cookie logs the user out everywhere.

No. The server must invalidate the session or refresh token. Clearing
one browser's cookie does not revoke every copy of the token.

### Misconception 5: Cookies belong to a frontend application.

Not exactly. Cookies are scoped according to host/domain, path,
security, and other browser rules. They are not inherently tied to a
particular JavaScript application.

### Misconception 6: CORS prevents CSRF.

No. CORS controls cross-origin access to responses by browser
JavaScript. It is not a complete CSRF defense.

### Misconception 7: Different origins always mean different sites.

No. `app.example.com` and `api.example.com` are different origins but
generally the same site.

<br>

## 15. A complete example: refresh-token cookie

Suppose your application has:

-   Frontend: `https://app.example.com`
-   Backend: `https://api.example.com`
-   Refresh endpoint: `/api/v1/auth/refresh`

The backend responds after login:

``` http
HTTP/1.1 200 OK
Set-Cookie: refresh_token=RANDOM_SECRET; Path=/api/v1/auth/refresh; Max-Age=604800; HttpOnly; Secure; SameSite=Lax
Content-Type: application/json

{
  "accessToken": "SHORT_LIVED_ACCESS_TOKEN"
}
```

The frontend stores the access token in memory and does not need to read
the refresh token.

When the access token expires, the frontend sends:

``` javascript
fetch("https://api.example.com/api/v1/auth/refresh", {
    method: "POST",
    credentials: "include"
});
```

The browser may attach the refresh cookie because the two subdomains are
generally same-site, and the cookie's host/path/security rules match.

The backend:

1.  Reads the refresh cookie.
2.  Validates the token.
3.  Checks its expiration and revocation status.
4.  Applies refresh-token rotation if the system uses it.
5.  Issues a new access token.
6.  Sends a replacement refresh cookie if appropriate.

This is an illustrative design, not a universal configuration. The
correct SameSite policy and CSRF defenses depend on the deployment
architecture.

<br>

# 16. Quick reference

  -----------------------------------------------------------------------
  Attribute               What it controls        Common security use
  ----------------------- ----------------------- -----------------------
  `Expires`               Absolute expiration     Limit cookie lifetime
                          date                    

  `Max-Age`               Lifetime in seconds     Limit cookie lifetime

  `Domain`                Hosts that can receive  Keep scope narrow
                          it                      

  `Path`                  URL paths that receive  Limit where it is sent
                          it                      

  `Secure`                HTTPS-only sending      Protect in transit

  `HttpOnly`              JavaScript access       Reduce direct cookie
                                                  theft through XSS

  `SameSite`              Cross-site sending      Reduce CSRF exposure

  `Priority`              Eviction preference in  Less important for most
                          supporting browsers     apps

  `Partitioned`           Partitioned cookie      Certain embedded
                          storage                 cross-site use cases
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 17. Practice questions

Try answering these without looking back.

1.  What is the difference between `Set-Cookie` and `Cookie`?
2.  What does the browser do after receiving a `Set-Cookie` response?
3.  What is the difference between a host-only cookie and a cookie with
    `Domain=example.com`?
4.  What does `HttpOnly` prevent, and what does it not prevent?
5.  Why should authentication cookies use `Secure`?
6.  What is the difference between `SameSite=Strict`, `Lax`, and `None`?
7.  Why are `app.example.com` and `api.example.com` different origins
    but generally the same site?
8.  Why does CORS not replace CSRF protection?
9.  Why should a server revoke a refresh token instead of only clearing
    its cookie?
10. Why can a malicious website sometimes cause a browser to send a
    cookie without being able to read that cookie?

## Suggested learning path

After understanding cookies, study these topics in order:

1.  HTTP requests and responses
2.  Browser origins and the Same-Origin Policy
3.  CORS
4.  CSRF and CSRF tokens
5.  XSS
6.  Cookie-based authentication
7.  Access tokens, refresh tokens, and token rotation

The goal is not to memorize cookie attributes. It is to understand what
problem each attribute solves, what it does not solve, and how the
browser applies the rules.
