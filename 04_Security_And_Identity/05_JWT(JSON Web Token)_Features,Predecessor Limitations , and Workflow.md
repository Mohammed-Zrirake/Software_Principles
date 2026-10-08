
#SpringBoot #Security #Authentication #JWT #Stateless #API

# JWT: Stateless Authentication for Modern APIs

> JWT (JSON Web Token) is the modern standard for securing [[REST]] APIs. It's a compact, digitally signed, self-contained token that proves a user's identity and permissions without requiring the server to store any session state.
---
## 🤔 The Problem: Limitations Before JWT

Before JWT became standard, server-side authentication created major challenges, especially for scalable, modern applications.

> [!danger] 1. Session-Based Authentication (The Monolith's Memory)
> -   **How it worked:** The server would create a session for a user upon login, store it in memory, and give the client a session ID (usually in a cookie). For every request, the server had to look up this session to identify the user.
> -   **The Limitation:** This creates **state** on the server. It doesn't scale well in a distributed [[Microservices]] environment. If a user's request hits a different server instance that doesn't have their session data, they are no longer authenticated.

> [!danger] 2. Opaque Tokens (The Database-Backed Token)
> -   **How it worked:** The server would generate a random, "opaque" string (a token), store it in a database table linked to a user, and send it to the client.
> -   **The Limitation:** For every single API request, the server had to perform a **database lookup** to validate the token and retrieve user information. This adds significant latency and database load.

---

## ✅ The Solution: JWT Features

JWT solves these problems by being **stateless** and **self-contained**.

✔️ **Stateless:** The server does not need to store any session information. All the data needed for authentication is in the token itself.
✔️ **Self-Contained:** The token's payload contains "claims"—pieces of information about the user, such as their ID, roles, and permissions.
✔️ **Digitally Signed:** The token is signed with a secret key (using HMAC) or a private key (using RSA/ECDSA). This signature guarantees that the token has not been tampered with.
✔️ **Compact & Portable:** JWTs are just Base64Url-encoded strings, making them small and easy to send in HTTP headers or URL parameters.
✔️ **Standardized:** It is an open standard (RFC 7519).

---

## 🔬 Anatomy of a JWT: `Header.Payload.Signature`

A JWT consists of three parts, separated by dots.

**Example Token:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwicm9sZSI6ImFkbWluIiwiZXhwIjoxNzEwMDAwMDAwfQ.6Yt7eYvFtY3hl_0jydEoI3vXOCGnLZpiQx`

1.  **Header (Base64Url Encoded):** Metadata about the token, like the signing algorithm.
    `{"alg": "HS256", "typ": "JWT"}`
2.  **Payload (Base64Url Encoded):** The "claims" or data about the user.
    `{"sub": "1234567890", "name": "John Doe", "role": "admin", "exp": 1710000000}`
3.  **Signature:** A cryptographic signature created by hashing the header, payload, and a secret key. This is how the server verifies the token's authenticity.

---

## ⚙️ The JWT Workflow

```mermaid
sequenceDiagram
    participant Client
    participant Server

    Client->>+Server: 1. Login Request (username, password)
    Server->>Server: Validate Credentials
    Server->>Server: Generate JWT (with claims & signature)
    Server-->>-Client: 2. Return JWT in Response

    Note over Client: Store JWT securely (e.g., HttpOnly Cookie)

    Client->>+Server: 3. Subsequent Request with JWT (in Authorization Header)
    Server->>Server: 4. Verify JWT Signature & Expiration
    Server-->>-Client: 5. Return Protected Resource
```

### The HTTP Exchange in Practice

**1. User Logs In (Client → Server)**
```http
POST /api/auth/login
Content-Type: application/json

{
  "username": "johndoe",
  "password": "mypassword123"
}
```

**2. JWT is Returned (Server → Client)**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM..."
}
```

**3. JWT is Used in Subsequent Requests (Client → Server)**
The client includes the JWT in the `Authorization` header, typically using the `Bearer` scheme.

```http
GET /api/dashboard
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM...
```

**4. Server Verifies the Token**
For every request, the server:
-   Extracts the token from the header.
-   Verifies the signature using its secret key. If it fails, the request is rejected immediately (401 Unauthorized).
-   Checks the `exp` (expiration) claim to ensure the token hasn't expired.
-   If everything is valid, it uses the claims (e.g., user ID, roles) to process the request.

---

> [!summary] Conclusion
> JWT revolutionized API security by eliminating the need for server-side session state. By creating a self-contained, verifiable token, it provides a scalable, efficient, and secure method for handling authentication and authorization in modern distributed applications like [[Microservices]].