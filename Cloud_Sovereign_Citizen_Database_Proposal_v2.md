# Secure Cloud Migration of Citizen Database
## Minimising Patriot Act Risk While Preserving PL/SQL Investment

**Version 2.0**

---

## Executive Summary

This document proposes an architecture for migrating a citizen database to AWS while ensuring that sensitive personal data cannot be accessed by the cloud provider under any circumstances, including legal compulsion via US legislation such as the Patriot Act or CLOUD Act.

The approach is built on a simple but powerful principle: **the cloud stores data, but cannot understand it**. All encryption keys and the ability to interpret sensitive information remain exclusively within a trusted environment under UK jurisdiction. AWS sees only ciphertext and mathematical transformations that cannot be reversed without access to keys that never leave our control.

Critically, this design preserves the vast majority of existing PL/SQL logic. Rather than rewriting decades of database code, we introduce a thin layer of views and wrapper functions that allow PL/SQL to continue operating while transparently routing sensitive operations to the trusted environment.

---

## Part 1: The Problem and the Solution

### 1.1 The Sovereignty Problem

Moving citizen data to AWS creates a specific legal exposure. Under US law—particularly the Patriot Act and the CLOUD Act—US authorities can compel American companies to provide access to data stored on their infrastructure, regardless of where that data is physically located. This means that even if we use AWS's European data centres, the data remains potentially accessible to US government demands.

For a citizen database containing names, National Insurance numbers, dates of birth, addresses, and family relationships, this exposure is unacceptable. The question becomes: how do we gain the operational benefits of cloud infrastructure while ensuring that even with full cooperation from AWS, no meaningful access to citizen data is possible?

### 1.2 The PL/SQL Problem

The existing database contains substantial PL/SQL logic developed over many years. This code handles complex business rules, data validation, reporting, and integration with other systems. A naive approach to encryption—such as encrypting all data at the application layer—would break this code entirely. PL/SQL procedures that compare names, join on identifiers, or filter by date of birth would all fail because they cannot operate on encrypted values.

The traditional answer to this problem is to rewrite everything. Move the logic out of the database, rebuild it in application code, and treat the database as pure storage. This is a multi-year undertaking that carries enormous risk and cost.

### 1.3 The Apparent Contradiction

These two requirements seem to be in direct conflict. If data must be encrypted so AWS cannot read it, the database cannot process it. If the database must process it, it must be able to read it. The architecture proposed here resolves this contradiction.

### 1.4 The Core Architectural Insight

The solution rests on a fundamental separation: AWS stores protected representations of data, but the ability to interpret those representations exists only outside AWS.

Think of it like storing a locked safe in a warehouse. The warehouse operator can move the safe, stack it, even tell you how heavy it is—but without the key, they cannot open it. In our architecture, AWS is the warehouse. The safe is the encrypted data. The key never enters the warehouse.

But we go further than simple encryption. We need the database to be able to do useful work on this protected data—filtering, joining, searching—without ever seeing the actual values. This requires different protection strategies for different types of operations.

### 1.5 Three Categories of Data

Every field in the database falls into one of three categories, each requiring a different approach:

**Fields that must be read** (Name, Date of Birth, Address): These are encrypted using strong symmetric encryption. The encrypted values are stored in the database, but decryption can only happen by calling out to a trusted service that holds the keys. When PL/SQL needs to display a name or validate a date of birth, it calls a wrapper function that reaches outside AWS to perform the decryption.

**Fields that must be matched exactly** (NINO, Citizen ID, Relationship links): These are transformed using a keyed hash called HMAC. The original value cannot be recovered from the hash, but importantly, the same input always produces the same output. This means the database can still perform equality comparisons and joins—it just compares hash values instead of the original identifiers. The crucial detail is that the hash requires a secret key. Without that key (which never leaves the trusted environment), an attacker cannot compute hashes for comparison or brute-force the original values.

**Fields that must support fuzzy matching** (Names for approximate search): Searching for "Alic Smth" and finding "Alice Smith" requires a completely different approach. We cannot encrypt these values because encryption destroys the ability to compare similarity. Instead, we generate mathematical fingerprints—called Bloom filters—that capture the structural patterns in names without revealing the actual text. Combined with phonetic encoding and a technique called locality-sensitive hashing, these fingerprints allow the database to efficiently identify potential matches without ever seeing the original names.

---

## Part 2: The Trusted Adapter

At the centre of this design is a service we call the **Trusted Adapter**. This is a dedicated component that runs entirely outside AWS, in infrastructure we control completely—either on-premise or in a sovereign UK cloud environment with no US jurisdiction exposure.

The Trusted Adapter has one job: it holds all the secrets and performs all the sensitive operations. It provides three core capabilities:

**Encryption and Decryption**: When data needs to be written, the adapter encrypts it before it reaches AWS. When data needs to be read, the adapter receives the encrypted value and returns the plaintext. The keys never leave the adapter; they are typically stored in Hardware Security Modules (HSMs) that make extraction physically impossible.

**HMAC Generation**: When an identifier needs to be stored or compared, the adapter computes the keyed hash. This allows the database to perform joins and lookups without ever knowing the actual identifier values.

**Fuzzy Artefact Computation**: When a name is stored, the adapter generates the phonetic codes, Bloom filters, and locality-sensitive hash buckets that enable approximate matching.

The database communicates with this adapter through Oracle's external procedure mechanism. From PL/SQL's perspective, it's simply calling a function. That function happens to reach out over a secure channel to the trusted environment, perform the sensitive operation, and return the result.

This is the critical insight: we are not asking PL/SQL to understand encryption or hashing. We are wrapping those operations in functions that look like ordinary database functions. The complexity is hidden behind a clean interface.

### 2.1 The Wrapper Functions

To make the adapter's capabilities accessible from PL/SQL, we create a set of wrapper functions. These are ordinary Oracle functions that, internally, call out to the trusted adapter over a secure channel. The calling code doesn't need to know this—it just calls the function and gets a result.

**Encryption functions** take plaintext and return ciphertext:

| Function | Purpose | Example |
|----------|---------|---------|
| `encrypt_name(plaintext)` | Encrypt a name | `encrypt_name('Alice Smith')` → `'x7Kp9mQ...'` |
| `encrypt_dob(plaintext)` | Encrypt a date of birth | `encrypt_dob('1990-05-15')` → `'Yw3nR8v...'` |
| `encrypt_address(plaintext)` | Encrypt an address | `encrypt_address('123 High St')` → `'mT4xL2j...'` |

**Decryption functions** take ciphertext and return plaintext:

| Function | Purpose | Example |
|----------|---------|---------|
| `decrypt_name(ciphertext)` | Decrypt a name | `decrypt_name('x7Kp9mQ...')` → `'Alice Smith'` |
| `decrypt_dob(ciphertext)` | Decrypt a date of birth | `decrypt_dob('Yw3nR8v...')` → `'1990-05-15'` |
| `decrypt_address(ciphertext)` | Decrypt an address | `decrypt_address('mT4xL2j...')` → `'123 High St'` |

**HMAC functions** take an identifier and return a keyed hash:

| Function | Purpose | Example |
|----------|---------|---------|
| `hmac_citizen_id(identifier)` | Hash a citizen ID | `hmac_citizen_id('ABC123')` → `'a1b2c3d4e5f6...'` |
| `hmac_nino(identifier)` | Hash a National Insurance Number | `hmac_nino('QQ123456C')` → `'f6e5d4c3b2a1...'` |

**Fuzzy artefact functions** take a name and return mathematical structures for approximate matching:

| Function | Purpose | Example |
|----------|---------|---------|
| `compute_phonetic(name)` | Generate phonetic code | `compute_phonetic('Alice Smith')` → `'ALSM'` |
| `compute_bloom(name)` | Generate Bloom filter | `compute_bloom('Alice Smith')` → `(bit array)` |
| `compute_lsh(name)` | Generate LSH bucket IDs | `compute_lsh('Alice Smith')` → `'B17,B42,B89'` |

Each of these functions internally makes a network call to the trusted adapter, which holds the encryption keys and HMAC secrets. The adapter performs the cryptographic operation and returns the result. To the PL/SQL code calling these functions, they appear to be ordinary database functions—the network call is invisible.

These wrapper functions are used throughout the architecture: in views for decryption, in triggers for encryption, and in queries for identifier lookups.

---

## Part 3: Oracle Concepts for Non-Database Readers

Before diving into how existing PL/SQL continues to work, this section explains several Oracle concepts that are fundamental to understanding the architecture. Readers familiar with Oracle can skip ahead.

### 3.1 Tables, Views, and Synonyms

**Tables** store data physically on disk. When you SELECT from a table, you're reading stored rows.

**Views** are saved queries that look like tables but don't store data themselves. When you SELECT from a view, Oracle executes the view's query and returns the results. A view can transform data, join tables, or apply functions to columns. To the code querying it, a view looks exactly like a table.

**Synonyms** are aliases—alternative names for database objects. When code references a synonym, Oracle transparently redirects to the actual object. This allows us to rename objects without changing calling code.

### 3.2 Stored Procedures

A **stored procedure** is a reusable block of code saved in the database that applications can call by name. Think of it like a function in any programming language—it takes inputs, does work, and returns results.

```sql
CREATE OR REPLACE PROCEDURE get_citizen_by_id(
  p_citizen_id IN VARCHAR2,      -- Input parameter
  p_results OUT SYS_REFCURSOR    -- Output parameter
) AS
BEGIN
  OPEN p_results FOR
    SELECT citizen_id, name, dob, address
    FROM citizens
    WHERE citizen_id = p_citizen_id;
END;
```

- `CREATE OR REPLACE PROCEDURE` defines (or redefines) a procedure
- `IN` parameters are inputs; `OUT` parameters are outputs
- `SYS_REFCURSOR` is Oracle's way of returning a set of rows—essentially a pointer to query results
- `OPEN ... FOR` executes a query and assigns the results to the cursor

### 3.3 Triggers

A **trigger** is code that Oracle executes automatically when a specified event occurs. Common uses include:
- **Audit triggers**: Log who changed what and when
- **Validation triggers**: Check data before allowing changes
- **Cascading triggers**: Update related tables when data changes

An **INSTEAD OF trigger** is special: it intercepts an operation on a view and replaces it with custom logic. Since views are read-only by default, INSTEAD OF triggers allow us to make views updatable.

```sql
CREATE OR REPLACE TRIGGER citizens_v_insert
INSTEAD OF INSERT ON citizens_v    -- Fires when inserting into the view
FOR EACH ROW                        -- Fires once per row
BEGIN
  -- :NEW represents the incoming data being inserted
  INSERT INTO citizens_base (name_enc, ...)
  VALUES (encrypt_name(:NEW.name), ...);
END;
```

### 3.4 Why These Concepts Matter

This architecture relies on all three:
- **Views** present decrypted data without changing how queries are written
- **Synonyms** redirect existing code to views without any code changes
- **INSTEAD OF triggers** handle writes to views, encrypting data before it reaches the base table

---

## Part 4: How Existing PL/SQL Continues to Work

This section addresses the most common concern: if data is encrypted, how can existing code still function?

### 4.1 The Three-Layer Architecture

The key insight is a three-layer structure that makes encryption invisible to existing code:

1. **Base table** (e.g., `citizens_base`): Stores encrypted data, HMAC identifiers, and fuzzy artefacts
2. **Decrypting view** (e.g., `citizens_v`): Reads from base table and decrypts on-the-fly
3. **Synonym** (e.g., `citizens`): Points to the view, using the original table name

When existing code says `SELECT * FROM citizens`, Oracle resolves this as:

```
citizens (synonym) → citizens_v (view) → citizens_base (encrypted table)
```

The view's definition includes decryption calls, so data arrives at the calling code already decrypted. The code doesn't know anything has changed.

### 4.2 Implementation: Three Steps

**Step 1: Rename the original table**

```sql
ALTER TABLE citizens RENAME TO citizens_base;
```

**Step 2: Create the decrypting view**

```sql
CREATE OR REPLACE VIEW citizens_v AS
SELECT 
  decrypt_citizen_id(citizen_id_hmac) AS citizen_id,
  decrypt_name(name_enc) AS name,
  decrypt_dob(dob_enc) AS dob,
  phonetic,
  bloom,
  lsh_buckets
FROM citizens_base;
```

**Step 3: Create the synonym**

```sql
CREATE OR REPLACE SYNONYM citizens FOR citizens_v;
```

Now any PL/SQL that references `citizens` automatically gets the view, which decrypts from the base table. **Zero changes to existing SELECT queries.**

### 4.3 Handling Writes with INSTEAD OF Triggers

The synonym→view approach handles reads. For writes, we use INSTEAD OF triggers.

When someone executes `INSERT INTO citizens (citizen_id, name, dob) VALUES ('ABC123', 'Alice Smith', '1990-05-15')`, the trigger intercepts this operation. Inside the trigger, Oracle provides access to the incoming values through a special pseudo-record called `:NEW`. Each column from the INSERT statement becomes accessible as `:NEW.column_name`:

- `:NEW.citizen_id` contains `'ABC123'`
- `:NEW.name` contains `'Alice Smith'`  
- `:NEW.dob` contains `'1990-05-15'`

The trigger uses these values to encrypt the data before writing to the base table:

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
    hmac_citizen_id(:NEW.citizen_id),   -- Hash the incoming citizen_id
    encrypt_name(:NEW.name),             -- Encrypt the incoming name
    encrypt_dob(:NEW.dob),               -- Encrypt the incoming dob
    compute_phonetic(:NEW.name),         -- Generate phonetic code from name
    compute_bloom(:NEW.name),            -- Generate Bloom filter from name
    compute_lsh(:NEW.name)               -- Generate LSH buckets from name
  );
END;
```

With this trigger, existing code can still say `INSERT INTO citizens (name, dob, ...) VALUES (...)` and the trigger transparently encrypts, hashes, and computes fuzzy artefacts before writing.

Similar triggers handle UPDATE and DELETE operations. For UPDATE, both `:NEW` (the new values) and `:OLD` (the previous values) are available.

### 4.4 Wrapper Functions for Identifier Lookups

Where synonyms cannot help is query logic. A query that says `WHERE citizen_id = :value` must change to use the HMAC wrapper:

**Before:**
```sql
SELECT * FROM citizens WHERE nino = :input_nino;
```

**After:**
```sql
SELECT * FROM citizens WHERE nino_hmac = hmac_nino(:input_nino);
```

The `hmac_nino` function calls the trusted adapter to compute the keyed hash. The database compares this against stored hashes. The actual NINO never appears in AWS.

This is a real code change, but it's mechanical and searchable. Every `WHERE identifier = :value` becomes `WHERE identifier_hmac = hmac_identifier(:value)`.

### 4.5 Summary: What Changes and What Doesn't

| Aspect | Change Required |
|--------|-----------------|
| Table references (SELECT FROM) | None (synonym handles it) |
| Column references in output | None (view decrypts them) |
| Display of sensitive values | None (view handles it) |
| Equality WHERE on identifiers | Mechanical change to HMAC wrapper |
| INSERT/UPDATE/DELETE | None (INSTEAD OF triggers handle it) |
| Business logic | None |
| Control flow | None |
| Error handling | Review for logged plaintext |
| Transaction management | None |

---

## Part 5: PL/SQL Patterns Requiring Specific Attention

Beyond the core patterns above, PL/SQL commonly performs operations that need individual consideration. This section examines each pattern with difficulty rating, recommended approach, required code changes, and performance implications.

### 5.1 Easy Patterns (Days of Effort)

#### SELECT with Display
**Difficulty**: Easy | **Effort**: Days

The synonym→view approach handles this completely. Existing SELECT queries continue working unchanged.

**Code change**: None.

**Performance**: Minimal. Decryption adds milliseconds per adapter call, but this happens only for fields actually retrieved and displayed.

---

#### Equality WHERE Clauses
**Difficulty**: Easy | **Effort**: Days

Any `WHERE identifier = :value` must use the HMAC wrapper.

**Code change**:
```sql
-- Before
WHERE nino = :input_nino

-- After  
WHERE nino_hmac = hmac_nino(:input_nino)
```

**Performance**: Minimal. HMAC computation is one adapter call. The subsequent comparison uses a standard indexed lookup.

---

#### CHECK Constraints
**Difficulty**: Easy | **Effort**: Days

Constraints like `CHECK (REGEXP_LIKE(nino, '^[A-Z]{2}[0-9]{6}[A-Z]$'))` fail when the column contains an HMAC hash instead of plaintext.

**Recommended approach**: Move validation into the INSTEAD OF trigger, where plaintext is still available, then drop the constraint from the base table.

**Code change**: Add validation logic to trigger; drop constraint.

**Performance**: Negligible. Validation in triggers performs similarly to CHECK constraints.

---

### 5.2 Easy-Medium Patterns (1-2 Weeks of Effort)

#### Logging and Debugging
**Difficulty**: Easy-Medium | **Effort**: 1-2 weeks

Statements like `DBMS_OUTPUT.PUT_LINE('Processing: ' || v_name)` expose plaintext in logs stored in AWS.

**Recommended approach**: Search for `DBMS_OUTPUT`, `UTL_FILE`, and INSERT statements to logging tables. Replace plaintext with HMAC values or remove sensitive data entirely.

**Code change**:
```sql
-- Before
DBMS_OUTPUT.PUT_LINE('Processing citizen: ' || v_name);

-- After
DBMS_OUTPUT.PUT_LINE('Processing citizen: ' || v_citizen_id_hmac);
```

**Performance**: None.

---

#### Exception Handlers
**Difficulty**: Easy-Medium | **Effort**: 1 week

Exception handlers often capture context including sensitive data:

```sql
EXCEPTION
  WHEN OTHERS THEN
    INSERT INTO error_log (context) 
    VALUES ('Failed for: ' || v_name || ', NINO: ' || v_nino);
```

**Recommended approach**: Log HMAC identifiers, operation names, and timestamps—enough to diagnose issues without exposing citizen details.

**Code change**: Remove sensitive data from error context strings.

**Performance**: None.

---

### 5.3 Medium Patterns (2-4 Weeks of Effort)

#### Triggers on Original Table
**Difficulty**: Medium | **Effort**: 2-3 weeks

Existing triggers attached to the original table may fire on the wrong object, receive encrypted data, or fail.

**Recommended approach**:
1. Inventory all triggers on affected tables
2. Categorise: audit, validation, cascading, integration
3. Migrate to INSTEAD OF triggers on the view, where plaintext is available

**Code change**: Recreate triggers as INSTEAD OF triggers on views.

**Performance**: Minimal beyond the INSTEAD OF trigger overhead.

---

#### Foreign Key Constraints
**Difficulty**: Medium | **Effort**: 2-4 weeks

Foreign keys must reference HMAC columns instead of plaintext identifiers.

**Recommended approach**:
1. Identify all foreign key relationships
2. Add HMAC columns to referencing tables
3. Backfill HMAC values via batch process
4. Update constraints to reference HMAC columns

**Code change**:
```sql
-- Before
ALTER TABLE benefits ADD CONSTRAINT fk_citizen 
  FOREIGN KEY (citizen_id) REFERENCES citizens(citizen_id);

-- After
ALTER TABLE benefits ADD CONSTRAINT fk_citizen 
  FOREIGN KEY (citizen_id_hmac) REFERENCES citizens(citizen_id_hmac);
```

**Performance**: Minimal. Index lookups on HMAC columns are identical to plaintext columns.

---

#### Partitioning
**Difficulty**: Medium | **Effort**: 2-4 weeks

Tables partitioned by encrypted columns lose partition pruning benefits. Encrypted values distribute randomly.

**Recommended approach**:
- If partitioned by record creation date: Usually continues working
- If partitioned by DOB: Partition by derived `birth_decade` instead
- If partitioned by name: Partition by phonetic code prefix

**Performance**: Losing partition pruning can cause 10x slowdown for queries that previously pruned 90% of partitions. New strategies must support key query patterns.

---

### 5.4 Medium-Hard Patterns (2-6 Weeks of Effort)

#### ORDER BY on Encrypted Columns
**Difficulty**: Medium-Hard | **Effort**: 2-4 weeks

`ORDER BY name` produces meaningless results when name is encrypted—ciphertext has no relationship to alphabetical order.

**Option 1: Sort after decryption** (for small result sets)

Remove ORDER BY from query; sort in calling application:

```sql
-- Database procedure returns unsorted
CREATE OR REPLACE PROCEDURE get_citizens_by_region(
  p_region IN VARCHAR2,
  p_results OUT SYS_REFCURSOR
) AS
BEGIN
  OPEN p_results FOR
    SELECT citizen_id, name, dob, address
    FROM citizens
    WHERE region = p_region;
    -- No ORDER BY
END;
```

```java
// Calling application sorts after retrieval
List<Citizen> citizens = citizenRepository.getByRegion(region);
citizens.sort(Comparator.comparing(Citizen::getName));
```

**Option 2: Precomputed sort key** (for large result sets)

Add a sortable column to the base table, populated by the INSTEAD OF trigger:

```sql
ALTER TABLE citizens_base ADD (name_sort_key VARCHAR2(100));

-- In INSTEAD OF trigger, add:
UPPER(SUBSTR(:NEW.name, 1, 100)) -- Stored as name_sort_key

-- Query uses sort key:
ORDER BY name_sort_key
```

**Security note**: The sort key is stored unencrypted. Assess whether alphabetical ordering exposure is acceptable.

**Option 3: Sort by non-sensitive attribute**

Change ORDER BY to use registration_date, case_number, or other non-sensitive field.

**Performance**: External sorting adds network round-trips. For <500 records, impact is minimal. For large sets, significantly slower. Precomputed keys have no query-time penalty.

---

#### Materialized Views
**Difficulty**: Medium-Hard | **Effort**: 2-6 weeks

Materialized views that contain encrypted columns produce inconsistent ciphertext on each refresh (due to encryption randomness). Aggregations produce incorrect results.

**Recommended approach**:
1. Inventory all materialized views referencing protected tables
2. Views aggregating non-sensitive columns: minimal change
3. Views storing sensitive values: redesign to store HMAC or encrypted values
4. Views aggregating by sensitive columns: use GROUP BY solutions below

**Performance**: Pre-computed benefits remain for non-sensitive data. Sensitive aggregations may need trusted environment processing.

---

### 5.5 Hard Patterns (4-8 Weeks of Effort)

#### GROUP BY and Aggregations
**Difficulty**: Hard | **Effort**: 4-8 weeks

`SELECT address, COUNT(*) FROM citizens GROUP BY address` fails because each encryption of "123 High Street" produces different ciphertext.

**Option 1: Group by non-sensitive attributes** (preferred)

Often the real question can be answered differently: "how many per postcode" instead of "how many per address".

**Option 2: Group by derived attributes**

Store phonetic codes or normalised hashes; group by those for approximate grouping.

**Option 3: Compute in trusted environment** (batch only)

Extract encrypted data, decrypt externally, aggregate, return summaries. Unsuitable for interactive queries.

**Option 4: Deterministic encryption** (use with caution)

Identical plaintexts produce identical ciphertexts. Enables grouping but weakens security—attackers can identify records with same values.

**Performance**: External grouping is slower and network-intensive. Grouping by derived attributes has no query-time penalty.

---

#### Date Arithmetic and Range Queries
**Difficulty**: Hard | **Effort**: 4-8 weeks

`WHERE dob BETWEEN '1990-01-01' AND '2000-12-31'` cannot work on encrypted dates.

**Option 1: Store derived flags** (most practical)

Precompute and store:
- `is_adult` (BOOLEAN)
- `age_bracket` (VARCHAR): 'CHILD', 'ADULT', 'SENIOR'
- `birth_decade` (NUMBER): 1980, 1990, 2000

Update via nightly batch process with adapter access.

**Option 2: Store plaintext year**

If year alone isn't sensitive, store `birth_year` unencrypted alongside `dob_enc`.

**Option 3: Query in trusted environment**

Retrieve candidates, decrypt externally, filter there. Practical only for moderate result sets.

**Performance**: Derived flags have zero query-time overhead. External filtering is 10-100x slower for large sets.

---

#### String Functions
**Difficulty**: Hard | **Effort**: 4-8 weeks

`UPPER(name)`, `SUBSTR(address, 1, 10)`, `INSTR(address, 'London')` operate on plaintext. Applying them to ciphertext produces garbage.

**Option 1: Store multiple forms**

Store both `name_enc` (original) and `name_upper_enc` (uppercase) if needed.

**Option 2: Use fuzzy search instead of INSTR**

The Bloom filter approach handles partial matching better than substring search.

**Option 3: Transform externally**

Decrypt, apply function, return result. Practical for display; expensive for filtering.

**Performance**: Storing multiple forms increases storage 2-3x and requires additional adapter calls on write.

---

#### Dynamic SQL
**Difficulty**: Hard | **Effort**: Case-by-case

`EXECUTE IMMEDIATE` with dynamically constructed column or table names requires case-by-case analysis.

**Recommended approach**:
1. Search for `EXECUTE IMMEDIATE`, `DBMS_SQL`, dynamic `OPEN ... FOR`
2. Analyse what's being dynamically constructed
3. If column might be sensitive, code must detect and use appropriate HMAC wrapper
4. Consider whether dynamic flexibility is still needed

**Performance**: Minimal beyond existing dynamic SQL overhead.

---

#### Bulk Operations (FORALL, BULK COLLECT)
**Difficulty**: Hard | **Effort**: 4-6 weeks

`FORALL` processes thousands of rows efficiently via single context switch. If each row triggers an adapter call, the bulk advantage is lost.

**Recommended approach**: Batch adapter calls—process in stages:

```sql
-- 1. Collect plaintext
SELECT name, dob BULK COLLECT INTO v_names, v_dobs FROM staging;

-- 2. Batch call to adapter (single round-trip for 1000 records)
adapter.batch_encrypt(v_names, v_encrypted_names);
adapter.batch_encrypt(v_dobs, v_encrypted_dobs);

-- 3. Bulk insert protected values
FORALL i IN 1..v_encrypted_names.COUNT
  INSERT INTO citizens_base VALUES (v_encrypted_names(i), ...);
```

**Performance**: Without batch APIs, bulk operations become 10-100x slower. With proper batch APIs, throughput remains hundreds to thousands of records per second.

---

### 5.6 Summary Table

| Pattern | Difficulty | Effort | Key Performance Impact |
|---------|------------|--------|------------------------|
| SELECT with display | Easy | Days | None |
| Equality WHERE | Easy | Days | Minimal |
| CHECK constraints | Easy | Days | None |
| Logging | Easy-Medium | 1-2 weeks | None |
| Exception handlers | Easy-Medium | 1 week | None |
| INSERT/UPDATE/DELETE | Medium | 2-4 weeks | 10-50ms per operation |
| Triggers | Medium | 2-3 weeks | Minimal |
| Foreign keys | Medium | 2-4 weeks | Minimal |
| Partitioning | Medium | 2-4 weeks | Must re-evaluate pruning |
| ORDER BY encrypted | Medium-Hard | 2-4 weeks | Varies by approach |
| Materialized views | Medium-Hard | 2-6 weeks | May need architecture change |
| GROUP BY encrypted | Hard | 4-8 weeks | May need external processing |
| Date ranges | Hard | 4-8 weeks | Derived flags maintain speed |
| String functions | Hard | 4-8 weeks | Storage multiplies |
| Dynamic SQL | Hard | Case-by-case | Depends on usage |
| Bulk operations | Hard | 4-6 weeks | Requires batch adapter APIs |

**Key insight**: Most patterns have solutions. A thorough inventory using static code analysis should be conducted early to size effort and identify high-risk areas.

---

## Part 6: How Fuzzy Search Works

One of the most sophisticated aspects of this architecture is enabling approximate name matching without exposing plaintext. When a user searches for "Alic Smth", we need to find "Alice Smith" despite the misspelling. This cannot be done on encrypted data—encryption deliberately destroys the ability to see patterns or similarity.

The solution uses a layered filtering approach that progressively narrows the candidate set before any decryption occurs.

### Layer 1: Phonetic Partitioning

Names are encoded using phonetic algorithms like Soundex or Metaphone. These algorithms assign a code based on how a name sounds, so "Alice", "Alyce", and "Alis" all produce the same phonetic code. This code is stored in plaintext (it reveals nothing meaningful about the actual name) and provides a fast first filter.

A search for "Alic" produces phonetic code "ALKS", and we immediately filter to only records with that code—typically reducing 50 million records to perhaps 200,000.

### Layer 2: Locality-Sensitive Hashing

Records passing the phonetic filter are further narrowed using locality-sensitive hashing (LSH).

Each name is pre-processed into a **Bloom filter**—a mathematical fingerprint representing which character patterns appear in the name. The name "Alice" is broken into overlapping fragments called n-grams: "ALI", "LIC", "ICE". Each fragment is hashed to a position in a bit array and set to 1. The resulting array is the Bloom filter.

Comparing Bloom filters one-by-one against 200,000 candidates is slow. LSH solves this by splitting each filter into segments and hashing each segment to produce bucket identifiers. Similar Bloom filters—meaning similar names—share at least one bucket.

Instead of comparing against all 200,000 candidates, we look up which candidates share any bucket with our search term. This typically reduces the set to a few thousand.

### Layer 3: Bloom Filter Similarity

The remaining candidates have their Bloom filters compared directly against the search term's filter. The comparison measures overlap percentage. High overlap (70%+) indicates names likely share many character patterns.

This step reduces 5,000 candidates to 200 or fewer.

### Layer 4: Decryption and Final Verification

Only now—with perhaps 200 candidates from 50 million—do we decrypt names via the trusted adapter. A final exact comparison determines actual matches.

**The critical point**: Decryption is expensive (network call to trusted environment) but happens on <0.0001% of records. The fuzzy artefacts do the heavy lifting without exposing names.

---

## Part 7: Why HMAC Instead of Plain Hashing

A natural question: why not simply hash identifiers like NINO using SHA-256?

The answer relates to the specific nature of identifiers. A NINO has a fixed format: two letters, six digits, one letter. There are approximately 450 million possible valid NINOs. With plain hashing, an attacker could compute the hash of every possible NINO in hours on commodity hardware, building a lookup table that defeats our protection.

HMAC prevents this by requiring a secret key. Without the key—which never leaves the trusted environment—the attacker cannot compute hashes. Their 450-million-entry lookup table is useless.

This secret key is sometimes called a "pepper" (as distinct from a salt). A salt is stored per-record and prevents rainbow table attacks; a pepper is secret and never stored with the data. If the database is compromised, the pepper remains secure externally.

---

## Part 8: Why Bloom Filters Cannot Be Reversed

If Bloom filters encode information about names, could an attacker reverse them?

No, for two reasons:

1. **Bloom filters are lossy.** Many different names produce the same filter. The filter says "these character patterns are present" but cannot distinguish between different combinations. It's knowing a room contains red, blue, and green balls without knowing how many of each.

2. **Verification requires what they don't have.** Even guessing the approximate pattern requires verification against either the encrypted name (can't decrypt) or the trusted adapter (no access).

The practical risk is statistical inference—an attacker might learn a name is likely 10-12 characters with common English patterns. This is far less than exposing actual names and insufficient to identify individuals.

---

## Part 9: Walking Through Common Operations

### Creating a New Citizen Record

The application collects citizen details. Before anything reaches AWS, the trusted adapter processes everything:

- Name "Alice Smith" is encrypted using AES with HSM-held key → stored as `name_enc`
- Simultaneously: phonetic code ("ALSM"), Bloom filter, and LSH buckets are generated
- NINO "QQ123456C" is processed through HMAC → stored as `nino_hmac`
- Relationship identifiers become HMACs

The database record in AWS contains only: encrypted fields, HMAC hashes, and non-reversible fuzzy artefacts.

### Looking Up a Citizen by NINO

The application receives NINO, sends to adapter, gets HMAC. This hash is used in the query:

```sql
SELECT * FROM citizens WHERE nino_hmac = :computed_hash;
```

The database finds the match. The view layer decrypts sensitive fields for display. At no point does the actual NINO appear in AWS.

### Searching by Name (Fuzzy)

Operator enters "Alic Smth". The adapter computes phonetic code, Bloom filter, and LSH buckets:

```sql
SELECT * FROM citizens 
WHERE phonetic = :phonetic_code 
  AND lsh_bucket IN (:bucket1, :bucket2, :bucket3);
```

Returns several thousand candidates. Application computes Bloom similarity scores, takes top 100. Only these go to adapter for decryption. Results displayed to operator.

The database narrowed 50 million to a handful using mathematical operations on non-sensitive artefacts. Sensitive decryption occurred only on the final shortlist.

### Updating a Citizen's Name

Citizen changes from "Alice Smith" to "Alice Johnson". Application sends new name to adapter, which encrypts it and generates new fuzzy artefacts. Database update modifies encrypted name and all artefacts. HMAC citizen ID unchanged—same person, different name.

### Querying Relationships

Find all children of a citizen. Parent's citizen ID is known. Application computes HMAC:

```sql
SELECT * FROM citizens WHERE parent_id_hmac = :parent_hash;
```

Database returns matching records. Relationship is maintained and queryable without identifiers appearing in AWS.

---

## Part 10: Integration with External Systems

The citizen database receives updates from numerous external systems—tax authorities, benefits agencies, health services, local councils—through real-time interfaces and batch transfers. Each integration pattern must flow through the trusted adapter.

### Real-Time Integration

When an external system sends a real-time update (e.g., change of address):

1. Integration layer receives inbound message with plaintext citizen data
2. Message routes to trusted adapter (outside AWS)
3. Adapter encrypts sensitive fields and generates HMACs and fuzzy artefacts
4. Protected payload written to cloud database
5. Acknowledgement flows back

**Critical point**: Plaintext never enters AWS. Integration components handling plaintext must reside in the trusted environment.

### Batch Integration

Partner agencies send nightly files with thousands or hundreds of thousands of records.

**Naive approach (problematic)**: Process each record individually. 50,000 records = 50,000 adapter round-trips = 8+ minutes of pure latency.

**Batch-optimised approach**: Adapter exposes batch APIs accepting arrays:

```
Batch flow:
1. Receive batch file (50,000 records)
2. Parse and validate
3. Group into batches of 1,000
4. For each batch:
   - Send to adapter batch API
   - Adapter encrypts, generates HMACs, computes artefacts
   - Return protected batch
5. Bulk insert to cloud database
6. Commit
```

### Adapter Scaling

Batch processing creates burst load. Nightly windows may need to process millions of records while daytime load is lighter.

**Requirements**:
- Horizontal scaling behind load balancer
- CPU optimization (AES-NI instructions)
- HSM throughput planning
- Auto-scaling for batch windows

### Integration Performance Benchmarks

| Type | Volume | Frequency | Latency Requirement |
|------|--------|-----------|---------------------|
| Real-time address change | 10,000/day | Continuous | < 500ms end-to-end |
| Nightly benefit status | 200,000 records | Daily | Complete in 2 hours |
| Weekly partner file | 500,000 records | Weekly | Complete in 4 hours |
| Annual bulk update | 5,000,000 records | Annual | Complete in 24 hours |
| Real-time death notification | 1,500/day | Continuous | < 200ms end-to-end |

### Change Data Capture (CDC)

CDC events contain plaintext and must be consumed in the trusted environment before transformation to protected format.

### Error Handling and Reconciliation

- **Idempotent operations**: Same record reprocessed produces same result
- **Partial batch recovery**: Resume from failure point, not restart
- **Reconciliation**: Periodic comparison against source systems
- **Audit trails**: Log what was received, transformed, and written—without logging plaintext

---

## Part 11: Migration Strategy

Implementation follows a phased approach maintaining operational continuity.

### Phase 1: Schema Extension (Month 1)

Add new columns without modifying originals. `citizens` gains `name_enc`, `nino_hmac`, `phonetic`, `bloom`, `lsh_buckets`. Original plaintext columns remain.

### Phase 2: Background Backfill (Months 1-2)

Batch process populates new columns. Runs off-peak, incrementally. Calls adapter for each existing record. Does not affect production.

### Phase 3: Dual-Write (Months 2-3)

Modify write operations to populate both original and protected columns. Validate new columns while original system functions normally.

### Phase 4: Read Migration (Months 3-6)

Introduce view layer and wrapper functions. Migrate read operations piece by piece—one report, one screen, one procedure. Compare old and new results. Roll back if needed.

### Phase 5: Plaintext Removal (Month 6+)

Once stable, drop original plaintext columns. Migration complete. AWS contains no plaintext sensitive data.

---

## Part 12: Security Analysis

### Threat: AWS compelled to provide data access

US authorities demand full database access. AWS complies.

**Result**: All sensitive fields are encrypted with keys that don't exist in AWS. Identifiers are HMAC hashes that can't be reversed. Fuzzy artefacts can't be reversed. Data is useless without trusted adapter.

### Threat: AWS compelled to provide encryption keys

Authorities demand keys.

**Result**: No keys exist in AWS. Keys reside only in trusted environment outside US jurisdiction. AWS cannot comply.

### Threat: Database exfiltrated

Attacker gains persistent access and copies all data.

**Result**: Encrypted fields useless without keys. HMAC hashes can't be brute-forced without key. Fuzzy artefacts non-reversible. Meaningful citizen records cannot be reconstructed.

### Threat: Brute-force identifier attack

Attacker computes hashes of all possible NINOs.

**Result**: HMAC requires secret key. Without key, attacker cannot produce hashes for comparison.

### Threat: Trusted adapter compromised

Attacker gains access to trusted environment.

**Result**: Security model fails. This is acknowledged—the trusted environment is the root of trust. However, it is a much smaller attack surface than full cloud deployment, protected with physical security, HSMs, and rigorous access controls.

### Acknowledged limitation

Traffic analysis attacks are not specifically addressed. Query pattern obfuscation could be added if this threat is prioritised.

---

## Part 13: Performance Characteristics

### Operations That Remain Fast

Most database work continues at normal speeds:
- Queries filtering on phonetic codes, LSH buckets, or HMAC values use standard index lookups
- Hash value comparisons are as fast as any string comparisons
- Joins on HMAC foreign keys perform like normal joins
- Bloom filter similarity uses efficient bitwise operations

### Operations That Incur Cost

Decryption and HMAC computation require adapter calls:
- Exact lookups: one round-trip to compute search hash (low milliseconds)
- Fuzzy searches: decryption only on small final candidate set
- Bulk operations: batch APIs reduce round-trip overhead

### Scale Example: Fuzzy Name Search

- Total database: 50,000,000 records
- After phonetic filter: ~200,000 records (0.4%)
- After LSH bucket intersection: ~5,000 records (0.01%)
- After Bloom similarity filter: ~200 records (0.0004%)
- Records requiring decryption: <50 (0.0001%)

Decryption touches <0.0001% of the database.

---

## Part 14: Operational Considerations

### High Availability

The trusted adapter must be highly available—failed adapter means no data access. Deploy with redundancy, automatic failover, and health monitoring.

### Key Management

Keys and HMAC secrets require extreme care. Establish rotation procedures, backup strategies, and disaster recovery. Loss of keys means permanent loss of data access.

Use Hardware Security Modules (HSMs) for physical protection and tamper-evident storage.

### Monitoring and Debugging

Error logs may contain encrypted values. The adapter should maintain comprehensive audit logs of all operations.

### Performance Tuning

Optimisation focuses on reducing adapter calls:
- Batch APIs
- Result caching where appropriate
- Efficient query patterns
- Proper indexing on phonetic, LSH, and HMAC columns

---

## Part 15: Cost Implications

### Storage Costs
2-3x overhead for fully protected tables (encrypted values plus fuzzy artefacts).

### Compute Costs
Dedicated trusted adapter infrastructure. HSM hardware if used. Scaling for peak loads.

### Operational Costs
Staff training on security model. Specialist expertise. More complex incident response.

### Offset by Avoided Costs
Alternative is multi-year PL/SQL rewrite with high risk. This approach preserves existing investment. Defensible security posture may be required for continued operation.

---

## Part 16: What This Approach Does Not Do

**Does not eliminate all risk.** Sufficiently resourced attacker with access to both AWS and trusted adapter could compromise the system. Goal is ensuring AWS access alone is insufficient.

**Does not preserve 100% of PL/SQL unchanged.** Some queries need HMAC wrappers. Some patterns (LIKE, range queries on encrypted) cannot work without changes. Claim is "minimal refactoring relative to full rewrite," not zero change.

**Does not make fuzzy matching perfect.** False positives and negatives exist. Layered approach mitigates but doesn't eliminate.

**Does not address all threats.** Traffic analysis, side-channel attacks, insider threats at trusted environment are not specifically mitigated.

---

## Part 17: Recommended Path Forward

### Immediate (1-2 months)
Build prototype adapter with encrypt, decrypt, HMAC. Demonstrate end-to-end with sample dataset. Validate Oracle external procedure integration.

### Short-term (3-6 months)
Implement full architecture for one core table. Complete production backfill. Migrate representative PL/SQL. Measure performance, validate security.

### Medium-term (6-18 months)
Extend to additional tables. Complete systematic PL/SQL migration. Remove plaintext columns. Establish operational procedures.

### Ongoing
Maintain key management. Monitor and tune performance. Review security posture as threats evolve.

---

## Conclusion

This architecture achieves what initially appears impossible: using AWS for citizen data while ensuring AWS cannot access that data, and preserving the vast majority of existing PL/SQL logic.

The key insight is **separation of storage from intelligibility**. AWS is an excellent place to run a database, but it does not need to understand what the database contains. By encrypting sensitive fields, hashing identifiers with secret keys, and generating non-reversible artefacts for fuzzy matching, we transform data into a form that serves all operational needs while being meaningless without access to the trusted environment.

This is not trivial, but it is achievable. It does not require rewriting decades of PL/SQL. It requires careful planning, phased execution, and ongoing operational discipline. The result is a cloud deployment that provides scaling, resilience, and operational efficiency while maintaining a genuinely defensible security posture for citizen data.

---

## Appendix: Glossary

| Term | Definition |
|------|------------|
| **HMAC** | Hash-based Message Authentication Code. A keyed hash producing consistent output for same input, but uncomputable without the secret key. |
| **Pepper** | Secret key used in HMAC, stored only in trusted environment. Never exposed even if database compromised. |
| **Bloom filter** | Space-efficient structure encoding which n-grams appear in text without revealing the text itself. |
| **LSH** | Locality-Sensitive Hashing. Technique grouping similar items into same buckets with high probability. |
| **N-gram** | Contiguous sequence of n characters. "Alice" with n=3: "ALI", "LIC", "ICE". |
| **Phonetic code** | Encoding representing how a word sounds rather than spelling. Similar-sounding names produce same code. |
| **Trusted environment** | Infrastructure outside AWS where keys reside and sensitive operations occur. On-premise or non-US sovereign cloud. |
| **Trusted adapter** | Service performing encryption, decryption, HMAC, and fuzzy artefact generation for the database. |
| **INSTEAD OF trigger** | Oracle trigger intercepting operations on views and replacing with custom logic. |
| **SYS_REFCURSOR** | Oracle data type representing a pointer to query results. |
| **Synonym** | Database alias that redirects references to another object transparently. |
