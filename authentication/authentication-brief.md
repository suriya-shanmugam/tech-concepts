
# Authentication, Explained Clearly: From Basic Auth to SSO

Authentication becomes confusing when different topics are mixed together without a clear path. Some methods are about sending a secret. Some are about remembering a logged-in user. Some are about granting an app permission. Some are about identifying the user. And some are about making one login work across many apps.

The easiest way to understand it is to learn in this order:

1. Basic Authentication
2. Digest Authentication
3. API Keys
4. Session-based Authentication
5. Bearer Tokens and JWT
6. Access Tokens and Refresh Tokens
7. OAuth 2.0
8. OpenID Connect
9. Single Sign-On

That order matters. Each topic is basically trying to fix a limitation of the previous one.

```mermaid
flowchart LR
    A[Basic Auth] --> B[Digest Auth]
    B --> C[API Keys]
    C --> D[Session-based Auth]
    D --> E[Bearer Tokens]
    E --> F[JWT]
    F --> G[Access + Refresh Tokens]
    G --> H[OAuth 2.0]
    H --> I[OpenID Connect]
    I --> J[SSO]
```

---

# 1) Basic Authentication

## Clear explanation

Basic Authentication is the simplest possible idea:

The client sends a username and password with the request.

Usually, the username and password are joined like this:

```text
username:password
```

Then that value is Base64-encoded and sent in the HTTP header.

Example shape:

```http
Authorization: Basic dXNlcjpwYXNz
```

The important thing is this:

**Base64 is not security.**
It is just a different way to represent the same text.

So Basic Auth is basically:

> “Here is my username and password on this request.”

---

## Why it exists

Because it is simple.

It works well for:

* quick testing
* internal tools
* scripts
* very simple systems

It is easy to add and easy to understand.

---

## Flow

```mermaid
sequenceDiagram
    participant Client
    participant Server

    Client->>Server: Request + username/password (Base64 encoded)
    Server->>Server: Decode and verify
    alt Valid
        Server-->>Client: Allow access
    else Invalid
        Server-->>Client: Reject request
    end
```

---

## Example shape

```http
GET /profile HTTP/1.1
Host: example.com
Authorization: Basic YWxpY2U6c2VjcmV0MTIz
```

---

## Main objects / terminologies

**Username**
The identity claim from the client.

**Password**
The secret used to prove the user knows the account secret.

**Authorization header**
The HTTP header where the credentials are sent.

**Base64 encoding**
A text encoding format. Not encryption.

---

## Caveats

* Password is effectively being sent with requests.
* Base64 does not protect the password.
* Should only be used over HTTPS.
* Not great for modern apps.
* No built-in concept of login session, token expiry, or logout.

---

# 2) Digest Authentication

## Clear explanation

Digest Authentication tries to improve Basic Auth.

Instead of sending the actual password, the client sends a **calculated response** based on:

* the username
* the password
* a server-provided challenge
* request details

So the client is proving:

> “I know the password”

without sending the raw password directly.

That is the key idea.

---

## Why it exists

Basic Auth is too direct.
Digest was created to avoid sending the password itself.

So Digest introduces a challenge-response pattern:

1. server sends a challenge
2. client uses the password to compute a digest
3. server verifies that digest

---

## Flow

```mermaid
sequenceDiagram
    participant Client
    participant Server

    Client->>Server: Request protected resource
    Server-->>Client: Challenge (nonce, realm, etc.)
    Client->>Client: Compute digest using password + challenge
    Client->>Server: Send digest response
    Server->>Server: Verify digest
    alt Valid
        Server-->>Client: Allow access
    else Invalid
        Server-->>Client: Reject request
    end
```

---

## Example shape

```http
Authorization: Digest username="alice", realm="admin", nonce="abc123", uri="/profile", response="computed-value"
```

---

## Main objects / terminologies

**Challenge**
Data sent by the server to the client before authentication.

**Nonce**
A server-generated value used in the calculation to reduce replay risk.

**Digest**
A computed hash-like response derived from the password and challenge.

**Realm**
A label for the protected area.

---

## Caveats

* More secure than Basic in concept, but more complex.
* Still part of older HTTP auth styles.
* Rare in modern app architecture.
* Does not solve sessions, SSO, delegated access, or identity federation.
* Often learned for understanding history, not because it is the main modern choice.

---

# 3) API Keys

## Clear explanation

An API key is a secret string given to a client application, which the client sends to the server to identify itself.

This usually means:

> “This request is coming from this app/project/integration.”

Not necessarily:

> “This request is coming from this human user.”

That distinction is very important.

API keys are often more about **client identification** than real end-user authentication.

---

## Why it exists

APIs need a simple way to know:

* which app is calling
* who should be rate limited
* which project should be billed
* which integration should be allowed or blocked

API keys are a lightweight solution for that.

---

## Flow

```mermaid
sequenceDiagram
    participant ClientApp
    participant APIServer
    participant PolicyStore

    ClientApp->>APIServer: Request + API key
    APIServer->>PolicyStore: Check key
    PolicyStore-->>APIServer: Valid / invalid / quota info
    alt Allowed
        APIServer-->>ClientApp: Response
    else Denied
        APIServer-->>ClientApp: Reject request
    end
```

---

## Example shape

```http
GET /weather HTTP/1.1
Host: api.example.com
X-API-Key: abcdef12345
```

Sometimes also:

```http
GET /weather?api_key=abcdef12345
```

---

## Main objects / terminologies

**API key**
A shared secret used by a client app to identify itself.

**Client application**
The app or system making the request.

**Quota / rate limit**
Rules controlling how much the client can use the API.

**Header vs query parameter**
Common places where the key is sent.

---

## Caveats

* Usually identifies the app, not the user.
* If stolen, it can often be reused until revoked.
* Should not be treated like a full user login system.
* Avoid putting it in URLs when possible.
* Best for simple API access control, not full identity.

---

# 4) Session-based Authentication

## Clear explanation

Session-based authentication is the classic web-app login model.

The user logs in once.
The server then creates a session and remembers that user.

After that, the browser sends a session ID on future requests, and the server uses that session ID to look up the logged-in user.

So the real idea is:

> the server remembers you

This is why it is called **stateful** authentication.

---

## Why it exists

Without sessions, the user would need to send credentials again and again.

Sessions solve that problem by separating:

* the login step
* from later authenticated requests

That makes web apps feel normal:

* log in once
* click around many pages
* stay signed in

---

## Flow

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant Server
    participant SessionStore

    User->>Browser: Enter username/password
    Browser->>Server: Login request
    Server->>Server: Verify credentials
    Server->>SessionStore: Create session
    SessionStore-->>Server: session_id
    Server-->>Browser: Set session cookie
    Browser->>Server: Later request + session cookie
    Server->>SessionStore: Look up session
    SessionStore-->>Server: Logged in user
    Server-->>Browser: Protected response
```

---

## Example shape

Login response:

```http
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax
```

Later request:

```http
Cookie: session_id=abc123
```

---

## Main objects / terminologies

**Session**
Server-side stored login state.

**Session ID**
The identifier sent by the browser so the server can find the session.

**Cookie**
The browser mechanism commonly used to send the session ID.

**Stateful**
The server stores and remembers session state.

---

## Caveats

* The browser usually stores only the session ID, not the whole session.
* Session auth is not the same as browser `sessionStorage`.
* If the session cookie is stolen, the session may be hijacked.
* Cookie security matters: `HttpOnly`, `Secure`, `SameSite`.
* Great for classic web apps, but less natural for distributed API-heavy systems.

---

# 5) Bearer Tokens and JWT

## Clear explanation

These two terms are related, but they are not the same.

A **bearer token** means:

> whoever holds the token can use it

A **JWT** means:

> a token formatted as a compact JSON-based structure with claims inside it

So:

* **Bearer** describes how the token is used
* **JWT** describes what the token looks like

A bearer token can be a JWT.
But not every bearer token is a JWT.

---

## Why they exist

Modern systems, especially APIs, often do not want to use server sessions for everything.

Instead, they want the client to send an access credential directly with each request.

That is where bearer tokens help.

JWT becomes popular because it can carry useful claims inside the token, such as:

* who the user is
* who issued the token
* who the audience is
* when it expires

---

## Flow

```mermaid
sequenceDiagram
    participant Client
    participant AuthServer
    participant API

    Client->>AuthServer: Authenticate / obtain token
    AuthServer-->>Client: Token
    Client->>API: Request + Bearer token
    API->>API: Validate token
    alt Valid
        API-->>Client: Protected response
    else Invalid
        API-->>Client: Unauthorized
    end
```

---

## Example shape

Bearer header:

```http
Authorization: Bearer eyJhbGciOi...
```

JWT shape:

```text
header.payload.signature
```

Example payload shape:

```json
{
  "sub": "user123",
  "iss": "auth.example.com",
  "aud": "api.example.com",
  "exp": 1717000000
}
```

---

## Main objects / terminologies

**Bearer token**
A token usable by whoever possesses it.

**JWT**
A structured token format carrying claims.

**Claims**
Data inside the token, like user ID or expiration time.

**Signature**
Protects integrity, meaning tampering can be detected.

**Issuer (`iss`)**
Who created the token.

**Audience (`aud`)**
Who the token is meant for.

**Expiration (`exp`)**
When the token is no longer valid.

---

## Caveats

* If a bearer token is stolen, it can often be used.
* JWT payloads are encoded, not automatically encrypted.
* Never trust a JWT just because you can decode it.
* The token must be validated properly.
* Bearer and JWT should not be treated as synonyms.

---

# 6) Access Tokens and Refresh Tokens

## Clear explanation

This topic answers a very practical question:

> Why not just issue one token and use it forever?

Because that would be bad for security.

So modern systems often split the job into two tokens:

* **Access token**: used to call the API
* **Refresh token**: used to get a new access token

This creates a balance:

* short-lived working token for normal use
* longer-lived renewal token for user convenience

---

## Why it exists

If access tokens last too long, stolen tokens remain useful for too long.

If access tokens expire too quickly without renewal, users must log in too often.

Refresh tokens solve that tradeoff.

So the design goal is:

> better security without terrible user experience

---

## Flow

```mermaid
sequenceDiagram
    participant Client
    participant AuthServer
    participant API

    Client->>AuthServer: Login / token request
    AuthServer-->>Client: Access token + Refresh token
    Client->>API: Use access token
    API-->>Client: Response
    Client->>API: Use expired access token
    API-->>Client: Unauthorized
    Client->>AuthServer: Send refresh token
    AuthServer-->>Client: New access token
    Client->>API: Retry with new access token
    API-->>Client: Response
```

---

## Example shape

Access token usage:

```http
Authorization: Bearer <access_token>
```

Refresh request shape:

```http
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=<refresh_token>
```

---

## Main objects / terminologies

**Access token**
The token used to access protected APIs.

**Refresh token**
The token used to obtain a new access token.

**Expiration**
Access tokens usually expire sooner.

**Rotation**
Replacing an old refresh token with a new one after use.

---

## Caveats

* Refresh tokens are usually more sensitive than people think.
* Refresh tokens should not be sent to normal APIs.
* Access tokens are often short-lived by design.
* Refresh tokens may also expire or be revoked.
* Rotation is often used to reduce abuse risk.

---

# 7) OAuth 2.0

## Clear explanation

OAuth 2.0 is not mainly “login.”
It is a framework for **delegated authorization**.

That means:

> one app can get limited permission to access something on behalf of a user

without asking the user to give that app their password directly.

This is the core problem OAuth solves.

Example:

* You want a calendar app to access your Google Calendar.
* The app redirects you to Google.
* You approve access there.
* The app gets tokens instead of your Google password.

That is OAuth thinking.

---

## Why it exists

Without OAuth, third-party apps might ask users for their passwords directly.

That is dangerous and hard to control.

OAuth improves this by introducing:

* permission screens
* scopes
* limited tokens
* central authorization

So the user can say:

> “I allow this app to do this specific thing”

instead of:

> “Here is my password.”

---

## Flow

```mermaid
sequenceDiagram
    participant User
    participant ClientApp
    participant AuthServer
    participant ResourceServer

    ClientApp->>User: Ask to connect account
    User->>AuthServer: Login and approve access
    AuthServer-->>ClientApp: Authorization result / code
    ClientApp->>AuthServer: Exchange for token
    AuthServer-->>ClientApp: Access token
    ClientApp->>ResourceServer: Call API with token
    ResourceServer-->>ClientApp: Protected resource
```

---

## Example shape

Authorization request shape:

```http
GET /authorize?
  response_type=code
  &client_id=client123
  &redirect_uri=https://app.example.com/callback
  &scope=read_profile
  &state=xyz
```

Token exchange shape:

```http
POST /token
grant_type=authorization_code
&code=abc123
&redirect_uri=https://app.example.com/callback
```

---

## Main objects / terminologies

**Resource owner**
Usually the user who owns the data.

**Client**
The application asking for access.

**Authorization server**
The system that authenticates the user and grants tokens.

**Resource server**
The API holding the protected data.

**Scope**
What level of access is being requested.

**Authorization code**
A short-lived code later exchanged for tokens.

**PKCE**
A protection added to modern authorization code flows.

---

## Caveats

* OAuth is about authorization, not identity by itself.
* OAuth does not automatically tell the client who the user is in a standardized way.
* Older flows are less preferred now.
* Redirect security matters a lot.
* Best understood as permission delegation.

---

# 8) OpenID Connect

## Clear explanation

OpenID Connect, or OIDC, is the identity layer added on top of OAuth 2.0.

OAuth answers:

> “Can this app access this resource?”

OIDC answers:

> “Who is the user who just signed in?”

That is the main job of OIDC.

It adds a special token called the **ID Token**, which tells the client about the user’s authentication.

---

## Why it exists

OAuth alone is not enough for modern login.

An application often needs to know:

* who signed in
* which provider authenticated them
* whether the sign-in result is trustworthy

OIDC standardizes that.

So instead of every provider inventing its own login identity format, OIDC gives a shared model.

---

## Flow

```mermaid
sequenceDiagram
    participant User
    participant App
    participant IdentityProvider
    participant UserInfo

    App->>IdentityProvider: Login request with openid scope
    User->>IdentityProvider: Authenticate
    IdentityProvider-->>App: ID Token + Access Token
    App->>App: Validate ID Token
    App->>UserInfo: Optional request for more profile data
    UserInfo-->>App: User claims
```

---

## Example shape

Authorization request:

```http
GET /authorize?
  response_type=code
  &client_id=client123
  &redirect_uri=https://app.example.com/callback
  &scope=openid profile email
  &state=xyz
  &nonce=abc
```

ID token payload shape:

```json
{
  "iss": "https://idp.example.com",
  "sub": "user123",
  "aud": "client123",
  "exp": 1717000000,
  "iat": 1716996400
}
```

---

## Main objects / terminologies

**OIDC**
Identity layer on top of OAuth 2.0.

**ID Token**
A token telling the client about the authentication result and user identity.

**`openid` scope**
The switch that makes the request OIDC.

**UserInfo endpoint**
An endpoint used to fetch more user claims.

**Nonce**
A value used to tie the request and token together more safely.

**Claims**
Identity data such as subject, email, or name.

---

## Caveats

* OIDC does not replace OAuth; it builds on top of it.
* The ID token is not the same as the access token.
* APIs usually want access tokens, not ID tokens.
* ID tokens must be validated properly.
* OIDC is the right model when the app needs standardized login identity.

---

# 9) Single Sign-On (SSO)

## Clear explanation

Single Sign-On means:

> sign in once, access many applications

SSO is not one specific protocol.
It is the overall login experience created when multiple apps trust the same identity provider.

So SSO is the user-facing result.

---

## Why it exists

Without SSO, users sign in separately to:

* email
* HR portal
* dashboard
* internal tools
* support systems

That is annoying and hard to manage.

SSO improves this by centralizing login.

One identity system handles authentication, and many apps trust it.

---

## Flow

```mermaid
sequenceDiagram
    participant User
    participant AppA
    participant AppB
    participant IdentityProvider

    User->>AppA: Open app A
    AppA->>IdentityProvider: Redirect for login
    User->>IdentityProvider: Sign in
    IdentityProvider-->>AppA: Successful authentication
    User->>AppB: Open app B
    AppB->>IdentityProvider: Redirect for login
    IdentityProvider->>IdentityProvider: Existing login session found
    IdentityProvider-->>AppB: Successful authentication
```

---

## Example shape

Conceptually:

* App A trusts the identity provider
* App B trusts the same identity provider
* user signs in once at the identity provider
* both apps accept that result

SSO is often implemented using:

* OIDC
* SAML

---

## Main objects / terminologies

**SSO**
Single Sign-On, one login used across multiple apps.

**Identity Provider (IdP)**
The central system that authenticates the user.

**Relying Party / Service Provider**
The application that trusts the identity provider.

**Federation**
Trust between separate systems for identity.

**Single Logout (SLO)**
A separate idea: signing out across apps.

---

## Caveats

* SSO is not the same as one app session.
* SSO is not the same thing as OAuth by itself.
* SSO does not automatically mean global logout.
* If the central identity provider is compromised, many apps may be affected.
* OIDC and SAML are common ways to implement SSO.

---

# Final comparison: how to think about all of them

Here is the simplest mental map.

## Direct credential methods

* **Basic Auth**: send username/password directly
* **Digest Auth**: prove knowledge of password with a challenge

## Client/app identification

* **API Key**: identify the calling application or integration

## App-managed login

* **Session-based auth**: server remembers the logged-in user

## Token-based API access

* **Bearer token**: whoever has it can use it
* **JWT**: a common format for token contents
* **Access token**: used to call APIs
* **Refresh token**: used to get a new access token

## Delegated authorization and identity

* **OAuth 2.0**: app gets permission to access resources
* **OIDC**: app learns who the user is
* **SSO**: one login works across many apps

---

# The big picture

If you remember only one thing, remember this:

* **Basic / Digest** are old-style ways to prove credentials on requests.
* **API keys** identify apps more than users.
* **Sessions** let the server remember a logged-in user.
* **Bearer/JWT** are modern token ideas for APIs.
* **Access + Refresh tokens** balance security and usability.
* **OAuth 2.0** is about permission delegation.
* **OIDC** is about identity.
* **SSO** is about one login across many apps.

