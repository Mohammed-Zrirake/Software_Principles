#API #REST #SystemDesign #BestPractice #WebDev #Contracts

> [!abstract] Brief Description
> A REST API is a contract between the server and the outside world. Designing clean, predictable, and resilient APIs requires adhering to established industry standards: resource-oriented URL modeling, strict HTTP verb semantics, robust versioning schemes, efficient pagination, and standardized error responses.

---

> [!note] 📖 The Core Analogy: The City Library Catalog
> A well-designed REST API is like a well-cataloged library:
> - **Resources (Nouns):** Books, authors, and sections have dedicated shelf addresses (`/authors`, `/authors/42/books`). You don't have shelves called `/get-book` or `/delete-author`.
> - **Actions (Verbs):** The visitor brings standard actions to the desk (`GET` to inspect, `POST` to donate a new book, `DELETE` to archive).
> - **Index Cards (Pagination):** When requesting 10,000 documents, the librarian doesn't dump them on your desk all at once; they hand you organized batches with a bookmark cursor for the next stack.

---

# 🏛️ 1. Resource-Oriented URL Design

REST is fundamentally **resource-oriented**, not action-oriented (unlike RPC).

### The Golden Rules of URI Modeling
1. **Use Nouns, Never Verbs:** The HTTP method is already the verb.
   - ❌ Bad: `POST /api/createUser`, `GET /api/getAllProducts`, `POST /api/deleteOrder/123`
   - ✅ Good: `POST /api/users`, `GET /api/products`, `DELETE /api/orders/123`
2. **Use Plural Nouns for Collections:**
   - Collection endpoint: `GET /api/v1/customers`
   - Single resource endpoint: `GET /api/v1/customers/894`
3. **Represent Hierarchies and Sub-Resources:**
   - All orders for a specific customer: `GET /api/v1/customers/894/orders`
   - Specific order of a customer: `GET /api/v1/customers/894/orders/12`
   - *Tip:* Avoid nesting deeper than 2 levels (e.g., avoid `/companies/1/departments/2/teams/3/employees/4`). Prefer top-level filtering: `GET /api/v1/employees?teamId=3`.

---

# 🚦 2. HTTP Verbs & Status Code Semantics

| Verb | Purpose | Idempotent? | Safe? | Success Status | Client Error Status |
| :--- | :--- | :---: | :---: | :--- | :--- |
| **`GET`** | Read resource | ✅ Yes | ✅ Yes | `200 OK` | `404 Not Found` |
| **`POST`** | Create new resource | ❌ No | ❌ No | `201 Created` (`Location` header) | `400 Bad Request` / `422 Unprocessable` |
| **`PUT`** | Full replacement of resource | ✅ Yes | ❌ No | `200 OK` or `204 No Content` | `400 Bad Request` / `404 Not Found` |
| **`PATCH`** | Partial modification of resource | ⚠️ Contextual | ❌ No | `200 OK` or `204 No Content` | `400 Bad Request` / `422 Unprocessable` |
| **`DELETE`**| Remove resource | ✅ Yes | ❌ No | `204 No Content` | `404 Not Found` |

> [!important] Safe vs. Idempotent
> - **Safe:** The operation does not modify server state (`GET`, `HEAD`). It is purely read-only.
> - **Idempotent:** Calling the operation once produces the exact same server state as calling it 100 times (`GET`, `PUT`, `DELETE`).

---

# 🔄 3. API Versioning Strategies

Breaking changes are inevitable as software evolves. An API must provide a graceful migration path for existing client consumers.

| Versioning Strategy | Example | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **1. URI Path** | `/api/v1/orders`<br>`/api/v2/orders` | • Highly visible & explicit<br>• Easy to test in browser/cURL<br>• Seamless CDN routing | • Violates URI purity (the resource itself didn't change identity) |
| **2. Query Parameter** | `/api/orders?version=2.0` | • Simple to implement<br>• Preserves base URL path | • Harder to cache at CDN layer<br>• Messy query strings |
| **3. Header / Content Negotiation** | `Accept: application/vnd.company.v2+json` | • Purest REST compliance<br>• Clean URLs | • Difficult to test via browser<br>• Requires complex client configurations |

> [!tip] Industry Standard
> **URI Path Versioning (`/api/v1/...`)** is by far the most widely adopted standard across Google, Microsoft, Stripe, and AWS due to clarity and operational simplicity.

---

# 📑 4. Pagination Strategies: Offset vs. Keyset (Cursor)

Never return unbounded collections (`SELECT * FROM Orders`). Returning 50,000 rows causes memory spikes, network serialization latency, and database query locks.

```
Offset Pagination:  [1][2][3][4] ... [Page 500]  ──► DB scans and discards 10,000 rows!
Cursor Pagination:  [Next: "ord_9842"]           ──► DB jumps directly via index (WHERE id > 9842)!
```

### Approach A: Offset-Based Pagination
- **Format:** `GET /api/orders?page=5&pageSize=20` (SQL: `OFFSET 80 LIMIT 20`)
- **Pros:** Trivial to implement, allows jumping directly to "Page 15".
- **Cons:**
  1. **Performance degradation:** As offset increases, the database must scan and discard thousands of rows ($O(N)$ complexity).
  2. **Page Drift Bug:** If a new row is inserted while the user flips from Page 1 to Page 2, the user sees duplicate items.

### Approach B: Keyset / Cursor-Based Pagination
- **Format:** `GET /api/orders?cursor=eyJpZCI6ODk0Mn0&limit=20` (SQL: `WHERE id > 8942 ORDER BY id ASC LIMIT 20`)
- **Pros:**
  1. Constant $O(1)$ performance even on tables with billions of rows (leverages index seeks).
  2. Immune to insertion and deletion drift.
- **Cons:** Cannot jump directly to an arbitrary page (must traverse sequentially forward/backward). Perfect for infinite scrolling and mobile feeds.

---

# 🚨 5. Standardized Error Handling: RFC 7807 (ProblemDetails)

Returning custom, inconsistent JSON errors across different endpoints frustrates client developers:
```json
// ❌ INCONSISTENT / AD-HOC ERROR
{ "error": true, "msg": "Something went wrong", "code": 901 }
```

### The Modern Standard: RFC 7807 `ProblemDetails`
The IETF standardized error payloads across HTTP APIs using the `application/problem+json` media type:

```json
// ✅ STANDARDIZED RFC 7807 ERROR SPECIFICATION
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient Funds",
  "status": 400,
  "detail": "Account balance of $42.00 is insufficient for transaction amount of $100.00.",
  "instance": "/api/v1/accounts/123/transfers/txn_984"
}
```

---

# 💻 Technology Illustration (.NET / C#)

In modern C# (ASP.NET Core Minimal APIs), implementing these standards is built into the framework:

```csharp
// Clean RESTful route with versioning, pagination, and RFC 7807 ProblemDetails:
app.MapGet("/api/v1/orders", async (int? cursor, int pageSize, IOrderRepository repo) =>
{
    if (pageSize > 100)
    {
        // Automatically returns RFC 7807 ProblemDetails JSON:
        return Results.Problem(
            detail: "Page size cannot exceed 100 items.",
            statusCode: StatusCodes.Status400BadRequest,
            title: "Validation Error"
        );
    }

    // High-performance Keyset Pagination:
    var orders = await repo.GetOrdersAfterCursorAsync(cursor ?? 0, pageSize);
    return Results.Ok(orders);
})
.WithName("GetOrders")
.Produces<IEnumerable<OrderDto>>(StatusCodes.Status200OK)
.ProducesProblem(StatusCodes.Status400BadRequest);
```

---

# 💡 Why This Matters to a Developer

1. **Client Predictability:** Frontend and mobile developers can write standardized API client interceptors that handle authentication, pagination, and error dialogs universally across all endpoints.
2. **Database Protection:** Using Keyset pagination and hard limits on `pageSize` prevents malicious or buggy clients from bringing down your production database with heavy unbounded queries.
3. **Clean Evolution:** Proper versioning allows you to deprecate old contracts without breaking existing mobile clients that haven't updated their app.
