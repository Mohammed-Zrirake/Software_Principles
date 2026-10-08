#CoreConcept #Security #Authentication #JWT #OAuth2 #OIDC

> These are the different standardized methods an application can use to manage user access after initial identity verification. **Session/Cookies** are stateful passes. **JWTs** are stateless, self-contained passes. **OAuth 2.0** is for giving *other apps* limited access. **OpenID Connect** is the "Login with..." feature built on top of OAuth.

---

### 1. Session-Based Authentication (Cookies)

> The traditional, **stateful** method of managing user logins.

> [!note] The Analogy: The Visitor's Sticker & The Logbook
> 1.  You show your ID at the front desk (you log in with a username/password).
> 2.  The guard gives you a simple, numbered visitor's sticker to wear (a **session ID cookie**) and makes an entry in a logbook: "Visitor #345 is Alice, access granted." (The server stores the **session** in its memory or a database).
> 3.  At every secure door you approach, the guard there must call the front desk to check if "Visitor #345" is still valid in the logbook.
>
> The sticker itself is dumb and contains no information; the server-side **logbook (the session store)** holds all the state.

### 2. JWT (JSON Web Token)

> A modern, **stateless** method for managing user logins, especially for APIs.

> [!note] The Analogy: The Smart Keycard
> 1.  You show your ID at the front desk.
> 2.  The guard gives you a sophisticated, digitally signed, laminated keycard (**a JWT**).
> 3.  The card itself contains encrypted information: "This is Alice, Role: Marketing, Access valid until 5 PM." The signature is tamper-proof.
> 4.  Any guard at any door can look at the card, verify the signature is authentic, and grant access **without having to call the front desk**.
>
> The keycard is self-contained and verifiable on its own. It is **stateless**.

### 3. OAuth 2.0

> This is **NOT** a protocol for logging in. It is a protocol for **delegated authorization**—allowing one application to access resources on behalf of a user from another application.

> [!note] The Analogy: The Limited-Use Keycard for a Third Party
> You want to use a third-party printing service (a **client application**) to access and print a document from your secure corporate document vault (the **resource server**, e.g., Google Drive).
>
> You would **never give the printing service your main corporate password**.
>
> Instead, you go to the vault's security office (Google's **authorization server**), prove who you are, and ask them to give the printing service a special, temporary keycard (**an access token**) that *only* allows them to read documents from a specific folder and nothing else.
>
> OAuth 2.0 is the standardized process for requesting and granting these limited-use keycards.

### 4. OpenID Connect (OIDC)

> This is a thin identity layer built **on top of OAuth 2.0**. It **IS** for logging in.

> [!note] The Analogy: The "Login with Google" Button
> OIDC uses the exact same OAuth 2.0 flow, but with one crucial extra step.
>
> After the security office gives the printing service its limited-access keycard (the OAuth **access token**), it *also* gives them a signed ID card (**an ID Token**) that explicitly says, "We have successfully authenticated this person, and their name is Alice Smith and here is their email."
>
> **OIDC provides the identity; OAuth provides the access.**

---

## Why This Matters to a Developer

Choosing the right authentication protocol is a foundational architectural decision that directly impacts your application's scalability, security, and ability to integrate with other services.

-   **✔️ Choosing the Right Architecture:**
    -   **Session/Cookies** are simple and effective for traditional, monolithic web applications where statefulness is not a major scaling concern.
    -   **JWTs** are essential for building stateless [[Microservices]], mobile backends, and SPAs where requests can be handled by any server instance.
    -   **OAuth 2.0 / OIDC** is the non-negotiable standard if you need to implement "Login with..." functionality (Single Sign-On, SSO) or allow third-party applications to securely access your users' data via your API.

-   **✔️ Building Scalable Systems:** **JWTs** are the key to building truly stateless authentication. This means any of your application servers can validate a user without needing to connect to a shared session database. This makes your system dramatically easier to [[Load Balancing & Reverse Proxies|scale horizontally]] by simply adding more server instances.

-   **✔️ Security and User Experience:** Using OIDC allows you to leverage the robust security and user trust of major identity providers like Google, Microsoft, or Okta. This offloads the responsibility of password management and can increase user adoption, as users often prefer the convenience of a familiar "Login with..." button.