# Architecting Modern Government Services

**A six-paper series on Domain-Driven Design, identity verification, and LLM-assisted development in complex public sector systems.**

This series was written with the UK Department for Work and Pensions (DWP) as its primary case study — one of the largest and most complex digital transformation programmes in UK government. The principles and patterns are applicable to any large-scale, federated, regulation-heavy organisation.

---

## Who This Series Is For

- **Solution architects** designing bounded context maps and microservice boundaries
- **Engineering leads** adopting LLM-assisted development and needing guardrails
- **Technical product managers** who need to understand why domain boundaries matter before committing to a service decomposition
- **New team members** joining a programme already applying these patterns

---

## The Series

| # | Title | What it covers |
|---|-------|---------------|
| 1 | [DDD & Clean Architecture](Paper_1_DDD_Clean_Architecture.md) | The full hierarchy from domain to deployment: bounded contexts, aggregates, microservices, clean architecture layers |
| 2 | [Evidence-Based Identity](Paper_2_Evidence_Based_Identity.md) | Modelling citizen identity as probabilistic evidence — claimed vs verified, the trust chain, fraud detection as structural property |
| 3 | [LLM RAG & Guardrails](Paper_3_LLM_RAG_Guardrails.md) | Scoping LLM context per service, enforcing domain boundaries via RAG + manifest contracts, CI pipeline enforcement |
| 4 | [Automated LLM Development](Paper_4_Automated_LLM_Development.md) | The full automation pipeline from business requirements to running service: discovery → contracts → scaffold → tests → deployment |
| 5 | [Getting Started](Paper_5_Getting_Started.md) | Practical implementation guide — tech stack, RAG setup, prototype path, common problems, what good looks like |
| 6 | [Case Study](Paper_6_Case_Study.md) | DWP applied: full bounded context decomposition, inter-context communication, Child Maintenance greenfield, strategic retrospective |

---

## Recommended Reading Order

**Reading the series for the first time:**  
Start at Paper 1 and read sequentially. Each paper builds on the last.

**Already working in a microservice architecture, evaluating LLM adoption:**  
Start at Paper 3, then read Paper 4, then Paper 5.

**Debugging a specific problem (wrong boundaries, LLM context pollution, team friction):**  
Go directly to Paper 5, Part Six (Common Problems) and trace back to whichever foundation paper it references.

**New to the DWP programme specifically:**  
Read Paper 2 (identity model) and Paper 6 (full case study) — these give the most complete picture of the applied architecture.

---

## Series Architecture

The papers form a deliberate stack:

```
Papers 1–2: Foundations
  What bounded contexts are, how to find them, how identity works

Paper 3: Rules
  How to give an LLM a correctly scoped view of your system

Paper 4: Automation
  How to turn those rules into an end-to-end development pipeline

Paper 5: Practice
  How to actually get it running in your team

Paper 6: Applied
  What it looks like when all of it is working at DWP scale
```

---

*March 2026*
