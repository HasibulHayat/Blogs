# Understanding CORS (Cross-Origin Resource Sharing)

<br>

## 1. What problem does CORS solve?

<br>

Imagine you have a frontend:

```text
https://mywebsite.com
```

and a backend API:

```text
https://api.mywebsite.com
```

Your frontend JavaScript wants to call:

```javascript
fetch("https://api.mywebsite.com/users");
```

But the frontend and backend have different **origins**.

Browsers have a security rule around cross-origin requests. CORS is the mechanism that lets a server tell the browser which other origins are allowed to access its resources.

> ### CORS is enforced by browser

So anyone can send a request to a publicly reachable API using Postman, curl, Python, etc. CORS does not stop them.

But they still need to pass your backend security:

So:

- CORS → protects browser-based cross-origin access
- Authentication/authorization → protects the API itself

That's why CORS should never be your API's primary security mechanism.

<br>
<br>


## 2. What is an origin?

<br>

An **origin** is basically:

```text
scheme + domain + port
```

For example:

```text
https://example.com
```

has:

```text
scheme = https
domain = example.com
port = 443
```

HTTPS normally uses port 443.

These are different origins:

```text
https://example.com
https://api.example.com
```

These are also different:

```text
http://example.com
https://example.com
```

And:

```text
https://example.com:443
https://example.com:8080
```

<br>
<br>


## 3. Why does the browser care?

<br>

Suppose you are logged into your bank:

```text
https://mybank.com
```

Now imagine you visit:

```text
https://evil.com
```

That website contains JavaScript that tries to access your bank:

```javascript
fetch("https://mybank.com/account");
```

Without browser protections, malicious websites could potentially make requests to other websites and read sensitive information.

Browsers therefore enforce the **Same-Origin Policy**.

Very roughly:

> A webpage from one origin cannot freely read resources from another origin.

This protection happens in the browser.

---

## 4. How can a frontend call a different backend?

Your backend can explicitly tell the browser:

> "I trust requests from this origin."

That is what CORS allows you to configure.

CORS stands for:

# Cross-Origin Resource Sharing

Break the name apart:

- **Cross-Origin** — the request is coming from another origin.
- **Resource Sharing** — the server says that another origin is allowed to access its resources.

---

## 5. A simple CORS example

Suppose:

```text
Frontend:
https://app.example.com

Backend:
https://api.example.com
```

The frontend makes a request.

The browser may send:

```http
GET /users
Origin: https://app.example.com
```

The backend can respond:

```http
Access-Control-Allow-Origin: https://app.example.com
```

The browser sees that the requested origin is allowed and lets the frontend JavaScript access the response.

Conceptually:

```text
Frontend origin:
https://app.example.com

Server allows:
https://app.example.com

        ↓

       ✅ Allowed
```

---

## 6. What if the server does not allow the origin?

Suppose a request comes from:

```text
https://evil.com
```

but the server only allows:

```text
https://app.example.com
```

The browser compares:

```text
Request came from:
https://evil.com

Server allows:
https://app.example.com
```

They do not match.

The browser prevents the JavaScript code from accessing the response.

You will typically see a **CORS error** in the browser console.

---

## 7. CORS is a browser security mechanism

This is one of the most important things to understand.

CORS is primarily about:

```text
Browser
   ↓
Should JavaScript be allowed to access
this cross-origin response?
```

CORS is NOT:

- Authentication
- Authorization
- A firewall
- A replacement for backend security
- A complete API security system

Think of these concepts separately:

```text
CORS
"Is this browser origin allowed?"

Authentication
"Who are you?"

Authorization
"What are you allowed to do?"
```

---

## 8. A real request

Suppose JavaScript does:

```javascript
fetch("https://api.example.com/users");
```

The browser may send:

```http
GET /users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
```

The important part is:

```http
Origin: https://app.example.com
```

The server responds with:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
```

The browser checks the origins.

If they match:

```text
Origin matches allowed origin
        ↓
       ✅
JavaScript can access the response
```

---

# 9. Understanding a Spring Boot CORS configuration

A typical configuration can look like:

```java
registry.addMapping("/**")
```

This means:

> Apply this CORS configuration to all backend paths.

For example:

```text
/api/users
/api/payments
/api/units
/api/expenses
```

---

### allowedOrigins

Example:

```java
.allowedOrigins(
    "https://app.example.com",
    "https://www.app.example.com"
)
```

This means:

> Allow browser requests coming from these origins.

Conceptually:

```text
https://app.example.com
        ↓
       ✅

https://www.app.example.com
        ↓
       ✅

https://randomwebsite.com
        ↓
       ❌
```

---

## 10. Allowed HTTP methods

Example:

```java
.allowedMethods(
    "GET",
    "POST",
    "PUT",
    "DELETE",
    "OPTIONS"
)
```

This tells the CORS configuration which HTTP methods are allowed.

### GET

Used for retrieving data:

```http
GET /users
```

### POST

Often used for creating data:

```http
POST /users
```

### PUT

Often used for updating data:

```http
PUT /users/123
```

### DELETE

Used for deleting data:

```http
DELETE /users/123
```

### OPTIONS

This one is special.

Browsers can use OPTIONS for a **CORS preflight request**.

---

# 11. What is OPTIONS?

Sometimes the browser does not immediately send the actual request.

Instead, it first asks the server:

> "I want to make this request. Are you okay with it?"

This is called a **preflight request**.

For example, the frontend wants to send:

```http
PUT /users/123
```

with headers such as:

```http
Authorization: Bearer ...
Content-Type: application/json
```

The browser may first send:

```http
OPTIONS /users/123
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: authorization, content-type
```

The server can respond with information such as:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: PUT
Access-Control-Allow-Headers: authorization, content-type
```

The browser checks the response.

If the request is allowed, the browser then sends the actual:

```http
PUT /users/123
```

---

# 12. The preflight flow

Think of it like this:

```text
Frontend JavaScript
        |
        | "I want to PUT /users/123"
        ↓
      Browser
        |
        | "Is this cross-origin request okay?"
        ↓
   OPTIONS request
        |
        ↓
     Backend
        |
        | "Yes, this origin can use PUT"
        ↓
      Browser
        |
        | Actual request
        ↓
   PUT /users/123
        |
        ↓
     Backend
        |
        ↓
     Response
        |
        ↓
      Browser
        |
        ↓
   JavaScript
```

---

# 13. What are request headers?

A request can contain headers.

For example:

```http
Authorization: Bearer abc123
Content-Type: application/json
```

A CORS configuration may specify allowed request headers.

For example:

```java
.allowedHeaders("*")
```

This basically means:

> Allow request headers.

The `*` means "any headers."

You can also explicitly specify headers, for example:

```java
.allowedHeaders(
    "Authorization",
    "Content-Type"
)
```

---

# 14. CORS does NOT decide user permissions

Suppose your API has:

```text
POST /admin/delete-user
```

and CORS allows:

```text
https://myfrontend.com
```

Does that mean every user can delete users?

**No.**

CORS does not decide whether the user has permission.

The backend still needs authentication and authorization.

Conceptually:

```text
Browser
   ↓
CORS
   ↓
"Is this origin allowed?"
   ↓
Authentication
   ↓
"Who is this?"
   ↓
Authorization
   ↓
"Is this person allowed to do this?"
   ↓
Controller
```

---

# 15. CORS vs Authentication vs Authorization vs CSRF

These concepts are different:

| Security concept | Question it answers |
|---|---|
| CORS | Which website origin may access this resource from a browser? |
| Authentication | Who is this user? |
| Authorization | What is this user allowed to do? |
| CSRF | Can another site trick a user's browser into sending an authenticated request? |

CORS and CSRF are related to browser security, but they are **not the same thing**.

---

# 16. The biggest mental model

If you remember only one thing, remember this:

```text
CORS is basically a conversation between
the SERVER and the BROWSER.

Server:
"I allow this origin."

Browser:
"Okay, then JavaScript from that origin
can access the response."
```

It is NOT:

```text
CORS = API authentication
```

It is NOT:

```text
CORS = user permission
```

And it is NOT:

```text
CORS = protection against every kind of attack
```

---

# 17. Final picture

```text
             INTERNET
                 |
                 |
        ┌────────▼─────────┐
        │     BROWSER      │
        │                  │
        │ app.example.com  │
        └────────┬─────────┘
                 |
                 | Cross-origin request
                 |
                 ▼
        ┌──────────────────┐
        │     BACKEND      │
        │                  │
        │ api.example.com  │
        └──────────────────┘
```

The browser sends:

```http
Origin: https://app.example.com
```

The backend says:

```http
Access-Control-Allow-Origin: https://app.example.com
```

The browser:

```text
✅ Allowed
```

If the backend does not allow that origin:

```text
❌ JavaScript cannot access the response
```

---

# 18. What to learn next

After understanding this foundation, the two most useful next topics are:

1. **Simple CORS requests vs preflight CORS requests**
2. **CORS + cookies + CSRF**

The second topic is especially important when designing authentication with **access tokens, refresh tokens, and cookies**.
