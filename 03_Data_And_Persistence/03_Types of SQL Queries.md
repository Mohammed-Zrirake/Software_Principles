#SQL #Database #CoreConcept

>  [[SQL]] commands are categorized into five distinct groups based on their function: querying data (DQL), changing data (DML), defining database structure (DDL), managing permissions (DCL), and controlling transactions (TCL).

---

## 1. DQL (Data Query Language)
> **Purpose:** To retrieve data from the database. It's the language of asking questions. This is the most frequently used category in day-to-day application development.

-   **Core Command:** `SELECT`

**Example:**
```sql
-- Retrieve the name and email for all users located in Canada.
SELECT full_name, email 
FROM users 
WHERE country = 'Canada';
```

---

## 2. DML (Data Manipulation Language)
> **Purpose:** To manipulate the data *within* existing database objects. This includes creating new records, updating existing ones, and deleting them.

-   **Core Commands:** `INSERT`, `UPDATE`, `DELETE`

**Examples:**
```sql
-- Add a new user to the 'users' table.
INSERT INTO users (username, password_hash) 
VALUES ('alex_p', 'hashed_password_xyz');

-- Change the email for a specific user.
UPDATE users 
SET email = 'alex.p@newdomain.com' 
WHERE user_id = 101;

-- Remove a specific user from the table.
DELETE FROM users 
WHERE user_id = 101;
```

---

## 3. DDL (Data Definition Language)
> **Purpose:** To define, modify, and destroy the structure of database objects themselves, such as tables, indexes, and schemas.

-   **Core Commands:** `CREATE`, `ALTER`, `DROP`, `TRUNCATE`

**Examples:**
```sql
-- Create a new table to store products.
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2)
);

-- Add a new column to the existing 'products' table.
ALTER TABLE products 
ADD COLUMN stock_quantity INT;

-- Completely remove the 'products' table and all its data.
DROP TABLE products;
```

> [!warning] `DROP` vs. `TRUNCATE`
> - **`TRUNCATE TABLE products;`**: Deletes **all rows** from the table quickly but leaves the table structure intact.
> - **`DROP TABLE products;`**: Deletes **all rows AND the table structure itself**. The table will no longer exist.

---

## 4. DCL (Data Control Language)
> **Purpose:** To manage access rights and permissions for database users. It controls who can do what.

-   **Core Commands:** `GRANT`, `REVOKE`

**Examples:**
```sql
-- Allow the 'analyst' user to read data from the 'orders' table.
GRANT SELECT ON orders TO analyst;

-- Remove the 'analyst' user's permission to update the 'orders' table.
REVOKE UPDATE ON orders FROM analyst;
```

---

## 5. TCL (Transaction Control Language)
> **Purpose:** To manage transactions within the database, ensuring data integrity by grouping a set of operations into a single atomic unit.

-   **Core Commands:** `COMMIT`, `ROLLBACK`, `SAVEPOINT`

**Example Scenario:** Transferring money between two bank accounts.
```sql
START TRANSACTION;

-- Step 1: Deduct $100 from Account A.
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';

-- Step 2: Add $100 to Account B.
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';

-- If both steps are successful, make the changes permanent.
COMMIT;

-- If an error occurred during the process, undo all changes.
-- ROLLBACK; 
```

---

> [!summary] Key Takeaways
> - **DQL** (`SELECT`): Reads/queries data.
> - **DML** (`INSERT`, `UPDATE`, `DELETE`): Modifies rows of data.
> - **DDL** (`CREATE`, `ALTER`, `DROP`): Manages the structure of database objects.
> - **DCL** (`GRANT`, `REVOKE`): Controls user permissions.
> - **TCL** (`COMMIT`, `ROLLBACK`): Manages transactions.