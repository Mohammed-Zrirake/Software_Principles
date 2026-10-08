#SpringBoot #Security #Cryptography #Password #CoreConcept

Never store passwords in plain text. Use a modern, slow, and salted hashing algorithm like **Bcrypt**. Spring Security's `BCryptPasswordEncoder` handles all the complexity for you, making it the industry standard for Java applications.

---
## 1. Hashing: The One-Way Street

> **Hashing** is a one-way cryptographic function that converts an input (like a password) into a fixed-length string of characters, called a "hash." It is computationally infeasible to reverse the process—you can't get the original password back from its hash.

-   **Purpose:** To store a representation of a password without storing the password itself. When a user logs in, you hash their submitted password and compare it to the stored hash.

### The Problem with Simple, Fast Hashing
Older algorithms like **MD5** and **SHA-1/256/512** were designed to be extremely fast. This is a major vulnerability.

> [!danger] Fast Hashing is Weak Hashing
> If an attacker gets ahold of your database of hashed passwords, they can use modern hardware to hash billions of common passwords per second, quickly finding matches. This is a **brute-force attack**.

---

## 2. Salting: Defeating Rainbow Tables

> **Salting** is the process of adding a unique, random string (the "salt") to each password *before* it is hashed. This salt is then stored alongside the hash.
### The Problem: Pre-computed Hashes (Rainbow Tables)
Attackers don't just brute-force; they use pre-computed lists of hashes for common passwords, called **rainbow tables**. If two users have the same password, like `password123`, a simple hash function will produce the same hash for both. An attacker only needs to crack it once.
### The Solution: Unique Salts
Salting ensures that even if two users have the identical password, their stored hashes will be completely different because they have different salts.

-   `hash("password123" + "salt_A")` → `hash_A`
-   `hash("password123" + "salt_B")` → `hash_B`

This renders rainbow tables useless, as the attacker would need a separate table for every possible salt, which is computationally impossible.

---
## 3. Bcrypt: The Modern Standard

> **Bcrypt** is a password hashing function designed specifically for securely storing passwords. It solves the problems of both fast hashing and manual salting.

### Key Features of Bcrypt

✔️ **It's Intentionally Slow**
Bcrypt has an adjustable "cost factor" or "work factor" that controls how many rounds of hashing it performs. You can increase this factor as computers get faster, ensuring the hashing process remains slow enough to thwart brute-force attacks.

✔️ **It Includes Automatic Salting**
You don't have to generate or manage salts yourself. Bcrypt automatically generates a cryptographically secure random salt for each password and embeds it directly into the final hash string.

### How to Use It in Spring Security
Spring Security provides the `BCryptPasswordEncoder` class, which is the recommended implementation.

**1. Define the Encoder as a Bean:**
```java
@Configuration
public class SecurityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**2. Use It to Encode and Match Passwords:**
```java
// In your user registration service
String rawPassword = "password123";
String hashedPassword = passwordEncoder.encode(rawPassword);
// The 'hashedPassword' is what you store in the database.

// Spring Security's AuthenticationProvider will use it automatically
// to match passwords during login.
boolean isMatch = passwordEncoder.matches("password123", hashedPassword); // returns true
```

### Anatomy of a Bcrypt Hash
A Bcrypt hash contains all the information needed to verify a password: the algorithm, the cost factor, the salt, and the hash itself.

`$2a$10$WZV8cOjMlyb0XxXlB2R2G.GJlE1ON7R/3pbJfn5NYlvFQkXU6z7eG`

-   `$2a$`: The Bcrypt algorithm version.
-   `$10$`: The cost factor (2^10 rounds).
-   `WZV8cOjMlyb0XxXlB2R2G.`: The 16-byte salt, Base64 encoded.
-   `GJlE1ON7R/3pbJfn5NYlvFQkXU6z7eG`: The 23-byte hash, Base64 encoded.

---

> [!summary] Final Recommendation
>| Technique | One-Way? | Auto-Salted? | Secure Against Brute-Force? | Recommended for Passwords? |
>|:---|:---:|:---:|:---:|:---:|
>| MD5 / SHA-1 | ✅ | ❌ | ❌ (Too fast) | **NO** |
>| SHA-256 / 512 | ✅ | ❌ | ❌ (Too fast) | **NO** |
>| **Bcrypt** | ✅ | ✅ | ✅ (Intentionally slow) | **YES** |
>
> **Always use `BCryptPasswordEncoder` for password storage in modern applications.**