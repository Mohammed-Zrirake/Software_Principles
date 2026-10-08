#SpringBoot #Security #JWT #Authentication #Web #API
# JWT Storage: Bearer Tokens vs. Cookies

> Once a server generates a [[JWT]], the client needs to store it. The two primary methods are storing it in **`localStorage`** (and sending it as a **Bearer Token**) or storing it in a secure **`HttpOnly` Cookie**. This choice has a major impact on your application's security against common web attacks like XSS and CSRF.

---
## 1. JWT as a Bearer Token (Stateless API Standard)

> In this approach, the server sends the JWT in the response body, and the client-side JavaScript is responsible for storing it (usually in `localStorage` or `sessionStorage`) and attaching it to every subsequent API request.
### The Flow
1.  **Storage:** The client receives the JWT and stores it: `localStorage.setItem('jwt', token);`
2.  **Transmission:** For every API call, the client manually adds an `Authorization` header:
    `Authorization: Bearer <jwt_token>`

### Security Implications
-   **Vulnerable to XSS (Cross-Site Scripting):** This is the **primary weakness**. If an attacker successfully injects a malicious script into your application, that script can read the entire contents of `localStorage` and steal the user's JWT. With the token, they can impersonate the user completely.
-   **Immune to CSRF (Cross-Site Request Forgery):** This approach is naturally safe from CSRF attacks. CSRF relies on the browser automatically including cookies with requests. Since the Bearer Token must be added manually by JavaScript, a malicious site cannot force a user's browser to make an authenticated request on their behalf.

### ✅ Best For
-   **[[REST]] APIs** consumed by non-browser clients (e.g., mobile apps, other services).
-   **Single-Page Applications (SPAs)** where the frontend and backend are on different domains (though this requires careful CORS configuration).
-   Situations where you need maximum flexibility and statelessness.

---

## 2. JWT in an `HttpOnly` Cookie (Secure Web App Standard)
> In this approach, the server sets the JWT inside a browser cookie with special security flags. The browser then handles storing the token and automatically attaching it to every request made to the same domain.

### The Flow
1.  **Storage:** The server sends a `Set-Cookie` header in the login response:
    `Set-Cookie: jwt=<jwt_token>; HttpOnly; Secure; SameSite=Strict`
2.  **Transmission:** The browser automatically includes the cookie in the `Cookie` header on every subsequent request. Your client-side JavaScript **does not need to do anything**.

### Security Implications
-   **Secure Against XSS:** This is the **primary strength**. The `HttpOnly` flag makes the cookie completely inaccessible to JavaScript. Even if an XSS attack occurs, the malicious script cannot read the cookie and steal the JWT.
-   **Vulnerable to CSRF:** This is the **primary weakness**. Because the browser sends cookies automatically, an attacker can trick a user into visiting a malicious site that makes a request to your application (e.g., via a hidden form). The browser will dutifully attach the JWT cookie, and the malicious request will be authenticated. **You must implement separate CSRF protection** (like a CSRF token) to mitigate this.

### ✅ Best For
-   **Traditional Web Applications:** Monolithic applications where the frontend and backend are served from the same domain.
-   **Security-Sensitive Applications:** When protecting against XSS is the highest priority.
-   Server-side rendered applications (e.g., using Thymeleaf).

---

> [!summary] At-a-Glance Comparison
>| Feature | JWT Bearer Token (via `localStorage`) | JWT in `HttpOnly` Cookie |
>| :--- | :--- | :--- |
>| **Storage** | `localStorage` / `sessionStorage` | Browser Cookie Jar |
>| **Transmission** | **Manual:** Added to `Authorization` header by JS. | **Automatic:** Added to `Cookie` header by browser. |
>| **XSS Protection** | **Vulnerable.** JS can access `localStorage`. | **Secure.** `HttpOnly` flag prevents JS access. |
>| **CSRF Protection** | **Immune.** Tokens are not sent automatically. | **Vulnerable.** Requires additional CSRF protection. |
>| **Best For** | APIs, SPAs, Mobile Apps | Traditional Web Apps, High-Security Apps |

---

> [!tip] Which One Should You Choose?
> There is no single "best" answer; it's a trade-off.
> -   If you are building a decoupled **SPA + API**, the **Bearer Token** approach is more common and flexible, but you must be diligent about preventing XSS vulnerabilities.
> -   If you are building a **monolithic web application**, the **HttpOnly Cookie** approach is generally considered more secure out-of-the-box against the most common client-side attack (XSS), provided you also implement CSRF protection.