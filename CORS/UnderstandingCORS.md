<br>

# Understanding CORS (Cross-Origin Resource Sharing)

<br>
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


## 3. CORS is a browser security mechanism

<br>

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

<br>
<br>


## 4. A real request

<br>

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

<br>
<br>


# 5. The biggest mental model

<br>

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



