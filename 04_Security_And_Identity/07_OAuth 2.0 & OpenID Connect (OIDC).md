#Security #Authentication #Authorization #OAuth #OIDC #CoreConcept #Architecture

> **OAuth 2.0** is the industry standard for **delegated authorization**. It allows a user to grant one application limited permission to access their data in another application *without* sharing their password. **OIDC** is a thin layer built on top of OAuth 2.0 that adds **authentication** and user identity information.

---

> [!note] 📖 The Core Analogy: The Hotel Valet Key
> Think of OAuth 2.0 as the system a modern hotel uses for its valet service.
> -   **You (The Resource Owner):** The owner of the car.
> -   **Your Car (The Resource Server):** Contains your valuable belongings and can be driven.
> -   **The Valet Service (The Client):** The third-party application that wants to perform an action (park your car).
> -   **The Hotel Front Desk (The Authorization Server):** The trusted entity that knows you're a registered guest and manages all the keys.
>
> **The Problem:** You don't want to give the valet your master key, which can open the trunk and glove box and start the car forever (your username and password).
>
> **The OAuth 2.0 Flow:**
> 1.  You go to the **Front Desk** and prove your identity.
> 2.  You ask them to grant the **Valet** a specific, limited permission (`scope`): "permission to park the car."
> 3.  The Front Desk gives the Valet a special, temporary **"valet key" (the Access Token)**.
> 4.  This key can only start the car and lock/unlock the doors. It cannot open the trunk, and it might expire after a few hours.
> 5.  You can tell the Front Desk to deactivate that key at any time (**revocable access**).

---

## 🔑 The Key OAuth 2.0 Terminology (The Cast of Characters)

-   **Resource Owner:** The user. You are the owner of your data and account.
-   **Client:** The application that wants to access your data (e.g., the Next.js app, Postman).
-   **Authorization Server:** The application that authenticates the user and issues access tokens (e.g., the "Identity Server," Google, GitHub).
-   **Resource Server:** The API or service that the Client wants to access (e.g., an auction service API, Google Calendar API).
-   **Redirect URI (Callback URL):** The URL where the user is sent back after they grant permission at the Authorization Server.
-   **Response Type:** The type of credential the Client expects. For the most secure flow, this is `code` (an Authorization Code).
-   **Scope:** The specific, granular permissions the Client is requesting (e.g., `read:profile`, `write:calendar`).
-   **Consent Form:** An optional form where the user explicitly approves the scopes requested by the Client.
-   **Client ID:** A public, unique identifier for the Client application.
-   **Client Secret:** A confidential password known only by the Client and Authorization Server, used for secure back-channel communication.
-   **Authorization Code:** A temporary, one-time-use code sent to the Client after the user grants consent.
-   **Access Token:** The final key that the Client uses to make authenticated requests to the Resource Server on behalf of the user.

---

## 📜 The OAuth 2.0 Authorization Code Flow

This is the most common and secure OAuth 2.0 flow, designed for web applications with a secure backend server.

1.  **Initial Registration:** Before the flow starts, the Client application registers with the Authorization Server, receiving a `Client ID` and `Client Secret`.
2.  **User Redirection:** The Client redirects the user's browser to the Authorization Server's login page. The request includes the `Client ID`, `Redirect URI`, `Response Type` (as `code`), and the desired `Scopes`.
3.  **User Authentication & Consent:** The Authorization Server authenticates the user (e.g., asks for a username/password). If it's the first time, it may show a consent form asking the user to approve the requested scopes.
4.  **Authorization Code is Sent:** After successful authentication, the Authorization Server redirects the user's browser back to the Client's `Redirect URI`, including the temporary **Authorization Code** in the URL.
5.  **Token Exchange (Back-Channel):** The Client's **backend server** receives the Authorization Code. It then makes a direct, private, server-to-server request to the Authorization Server, sending its `Client ID`, `Client Secret`, and the `Authorization Code`.
6.  **Access Token is Issued:** The Authorization Server validates all the information. If correct, it issues an **Access Token** and sends it back to the Client's server.
7.  **Accessing Protected Resources:** The Client's server can now use this Access Token to make authenticated API calls to the Resource Server.
8.  **Data is Returned:** The Resource Server validates the Access Token. If valid, it processes the request and returns the protected data to the Client's server.

> [!warning] Critical Security Note
> The **Access Token** is a sensitive credential. In the Authorization Code Flow, it is **never** sent to the user's browser. It lives securely on the Client's backend server.

---

## 🆔 OpenID Connect (OIDC): The Identity Layer

OIDC is a thin, interoperable layer built on top of OAuth 2.0 that adds user identity information.

-   **Limitation of OAuth 2.0:** OAuth 2.0 is purely for **authorization** (what the user *can do*). It does not, by itself, tell the Client *who the user is*.
-   **Function of OIDC:** OIDC adds a standardized authentication process. It introduces a special type of token called an **ID Token** (usually a JWT), which contains verifiable information about the user, such as their ID, email, and name.

> [!tip] Identity Provider (IdP)
> When an Authorization Server supports OIDC, it is often called an **Identity Provider (IdP)** because it provides the Client with verifiable identity information about the user, not just an access key.
>
> Most "Login with Google/Facebook/Apple" features use **OIDC** because the application needs both to **log you in (Authentication)** and to **access your profile information (Authorization)**.

---

> [!summary] Key Takeaways
> - **OAuth 2.0 = Authorization:** It's about granting permission for one app to access resources in another on your behalf.
> - **OIDC = Authentication + Identity:** It's a layer on top of OAuth 2.0 that handles the login process and provides information about *who* the user is.
> - The **Authorization Code Flow** is the standard, secure method for web applications, ensuring that sensitive tokens are never exposed to the user's browser.
> - The core principle is **delegated access without sharing your primary credentials** (username and password).