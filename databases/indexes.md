# Database Indexes - Interview Prep

## Question: Optimizing a Slow Query on a Large Table

You have a `users` table with 10 million rows. A query like this is running slow:

```sql
SELECT * FROM orders WHERE user_id = 123 AND status = 'pending' ORDER BY created_at DESC;
```

You add an index on `user_id`. The query is still slow. Walk through the complete thought process for diagnosing and fixing this — from understanding why the single-column index isn't enough, to what the ideal index looks like, to any tradeoffs you'd consider.

---

### My Initial Answer
Creating an index will solve only 1/3rd of the problem because there are other two thing which also need to taken into account like status = 'pending' and sorting by `created_at` DESC.

So what I propose is creating a new composite index with `(user_id, status, created_at DESC)`.
Here, instead of the database doing a separate sort operation, we make the index already store data in that order.

`user_id` comes first because it is more selective than other two, sorting is done not in memory.

**Tradeoffs to consider:**
1. **Write performance**: Every insert/update must update this index. Composite indexes are heavier.
2. **Storage cost**: Large index for 10M rows.
3. **Over-indexing risk**: Too many indexes = slower writes + memory pressure.

---

### What I Missed & Refinement

While the core intuition of using a composite index was correct, several critical nuances were missing:

#### 1. Column Order Reasoning
The reasoning that `user_id` comes first because it's "more selective" is only partially correct. In a composite index, the order should follow the **Equality, Sort, Range** rule:
*   Columns used in **equality filters** (`user_id = 123` and `status = 'pending'`) should come first.
*   The DB then selects based on cardinality among equality filters (putting the more selective one first helps narrow the scan early).
*   The `created_at` column comes last for the **ordered tail**, allowing the database to skip a manual sort step.

#### 2. Index-Only Scans & Covering Indexes
Using `SELECT *` forces the database to perform "heap fetches" (looking up the actual table rows) even if the index finds the correct entries. 
*   **Fix:** Only select needed columns or use a **Covering Index** (e.g., in Postgres: `INCLUDE(other_cols)`) to allow an **Index-only scan**.

#### 3. Diagnosis with `EXPLAIN ANALYZE`
A senior approach starts with diagnosis. Before changing anything, run `EXPLAIN ANALYZE`. This confirms:
*   Whether the DB is doing a sequential scan vs. index scan.
*   Actual row estimates vs. real rows.
*   If a "sorting" step is happening in memory/disk.

#### 4. Partial Indexes
If `status = 'pending'` has low cardinality (few distinct values) and is a very common query filter, a **Partial Index** is often better:
```sql
CREATE INDEX idx_orders_pending ON orders(user_id, created_at DESC) 
WHERE status = 'pending';
```
This index is significantly smaller and faster because it only contains pointers to "pending" rows.

---

### The Ideal Answer (Summary)
The solution is a composite index on `(user_id, status, created_at DESC)`. However, the full process involves:
1.  **Diagnosing** with `EXPLAIN ANALYZE` to confirm the bottleneck.
2.  **Ordering** columns by Equality, Sort.
3.  **Optimizing** for "Index-only scans" by avoiding `SELECT *`.
4.  **Considering a Partial Index** if the filter is on a low-cardinality column like `status = 'pending'`.
5.  **Balancing tradeoffs** like write amplification and storage overhead.
