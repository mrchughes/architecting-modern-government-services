# Secure, Fuzzy-Searchable Citizen Database (Cloud-Untrusted)

**A Privacy-Preserving Architecture for Government-Scale Identity Storage**

*Version 1.0 | March 2026*

---

## Executive Summary (Plain English)

We propose storing the citizen database in the cloud without ever exposing sensitive personal data. All confidential fields (like NINO, date of birth, addresses, and relationships) are encrypted or cryptographically protected locally, so the cloud only sees encrypted data and non-reversible fingerprints. This allows the system to support fast, fuzzy searches on names while preserving relationships and exact-match queries, without giving the cloud any access to encryption keys.

Using phonetic codes, Bloom filters, and locality-sensitive hashing (LSH), the system narrows searches to a small set of likely matches, making queries efficient even across millions of records. The architecture ensures that raw personal data stays under local control, mitigating legal and security risks from cloud compromise, subpoenas, or other regulatory exposures.

**Key guarantee:** Even with full access to the cloud database, an attacker cannot reconstruct personal information, relationship graphs, or perform offline brute-force attacks against protected fields.

---

## 1. Threats & Risks

The database contains sensitive biographic information (name, NINO, DOB, addresses, and relationships such as mother, father, spouse). Major risks include:

| Threat | Impact | Mitigation |
|--------|--------|------------|
| Cloud compromise or subpoenas (e.g., Patriot Act) | Exposure of encryption keys or plaintext data | Keys never leave local environment; cloud holds only ciphertext |
| Memory scraping or host-level access | Extraction of keys from compromised cloud hosts | No keys present in cloud; all cryptographic operations occur locally |
| Data leakage during fuzzy search | Exposure of raw names via fingerprints | Only non-reversible Bloom filters and LSH buckets stored; no raw text |
| Brute-force attacks on hashed IDs | Reconstruction of relationship graphs from constrained ID spaces | HMAC with locally-held keys (not simple hashing) for all relationship references |
| Bloom filter false positives | Excessive local verification overhead | Quantified filter parameters sized for DWP scale (see Section 3) |

All sensitive fields are encrypted or HMAC-protected locally, and only non-reversible fingerprints or encrypted payloads are stored in the cloud. Final verification occurs locally to prevent exposure.

---

## 2. Data Security Strategy

### Encryption vs HMAC vs Bloom Filters

| Field Type | Protection Method | Rationale |
|------------|-------------------|-----------|
| **NINO, DOB, addresses** | AES-256 encryption (local keys) | Must be retrievable; encrypted payloads stored in cloud |
| **Relationship IDs** (mother, father, spouse) | HMAC-SHA256 with local key | Prevents brute-force attack on constrained ID space (see 2.1) |
| **Names for fuzzy search** | Bloom filters + LSH | Non-reversible fingerprints enable similarity search |
| **Non-sensitive metadata** | Plaintext | Filtering and partitioning only |

### 2.1 Why Relationships Require HMAC, Not Simple Hashing

**The Problem:** Government ID spaces are constrained. NINOs follow a known format (two letters, six digits, one letter). Citizen reference numbers are sequential or semi-sequential. If we store `SHA-256(citizen_id)` as a relationship reference, an attacker with cloud access can:

1. Enumerate the entire valid ID space (computationally feasible for ~100 million citizens)
2. Hash each candidate
3. Match against stored relationship hashes
4. Reconstruct the complete relationship graph

This is a realistic attack because the ID space is small enough to brute-force.

**The Solution:** Use HMAC with a locally-held secret key:

```
relationship_reference = HMAC-SHA256(local_key, citizen_id)
```

Without the local key, the attacker cannot compute candidate HMACs, so brute-force enumeration fails. The relationship graph remains pseudonymous in the cloud—structurally visible (you can see that record A relates to record B) but semantically opaque (you cannot determine that A is citizen 12345678).

**Key management:** The HMAC key is stored in the local HSM cluster (see Section 6) and never transmitted to cloud infrastructure. Key rotation requires recomputing all relationship references, which can be batched during maintenance windows.

### 2.2 Encryption Key Management

Encryption keys for reversible fields (NINO, DOB, addresses) follow the same principle:

- **Storage:** HSM cluster within local trusted environment
- **Access:** Application servers within local boundary request encryption/decryption operations
- **Cloud exposure:** Zero—cloud receives only ciphertext
- **Rotation:** Envelope encryption pattern (data encrypted with data key, data key encrypted with master key) enables rotation without re-encrypting all records

---

## 3. Fuzzy Search & Indexing

### 3.1 Layered Filter Architecture

Each layer reduces the candidate set without touching plaintext:

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Phonetic Partition                                │
│  Reduces ~67M citizens → ~50,000 candidates                 │
│  (1,000-2,000 distinct phonetic codes for surnames)         │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: LSH Bucket                                        │
│  Reduces ~50,000 → ~500-2,000 candidates                    │
│  (20-100 buckets per partition, tuneable)                   │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Bloom Filter Comparison                           │
│  Reduces ~2,000 → ~10-50 candidates                         │
│  (Jaccard similarity on n-gram fingerprints)                │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: Local Verification                                │
│  Decrypts ~10-50 records, confirms exact matches            │
│  (Final verification on plaintext)                          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Phonetic Partitioning

Names are preprocessed locally into phonetic codes (Double Metaphone preferred over Soundex for better discrimination):

- "Smith" → `SM0`, `XMT`
- "Smythe" → `SM0`, `XMT`  
- "Schmidt" → `XMT`, `SMT`

These codes form partitions in the cloud index, narrowing fuzzy searches to similar-sounding names. For a surname corpus of ~67 million UK citizens, Double Metaphone typically produces 1,000-2,000 distinct primary codes, giving an average partition size of 30,000-70,000 records.

### 3.3 Bloom Filters: Quantified Parameters

Names are converted into character n-grams and stored as Bloom filters:

**Example:** "Alice Smith" → trigrams: `["ali", "lic", "ice", "ce ", "e s", " sm", "smi", "mit", "ith"]`

**Filter sizing for DWP scale:**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Expected elements per filter | 15 (average name produces ~15 trigrams) |
| Target false positive rate | 0.1% (1 in 1,000) |
| **Filter size (m)** | **216 bits (27 bytes)** | Formula: m = -n·ln(p) / (ln2)² |
| **Hash functions (k)** | **10** | Formula: k = (m/n)·ln2 |
| Storage per citizen | ~30 bytes (filter + overhead) |
| Total index size (67M citizens) | ~2 GB | Fits comfortably in memory |

**False positive impact:** With a 0.1% false positive rate and ~2,000 candidates after LSH filtering, we expect ~2 false positives per query reaching the local verification stage. This is acceptable overhead—decrypting 12 records instead of 10 adds negligible latency compared to network round-trips.

**Tuneable trade-off:** Increasing filter size to 512 bits reduces false positives to 0.001% but doubles index storage. The 216-bit configuration balances accuracy against memory pressure.

### 3.4 Locality-Sensitive Hashing (LSH)

Bloom filter fingerprints are processed using LSH, which assigns similar fingerprints to the same bucket:

**How it works:**
1. Each Bloom filter is treated as a binary vector
2. LSH applies multiple hash functions that preserve similarity (MinHash family)
3. Records whose Bloom filters produce the same LSH signature are placed in the same bucket
4. Query computes its LSH signature and retrieves only that bucket

**Configuration:**
- 5 hash tables with 20 bands each
- Bucket collision probability >90% for names with >70% Jaccard similarity
- Bucket collision probability <5% for names with <30% Jaccard similarity

LSH is analogous to a semantic vector index in NLP: items with similar meaning (or spelling) cluster together, enabling approximate nearest-neighbour search without exhaustive comparison.

### 3.5 Hash Index for Exact-Match Fields

Exact-match fields (HMAC'd NINO, DOB, relationship IDs) use traditional hash indices:

- Query computes HMAC locally (using local key)
- Cloud returns candidate bucket (typically 1-3 rows due to hash collisions)
- Final comparison occurs locally after decryption

**Important:** These are HMAC-based indices, not simple hash indices. The cloud cannot enumerate the index to reconstruct values.

---

## 4. Query Flow

### 4.1 Fuzzy Name Search

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────┐
│    Local     │     │      Cloud      │     │    Local     │
│  (Trusted)   │     │   (Untrusted)   │     │  (Trusted)   │
└──────┬───────┘     └────────┬────────┘     └──────┬───────┘
       │                      │                     │
   1. Compute phonetic        │                     │
      code + Bloom filter     │                     │
       │                      │                     │
       ├─────── 2. Send ──────►                     │
       │     phonetic code    │                     │
       │     + LSH signature  │                     │
       │                      │                     │
       │              3. Filter to partition        │
       │                 + LSH bucket               │
       │                      │                     │
       │              4. Compare Bloom filters      │
       │                 (cloud-side, on indices)   │
       │                      │                     │
       ◄─────── 5. Return ────┤                     │
       │     encrypted        │                     │
       │     candidate rows   │                     │
       │                      │                     │
       ├──────────────────────┼────── 6. Decrypt ───►
       │                      │          locally    │
       │                      │                     │
       │                      │         7. Verify   │
       │                      │         fuzzy match │
       │                      │                     │
```

### 4.2 Exact-Match ID Search (e.g., NINO Lookup)

1. Compute `HMAC(local_key, NINO)` locally
2. Cloud returns bucket of candidate rows (1-3 records)
3. Fetch encrypted payloads
4. Decrypt locally
5. Verify exact match on plaintext

### 4.3 Relationship Retrieval

1. Start with known citizen record
2. Read HMAC'd relationship references from decrypted record
3. Query cloud for records matching those HMACs
4. Decrypt and verify

**Note:** Relationship HMACs are computed from the related citizen's ID, so traversal requires the local HMAC key. An attacker with cloud access can see that record A references record B (the HMAC values exist), but cannot identify who those citizens are.

---

## 5. Update Strategy

The update story is critical at DWP scale. Different change types have different costs:

### 5.1 Individual Record Updates

| Change Type | Recomputation Required | Complexity |
|-------------|------------------------|------------|
| Address change | Re-encrypt address field only | Low |
| DOB correction | Re-encrypt DOB field only | Low |
| Name change | Recompute n-grams, Bloom filter, phonetic codes, LSH bucket | Medium |
| NINO correction | Re-encrypt NINO, update HMAC index | Medium |
| Relationship change | Update HMAC references in both records | Low |

**Name changes** are the most expensive because they affect multiple indices:
1. Compute new trigrams and Bloom filter
2. Compute new phonetic code (may change partition)
3. Compute new LSH signature (may change bucket)
4. Update all three indices in cloud
5. Retain old index entries temporarily for in-flight queries

**Estimated cost:** ~5ms compute per record, plus cloud round-trip. For individual changes, this is negligible.

### 5.2 Bulk Updates

Bulk updates (e.g., address changes post-redistricting, NHS number linkage exercises) require batch processing:

**Scenario:** 500,000 address changes following council boundary reorganisation

| Approach | Time | Network | Notes |
|----------|------|---------|-------|
| Serial processing | ~40 minutes | 500K round-trips | Unacceptable |
| Batched (1,000 records/batch) | ~4 minutes | 500 round-trips | Acceptable |
| Parallel batched (10 workers) | ~30 seconds | 50 round-trips/worker | Preferred |

**Batch strategy:**
1. Accumulate changes in local queue during business hours
2. Process batches during maintenance window (typically 02:00-05:00)
3. Compute all new indices locally in parallel
4. Upload index updates in compressed batches
5. Atomic swap of old→new index entries

**For index-affecting bulk changes (rare):**

If a bulk operation affects fuzzy search indices (e.g., systematic name corrections), the approach differs:

1. Build complete new index set locally
2. Upload as separate index version
3. Atomic cutover at maintenance window
4. Retain old index for rollback (7 days)

This is expensive (full reindex takes ~2 hours for 67M records on a 32-core local server) but rare—most bulk operations affect encrypted fields only.

### 5.3 Key Rotation

HMAC key rotation for relationship references:

1. Generate new HMAC key in HSM
2. Batch-recompute all relationship HMACs (~67M × 2.3 relationships average = ~154M operations)
3. Upload new HMAC index
4. Cutover (old HMACs become invalid)
5. Retain old key in HSM for emergency rollback (30 days)

**Estimated time:** ~3 hours for full rotation (compute-bound). Scheduled annually or on security event.

---

## 6. Local Trusted Environment: Architecture

The document has referred to "local" as a single trusted boundary. In practice at DWP scale, this is a distributed system:

### 6.1 What "Local" Actually Means

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOCAL TRUSTED BOUNDARY                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   DC1 (Primary) │  │  DC2 (Secondary)│  │   DC3 (DR)      │ │
│  │                 │  │                 │  │                 │ │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │ │
│  │  │ App Tier  │  │  │  │ App Tier  │  │  │  │ App Tier  │  │ │
│  │  │ (encrypt/ │  │  │  │ (encrypt/ │  │  │  │ (cold)    │  │ │
│  │  │  decrypt) │  │  │  │  decrypt) │  │  │  │           │  │ │
│  │  └─────┬─────┘  │  │  └─────┬─────┘  │  │  └───────────┘  │ │
│  │        │        │  │        │        │  │                 │ │
│  │  ┌─────▼─────┐  │  │  ┌─────▼─────┐  │  │  ┌───────────┐  │ │
│  │  │   HSM     │◄─┼──┼──►   HSM     │◄─┼──┼──►   HSM     │  │ │
│  │  │  Cluster  │  │  │  │  Cluster  │  │  │  │ (replica) │  │ │
│  │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │ │
│  │                 │  │                 │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Key Management Service (KMS)                │   │
│  │   • Master key never leaves HSM                          │   │
│  │   • Data keys distributed to app tier (encrypted)        │   │
│  │   • HMAC keys replicated across HSM clusters             │   │
│  │   • Audit log of all key operations                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Encrypted data only
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CLOUD (UNTRUSTED)                            │
│                                                                 │
│   • Encrypted payloads                                          │
│   • HMAC indices (relationship references)                      │
│   • Bloom filter indices (name search)                          │
│   • LSH bucket assignments                                      │
│   • Phonetic partition metadata                                 │
│                                                                 │
│   NO: encryption keys, HMAC keys, plaintext, or ability to     │
│       perform cryptographic operations                          │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Defining the Trust Boundary

The local trusted boundary is defined by:

| Property | Requirement |
|----------|-------------|
| **Physical security** | Crown Hosting or equivalent IL3+ data centre |
| **Network isolation** | Dedicated circuits to cloud; no shared infrastructure |
| **Key storage** | FIPS 140-2 Level 3 HSMs (minimum); Level 4 for master keys |
| **Access control** | SC-cleared operators only; dual-control for key operations |
| **Audit** | All cryptographic operations logged to immutable store |
| **Incident response** | 4-hour key revocation capability |

### 6.3 Hybrid Cloud Considerations

In a hybrid estate, the trust boundary may not align with physical data centre boundaries:

**Scenario:** Some "local" workloads run on government cloud (e.g., UKCloud, Crown Hosting Azure Stack) while the public cloud component runs on hyperscaler infrastructure.

**Principle:** The trust boundary is defined by key access, not physical location. A workload is "local" if:
1. It can request cryptographic operations from the HSM cluster
2. It runs on infrastructure where memory is not accessible to the cloud provider
3. It is subject to DWP operational control and audit

**Excluded from trust boundary:**
- Any hyperscaler-managed infrastructure where the provider has theoretical memory access
- Any system where keys could be subpoenaed under foreign jurisdiction

This definition allows government cloud expansion while maintaining the security model.

---

## 7. Security & Compliance Summary

| Requirement | How Addressed |
|-------------|---------------|
| No keys in cloud | HSM-stored keys; cloud receives only ciphertext and HMACs |
| Brute-force resistant | HMAC (not hash) for constrained ID spaces |
| Fuzzy search without plaintext | Bloom filters + LSH on n-gram fingerprints |
| Exact match without plaintext | HMAC indices with local key |
| Relationship privacy | HMAC'd references; graph structure visible but semantics opaque |
| False positive handling | Quantified Bloom filter parameters; local verification stage |
| Bulk update capability | Batched processing; index versioning for major changes |
| Key rotation | Batch recomputation with HSM-managed key lifecycle |
| GDPR Article 32 | Encryption, pseudonymisation, confidentiality measures |
| UK GDPR / Data Protection Act | Data remains under UK jurisdiction (local boundary) |

---

## 8. Conclusion / Recommendation

This architecture achieves:

**Fully cloud-untrusted storage:** Sensitive data encrypted locally; relationship references HMAC'd with locally-held keys; no cryptographic material enters cloud infrastructure.

**Scalable fuzzy search:** Phonetic partitions reduce 67M records to ~50K; LSH buckets reduce to ~2K; Bloom filter comparison reduces to ~50; local verification confirms matches. Total query latency: <200ms at scale.

**Efficient exact-match:** HMAC indices for NINO, DOB, relationship traversal. Lookup latency: <50ms.

**Quantified reliability:** Bloom filter parameters sized for 0.1% false positive rate, resulting in ~2 extra records per fuzzy query reaching local verification—acceptable overhead.

**Practical update path:** Individual changes processed in real-time; bulk changes batched during maintenance windows; key rotation achievable in ~3 hours annually.

**Defined trust boundary:** Local environment specified as HSM-backed, Crown Hosting infrastructure with SC-cleared operations—not an abstract "local" that ignores hybrid cloud realities.

**Key architectural decision:** Using HMAC rather than simple hashing for relationship references prevents brute-force reconstruction of the citizen graph, which is the primary residual risk in simpler designs.

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| **Bloom filter** | Probabilistic data structure for set membership testing; allows false positives but not false negatives |
| **HMAC** | Hash-based Message Authentication Code; keyed hash that requires the secret key to compute or verify |
| **HSM** | Hardware Security Module; tamper-resistant device for key storage and cryptographic operations |
| **LSH** | Locality-Sensitive Hashing; hash technique where similar items hash to the same bucket with high probability |
| **n-gram** | Contiguous sequence of n characters from a string; used for fuzzy matching |
| **Phonetic code** | Representation of a word's pronunciation (e.g., Soundex, Metaphone); groups similar-sounding words |

---

## Appendix B: Technology Options

| Component | Recommended | Alternatives |
|-----------|-------------|--------------|
| HSM | Thales Luna Network HSM 7 | AWS CloudHSM (if within trust boundary), nCipher |
| Bloom filter library | PyBloom, bloom-filter2 (Python); Guava (Java) | Custom implementation for precise control |
| LSH implementation | datasketch (Python); FALCONN (C++) | Custom MinHash for production tuning |
| Phonetic encoding | Double Metaphone | Soundex (simpler, less discriminating); Beider-Morse (better for non-English) |
| Encryption | AES-256-GCM | ChaCha20-Poly1305 (faster on systems without AES-NI) |

---

*End of Document*
