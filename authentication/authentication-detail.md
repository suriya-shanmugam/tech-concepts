Great set of topics. I’d teach them in this order so the reader moves from **simple credential sending** to **modern delegated identity**.

## Learning path

### Group 0. Foundations

Before the named mechanisms, the reader should know:

* **Authentication** = proving who you are
* **Authorization** = what you are allowed to do
* **Credential** = secret or proof used for auth
* **Stateful vs stateless**
* **Cookie vs token**
* **Identity provider vs application**

### Group 1. Direct client authentication

1. **Basic Authentication**
2. **Digest Authentication**
3. **API Keys**

### Group 2. Application-managed login

4. **Session-based authentication**

### Group 3. Token-based authentication

5. **Bearer tokens**
6. **JWT**
7. **Access token and Refresh token**

### Group 4. Delegated identity and federation

8. **OAuth 2.0**
9. **OpenID Connect**
10. **SSO**

A small correction that will make the notes much clearer:

* **Basic auth** uses **Base64 encoding**, not encryption.
* **Digest auth** historically uses **MD5 hashing**, but it is old and much less common now.
* **Session auth** is not really “SessionStorage auth.” Usually the browser stores a **session cookie**, while the real session state lives on the **server**.
* **Bearer** and **JWT** are not the same thing. A bearer token is a *usage style*; JWT is one *token format*.

---

# Topic 1 — Basic Authentication

## 1. What it is

**Basic Authentication** is one of the simplest HTTP authentication methods.

The client sends:

* username
* password

combined as:

```text
username:password
```

Then encoded with **Base64** and placed in the HTTP header:

```http
Authorization: Basic <base64(username:password)>
```

Example:

```text
alice:secret123
```

becomes something like:

```http
Authorization: Basic YWxpY2U6c2VjcmV0MTIz
```

## 2. Important idea

**Base64 is not encryption.**
It is only an encoding format.

Anyone who gets the header can easily decode it back to:

```text
alice:secret123
```

So Basic Auth should only be used over **HTTPS**.

---

## 3. How it works

### Request flow

```mermaid
sequenceDiagram
    participant User
    participant Browser/Client
    participant Server

    User->>Browser/Client: Enter username and password
    Browser/Client->>Browser/Client: Combine as username:password
    Browser/Client->>Browser/Client: Base64 encode
    Browser/Client->>Server: GET /resource\nAuthorization: Basic <encoded-value>
    Server->>Server: Decode and verify credentials
    alt Valid credentials
        Server-->>Browser/Client: 200 OK
    else Invalid credentials
        Server-->>Browser/Client: 401 Unauthorized
    end
```

---

## 4. Example HTTP exchange

### Client request

```http
GET /profile HTTP/1.1
Host: example.com
Authorization: Basic YWxpY2U6c2VjcmV0MTIz
```

### Server checks

The server decodes the value:

```text
alice:secret123
```

Then compares it to stored credentials.

---

## 5. Why it is called “basic”

It is called basic because:

* the mechanism is simple
* no login session is created by default
* the username and password may be sent on many requests
* there is no token rotation, claims, or federation

It is basically “send your credentials with the request.”

---

## 6. Where it is used

Basic Auth is still seen in:

* internal tools
* quick prototypes
* simple scripts
* server-to-server calls in controlled environments
* local development
* old APIs and admin panels

It is **not ideal for modern public web apps**.

---

## 7. Pros

* very simple
* built into HTTP
* easy to test with curl or Postman
* easy for small internal tools

## 8. Cons

* sends password repeatedly
* Base64 is easily reversible
* poor user experience for browser apps
* no logout concept by itself
* hard to manage securely at scale
* not suitable for modern delegated login flows

---

## 9. Security notes

Basic Auth is only acceptable when combined with:

* **HTTPS**
* proper password storage on the server
* rate limiting
* brute-force protection
* least privilege access

Without HTTPS, it is unsafe.

Even with HTTPS, modern systems usually prefer:

* session-based auth
* bearer tokens
* OAuth/OIDC

---

## 10. Simple mental model

Think of Basic Auth like this:

> “For every request, the client says:
> ‘Hi, I am alice, and my password is secret123.’”

That is why it is simple, but also why it is risky.

---

## 11. Basic Auth vs form login

They sound similar but are different.

### Basic Auth

* built into HTTP
* credentials sent in `Authorization` header
* often repeated on each request
* no normal application login flow

### Form login

* user submits a login form
* server verifies credentials
* server usually creates a session
* browser stores session cookie

So in modern websites, users usually experience **form login + session/cookie**, not raw Basic Auth.

---

## 12. Small code examples

### curl

```bash
curl -u alice:secret123 https://example.com/profile
```

This automatically sends Basic Auth.

### Raw header example

```bash
curl https://example.com/profile \
  -H "Authorization: Basic YWxpY2U6c2VjcmV0MTIz"
```

---

## 13. Interview-style summary

**What is Basic Authentication?**
A simple HTTP auth scheme where the client sends `username:password` encoded with Base64 in the `Authorization` header.

**Is Base64 secure?**
No. It is encoding, not encryption.

**When is it acceptable?**
Mostly for internal tools, scripts, and controlled environments over HTTPS.

**Why is it less preferred today?**
Because it repeatedly exposes credentials and lacks modern security and UX features.

---

## 14. Final note for this topic

Basic Auth is useful as the **starting point** because it teaches the core idea:

* client proves identity
* server verifies credentials
* request is either accepted or rejected

Almost every later auth system improves on one of its weaknesses:

* don’t send the password every time
* separate login from request authorization
* support expiration and renewal
* let third-party identity providers handle login

---

## Notes structure for the rest

For each next topic, I’ll keep the same pattern:

1. What it is
2. Why it exists
3. How it works
4. Example
5. Diagram
6. Pros and cons
7. Where it fits compared with other methods

Next topic: **Digest Authentication**.

Perfect. Here’s the next note in the same structure.

# Topic 2 — Digest Authentication

## 1. What it is

**Digest Authentication** is an HTTP authentication scheme built on a **challenge-response** model. Instead of sending the password directly on the request, the server first sends a challenge, and the client responds with a computed digest value that proves it knows the password. ([RFC Editor][1])

## 2. Why it exists

Digest Auth was designed as an improvement over Basic Auth. With Basic Auth, the client sends credentials that are only Base64-encoded, which is easy to reverse. With Digest Auth, the password is **not sent in the clear**; instead, the response is derived from values such as the username, password, server nonce, HTTP method, and requested URI. ([RFC Editor][1])

## 3. Important correction

It is common to hear “Digest = MD5,” but that is only partly true. The current Digest spec, RFC 7616, adds **SHA-256** and **SHA-512/256**; **MD5 is still supported only for backward compatibility and is not recommended**. ([RFC Editor][1])

---

## 4. Core idea

The server says, in effect:

> “Do not send me your password. Prove you know it by hashing it together with my challenge.”

That challenge usually includes values such as:

* `realm`
* `nonce`
* `qop`
* `algorithm`
* sometimes `opaque` and `stale` ([RFC Editor][1])

---

## 5. How it works

### Step-by-step flow

1. The client requests a protected resource without auth.
2. The server replies with **401 Unauthorized** and a `WWW-Authenticate: Digest ...` challenge.
3. The client uses the challenge plus its username/password to compute a digest response.
4. The client retries with an `Authorization: Digest ...` header.
5. The server performs the same calculation and compares the result. If they match, access is granted. ([MDN Web Docs][2])

### Mermaid flow

```mermaid
sequenceDiagram
    participant Client
    participant Server

    Client->>Server: GET /account
    Server-->>Client: 401 Unauthorized\nWWW-Authenticate: Digest realm, nonce, qop, algorithm
    Client->>Client: Compute digest using username, password, nonce, method, URI
    Client->>Server: GET /account\nAuthorization: Digest ...
    Server->>Server: Recompute and verify digest
    alt Valid digest
        Server-->>Client: 200 OK
    else Invalid digest
        Server-->>Client: 401 Unauthorized
    end
```

This is the same general HTTP challenge/response framework used by HTTP auth schemes, but with Digest-specific parameters and calculations. ([MDN Web Docs][2])

---

## 6. What the client actually sends

The `Authorization` header for Digest includes named fields rather than one single encoded blob. MDN shows a Digest auth header with fields like `username`, `realm`, `uri`, `algorithm`, `nonce`, `nc`, `cnonce`, `qop`, `response`, and `opaque`. ([MDN Web Docs][3])

Example shape:

```http
Authorization: Digest username="alice",
  realm="admin@example.com",
  uri="/account",
  algorithm=SHA-256,
  nonce="server-generated-nonce",
  nc=00000001,
  cnonce="client-random-value",
  qop=auth,
  response="computed-digest",
  opaque="server-data"
```

The exact digest value is computed from the password and the challenge data, not sent as the raw password itself. ([MDN Web Docs][3])

---

## 7. The formula idea

For the common `qop=auth` case, RFC 7616 defines the response using values often described like this:

```text
A1 = username : realm : password
A2 = METHOD : request-uri
response = KD(H(A1), nonce : nc : cnonce : qop : H(A2))
```

RFC 7616 explicitly defines `A1`, `A2`, and the `response` construction this way for Digest. ([RFC Editor][1])

You do **not** need to memorize the whole formula. What matters is the idea:

* the digest depends on the password
* it also depends on the request and the server challenge
* so the client proves knowledge of the password without directly transmitting it ([RFC Editor][1])

---

## 8. Meaning of the important fields

### `realm`

A label telling the client which protected area or credential scope is being requested. It helps the client know which username/password to use. ([RFC Editor][1])

### `nonce`

A server-generated value used in the challenge. It is opaque to the client and helps defend against replay by making each challenge context-specific or time-limited. ([RFC Editor][1])

### `qop`

“Quality of protection.” MDN lists common values:

* `auth` = authentication
* `auth-int` = authentication plus integrity protection for the message body hash. ([MDN Web Docs][3])

### `cnonce`

A client-generated nonce. It is used together with the server nonce in the response calculation. ([MDN Web Docs][3])

### `nc`

Nonce count. This is a counter for how many times the client has used that nonce. The server can use duplicate `nc` values to detect replay attempts. ([MDN Web Docs][3])

### `opaque`

A server value that the client should send back unchanged. ([RFC Editor][1])

### `stale=true`

If the nonce expired, the server can indicate `stale=true`, which tells the client it may retry with a new challenge rather than asking the user for credentials again. ([RFC Editor][1])

---

## 9. Example request cycle

### First request

```http
GET /account HTTP/1.1
Host: example.com
```

### Server challenge

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Digest realm="admin@example.com",
  nonce="abc123",
  algorithm=SHA-256,
  qop="auth"
```

A protected resource that receives no acceptable `Authorization` header is supposed to return `401 Unauthorized` with `WWW-Authenticate`, and that header can include the Digest challenge parameters. ([RFC Editor][1])

### Client retry

```http
GET /account HTTP/1.1
Host: example.com
Authorization: Digest username="alice",
  realm="admin@example.com",
  uri="/account",
  algorithm=SHA-256,
  nonce="abc123",
  nc=00000001,
  cnonce="xyz789",
  qop=auth,
  response="...",
  opaque="..."
```

That second request proves knowledge of the password by sending the computed `response` value and the rest of the challenge-related parameters. ([MDN Web Docs][3])

---

## 10. Why it is better than Basic

Digest improves on Basic in one important way: **the password itself is not transmitted as a directly reversible header value**. The response digest is tied to the challenge and request details. ([RFC Editor][1])

A useful mental model is:

* **Basic**: “Here is my username and password.”
* **Digest**: “Here is proof that I know the password for this challenge.” ([RFC Editor][1])

---

## 11. But it is still not “modern app auth”

Digest Auth is still part of HTTP auth, but it remains tied to the request/response header challenge model. It does not by itself give you app-style login sessions, token lifecycles, third-party delegated login, or identity federation. Those ideas come later with sessions, bearer tokens, OAuth 2.0, OIDC, and SSO. This is a conceptual comparison, but it follows directly from the fields and flow defined by the HTTP auth framework and Digest scheme. ([RFC Editor][1])

---

## 12. Security strengths

Digest Auth helps with:

* avoiding transmission of the raw password in the request
* challenge-response verification
* some replay resistance through nonce design and nonce count
* optional body-integrity participation with `qop=auth-int` ([RFC Editor][1])

RFC 7616 also discusses limited-use nonces and how nonce construction can reduce replay risk. ([RFC Editor][1])

---

## 13. Security limits

Digest is **not magic security**. Its safety depends heavily on good nonce generation, proper replay protection, and stronger algorithms. RFC 7616 explicitly says MD5 remains only for backward compatibility and is not recommended. It also notes that replay protection depends on how the server generates and validates nonces. ([RFC Editor][1])

Also, Digest does not solve broader application concerns like logout behavior, centralized identity, delegated consent, or mobile/API token refresh patterns. That is why later systems became more common for full applications. This last sentence is an architectural conclusion based on what Digest does and does not define. ([RFC Editor][1])

---

## 14. Digest vs Basic

### Basic

* sends credentials as Base64-encoded `username:password`
* simple but easily reversible if intercepted
* very little built-in protection beyond transport security ([MDN Web Docs][2])

### Digest

* uses a challenge-response flow
* sends a computed digest instead of the raw password
* includes nonce-related replay defenses
* supports stronger algorithms like SHA-256 in RFC 7616 ([RFC Editor][1])

---

## 15. Interview-style summary

**What is Digest Authentication?**
An HTTP challenge-response authentication scheme where the client proves knowledge of a password by sending a calculated digest instead of the raw password. ([RFC Editor][1])

**Does Digest always use MD5?**
No. RFC 7616 adds SHA-256 and SHA-512/256. MD5 is only for backward compatibility and is not recommended. ([RFC Editor][1])

**What is the role of the nonce?**
It is a server challenge value used in the digest calculation and replay defense. ([RFC Editor][1])

**Why is it better than Basic?**
Because the raw password is not sent directly in the request header. ([RFC Editor][1])

**Why don’t modern apps usually stop here?**
Because Digest is still just HTTP request authentication; it does not provide the broader identity, token, and federation capabilities used in modern app ecosystems. This is an architectural inference from the standards’ scope. ([RFC Editor][1])

---

## 16. One-sentence takeaway

**Digest Authentication is a challenge-response upgrade over Basic Auth: it avoids sending the raw password directly, but it is still an older HTTP auth model rather than the full application identity system used by most modern apps.** ([RFC Editor][1])

Next I’d go to **API Keys**, because that’s the first big shift from “user password verification” to “client/app credential identification.”

[1]: https://www.rfc-editor.org/rfc/rfc7616 "RFC 7616: HTTP Digest Access Authentication"
[2]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Authentication "HTTP authentication - HTTP | MDN"
[3]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Authorization "Authorization header - HTTP | MDN"
----
Absolutely.

# Topic 3 — API Keys

## 1. What they are

An **API key** is a secret string that a client sends with an API request so the server can recognize the calling application, project, or integration. In OpenAPI, API keys are commonly sent in a **header**, **query parameter**, or **cookie**. ([Swagger][1])

## 2. Why they exist

API keys are mainly useful when an API provider wants to identify **which app or project** is calling the API, apply **quota or billing**, restrict anonymous use, filter logs, and observe usage patterns. Google Cloud’s docs describe API keys as identifying the calling project, and note they are useful for project identification and project-level authorization checks such as whether the API is enabled for that project. ([Google Cloud Documentation][2])

## 3. The most important idea

API keys are often described as an “authentication” method, but they usually do **not** identify an end user. Current Google Cloud guidance states that API keys identify the **calling project/application**, while authentication tokens identify the **user**; it also notes that a standard API key does not authenticate a principal, so it cannot by itself support IAM-style authorization checks on behalf of a user. ([Google Cloud Documentation][2])

So the clean mental model is:

* **Basic / Digest** → “Who is the user?”
* **API key** → “Which app or project is calling me?” ([Google Cloud Documentation][2])

---

## 4. How it works

The client receives an API key from the API provider, stores it, and sends it with each request. The server reads the key, looks it up, checks whether it is valid, and then may apply rules such as quota limits, environment restrictions, or API enablement checks before serving the request. Swagger’s OpenAPI docs show the key being sent in the request, and Google Cloud explains that the key lets the provider associate the request with a project for quota and usage purposes. ([Swagger][1])

### Mermaid flow

```mermaid id="67y9k0"
sequenceDiagram
    participant Client App
    participant API Server
    participant Key Store / Policy

    Client App->>API Server: GET /weather\nX-API-Key: abc123
    API Server->>Key Store / Policy: Validate key
    Key Store / Policy-->>API Server: Valid, project=mobile-app, quota OK
    alt Allowed
        API Server-->>Client App: 200 OK + response
    else Invalid or blocked
        API Server-->>Client App: 401/403/429
    end
```

This diagram reflects the standard pattern: key attached to the request, server lookup, then policy checks such as validity and quota. ([Swagger][1])

---

## 5. Common ways to send an API key

Swagger documents three common locations:

### In a request header

```http
GET /data HTTP/1.1
Host: api.example.com
X-API-Key: abcdef12345
```

### In a query parameter

```http
GET /data?api_key=abcdef12345 HTTP/1.1
Host: api.example.com
```

### In a cookie

```http
GET /data HTTP/1.1
Host: api.example.com
Cookie: X-API-KEY=abcdef12345
```

These are all standard documented patterns in OpenAPI’s API key scheme. ([Swagger][1])

---

## 6. Best practice on placement

Although query parameters are supported, they are usually the weaker choice because URLs are more likely to be logged, copied, cached, or exposed in browser history and monitoring systems. Google Cloud explicitly warns that putting an API key in the URL can expose it to theft through URL scans. A header like `X-API-Key` is usually the cleaner and safer default. ([Google Cloud Documentation][3])

---

## 7. Example request

```bash
curl https://api.example.com/data \
  -H "X-API-Key: abcdef12345"
```

This matches the common header-based style shown in Swagger’s API key documentation. ([Swagger][1])

---

## 8. What API keys are good for

API keys are good when you want to:

* identify the calling app or project
* block anonymous traffic
* apply rate limits or quota
* measure usage per integration
* restrict usage to certain apps, IPs, or environments
* give simple access to low-risk or public-ish API capabilities ([Google Cloud Documentation][2])

Google Cloud specifically mentions identifying application traffic, controlling number of calls, filtering logs by key, and restricting key usage to environments such as IP ranges or mobile apps. ([Google Cloud Documentation][2])

---

## 9. What API keys are bad at

API keys are a poor fit when you need to know:

* **which user** is making the request
* what permissions that specific user has
* token expiry and renewal
* delegated login
* SSO
* fine-grained identity and consent flows ([Google Cloud Documentation][2])

Google Cloud explicitly says API keys cannot be used for identifying individual users or for secure authorization, and that standard API keys do not identify a principal. ([Google Cloud Documentation][2])

---

## 10. Security reality

API keys should be treated as **secrets**, and they should be used only over **HTTPS**. OWASP states secure REST services should only provide HTTPS endpoints because credentials in transit, including API keys, need protection. Swagger also notes API key-based security should be combined with HTTPS/SSL. ([OWASP Cheat Sheet Series][4])

But even with HTTPS, API keys are still limited. Google Cloud says API keys are generally not considered secure because they are often accessible to clients, easy to steal, and usually have **no built-in expiration** unless the owner revokes or regenerates them. ([Google Cloud Documentation][2])

That is the key weakness:

> an API key is often a long-lived shared secret.
> If it leaks, someone else can often use it until it is revoked. ([Google Cloud Documentation][2])

---

## 11. Good practices

Clear notes for best practice:

1. **Use HTTPS only.** Credentials like API keys should never travel over plain HTTP. ([OWASP Cheat Sheet Series][4])
2. **Prefer headers over query parameters.** URLs are easier to leak into logs and monitoring. ([Google Cloud Documentation][3])
3. **Restrict the key.** Limit by IP range, app, platform, or API scope when the platform supports it. ([Google Cloud Documentation][2])
4. **Rotate and revoke keys.** Because many API keys are long-lived, rotation matters when exposure is suspected. This follows directly from Google Cloud’s warning that stolen keys may be usable indefinitely until revoked or regenerated. ([Google Cloud Documentation][2])
5. **Do not hard-code keys in public frontend code unless the API is explicitly designed for restricted public-client keys.** API keys present in client code are easier to extract; Google Cloud notes that API keys are typically accessible to clients. ([Google Cloud Documentation][2])
6. **Do not use API keys as your main user identity system.** Use real user authentication and access tokens when you need user-level security. ([Google Cloud Documentation][2])

---

## 12. API key vs Basic vs Digest

### Basic Auth

The client sends user credentials directly, Base64-encoded, on requests. It is about verifying a username/password pair. ([MDN Web Docs][5])

### Digest Auth

The client proves knowledge of a password using a challenge-response calculation rather than sending the raw password directly. It is still user-credential-oriented HTTP auth. ([OWASP Cheat Sheet Series][4])

### API Key

The client sends a key that identifies the calling app/project, usually for quota, integration control, or simple access management, not end-user identity. ([Google Cloud Documentation][2])

So the progression is:

* **Basic**: send user secret directly
* **Digest**: prove knowledge of user secret
* **API key**: identify the calling application/integration ([Google Cloud Documentation][2])

---

## 13. Real-world examples

Typical uses include:

* weather or map APIs for quota tracking
* internal service integrations with low-risk access
* partner APIs where each partner gets a key
* public APIs that want to block anonymous traffic and measure usage per client ([Google Cloud Documentation][2])

These examples are directly consistent with Google Cloud’s documented use cases: identifying application traffic, controlling call volume, and tracking usage patterns. ([Google Cloud Documentation][2])

---

## 14. Where API keys fit in the bigger auth story

API keys are the first major step away from “user password on every request.” But they are **not** yet modern user identity. They answer:

> “Which application is calling?” ([Google Cloud Documentation][2])

They do **not** fully answer:

> “Which user is this, what can they do, when does the credential expire, and can another identity provider log them in?” ([Google Cloud Documentation][2])

Those questions are why later topics exist:

* sessions
* bearer tokens
* JWT
* access/refresh tokens
* OAuth 2.0
* OIDC
* SSO ([OWASP Cheat Sheet Series][4])

---

## 15. Interview-style summary

**What is an API key?**
A secret token sent by a client so the API can identify the calling application or project. ([Swagger][1])

**Does an API key identify a user?**
Usually no. Standard API keys generally identify the app/project, not the end user, and may not authenticate a principal at all. ([Google Cloud Documentation][2])

**Where is it sent?**
Usually in a header such as `X-API-Key`, but it can also be sent in a query parameter or cookie. ([Swagger][1])

**Why is it risky?**
Because it is often long-lived, client-accessible, and usable until revoked if stolen. ([Google Cloud Documentation][2])

**When is it okay to use?**
When project-level identification, quota, traffic control, or simple integration security is enough. ([Google Cloud Documentation][2])

---

## 16. One-sentence takeaway

**API keys are best understood as app/project identifiers and shared secrets for API access control, not as a full user authentication system.** ([Google Cloud Documentation][2])

Next should be **Session-based authentication**, because that is the first topic where the server manages logged-in user state.

[1]: https://swagger.io/docs/specification/v3_0/authentication/api-keys/ "API Keys | Swagger Docs"
[2]: https://docs.cloud.google.com/endpoints/docs/openapi/when-why-api-key "Why and when to use API keys  |  Cloud Endpoints with OpenAPI  |  Google Cloud Documentation"
[3]: https://docs.cloud.google.com/docs/authentication/rest?utm_source=chatgpt.com "Authenticate with REST | Authentication - Google Cloud"
[4]: https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html "REST Security - OWASP Cheat Sheet Series"
[5]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Authorization?utm_source=chatgpt.com "Authorization header - HTTP | MDN - MDN Web Docs"
------
Absolutely — this is one of the most important topics.

# Topic 4 — Session-based Authentication

## 1. What it is

**Session-based authentication** is a **stateful** authentication model where the server keeps track of the logged-in user across multiple HTTP requests by storing session state on the server and giving the client a **session identifier** to send back on later requests. MDN describes centralized session management this way: after authentication, the server records the user’s state, generates a session ID associated with that state, returns a copy of the ID to the client, and the client sends that ID back with future requests. ([MDN Web Docs][1])

## 2. The first important correction

Session-based auth is **not the same thing as browser `sessionStorage`**. `sessionStorage` is part of the Web Storage API and stores key/value data in the browser for the duration of a page session; it is partitioned by origin and browser tab. Traditional session-based authentication usually relies on a **session cookie** carrying a session ID, while the actual authenticated session state lives on the **server**. ([MDN Web Docs][2])

So the clean mental model is:

* **Session auth** = server remembers the logged-in user
* **Cookie** = browser sends the session ID back
* **`sessionStorage`** = browser-side storage API, not the core of classic session auth ([MDN Web Docs][1])

---

## 3. Why it exists

HTTP is stateless by default, which means one request does not automatically remember what happened in the previous request. Cookies let web applications remember state information, and session management provides a way to associate a sequence of HTTP requests with the same authenticated user. ([MDN Web Docs][3])

This solves a practical problem:

> the user should not need to type their username and password on every request.

Instead, they log in once, and the server remembers them through the session. MDN’s authentication guidance notes that websites commonly keep users signed in by setting a cookie containing a secret session identifier. ([MDN Web Docs][4])

---

## 4. Core idea

The login step and the later requests are separated:

1. User submits credentials once
2. Server verifies them
3. Server creates a session record
4. Server sends a session ID to the browser, usually in a cookie
5. Browser automatically sends the cookie on later requests
6. Server looks up the session ID and recognizes the user ([MDN Web Docs][1])

That is why this model is called **stateful**: the server keeps state about the user between requests. ([MDN Web Docs][1])

---

## 5. How it works

### Mermaid flow

```mermaid id="xr5t9h"
sequenceDiagram
    participant User
    participant Browser
    participant Server
    participant Session Store

    User->>Browser: Enter username and password
    Browser->>Server: POST /login
    Server->>Server: Verify credentials
    Server->>Session Store: Create session for user
    Session Store-->>Server: session_id=abc123
    Server-->>Browser: Set-Cookie: session_id=abc123
    Browser->>Server: GET /profile + Cookie: session_id=abc123
    Server->>Session Store: Look up session_id
    Session Store-->>Server: user=alice, authenticated=true
    Server-->>Browser: 200 OK + protected data
```

This reflects the centralized session model MDN describes: server-side session state plus a client-held session ID that is returned on later requests. ([MDN Web Docs][1])

---

## 6. Example HTTP flow

### Login request

```http id="pdj0v9"
POST /login HTTP/1.1
Host: example.com
Content-Type: application/json

{"username":"alice","password":"secret123"}
```

### Server response

```http id="2b9l8z"
HTTP/1.1 200 OK
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax
```

Cookies are sent by the server and then returned by the browser on later requests, and MDN’s secure cookie guidance recommends attributes such as `Secure`, `HttpOnly`, and `SameSite` to reduce common risks. ([MDN Web Docs][3])

### Later authenticated request

```http id="cu3g1m"
GET /profile HTTP/1.1
Host: example.com
Cookie: session_id=abc123
```

The server then uses the session ID to find the session state and determine who the user is. ([MDN Web Docs][1])

---

## 7. Where the real data lives

In classic session auth, the browser usually stores only the **session identifier**, while the server stores the actual state, such as:

* user ID
* login time
* role/permission context
* session expiration
* CSRF-related state
* other login metadata ([MDN Web Docs][1])

This is why losing the session cookie can be dangerous: if an attacker gets a valid session cookie, they may be able to hijack the session for as long as it remains valid. OWASP’s cookie theft guidance explicitly warns that stealing a valid session cookie can allow session hijacking for the session’s lifetime. ([OWASP Cheat Sheet Series][5])

---

## 8. Why cookies are commonly used

Browsers automatically send matching cookies back to the server on future requests, which makes cookies a natural mechanism for maintaining login state in traditional web applications. MDN explains that cookies let browsers store and return state information, and that websites typically keep users signed in with a cookie containing a secret session identifier. ([MDN Web Docs][3])

This is the usual browser experience:

* log in once
* visit many pages
* stay signed in
* log out later ([MDN Web Docs][4])

---

## 9. Why `sessionStorage` is not the same thing

`sessionStorage` only stores data in the browser for that tab’s page session, and it is a Web Storage feature, not a server-managed authentication protocol. MDN says `sessionStorage` is partitioned by origin and top-level browsing context and lasts for the duration of the page session. ([MDN Web Docs][2])

You *can* technically put tokens or IDs in `sessionStorage`, but that is a different architecture. Classic session-based auth usually means:

* server-managed session
* session ID in cookie
* browser automatically sends the cookie
* server looks up the user state ([MDN Web Docs][1])

So for your notes, I would rename the topic from:

> “Session - SessionStorage (Stateful)”

to:

> **Session-based authentication (stateful, usually cookie + server session)** ([MDN Web Docs][1])

---

## 10. Security strengths

Session-based auth has several strengths for traditional web apps:

* the password is not sent on every request after login
* the server has centralized control over session validity
* logout can be enforced by deleting or invalidating the server-side session
* permissions and session state can be changed centrally without reissuing a client token ([MDN Web Docs][1])

That central control is one reason server-managed sessions are still common in classic web applications. MDN specifically describes this centralized session management pattern. ([MDN Web Docs][1])

---

## 11. Security risks

The main risks include:

* **session hijacking** if a session cookie is stolen
* **session fixation** if a session ID is not regenerated appropriately
* **CSRF** because browsers may automatically send cookies
* weak cookie settings
* sessions that live too long or are not invalidated properly ([OWASP Cheat Sheet Series][6])

OWASP’s session management guidance covers session ID life cycle, expiration, and abuse defenses, while MDN’s secure cookie guidance highlights the importance of cookie flags like `Secure`, `HttpOnly`, and `SameSite`. ([OWASP Cheat Sheet Series][6])

---

## 12. Cookie security flags you should know

### `Secure`

The cookie is only sent over HTTPS. MDN recommends using `Secure` for all cookies, especially sensitive ones. ([MDN Web Docs][7])

### `HttpOnly`

Prevents JavaScript from accessing the cookie via `document.cookie`, which helps reduce the impact of some XSS attacks on cookie theft. MDN recommends `HttpOnly` for cookies that do not need JavaScript access, including session identifiers. ([MDN Web Docs][7])

### `SameSite`

Helps control when cookies are sent with cross-site requests, reducing CSRF risk. MDN recommends `SameSite=Strict` where possible or `Lax` as a practical choice for many cases. ([MDN Web Docs][7])

These flags do not solve every problem, but they significantly improve the safety of cookie-based sessions. ([MDN Web Docs][7])

---

## 13. Session expiration and logout

Because the server owns the session state, it can expire sessions after inactivity or after an absolute lifetime, and it can invalidate them on logout. OWASP’s session guidance explicitly includes timeout and session termination practices. ([OWASP Cheat Sheet Series][6])

This gives session auth a practical advantage over some purely self-contained client-side tokens:

> the server can centrally decide that a session is no longer valid. ([OWASP Cheat Sheet Series][6])

---

## 14. Session rotation after login

A strong implementation regenerates or rotates the session identifier after authentication and other privilege changes to reduce session fixation risk. OWASP’s Session Management Cheat Sheet covers session ID renewal and fixation defenses as part of session security. ([OWASP Cheat Sheet Series][6])

This is an important interview point:

> do not keep using the exact same pre-login session ID after the user becomes authenticated. ([OWASP Cheat Sheet Series][6])

---

## 15. Session-based auth vs API keys

### API keys

Identify the calling app or project, usually for quota, integration, or simple access control. They are not mainly for browser user login. This is the conceptual distinction we established in the previous topic.

### Session auth

Identifies a logged-in user across multiple browser requests by storing session state on the server and using a client-held session identifier, usually via a cookie. ([MDN Web Docs][1])

So:

* **API key** = “which application is calling?”
* **Session** = “this browser is already logged in as this user.” ([MDN Web Docs][1])

---

## 16. Session-based auth vs Basic Auth

### Basic Auth

The client sends credentials on requests using the `Authorization` header.

### Session-based auth

The client sends credentials once during login, then uses a session cookie afterward while the server keeps the authenticated state. MDN notes that websites typically keep users signed in by using a cookie containing a secret session identifier. ([MDN Web Docs][4])

That is a big usability improvement:

* no password on every request
* easier persistent login experience
* centralized session control ([MDN Web Docs][4])

---

## 17. Where it fits in the bigger learning path

Session-based authentication is the first major model where the application says:

> “You authenticated once. I’ll remember you.” ([MDN Web Docs][1])

It is especially natural for:

* server-rendered web apps
* browser-based login flows
* traditional monolithic web applications
* apps where the server wants full control over session lifetime and logout ([MDN Web Docs][1])

Later topics—bearer tokens, JWT, access/refresh tokens, OAuth 2.0, and OIDC—become more useful when systems are more distributed, API-driven, mobile-heavy, or federated. This is an architectural comparison built from the centralized, server-held nature of session auth documented by MDN and OWASP. ([MDN Web Docs][1])

---

## 18. Interview-style summary

**What is session-based authentication?**
A stateful authentication model where the server stores session state for the logged-in user and the client sends back a session ID, usually in a cookie. ([MDN Web Docs][1])

**Why is it called stateful?**
Because the server remembers the user’s authenticated state across requests. ([MDN Web Docs][1])

**Is `sessionStorage` the same thing as session auth?**
No. `sessionStorage` is browser-side Web Storage for the duration of a page session; classic session auth is typically server-side session state plus a cookie-based session ID. ([MDN Web Docs][2])

**Why are cookies commonly used?**
Because the browser can automatically send them back on later requests, making them convenient for preserving login state. ([MDN Web Docs][3])

**What are the main security controls?**
Use HTTPS, strong session IDs, cookie flags like `Secure`, `HttpOnly`, and `SameSite`, session expiration, and session ID rotation. ([MDN Web Docs][7])

---

## 19. One-sentence takeaway

**Session-based authentication is a stateful model where the server remembers the logged-in user and the browser usually carries only a session ID—most often in a secure cookie.** ([MDN Web Docs][1])

Next, the cleanest move is **Bearer tokens and JWT**, because that is where auth becomes more stateless and API-friendly.

[1]: https://developer.mozilla.org/en-US/docs/Web/Security/Authentication/Session_management?utm_source=chatgpt.com "Session management - Security - MDN"
[2]: https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage?lang=en-US&utm_source=chatgpt.com "Window: sessionStorage property - Web APIs | MDN"
[3]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies?utm_source=chatgpt.com "Using HTTP cookies - MDN Web Docs"
[4]: https://developer.mozilla.org/en-US/docs/Web/Security/Authentication?utm_source=chatgpt.com "Authentication - Security | MDN"
[5]: https://cheatsheetseries.owasp.org/cheatsheets/Cookie_Theft_Mitigation_Cheat_Sheet.html?utm_source=chatgpt.com "Cookie Theft Mitigation - OWASP Cheat Sheet Series"
[6]: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html?utm_source=chatgpt.com "Session Management - OWASP Cheat Sheet Series"
[7]: https://developer.mozilla.org/en-US/docs/Web/Security/Practical_implementation_guides/Cookies?utm_source=chatgpt.com "Secure cookie configuration - Security | MDN - MDN Web Docs"
------
Absolutely — this is the point where auth starts to look like modern API security.

# Topic 5 — Bearer Tokens and JWT

## 1. Start with the key distinction

**Bearer token** and **JWT** are not the same thing.

* A **bearer token** is a **usage model**: whoever possesses the token can use it. RFC 6750 defines a bearer token as a security token where any party in possession of it can use it, without proving possession of a cryptographic key. ([RFC Editor][1])
* A **JWT** is a **token format**: a compact, URL-safe JSON-based structure for carrying claims, which can be signed, MACed, and/or encrypted. RFC 7519 defines JWT as a string representing claims encoded in JWS or JWE form. ([RFC Editor][2])

A clean mental model is:

> **Bearer** = how the token is used
> **JWT** = what the token may look like ([RFC Editor][1])

So:

* a bearer token **can** be a JWT
* a bearer token does **not have to be** a JWT
* a JWT does **not have to be used as a bearer token** ([RFC Editor][1])

---

## 2. What a bearer token is

A bearer token is usually sent in the HTTP `Authorization` header using the `Bearer` scheme:

```http id="0w32m3"
Authorization: Bearer <token>
```

RFC 6750 says clients should send bearer tokens in the `Authorization` request header with the `Bearer` scheme, and resource servers must support that method. ([RFC Editor][1])

The most important security property is also the biggest risk:

> if someone steals the token, they can usually use it. ([RFC Editor][1])

That is why RFC 6750 requires TLS and says bearer tokens must be protected in transport and storage. ([RFC Editor][1])

---

## 3. What a JWT is

A JWT is a compact token format for carrying **claims**. RFC 7519 says JWTs are intended for space-constrained environments such as HTTP `Authorization` headers and URI query parameters, and that they encode claims as a JSON object. ([RFC Editor][2])

A **claim** is just a piece of information asserted about a subject, represented as a name/value pair. ([RFC Editor][2])

Common registered claims include:

* `iss` = issuer
* `sub` = subject
* `aud` = audience
* `exp` = expiration time
* `nbf` = not before
* `iat` = issued at
* `jti` = JWT ID ([RFC Editor][2])

---

## 4. The common confusion

People often say:

> “We use JWT authentication.”

That is incomplete.

What they usually mean is:

> “We issue a token, often a JWT, and the client sends it as a Bearer token.” ([RFC Editor][1])

So the real stack is often:

1. authentication happens
2. server issues an access token
3. client sends that token as `Authorization: Bearer ...`
4. the token’s format may be JWT or something opaque ([RFC Editor][1])

---

## 5. How bearer-token auth works

In the normal HTTP pattern, the client calls a protected API with a bearer token. If the token is missing or unacceptable, the server can return `401 Unauthorized` with `WWW-Authenticate: Bearer`. MDN documents this exact pattern, and RFC 6750 defines the `WWW-Authenticate` response header field for bearer-protected resources. ([MDN Web Docs][3])

### Mermaid flow

```mermaid id="jlwmvx"
sequenceDiagram
    participant User
    participant Client App
    participant Auth Server
    participant API Server

    User->>Client App: Sign in
    Client App->>Auth Server: Send credentials / login request
    Auth Server-->>Client App: Access token (often a JWT)
    Client App->>API Server: GET /profile + Authorization: Bearer <token>
    API Server->>API Server: Validate token
    alt Token valid
        API Server-->>Client App: 200 OK + protected data
    else Token invalid or missing
        API Server-->>Client App: 401 Unauthorized + WWW-Authenticate: Bearer
    end
```

This is the standard bearer-token flow shape: token acquisition first, then token presentation to the resource server in the `Authorization` header. ([RFC Editor][1])

---

## 6. A simple bearer token example

### Client request

```http id="yk7c6k"
GET /api/profile HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOi...
```

That header format is directly defined by RFC 6750. ([RFC Editor][1])

### If unauthenticated

```http id="qv4gbn"
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer
```

MDN shows this exact response pattern for a protected API expecting bearer auth. ([MDN Web Docs][3])

---

## 7. Where bearer tokens can be sent

RFC 6750 allows three transport patterns:

* `Authorization` header
* form-encoded body parameter
* URI query parameter ([RFC Editor][1])

But the spec clearly prefers the `Authorization` header. It says clients should use the header method, resource servers must support it, and the URI query method is not recommended because URLs with access tokens are likely to be logged. ([RFC Editor][1])

So for your notes, the best rule is:

> **Use `Authorization: Bearer <token>`**
> Avoid query parameters unless you are forced into a legacy edge case. ([RFC Editor][1])

---

## 8. What a JWT looks like

RFC 7519 says a JWT is a sequence of URL-safe parts separated by periods (`.`), with each part containing a base64url-encoded value. In the common signed JWT example, you can see three dot-separated parts. ([RFC Editor][2])

A typical signed JWT looks like:

```text id="crp8ax"
xxxxx.yyyyy.zzzzz
```

People often teach this as:

```text id="6cxn9f"
header.payload.signature
```

That is a useful teaching shorthand for the common signed JWS form shown in RFC 7519’s example. ([RFC Editor][2])

---

## 9. What the parts mean

### Header

The header describes cryptographic metadata such as token type and algorithm. RFC 7519’s example header contains values like `"typ":"JWT"` and `"alg":"HS256"`. ([RFC Editor][2])

### Payload

The payload contains the claims, such as issuer, subject, audience, or expiration time. RFC 7519 defines the JWT Claims Set as a JSON object containing the conveyed claims. ([RFC Editor][2])

### Signature

In the signed JWS form, the signature or MAC protects integrity, so the receiver can detect tampering if validation is done correctly. RFC 7519 describes JWTs as encoded in JWS or JWE, where JWS signs or MACs the claims and JWE encrypts them. ([RFC Editor][2])

---

## 10. Very important security note about JWTs

A **signed JWT is not the same as an encrypted JWT**.

RFC 7519 says that if the JOSE header is for a **JWS**, the claims are digitally signed or MACed; if it is for a **JWE**, the claims are encrypted. That means a signed JWT gives integrity protection, but not confidentiality by itself. ([RFC Editor][2])

So the safe mental rule is:

> **Do not put sensitive secrets into JWT payloads just because they are encoded.**
> Encoding is not encryption, and signing is not encryption. ([RFC Editor][2])

---

## 11. Important claims to know

### `iss` — issuer

Identifies the principal that issued the JWT. ([RFC Editor][2])

### `sub` — subject

Identifies the principal that is the subject of the JWT. ([RFC Editor][2])

### `aud` — audience

Identifies the intended recipients. If a processor does not identify itself in the audience when `aud` is present, the JWT must be rejected. ([RFC Editor][2])

### `exp` — expiration time

Identifies the time on or after which the JWT must not be accepted. ([RFC Editor][2])

These claims matter because token validation is not just “does the signature verify?” It is also “is this token meant for me, issued by someone I trust, and still valid now?” ([RFC Editor][2])

---

## 12. What the server must validate

JWT best practices are stricter than many beginner tutorials suggest. RFC 8725 says:

* all cryptographic operations in the JWT must be validated
* the entire JWT must be rejected if any validation step fails
* audience must be used and validated where relevant
* issuer and subject must be validated appropriately
* claims must not be blindly trusted ([RFC Editor][4])

That means a secure JWT implementation does **not** just decode the payload and trust it. It validates the cryptography and then validates the claims against application rules. ([RFC Editor][4])

---

## 13. Why bearer tokens are popular

Bearer tokens fit modern API-driven systems well because they separate:

* the act of logging in
* from the act of calling APIs later with a token ([RFC Editor][1])

Compared with server sessions, bearer-token requests are easier to send across mobile apps, SPAs, backend services, and distributed APIs because the request carries its own authorization credential in the header. That last sentence is an architectural inference from the HTTP bearer model defined in RFC 6750. ([RFC Editor][1])

---

## 14. Why JWTs are popular

JWTs are popular because they are compact, URL-safe, easy to carry in headers, and can bundle claims in one token. RFC 8725 notes that JWTs have seen rapid adoption because they encapsulate security-relevant information in one easy-to-protect location and are easy to implement using widely available tools. ([RFC Editor][4])

That is why many systems choose:

* bearer token as the transport style
* JWT as the token format ([RFC Editor][1])

---

## 15. Risks and trade-offs

### Bearer-token risk

Because bearer tokens are possession-based, theft is the main danger. Anyone holding the token may be able to use it. RFC 6750 explicitly defines bearer tokens this way and requires protection in transport and storage. ([RFC Editor][1])

### JWT risk

JWTs are easy to misuse if developers:

* skip signature verification
* fail to validate `aud`, `iss`, or `exp`
* use weak keys
* trust received claims blindly ([RFC Editor][4])

RFC 8725 exists specifically because JWT deployments had widely published implementation and deployment mistakes. ([RFC Editor][4])

---

## 16. Bearer token vs session cookie

### Session cookie

The browser usually sends a cookie, and the server looks up session state it stored earlier. That is stateful. This follows the centralized session model MDN describes. ([MDN Web Docs][5])

### Bearer token

The client explicitly sends a token on each protected request, usually in `Authorization: Bearer ...`. That is the stateless-looking request style RFC 6750 defines. ([RFC Editor][1])

The practical difference is:

* **session**: “server remembers me”
* **bearer token**: “I present proof of access on each request” ([RFC Editor][1])

---

## 17. Bearer token vs API key

They can look similar because both may be sent in headers, but they usually serve different roles.

* **API key** usually identifies the calling app or project.
* **Bearer token** usually represents granted access and is treated as an authorization credential for protected resources. RFC 6750 is specifically about using bearer tokens to access protected resources. ([RFC Editor][1])

So the mental shift is:

* **API key** = “which client/app is this?”
* **Bearer token** = “this request carries access rights” ([RFC Editor][1])

---

## 18. One example JWT payload

A JWT payload might look like this:

```json id="fwz0bl"
{
  "iss": "https://auth.example.com",
  "sub": "user_123",
  "aud": "api.example.com",
  "exp": 1717000000,
  "iat": 1716996400
}
```

Those claim names are all registered in RFC 7519, though applications decide which are required in their own systems. ([RFC Editor][2])

---

## 19. Interview-style summary

**What is a bearer token?**
A token that grants access based on possession: whoever has it can usually use it. It is commonly sent as `Authorization: Bearer <token>`. ([RFC Editor][1])

**What is a JWT?**
A compact, URL-safe token format that carries claims as JSON and can be signed and/or encrypted. ([RFC Editor][2])

**Are bearer tokens and JWT the same?**
No. Bearer is a usage model; JWT is a format. A bearer token may be a JWT, but it does not have to be. ([RFC Editor][1])

**Is a JWT encrypted?**
Not necessarily. Signed JWS JWTs provide integrity protection, while JWE JWTs provide encryption. ([RFC Editor][2])

**What must be validated in a JWT?**
At minimum, the cryptographic validation and the relevant claims such as issuer, audience, and expiration. RFC 8725 emphasizes validating all cryptographic operations and not blindly trusting claims. ([RFC Editor][4])

---

## 20. One-sentence takeaway

**Bearer tokens define how access is presented; JWT defines one common way that token can be packaged.** ([RFC Editor][1])

Next should be **Access tokens and Refresh tokens**, because that naturally answers the next question:
“Why not just keep one bearer token forever?”

[1]: https://www.rfc-editor.org/rfc/rfc6750 "RFC 6750: The OAuth 2.0 Authorization Framework: Bearer Token Usage"
[2]: https://www.rfc-editor.org/rfc/rfc7519 "RFC 7519: JSON Web Token (JWT)"
[3]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/401 "401 Unauthorized - HTTP | MDN"
[4]: https://www.rfc-editor.org/rfc/rfc8725 "RFC 8725: JSON Web Token Best Current Practices"
[5]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Authorization "Authorization header - HTTP | MDN"
-----
Absolutely — this topic explains **why modern systems usually use two tokens instead of one**.

# Topic 6 — Access Tokens and Refresh Tokens

## 1. What they are

An **access token** is the credential a client uses to call a protected API or resource server. OAuth 2.0 defines it as a string representing authorization issued to the client, with a specific scope, lifetime, and other access attributes. ([RFC Editor][1])

A **refresh token** is a separate credential used to get a **new access token** from the authorization server when the current access token expires or becomes invalid. OAuth 2.0 also makes an important boundary clear: refresh tokens are for the **authorization server only** and are **never sent to resource servers**. ([RFC Editor][1])

So the clean model is:

* **Access token** → goes to the API
* **Refresh token** → goes to the authorization server
* **Do not swap those roles**. ([RFC Editor][1])

---

## 2. Why both exist

The main reason for using both is to balance **security** and **user experience**. OAuth’s current security guidance says refresh tokens are convenient and user-friendly because they let the authorization server issue access tokens with a **short lifetime** and possibly **reduced scope**, which reduces the impact of access-token leakage. ([IETF Datatracker][2])

So instead of giving one long-lived token to the client and hoping it never leaks, the system commonly does this:

* keep **access tokens short-lived**
* let the client use a **refresh token** to quietly obtain a new one
* keep the user signed in without forcing login on every expiration. ([RFC Editor][1])

This is why your summary phrase is good:

> **short-lived access token + refresh token = better security with better UX**. ([IETF Datatracker][2])

---

## 3. Access token: what it does

The access token is what the client presents when requesting protected resources. OAuth says the client accesses protected resources by presenting the access token to the resource server, and the resource server must validate the token, check that it has not expired, and ensure its scope covers the requested resource. ([RFC Editor][1])

In practice, that often looks like:

```http
Authorization: Bearer <access_token>
```

That bearer usage is defined in RFC 6750, which says bearer tokens are used in HTTP requests to access protected resources and that anyone possessing one can use it unless other protections are added. ([RFC Editor][3])

So an access token is the “working credential” used in normal API traffic. ([RFC Editor][1])

---

## 4. Refresh token: what it does

A refresh token is used when the client needs another access token. OAuth 2.0 says refresh tokens are credentials used to obtain access tokens and can be used when the current access token expires or becomes invalid. It also says issuing a refresh token is **optional** and up to the authorization server. ([RFC Editor][1])

That means refresh tokens are not guaranteed in every system or every flow. Modern OAuth security guidance goes further and says the authorization server should decide, based on risk, whether to issue refresh tokens to a given client at all. ([IETF Datatracker][2])

So a better beginner rule is:

> **Access tokens are common. Refresh tokens are conditional.** ([RFC Editor][1])

---

## 5. The biggest difference

The most important distinction is **where each token is allowed to go**.

An access token is meant to be shown to the **resource server** so the client can access a protected API. A refresh token is meant to be shown only to the **authorization server** so the client can obtain a new access token. RFC 6749 states this directly: refresh tokens are intended only for authorization servers and are never sent to resource servers. ([RFC Editor][1])

That leads to a simple memory trick:

* **access token = “use it” token**
* **refresh token = “renew it” token**. ([RFC Editor][1])

---

## 6. How the flow works

Here is the normal flow:

1. The client gets an access token, and maybe a refresh token, from the authorization server. ([RFC Editor][1])
2. The client uses the access token to call the API. ([RFC Editor][1])
3. When the access token expires or is rejected, the client sends the refresh token to the authorization server’s token endpoint. ([RFC Editor][1])
4. The authorization server validates the refresh token and, if valid, issues a new access token and possibly a new refresh token. ([RFC Editor][1])

### Mermaid flow

```mermaid
sequenceDiagram
    participant Client
    participant Auth Server
    participant API Server

    Client->>Auth Server: Login / grant exchange
    Auth Server-->>Client: Access token + optional refresh token

    Client->>API Server: Request with access token
    API Server-->>Client: Protected resource

    Client->>API Server: Request with expired access token
    API Server-->>Client: Invalid token / unauthorized

    Client->>Auth Server: POST /token with refresh_token
    Auth Server->>Auth Server: Validate client + validate refresh token
    Auth Server-->>Client: New access token + optional new refresh token

    Client->>API Server: Retry with new access token
    API Server-->>Client: Protected resource
```

That sequence matches the refresh flow in RFC 6749 Figure 2 and the token refresh rules in the token endpoint section. ([RFC Editor][1])

---

## 7. Example refresh request

OAuth 2.0 gives a token refresh example like this:

```http
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic <client credentials>
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
```

That example comes directly from RFC 6749’s refresh-token grant section, which also says confidential clients or clients issued credentials must authenticate when required by policy. ([RFC Editor][1])

And if the refresh request is valid, the authorization server issues a new access token and may also issue a new refresh token. ([RFC Editor][1])

---

## 8. Why access tokens are usually short-lived

OAuth 2.0 says access tokens represent a scope and duration of access, and current OAuth security guidance explicitly says refresh tokens make it possible to issue access tokens with a **short lifetime**, reducing the impact of leakage. ([RFC Editor][1])

This matters because access tokens travel more often:

* they are sent to APIs
* they are used on normal application requests
* they are more exposed during day-to-day operation. ([RFC Editor][1])

So short-lived access tokens reduce the damage window if one is stolen. ([IETF Datatracker][2])

---

## 9. Why refresh tokens are usually treated more carefully

Refresh tokens are typically longer-lasting credentials. RFC 6749 says they are typically long-lasting and binds them to the client they were issued to. OAuth’s current security guidance also warns that refresh tokens are an attractive target because they can be replayed to mint new access tokens on behalf of the user. ([RFC Editor][1])

That is why refresh tokens require stricter handling:

* keep them confidential in transit and storage
* send them only over TLS
* bind them to the client
* protect them from replay or theft. ([RFC Editor][1])

So although access tokens are the tokens you use more often, refresh tokens are often the **more sensitive long-term credential**. ([RFC Editor][1])

---

## 10. Validation rules for refresh

When a refresh token is used, the authorization server must validate it. RFC 6749 says the authorization server must require client authentication for confidential clients when applicable, authenticate the client if authentication is included, ensure the refresh token was issued to that client, and validate the refresh token before issuing a new access token. ([RFC Editor][1])

This is a very important difference from beginner mental models:

> a refresh token is not just a magic “get new token” string.
> The server checks **who is presenting it** and whether it is still valid. ([RFC Editor][1])

---

## 11. Refresh token rotation

OAuth 2.0 already allowed the authorization server to issue a **new refresh token** during refresh and require the client to discard the old one. ([RFC Editor][1])

Current OAuth best current practice goes further and recommends replay detection for public clients, including **refresh token rotation**. RFC 9700 explains that with rotation, the server issues a new refresh token on every refresh response and invalidates the previous one. If both an attacker and the real client try using the token family, one of them will present an invalidated token, helping the server detect a breach. ([IETF Datatracker][2])

That makes refresh-token rotation one of the clearest modern best practices in this area. ([IETF Datatracker][2])

---

## 12. Expiration and revocation

Refresh tokens do not have to live forever. RFC 9700 says refresh tokens should expire after inactivity, and authorization servers may revoke them automatically after security events such as password change or logout at the authorization server. ([IETF Datatracker][2])

So the usual mental model is:

* **access token expires frequently**
* **refresh token lives longer, but still can expire, be rotated, or be revoked**. ([RFC Editor][1])

That is more accurate than saying “refresh token is permanent.” It is not. ([IETF Datatracker][2])

---

## 13. Security rules worth memorizing

Access tokens must be protected in transit and storage, and bearer-token usage requires protection from disclosure because possession is enough to use the token. RFC 6750 makes TLS mandatory for bearer-token usage, and RFC 6749 requires access token confidentiality in transit and storage. ([RFC Editor][3])

Refresh tokens also must be protected in transit and storage, must be sent only over TLS, and must remain confidential between the authorization server and the client they were issued to. ([RFC Editor][1])

A strong note to remember is:

> **Never send refresh tokens to resource servers. Never treat either token as safe just because it is encoded.** ([RFC Editor][1])

---

## 14. Access token vs refresh token

### Access token

Used to call the API, carries authorization for protected resources, and is validated by the resource server for expiry and scope. It is commonly presented with the `Bearer` scheme. ([RFC Editor][1])

### Refresh token

Used only at the authorization server to obtain a new access token, is typically longer-lasting, and requires stronger protection because it can mint more access tokens. ([RFC Editor][1])

That is the core split of responsibilities. ([RFC Editor][1])

---

## 15. Common misunderstandings

**“Refresh tokens are just longer access tokens.”**
No. They have a different purpose, different audience, and different transmission path. Refresh tokens are not for APIs; they are for the authorization server. ([RFC Editor][1])

**“Every login should always get a refresh token.”**
No. Refresh-token issuance is optional in RFC 6749, and RFC 9700 says issuance should depend on risk assessment. ([RFC Editor][1])

**“A refresh token means the user never has to log in again.”**
Not necessarily. Refresh tokens can expire, be revoked, or be invalidated by rotation or security events. ([IETF Datatracker][2])

**“Only the access token matters.”**
No. A stolen refresh token can be worse because it can be replayed to mint fresh access tokens. ([IETF Datatracker][2])

---

## 16. Interview-style summary

**What is an access token?**
A credential used by the client to access protected resources at the resource server. It has scope and lifetime, and the resource server validates it. ([RFC Editor][1])

**What is a refresh token?**
A credential used to obtain a new access token from the authorization server when the current one expires or becomes invalid. It is never sent to the resource server. ([RFC Editor][1])

**Why use both?**
To allow short-lived access tokens for better security while preserving user experience through controlled renewal. ([IETF Datatracker][2])

**Is a refresh token always issued?**
No. Issuing it is optional and should be based on authorization-server policy and risk. ([RFC Editor][1])

**What is refresh token rotation?**
A security pattern where each refresh produces a new refresh token and invalidates the old one, helping detect replay and limit abuse. ([IETF Datatracker][2])

---

## 17. One-sentence takeaway

**Access tokens are the short-lived credentials you use to call APIs; refresh tokens are the more protected credentials you use to get new access tokens without forcing the user to log in again.** ([RFC Editor][1])

Next is **OAuth 2.0**, where these token ideas finally fit into the full authorization framework.

[1]: https://www.rfc-editor.org/rfc/rfc6749.html "RFC 6749: The OAuth 2.0 Authorization Framework"
[2]: https://datatracker.ietf.org/doc/rfc9700/ "
            
        RFC 9700 - Best Current Practice for OAuth 2.0 Security

        "
[3]: https://www.rfc-editor.org/rfc/rfc6750.html "RFC 6750: The OAuth 2.0 Authorization Framework: Bearer Token Usage"
---------
Absolutely — this is the point where the whole picture starts to connect.

# Topic 7 — OAuth 2.0

## 1. What it is

**OAuth 2.0 is an authorization framework**, not primarily a login protocol. RFC 6749 defines it as a way for a third-party application to obtain **limited access** to an HTTP service, either on behalf of a resource owner or on its own behalf. ([RFC Editor][1])

That means OAuth 2.0 is mainly about this question:

> “How can an app get permission to access a protected resource?”

not:

> “How do we prove who the user is?”

That second question is what **OpenID Connect** adds later. OAuth 2.0 by itself is about delegated access. ([RFC Editor][1])

---

## 2. Why it exists

OAuth 2.0 solves a major problem from older integration patterns: a user should not have to give their password to every third-party app. Instead, the user authorizes access through an authorization server, and the app receives tokens with limited scope and lifetime. RFC 6749’s abstract and role model are built around this delegation idea. ([RFC Editor][1])

A simple mental model:

* User has data at Service A
* App B wants limited access
* User approves that access
* App B gets a token instead of the user’s password ([RFC Editor][1])

---

## 3. The four main roles

RFC 6749 defines four core roles:

### Resource Owner

The entity capable of granting access to a protected resource. If it is a person, that person is the end user. ([RFC Editor][1])

### Client

The application requesting access. This could be a web app, mobile app, SPA, or backend service. ([RFC Editor][1])

### Authorization Server

The server that authenticates the resource owner, obtains authorization, and issues tokens to the client. ([RFC Editor][1])

### Resource Server

The API or service that hosts the protected resource and accepts access tokens. ([RFC Editor][1])

This is one of the most important interview foundations in OAuth. ([RFC Editor][1])

---

## 4. Core idea

OAuth separates:

* the place where permission is granted
* from the place where protected data is served ([RFC Editor][1])

So instead of the client sending a password to the API, the client gets an **access token** from the authorization server and presents that token to the resource server. RFC 6749 defines this split directly in its protocol flow and token model. ([RFC Editor][1])

---

## 5. High-level flow

At a high level, OAuth 2.0 works like this:

1. The client asks for authorization.
2. The resource owner approves or denies it.
3. The client receives an authorization grant.
4. The client exchanges that grant for an access token.
5. The client uses the access token to call the resource server. ([RFC Editor][1])

### Mermaid overview

```mermaid
sequenceDiagram
    participant User as Resource Owner
    participant Client App
    participant Auth Server as Authorization Server
    participant API as Resource Server

    Client App->>User as Resource Owner: Request permission
    User as Resource Owner->>Auth Server as Authorization Server: Approve access
    Auth Server as Authorization Server-->>Client App: Authorization grant
    Client App->>Auth Server as Authorization Server: Exchange grant for access token
    Auth Server as Authorization Server-->>Client App: Access token
    Client App->>API as Resource Server: Call API with access token
    API as Resource Server-->>Client App: Protected resource
```

This is the framework-level pattern in RFC 6749. ([RFC Editor][1])

---

## 6. What an authorization grant is

An **authorization grant** is the credential the client uses to obtain an access token. RFC 6749 defines several grant types. ([RFC Editor][1])

Historically, the spec included:

* authorization code
* implicit
* resource owner password credentials
* client credentials
* extension grants ([RFC Editor][1])

But modern best practice has changed which of these should still be used. RFC 9700 says some older modes are now deprecated or considered insecure. ([RFC Editor][2])

---

## 7. The grant types you should really know

### Authorization Code

This is the most important user-facing OAuth flow. The client first gets an authorization code, then exchanges it for tokens at the token endpoint. RFC 6749 defines this as the core grant for confidential clients and redirection-based flows. ([RFC Editor][1])

### Authorization Code + PKCE

This is the modern best-practice form of the authorization code flow, especially for public clients such as mobile apps and SPAs. RFC 7636 defines PKCE as protection against authorization-code interception, and RFC 9700 recommends PKCE broadly. ([IETF Datatracker][3])

### Client Credentials

Used when the client is acting on its own behalf rather than on behalf of a user. This is common for server-to-server communication. RFC 6749 defines this grant for cases where the client is requesting access to protected resources under its own control or previously arranged access. ([RFC Editor][1])

### Implicit

Defined in RFC 6749, but modern best practice has moved away from it. RFC 9700 deprecates the implicit grant. ([RFC Editor][1])

### Resource Owner Password Credentials

Also defined in RFC 6749, but RFC 9700 deprecates it because it requires the client to handle the user’s password directly, which defeats one of OAuth’s main goals. ([RFC Editor][1])

So for modern notes, I would present the flows like this:

* **Use most often:** Authorization Code + PKCE
* **Use for machine-to-machine:** Client Credentials
* **Know historically but avoid:** Implicit, Password grant ([RFC Editor][2])

---

## 8. The most important modern flow: Authorization Code + PKCE

This is the flow you should understand best.

### Why it exists

The authorization code flow keeps tokens away from the browser redirect itself and exchanges the code at the token endpoint. PKCE adds a defense against interception of that code by requiring proof of possession of a `code_verifier`. RFC 7636 was created specifically to mitigate authorization-code interception attacks for public clients. ([IETF Datatracker][3])

### Mermaid flow

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant Auth Server
    participant API

    Client->>Client: Generate code_verifier and code_challenge
    Client->>Auth Server: /authorize?response_type=code&code_challenge=...
    User->>Auth Server: Login and consent
    Auth Server-->>Client: Redirect back with authorization code
    Client->>Auth Server: POST /token with code + code_verifier
    Auth Server->>Auth Server: Verify code_verifier matches challenge
    Auth Server-->>Client: Access token (+ optional refresh token)
    Client->>API: Authorization: Bearer <access_token>
    API-->>Client: Protected data
```

That is the modern OAuth mental model in one diagram: user consent, code, PKCE proof, then token exchange. ([IETF Datatracker][3])

---

## 9. A simple authorize request example

A browser redirect for the authorization code flow typically includes parameters such as:

* `response_type=code`
* `client_id`
* `redirect_uri`
* `scope`
* `state`

RFC 6749 defines these authorization request parameters for the authorization endpoint. ([RFC Editor][1])

Example shape:

```http
GET /authorize?response_type=code
  &client_id=client123
  &redirect_uri=https://app.example.com/callback
  &scope=read:profile
  &state=xyz123
  &code_challenge=...
  &code_challenge_method=S256
```

The PKCE parameters come from RFC 7636, while the base authorization-code parameters come from RFC 6749. ([RFC Editor][1])

---

## 10. A simple token exchange example

After the client gets the authorization code, it sends a token request to the authorization server’s token endpoint. RFC 6749 defines the token endpoint and code-exchange pattern, and RFC 7636 adds the `code_verifier` for PKCE. ([RFC Editor][1])

Example shape:

```http
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=abc123
&redirect_uri=https://app.example.com/callback
&client_id=client123
&code_verifier=very-random-value
```

If valid, the authorization server issues an access token and may also issue a refresh token. ([RFC Editor][1])

---

## 11. Important parameters to know

### `scope`

OAuth uses scope to describe the level of access requested. RFC 6749 defines `scope` as a space-delimited list of case-sensitive strings representing requested access. ([RFC Editor][1])

### `state`

Used by the client to maintain state between request and callback. In practice, it is also important for CSRF protection in redirect flows. RFC 6749 defines the parameter, and RFC 9700 continues to stress protections around redirect-based attacks. ([RFC Editor][1])

### `redirect_uri`

The callback URI where the authorization server sends the response. This must be tightly validated because redirect URI weaknesses can break the security of the whole flow. RFC 6749 defines it, and RFC 9700 strengthens modern redirect URI guidance. ([RFC Editor][1])

### `client_id`

Identifies the client to the authorization server. It is not usually a secret by itself. RFC 6749 treats it as the public client identifier. ([RFC Editor][1])

---

## 12. OAuth 2.0 is not the same as authentication

This is the most common confusion.

OAuth 2.0 tells you how a client gets **access** to protected resources. It does not by itself standardize how the client learns verified end-user identity details such as “who is this user?” in an interoperable login format. RFC 6749 is explicitly an authorization framework. ([RFC Editor][1])

That is exactly why **OpenID Connect** was created on top of OAuth 2.0. OIDC adds identity. OAuth 2.0 alone adds delegated authorization. This is the clean bridge to your next topic. ([RFC Editor][1])

A useful memory line:

* **OAuth 2.0** = “Can this app access this resource?”
* **OIDC** = “Who is the user who just signed in?” ([RFC Editor][1])

---

## 13. Why OAuth 2.0 is powerful

OAuth 2.0 gives you:

* delegated access without sharing user passwords
* limited scopes
* token lifetimes
* separation of client, authorization server, and resource server
* support for third-party integrations and APIs ([RFC Editor][1])

That is why it became the foundation for modern API ecosystems. RFC 6749 was designed around limited delegated access rather than password sharing. ([RFC Editor][1])

---

## 14. Modern security guidance you should know

Current best practice is not just “implement RFC 6749 literally.” RFC 9700 updates and extends the security advice for OAuth 2.0 and explicitly deprecates some less secure modes. ([RFC Editor][2])

The important modern takeaways are:

* prefer **Authorization Code + PKCE**
* avoid the **Implicit** grant
* avoid the **Resource Owner Password Credentials** grant
* treat redirect URI validation as critical
* use current security guidance, not only the 2012 base spec ([IETF Datatracker][3])

---

## 15. OAuth 2.0 vs what you learned before

### vs Session auth

Session auth is usually app-local and server-stateful: “this server remembers this logged-in browser.” OAuth 2.0 is about delegated access across clients, authorization servers, and resource servers. That contrast follows directly from RFC 6749’s multi-party role model. ([RFC Editor][1])

### vs Bearer tokens

Bearer tokens are one way tokens are presented to APIs. OAuth 2.0 is the broader framework that defines how clients obtain tokens and use them to access protected resources. RFC 6749 and RFC 6750 fit together this way. ([RFC Editor][1])

### vs Access/Refresh tokens

Access and refresh tokens are pieces within OAuth 2.0. OAuth 2.0 defines how and when they are issued and used. ([RFC Editor][1])

---

## 16. Common misunderstandings

**“OAuth means login.”**
Not by itself. OAuth 2.0 is authorization. Login identity is added by OIDC. ([RFC Editor][1])

**“OAuth means JWT.”**
No. OAuth defines a framework and token handling. Access tokens might be JWTs or opaque strings. RFC 6749 does not require JWT as the token format. ([RFC Editor][1])

**“The implicit flow is normal for browser apps.”**
Historically maybe, but current best practice deprecates it. Authorization Code + PKCE is the modern recommendation. ([RFC Editor][2])

**“Password grant is fine because it is simple.”**
It is simple, but modern best practice deprecates it because it requires the client to collect the user’s password directly. ([RFC Editor][2])

---

## 17. Interview-style summary

**What is OAuth 2.0?**
An authorization framework that lets a client obtain limited access to protected resources, either on behalf of a resource owner or on its own behalf. ([RFC Editor][1])

**What are the four roles?**
Resource owner, client, authorization server, and resource server. ([RFC Editor][1])

**What is the most important modern flow?**
Authorization Code with PKCE. PKCE mitigates authorization-code interception attacks and is part of current best practice. ([IETF Datatracker][3])

**Is OAuth 2.0 authentication?**
No. It is authorization. Identity login standardization comes later with OIDC. ([RFC Editor][1])

**Which legacy flows should you know but avoid?**
Implicit and Resource Owner Password Credentials are defined historically but deprecated in current best practice. ([RFC Editor][2])

---

## 18. One-sentence takeaway

**OAuth 2.0 is the framework for delegated access: it lets an app get limited permission to call protected APIs without handling the user’s password directly.** ([RFC Editor][1])

Next is **OpenID Connect**, which answers the missing identity question:
“Okay, but who is the user?”

[1]: https://www.rfc-editor.org/rfc/rfc6749?utm_source=chatgpt.com "RFC 6749: The OAuth 2.0 Authorization Framework"
[2]: https://www.rfc-editor.org/rfc/rfc9700?utm_source=chatgpt.com "RFC 9700: Best Current Practice for OAuth 2.0 Security"
[3]: https://datatracker.ietf.org/doc/html/rfc7636?utm_source=chatgpt.com "RFC 7636: Proof Key for Code Exchange by OAuth Public Clients"
-----
Absolutely — this is the missing identity layer after OAuth 2.0.

# Topic 8 — OpenID Connect

## 1. What it is

**OpenID Connect (OIDC)** is a simple **identity layer on top of OAuth 2.0**. Its job is to let a client verify the identity of the end user based on authentication performed by an authorization server, and to obtain basic profile information in an interoperable way. ([OpenID Foundation][1])

So the cleanest summary is:

* **OAuth 2.0** answers: “Can this app access this API?”
* **OIDC** answers: “Who is the user who just authenticated?” ([OpenID Foundation][1])

---

## 2. Why it exists

OAuth 2.0 gives you token-based delegated access, but by itself it does **not** define standard methods to provide identity information about the authenticated end user. OIDC fills that gap by adding identity-specific pieces such as the **ID Token**, standard user claims, and provider discovery. ([OpenID Foundation][1])

That is why people often say:

> OAuth 2.0 is for **authorization**
> OIDC is for **authentication + identity** ([OpenID Foundation][1])

---

## 3. The most important difference from OAuth 2.0

OIDC is still built on OAuth 2.0 flows, endpoints, and tokens, but it adds one critical thing: an **ID Token** that communicates the result of the user authentication to the client. The OIDC spec says this identity extension is requested by including the `openid` scope in the authorization request, and the authentication information is returned in a JWT called an **ID Token**. ([OpenID Foundation][1])

So a good mental model is:

* **OAuth access token** → for the API
* **OIDC ID token** → for the client application to learn about the user’s authentication ([OpenID Foundation][1])

---

## 4. The main terms

In OIDC, the client application is often called the **Relying Party (RP)**, and the server providing identity is the **OpenID Provider (OP)**, also often called an **Identity Provider (IdP)** or authorization server. The OpenID Foundation’s explainer says the RP is the app or website that outsources user authentication, and the OP is the entity implementing the OIDC and OAuth 2.0 protocols. ([OpenID Foundation][2])

So in plain language:

* **RP** = your app
* **OP/IdP** = the login provider ([OpenID Foundation][2])

---

## 5. The core OIDC objects

OIDC revolves around three big pieces:

1. the **authorization request**
2. the **ID Token**
3. the **UserInfo endpoint** ([OpenID Foundation][1])

The spec’s abstract overview says the RP sends a request to the OP, the OP authenticates the user and returns an **ID Token** and usually an **Access Token**, and then the RP can call the **UserInfo endpoint** with the access token to get claims about the user. ([OpenID Foundation][1])

---

## 6. High-level flow

Here is the basic OIDC story:

1. The app sends the user to the OpenID Provider.
2. The provider authenticates the user.
3. The provider returns an **ID Token**, and usually an **Access Token**.
4. The app can optionally call the **UserInfo endpoint** using the access token.
5. The UserInfo endpoint returns claims about the user. ([OpenID Foundation][1])

```mermaid
sequenceDiagram
    participant User
    participant App as RP / Client
    participant IdP as OP / Identity Provider
    participant UI as UserInfo Endpoint

    App->>IdP: Authorization request (scope includes openid)
    User->>IdP: Authenticate and consent
    IdP-->>App: ID Token + usually Access Token
    App->>App: Validate ID Token
    App->>UI: Request user claims with Access Token
    UI-->>App: User claims (profile, email, etc.)
```

This reflects the abstract protocol overview in the OIDC Core spec. ([OpenID Foundation][1])

---

## 7. How a request becomes “OIDC” instead of plain OAuth

The request becomes an OpenID Connect request when the client includes the **`openid`** scope value. The OIDC Core spec says OpenID Connect requests **must** contain `openid`; without it, the request may still be valid OAuth 2.0, but it is **not** an OIDC request. ([OpenID Foundation][1])

That is one of the easiest interview points to remember:

> **`scope=openid` is the switch that turns OAuth into OIDC.** ([OpenID Foundation][1])

Example shape:

```http
GET /authorize?response_type=code
  &client_id=client123
  &redirect_uri=https://app.example.com/callback
  &scope=openid profile email
  &state=xyz
  &nonce=abc
```

In OIDC, `profile`, `email`, `address`, and `phone` are standard scope values used to request corresponding claim sets. ([OpenID Foundation][1])

---

## 8. What an ID Token is

The **ID Token** is the primary OIDC addition to OAuth 2.0. The Core spec defines it as a security token that contains claims about the authentication of an end user by an authorization server, and it is represented as a **JWT**. ([OpenID Foundation][1])

At minimum, the ID Token includes required claims such as:

* `iss` — issuer
* `sub` — subject identifier for the user
* `aud` — audience, which must include the client’s `client_id`
* `exp` — expiration time
* `iat` — issued-at time ([OpenID Foundation][1])

The OpenID Foundation’s explainer also summarizes the ID Token as representing the outcome of an authentication process and says it contains, at bare minimum, a user identifier (`sub`) and information about how and when the user authenticated. ([OpenID Foundation][2])

---

## 9. What the ID Token is for

The ID Token is mainly for the **client application**, not for the resource API. It tells the client that the OP authenticated the user and gives the client standardized identity claims it can rely on after validation. That is why OIDC says the client must validate the ID Token and its claims before trusting it. ([OpenID Foundation][1])

So:

* **Access token** → call APIs
* **ID token** → tell the client who authenticated ([OpenID Foundation][1])

---

## 10. The `nonce` parameter

OIDC introduces the **`nonce`** parameter to associate a client session with an ID Token and to mitigate replay attacks. The spec says that if a `nonce` is included in the authentication request, the OP must include that same value in the ID Token, and the client must verify they match. ([OpenID Foundation][1])

This is an important OIDC-specific detail because it helps the client detect replayed or mismatched authentication responses. ([OpenID Foundation][1])

---

## 11. What the UserInfo endpoint does

The **UserInfo endpoint** is where the client can use an **access token** to retrieve claims about the end user. The Core spec says the UserInfo endpoint must accept access tokens as OAuth 2.0 bearer tokens, and it returns claims in a JSON response. ([OpenID Foundation][1])

This means the common pattern is:

* identity result in the **ID Token**
* more profile attributes from **UserInfo** using the **access token** ([OpenID Foundation][1])

For the standard scopes, the spec says:

* `profile` requests default profile claims
* `email` requests `email` and `email_verified`
* `address` requests `address`
* `phone` requests `phone_number` and `phone_number_verified` ([OpenID Foundation][1])

And when an access token is issued, claims requested through these scopes are typically returned from the **UserInfo endpoint**. ([OpenID Foundation][1])

---

## 12. Discovery

OIDC also standardizes **Discovery**, which lets a client learn where the provider is and fetch its configuration from a **well-known JSON document**, including endpoint locations. The Discovery spec says the RP first identifies the OP and then retrieves its configuration, including OAuth 2.0 endpoint locations, from a well-known location. ([OpenID Foundation][3])

This is one reason OIDC is so practical across different providers: apps do not need every endpoint hardcoded if the provider supports standard discovery. ([OpenID Foundation][3])

---

## 13. What the client must validate

OIDC is not “just decode the JWT.” The client must validate the ID Token properly. In the Core spec’s validation rules, the client must check that the `iss` exactly matches the expected provider, that the `aud` contains its own `client_id`, and that the token is still valid with respect to time-based claims like `exp`. ([OpenID Foundation][1])

It also must verify the `nonce` when one was sent in the request. ([OpenID Foundation][1])

A clean summary of OIDC validation is:

* trust the right issuer
* ensure the token is meant for this client
* ensure it is not expired
* ensure the nonce matches when used ([OpenID Foundation][1])

---

## 14. OIDC vs OAuth 2.0

This is the distinction you want your notes to make very clear.

**OAuth 2.0** gives a framework for delegated access to protected resources.
**OIDC** uses that framework and adds standardized authentication and identity information, especially via the **ID Token**, claims, and discovery. ([OpenID Foundation][1])

So if someone says, “We use OAuth login,” what they often really mean is:

> “We use OAuth 2.0 + OpenID Connect.” ([OpenID Foundation][1])

---

## 15. Common misunderstandings

**“OIDC replaces OAuth.”**
No. OIDC is built **on top of** OAuth 2.0. ([OpenID Foundation][1])

**“The ID Token is the token the API should use.”**
Usually no. The API normally expects an **access token**; the ID Token is for the client to learn about the authentication event and user identity. ([OpenID Foundation][1])

**“Just decode the ID Token payload and trust it.”**
No. The client must validate the token and claims, including issuer and audience, before relying on it. ([OpenID Foundation][1])

**“`profile` or `email` alone makes it OIDC.”**
No. The required switch is `openid`. Without that scope, it is not an OpenID Connect request. ([OpenID Foundation][1])

---

## 16. Interview-style summary

**What is OpenID Connect?**
An identity layer on top of OAuth 2.0 that lets a client verify the identity of the end user and obtain profile information in a standard way. ([OpenID Foundation][1])

**What is the main addition over OAuth 2.0?**
The **ID Token**, plus standardized claims and discovery. ([OpenID Foundation][1])

**How do you know a request is OIDC?**
It includes the required `openid` scope. ([OpenID Foundation][1])

**What is the UserInfo endpoint for?**
To return claims about the end user when called with an access token. ([OpenID Foundation][1])

**What must the client validate in the ID Token?**
At least issuer, audience, expiration-related timing, and nonce when present. ([OpenID Foundation][1])

---

## 17. One-sentence takeaway

**OpenID Connect adds standardized user identity to OAuth 2.0 by introducing the ID Token, standard claims, UserInfo, and discovery.** ([OpenID Foundation][1])

Next is **SSO**, where these OIDC ideas turn into the user experience of “sign in once, access many apps.”

[1]: https://openid.net/specs/openid-connect-core-1_0.html "Final: OpenID Connect Core 1.0 incorporating errata set 2"
[2]: https://openid.net/developers/how-connect-works/ "How OpenID Connect Works - OpenID Foundation"
[3]: https://openid.net/specs/openid-connect-discovery-1_0-final.html "Final: OpenID Connect Discovery 1.0"
-----
Absolutely — this is the user-facing outcome that the previous topics have been building toward.

# Topic 9 — Single Sign-On (SSO)

## 1. What it is

**Single Sign-On (SSO)** is an authentication process where one account and one set of authenticators can be used to access multiple applications in a seamless way. NIST defines SSO as using one account and its authenticators to access multiple applications, generally through a federation protocol. Microsoft’s documentation says SSO lets users sign in once with one set of credentials and then access multiple independent software systems without signing in again for each one. ([NIST Computer Security Resource Center][1])

So the simplest mental model is:

> **Log in once, access many apps.** ([NIST Computer Security Resource Center][1])

---

## 2. The most important idea

**SSO is not a single protocol.**
It is a **login experience or architectural pattern** that is usually implemented with federation protocols. NIST says SSO is generally implemented with a federation protocol, and Microsoft lists federated SSO options such as **OpenID Connect** and **SAML**. ([NIST Computer Security Resource Center][1])

That means:

* **OIDC** can be used to implement SSO
* **SAML** can be used to implement SSO
* **SSO** is the bigger idea: one login reused across multiple apps. ([Microsoft Learn][2])

---

## 3. Why it exists

SSO exists to reduce repeated logins and centralize authentication. Microsoft says that with SSO, users can access all needed applications without being required to authenticate with different credentials for each one. NIST’s federation guidance explains that identity federation allows one credential service provider to provide authentication attributes to multiple separately administered relying parties. ([Microsoft Learn][2])

So SSO improves both:

* **user experience** — fewer repeated sign-ins
* **identity management** — one central place for authentication and policy. ([Microsoft Learn][2])

---

## 4. How SSO works at a high level

In a typical **federated SSO** setup:

1. The user tries to open App A.
2. App A redirects the user to a central Identity Provider.
3. The user authenticates there.
4. The Identity Provider sends a successful authentication result back to App A.
5. Later, the user opens App B.
6. App B also redirects to the same Identity Provider.
7. Because the user already has a valid IdP session, the Identity Provider can sign them into App B without prompting again. This behavior follows the federated SSO model described by Microsoft and the OIDC flow model described by the OpenID Foundation. ([Microsoft Learn][2])

### Mermaid flow

```mermaid
sequenceDiagram
    participant User
    participant AppA
    participant AppB
    participant IdP as Identity Provider

    User->>AppA: Open App A
    AppA->>IdP: Redirect for login
    User->>IdP: Authenticate
    IdP-->>AppA: Authentication result
    AppA-->>User: Signed in

    User->>AppB: Open App B
    AppB->>IdP: Redirect for login
    IdP->>IdP: Existing IdP session found
    IdP-->>AppB: Authentication result
    AppB-->>User: Signed in without re-login
```

This is the essence of SSO: multiple apps trust the same identity provider, so one central sign-in can be reused. ([NIST Computer Security Resource Center][1])

---

## 5. The central building block: federation

SSO in modern environments is usually based on **federation**. NIST’s federation guidance says federation allows a credential service provider to provide authentication attributes, and optionally subscriber attributes, to a number of separately administered relying parties. ([NIST Computer Security Resource Center][3])

That means the apps do not all authenticate the user independently. Instead, they **trust a common identity provider**. ([NIST Computer Security Resource Center][3])

Common names in this setup are:

* **Identity Provider (IdP)** or **OpenID Provider (OP)** — the central login system
* **Relying Party (RP)** or **Service Provider (SP)** — the application that trusts that login system. OpenID Foundation uses RP/OP terminology for OIDC, while SAML commonly uses SP/IdP terminology. ([OpenID Foundation][4])

---

## 6. Common ways SSO is implemented

### OpenID Connect SSO

OIDC is an interoperable authentication protocol based on OAuth 2.0, and it is commonly used for web, mobile, and modern app login. The OpenID Foundation describes OIDC as a way to verify user identity based on authentication performed by an authorization server. Microsoft also lists OpenID Connect as a federated SSO option. ([OpenID Foundation][4])

### SAML SSO

SAML is another common federated SSO approach, especially in enterprise applications. Microsoft’s SAML SSO documentation describes the SAML single sign-on sequence using an `AuthnRequest` from the service provider and a `Response` from the identity provider. ([Microsoft Learn][5])

So in practice:

* **modern apps** often use **OIDC**
* **many enterprise apps** still use **SAML**
* both can deliver the **SSO experience**. ([Microsoft Learn][2])

---

## 7. SSO is not the same as “session in one app”

This is a very important distinction.

A normal application session means:

> “I logged into this app, and this app remembers me.”

SSO means:

> “I logged into a central identity system, and multiple apps can rely on that login.” This follows from NIST’s definition of SSO across multiple applications and its federation model across multiple relying parties. ([NIST Computer Security Resource Center][1])

So:

* **session auth** = one app remembers the user
* **SSO** = many apps reuse one identity provider login. ([NIST Computer Security Resource Center][1])

---

## 8. SSO vs OAuth 2.0 vs OIDC

This is where learners often get confused.

### OAuth 2.0

OAuth 2.0 is an **authorization framework**. It tells an app how to get limited access to protected resources. By itself, it does not define the standard login identity layer. This follows from RFC 6749 and the way OIDC is positioned on top of it. ([OpenID Foundation][4])

### OIDC

OIDC is an **authentication and identity protocol** on top of OAuth 2.0. It adds identity information and is a common protocol used to achieve SSO. ([OpenID Foundation][4])

### SSO

SSO is the **resulting cross-application login experience**. It is often delivered using OIDC or SAML. ([NIST Computer Security Resource Center][1])

A very useful memory line is:

> **OAuth 2.0 = delegated access**
> **OIDC = login identity**
> **SSO = sign in once across apps**. ([Microsoft Learn][2])

---

## 9. A concrete example

Imagine a company uses one identity platform for:

* email
* HR portal
* expense system
* internal dashboard

The employee signs into the identity platform once. Later, each app redirects to that same identity platform and trusts its authentication result, so the employee is not asked to re-enter credentials each time. This is exactly the kind of multi-application, federation-based experience described by NIST and Microsoft. ([NIST Computer Security Resource Center][1])

---

## 10. Why organizations like SSO

SSO is attractive because it reduces sign-in friction and centralizes authentication choices. Microsoft notes that federated SSO is the richest SSO mode and is preferred when applications support it. Centralized federation also fits NIST’s model of one credential provider serving multiple relying parties. ([Microsoft Learn][2])

In practice, this often enables one place to enforce things like:

* MFA
* conditional access
* central account disablement
* standardized sign-in policy

This last sentence is a common architectural consequence of centralized identity systems; Microsoft’s SSO and enterprise identity documentation reflects that centralized approach even though the exact controls vary by platform. ([Microsoft Learn][2])

---

## 11. Security strengths

SSO can improve security when it reduces password reuse across apps and centralizes authentication controls. Federation lets applications rely on authentication attributes from a central provider instead of each app inventing its own login system. ([NIST Computer Security Resource Center][3])

A practical benefit is that security improvements at the identity provider can benefit many connected apps at once, because those apps trust the same authentication source. This is an inference from the federated model described by NIST and Microsoft. ([Microsoft Learn][2])

---

## 12. Security trade-offs

SSO also creates concentration of trust:

> if the central identity session is compromised, multiple connected apps may be exposed.

That is the natural trade-off of one login unlocking many relying parties in a federation. NIST’s definition of SSO and federation implies this concentration of effect, even though it is a design implication rather than a single quoted sentence. ([NIST Computer Security Resource Center][1])

So SSO is powerful, but the identity provider becomes highly important infrastructure. ([NIST Computer Security Resource Center][3])

---

## 13. SSO is not the same as Single Logout

Another major misunderstanding:

**SSO** means one login can be reused across apps.
It does **not automatically mean** that logging out of one app logs you out of every related app.

Logout across participating apps is usually a separate feature called **Single Logout (SLO)**. The OpenID Foundation has separate final specifications for OIDC session management and logout, and Microsoft has separate documentation for **SAML single sign-out**. ([OpenID Foundation][6])

So remember:

* **SSO** = sign in once
* **SLO** = sign out everywhere, if implemented. ([OpenID Foundation][6])

---

## 14. Common misunderstandings

**“SSO is the same as OAuth.”**
No. OAuth is a framework for authorization. SSO is a user experience pattern that may be implemented using OIDC or SAML. ([Microsoft Learn][2])

**“SSO is a protocol.”**
Not exactly. It is usually implemented through protocols such as OIDC or SAML. ([NIST Computer Security Resource Center][1])

**“If one app remembers me, that is SSO.”**
Not necessarily. That may just be a single-app session. SSO means the login carries across multiple apps through a shared identity system. ([NIST Computer Security Resource Center][1])

**“SSO automatically means global logout.”**
No. Single logout is separate and requires its own support. ([OpenID Foundation][6])

---

## 15. Interview-style summary

**What is SSO?**
An authentication process where one account and one authentication event can be used to access multiple applications seamlessly, usually through federation. ([NIST Computer Security Resource Center][1])

**Is SSO a protocol?**
No. It is usually implemented with federation protocols such as OIDC or SAML. ([NIST Computer Security Resource Center][1])

**How is SSO different from OIDC?**
OIDC is a protocol for authentication and identity; SSO is the cross-application sign-in experience that OIDC can help provide. ([Microsoft Learn][2])

**How is SSO different from a normal session?**
A session is one app remembering a user; SSO is multiple apps trusting a shared identity provider login. ([NIST Computer Security Resource Center][1])

**Does SSO imply logout everywhere?**
No. That requires separate single-logout support. ([OpenID Foundation][6])

---

## 16. One-sentence takeaway

**SSO is the experience of signing in once and accessing many applications, usually by having multiple apps trust the same identity provider through protocols like OIDC or SAML.** ([NIST Computer Security Resource Center][1])

We’ve now covered the full sequence. The best next step is to turn all nine topics into one polished set of ordered study notes with consistent diagrams, comparisons, and interview summaries.

[1]: https://csrc.nist.gov/glossary/term/Single_Sign_On "Single Sign-On - Glossary | CSRC"
[2]: https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/what-is-single-sign-on "What is single sign-on? - Microsoft Entra ID | Microsoft Learn"
[3]: https://csrc.nist.gov/pubs/sp/800/63/c/4/final "SP 800-63C-4, Digital Identity Guidelines: Federation and Assertions | CSRC"
[4]: https://openid.net/developers/how-connect-works/ "How OpenID Connect Works - OpenID Foundation"
[5]: https://learn.microsoft.com/en-us/entra/identity-platform/single-sign-on-saml-protocol "Single sign-on SAML (Security Assertion Markup Language) protocol - Microsoft identity platform | Microsoft Learn"
[6]: https://openid.net/the-openid-connect-logout-specifications-are-now-final-specifications/?utm_source=chatgpt.com "The OpenID Connect Logout specifications are now Final Specifications ..."
