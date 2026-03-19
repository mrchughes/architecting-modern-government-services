# Secure, Searchable Citizen Database in the Cloud

**An Architecture for Privacy-Preserving Storage at Scale**

*Executive Overview | March 2026*

---

## Executive Summary

We propose storing the citizen database in the cloud without ever exposing sensitive personal data to the cloud provider. All confidential fields—NINOs, dates of birth, addresses, and family relationships—are encrypted or cryptographically protected before they leave our infrastructure. The cloud only ever sees scrambled data and mathematical fingerprints that cannot be reversed into personal information.

Despite this protection, the system still supports fast searching—including fuzzy name matching ("find records that sound like 'Smith'") and exact lookups ("find this specific NINO")—across tens of millions of records.

**The key guarantee:** Even with complete access to the cloud database, an attacker, a foreign government, or the cloud provider itself cannot reconstruct personal information or relationship graphs.

---

## Why This Matters

### The Problem

Storing citizen data in public cloud infrastructure creates legal and security exposure:

- **Subpoena risk:** Foreign jurisdiction laws (such as the US PATRIOT Act) could compel cloud providers to hand over data—including encryption keys if they hold them
- **Breach exposure:** If the cloud is compromised, unprotected data is immediately usable by attackers
- **Compliance burden:** GDPR and UK data protection law require demonstrable control over personal data

Traditional cloud encryption solves some of this, but typically the cloud provider manages the keys—meaning they *could* decrypt if compelled.

### Our Approach

We take a fundamentally different position: **the cloud never holds encryption keys**. All cryptographic operations happen within our own infrastructure before data is transmitted. The cloud is treated as untrusted storage—useful for scale and availability, but never trusted with secrets.

This means:
- A cloud breach exposes only scrambled data
- Subpoenas to the cloud provider yield nothing usable
- We retain full compliance control over personal data

---

## What Gets Protected and How

We use different protection strategies depending on what the data is and how it needs to be used:

| Data Type | Protection | Why This Approach |
|-----------|------------|-------------------|
| **NINO, DOB, addresses** | Encrypted (we hold the keys) | We need to retrieve and display this data, so it must be reversible—but only by us |
| **Family relationships** | Cryptographic signatures (we hold the secret) | We need to traverse relationships, but attackers shouldn't be able to reconstruct family trees |
| **Names (for search)** | Mathematical fingerprints | We need to search, but the cloud should never see actual names |

### The Relationship Protection Problem

This deserves special attention. If we simply scrambled a NINO using a standard method, an attacker could try every possible NINO (there are only about 70 million valid combinations) and match against what's stored. This is called a brute-force attack.

We prevent this by using a technique that requires a secret key we control. Without our key, trying every combination gets the attacker nowhere—they can't even check if a guess is correct.

---

## How Search Works Without Exposing Data

The clever part of this architecture is enabling search—including fuzzy "sounds like" matching—without the cloud ever seeing actual names.

### The Concept

Think of it like a library card catalogue, but instead of book titles, each card contains a coded description of what the book sounds like, how it's spelled, and what topics it covers. You can find books that match your query, but reading the catalogue doesn't tell you the actual titles.

We create similar coded descriptions (fingerprints) for each citizen's name. These fingerprints are designed so that similar names produce similar fingerprints—allowing "fuzzy" matching—but you cannot reverse a fingerprint back into a name.

### The Search Process

When searching for a name:

1. **We compute a fingerprint locally** for the search term
2. **The cloud finds records with similar fingerprints** (this happens entirely on scrambled data)
3. **We retrieve the matching encrypted records** (typically a small handful)
4. **We decrypt and verify locally** (only now do we see actual names)

The cloud does the heavy lifting of narrowing millions of records to a few candidates, but it never sees a single actual name throughout the process.

### Performance

This layered approach is efficient at scale:

| Stage | Records | What Happens |
|-------|---------|--------------|
| Start | 67 million | Full citizen database |
| After sound-alike grouping | ~50,000 | Similar-sounding names clustered |
| After fingerprint matching | ~50 | High-probability matches |
| After local verification | 1-5 | Confirmed results |

Total query time: under 200 milliseconds for fuzzy search; under 50 milliseconds for exact NINO lookup.

---

## What "Local" Means in Practice

The architecture refers to "local" infrastructure that we trust versus "cloud" infrastructure that we don't. For a CTO, the practical question is: what counts as local?

**Local (trusted) infrastructure includes:**
- Government-controlled data centres (Crown Hosting or equivalent)
- Hardware security modules (HSMs) that physically protect our encryption keys
- Application servers that perform encryption and decryption
- Networks that don't traverse public cloud infrastructure

**Cloud (untrusted) infrastructure includes:**
- Hyperscaler storage and compute (AWS, Azure, GCP)
- Any system where the provider has theoretical access to memory or keys
- Any jurisdiction where foreign law could compel disclosure

The dividing line is **key access**: if a system can see our encryption keys, it must be inside our trusted boundary. If it can't, it's untrusted, and we treat it accordingly.

In a hybrid cloud estate, this boundary can cross physical locations. A government cloud instance (e.g., UKCloud) might be trusted if keys remain in our HSMs, while a hyperscaler instance is always untrusted.

---

## Handling Updates

Data changes regularly—addresses update, names change, relationships evolve. The architecture handles this efficiently:

| Change Type | Impact | Typical Time |
|-------------|--------|--------------|
| Address change | Re-encrypt one field | Milliseconds |
| Name change | Update search fingerprints | ~5ms per record |
| Bulk changes (e.g., 500K addresses) | Batch processing | ~30 seconds |
| Annual security refresh | Regenerate all relationship signatures | ~3 hours (scheduled maintenance) |

Routine changes happen in real-time. Bulk operations are batched during maintenance windows to minimise impact.

---

## Security and Compliance Position

| Concern | How Addressed |
|---------|---------------|
| Cloud provider breach | They have only encrypted data; keys are in our HSMs |
| Foreign government subpoena | Cloud provider cannot comply—they have nothing usable |
| Brute-force ID enumeration | Blocked by keyed signatures (not simple scrambling) |
| GDPR / UK Data Protection | Personal data never leaves UK jurisdiction or our control |
| Audit requirements | All cryptographic operations logged |

---

## Key Decisions and Trade-offs

**Why not just use cloud provider encryption?**

Cloud-managed encryption means the provider holds keys. They can be compelled to use those keys. We eliminate that risk by never sharing keys.

**Why not keep everything on-premises?**

Scale, availability, and cost. The cloud provides elastic storage and global replication that would be prohibitively expensive to build ourselves. This architecture gets cloud benefits without cloud trust.

**What's the operational overhead?**

Managing HSMs and key rotation adds complexity. We estimate ~3 hours of scheduled maintenance annually for security refreshes, plus standard HSM operational practices. This is modest compared to the compliance and security benefits.

**What if the cloud database is deleted or corrupted?**

Standard backup and replication strategies apply. The cloud is just storage—we can replicate encrypted data across regions and restore from backups. Our local infrastructure maintains the ability to decrypt and verify.

---

## Recommendation

This architecture achieves secure, scalable citizen data storage that:

- **Keeps secrets out of the cloud:** Encryption keys and raw personal data never leave our trusted infrastructure
- **Enables efficient search at scale:** Fuzzy name matching and exact lookups work across 67 million records in sub-second time
- **Meets compliance requirements:** Data remains under UK jurisdiction and our operational control
- **Handles real-world operations:** Updates, bulk changes, and security refreshes are practical and scheduled

The core design principle is treating the cloud as untrusted storage—gaining scale and availability benefits while maintaining complete control over sensitive information.

For a more detailed technical specification—including specific algorithms, parameter sizing, and infrastructure diagrams—see the companion technical architecture document.

---

*End of Executive Overview*
