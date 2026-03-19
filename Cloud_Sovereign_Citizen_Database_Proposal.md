# Secure Cloud Migration of Citizen Database
## Minimising Patriot Act Risk While Preserving PL/SQL Investment

---

## Executive Summary

This document proposes an architecture for migrating a citizen database to AWS while ensuring that sensitive personal data cannot be accessed by the cloud provider under any circumstances, including legal compulsion via US legislation such as the Patriot Act or CLOUD Act.

The approach is built on a simple but powerful principle: the cloud stores data, but cannot understand it. All encryption keys and the ability to interpret sensitive information remain exclusively within a trusted environment under UK jurisdiction. AWS sees only ciphertext and mathematical transformations that cannot be reversed without access to keys that never leave our control.

Critically, this design preserves the vast majority of existing PL/SQL logic. Rather than rewriting decades of database code, we introduce a thin layer of views and wrapper functions that allow PL/SQL to continue operating while transparently routing sensitive operations to the trusted environment.

---

## 1. Understanding the Challenge

### The Sovereignty Problem

Moving citizen data to AWS creates a specific legal exposure. Under US law—particularly the Patriot Act and the CLOUD Act—US authorities can compel American companies to provide access to data stored on their infrastructure, regardless of where that data is physically located. This means that even if we use AWS's European data centres, the data remains potentially accessible to US government demands.

For a citizen database containing names, National Insurance numbers, dates of birth, addresses, and family relationships, this exposure is unacceptable. The question becomes: how do we gain the operational benefits of cloud infrastructure while ensuring that even with full cooperation from AWS, no meaningful access to citizen data is possible?

### The PL/SQL Problem

The existing database contains substantial PL/SQL logic developed over many years. This code handles complex business rules, data validation, reporting, and integration with other systems. A naive approach to encryption—such as encrypting all data at the application layer—would break this code entirely. PL/SQL procedures that compare names, join on identifiers, or filter by date of birth would all fail because they cannot operate on encrypted values.

The traditional answer to this problem is to rewrite everything. Move the logic out of the database, rebuild it in application code, and treat the database as pure storage. This is a multi-year undertaking that carries enormous risk and cost.

### The Apparent Contradiction

These two requirements seem to be in direct conflict. If data must be encrypted so AWS cannot read it, the database cannot process it. If the database must process it, it must be able to read it. The architecture proposed here resolves this contradiction.

---

## 2. The Core Architectural Insight

The solution rests on a fundamental separation: AWS stores protected representations of data, but the ability to interpret those representations exists only outside AWS.

Think of it like storing a locked safe in a warehouse. The warehouse operator can move the safe, stack it, even tell you how heavy it is—but without the key, they cannot open it. In our architecture, AWS is the warehouse. The safe is the encrypted data. The key never enters the warehouse.

But we go further than simple encryption. We need the database to be able to do useful work on this protected data—filtering, joining, searching—without ever seeing the actual values. This requires different protection strategies for different types of operations.

### Three Categories of Data

Every field in the database falls into one of three categories, each requiring a different approach:

**Fields that must be read** (Name, Date of Birth, Address): These are encrypted using strong symmetric encryption. The encrypted values are stored in the database, but decryption can only happen by calling out to a trusted service that holds the keys. When PL/SQL needs to display a name or validate a date of birth, it calls a wrapper function that reaches outside AWS to perform the decryption.

**Fields that must be matched exactly** (NINO, Citizen ID, Relationship links): These are transformed using a keyed hash called HMAC. The original value cannot be recovered from the hash, but importantly, the same input always produces the same output. This means the database can still perform equality comparisons and joins—it just compares hash values instead of the original identifiers. The crucial detail is that the hash requires a secret key. Without that key (which never leaves the trusted environment), an attacker cannot compute hashes for comparison or brute-force the original values.

**Fields that must support fuzzy matching** (Names for approximate search): Searching for "Alic Smth" and finding "Alice Smith" requires a completely different approach. We cannot encrypt these values because encryption destroys the ability to compare similarity. Instead, we generate mathematical fingerprints—called Bloom filters—that capture the structural patterns in names without revealing the actual text. Combined with phonetic encoding and a technique called locality-sensitive hashing, these fingerprints allow the database to efficiently identify potential matches without ever seeing the original names.

---

## 3. The Trusted Adapter: The Heart of the Architecture

At the centre of this design is a service we call the Trusted Adapter. This is a dedicated component that runs entirely outside AWS, in infrastructure we control completely—either on-premise or in a sovereign UK cloud environment with no US jurisdiction exposure.

The Trusted Adapter has one job: it holds all the secrets and performs all the sensitive operations. It provides three core capabilities:

**Encryption and Decryption**: When data needs to be written, the adapter encrypts it before it reaches AWS. When data needs to be read, the adapter receives the encrypted value and returns the plaintext. The keys never leave the adapter; they are typically stored in Hardware Security Modules (HSMs) that make extraction physically impossible.

**HMAC Generation**: When an identifier needs to be stored or compared, the adapter computes the keyed hash. This allows the database to perform joins and lookups without ever knowing the actual identifier values.

**Fuzzy Artefact Computation**: When a name is stored, the adapter generates the phonetic codes, Bloom filters, and locality-sensitive hash buckets that enable approximate matching.

The database communicates with this adapter through Oracle's external procedure mechanism. From PL/SQL's perspective, it's simply calling a function. That function happens to reach out over a secure channel to the trusted environment, perform the sensitive operation, and return the result.

This is the critical insight: we are not asking PL/SQL to understand encryption or hashing. We are wrapping those operations in functions that look like ordinary database functions. The complexity is hidden behind a clean interface.

---

## 4. How Existing PL/SQL Continues to Work

This section addresses the most common concern: if data is encrypted, how can existing code still function?

### The View Layer

Rather than modifying every piece of PL/SQL that touches citizen data, we introduce a layer of database views. These views sit between PL/SQL and the underlying encrypted tables, presenting the data as if it were unencrypted.

For example, the base table `citizens` contains columns like `name_enc` (encrypted name) and `nino_hmac` (hashed NINO). We create a view called `citizens_v` that presents these as plain `name` and `nino` columns, using wrapper functions to perform decryption on access:

```sql
CREATE OR REPLACE VIEW citizens_v AS
SELECT 
  citizen_id_hmac,
  decrypt_name(name_enc) AS name,
  decrypt_dob(dob_enc) AS dob,
  phonetic,
  bloom,
  lsh_buckets
FROM citizens;
```

Existing PL/SQL that does `SELECT name FROM citizens WHERE ...` simply needs to change to `SELECT name FROM citizens_v WHERE ...`. In many cases, this can be accomplished through synonyms, requiring no code changes at all.

### How Synonyms Eliminate Code Changes

Oracle synonyms provide a powerful mechanism for redirecting table references without modifying any code. A synonym is essentially an alias—when code references the synonym, Oracle transparently redirects to the actual object.

Here's how we use this to our advantage:

**Step 1: Rename the original table**

The existing `citizens` table is renamed to `citizens_base` (or `citizens_protected`, or similar):

```sql
ALTER TABLE citizens RENAME TO citizens_base;
```

**Step 2: Create the protected view**

We create a view that presents decrypted data, using the original table name with a suffix:

```sql
CREATE OR REPLACE VIEW citizens_v AS
SELECT 
  citizen_id_hmac,
  decrypt_name(name_enc) AS name,
  decrypt_dob(dob_enc) AS dob,
  phonetic,
  bloom,
  lsh_buckets
FROM citizens_base;
```

**Step 3: Create a synonym pointing to the view**

We create a synonym with the original table name, pointing to the view:

```sql
CREATE OR REPLACE SYNONYM citizens FOR citizens_v;
```

Now, any existing PL/SQL that references `citizens` automatically gets the view instead. The code still says `SELECT name FROM citizens`, but Oracle resolves `citizens` to the synonym, which points to `citizens_v`, which decrypts from `citizens_base`.

**The result**: Existing PL/SQL continues to work without any text changes. The SELECT statements, the column references, the FROM clauses—all remain exactly as written. Only the underlying resolution has changed.

**Where synonyms don't help**: Synonyms redirect object references, but they don't transform query logic. A query that says `WHERE nino = :value` still needs to become `WHERE nino_hmac = hmac_nino(:value)` because the comparison logic itself must change. Synonyms handle the "which table" question; they cannot handle the "how to compare" question.

**Schema considerations**: Synonyms can be public (visible to all schemas) or private (visible only within a schema). For existing applications that reference tables with schema prefixes (e.g., `CITIZEN_SCHEMA.citizens`), you may need to ensure the synonym exists in the expected schema or use public synonyms.

### Wrapper Functions for Comparisons

When PL/SQL needs to look up a citizen by NINO, the logic changes slightly. Instead of:

```sql
SELECT * FROM citizens WHERE nino = :input_nino;
```

It becomes:

```sql
SELECT * FROM citizens WHERE nino_hmac = hmac_nino(:input_nino);
```

The `hmac_nino` function calls the trusted adapter to compute the keyed hash of the input value. The database then compares this hash against the stored hashes. The actual NINO value never appears in the database—neither the stored value nor the search value.

This is a real change to the code, but it is a mechanical, searchable change. Every occurrence of `WHERE nino = :value` becomes `WHERE nino_hmac = hmac_nino(:value)`. This can be identified and modified systematically.

### What Actually Changes

Let me be direct about what modifications are required:

**Table references become view references**: Using the synonym approach described above, this requires zero code changes. The original table is renamed, a decrypting view is created, and a synonym with the original table name points to the view. Existing PL/SQL continues to reference `citizens` and Oracle resolves it through the synonym chain.

**Equality comparisons on identifiers**: Any `WHERE identifier = :value` must change to use the HMAC wrapper. This is a mechanical change that can be largely automated, but it is a real code change—synonyms cannot help here because the comparison logic itself must be different.

**Display of sensitive values**: Any code that retrieves and uses encrypted fields works through the view layer, which handles decryption automatically.

**What does not change**: Business logic, control flow, error handling, transaction management, integration interfaces, reporting logic (other than field access), and the vast majority of procedural code. The core intellectual property embedded in the PL/SQL estate is preserved.

### Additional PL/SQL Patterns Requiring Attention

Beyond the core read patterns, PL/SQL commonly performs operations that need specific consideration in this architecture. This section examines each pattern in detail, including what it is, why it matters, how difficult it is to address, the recommended approach, and the performance implications.

#### INSERT, UPDATE, and DELETE Operations

**What this is**: The synonym-to-view approach described above handles SELECT operations elegantly—when code reads data, the view transparently decrypts it. However, views in Oracle are read-only by default. When PL/SQL attempts to write data (INSERT new records, UPDATE existing records, or DELETE records), a simple view will reject the operation with an error.

This matters because citizen data changes constantly: new citizens are registered, addresses change, relationships are updated, and records are occasionally removed.

**Difficulty**: Medium. The solution is well-understood and uses standard Oracle features, but requires careful implementation for each affected table.

**Recommended approach**: Use INSTEAD OF triggers. In Oracle, a trigger is a piece of code that automatically executes when a specified event occurs. An "INSTEAD OF" trigger is a special type that intercepts an operation on a view and replaces it with custom logic. When code tries to INSERT into the view, the trigger fires instead, allowing us to encrypt the incoming data before writing it to the base table.

```sql
CREATE OR REPLACE TRIGGER citizens_v_insert
INSTEAD OF INSERT ON citizens_v
FOR EACH ROW
BEGIN
  INSERT INTO citizens_base (
    citizen_id_hmac,
    name_enc,
    dob_enc,
    phonetic,
    bloom,
    lsh_buckets
  ) VALUES (
    hmac_citizen_id(:NEW.citizen_id),
    encrypt_name(:NEW.name),
    encrypt_dob(:NEW.dob),
    compute_phonetic(:NEW.name),
    compute_bloom(:NEW.name),
    compute_lsh(:NEW.name)
  );
END;
```

With this trigger in place, existing PL/SQL can still say `INSERT INTO citizens (name, dob, ...) VALUES (...)` and the trigger transparently encrypts, hashes, and computes fuzzy artefacts before writing to the base table. Similar triggers handle UPDATE and DELETE.

**Alternative approach**: Create wrapper stored procedures (e.g., `insert_citizen(...)`, `update_citizen(...)`) that handle the transformation. Existing code changes from direct DML to procedure calls. This is more intrusive but may be clearer for maintenance.

**Performance implications**: Each INSERT, UPDATE, or DELETE now incurs adapter calls for encryption and HMAC computation. For single-record operations (the common case in interactive use), this adds perhaps 10-50ms of latency—noticeable but acceptable. For bulk operations inserting thousands of rows, this becomes significant and requires the batch optimisation approaches described later. The INSTEAD OF trigger fires once per row, so a 1,000-row insert means 1,000 trigger executions, each calling the adapter. Plan for this in capacity sizing.

---

#### Triggers on the Original Table

**What this is**: Triggers are automated code that Oracle executes when data changes. Many production databases have triggers attached to important tables for purposes like:
- **Audit triggers**: Automatically log who changed what and when
- **Validation triggers**: Check that data meets business rules before allowing the change
- **Cascading triggers**: Automatically update related tables when a record changes
- **Integration triggers**: Send notifications to other systems when data changes

When we rename the original table and introduce a view, existing triggers may fire on the wrong object, receive encrypted data they cannot interpret, or fail entirely.

**Difficulty**: Medium. Each trigger must be reviewed individually to understand what it does and how it should work in the new architecture.

**Recommended approach**: 

1. **Inventory all triggers**: Run a query against the Oracle data dictionary to list every trigger on affected tables. Document what each one does.

2. **Categorise by type**:
   - Audit triggers: Modify to log non-sensitive fields only (timestamps, operation types, HMAC identifiers). Alternatively, move auditing to a separate layer that has adapter access.
   - Validation triggers: Move validation logic into the INSTEAD OF triggers on the view, where plaintext is still available before encryption.
   - Cascading triggers: Usually can continue working if they reference HMAC identifiers rather than plaintext values.
   - Integration triggers: May need adapter access to decrypt data before sending to external systems, or send protected data to systems that can handle it.

3. **Migrate triggers to the view**: INSTEAD OF triggers on the view replace triggers on the base table. The view trigger has access to the plaintext (the `:NEW` values being inserted) so it can perform validation and auditing before encrypting.

**Performance implications**: Minimal additional impact beyond the INSTEAD OF trigger overhead already discussed. Audit logging to the database adds the usual INSERT overhead. If audit triggers previously wrote plaintext and now write encrypted/HMAC values, the log table size may increase slightly due to longer hash values.

---

#### ORDER BY on Encrypted Columns

**What this is**: Queries often need to return results in a specific order—typically alphabetical by name for human readability. The SQL `ORDER BY name` clause tells the database to sort results.

When the name column contains encrypted ciphertext rather than plaintext, sorting becomes meaningless. Ciphertext has no relationship to the original alphabetical order—"Alice" encrypted might sort after "Zebra" encrypted purely by chance of the encryption algorithm.

**Difficulty**: Medium to Hard, depending on how frequently this pattern appears and how critical alphabetical ordering is to business processes.

**Recommended approach**: Several options exist, each with trade-offs:

1. **Sort after decryption** (simplest, limited scale): Retrieve the matching records with encrypted names, send to the trusted adapter for decryption, sort the decrypted results there, return to the application. This only works for small result sets—sorting 100 records externally is fine; sorting 100,000 is not.

2. **Precomputed sort keys** (scalable, some information leakage): Store an additional column containing a sortable representation of the name. This could be the phonetic code (coarse sorting—all "Smith" variants together), or a normalised/lowercased form that is stored unencrypted. This leaks some information (an attacker can see relative ordering) but may be acceptable if the risk is judged low.

3. **Eliminate the requirement**: Challenge whether alphabetical sorting is truly necessary. Can results be sorted by registration date, case number, or other non-sensitive attributes? Can pagination eliminate the need to sort large sets?

**What changes in the PL/SQL**:

*Existing code (before):*

A **stored procedure** is a reusable block of code saved in the database that applications can call by name. Think of it like a function in any programming language—it takes inputs, does work, and returns results. The `CREATE OR REPLACE PROCEDURE` statement defines (or redefines) such a procedure.

In this example, `get_citizens_by_region` is a procedure that:
- Takes an input parameter `p_region` (the `IN` keyword means it's an input)
- Returns results through `p_results` (the `OUT` keyword means it's an output)
- Uses a `SYS_REFCURSOR`, which is Oracle's way of returning a set of rows—essentially a pointer to query results that the calling application can iterate through

```sql
-- A procedure that returns citizens sorted alphabetically
CREATE OR REPLACE PROCEDURE get_citizens_by_region(
  p_region IN VARCHAR2,       -- Input: the region to search for
  p_results OUT SYS_REFCURSOR  -- Output: a set of matching rows
) AS
BEGIN
  -- OPEN ... FOR executes a query and assigns the results to the cursor
  OPEN p_results FOR
    SELECT citizen_id, name, dob, address
    FROM citizens
    WHERE region = p_region
    ORDER BY name;  -- This line is the problem
END;
```

*Option 1: Sort after decryption (for small result sets)*

The PL/SQL retrieves unsorted records; sorting moves to the calling application or a trusted-environment service:

```sql
CREATE OR REPLACE PROCEDURE get_citizens_by_region(
  p_region IN VARCHAR2,
  p_results OUT SYS_REFCURSOR
) AS
BEGIN
  OPEN p_results FOR
    SELECT citizen_id, name, dob, address
    FROM citizens                    -- This references the SYNONYM
    WHERE region = p_region;
    -- ORDER BY removed—sorting happens in calling application
END;
```

**Understanding "return via the view"**: When the procedure says `FROM citizens`, it's not actually reading from a table called `citizens`. Remember the three-layer architecture we set up:

1. The original `citizens` table was renamed to `citizens_base` (which stores encrypted data)
2. We created a view called `citizens_v` that reads from `citizens_base` and decrypts the data
3. We created a synonym called `citizens` that points to the view `citizens_v`

So when the procedure executes `SELECT ... FROM citizens`, Oracle resolves this as:
- `citizens` (synonym) → `citizens_v` (view) → `citizens_base` (table with encrypted data)

The view's definition includes the decryption calls, so by the time the data reaches the procedure, it's already been decrypted. The procedure doesn't need to know anything about encryption—it just queries "citizens" and gets back readable names, dates, and addresses. This is the magic of the synonym-to-view approach: **zero changes to the query itself**, yet decryption happens automatically behind the scenes.

The calling application (Java, .NET, etc.) must then sort the returned collection:

```java
// Java example: sort after retrieval
List<Citizen> citizens = citizenRepository.getByRegion(region);
citizens.sort(Comparator.comparing(Citizen::getName));  // Sort in application memory
```

*Option 2: Precomputed sort key (for large result sets needing database sorting)*

This approach requires a schema change—adding a new column to the table. `ALTER TABLE` modifies an existing table's structure:

```sql
-- Add column to base table
ALTER TABLE citizens_base ADD (name_sort_key VARCHAR2(100));
```

Next, we modify the **trigger** that handles inserts. Recall that a trigger is code that runs automatically when a database event occurs. An `INSTEAD OF` trigger on a view intercepts INSERT/UPDATE/DELETE operations and lets us control exactly what happens. The `:NEW` pseudo-record represents the incoming data—the values being inserted.

```sql
-- CREATE OR REPLACE TRIGGER defines automatic code that fires on database events
CREATE OR REPLACE TRIGGER citizens_v_insert
INSTEAD OF INSERT ON citizens_v    -- Fires when someone inserts into the view
FOR EACH ROW                        -- Fires once per row being inserted
BEGIN
  INSERT INTO citizens_base (
    citizen_id_hmac,
    name_enc,
    name_sort_key,  -- New column
    dob_enc,
    -- ... other columns
  ) VALUES (
    hmac_citizen_id(:NEW.citizen_id),     -- :NEW.citizen_id is the value being inserted
    encrypt_name(:NEW.name),               -- :NEW.name is the incoming name
    UPPER(SUBSTR(:NEW.name, 1, 100)),      -- Sortable key (plaintext, normalised)
    encrypt_dob(:NEW.dob),
    -- ...
  );
END;
```

Now the stored procedure changes to sort by the new column instead of the encrypted name:

```sql
CREATE OR REPLACE PROCEDURE get_citizens_by_region(
  p_region IN VARCHAR2,
  p_results OUT SYS_REFCURSOR
) AS
BEGIN
  OPEN p_results FOR
    SELECT citizen_id, name, dob, address
    FROM citizens
    WHERE region = p_region
    ORDER BY name_sort_key;  -- Changed from ORDER BY name
END;
```

**Security note**: The `name_sort_key` column is stored unencrypted. An attacker with database access can see the first 100 characters of names in uppercase. Assess whether this exposure is acceptable for your threat model. Alternatives include storing only the first letter (`SUBSTR(:NEW.name, 1, 1)`) for coarser sorting, or using the phonetic code.

*Option 3: Sort by non-sensitive attribute*

The simplest approach—just change the ORDER BY clause to reference a different column:

```sql
CREATE OR REPLACE PROCEDURE get_citizens_by_region(
  p_region IN VARCHAR2,
  p_results OUT SYS_REFCURSOR
) AS
BEGIN
  OPEN p_results FOR
    SELECT citizen_id, name, dob, address
    FROM citizens
    WHERE region = p_region
    ORDER BY registration_date DESC;  -- Changed to non-sensitive field
END;
```

This requires no schema changes but does require business agreement that alphabetical sorting is not essential.

**Performance implications**: Sorting after decryption adds network round-trips to the adapter and sorts data in application memory rather than the database. For small sets (<500 records), impact is minimal. For large sets, this can significantly increase response time and memory usage. Precomputed sort keys have no runtime penalty—they're just another indexed column—but require storage and maintenance.

---

#### GROUP BY and Aggregations

**What this is**: SQL aggregation operations summarise data: counting records, calculating averages, finding maximums, etc. The `GROUP BY` clause groups records by common values before aggregating. For example, `SELECT address, COUNT(*) FROM citizens GROUP BY address` counts how many citizens share each address.

When values are encrypted, grouping fails. Each encryption of "123 High Street" produces different ciphertext (due to initialisation vectors in secure encryption), so the database sees them as different values and groups them separately, producing wrong counts.

**Difficulty**: Hard. This often requires rethinking how reports and analytics work.

**Recommended approach**:

1. **Group by non-sensitive attributes** (preferred): Re-examine what the business question actually is. "How many citizens per address" might really mean "how many citizens per postcode"—and postcode can be stored unencrypted as a derived attribute. Similarly, grouping by region, status, date range, or other non-sensitive fields often answers the real business need.

2. **Group by derived attributes**: Store and group by phonetic codes, address normalisation hashes (where all variations of an address produce the same hash), or other lossy representations. This gives approximate grouping.

3. **Compute in trusted environment** (for batch/reporting): For analytical queries that can run asynchronously, extract encrypted data, decrypt in the trusted environment, perform aggregations there, and return summarised results. This is unsuitable for interactive queries but works for overnight reporting.

4. **Deterministic encryption** (use with caution): If identical plaintexts must produce identical ciphertexts (enabling grouping), deterministic encryption can be used. However, this weakens security—an attacker can identify which records have the same value even without decrypting. Only appropriate for low-sensitivity fields.

**Performance implications**: Grouping in the trusted environment moves data processing outside the database, which is typically slower and uses more network bandwidth. For large datasets, this can be significant. Grouping by derived attributes has no performance penalty at query time but requires computing and maintaining those attributes during writes.

---

#### Date Arithmetic and Range Queries

**What this is**: Many business operations involve date calculations: finding citizens over 18 years old, selecting records modified in the last 30 days, filtering by date of birth range. SQL handles this with expressions like `WHERE dob BETWEEN '1990-01-01' AND '2000-12-31'` or `WHERE TRUNC(SYSDATE) - dob > 365*18`.

Encrypted dates cannot be compared. The database cannot determine whether an encrypted value represents a date before or after a given threshold.

**Difficulty**: Hard. Date-based filtering is common in government systems, and there is no single solution that works for all cases.

**Recommended approach**:

1. **Store derived flags** (most practical for common queries): Precompute and store boolean or categorical fields that answer common questions:
   - `is_adult` (BOOLEAN): True if citizen is 18+
   - `age_bracket` (VARCHAR): 'CHILD', 'ADULT', 'SENIOR'
   - `birth_decade` (NUMBER): 1980, 1990, 2000
   
   These derived fields are updated by a nightly batch process that has adapter access. The process decrypts DOB, computes the flags, and updates the records.

2. **Store plaintext year/month**: If the precise date is sensitive but the year is not, store `birth_year` unencrypted alongside `dob_enc`. This allows range queries on year while keeping the full date protected.

3. **Query in trusted environment**: For ad-hoc date range queries, retrieve candidate records (possibly pre-filtered by non-sensitive criteria), decrypt DOBs in the trusted environment, and filter there. Only practical for moderate result sets.

4. **Order-preserving encryption** (specialised, proceed with caution): Cryptographic schemes exist that preserve ordering—but they leak information about relative values. An attacker can tell that citizen A is older than citizen B without knowing either's actual age. Only appropriate if this leakage is acceptable.

**Performance implications**: Derived flags have zero query-time overhead—they're ordinary indexed columns. Maintaining them requires batch processing capacity. Querying in the trusted environment has the same implications as sorting externally: fine for small sets, problematic for large ones. Expect 10-100x slower queries if substantial external processing is required.

---

#### String Functions

**What this is**: SQL and PL/SQL provide functions to manipulate text: `UPPER()` converts to uppercase, `LOWER()` to lowercase, `SUBSTR()` extracts a portion, `INSTR()` finds a substring's position, `REPLACE()` substitutes characters, etc.

These functions operate on plaintext. Applying `UPPER()` to ciphertext produces garbage, not the uppercase version of the original name. Similarly, `INSTR(address_enc, 'London')` will never find 'London' inside encrypted text.

**Difficulty**: Hard. String manipulation is common throughout database code.

**Recommended approach**:

1. **Store multiple normalised forms**: If you need both original case and uppercase, store both:
   - `name_enc` (encrypted original: "Alice Smith")
   - `name_upper_enc` (encrypted uppercase: "ALICE SMITH")
   
   Compute both at insert/update time. This multiplies storage but allows queries on either form.

2. **Use fuzzy search instead of INSTR**: The Bloom filter approach described in this document handles partial matching more effectively than substring search. Instead of `WHERE INSTR(address, 'London') > 0`, use the fuzzy search pipeline which will find addresses containing "London" as part of the matching process.

3. **Perform transformations in trusted environment**: When a string function result is needed, decrypt the value, apply the function externally, return the result. For display purposes this is straightforward. For queries that filter by transformed values, the performance implications are significant.

**Performance implications**: Storing multiple encrypted forms increases storage 2-3x per affected field and requires additional adapter calls during writes. Using fuzzy search instead of INSTR changes query patterns but uses the existing indexed structures. External transformation for queries has similar performance implications to external sorting—feasible for small sets, costly for large ones.

---

#### Foreign Key Constraints

**What this is**: A foreign key is a database constraint that enforces relationships between tables. If the `benefits` table has a column `citizen_id` that must reference a valid citizen, a foreign key constraint ensures you cannot insert a benefit record for a non-existent citizen, and you cannot delete a citizen who has benefit records.

When the `citizens` table stores `citizen_id_hmac` instead of plaintext `citizen_id`, the foreign key relationship must also use the HMAC column. This cascades the protection model to all related tables.

**Difficulty**: Medium. The changes are mechanical but must be applied consistently across all related tables.

**Recommended approach**:

1. **Identify all foreign key relationships**: Query the data dictionary to find every table that references protected tables.

2. **Add HMAC columns to referencing tables**: The `benefits` table gains a `citizen_id_hmac` column alongside (temporarily) or replacing the plaintext `citizen_id`.

3. **Backfill with HMAC values**: Batch process to compute and populate the HMAC values for existing relationships.

4. **Update foreign key constraints**: Point to HMAC columns on both sides:
   ```sql
   ALTER TABLE benefits ADD CONSTRAINT fk_citizen 
     FOREIGN KEY (citizen_id_hmac) REFERENCES citizens(citizen_id_hmac);
   ```

5. **Update application code**: Any code that creates relationships must use the HMAC version. The INSTEAD OF trigger approach can help here—application inserts plaintext citizen_id into a view, the trigger converts to HMAC before storing.

**Performance implications**: Minimal. Foreign key checks are index lookups, and HMAC columns can be indexed identically to plaintext columns. The backfill process requires adapter calls to compute HMACs for all existing relationships, which should be planned as a batch operation during migration.

---

#### CHECK Constraints

**What this is**: A CHECK constraint enforces data validity rules within the database. For example, a constraint might ensure NINOs match the expected pattern (two letters, six digits, one letter) by using a regular expression: `CHECK (REGEXP_LIKE(nino, '^[A-Z]{2}[0-9]{6}[A-Z]$'))`.

When the NINO column contains an HMAC hash rather than the plaintext value, this constraint fails—the hash doesn't match the NINO pattern because it's not a NINO anymore.

**Difficulty**: Easy to Medium. The solution is straightforward; the effort is in identifying all affected constraints.

**Recommended approach**:

1. **Move validation earlier**: Validate data in the application layer or in the INSTEAD OF trigger, before encryption/hashing occurs. At that point, the plaintext is available and can be validated.

2. **Drop or modify database constraints**: The constraint on the base table can be removed (validation happens elsewhere) or changed to validate the expected format of the protected value (e.g., HMAC values should be 64 hexadecimal characters).

3. **Document the validation location**: Clearly record where each validation rule is now enforced so future developers understand the architecture.

**Performance implications**: Negligible. Validation in application code or triggers has similar performance to database CHECK constraints. In fact, removing CHECK constraints slightly improves INSERT/UPDATE performance since the database has one fewer check to perform.

---

#### Dynamic SQL

**What this is**: Most SQL in PL/SQL is static—the query text is written directly in the code. Dynamic SQL builds query text at runtime, allowing variable table names, column names, or query structures. This is done using `EXECUTE IMMEDIATE`:

```sql
v_sql := 'SELECT * FROM ' || p_table_name || ' WHERE ' || p_column || ' = :val';
EXECUTE IMMEDIATE v_sql INTO v_result USING p_value;
```

Dynamic SQL is common in flexible reporting systems, data-driven applications, and administrative utilities.

**Difficulty**: Hard. Each dynamic SQL statement must be analysed individually because the solution depends on what is being dynamically constructed.

**Recommended approach**:

1. **Inventory all dynamic SQL**: Search the codebase for `EXECUTE IMMEDIATE`, `DBMS_SQL`, and `OPEN ... FOR` with string variables. Document what each builds and why.

2. **Analyse each case**:
   - If the dynamic column might be a sensitive field, the code must detect this and use the `*_hmac` column name with an HMAC wrapper for the value.
   - If the dynamic table might be a protected table, ensure the view/synonym approach routes correctly.
   - Consider whether the dynamic flexibility is still needed or if a more static approach would be acceptable.

3. **Implement case-by-case**: There is no single pattern. Some dynamic SQL can be replaced with static SQL using CASE expressions. Some can use metadata tables that map "logical" column names to physical column names plus transformation functions.

**Performance implications**: Dynamic SQL already has performance overhead (query parsing for each execution). The additional logic to detect and handle sensitive columns adds minimal overhead—a few string comparisons. The HMAC wrapper calls have the same performance implications as in static SQL.

---

#### Bulk Operations (FORALL, BULK COLLECT)

**What this is**: PL/SQL can process data row-by-row in loops, but this is slow for large datasets due to context switching between the SQL and PL/SQL engines. Oracle provides bulk operations that process many rows at once:

- `BULK COLLECT` fetches many rows into a collection with a single context switch
- `FORALL` performs DML on many rows with a single context switch

```sql
FORALL i IN 1..v_data.COUNT
  INSERT INTO citizens VALUES v_data(i);
```

This processes thousands of rows far more efficiently than a row-by-row loop.

**Difficulty**: Hard. Bulk operations require batch adapter calls to maintain their performance advantage.

**Recommended approach**:

1. **Understand the problem**: If bulk INSERT triggers the INSTEAD OF trigger for each row, you lose the bulk advantage. A 10,000-row bulk insert becomes 10,000 adapter round-trips.

2. **Batch adapter calls**: Process in stages:
   - BULK COLLECT the plaintext data into PL/SQL collections
   - Call a batch adapter API that accepts arrays and returns arrays of encrypted values
   - FORALL to insert the pre-encrypted values directly into the base table (bypassing the view trigger for this controlled case)

   ```sql
   -- Pseudocode
   SELECT name, dob, nino BULK COLLECT INTO v_names, v_dobs, v_ninos FROM staging;
   
   -- Single batch call to adapter
   adapter.batch_encrypt(v_names, v_encrypted_names);
   adapter.batch_encrypt(v_dobs, v_encrypted_dobs);
   adapter.batch_hmac(v_ninos, v_hmac_ninos);
   -- ... compute fuzzy artefacts in batch
   
   -- Bulk insert protected values
   FORALL i IN 1..v_encrypted_names.COUNT
     INSERT INTO citizens_base VALUES (v_hmac_ninos(i), v_encrypted_names(i), ...);
   ```

3. **Staging table pattern**: Write plaintext to a staging table in the trusted environment. A batch process reads the staging table, transforms all data, and bulk-inserts to the cloud database.

**Performance implications**: Without batch adapter APIs, bulk operations lose most of their performance advantage and can become 10-100x slower. With proper batch APIs, the adapter round-trip is amortised across many records, and bulk throughput remains acceptable. Expect transformation throughput of hundreds to thousands of records per second depending on adapter capacity, versus tens of thousands for unprotected bulk operations. This must be factored into ETL and batch processing time estimates.

---

#### Logging and Debugging

**What this is**: Developers commonly add logging statements throughout PL/SQL code to trace execution and diagnose problems:

```sql
DBMS_OUTPUT.PUT_LINE('Processing citizen: ' || v_name);
INSERT INTO debug_log (message) VALUES ('Loaded NINO: ' || v_nino);
```

These logs may be visible in SQL*Plus output, written to database tables, or captured in trace files. If they contain plaintext sensitive data, they create an exposure in AWS.

**Difficulty**: Medium. The solution is clear, but finding all logging statements requires thorough code review.

**Recommended approach**:

1. **Search for logging patterns**: Identify all uses of `DBMS_OUTPUT`, `UTL_FILE`, custom logging procedures, and INSERT statements to logging tables.

2. **Categorise by sensitivity**:
   - Logging non-sensitive data (operation types, timestamps, record counts): No change needed.
   - Logging sensitive data: Must change.

3. **Modify sensitive logging**:
   - Replace plaintext with HMAC values: `'Processing citizen: ' || v_citizen_id_hmac` (traceable, not readable)
   - Log only non-sensitive context: 'Processing citizen' without the name
   - Remove debug logging from production code: The best time to question why production code prints citizen names

4. **Audit log considerations**: If formal audit logs record "what changed", they might need to record encrypted values that can be decrypted via the adapter when legitimate audit review occurs.

**Performance implications**: None for the logging itself. Logging encrypted or HMAC values instead of plaintext has no performance difference.

---

#### Exception Handlers

**What this is**: PL/SQL code typically includes exception handlers to catch errors and respond appropriately. A common pattern logs error context:

```sql
EXCEPTION
  WHEN OTHERS THEN
    INSERT INTO error_log (
      error_message,
      context
    ) VALUES (
      SQLERRM,
      'Failed processing citizen: ' || v_name || ', NINO: ' || v_nino
    );
    RAISE;
END;
```

If sensitive data is captured in error logs, it creates the same exposure as debug logging.

**Difficulty**: Easy to Medium. The fix is similar to logging, but exception handlers are sometimes hard to find in deeply nested code.

**Recommended approach**:

1. **Search for exception patterns**: Look for `EXCEPTION`, `WHEN OTHERS`, `WHEN ... THEN` followed by logging or INSERT statements.

2. **Remove sensitive data from error context**: Log the HMAC identifier, operation name, and timestamp—enough to diagnose issues without exposing citizen details.

3. **Consider structured error objects**: Instead of string concatenation, use error code and parameters that reference record IDs rather than plaintext values.

**Performance implications**: None.

---

#### Materialized Views

**What this is**: A materialized view is a database object that stores the result of a query physically, like a table. Unlike a regular view (which executes its query each time), a materialized view contains actual data that is refreshed periodically. They're used to pre-compute expensive queries, support data warehousing, and enable fast reporting.

If a materialized view contains sensitive columns from protected tables, those columns now contain ciphertext that changes each time the view refreshes (due to encryption randomness). Aggregations on those columns produce incorrect results.

**Difficulty**: Medium to Hard, depending on how materialized views are used.

**Recommended approach**:

1. **Inventory all materialized views**: Query the data dictionary for views referencing protected tables.

2. **Categorise by content**:
   - Views that aggregate/join only non-sensitive columns: May continue working with minimal change
   - Views that store sensitive column values: Need redesign
   - Views that aggregate by sensitive columns: Need the GROUP BY solutions discussed earlier

3. **Redesign as needed**:
   - Replace sensitive column storage with HMAC or encrypted values
   - Move aggregations to the trusted environment if necessary
   - Consider whether the materialized view is still needed or if the underlying query can use the fuzzy search approach

4. **Update refresh processes**: If refresh involves transformations, ensure adapter calls are appropriately batched for performance.

**Performance implications**: The original performance benefit of materialized views—pre-computed results—remains available for non-sensitive data. For sensitive data, the trade-off is between storing protected values (which limits what queries can do) versus computing in the trusted environment (which has throughput limits). This may change the overall analytics architecture.

---

#### Partitioning

**What this is**: Large tables are often partitioned—physically divided into smaller pieces based on column values. Common partitioning strategies include:
- **Range partitioning by date**: Each month's or year's data in its own partition
- **List partitioning by region**: Each region's data in its own partition
- **Range partitioning by name**: A-G in one partition, H-N in another, etc.

Partitioning improves query performance by allowing the database to scan only relevant partitions ("partition pruning").

If a table is partitioned by a column that is now encrypted or hashed, the partitioning becomes ineffective. Encrypted values distribute randomly across partitions, eliminating any pruning benefit.

**Difficulty**: Medium. Repartitioning a large table requires careful planning but is operationally straightforward.

**Recommended approach**:

1. **Analyse current partitioning strategy**: Understand why the table was partitioned this way and what queries benefit.

2. **Choose new partitioning key**:
   - If partitioned by date (record creation date, not citizen DOB): Usually can continue as-is
   - If partitioned by DOB: Consider partitioning by derived `birth_decade` or `age_bracket`
   - If partitioned by name: Consider partitioning by phonetic code prefix
   - If partitioned by region: If region is non-sensitive, continue as-is

3. **Evaluate HMAC prefix partitioning**: HMAC values are deterministic, so partitioning by the first few characters of an HMAC provides consistent distribution. This doesn't enable meaningful partition pruning for business queries, but does spread load evenly.

4. **Plan the repartitioning migration**: This is a significant database operation requiring maintenance windows or online redefinition.

**Performance implications**: Losing effective partition pruning can significantly impact query performance on large tables—potentially 10x slower for queries that previously pruned 90% of partitions. New partitioning strategies must be chosen to support the most important query patterns. Phonetic code partitioning, for example, enables pruning during fuzzy name searches.

---

### Summary: Patterns by Migration Difficulty

| Pattern | Difficulty | Effort Estimate | Key Performance Impact |
|---------|------------|-----------------|------------------------|
| SELECT with display | Easy | Days | None (view handles it) |
| Equality WHERE | Easy | Days | Minimal (HMAC lookup is indexed) |
| CHECK constraints | Easy | Days | None (validation moves earlier) |
| Logging | Easy-Medium | 1-2 weeks | None |
| Exception handlers | Easy-Medium | 1 week | None |
| INSERT/UPDATE/DELETE | Medium | 2-4 weeks | 10-50ms per operation |
| Simple triggers | Medium | 2-3 weeks | Minimal |
| Foreign keys | Medium | 2-4 weeks | Minimal (backfill is batch) |
| ORDER BY encrypted | Medium-Hard | 2-4 weeks | Varies (small sets OK, large sets costly) |
| Partitioning | Medium | 2-4 weeks | Must re-evaluate pruning strategy |
| Materialized views | Medium-Hard | 2-6 weeks | May need architecture change |
| GROUP BY encrypted | Hard | 4-8 weeks | May require trusted env processing |
| Date ranges | Hard | 4-8 weeks | Derived flags maintain performance |
| String functions | Hard | 4-8 weeks | Storage multiplies |
| Dynamic SQL | Hard | Case-by-case | Depends on usage |
| Bulk operations | Hard | 4-6 weeks | Requires batch adapter APIs |

**The key insight** is that most patterns have solutions, but some require more significant changes than others. A thorough inventory of existing PL/SQL patterns—ideally automated through static code analysis tools—should be conducted early in the project to size the effort accurately and identify high-risk areas.

---

## 5. How Fuzzy Search Works

One of the most sophisticated aspects of this architecture is enabling approximate name matching without exposing plaintext. When a user searches for "Alic Smth", we need to find "Alice Smith" despite the misspelling. This cannot be done on encrypted data—encryption deliberately destroys the ability to see patterns or similarity.

The solution uses a layered filtering approach that progressively narrows the candidate set before any decryption occurs.

### Layer 1: Phonetic Partitioning

Names are encoded using phonetic algorithms like Soundex or Metaphone. These algorithms assign a code based on how a name sounds, so "Alice", "Alyce", and "Alis" all produce the same phonetic code. This code is stored in plaintext (it reveals nothing meaningful about the actual name) and provides a fast first filter. A search for "Alic" produces phonetic code "ALKS", and we immediately filter to only records with that code—typically reducing 50 million records to perhaps 200,000.

### Layer 2: Locality-Sensitive Hashing

The records that passed the phonetic filter are further narrowed using a technique called locality-sensitive hashing, or LSH. This is where it gets clever.

Each name has been pre-processed into a Bloom filter—a mathematical fingerprint that represents which character patterns appear in the name. The name "Alice" is broken into overlapping fragments called n-grams: "ALI", "LIC", "ICE". Each of these fragments is hashed to a position in a fixed-size bit array, and that position is set to 1. The resulting array of 1s and 0s is the Bloom filter.

Now, here's the problem with Bloom filters: comparing them one-by-one against 200,000 candidates is slow. We need a way to quickly identify which candidates are likely to be similar.

LSH solves this by splitting each Bloom filter into segments and hashing each segment separately to produce bucket identifiers. The key property of LSH is that similar Bloom filters—meaning similar names—will share at least one bucket. Instead of comparing against all 200,000 candidates, we look up which candidates share any bucket with our search term. This typically reduces the set to a few thousand.

To understand why this works, imagine each Bloom filter as a fingerprint made of many ridges. A normal hash of the entire fingerprint would only match exactly identical fingerprints—useless for fuzzy matching. LSH looks at sections of the fingerprint separately. Two similar names might differ in some sections but match in others. As long as they match in at least one section, they end up in the same bucket and become candidates for comparison.

### Layer 3: Bloom Filter Similarity

The remaining candidates have their Bloom filters compared directly against the search term's Bloom filter. This comparison measures what percentage of the 1-bits overlap between the two filters. A high overlap percentage (say, 70% or more) indicates the names likely share many of the same character patterns.

This step might reduce 5,000 candidates to 200 or fewer.

### Layer 4: Decryption and Final Verification

Only at this point—with perhaps 200 candidates from an original 50 million—do we involve the trusted adapter. These candidate records have their encrypted names decrypted, and a final exact string comparison determines the actual matches.

The critical point is that decryption is expensive (it requires a network call to the trusted environment) but it only happens on a tiny fraction of records. The fuzzy artefacts—phonetic codes, Bloom filters, LSH buckets—do the heavy lifting of narrowing the search, all without exposing any actual names.

---

## 6. Walking Through Common Operations

### Creating a New Citizen Record

When a new citizen is registered, the application collects their details: name, date of birth, NINO, address, and relationships (e.g., parent identifiers). Before anything reaches AWS, the trusted adapter processes all of this:

The name "Alice Smith" is encrypted using AES with a key held in the adapter's HSM. The resulting ciphertext is stored as `name_enc`. Simultaneously, the adapter generates the phonetic code ("ALSM"), breaks the name into n-grams, creates a Bloom filter, and computes LSH bucket identifiers. These are stored alongside the encrypted name.

The NINO "QQ123456C" is processed through HMAC using the adapter's secret key. The resulting hash is stored as `nino_hmac`. The plaintext NINO is never written to the AWS-hosted database.

Relationship identifiers (such as a parent's citizen ID) are similarly converted to HMACs before storage. This allows the database to join citizens to their relatives using hash comparisons, without the actual identifiers being present.

The database record that ultimately lands in AWS contains only: encrypted fields that cannot be read without the key, HMAC hashes that cannot be reversed without the key, and mathematical artefacts for searching that cannot be reversed at all.

### Looking Up a Citizen by NINO

An operator needs to retrieve a citizen's record using their National Insurance Number. The application receives the NINO from the operator, sends it to the trusted adapter, and receives back the HMAC hash. This hash is then used in the database query:

```sql
SELECT * FROM citizens_v WHERE nino_hmac = :computed_hash;
```

The database finds matching records (there should be exactly one for a valid NINO) and returns them. The view layer invokes decryption for any sensitive fields the application needs to display. The trusted adapter decrypts the name, date of birth, and address, returning them to the application for presentation.

At no point does the actual NINO appear in the AWS database—not in the stored data, not in the query, not in the logs. AWS sees only hash values being compared.

### Searching for a Citizen by Name

An operator needs to find a citizen but isn't sure of the exact spelling. They enter "Alic Smth" into the search interface. The application sends this to the trusted adapter, which computes the phonetic code, generates a Bloom filter, and determines LSH bucket identifiers.

The application then queries the database for candidates:

```sql
SELECT * FROM citizens 
WHERE phonetic = :phonetic_code 
  AND lsh_bucket IN (:bucket1, :bucket2, :bucket3);
```

This returns several thousand potential matches. The application computes Bloom filter overlap scores to rank these candidates and takes the top 100 or so. Only these records are sent to the trusted adapter for decryption.

The adapter decrypts the names and returns them. The application displays ranked results to the operator, who identifies "Alice Smith" as the person they were looking for.

The database efficiently narrowed 50 million records to a handful using mathematical operations on non-sensitive artefacts. Sensitive decryption only occurred on the final shortlist.

### Updating a Citizen's Name

A citizen changes their name from "Alice Smith" to "Alice Johnson". The application receives the new name and sends it to the trusted adapter, which encrypts it and generates new fuzzy artefacts (phonetic code, Bloom filter, LSH buckets).

The database update modifies the encrypted name column and all associated artefacts. The HMAC of the citizen ID remains unchanged—it's still the same person, just with a different name.

### Querying Relationships

A caseworker needs to find all children of a particular citizen. The parent's citizen ID is known. The application computes the HMAC of this identifier via the trusted adapter, then queries:

```sql
SELECT * FROM citizens_v WHERE parent_id_hmac = :parent_hash;
```

The database returns all records where the stored parent hash matches the computed hash. The relationship is maintained and queryable without the underlying identifier values being present in AWS.

---

## 7. Why HMAC Instead of Plain Hashing

A natural question arises: why not simply hash identifiers like NINO using a standard hash function like SHA-256? The answer relates to the specific nature of identifiers like National Insurance Numbers.

A NINO has a fixed format: two letters, six digits, and a final letter. There are approximately 450 million possible valid NINOs. If we used a plain hash, an attacker with database access could compute the hash of every possible NINO in a few hours on commodity hardware. They would build a lookup table mapping hashes to NINOs, and our protection would be worthless.

HMAC prevents this attack by requiring a secret key to compute the hash. Without the key—which never leaves the trusted environment—the attacker cannot compute the hash of even a single NINO to check against the database. The 450-million-entry lookup table is useless because they cannot produce the same hashes we did.

This is sometimes called a "pepper" (as distinct from a "salt"). A salt is stored alongside each record and prevents pre-computed rainbow table attacks; a pepper is a secret key that is never stored with the data. If the database is compromised, salts are compromised too—but a pepper held externally remains secure.

---

## 8. Why Bloom Filters Cannot Be Reversed

Another important question: if Bloom filters encode information about names, could an attacker reverse them to recover the original names?

The answer is no, for two reasons.

First, Bloom filters are lossy. Many different names can produce the same Bloom filter. The filter only says "these character patterns are present" but cannot distinguish between different combinations of those patterns. It's like knowing a room contains some combination of red, blue, and green balls, but not knowing how many of each or in what arrangement.

Second, even if an attacker knew approximately what the name pattern looked like, they would still need to verify their guess. But verification requires either the encrypted name (which they cannot decrypt) or access to the trusted adapter (which they don't have).

The practical risk from Bloom filters is statistical inference—an attacker might learn that a name is likely 10-12 characters long and contains common English letter patterns. This is not nothing, but it is far less than exposing the actual names, and it is insufficient to identify specific individuals.

---

## 9. Migration Strategy

Implementing this architecture does not require a disruptive "big bang" cutover. Instead, we follow a phased approach that maintains operational continuity throughout.

### Phase 1: Schema Extension

We add new columns to existing tables without modifying the original columns. The `citizens` table gains `name_enc`, `nino_hmac`, `phonetic`, `bloom`, and `lsh_buckets` columns. At this stage, the original plaintext columns remain in place and continue to be used by all existing code.

### Phase 2: Background Backfill

A batch process runs during off-peak hours to populate the new columns. For each existing record, it calls the trusted adapter to encrypt sensitive fields, generate HMACs, and compute fuzzy artefacts. This process can run incrementally and be paused or resumed as needed. It does not affect production operations.

### Phase 3: Dual-Write

Once backfill is complete, we modify write operations to populate both the original columns and the new protected columns. Every insert or update writes data in both formats. This allows us to validate that the new columns are being maintained correctly while the original system continues to function normally.

### Phase 4: Read Migration

We introduce the view layer and wrapper functions, then gradually migrate read operations to use the protected columns via the views. This can be done piece by piece—one report, one screen, one procedure at a time—with ability to roll back if issues arise. At each step, we can compare results between the old direct reads and the new view-based reads.

### Phase 5: Plaintext Removal

Once all read and write operations are flowing through the protected columns and the system has been stable for an appropriate period, we drop the original plaintext columns. At this point, the migration is complete. AWS contains no plaintext sensitive data and has no access to encryption keys.

---

## 10. Performance Characteristics

The architecture introduces new performance considerations that must be understood and managed.

### Operations That Remain Fast

The overwhelming majority of database work continues at normal database speeds. Queries that filter on phonetic codes, LSH buckets, or HMAC values use standard index lookups. Comparisons of hash values are as fast as comparisons of any other strings. Joins on HMAC foreign keys perform exactly like joins on regular columns.

Bloom filter similarity calculations involve basic bitwise operations that databases handle efficiently. Even computing overlap percentages across thousands of candidates completes in milliseconds.

### Operations That Incur Cost

Decryption and HMAC computation require calls to the trusted adapter. These incur network latency and are inherently slower than in-database operations. However, the architecture is specifically designed to minimise how often these expensive operations occur:

For exact lookups by identifier, HMAC computation happens once per query to generate the search hash. This is a single network round-trip, typically measured in low milliseconds.

For fuzzy searches, the candidate filtering layers ensure that decryption only happens on a small final set. Decrypting 50 records is operationally very different from decrypting 50 million.

For bulk operations that need to process many records, the adapter can support batch APIs to reduce round-trip overhead.

### Scale Considerations

At the scale of tens of millions of citizen records, the layered filtering approach keeps performance manageable. Consider a fuzzy name search:

- Total database: 50,000,000 records
- After phonetic filter: ~200,000 records (0.4%)
- After LSH bucket intersection: ~5,000 records (0.01%)
- After Bloom similarity filter: ~200 records (0.0004%)
- Records requiring decryption: <50 (0.0001%)

The expensive operation (decryption) touches less than 0.0001% of the database. This is the power of the layered approach.

---

## 11. Security Analysis

Let us examine specific threat scenarios and how this architecture addresses them.

### Threat: AWS is compelled to provide data access

Under the Patriot Act or CLOUD Act, US authorities demand that AWS provide access to the database. AWS complies fully, granting complete read access to all stored data.

**Mitigation**: All sensitive fields are encrypted with keys that do not exist in AWS. The authorities receive ciphertext that is computationally infeasible to decrypt. Identifiers are HMAC hashes that cannot be reversed without the key. Fuzzy artefacts cannot be reversed to original names. The data is useless without the trusted adapter.

### Threat: AWS is compelled to provide encryption keys

Authorities demand not just data access, but access to encryption keys.

**Mitigation**: There are no keys to provide. The keys exist only in the trusted environment, which is outside AWS and outside US jurisdiction. AWS cannot comply with this demand because they do not have what is being demanded.

### Threat: Database is exfiltrated by an attacker

A sophisticated attacker gains persistent access to the database and exfiltrates all data.

**Mitigation**: The attacker obtains encrypted fields (useless without keys), HMAC hashes (cannot be brute-forced without the key), and fuzzy artefacts (non-reversible). They cannot reconstruct meaningful citizen records. They lack the key material needed to decrypt or verify guesses.

### Threat: Attacker attempts to brute-force identifiers

An attacker with database access tries to identify records by computing hashes of all possible NINOs.

**Mitigation**: HMAC requires the secret key to compute. Without the key, the attacker cannot produce hashes to compare against the database. The key is held in HSMs in the trusted environment and is computationally infeasible to extract.

### Threat: Attacker observes query patterns

An attacker monitors database queries over time to infer information from access patterns.

**Mitigation**: This architecture does not specifically address traffic analysis attacks. Additional mitigations (query padding, access pattern obfuscation) could be added if this threat is prioritised. This is an acknowledged limitation.

### Threat: Trusted adapter is compromised

An attacker gains access to the trusted environment and the adapter service.

**Mitigation**: This is the scenario the architecture cannot prevent. The trusted environment is the root of trust. If it is compromised, the security model fails. However, this is a much smaller attack surface than the full cloud deployment, and it can be protected with physical security, HSMs, rigorous access controls, and regulatory compliance frameworks designed for sensitive data handling.

---

## 12. Operational Considerations

### High Availability of the Trusted Adapter

Because the trusted adapter is required for decryption and HMAC operations, it must be highly available. A failed adapter means the application cannot retrieve or verify citizen data. The adapter should be deployed with redundancy across multiple physical locations, with automatic failover and health monitoring.

### Key Management

Encryption keys and HMAC secrets must be managed with extreme care. Key rotation procedures, backup strategies, and disaster recovery plans must be established. Loss of keys means permanent loss of access to encrypted data—this is the flip side of the security guarantee.

Hardware Security Modules (HSMs) should be used to store keys, providing both physical protection and tamper-evident key material.

### Monitoring and Debugging

Traditional database debugging becomes more complex when data is encrypted. Error logs may contain encrypted values that cannot be read without adapter access. Monitoring tools must be adapted to work with protected data.

The trusted adapter should maintain comprehensive audit logs of all operations, providing visibility into who accessed what data and when.

### Performance Tuning

Because adapter calls are the primary performance bottleneck, optimisation efforts should focus on reducing unnecessary calls. Batch APIs, result caching (where appropriate), and efficient query patterns all help.

Index design remains critical. Phonetic codes, LSH buckets, and HMAC columns all need appropriate indexes for efficient filtering.

---

## 13. Integration with External Systems

The citizen database does not exist in isolation. It receives updates from numerous external systems—tax authorities, benefits agencies, health services, local councils, and others—through a combination of real-time interfaces and batch file transfers. This architecture must accommodate all of these integration patterns while maintaining its security properties.

### The Integration Challenge

External systems send citizen data changes in various forms: real-time API calls when a citizen reports a change of address, nightly batch files containing thousands of updates from partner agencies, event-driven messages when life events occur (births, deaths, marriages), and bulk corrections when data quality issues are identified.

Each of these integration patterns must flow through the trusted adapter before data reaches the cloud database. The adapter becomes not just a query-time service, but a critical component of the data ingestion pipeline.

### Real-Time Integration Pattern

When an external system sends a real-time update—say, a change of address notification from a local council—the flow is:

1. The integration layer receives the inbound message containing plaintext citizen data
2. The message is routed to the trusted adapter (which runs outside AWS)
3. The adapter encrypts sensitive fields and generates HMACs for identifiers
4. For any fields requiring fuzzy matching (like names), the adapter also generates phonetic codes, Bloom filters, and LSH buckets
5. The protected payload is then written to the cloud database
6. Acknowledgement flows back to the originating system

The critical point is that plaintext citizen data never enters AWS. The integration layer that receives external messages must itself run in the trusted environment, or it must immediately forward to the trusted adapter before any persistence occurs.

This has implications for integration architecture. Message queues, API gateways, and transformation logic that handle plaintext citizen data must all reside in the trusted environment. Only after transformation to protected format does data cross into AWS.

### Batch Integration Pattern

Many external systems still operate on batch cycles. A partner agency might send a nightly file containing 50,000 address changes, or a weekly file with 200,000 benefit status updates. Processing these through the trusted adapter requires careful design.

**Naive approach (problematic)**: Process each record individually through the adapter. For 50,000 records, this means 50,000 round-trips to the adapter. Even at 10ms per call, this is over 8 minutes of pure adapter latency, plus database write time. For larger batches, this becomes impractical.

**Batch-optimised approach**: The trusted adapter exposes batch APIs that accept arrays of records and return arrays of protected values. A single call might process 1,000 records, reducing 50,000 round-trips to 50. The adapter parallelises internally across multiple worker threads or processes.

```
Batch processing flow:

1. Receive batch file (e.g., 50,000 address updates)
2. Parse and validate records
3. Group into batches of 1,000 records
4. For each batch:
   a. Send to adapter batch API
   b. Adapter encrypts all sensitive fields
   c. Adapter generates HMACs for all identifiers
   d. Adapter computes fuzzy artefacts for all name fields
   e. Return protected batch
5. Bulk insert/update to cloud database
6. Commit and log completion
```

### Adapter Scaling for Batch Workloads

Batch processing creates burst load on the trusted adapter. A nightly batch window might need to process millions of records in a few hours, while daytime interactive load is much lighter.

The adapter architecture must accommodate this:

**Horizontal scaling**: Multiple adapter instances behind a load balancer, with the ability to scale up during batch windows and scale down during quiet periods.

**Compute optimisation**: Encryption and HMAC operations are CPU-intensive but highly parallelisable. Modern CPUs with AES-NI instructions can encrypt at gigabytes per second. The adapter should be designed to saturate available CPU cores.

**HSM considerations**: If keys are held in HSMs, the HSM becomes a potential bottleneck. Some designs use HSM-derived keys that are held in protected memory for the duration of a batch operation, reducing HSM round-trips while maintaining security properties. This is a trade-off that must be evaluated against specific security requirements.

### Handling Bulk Updates

Some scenarios involve truly massive bulk updates. A redistricting exercise might change address administrative codes for millions of citizens. A data quality remediation might correct name spellings across hundreds of thousands of records.

For these scenarios, the batch approach above may still be insufficient. Alternative strategies include:

**Offline processing**: Export encrypted data from the cloud database to the trusted environment (data remains protected during transit), perform transformations with full access to keys and plaintext, then re-encrypt and reload. This is essentially treating the cloud database as cold storage during the bulk operation.

**Streaming processing**: Implement a continuous streaming pipeline where changes flow through the adapter in a sustained stream rather than discrete batches. Technologies like Kafka can buffer and manage backpressure while the adapter processes at its maximum sustainable rate.

**Incremental computation**: For changes that only affect specific artefacts (e.g., a new administrative code that doesn't change the actual address text), update only the affected columns without recomputing everything. If the name hasn't changed, the Bloom filters and LSH buckets don't need regeneration.

### Change Data Capture (CDC)

Many integration patterns rely on Change Data Capture—detecting what changed in the source system and propagating only the deltas. This architecture supports CDC, but with a nuance: the CDC events themselves must flow through the trusted adapter.

If an external system publishes a CDC event saying "Citizen X changed address from A to B", that event contains plaintext data. It must be consumed in the trusted environment, transformed to protected format, and only then written to the cloud database.

This means CDC consumers (the processes that read change events and apply them to the database) must run in the trusted environment or route through it.

### Event-Driven Integration

Modern integration increasingly uses event-driven patterns: a citizen death triggers updates across multiple systems, a benefit decision triggers payment system updates, a fraud alert triggers case management actions.

In this architecture, events flowing *into* the citizen database must pass through the adapter for protection. Events flowing *out* of the citizen database may contain protected data that downstream systems cannot interpret without their own adapter access.

This raises a design question: should downstream systems share access to the trusted adapter, or should they receive plaintext events?

The answer depends on the downstream system's security posture. A system running in the same trusted environment can receive plaintext. A system running in AWS or another cloud should receive protected data and either have its own adapter access or work with already-decrypted data that was processed in the trusted environment.

### Error Handling and Reconciliation

Integration failures happen. Batch jobs abort halfway through. Network issues interrupt real-time feeds. Messages get lost or duplicated.

The architecture must support:

**Idempotent operations**: Reprocessing the same record should produce the same result. Since encryption with the same key and HMAC with the same key are deterministic, this is naturally supported.

**Partial batch recovery**: If a batch of 50,000 records fails after processing 30,000, we need to resume from record 30,001, not restart from the beginning.

**Reconciliation**: Periodic processes that compare the protected database against source systems (via the adapter) to identify and correct discrepancies.

**Audit trails**: Comprehensive logging of what external systems sent what data when, what transformations were applied, and what was written to the database. This must be done carefully to avoid logging plaintext sensitive data.

### Integration Performance Benchmarks

To size the trusted adapter appropriately, we need to understand the integration workload:

| Integration Type | Volume | Frequency | Latency Requirement |
|-----------------|--------|-----------|---------------------|
| Real-time address change | 10,000/day | Continuous | < 500ms end-to-end |
| Nightly benefit status | 200,000 records | Daily batch | Complete in 2 hours |
| Weekly partner file | 500,000 records | Weekly batch | Complete in 4 hours |
| Annual bulk update | 5,000,000 records | Annual | Complete in 24 hours |
| Real-time death notification | 1,500/day | Continuous | < 200ms end-to-end |

These numbers determine adapter capacity requirements. The adapter must handle sustained throughput for batch processing while maintaining low latency for real-time operations.

### Integration Architecture Summary

The trusted adapter is not just a query-time component—it is the mandatory gateway for all data entering the system. This has significant architectural implications:

1. **All integration endpoints must be in or route through the trusted environment**: Message brokers, API gateways, file landing zones—anything that touches plaintext citizen data before it's protected.

2. **Batch processing requires batch APIs**: The adapter must support high-throughput bulk operations, not just single-record calls.

3. **Scaling the adapter is critical**: The adapter must handle both interactive query load and burst batch processing load, potentially simultaneously.

4. **Integration testing is complex**: Testing integrations requires either a test adapter with test keys, or careful use of synthetic data that mimics real patterns without containing real citizen information.

5. **Downstream systems need consideration**: Systems that consume data from the citizen database must either have adapter access or receive data that was decrypted in the trusted environment.

---

## 14. Cost Implications

### Storage Costs

Each sensitive field now has multiple representations: the encrypted value, and for names, additional columns for phonetic codes, Bloom filters, and LSH buckets. Rough estimates suggest 2-3x storage overhead for fully protected tables. At cloud storage pricing, this is likely significant but not prohibitive.

### Compute Costs

The trusted adapter environment requires dedicated infrastructure. If deployed with HSMs for key protection, hardware costs increase. The adapter must scale to handle peak query loads with acceptable latency.

Batch processing for backfill and ongoing artefact generation also consumes compute resources, though this can often run during off-peak periods.

### Operational Costs

The architecture introduces operational complexity. Staff must understand the security model, the trust boundaries, and the operational procedures. Incident response becomes more complex when data investigation requires adapter access. Training, documentation, and specialist expertise all carry costs.

### Offset by Avoided Costs

These costs should be evaluated against the alternative: a multi-year effort to rewrite PL/SQL logic, rebuild application layers, and potentially re-engineer entire business processes. The approach described here preserves existing investments and avoids the risk of failed large-scale rewrites.

Additionally, there may be regulatory or reputational costs associated with inadequate data protection. This architecture provides a defensible security posture that may be required for continued operation.

---

## 15. What This Approach Does Not Do

Intellectual honesty requires acknowledging limitations.

**This approach does not eliminate all risk.** A sufficiently resourced attacker with long-term access to both the AWS environment and the trusted adapter could potentially compromise the system. The goal is to ensure that AWS access alone is insufficient.

**This approach does not preserve 100% of PL/SQL unchanged.** Some queries must be modified to use HMAC wrappers. Some patterns (LIKE searches, range queries on encrypted fields) cannot work at all without flow changes. The claim is "minimal refactoring relative to full rewrite," not zero change.

**This approach does not make fuzzy matching perfect.** There will be false positives (records that match the filter but are not actually similar) and false negatives (records that should match but don't). The layered approach and final verification mitigate this, but they do not eliminate it.

**This approach does not address all possible threats.** Traffic analysis, side-channel attacks, and insider threats at the trusted environment are not specifically mitigated. Defence in depth, operational security, and complementary controls remain necessary.

---

## 16. Recommended Path Forward

Given the constraints—strong sovereignty requirements and substantial existing PL/SQL—this architecture represents the most practical approach. We recommend proceeding as follows:

### Immediate (1-2 months)

Build a prototype trusted adapter with encrypt, decrypt, and HMAC capabilities. Demonstrate end-to-end operation with a small sample dataset. Validate integration patterns with Oracle external procedures.

### Short-term (3-6 months)

Implement the full architecture for one core table (e.g., the primary citizen record table). Complete backfill for production data. Migrate representative PL/SQL procedures to use the view layer and wrapper functions. Measure performance and validate security properties.

### Medium-term (6-18 months)

Extend the architecture to additional tables following the established patterns. Complete PL/SQL migration systematically. Remove plaintext columns once validated. Establish full operational procedures.

### Ongoing

Maintain and evolve key management procedures. Monitor performance and tune as needed. Review security posture as threats evolve. Consider future enhancements such as query pattern obfuscation if required.

---

## Conclusion

This document has described an architecture that achieves what initially appears impossible: using AWS for citizen data while ensuring AWS cannot access that data, and preserving the vast majority of existing PL/SQL logic.

The key insight is separation of storage from intelligibility. AWS is an excellent place to run a database, but it does not need to understand what the database contains. By encrypting sensitive fields, hashing identifiers with secret keys, and generating non-reversible artefacts for fuzzy matching, we transform the data into a form that serves all operational needs while being meaningless to anyone without access to the trusted environment.

This is not a trivial implementation, but it is achievable. It does not require rewriting decades of PL/SQL. It does require careful planning, phased execution, and ongoing operational discipline. The result is a cloud deployment that provides scaling, resilience, and operational efficiency while maintaining a genuinely defensible security posture for citizen data.

---

## Appendix: Glossary

| Term | Definition |
|------|------------|
| **HMAC** | Hash-based Message Authentication Code. A keyed hash that produces consistent output for the same input, but cannot be computed without the secret key, preventing brute-force attacks. |
| **Pepper** | A secret key used in HMAC computation, stored only in the trusted environment. Unlike a salt (stored per-record), a pepper is never exposed even if the database is compromised. |
| **Bloom filter** | A space-efficient data structure representing membership of elements in a set. In this context, it encodes which n-grams appear in a name without revealing the name itself. |
| **LSH** | Locality-Sensitive Hashing. A technique that groups similar items into the same buckets with high probability, enabling efficient approximate matching without comparing all pairs. |
| **N-gram** | A contiguous sequence of n characters from a string. "Alice" with n=3 produces: "ALI", "LIC", "ICE". Used to represent structural patterns in text for fuzzy matching. |
| **Phonetic code** | An encoding that represents how a word sounds rather than how it is spelled. Different spellings of similar-sounding names produce the same code. |
| **Trusted environment** | Infrastructure outside the AWS trust boundary where encryption keys reside and sensitive operations are performed. Typically on-premise or in non-US-jurisdiction sovereign cloud. |
| **Trusted adapter** | The service within the trusted environment that performs encryption, decryption, HMAC computation, and fuzzy artefact generation for the database. |
