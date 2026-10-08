#CoreConcept #Security #BestPractice #API #OWASP #Vulnerability
> 
> These are the most common and dangerous vulnerabilities found in modern APIs. They are not infrastructure problems, but **logic flaws in your application code**. Understanding them is essential for building secure services.

---

> [!danger] The Core Scenario: The Malicious Insider
> An attacker has gained basic access to the building (they're logged in as a regular user). Now, instead of a frontal assault, they will try to exploit flaws in the **internal rules and systems** to escalate their privileges or access data they shouldn't.

---

### 1. Broken Object Level Authorization (BOLA / IDOR)

> This is the **#1 most common and severe** API vulnerability (also known as Insecure Direct Object Reference - IDOR).

-   **The Flaw:** The API correctly checks if a user is *authenticated*, but it fails to check if they are *authorized* to access the **specific piece of data** they are requesting.
-   **Analogy: The Nosy Employee**
    -   An employee, Alice, logs into the internal HR portal. She clicks a link to view her own performance review at `/api/reviews/123`. The system correctly validates that she is a logged-in employee.
    -   Now, curious, she manually types `/api/reviews/124` into her browser's address bar.
    -   The server responds with the performance review for user #124—the company's CEO.
    -   The system never asked the crucial second question: **"Is Alice allowed to see review #124?"**
-   **Prevention:** For every single API endpoint that takes an ID (e.g., `/users/{id}`, `/documents/{id}`), you **MUST** perform an explicit ownership check in your service layer or use method-level security.
    -   **Code Example:** `@PreAuthorize("#reviewId.owner == authentication.name or hasRole('ADMIN')")`

### 2. Mass Assignment

> **The Flaw:** Modern frameworks make it easy to automatically bind incoming JSON data from a request directly to a database object. An attacker exploits this convenience by adding extra, malicious fields to the JSON payload that they are not supposed to control.

-   **Analogy: The Unwitting Clerk**
    -   The HR portal has a form for updating your profile with fields for "PhoneNumber" and "Address." When you submit it, the backend clerk has a simple rule: "Take every field from the submitted form and blindly copy it onto the employee's official record."
    -   An attacker uses a web proxy tool to intercept their own request and adds a hidden field to the JSON: `"isAdmin": true`.
    -   The unsuspecting, automated clerk copies this new field along with the others, and the attacker is instantly promoted to an administrator.
-   **Prevention:** **Never trust incoming data.** Use the **[[DTOs|Data Transfer Object (DTO) Pattern]]**. Create a separate, dedicated class (e.g., `ProfileUpdateDTO`) that represents *only* the data you expect and allow to be changed. Manually map the fields from this DTO to your internal database entity.

### 3. Excessive Data Exposure

> **The Flaw:** The API returns an entire internal database object, including sensitive fields that the client application doesn't need, relying on the client to filter what it shows to the user.

-   **Analogy: The Oversharing HR Department**
    -   You use the company mobile app to view a colleague's profile to get their name and team. The app calls the API endpoint `/api/users/456`.
    -   To be "helpful," the backend developer just serialized the entire internal `User` object from the database and sent it back.
    -   The JSON response contains not only the `name` and `team`, but also their `passwordHash`, `homeAddress`, `salary`, and `dateOfBirth`.
    -   The mobile app correctly only displays the name and team, but a curious attacker using their browser's developer tools can see **all of it**.
-   **Prevention:** Just like with Mass Assignment, use **[[DTOs]]** to precisely shape your API's output. Create specific response DTOs (e.g., `UserProfileDTO`, `UserAdminViewDTO`) and explicitly define what data should be returned for each use case.

---

## Why This Matters to a Developer

These are not abstract infrastructure problems that someone else will solve; they are **bugs in your application code** that you are directly responsible for preventing.

-   **✔️ It's Your Direct Responsibility:** Writing secure code means being constantly vigilant about these logical patterns. You must actively think, "For this endpoint, am I checking ownership? Am I only accepting the fields I expect? Am I only returning the fields I intend?"

-   **✔️ Build Hardened APIs:** Understanding these common attack vectors allows you to build APIs that are not just functional but also hardened against the most prevalent and dangerous real-world attacks. This is a hallmark of a senior, professional developer.