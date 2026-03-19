# Unified Paper Series: Detailed Outlines & Content Mapping

**Working Title:** *Architecting Modern Government Services*  
**Author:** Chris Hughes  
**Date:** February 2026

---

## Series Overview

| Paper | Title | Pages | Primary Audience | Dependency |
|-------|-------|-------|------------------|------------|
| **1** | Domain-Driven Design & Clean Architecture for Enterprise Systems | ~30 | Solution Architects, Tech Leads | None (Foundation) |
| **2** | Evidence-Based Identity: A Semantic Coordination Architecture | ~35 | CTOs, Enterprise Architects, Identity Specialists | References Paper 1 |
| **3** | AI-Assisted Development with Architectural Guardrails | ~15 | Engineering Leaders, Platform Teams | References Paper 1 |
| **4** | End-to-End Automated LLM-Assisted Development | ~45 | Platform Engineers, DevOps Leads | References Papers 1, 3 |
| **5** | Getting Started: A Practical Implementation Guide | ~25 | Engineering Teams, Individual Contributors | References Papers 1, 3, 4 |
| **6** | Putting It All Together: A Government Service Case Study | ~20 | All Audiences | References Papers 1-5 |

### Reading Paths

```
                         ┌──────────────────────┐
                         │      Paper 1         │
                         │   DDD & Clean Arch   │
                         │    (Foundation)      │
                         └──────────┬───────────┘
                                    │
               ┌────────────────────┴────────────────────┐
               │                                         │
               ▼                                         ▼
 ┌─────────────────────────┐            ┌─────────────────────────┐
 │        Paper 2          │            │        Paper 3          │
 │  Evidence-Based Identity│            │  LLM/RAG Guardrails     │
 │   (Domain Application)  │            │  (Framework Intro)      │
 └─────────────────────────┘            └────────────┬────────────┘
                                                     │
                                                     ▼
                                        ┌─────────────────────────┐
                                        │        Paper 4          │
                                        │  Automated Development  │
                                        │  (End-to-End Pipeline)  │
                                        └────────────┬────────────┘
                                                     │
                                                     ▼
                                        ┌─────────────────────────┐
                                        │        Paper 5          │
                                        │    Getting Started      │
                                        │  (Implementation Guide) │
                                        └─────────────────────────┘
                                                     
               ┌─────────────────────────────────────────────────────┐
               │                      Paper 6                        │
               │              Case Study: DWP                        │
               │  (Applies Papers 1-5 to Government Services)        │
               └─────────────────────────────────────────────────────┘
```

---

# Paper 1: Domain-Driven Design & Clean Architecture for Enterprise Systems

## Target Audience
Solution Architects, Technical Leads, Senior Engineers building complex distributed systems

## Purpose
Foundational reference for understanding how to decompose complex domains into maintainable, evolvable systems using DDD and Clean Architecture principles.

---

## Detailed Outline

### 1. Executive Summary (~1 page)
- Why boundaries matter in complex systems
- The hierarchy: Domain → Subdomain → Bounded Context → Aggregate → Microservice
- Clean Architecture as the internal structure
- Key insight: "You're not designing services. You're discovering natural fault lines in reality."

**Source:** NEW (synthesize from both docs)

---

### 2. The Hierarchy of Boundaries (~8 pages)

#### 2.1 Domains and Subdomains
- Domain = sphere of knowledge and activity (problem space)
- Subdomains = distinct problem areas within a domain
- Classification: Core, Supporting, Generic
- Generic subdomains becoming platform services

**Source:** LLM doc, Part 1 "The Hierarchy of Boundaries" (lines ~45-120)

#### 2.2 Bounded Contexts
- Problem space vs solution space distinction
- What is a model? (Value objects, Entities, Aggregates, Invariants, Ubiquitous Language)
- Mapping subdomains to bounded contexts (1:1, 1:many, many:1)
- When to split: linguistic clarity vs distributed systems cost

**Source:** LLM doc, Part 1 (lines ~120-200)

#### 2.3 How to Identify Bounded Context Boundaries
- Six reliable signals:
  1. Different Business Authority
  2. Different Language (Ubiquitous Language Clash)
  3. Different Change Drivers
  4. Different Invariants
  5. Different Data Lifecycles
  6. Cognitive Load Test
- The practical heuristic (3+ yes → split)

**Source:** LLM doc, Part 1 "How to Identify Bounded Context Boundaries" (lines ~200-280)

#### 2.4 Bounded Contexts and Microservice Count
- Default rule: 1 BC → 1 microservice
- Legitimate reasons for multiple microservices per BC (rare)
- The consistency test
- What people wrongly do (distributed layers antipattern)

**Source:** LLM doc, Part 1 (lines ~280-320)

#### 2.5 Aggregates
- Consistency boundaries within bounded contexts
- Aggregate roots control access
- Aggregates enforce invariants
- When multiple aggregates must cooperate: domain events and sagas
- Principle: same instant → same aggregate; eventually → saga

**Source:** LLM doc, Part 1 (lines ~130-160, scattered)

---

### 3. Context Relationships and Communication (~5 pages)

#### 3.1 The Technical Mechanics
- REST, gRPC, message queues, events, GraphQL—same everywhere
- What changes is semantic handling

#### 3.2 Intra-Context Communication
- Same model, same language, no translation needed
- DTOs for wire format, not conceptual translation

#### 3.3 Inter-Context Communication
- Different models, different languages, need semantic agreements
- Context relationship patterns:
  - Anti-Corruption Layer (ACL)
  - Shared Kernel
  - Open Host Service (OHS)
  - Customer-Supplier
  - Conformist
  - Partnership

#### 3.4 Why Microservices Don't Share Domain Code
- BC defines complete model; microservice implements a slice
- Shared domain libraries create coupling
- What they share: conceptual understanding, contracts, language
- What they don't share: entity implementations, business rules, schemas

**Source:** LLM doc, Part 1 "Context Relationships and Communication" and "The Bounded Context vs. Microservice Implementation Question" (lines ~320-420)

---

### 4. Clean Architecture Within Microservices (~8 pages)

#### 4.1 The Core Principle
- All dependencies point inward
- Business logic doesn't know what database/framework you use
- "Your business is the product. Everything else is a plugin."

#### 4.2 What is a Dependency?
- Code coupling through imports, calls, compile/runtime needs
- Direction of dependencies determines what breaks what
- Dependency inversion explained

#### 4.3 The Layers (with diagram)
- **Entities** (innermost): Core business rules and invariants
- **Use Cases**: Application-specific workflows
- **Interface Adapters**: Controllers, presenters, gateways
- **Frameworks & Drivers** (outermost): FastAPI, DynamoDB, etc.

#### 4.4 Layer-by-Layer Python Example
- Layer 1: Domain (Entities) — pure business logic
- Layer 2: Application (Use Cases) — defines interfaces, orchestrates
- Layer 3: Infrastructure — implements interfaces
- Layer 4: Interface Adapters — translation
- Layer 5: Wiring (Composition Root) — dependency injection

**Source:** LLM doc, Part 1 "Clean Architecture Within Microservices" (lines ~420-600) — the full Python examples

#### 4.5 Why Architects Should Care
- Controlled blast radius for changes
- Testability without infrastructure
- Portability across frameworks
- Refactoring without rewrites

---

### 5. Testing Strategy at Each Tier (~4 pages)

#### 5.1 The Testing Pyramid for DDD/Clean Architecture
- Diagram showing hierarchy

#### 5.2 Domain Rule Tests
- BDD-style executable specs for business invariants
- Live in domain documentation, not service code

#### 5.3 Contract Tests
- Provider tests: Does my service fulfill its promises?
- Consumer tests: Can I handle what I depend on?
- Key insight: contract tests test boundaries, not business logic

#### 5.4 Unit and Integration Tests
- Unit: Domain logic and use cases, no infrastructure
- Integration: Infrastructure adapters work correctly
- Clean Architecture enables isolation

#### 5.5 Infrastructure Contract Tests
- Helm deploys correctly, health checks work, config is correct

#### 5.6 End-to-End Journey Tests
- Expensive, brittle, use sparingly
- Critical business journeys only

**Source:** LLM doc, Part 4 "Testing Strategy at Each Tier" (lines ~1280-1380)

---

### 6. Governance and Team Alignment (~4 pages)

#### 6.1 Conway's Law
- Organizations design systems mirroring communication structure
- Team structure must align with architectural boundaries

#### 6.2 Team Alignment Principles
- One team per bounded context (or multiple if context is large)
- Single owning team per microservice
- Options for organizing teams around related services

#### 6.3 Platform Teams vs Domain Teams
- Platform teams own generic subdomains (Identity, Payments, Notifications)
- Domain teams own business subdomains
- Different governance: platform changes require broader impact analysis

#### 6.4 Decision Rights and Escalation
- Service teams → Service leads → Solution architects → Enterprise architects
- Clear escalation paths for cross-boundary conflicts

**Source:** LLM doc, Part 7 "Organizational Structure" (lines ~1900-2000)

---

### 7. Conclusion (~1 page)
- The hierarchy provides vocabulary for discussing architecture
- Boundaries exist because human minds have limits
- Microservices are the deployment reflection of natural fault lines
- Next steps: Apply to specific domains (Paper 2) or tooling (Paper 3)

---

### Appendix A: Glossary
- Domain, Subdomain, Bounded Context, Aggregate, Entity, Value Object, Invariant, Ubiquitous Language, Anti-Corruption Layer, Shared Kernel, etc.

### Appendix B: Quick Reference Diagrams
- The hierarchy diagram
- Clean Architecture circles
- Testing pyramid
- Context relationship patterns

---

# Paper 2: Evidence-Based Identity: A Semantic Coordination Architecture

## Target Audience
CTOs, Enterprise Architects, Identity Specialists, Government Digital Leaders

## Purpose
Technical architecture for evidence-based identity coordination using semantic technologies, explicitly framed using DDD concepts from Paper 1.

## Prerequisites
Paper 1 recommended for full understanding of DDD concepts; key terms summarized inline.

---

## Detailed Outline

### 1. Executive Summary (~2 pages)
- UK government coordination failures despite major investments
- Core challenge: semantic translation and probabilistic reasoning, not just technical integration
- Core technical argument: RDF triples + semantic translation + probabilistic clustering
- Estonia demonstrates viability; UK requires extensions
- This paper applies DDD principles to the identity coordination domain

**Source:** Architect's Reference, Executive Summary (lines ~1-25) + NEW framing

---

### 2. The Systematic Failure Pattern (~6 pages)

#### 2.1 Why Shared Schema Approaches Fail
- The repeated mistake and why it fails
- Semantic diversity isn't a bug—it's policy
- Facades emerge; standardization collapses

**Source:** Architect's Reference, Problem 1 (lines ~30-65)

#### 2.2 The Three Fatal Assumptions
1. Identity is something you can "verify" (actually: evidence assessment)
2. Attributes are properties of identity (actually: conclusions from evidence)
3. Integration means making systems talk (actually: semantic problem)

**Source:** Architect's Reference, The Three Fatal Assumptions (lines ~65-95)

#### 2.3 Identity Without Universal Identifier
- The circular dependency: can't create identity without evidence, can't correlate evidence without identity
- Current state: 6+ document uploads, separate verification systems, no reuse

**Source:** Architect's Reference, Problem 2 (lines ~95-115)

#### 2.4 Evidence Processing Without Context
- What traditional systems store vs what they lose
- Example: HMRC RTI → relational database → context destroyed
- Lost: WHO, HOW, WHEN, CONFIDENCE, semantic context

**Source:** Architect's Reference, Problem 3 (lines ~115-145)

---

### 3. The Technical Breakthrough (~8 pages)

#### 3.1 Core Insight: Model Assertions, Not Facts
- Traditional: evidence records facts
- Evidence-based: evidence records assertions (X asserts Y about Z)
- What the RDF model preserves that relational destroys

**Source:** Architect's Reference, Part II intro + X-Y-Z Model (lines ~145-185)

#### 3.2 Three-Layer Confidence Model
- **Layer 1:** Evidence Quality = Source × Verification × Transmission
  - Examples: HMRC RTI (0.88), Utility bill (0.85), Self-report (0.13)
- **Layer 2:** Evidence-Identifier Binding
  - NINO (0.98), Driving license (0.96), "C Hughes" utility bill (0.45)
- **Layer 3:** Cluster Matching (system's inference work)
  - Fellegi-Sunter probabilistic record linkage
- Overall Confidence = Layer 1 × Layer 2 × Layer 3
- Critical distinction: Layer 2 is property of evidence; Layer 3 is system inference

**Source:** Architect's Reference, Three-Layer Confidence Model (lines ~185-240)

#### 3.3 Probabilistic Identity Clustering
- No master identity record
- Clusters represent system inferences, not source facts
- Confidence evolves as evidence accumulates
- Two-track processing: Fast track (95% pre-bound) vs Full resolution (5%)

**Source:** Architect's Reference, Probabilistic Identity Clustering (lines ~240-290)

#### 3.4 Semantic Translation Without Standardization
- Example: HMRC income vs Universal Credit income
- Bilateral mappings with confidence penalties
- Governance: version control, statistical validation, appeal monitoring
- Mapping counts by pattern: P1 (100-500), P2 (thousands), P3 (minimal)

**Source:** Architect's Reference, Semantic Translation (lines ~290-330)

---

### 4. Social Evidence: Community Vouching (~4 pages)

#### 4.1 The Inclusion Problem
- 27% lowest-income households lack passports
- Asylum seekers granted refugee status can't verify identity
- Traditional verification creates systematic exclusion

#### 4.2 Vouching as Structured Evidence
- Follows identical triple structure as organizational evidence
- Same three-layer confidence model applies
- Multiple independent vouches create triangulation

#### 4.3 Fraud Prevention: Five-Layer Detection
1. Geographic and temporal clustering
2. Circular vouching detection
3. Behavioral tracking
4. Cross-reference validation
5. Asymmetry analysis
- Real-world example: 15-person fraud ring detected

#### 4.4 Trust Propagation
- Well-verified community members become anchor points
- System tracks vouch accuracy over time

**Source:** Architect's Reference, Social Evidence section (lines ~330-420)

---

### 5. The Domain Model: DDD Applied to Identity (~6 pages) — NEW SECTION

*This section explicitly frames the architecture using DDD concepts from Paper 1*

#### 5.1 The Identity Domain Landscape
- Identity as a federation of bounded contexts, not one system
- Diagram showing the domain decomposition

#### 5.2 Key Bounded Contexts

**Identity Bounded Context** (Platform Service - Generic Subdomain)
- Aggregates: Identity Profile, Claimed Identity, Verification Session
- Microservices: identity-resolution-service
- Invariant: one verified identity per real-world person

**Evidence Bounded Context** (Platform Service - Generic Subdomain)
- Aggregates: Evidence Item, Extracted Attributes, Authenticity Score
- Invariants: evidence immutable once ingested, evidence independent of identity

**Trust & Provenance Bounded Context** (Platform Service)
- Aggregates: Issuer, Trust Tier, Verification Capability
- Invariants: issuers validated before trusted, trust tiers immutable for issued evidence

**Resolution & Corroboration Bounded Context**
- Aggregates: Evidence Link, Resolution Decision, Identity Graph
- Compute-intensive graph-based reasoning

**Customer360 (Shared Kernel)**
- Read model of verified attributes
- Co-owned by all consuming contexts
- Provides verified relationships, not policy interpretations

**Eligibility Bounded Contexts** (one per benefit - Core Subdomains)
- Aggregates: Eligibility Request, Policy Rule Set, Decision Result
- Each benefit has own eligibility-decision-service

**Source:** LLM doc, Part 2 "Key Bounded Contexts" (lines ~830-1000) — adapted for identity focus

#### 5.3 The Chicken-and-Egg Problem Solved
- Claimed Identity as the anchor
- Evidence bound to Claimed Identity, not verified person
- Resolution determines: Merge, Split, Create, or Ambiguous
- Evidence preservation through resolution

**Source:** LLM doc, Part 2 "The Chicken-and-Egg Problem" (lines ~750-830)

#### 5.4 Customer360: The "360 View" Challenge
- Different benefits define "customer" differently
- Place relationships vs "address"
- Verified relationships with confidence; policy interprets meaning

**Source:** LLM doc, Part 2 Customer360 discussion (lines ~800-850)

#### 5.5 Context Relationships in the Identity Domain
- Identity → Evidence: ACL (evidence doesn't know identity exists)
- Eligibility → Customer360: Shared Kernel (read-only consumption)
- Business Subdomains → Platform Services: API contracts only

---

### 6. The Three Coordination Patterns (~6 pages)

#### 6.1 Pattern 1: Internal Organizational Coordination
- Scope, architecture (single RDF store, microservices, Kafka)
- Scaling characteristics (2B triples, single node handles 50B)
- Key services (10 core services described)
- Example flow: Sarah's Housing Benefit

**Source:** Architect's Reference, Pattern 1 (lines ~420-480)

#### 6.2 Pattern 2: Cross-Government Coordination
- Scope, X-Road federated query model
- Three-layer infrastructure per organization
- Query flow with semantic translation
- Confidence composition through the journey
- Governance: legal basis, purpose declaration, citizen dashboards

**Source:** Architect's Reference, Pattern 2 (lines ~480-520)

#### 6.3 Pattern 3: Business Interactions (Verifiable Credentials)
- W3C Verifiable Credentials + DIDs
- Government issues → Citizen wallet → Business verifies
- Zero-knowledge proofs for selective disclosure
- Trust infrastructure: registries, revocation, key recovery
- Bidirectional credential flow

**Source:** Architect's Reference, Pattern 3 (lines ~520-600)

---

### 7. Technical Architecture Details (~3 pages)

#### 7.1 Data Model (RDF + OWL)
- Core namespaces
- Evidence, Identity Cluster, Decision schemas

#### 7.2 Technology Stack
- Storage: Apache Jena Fuseki, PostgreSQL, Redis
- Events: Kafka
- Observability: Prometheus/Grafana, Jaeger, ELK

#### 7.3 Data Flow Example
- Event-driven flow from ingestion through notification

**Source:** Architect's Reference, Part IV (lines ~600-650)

---

### 8. Security & Operational Resilience (~4 pages)

#### 8.1 Novel Security Threats
- Semantic manipulation (insider modifies mappings)
- Confidence threshold gaming (adversary learns thresholds)
- Privacy inference (graph structure reveals sensitive info)
- Query DoS (complex queries exhaust resources)

#### 8.2 Failure Modes & Degradation
- Triple store unavailability
- Ambiguous identity clustering
- Semantic translation errors
- Contradictory evidence
- Cascade failures across patterns

#### 8.3 Operational Resilience Principles
- Redundancy, isolation, observability, recovery

**Source:** Architect's Reference, Part V (lines ~650-750)

---

### 9. Implementation Strategy (~3 pages)

#### 9.1 Phased Delivery
- Phase 0: PoC (4-6 months)
- Phase 1: MVP Pilot (12-18 months)
- Phase 2: Production Scale (18-24 months)

#### 9.2 Risk Mitigation
- Technical risks table
- Organizational risks table
- Delivery risks table

**Source:** Architect's Reference, Part VI (lines ~750-820)

---

### 10. Key Technical Decisions (~4 pages)

#### 10.1 Why RDF Over Relational?
- What relational cannot do
- What RDF enables
- Trade-offs

#### 10.2 Why Probabilistic Identity Over Master Record?
- Why master record fails without universal ID
- How probabilistic clustering handles reality

#### 10.3 Why Semantic Translation Over Standardization?
- Standardization failures
- Semantic translation benefits

#### 10.4 Why Three Patterns, Not One?
- Each pattern addresses different coordination constraints

**Source:** Architect's Reference, Part VII (lines ~820-920)

---

### 11. Conclusion (~2 pages)

#### 11.1 What This Architecture Solves
- Problems relational databases cannot address
- Problems standardization cannot address
- Problems manual coordination cannot address

#### 11.2 What Estonia Proved (and Didn't)
- X-Road transport works
- UK must solve additional challenges: probabilistic identity, 50× scale, 100+ orgs
- NINO cannot serve as universal identifier

#### 11.3 The Realistic Assessment
- What's proven, what's uncertain, what's required

**Source:** Architect's Reference, Conclusion (lines ~920-980)

---

### Appendix A: Glossary
- Evidence, Assertion, Triple, Cluster, Confidence, Semantic Translation, etc.

### Appendix B: Worked Examples
- Complete evidence flow for Sarah's housing benefit
- Cross-government query example
- Verifiable credential issuance and verification

### Appendix C: References
- Estonia X-Road documentation
- W3C Verifiable Credentials
- RDF/OWL specifications
- Fellegi-Sunter algorithm

---

# Paper 3: AI-Assisted Development with Architectural Guardrails

## Target Audience
Engineering Leaders, Platform Teams, DevOps Engineers, Solution Architects adopting LLM tools

## Purpose
Strategic framework for adopting LLMs in microservice development while preserving architectural integrity.

## Prerequisites
Paper 1 recommended; key DDD concepts summarized in Section 2.

---

## Detailed Outline

### 1. Executive Summary (~2 pages)
- LLMs promise acceleration; uncontrolled usage creates chaos
- The framework: Service-Scoped RAG + Four-Tier Model + CI/CD Guardrails
- Token cost reduction (95%), boundary preservation, scale to hundreds of services
- The choice: architectural control or technical debt

**Source:** LLM doc, Executive Summary (lines ~1-40)

---

### 2. Architectural Foundations (Summary) (~4 pages)

*Condensed from Paper 1 for standalone readability; references Paper 1 for depth*

#### 2.1 The Hierarchy in Brief
- Domain → Subdomain → Bounded Context → Aggregate → Microservice
- Table summarizing each level
- "For the complete hierarchy, see Paper 1, Section 2"

#### 2.2 Why Boundaries Exist
- Human minds have limits
- Teams need parallel work
- Systems must evolve
- Boundaries make comprehension possible

#### 2.3 Clean Architecture in Brief
- Dependencies point inward
- Business logic doesn't know infrastructure
- Layers: Entities, Use Cases, Adapters, Frameworks
- "For implementation details, see Paper 1, Section 4"

**Source:** LLM doc, Part 1 — heavily condensed with Paper 1 references

---

### 3. The Problem: Why LLMs Break Architecture (~6 pages)

#### 3.1 LLMs Don't Respect Boundaries
- See patterns and connections everywhere
- Trained to be "helpful" beyond what's asked
- Perfect recall—every file equally accessible

#### 3.2 The Catastrophic Tinkering Cascade
- Example: optimize one query → touch six services → success rate drops
- Risk isn't gradual erosion; it's rapid destabilization
- No protective friction like humans have

#### 3.3 The Three Dimensions of Failure
1. Boundary violations compound
2. Costs explode exponentially
3. Quality degrades paradoxically (more context → worse results)

#### 3.4 The Conversation Scope Problem
- Conversations accumulate irrelevant history
- Token cost compounds across unrelated tasks
- Solution: deliberate conversation scoping

#### 3.5 Why Traditional Approaches Fail
- Better prompts: LLMs are helpful, not obedient
- Code review: wrong point in workflow
- Single-folder limits: still too much context

**Source:** LLM doc, Part 1 "Why LLMs Make Boundaries More Critical" through end of Part 1 (lines ~580-700)

---

### 4. The Solution Architecture (~8 pages)

#### 4.1 The Core Principle
- Give the LLM exactly what it needs, nothing more
- Make violations physically impossible

#### 4.2 Understanding RAG
- You don't train your own model
- Three ways to give LLM knowledge: training, context stuffing, RAG
- Why RAG is right for microservices
- Practical implication: infrastructure engineers, not data scientists

**Source:** LLM doc, Part 3 "Understanding RAG" (lines ~1150-1200)

#### 4.3 Why Source Code? Why Not Cut Out the Middle Man?
- Source code is for humans, not computers
- Boundaries exist for human comprehension
- LLMs augment human capability, not replace judgment

**Source:** LLM doc, Part 3 (lines ~1200-1230)

#### 4.4 How Service-Scoped RAG Works
- Traditional (400K tokens, 90% noise) vs Scoped (25K tokens, high signal)
- Semantic search across pre-indexed service representation
- Multi-turn conversations that don't collapse

#### 4.5 The Four-Tier Architectural Model
- **Tier 1: Domain Architecture** — bounded contexts, domain language
- **Tier 2: Microservice Architecture** — contracts, inter-service communication
- **Tier 3: Clean Architecture** — internal service structure
- **Tier 4: Infrastructure** — deployment, platforms
- Diagram showing tiers and their relationships

#### 4.6 Design Phase vs Evolution Phase
- Design: broad RAG, LLM proposes, humans decide
- Evolution: narrow RAG, LLM implements within constraints
- Explicit transition when contracts/boundaries are established

#### 4.7 The Critical Insight
- RAG follows architecture; architecture does not follow RAG
- Design wide, contract sharp, build narrow, evolve safely

**Source:** LLM doc, Part 3 (lines ~1080-1280)

---

### 5. Implementation: Building the Infrastructure (~8 pages)

#### 5.1 Building the RAG Index
- Deciding what belongs (manifest-driven)
- Chunking code and documentation
- Creating embeddings
- One index per service

#### 5.2 Keeping RAG Current
- Automatic updates on merge/release/nightly
- Incremental processing
- What evolves by phase (design vs evolution)
- What never goes into Service RAG

**Source:** LLM doc, Part 4 "Building the RAG Index" and "Keeping RAG Current" (lines ~1380-1440)

#### 5.3 The Service Manifest
- Machine-readable architectural truth
- Schema: microservices, paths, dependencies, contracts, owners
- Complete manifest example

**Source:** LLM doc, Part 4 + Part 5 manifest examples (lines ~1440-1500, 1600-1650)

#### 5.4 Layered Enforcement Controls
- Diagram: 6 layers of defense
1. RAG scoping (95% blocked)
2. Path validation (4% blocked)
3. Dependency checking (0.5% blocked)
4. Pre-flight diff guard (0.4% blocked)
5. CODEOWNERS (0.09% blocked)
6. Contract tests (0.01% caught)

#### 5.5 The LLM Workflow Step by Step
1. Service identification
2. Semantic search
3. Intelligent re-ranking
4. Context assembly
5. Reasoning
6. Diff proposal
7. Automated validation
8. Application and testing

**Source:** LLM doc, Part 4 "Layered Enforcement Controls" and "The LLM Workflow" (lines ~1440-1560)

---

### 6. Testing Strategy Integration (~4 pages)

#### 6.1 Testing at Each Tier (Brief)
- Domain Rule Tests
- Contract Tests (Provider + Consumer)
- Unit and Integration Tests
- Infrastructure Contract Tests
- End-to-End Journey Tests
- "For complete testing framework, see Paper 1, Section 5"

#### 6.2 Testing and LLM Assistance
- LLM generates code AND tests
- Tests become executable specifications
- Safety net: code must prove correctness before merging

**Source:** LLM doc, Part 4 "Testing Strategy at Each Tier" (lines ~1280-1380) — condensed

---

### 7. Data Governance and LLM Assistance (~3 pages)

#### 7.1 Each Microservice Owns Its Data
- Database-per-service, never direct access
- Data contracts versioned and tested

#### 7.2 RAG and Data: What LLMs See
- Production data NEVER in RAG or prompts
- RAG contains schemas, not records
- Synthetic/anonymized data for examples

#### 7.3 Schema Evolution
- Follows contract versioning
- Backwards-compatible migrations
- LLM can generate migrations within rules

#### 7.4 Handling Cross-Service Data Needs
- API calls, event consumption, data passed in
- LLM sees contracts, not implementation

**Source:** LLM doc, Part 4 "Data Governance" (lines ~1500-1570)

---

### 8. Practical Examples (~10 pages)

#### 8.1 Starting From Scratch: Greenfield Project
- Phase 1: Discovering the domain (broad RAG)
- Phase 2: Defining aggregates and microservices
- Phase 3: Establishing infrastructure
- Phase 4: Creating Clean Architecture templates
- Phase 5: Defining contracts first
- Phase 6: Implementing within boundaries

**Source:** LLM doc, Part 5 "Starting From Scratch" (lines ~1580-1720)

#### 8.2 Migrating an Existing System: Strangler Pattern
- Phase 1: Build framework infrastructure (months 1-2)
- Phase 2: Map and document existing system (months 2-3)
- Phase 3: Define target architecture (month 3)
- Phase 4: Implement strangler pattern (months 4-8)
- Phase 5: Iterate and extract more (months 9-18)

**Source:** LLM doc, Part 5 "Migrating an Existing System" (lines ~1720-1800)

#### 8.3 Adding to an Established System: Feature Addition
- Domain question → Contract impact → Implementation → Infrastructure → CI/CD
- Detailed walkthrough: subscription refund feature
- Costs and timing breakdown

**Source:** LLM doc, Part 5 "Adding to an Established System" (lines ~1800-1880)

#### 8.4 Integrating with Platform Services
- Cross-subdomain dependency pattern
- Discovering the platform contract
- Implementing Anti-Corruption Layer
- Contract testing across boundaries
- Governance and communication

**Source:** LLM doc, Part 5 "Integrating with a Shared Platform Service" (lines ~1880-1950)

---

### 9. The Economics (~3 pages)

#### 9.1 The Cost Comparison
- 100 services, 10 changes/month, 4 interactions/change
- Without framework: ~£453K/year
- With framework: ~£37K/year
- Savings: £416K/year

#### 9.2 What the Numbers Don't Capture
- Architectural debt accumulation vs integrity preservation
- Strategic advantage, not just cost savings

**Source:** LLM doc, Part 6 (lines ~1950-2000)

---

### 10. Implementation Roadmap (~4 pages)

#### 10.1 Phase One: Foundation (Months 1-4)
- Platform engineering (months 1-2)
- Pilot with real teams (months 3-4)
- Success criteria

#### 10.2 Phase Two: Expansion (Months 5-8)
- Controlled expansion (months 5-6)
- Infrastructure and automation (months 7-8)

#### 10.3 Phase Three: Organizational Scale (Months 9-18)
- Scaling adoption (months 9-12)
- Full organizational adoption (months 13-18)

**Source:** LLM doc, Part 7 "Implementation Roadmap" (lines ~2000-2050)

---

### 11. Governance and Organization (~4 pages)

#### 11.1 Team Alignment
- Ownership aligned to boundaries
- Platform teams vs domain teams
- Decision rights at each level

#### 11.2 Governance That Scales
- Automation reviews structure; humans review judgment
- Each tier has clear ownership
- Risk-based approval (largely automated)

#### 11.3 Critical Success Factors
- Executive sponsorship
- Dedicated platform team
- Contract testing culture
- CI/CD investment
- Metrics-driven approach
- Team training

#### 11.4 Early Warning Signs
- Teams bypassing RAG
- CI pass rates dropping
- Token costs rising
- Violations increasing

**Source:** LLM doc, Part 7 "Organizational Structure" and "Critical Success Factors" (lines ~1900-2000, 2050-2080)

---

### 12. Conclusion (~1 page)
- LLM adoption is inevitable
- Uncontrolled → architectural debt
- Framework → sustained productivity with integrity
- The choice: adopt with discipline or without

**Source:** LLM doc, Conclusion (lines ~2080-2092)

---

### Appendix A: Service Manifest Schema
- Complete JSON schema with annotations

### Appendix B: CI/CD Pipeline Templates
- GitHub Actions / GitLab CI examples

### Appendix C: RAG Configuration Examples
- Vector database setup
- Embedding model selection
- Chunking strategies

---

# Paper 4: End-to-End Automated LLM-Assisted Development

## Target Audience
Platform Engineers, DevOps Leads, Senior Engineers building LLM-assisted development infrastructure

## Purpose
Detailed technical guide for implementing the full automation pipeline: from domain discovery through service generation, testing, and deployment.

---

## Detailed Outline

### 1. Executive Summary (~1 page)
- Why automation matters at scale
- The pipeline: Discovery → Generation → Testing → Deployment
- Key insight: "Automation reviews structure; humans review judgment"

---

### 2. Automated Domain Discovery (~4 pages)

#### 2.1 LLM-Assisted Domain Analysis
- Using LLMs to identify bounded contexts from requirements
- Extracting ubiquitous language glossaries
- Proposing aggregate boundaries

#### 2.2 Manifest Generation
- Auto-generating service manifest entries
- Dependency inference
- Contract scaffolding

---

### 3. Automated Service Generation (~6 pages)

#### 3.1 Template-Based Scaffolding
- Clean Architecture templates
- Language-specific patterns (TypeScript, Python, Java)
- Test skeleton generation

#### 3.2 Contract-First Code Generation
- OpenAPI → Controllers/Handlers
- Event schemas → Publishers/Subscribers
- Database schemas → Repositories

#### 3.3 Infrastructure as Code Generation
- Kubernetes manifests from service definitions
- Terraform modules
- CI/CD pipeline configuration

---

### 4. Automated Testing Pipeline (~6 pages)

#### 4.1 Test Generation
- Unit tests from domain logic
- Integration tests from contracts
- Contract tests from API specifications

#### 4.2 Architectural Fitness Functions
- Layer dependency validation
- Path ownership enforcement
- Import restriction checking

#### 4.3 Continuous Validation
- Pre-commit hooks
- CI pipeline integration
- Automated architectural drift detection

---

### 5. Automated Deployment (~4 pages)

#### 5.1 Progressive Delivery
- Canary deployments
- Feature flags with LLM-generated configs
- Automated rollback triggers

#### 5.2 Environment Management
- Development, staging, production parity
- Secret management
- Configuration drift detection

---

### 6. The Complete Development Lifecycle (~4 pages)

#### 6.1 From Idea to Production
- Step-by-step walkthrough
- Human checkpoints vs automated gates
- Feedback loops

#### 6.2 Maintenance and Evolution
- Handling breaking changes
- Contract versioning automation
- Deprecation workflows

---

### 7. Conclusion (~1 page)
- The automation pyramid
- Starting points for different maturity levels
- Measuring success

---

# Paper 5: Getting Started — A Practical Implementation Guide

## Target Audience
Engineering Teams, Individual Contributors, anyone implementing the framework

## Purpose
Step-by-step practical guide to implementing the framework, with concrete tool recommendations and week-by-week milestones.

---

## Detailed Outline

### 1. Executive Summary (~1 page)
- What you'll build in 8 weeks
- Minimum viable framework
- Key insight: "Start with one service, prove the pattern, then expand"

---

### 2. Tool Selection (~3 pages)

#### 2.1 RAG Infrastructure
- Vector database options (ChromaDB, Pinecone, Weaviate)
- Embedding models (OpenAI ada-002, Cohere, open-source)
- Cost comparison

#### 2.2 LLM Selection
- GPT-4 vs Claude vs Gemini for code assistance
- Self-hosted options
- API cost management

#### 2.3 CI/CD Tooling
- GitHub Actions templates
- GitLab CI alternatives
- Contract testing frameworks (Pact, Spring Cloud Contract)

---

### 3. Week-by-Week Implementation (~8 pages)

#### 3.1 Week 1-2: Foundation
- Create service manifest schema
- Set up vector database
- Build first RAG index

#### 3.2 Week 3-4: First Service
- Pick pilot service
- Implement CI guardrails
- Train pilot team

#### 3.3 Week 5-6: Contract Testing
- Define API contracts
- Implement contract tests
- Integrate with CI

#### 3.4 Week 7-8: Team Rollout
- Document patterns
- Train additional teams
- Measure and iterate

---

### 4. Common Pitfalls (~3 pages)

#### 4.1 Technical Pitfalls
- RAG index staleness
- Over-broad context retrieval
- Contract test brittleness

#### 4.2 Organisational Pitfalls
- Insufficient executive sponsorship
- Skipping the pilot phase
- Under-investing in training

---

### 5. Measuring Success (~2 pages)

#### 5.1 Key Metrics
- Token cost per interaction
- Boundary violation rate
- CI pass rate
- Developer satisfaction

#### 5.2 Maturity Model
- Level 1: Basic RAG
- Level 2: Service scoping
- Level 3: Full automation

---

### 6. Conclusion (~1 page)
- Checklist for launch readiness
- Resources and references
- Community and support

---

# Paper 6: Putting It All Together — A Government Service Case Study

## Target Audience
All audiences — this paper demonstrates Papers 1-5 applied to a real-world problem

## Purpose
Show how DDD, Clean Architecture, RAG scoping, and CI/CD guardrails combine in practice through a comprehensive case study of UK government service modernisation.

---

## Detailed Outline

### 1. Introduction (~1 page)
- Why a case study matters
- What we'll demonstrate from each paper
- The DWP modernisation challenge

---

### 2. The Problem Space (~2 pages)

#### 2.1 Understanding DWP
- Multiple benefit types (UC, Pension Credit, Disability, etc.)
- Modernisation goals: fraud reduction, faster decisions, customer experience
- The "god-model" antipattern to avoid

#### 2.2 The Domain Federation
- DWP as federation of domains, not one domain
- Business domains vs platform services
- The architectural shift: "What do we know?" before "Is this person eligible?"

---

### 3. Architectural Decomposition (~4 pages)

#### 3.1 Business Domains and Platform Services
- Benefits, Pensions, Child Maintenance, Employment Support
- Identity, Payments, Notifications, Evidence
- Core vs generic subdomain classification (Paper 1)

#### 3.2 The Two-Trust-Axis Architecture
- Identity match axis
- Issuer trust axis
- Evidence confidence computation

---

### 4. The Identity Challenge (~3 pages)

#### 4.1 Claimed vs Verified Identity
- The chicken-and-egg problem
- Claimed Identity as digital anchor
- Resolution outcomes: Create, Merge, Split, Ambiguous

#### 4.2 Evidence Binding
- Evidence linked to Claimed Identity, not verified person
- Fraud pattern detection through divergence/convergence
- Audit trail preservation

---

### 5. Bounded Contexts in Detail (~4 pages)

#### 5.1 Platform Contexts
- Identity (resolution, verification)
- Evidence (ingestion, validation)
- Trust & Provenance (issuer management)
- Resolution & Corroboration (confidence computation)

#### 5.2 Shared Kernel
- Customer360 as read model
- Place relationships vs "address"
- Person relationships with confidence
- Policy-independent verified facts

#### 5.3 Business Contexts
- Eligibility (one per benefit type)
- Policy interpretation of shared kernel
- Payments as platform service

---

### 6. Inter-Context Communication (~2 pages)

#### 6.1 Anti-Corruption Layers
- Benefits → Identity translation
- Code examples: IdentityProfile → Claimant
- Why ACLs matter for evolution

#### 6.2 Service Manifest
- Declaring boundaries machine-readably
- Platform dependencies
- Contract references

---

### 7. Implications for LLM Assistance (~2 pages)

#### 7.1 RAG Scoping Applied
- Four-tier model mapped to DWP contexts
- Domain RAG: Benefits team sees Benefits + contracts
- Service RAG: Identity team sees implementation + consumers
- Shared Kernel RAG: Customer360 read-only

#### 7.2 Governance Implications
- Platform service change protocols
- Business service autonomy
- Contract versioning requirements

---

### 8. Practical Examples (~3 pages)

#### 8.1 Greenfield: Building from Scratch
- Domain discovery with LLM assistance
- Aggregate identification
- Contract-first design
- Implementation within boundaries

#### 8.2 Brownfield: Extracting from Monolith
- Strangler fig pattern
- Gradual manifest adoption
- Legacy system coexistence

---

### 9. Strategic Insights (~1 page)

#### 9.1 The Pattern for Federated Organisations
- National trust platform that benefits consume
- Truth emerges through bounded context chain
- Separation enables reuse, fraud detection, agility

#### 9.2 Boundaries as Enforcement Points
- LLM can't couple what it can't see
- Contract tests catch what slips through
- DDD + automation = maintainable complexity

---

### 10. Conclusion (~1 page)
- How each paper contributed
- The complete picture
- Applicability beyond government services

---

# Cross-Reference Matrix

## Paper 1 Sections Referenced by Paper 2

| Paper 2 Section | Paper 1 Reference |
|-----------------|-------------------|
| 5.1 Domain landscape | 2.1 Domains and Subdomains |
| 5.2 Bounded contexts | 2.2 Bounded Contexts |
| 5.5 Context relationships | 3.3 Inter-Context Communication |
| 5.4 Shared Kernel | 3.2-3.3 Context Relationships |

## Paper 1 Sections Referenced by Paper 3

| Paper 3 Section | Paper 1 Reference |
|-----------------|-------------------|

## Paper 1 Sections Referenced by Paper 6

| Paper 6 Section | Paper 1 Reference |
|-----------------|-------------------|
| 3.1 Business domains | 2.1 Domains and Subdomains |
| 5.1-5.3 Bounded contexts | 2.2 Bounded Contexts |
| 6.1 Anti-corruption layers | 3.3 Inter-Context Communication |
| 5.2 Shared Kernel | 3.2-3.3 Context Relationships |

## Papers 3-4 Sections Referenced by Paper 6

| Paper 6 Section | Reference |
|-----------------|-----------|
| 7.1 RAG scoping | Paper 3: Service-Scoped RAG |
| 7.2 Governance | Paper 3: CI/CD Guardrails |
| 8.1-8.2 Practical examples | Paper 4: Automated Development |
| 2.1 Hierarchy summary | 2.1-2.5 (all) |
| 2.3 Clean Architecture summary | 4.1-4.5 (all) |
| 6.1 Testing tiers | 5.1-5.6 (all) |
| 11.1 Team alignment | 6.1-6.4 (all) |

---

# Shared Glossary (All Papers)

To be maintained as a separate document referenced by all three papers:

| Term | Definition | Primary Paper |
|------|------------|---------------|
| Domain | Sphere of knowledge and activity; problem space | 1 |
| Subdomain | Distinct problem area within a domain | 1 |
| Bounded Context | Linguistic/model boundary where terms have consistent meaning | 1 |
| Aggregate | Consistency boundary; cluster of entities that change together | 1 |
| Ubiquitous Language | Precise shared vocabulary within a bounded context | 1 |
| Anti-Corruption Layer | Translation layer between bounded contexts | 1 |
| Evidence | An assertion made by an organization about a subject | 2 |
| Triple | Subject-Predicate-Object statement (RDF) | 2 |
| Confidence | Probabilistic measure of evidence reliability | 2 |
| Semantic Translation | Mapping concepts between different vocabularies | 2 |
| Identity Cluster | Probabilistic grouping of evidence about a person | 2 |
| RAG | Retrieval-Augmented Generation | 3 |
| Service-Scoped RAG | RAG index limited to single service boundary | 3 |
| Service Manifest | Machine-readable declaration of service boundaries | 3 |
| Contract Test | Test verifying service honors boundary promises | 3 |
| Domain Discovery | LLM-assisted identification of bounded contexts | 4 |
| Service Generation | Automated scaffolding of microservice code | 4 |
| Architectural Fitness Function | Automated test enforcing architectural rules | 4 |
| Claimed Identity | Unverified digital anchor for evidence gathering | 6 |
| Verified Identity | Resolved identity with confidence scores | 6 |
| Trust Axis | Dimension of evidence evaluation (identity match, issuer trust) | 6 |
| Customer360 | Shared kernel providing read-only verified attributes | 6 |

---

# Series Status

| Paper | File | Status |
|-------|------|--------|
| 1 | `Paper_1_DDD_Clean_Architecture.md` | ✅ Complete |
| 2 | `Paper_2_Evidence_Based_Identity.md` | ✅ Complete |
| 3 | `Paper_3_LLM_RAG_Guardrails.md` | ✅ Complete |
| 4 | `Paper_4_Automated_LLM_Development.md` | ✅ Complete |
| 5 | `Paper_5_Getting_Started.md` | ✅ Complete |
| 6 | `Paper_6_Case_Study.md` | ✅ Complete |

---

# Next Steps

1. **Cross-link documents** — Add "See Paper X, Section Y" references throughout
2. **Create shared glossary** — Single source of truth for terminology (extract from table above)
3. **Review consistency** — Ensure terminology is consistent across all 6 papers
4. **Add navigation** — Header/footer links between papers in the series
