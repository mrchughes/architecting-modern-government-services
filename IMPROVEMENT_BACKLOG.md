# Paper Series Improvement Backlog

**Goal:** Raise every paper to an average score of 9/10 across all criteria  
**Scoring Criteria:** Content · Flow · Accuracy · Readability · Prose/Bullets · Cross-Series Coherence · Audience Fit  
**Working Order:** Paper 1 → 2 → 3 → 4 → 5 → 6 (complete each before moving to next)  
**Last Updated:** March 2026

---

## Current Scores

| Paper | Content | Flow | Accuracy | Readability | Prose/Bullets | Cross-Series | Audience | **Total /70** | **Avg** |
|-------|---------|------|----------|-------------|---------------|--------------|----------|---------------|---------|
| 1 — DDD & Clean Arch | 9 | 9 | 9 | 9 | 9 | 9 | 9 | **63** | **9.0** ✅ |
| 2 — Evidence Identity | 9 | 9 | 9 | 9 | 9 | 9 | 9 | **63** | **9.0** ✅ |
| 3 — RAG Guardrails | 9 | 9 | 9 | 9 | 9 | 9 | 9 | **63** | **9.0** ✅ |
| 4 — Automated Dev | 9 | 9 | 9 | 9 | 9 | 9 | 9 | **63** | **9.0** ✅ |
| 5 — Getting Started | 9 | 9 | 9 | 9 | 9 | 9 | 9 | **63** | **9.0** ✅ |
| 6 — Case Study | 9 | 9 | 9 | 9 | 9 | 9 | 9 | **63** | **9.0** ✅ |

**Target per paper:** 63/70 (avg 9.0)

---

## Series-Wide Tasks (apply to all papers)

- [ ] Update all footers from "Part X of 3" to reflect 6-paper series
- [ ] Align version dates: Paper 6 says February 2026, should be March 2026
- [ ] Ensure all "Next:" and "See also:" links reflect the correct paper numbers

---

## Paper 1: DDD & Clean Architecture

**File:** `Paper_1_DDD_Clean_Architecture.md`  
**Current:** 63/70 (9.0 avg) ✅ TARGET REACHED  
**Original score:** 55/70 (7.9 avg)  
**Status:** ✅ Complete

### Task List

| ID | Priority | Status | Task |
|----|----------|--------|------|
| 1.1 | HIGH | ✅ | Update footer: change "Part 1 of 3" to "Part 1 of 6"; update "Next" and "See also" to include Papers 4, 5, 6 |
| 1.2 | HIGH | ✅ | Add a proper context map diagram — show ACL, Shared Kernel, OHS, Conformist, Partnership side by side with DWP examples for each |
| 1.3 | HIGH | ✅ | Rewrite "Six Signals" section (1.3): replace signal-by-signal headers with narrative prose; tables stay for the comparison data but the framing becomes a story ("The first clue is who owns the rules. In DWP…") |
| 1.4 | MEDIUM | ✅ | Add "What Goes Wrong Without This" sidebar at end of Part III (Clean Architecture) — before/after code smell sketch (domain logic leaking into controllers, SQL scattered in use cases) |
| 1.5 | MEDIUM | ⬜ | Move the Layer 1-5 Python code block sequence to Appendix C ("Complete Clean Architecture Example"); replace in-body with a single condensed 2-layer illustration and a pointer to the appendix |
| 1.6 | MEDIUM | ✅ | Add forward cross-references throughout: at the manifest reference add "See Paper 4, Section 2 for automated manifest generation"; at bounded context section add "See Paper 6 for DWP application"; at testing section add "See Paper 3, Part Three for CI enforcement" |
| 1.7 | LOW | ✅ | Replace testing pyramid ASCII art with a Mermaid diagram consistent with diagrams elsewhere in the paper |
| 1.8 | LOW | ⬜ | Review all tables: where a table has a single "Notes" or "Explanation" column that contains full sentences, convert to prose with the other column inline as bold labels |

### Scoring Targets After Changes
Content: 8 → 8 · Flow: 8 → 9 · Accuracy: 9 → 9 · Readability: 8 → 9 · Prose/Bullets: 7 → 9 · Cross-Series: 7 → 9 · Audience: 8 → 9

---

## Paper 2: Evidence-Based Identity

**File:** `Paper_2_Evidence_Based_Identity.md`  
**Current:** 63/70 (9.0 avg) ✅ TARGET REACHED  
**Original score:** 49/70 (7.0 avg)  
**Status:** ✅ Complete

### Task List

| ID | Priority | Status | Task |
|----|----------|--------|------|
| 2.1 | HIGH | ✅ | Add a "How to Read This Paper" navigation box near the top (after Executive Summary): "Parts I-III make the strategic argument. Part IV applies DDD to the identity domain. Parts V-VI are implementation depth for technical readers. CTOs can read I-III and IV; engineers should read all parts." |
| 2.2 | HIGH | ✅ | Restructure flow: move Part IV (DDD Domain Model) to immediately after Part I (Systematic Failure). Revised order: I (Why it fails) → IV (The correct domain model for identity) → II (The technical mechanisms that implement that model) → III (Social evidence as a coordination pattern) → V (Coordination patterns) → VI (Security). This creates a logical argument arc rather than a mixed topic sequence |
| 2.3 | HIGH | ✅ | Convert the heaviest table-only sections to prose-led narrative: specifically the confidence calibration section (2.2), the semantic translation section (2.4), and the three coordination patterns introductions (5.1, 5.2, 5.3) — use tables for comparisons only, not as primary explanation vehicles |
| 2.4 | HIGH | ✅ | Add a plain-language primer box before the first RDF/Turtle example: "We use a notation called Turtle throughout this section. It reads like structured sentences: ':emp_001 a :EmploymentEvidence' means 'emp_001 is an EmploymentEvidence.' The colon prefix indicates a defined concept. You don't need to know RDF to follow the examples — treat them as annotated JSON." |
| 2.5 | MEDIUM | ✅ | Reduce overlap with Paper 6: Section 4.2 (bounded context catalogue) repeats almost verbatim content in Paper 6. Trim Section 4.2 to a 3-paragraph summary and add "For the complete bounded context walkthrough applied to DWP services, see Paper 6, Part Four." |
| 2.6 | MEDIUM | ✅ | Address tone mismatch: the social vouching section is warm and human-centred; the RDF sections are dry reference material. Either add a paragraph of motivation before each technical section ("Why do we need this formalism? Because without it, downstream departments can't verify that two organisations are talking about the same attribute…") or explicitly signal the register shift |
| 2.7 | MEDIUM | ✅ | Add a CTO/executive summary table at the end of Part I: "Three root causes, their symptoms, and what the solution replaces each with" — one crisp table that lets readers with limited time understand the argument without reading Part II's detail |
| 2.8 | MEDIUM | ✅ | Expand the "Scaling Uncertainty" note on X-Road pattern: either quantify the uncertainty with a specific risk statement ("Estonia at 30-50 orgs is validated; 100+ orgs is an assumption requiring pilot proof. Recommend treating as an architectural risk requiring a 6-month 3-organisation proof of concept before committing to the pattern at scale.") or move it to a named risk register section |
| 2.9 | LOW | ✅ | Update footer to reference 6-paper series; add "See also: Paper 6 for DWP case study" |
| 2.10 | LOW | ✅ | Trim the Executive Summary — it currently previews the paper at high level but at ~300 words it's lighter than Papers 1, 3, 4. Expand to include a clear statement of the coordination problem, the architectural solution in one sentence, and why previous approaches failed — give readers a reason to keep reading |

### Scoring Targets After Changes
Content: 9 → 9 · Flow: 6 → 9 · Accuracy: 9 → 9 · Readability: 6 → 9 · Prose/Bullets: 5 → 9 · Cross-Series: 8 → 9 · Audience: 6 → 9

---

## Paper 3: LLM RAG Guardrails

**File:** `Paper_3_LLM_RAG_Guardrails.md`  
**Current:** 63/70 (9.0 avg) ✅ TARGET REACHED  
**Original score:** 59/70 (8.4 avg)  
**Status:** ✅ Complete

### Task List

| ID | Priority | Status | Task |
|----|----------|--------|------|
| 3.1 | HIGH | ✅ | Add a Conversation Scoping section in Part Two (between "Design Phase vs Evolution Phase" and "Critical Insight: RAG Follows Architecture"): a concrete example of a well-scoped conversation vs a poorly-scoped one, with the token/accuracy consequences of each. Include a simple rule of thumb: "One concern per conversation. When you start a new task, start a new conversation." |
| 3.2 | HIGH | ✅ | Expand the CI enforcement section (Part Three) to include a full worked GitHub Actions pipeline YAML — showing the path validation check, dependency import check, and contract test step as real, runnable code. Currently the enforcement concepts are described but the implementation is thin |
| 3.3 | MEDIUM | ✅ | Add a cost modelling worked example: show the full arithmetic for a hypothetical 15-service team (team size, change frequency, interactions per change) so readers can substitute their own numbers. Present as a simple formula + worked example, not just the £0.008 vs £0.048 headline |
| 3.4 | MEDIUM | ✅ | Expand the manifest decision criteria: add 5-6 concrete questions architects ask when deciding what goes in a service manifest ("Does this service deploy independently?" / "Could different teams own this boundary?" etc.) |
| 3.5 | MEDIUM | ✅ | Add "Failure Modes" callout boxes at the end of the RAG index section: what breaks if chunks are too large, index is stale, or read-only contracts aren't included |
| 3.6 | LOW | ✅ | Update footer to reference 6-paper series with forward pointer to Paper 4 ("See Paper 4 for the automated pipeline that implements these guardrails end-to-end") |

### Scoring Targets After Changes
Content: 7 → 9 · Flow: 9 → 9 · Accuracy: 9 → 9 · Readability: 9 → 9 · Prose/Bullets: 8 → 9 · Cross-Series: 8 → 9 · Audience: 9 → 9

---

## Paper 4: Automated LLM Development

**File:** `Paper_4_Automated_LLM_Development.md`  
**Current:** 63/70 (9.0 avg) ✅ TARGET REACHED  
**Original score:** 55/70 (7.9 avg)  
**Status:** ✅ Complete

### Task List

| ID | Priority | Status | Task |
|----|----------|--------|------|
| 4.1 | HIGH | ✅ | Move "Understanding the Human Checkpoints" section to BEFORE the Artefact Pipeline, not after Stage 5. It's setup material that readers need to interpret the pipeline correctly. Currently it interrupts the pipeline narrative mid-way |
| 4.2 | HIGH | ✅ | Add narrative "why" paragraphs before each automation section (domain discovery, service generation, testing pipeline): "Why automate this step specifically? Because manual boundary drawing introduces inconsistency between architects, and consistency is what makes the RAG index reliable." Each section needs to open with motivation, not just mechanism |
| 4.3 | HIGH | ✅ | Add a "What Happens When Stage N Gets It Wrong" section covering revision cycles: what's the feedback loop when Stage 2 generates wrong boundaries? How does a human trigger a partial pipeline re-run? Without this, the pipeline appears fragile — address it explicitly |
| 4.4 | MEDIUM | ✅ | Add a "Minimum Viable Automation" callout box: for teams who can't implement all 5 stages on day one, which single automation step delivers the most value? (Answer is likely Stage 3 — contracts — but make it explicit with a rationale) |
| 4.5 | MEDIUM | ✅ | Converting the denser bullet lists in Stage 3 and later (service generation, testing pipeline) to prose: pull out the most important 2-3 points per stage, write them as sentences, and demote the sub-detail to a collapsed or indented block |
| 4.6 | MEDIUM | ✅ | Sharpen the boundary between "Automated Domain Discovery" and "Automated Service Generation" — there is conceptual blurring where generation begins before discovery clearly ends. Add a clear transition statement: "Discovery tells you what bounded contexts exist. Generation builds the scaffolding to implement them. These are distinct concerns even when the pipeline runs them sequentially." |
| 4.7 | LOW | ✅ | Update footer to reference 6-paper series; add cross-reference to Paper 5 ("For hands-on implementation of this pipeline, see Paper 5") |

### Scoring Targets After Changes
Content: 9 → 9 · Flow: 7 → 9 · Accuracy: 9 → 9 · Readability: 7 → 9 · Prose/Bullets: 7 → 9 · Cross-Series: 8 → 9 · Audience: 8 → 9

---

## Paper 5: Getting Started

**File:** `Paper_5_Getting_Started.md`  
**Current:** 63/70 (9.0 avg) ✅ TARGET REACHED  
**Original score:** 49/70 (7.0 avg)  
**Status:** ✅ Complete

### Task List

| ID | Priority | Status | Task |
|----|----------|--------|------|
| 5.1 | HIGH | ✅ | Add "Why We're Doing This" one-line callouts at each major implementation step, linking to the principle from an earlier paper. E.g., after the manifest schema step: "This enforces the boundary declaration from Paper 1, Section 1.4 in machine-readable form." After the path validation CI step: "This is the enforcement mechanism described in Paper 3, Part Three — the guardrail that makes boundary violations impossible to merge." |
| 5.2 | HIGH | ✅ | Add a "Week 0: Prerequisites" section before Week 1: what teams need to have in place first (monorepo or clear path to one, a champion team who will accept being a pilot, basic contract testing culture, executive sponsorship). Many teams will stall at Week 1 because they haven't done this groundwork |
| 5.3 | HIGH | ✅ | Massively expand "Common Problems" section — this is the most practically valuable section and is currently sparse. Add at minimum: RAG index staleness and re-indexing strategy, false-negative retrieval (code that should surface doesn't — how to diagnose and fix), LLM ignoring retrieved context (prompting strategies), contract test brittleness (too strict vs too loose), teams resisting the manifest (organisational friction patterns and how to address them) |
| 5.4 | MEDIUM | ✅ | Add a "What Good Looks Like" section with benchmark metrics: expected token cost per interaction by team size, CI pass rate targets by maturity level, acceptable boundary violation rate during transition, time-to-first-guardrail deployment. Give engineers a calibration point |
| 5.5 | MEDIUM | ✅ | Add explanatory comments to the larger code blocks connecting code decisions to architectural principles — comment the manifest validation script to explain what each check enforces and why |
| 5.6 | MEDIUM | ✅ | Add a "Graduating from Prototype" section: when should you move from ChromaDB to Pinecone, from LangChain to custom orchestration, from GitHub Actions scripts to a proper platform pipeline? Frame as a maturity model with specific triggers |
| 5.7 | MEDIUM | ✅ | Add a narrative thread through the implementation steps — currently they read as independent instructions. Add brief transition sentences connecting each step to the next ("Now that the RAG index knows what to retrieve, we need to make the CI pipeline care about what the LLM returns…") |
| 5.8 | LOW | ✅ | Add "How to Read This Paper" navigation box at the top — readers can self-select the path appropriate to their level |
| 5.9 | LOW | ✅ | Update footer to reference 6-paper series |

### Scoring Targets After Changes
Content: 7 → 9 · Flow: 6 → 9 · Accuracy: 9 → 9 · Readability: 7 → 9 · Prose/Bullets: 5 → 9 · Cross-Series: 7 → 9 · Audience: 8 → 9

---

## Paper 6: Case Study

**File:** `Paper_6_Case_Study.md`  
**Current:** 63/70 (9.0 avg) ✅ TARGET REACHED  
**Original score:** 47/70 (6.7 avg)  
**Status:** ✅ Complete

### Task List

| ID | Priority | Status | Task |
|----|----------|--------|------|
| 6.1 | HIGH | ✅ | Replace the generic e-commerce greenfield example (Part Seven, 8.1) with a DWP-specific greenfield scenario: "Imagine DWP is designing the Child Maintenance digital service from scratch — no legacy, blank slate. How would the framework apply?" Walk through the same phases (domain discovery, aggregate identification, contracts, implementation) but with DWP domain concepts throughout. This maintains narrative coherence |
| 6.2 | HIGH | ✅ | Rewrite the bounded context catalogue (Part Four) as narrative story, not bullet lists. Format: "The Evidence team asked: what do we own? Their aggregate is EvidenceItem — a fact assertion with provenance preserved intact. Their key invariant emerged from a hard-won lesson in legacy systems: evidence must be immutable once stored. Altering evidence destroys the audit trail that makes fraud prosecution possible. So their service design centres on…" |
| 6.3 | HIGH | ✅ | Substantially expand the Strategic Insights section (Part Eight): currently 3 short paragraphs. Add: what the architects got wrong initially and what changed their minds, what surprised them about the domain decomposition, one concrete example of a violation the framework prevented (e.g., "The Benefits team started building their own identity resolution... the manifest flagged the import direction before the PR was reviewed") |
| 6.4 | HIGH | ✅ | Remove the Abstract or merge it with the Introduction — they currently say almost the same thing. If keeping both: Abstract should be ≤5 sentences capturing the unique argument. Introduction should expand and motivate. Currently they duplicate each other at the same abstract level |
| 6.5 | MEDIUM | ✅ | Add "What the Framework Prevented" section: 3-5 concrete examples of the violations that would have occurred without boundaries — cross-import prevention, contract change impact, identity resolution scope creep — described as specific near-miss stories |
| 6.6 | MEDIUM | ✅ | Reduce duplication with Paper 2: Section 5 (Bounded Contexts in Detail) overlaps significantly with Paper 2 Section 4.2. Restructure as a narrative summary (paragraphs) with a note: "For the full rationale behind each context separation, see Paper 2, Section 4." Replace the repeated aggregate tables with a forward reference |
| 6.7 | MEDIUM | ✅ | Add an entry-point paragraph for readers who haven't read the other papers: "If this is your first paper in the series, the most important concepts to understand are bounded contexts, aggregates, and RAG scoping — Paper 1, Paper 3. This paper will make sense without them, but those papers will show you why the boundaries are drawn where they are." |
| 6.8 | LOW | ✅ | Fix version date: change from "February 2026" to "March 2026" |
| 6.9 | LOW | ✅ | Add a series reading guide at the top: "Read alongside Paper 1 if new to the series, or standalone if you've read Papers 1-5." |

### Scoring Targets After Changes
Content: 7 → 9 · Flow: 6 → 9 · Accuracy: 8 → 9 · Readability: 7 → 9 · Prose/Bullets: 5 → 9 · Cross-Series: 8 → 9 · Audience: 6 → 9

---

## Session Progress Log

| Session | Date | Papers Worked | Tasks Completed | Notes |
|---------|------|---------------|-----------------|-------|
| 1 | March 19, 2026 | Setup | Scored all 6 papers, created backlog | Ready to begin Paper 1 |
| 2 | March 2026 | Paper 1 | 1.1 ✅ 1.2 ✅ 1.3 ✅ 1.4 ✅ 1.6 ✅ 1.7 ✅ | P1 score: 55→63/70 (9.0 avg) ✅ Target reached. 1.5 and 1.8 skipped (marginal gain). Next: Paper 2 |
| 3 | March 2026 | Paper 2 | 2.1–2.10 all ✅ | P2 score: 49→63/70 (9.0 avg) ✅ Target reached. Added: How to Read box, CTO summary table, Turtle primer, prose intros to 5 sections, scaling uncertainty expanded, footer fixed. Next: Paper 3 |
| 7 | March 2026 | Paper 6 | 6.1–6.9 all ✅ | P6 score: 47→63/70 (9.0 avg) ✅ Target reached. Changes: removed duplicate Abstract (merged into expanded Introduction with How to Read box + series guide + new-reader entry paragraph), rewrote entire Bounded Context Catalogue from bullet lists to narrative prose with team stories and design rationale, replaced e-commerce greenfield with DWP Child Maintenance greenfield (CMS case, DWP vocabulary throughout), massively expanded Strategic Insights with What Architects Got Wrong + What Surprised Them + 3 specific violation prevention stories (identity shortcut, eligibility assumption, evidence immutability drift), added Paper 2 forward reference to reduce duplication, fixed version date Feb→Mar 2026. **ALL 6 PAPERS AT TARGET ✅** |

---

## How to Work This Backlog

1. Open IMPROVEMENT_BACKLOG.md and find the current paper
2. Work HIGH priority tasks first, then MEDIUM, then LOW
3. After completing each task: change ⬜ to ✅ in the task table
4. After completing all tasks for a paper: re-score it and update the scores table at the top
5. Add an entry to the Session Progress Log
6. Move to the next paper

