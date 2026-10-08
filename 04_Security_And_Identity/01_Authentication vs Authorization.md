#CoreConcept #Security #Authentication #Authorization #BestPractice

> This is the most fundamental concept in application security. **Authentication (AuthN)** is about verifying *who you are*. **Authorization (AuthZ)** is about determining *what you are allowed to do*.

---

## 1. Authentication (AuthN): "Who are you?"
> **Authentication** is the process of verifying that a user or system is who they claim to be. It's the act of proving an identity.

> [!note] The Analogy: The Security Guard at the Front Door
> This is the security guard at the main entrance of the corporate headquarters.
> -   You present your employee ID badge or a government-issued ID.
> -   The guard checks that your face matches the photo and that the ID is valid.
>
> At this point, the guard has **authenticated** your identity. You are now inside the building.

-   **Mechanisms:** Passwords, [[JWT]], API Keys, Biometrics, [[OAuth2]].
-   **In [[Spring Security]]:** This is handled by the `AuthenticationManager` and its `AuthenticationProvider`s.

## 2. Authorization (AuthZ): "What are you allowed to do?"

> **Authorization** is the process that happens *after* successful authentication. It determines if an authenticated user has the necessary permissions to perform a specific action or access a specific resource.

> [!note] The Analogy: The Keycard for Internal Doors
> Once you are inside the building, your employee keycard determines which doors you can open.
> -   Can you enter the server room? **Access Denied.**
> -   Can you enter the executive boardroom? **Access Denied.**
> -   Can you enter the 3rd-floor marketing department? **Access Granted.**
>
> The keycard system is **authorizing** your access to specific areas based on your assigned role (e.g., "Marketing Team Member").

-   **Mechanisms:** Roles (e.g., `ROLE_ADMIN`), Permissions (e.g., `CAN_DELETE_POSTS`), Access Control Lists (ACLs).
-   **In [[Spring Security]]:** This is handled by configuring access rules in the `SecurityFilterChain` or using method-level annotations like [[@PreAuthorize]].

---

## Why This Matters to a Developer

Confusing these two concepts is the root of many critical security vulnerabilities. Understanding and separating them is a hallmark of a professional developer.

-   **✔️ Clarity of Thought and Code:** You build better, more secure systems by strictly separating these two concerns. The code that checks if a user is logged in (Authentication) should be completely separate from the code that checks if a logged-in user is an administrator (Authorization). This leads to a cleaner, more maintainable security model.

-   **✔️ Preventing Critical Vulnerabilities:** A very common and dangerous bug is to correctly check if a user is **authenticated** but completely forget to check if they are **authorized**.
    -   **The Classic Example:** An API endpoint like `DELETE /api/users/123` might correctly check for a valid login token, confirming the request is from an authenticated user (e.g., user #456).
    -   **The Failure:** If it doesn't *also* check that the logged-in user is either user #123 themselves or an administrator, then **any authenticated user could delete any other user's account**.
    -   By clearly distinguishing between AuthN and AuthZ, you are far less likely to make this critical mistake.