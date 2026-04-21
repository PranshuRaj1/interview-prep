# ACID Transactions - Interview Prep

## Question: Bank Transfer Safety & ACID Properties

You're building a financial transactions system. A user initiates a bank transfer — money needs to be debited from Account A and credited to Account B. Walk through how a database transaction guarantees this operation is safe. Specifically, explain what ACID means — not just the definitions, but what actually breaks if any one of those four properties is violated in this exact scenario.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';
UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';
COMMIT;
```

---

### My Initial Answer

#### A — Atomicity (All or Nothing)
*   **What should happen:** Either both debit (A) and credit (B) happen, or neither happens.
*   **If it breaks:** If ₹1000 is deducted from A but the system crashes before crediting B, the money effectively disappears. This leads to financial corruption where the bank cannot reconcile its books.
*   **How it's prevented:** DB uses Write Ahead Logs (WAL). If a crash happens, it triggers a rollback on restart.

#### C — Consistency (Rules must hold)
*   **What should happen:** Rules like "total money in system remains constant" or "balance should never go negative" must hold.
*   **If it breaks:** If A only had ₹500 but the system allows deducting ₹1000, A reaches an invalid state (-₹500). This violates auditing rules and makes the system untrustworthy.
*   **How it's prevented:** DB enforces this via constraints (CHECK, Foreign Keys, etc.) and application logic.

#### I — Isolation (No interference)
*   **What should happen:** Multiple transfers shouldn't interfere with each other.
*   **If it breaks:**
    1.  **Dirty Read:** T2 reads A's reduced balance before T1 commits. If T1 rolls back, T2 saw money that never existed.
    2.  **Lost Update:** T1 and T2 both read ₹5000, both deduct money, and one overwrite's the other's result (e.g., final balance is ₹4000 instead of ₹3500).
    3.  **Double Spend:** Two simultaneous transfers both see A has ₹1000 and both succeed in sending ₹1000.
*   **How it's prevented:** Locks (row-level), Isolation levels, and MVCC (Multi-Version Concurrency Control).

#### D — Durability (Committed stays committed)
*   **What should happen:** Once the user sees "Success," the data must survive any subsequent crash.
*   **If it breaks:** If the system crashes right after a successful commit and the data is lost, the state becomes unpredictable, leading to massive customer disputes and legal issues.
*   **How it's prevented:** Writes to disk (not just memory). WAL logs are flushed to disk before the commit is acknowledged.

---

### What I Missed & Refinement

#### 1. ACID Consistency vs. CAP Consistency
In ACID, "Consistency" is often the most misunderstood property. 
*   **ACID Consistency**: Is about **valid state transitions**. The DB ensures that rules (constraints, keys) you declared are followed. It is primarily the developer's responsibility to define what "valid" means.
*   **CAP Consistency**: Is about **all nodes seeing the same data** at the same time in a distributed system. 
*   *Key takeaway:* In an interview, clarifying that ACID consistency is about integrity constraints rather than distributed state shows senior-level depth.

#### 2. Mapping Isolation Levels to Failures
A strong answer maps the specific failures to the levels that prevent them:
*   **Dirty Read**: Prevented at `READ COMMITTED` and above.
*   **Lost Update**: Prevented at `REPEATABLE READ` and above.
*   **Double Spend (Phantom Read/Write Skew)**: Only fully prevented at `SERIALIZABLE`.

#### 3. The Production Pattern: Pessimistic Locking
In high-throughput systems (like banks), relying solely on `SERIALIZABLE` isolation is rare because it essentially executes transactions sequentially, killing performance. Instead, we use **Explicit Pessimistic Locking** with `SELECT FOR UPDATE`:

```sql
BEGIN;
-- Explicitly lock the row before doing anything
SELECT balance FROM accounts WHERE id = 'A' FOR UPDATE;

-- Now no other transaction can touch this row until we commit
UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';
UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';
COMMIT;
```
This is the standard production pattern: using a lower isolation level (like Read Committed) combined with explicit locks on critical rows.

---

### Summary Table

| Property | If it breaks | Real-world Result |
| :--- | :--- | :--- |
| **Atomicity** | Partial execution | Money disappears |
| **Consistency** | Invalid rules | Negative balances / wrong totals |
| **Isolation** | Concurrency issues | Double spending / wrong balances |
| **Durability** | Data loss after commit | "Successful" transfers vanish |
