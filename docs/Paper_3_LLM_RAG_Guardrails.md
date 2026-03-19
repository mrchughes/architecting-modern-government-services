# LLM-Assisted Development in Microservice Architectures

**A Framework for Safe AI Assistance at Scale**

*Part 3 of 6 in the Architecting Modern Government Services Series*

*Version 1.0 | February 2026*

---

## Version History

| Version | Date | Changes |
|---------|------|--------|
| 1.0 | Feb 2026 | Initial release |

---

## Executive Summary

Large Language Models promise to accelerate software delivery and democratise technical knowledge. But in microservice architectures—where boundaries matter, contracts are sacred, and coupling is the enemy—uncontrolled LLM assistance creates chaos, not productivity.

This paper presents a framework that dramatically reduces token costs through intelligent context retrieval, prevents boundary violations by design through architectural enforcement, and scales to hundreds of services. The approach combines Service-Scoped RAG (Retrieval-Augmented Generation), a four-tier architectural model, and automated CI/CD guardrails to make safe LLM assistance possible at scale.

The choice is clear: adopt LLMs with architectural control, or watch short-term productivity gains erode into technical debt that costs millions to unwind.

---

## Prerequisites

This paper assumes familiarity with Domain-Driven Design concepts (bounded contexts, aggregates, ubiquitous language) and Clean Architecture principles (dependency inversion, layer separation). For a complete treatment of these foundations, see **Paper 1: Domain-Driven Design & Clean Architecture for Enterprise Systems**.

For readers applying this framework to identity and evidence coordination systems, **Paper 2: Evidence-Based Identity** provides the domain-specific context for that problem space.

---

## Part One: Why LLMs Need Guardrails

### The Promise and the Peril

LLMs are remarkable tools. They understand code, recognise patterns, suggest improvements, and accelerate routine tasks. A developer can describe a feature in natural language and receive working code in seconds. Refactoring that once took hours happens in minutes. Documentation writes itself. Tests materialise from specifications.

But this power comes with a fundamental problem: LLMs don't understand architectural boundaries. They see connections and patterns everywhere. They're trained to be helpful, which means "improving" things beyond what you asked for. And they have perfect recall—every file they've seen is equally accessible in their reasoning.

In a well-structured microservice architecture, this creates a dangerous failure mode.

### The Catastrophic Tinkering Cascade

Imagine your payments service processes hundreds of thousands of transactions daily with 99.5% success rates. Someone asks an LLM to optimise a query. But the LLM has visibility across the entire codebase. It notices the notification service has similar patterns, "standardises" them, refactors shared code, and touches six services instead of one. Each change might be correct in isolation, but together they introduce subtle incompatibilities. Success rates drop from 99.5% to 87% to 45%. The incident response team struggles because modifications are scattered across twenty files in six services.

**The critical insight: with LLMs, the risk isn't gradual architectural erosion. It's rapid, catastrophic destabilisation through well-intentioned but boundary-ignorant modifications.** Traditional development has protective friction—humans get tired, lose context, focus on one thing at a time. LLMs have no such friction. They can refactor twenty services as easily as one function, never losing focus or forgetting what they saw.

### The Three Dimensions of Failure

When organisations adopt LLMs without guardrails, failure manifests in three interconnected ways.

**First, boundary violations compound.** LLMs treat everything as potentially relevant. They don't know that the payments service shouldn't import from the notifications service implementation. They don't understand that domain logic belongs in the domain layer, not scattered throughout infrastructure code. Services designed to be independent become coupled. Business logic leaks across domain boundaries. The properties that make microservices valuable—loose coupling, independent deployability, clear ownership—evaporate one "helpful" modification at a time.

**Second, costs explode exponentially.** Token usage doesn't grow linearly with scale. Loading a service folder into context might consume 400,000 tokens. That's manageable for a single request. But conversations accumulate. First request: 400,000 tokens. Second request: 800,000 tokens. Third request: 1.2 million tokens. By the fourth or fifth turn, you're approaching context window limits and the conversation must reset. The developer loses the thread of reasoning, must re-explain the problem, and the LLM starts from scratch.

Scale this to an organisation with a hundred services making ten changes per service per month, and you're looking at costs between £500,000 and over £1 million annually—just in token usage. That doesn't count the hidden costs: architectural cleanup, integration fixes, review overhead from changes that touch too many things.

**Third, quality degrades paradoxically.** More context produces worse results. When an LLM processes 2 million tokens to answer a question requiring 25,000 tokens of relevant information, the signal-to-noise ratio plummets. Relevant business rules are buried in irrelevant files. The LLM gets distracted by patterns that look similar but aren't actually related. Accuracy drops from 85-95% with focused context down to 45-60% with full repository access. The LLM changes the wrong files, misses edge cases, and introduces bugs that wouldn't exist if it had just been given the right information in the first place.

### The Conversation Scope Problem

There's another dimension that's easy to miss: conversations accumulate irrelevant history that continues to consume tokens and dilute accuracy.

A developer fixes a refund bug, loading 25,000 tokens of context about refund logic. Great. But then, without starting a new conversation, they ask the LLM about implementing a fraud detection feature. The LLM now carries both the refund context and the fraud detection context—50,000 tokens. Then they ask about optimising database queries. Now it's 75,000 tokens. By day's end, the conversation has touched six different features, accumulated 150,000 tokens of context, and the LLM is reasoning over a mental model that includes irrelevant information from hours ago.

This creates compounding problems. Each new request pays the token cost for all previous context, even though 90% of it is now irrelevant. Accuracy degrades as the context becomes polluted—the LLM might try to apply refund patterns where they don't belong. Conversations hit context window limits after 5-6 turns instead of 20-30.

The solution is deliberate conversation scoping: one conversation per feature, per bug fix, per distinct task. When you fix the refund bug, that conversation ends. When you start working on fraud detection, you start a fresh conversation with fresh context. Think of it like pair programming with a human colleague—you don't expect them to keep all the refund details in their head while reasoning about fraud. You give them permission to focus fully on the new task. LLMs need the same permission, and conversation boundaries provide it.

### Why Traditional Approaches Fail

Organisations facing these problems typically try three approaches, and all three fail for the same fundamental reason: they try to control LLM behaviour through instructions and process rather than through architecture.

The first approach is to write better prompts. "Only change files in the payments service. Do not touch other services. Follow Clean Architecture principles." It's appealing because it's easy and requires no infrastructure investment. It's also ineffective. LLMs are helpful, not obedient. When they see an "obvious improvement" in a file they shouldn't touch, they touch it anyway. Instructions don't constrain what an LLM can see, and visibility equals capability.

The second approach is to rely on code review. Humans catch boundary violations after the code is written, reject the pull request, and ask for changes. This is better than nothing, but it operates at the wrong point in the workflow. The work has already been done. The tokens have already been spent. And as you scale to 50, 100, 200 services, code review becomes a bottleneck that nobody can keep up with.

The third approach is to limit LLM access to a single service folder. This is closer to the right answer, but it still suffers from the context explosion problem. A service folder might contain 150 files and 400,000 tokens. The LLM loads everything with equal weight—domain logic, infrastructure boilerplate, test utilities, configuration files—and the conversation still collapses after four or five turns.

What's actually needed is a fundamental shift in approach: enforce boundaries through architecture, not instructions. Make violations impossible, not just discouraged.

---

## Part Two: The Solution Architecture

### The Core Principle

The solution rests on a deceptively simple principle: **give the LLM exactly what it needs to know, nothing more, and make it physically impossible to violate architectural boundaries.**

This is accomplished through three complementary mechanisms working in concert. Service-Scoped RAG provides focused context by retrieving only the relevant code and documentation for a specific task. A multi-tier architectural model segregates concerns so that domain questions, contract questions, implementation questions, and infrastructure questions are handled separately with appropriate context. And CI/CD guardrails enforce boundaries automatically, blocking any change that violates architectural rules before a human even sees it.

The genius is in the combination. RAG alone would reduce token costs but wouldn't prevent violations. Architectural tiers alone would provide structure but wouldn't solve the context explosion. CI/CD guardrails alone would catch problems but only after the work is done. Together, they create a system where efficient assistance and architectural robustness are two sides of the same coin.

### Understanding RAG

Before diving into how Service-Scoped RAG works, let's clarify what RAG actually is and why it's the right approach for this problem.

**You don't train your own model.** This is critical to understand. This framework uses existing, commercially available LLMs—GPT-4, Claude, Gemini, or similar models that are already trained on vast amounts of code and natural language. These models already understand programming languages, software patterns, testing practices, and architectural concepts. They're general-purpose reasoning engines that don't need to be taught what a microservice is or how to write a unit test.

The challenge isn't that LLMs don't understand code. The challenge is that they don't understand *your* code, *your* domain, *your* architectural decisions, *your* business rules. And more specifically, they don't know what they should and shouldn't change when helping you.

There are three ways to give an LLM knowledge about your codebase.

**Training or fine-tuning** means taking a base model and continuing its training on your codebase. This sounds appealing but has critical flaws: it's expensive (hundreds of thousands of pounds for serious training runs), the knowledge freezes at training time, and the model learns your mistakes as thoroughly as your good practices. It also doesn't solve the boundary problem—the model still sees everything equally.

**Context window stuffing** means putting all your relevant code directly into each request. This works for small projects, but breaks catastrophically at scale. A 100-service microservice architecture might contain 200 million tokens. You can't fit that in any context window.

**Retrieval-Augmented Generation** is the middle path that solves both problems. You don't modify the model at all. Instead, you build an index of your codebase—a searchable knowledge base. When you ask a question, the system searches this index to find the 8-12 most relevant chunks of code and documentation. These chunks are inserted into the LLM's context along with your question. The LLM reasons over this focused context and provides an answer. When your code changes, you update the index, not the model.

RAG is the right answer for microservice architectures because it's cost-effective (no expensive training runs), stays current automatically (the index regenerates on every commit), provides focused context (25,000 high-signal tokens instead of 400,000 noisy ones), and enforces boundaries by construction (separate indexes per service mean the payments LLM assistance physically cannot see notifications). It works with any model and any language.

The practical implication: you need infrastructure engineers, not data scientists. RAG uses standard tools like ChromaDB, Pinecone, or Weaviate, semantic search, and LLM API integration.

### How Service-Scoped RAG Works

Traditional approaches to LLM assistance operate like handing someone a library card and saying "find what you need." Service-Scoped RAG operates like having a reference librarian who knows exactly which eight books contain the answer to your question.

When a developer asks to add partial refund support to the payments service, the system doesn't load 150 files containing 400,000 tokens. Instead, it performs a semantic search across a pre-indexed representation of the service, finding the 8-12 files that are actually relevant. These might include the refund test suite (which encodes the expected behaviour), the Payment aggregate root (which contains the current refund logic), the ProcessRefund use case (which orchestrates the operation), and an architectural decision record explaining why settlement works the way it does.

The context provided to the LLM is 25,000 tokens instead of 400,000 tokens. But more importantly, it's the *right* 25,000 tokens. Tests appear first, making expected behaviour clear. Domain logic is prioritised over infrastructure boilerplate. ADRs provide the "why" behind design decisions. The LLM isn't distracted by irrelevant patterns because irrelevant files simply aren't included.

The real magic happens over multiple conversation turns. Without RAG, each turn doubles the context size until the conversation becomes unusable. With RAG, each turn adds only the newly retrieved relevant context, and the conversation flows naturally like a pair programming session. The developer can explore, refine, test, and iterate without losing the thread of reasoning.

### The Four-Tier Architectural Model

The framework organises concerns into four distinct tiers, each with its own sphere of influence, its own RAG context, and its own verification mechanisms.

**The Domain Architecture tier** sits at the strategic level, dealing with bounded contexts, domain language, and business capabilities. This is where you answer questions like "Should refund logic live in the payments domain or the finance domain?" The RAG at this tier contains business capability maps, domain glossaries, context boundaries, and cross-context contracts. Verification comes through domain rule tests—executable specifications that encode business invariants. Changes at this tier are rare but impactful, typically requiring enterprise architect approval.

For a detailed treatment of bounded contexts, aggregates, and domain modelling, see **Paper 1, Part I: The Hierarchy of Boundaries**.

**The Microservice Architecture tier** governs service boundaries, contracts, and inter-service communication. This is where you answer questions like "What should the refund API contract look like?" and "Is this change backwards compatible?" The RAG contains OpenAPI specifications, event schemas, versioning policies, and lists of consumer services. Verification comes through contract testing—both provider tests (does my service fulfil its promises?) and consumer tests (do I satisfy what other services need?). Changes at this tier are more frequent but must maintain backwards compatibility.

**The Clean Architecture tier** controls the internal structure of individual services—the domain, application, and infrastructure layers that keep business logic pure and testable. This is where you answer questions like "How do I implement partial refunds?" The RAG contains the actual service code, prioritising tests and domain logic over infrastructure boilerplate. Verification comes through TDD and architectural fitness functions that ensure layers don't become coupled. Changes at this tier are daily and can be largely automated if tests pass.

For a complete explanation of Clean Architecture layers and dependency inversion, see **Paper 1, Part II: Clean Architecture Within Microservices**.

**The Infrastructure & Deployment tier** handles platforms, deployment pipelines, and operational concerns. This is where you answer questions like "How do I add Redis to my service deployment?" The RAG contains Terraform configurations, Kubernetes manifests, security baselines, and deployment patterns. Verification comes through infrastructure contract tests that ensure deployments actually work in staging before going to production. Changes at this tier follow a different cadence than application code and are owned by platform teams.

The tiers are nested and coordinated. Domain rules constrain service contracts. Service contracts guide internal implementation. Internal implementation is deployed through infrastructure. Each tier can only make changes that respect the constraints of the tier above it, and the CI/CD system enforces this automatically.

### Design Phase vs. Evolution Phase

Each tier moves through two distinct phases with fundamentally different characteristics, and understanding this distinction is critical to using LLMs effectively.

**The Design Phase** is when boundaries are being discovered and established. The RAG scope is broad because you're exploring possibilities. The LLM's role is to propose options, generate alternatives, and help humans think through trade-offs. Human involvement is high because these are decisions that shape the architecture for years to come. The goal is to establish boundaries and contracts that will guide all future work.

During design, you might ask an LLM to analyse a legacy system and suggest bounded context candidates, or to propose an event-driven architecture for cross-service communication. The LLM has access to a wide range of reference material—patterns, examples, anti-patterns, business requirements—and its job is to help architects make informed decisions. Once decisions are made, they're frozen in contracts, manifests, and test suites.

**The Evolution Phase** is when boundaries are established and work happens within them. The RAG scope is narrow because you're implementing within constraints. The LLM's role is to write code, refactor, and maintain implementations that respect existing boundaries. Human involvement is low for routine changes because the CI/CD system catches violations automatically. The goal is to maintain architectural integrity while moving quickly.

During evolution, you might ask an LLM to add a new feature to an existing service or refactor domain logic for clarity. The LLM has access to only the specific service's code and the contracts it must honour. It cannot see other services' implementations, cannot change shared contracts without permission, and cannot violate layer boundaries. The guardrails make incorrect changes impossible to merge.

The transition from design to evolution is explicit and deliberate. A service moves from design to evolution when its contracts are written, its boundaries are defined, its test suite is in place, and its RAG index is built. From that point forward, the architecture guides the LLM rather than the LLM proposing the architecture.

**Knowing When Design Phase Is Complete.** A common question is: how do you know when to stop designing and start building? The answer is that Design Phase completes when you have a specific set of frozen artefacts: a manifest entry for the service declaring its paths, dependencies, and ownership; written contracts (OpenAPI specs, event schemas) that define how other services will interact with it; at least one aggregate identified with its invariants documented; a glossary entry in the bounded context defining the key terms; and a passing domain rule test that encodes the core business invariant. These artefacts don't need to be perfect—they need to be *sufficient* to constrain Evolution Phase work. If you find yourself constantly revising boundaries during implementation, you're still in Design Phase and should acknowledge that rather than pretending you're evolving a stable design.

### Conversation Scoping: The Practical Rule

The conversation scope principle is simple to state: **one task, one conversation**. Start a new conversation for each distinct bug fix, feature, or refactoring concern. When you switch services, start fresh. When a conversation exceeds ten turns, start fresh and re-state context.

The cost of violating this is concrete, not theoretical.

**Poorly scoped — one long conversation:**

| Turn | Developer asks | Context added | Running total |
|------|-----------|---------------|---------------|
| 1 | Fix refund validation bug | Refund service code | 25,000 tokens |
| 2 | Add fraud detection webhook | + Fraud service code | 52,000 tokens |
| 3 | Optimise the payment DB queries | + Payment DB code | 81,000 tokens |
| 4 | Review the notifications logic | + Notification code | 118,000 tokens |
| 5 | Back to refunds — add proration | LLM accuracy degrades; refund context buried | 143,000 tokens |
| 6 | Why is the proration wrong? | Context window nearly full | ❌ Limit exceeded |

By turn 5, the LLM is reasoning over 143,000 tokens and the refund context — which should be dominant — is buried under three other services' code. Accuracy drops from ~90% to ~55%. The developer spends more time diagnosing bad suggestions than writing code.

**Well scoped — five focused conversations:**

| Conversation | Scope | Turns | Tokens | LLM accuracy |
|---|---|---|---|---|
| #1 | Refund validation bug | 3 | ~25K | 92% |
| #2 | Fraud detection webhook | 4 | ~30K | 91% |
| #3 | Payment DB optimisation | 3 | ~28K | 93% |
| #4 | Notifications review | 3 | ~22K | 90% |
| #5 | Refund proration | 5 | ~40K | 89% |

Same work, five clean conversations, ~145K tokens total — comparable cost, dramatically better accuracy. Each conversation carries only high-signal context for its specific task.

**Practical rule:** When you finish one task and move to another, close the conversation. This feels unnatural at first — it seems like you're "losing context." But the LLM doesn't benefit from knowing you spent the last hour fixing refund bugs when it now needs to reason clearly about fraud detection. Fresh context is a feature, not a cost.

### The Critical Insight: RAG Follows Architecture

**RAG follows architecture—architecture does not follow RAG.** RAG is an enforcement mechanism, not a design tool. You use Domain-Driven Design to discover what your services should be. You use business capability mapping to decide bounded contexts. RAG then enforces those decisions at the AI assistance level, making the LLM behave like a disciplined developer who respects boundaries.

This means the process of building a microservice estate with LLM assistance follows a very specific flow: design wide, contract sharp, build narrow, evolve safely. Each step narrows the LLM's world until it's operating within constraints that make autonomous assistance safe.

---

## Part Three: Implementation

### Building the RAG Index

Let's demystify what actually happens when you create a Service RAG index. There's no magic here—just deliberate curation and engineering.

First, you decide what belongs. This is a human decision encoded in a manifest. For a payments service, you might include everything under `services/payments/**` plus explicitly allowed shared contracts like `libs/payment-interfaces/**`. You exclude other services, old experiments, and global utilities unless explicitly permitted. This manifest becomes law—the single source of truth for what the LLM can see.

```json
{
  "services": {
    "payments": {
      "paths": ["services/payments/**", "libs/payment-interfaces/**"],
      "dependencies": ["libs/money-types", "libs/events-common"],
      "owner": "payments-team",
      "contracts": {
        "provides": ["refund-api-v2", "payment-events-v1"],
        "consumes": ["user-api-v3"],
        "consumers": ["notifications-service", "finance-service"]
      }
    }
  }
}
```

**When should something be a separately manifested service?** Use these six questions to decide:

1. **Does it deploy independently?** If you can release it without coordinating with another team, it deserves its own manifest entry.
2. **Does a different team own it?** Ownership boundaries map to manifest boundaries. If two teams will be pushing changes to "the same service," it should be two services.
3. **Does it have its own API contract?** If the thing you're building makes promises to external consumers via an API or event schema, it needs a manifest entry so those consumers can be tracked.
4. **Would a change in this code break something the payments-team couldn't predict?** If yes, the coupling should be made explicit in the manifest's `consumers` list rather than discovered in production.
5. **Does it need a separate RAG scope?** If it contains logic that LLMs should never mix with another service's logic (because they're different domains), they need separate indexes — which requires separate manifest entries.
6. **Does it have distinct infrastructure requirements?** Different scaling profiles, different data stores, or different compliance boundaries are signals for a separate manifest entry even if teams are shared.

Second, you chunk the code and documentation. LLMs don't read files; they read chunks. A chunk might be a single function, a class, a test, or a documentation section—typically 200 to 800 tokens each. Each chunk gets metadata: its file path, its type (test, domain logic, infrastructure, documentation), which service it belongs to, and what concepts it relates to. This metadata enables smart retrieval later.

Third, you create embeddings. Each chunk is converted into a vector—a list of numbers that represents its semantic meaning. Similar meanings produce similar vectors. Tests about refunds cluster together. Documentation about settlement rules clusters together. These vectors are stored in a service-specific index. Critically, you maintain one index per service, not one global repository index. This physical separation prevents cross-service leakage.

The RAG index is derived data, like build artefacts. You never edit it directly. It's just a fast lookup system that answers: "Given this question, which bits of this service matter?"

### Keeping RAG Current

The beauty of the approach is that RAG maintenance is boring and automatic—which is exactly what you want.

RAG updates happen automatically on three triggers: when code merges to main, when a release is cut, and optionally on a nightly schedule. When an update runs, the system detects which files changed, re-chunks only the affected code, re-embeds those chunks, and updates the index in place. No human involvement required.

The update process is incremental and efficient. If you change three files in a 150-file service, only those three files are re-processed. The git diff tells you exactly what changed. A simple script updates just those chunks.

What goes into RAG evolves by phase. During the Design Phase, you include service templates, architecture guidelines, similar services for reference, domain glossaries, and contract examples. During the Evolution Phase, you include actual service code prioritising domain and application layers, the complete test suite, service-specific ADRs, service contracts, and relevant shared library interfaces marked read-only.

What never goes into Service RAG: other services' implementation code, global utilities unless explicitly allowed, deprecated or experimental code, and infrastructure code (which has its own separate Infrastructure RAG).

> **RAG Index Failure Modes**
>
> - **Stale index:** Symptoms: LLM references functions you renamed last week, suggests patterns from deprecated code. Fix: verify the CI trigger is hitting the indexing step on merge; run `rebuild-index.sh payments` manually to confirm.
> - **Over-chunked (too small):** Symptoms: 200-token chunks split functions mid-definition; LLM sees signature but not body. Fix: increase chunk size to 500–800 tokens; use AST-aware chunking rather than character count.
> - **Under-chunked (too large):** Symptoms: Semantic search can't distinguish relevant from irrelevant within a 2,000-token chunk; retrieval quality drops. Fix: split large files more aggressively; one class or function per chunk.
> - **Metadata missing:** Symptoms: Tests aren't prioritised in re-ranking; LLM sees infrastructure boilerplate before business logic. Fix: ensure the chunker writes `file_type` and `layer` metadata for every chunk.
> - **Wrong index queried:** Symptoms: LLM references code from a different service not in scope. Fix: confirm the service routing step at the start of the workflow is identifying the correct service name; add an assertion that validates the target index name before querying.

### When RAG Retrieves Wrong or Insufficient Context

Even a well-configured RAG system will occasionally retrieve unhelpful content. Understanding why helps you diagnose and fix these situations.

**Symptoms of poor retrieval:** The LLM suggests changes that don't match your codebase style. The LLM references patterns or APIs that don't exist in your service. The LLM gives generic advice instead of specific guidance based on your code. The LLM proposes changes to files that aren't in its context.

**Common causes and fixes:**

*Stale index:* If recent changes aren't reflected, the index hasn't been updated. Run the indexing script manually or check the CI trigger that should update it on merge.

*Over-chunking:* If chunks are too small, the LLM gets fragments without context. Increase chunk size from 200 tokens to 500-800 tokens, ensuring functions and classes aren't split mid-definition.

*Under-chunking:* If chunks are too large, semantic search can't distinguish between relevant and irrelevant sections. Split large files more aggressively.

*Missing metadata:* If the re-ranker can't prioritise tests over infrastructure, the metadata probably isn't set. Ensure the chunker adds file type and layer information to each chunk.

*Ambiguous queries:* If your question is vague, retrieval will be vague. "How do I handle errors?" retrieves everything about errors. "How does RefundService handle InsufficientFundsException?" retrieves the specific handling.

**Diagnosing retrieval problems:** Most RAG systems can show you what was retrieved before it went to the LLM. If yours doesn't, add logging. Compare what the system retrieved against what you expected. If the right files aren't in the top 8-12 results, the problem is retrieval. If the right files are present but the LLM still gives bad answers, the problem is elsewhere—prompt construction, LLM temperature, or insufficient context even in the right files.

### Layered Enforcement Controls

The real power of this approach comes from layered segregation controls that make boundary violations not just discouraged but physically impossible.

At the foundation sits the monorepo manifest—a single `monorepo.json` file at your repository root that defines the truth about service boundaries. For each service, it declares allowed paths, permitted dependencies, ownership, and contracts. This manifest enables everything else: automated path validation, dependency checking, contract test discovery, ownership enforcement, and impact analysis.

**Layer 1: RAG Scoping.** When you request a change to the payments service, the system locks RAG scope to only those declared paths. The LLM physically cannot see other services because they're not in the index it queries. This is pre-emptive control—95% of problems are prevented here simply by controlling visibility.

**Layer 2: Path Validation.** Before any code is committed, an automated check validates that all changed files are within the allowed scope for the target service. A simple script loads the manifest, examines the git diff, and fails if any file is outside the permitted paths. This catches any attempt to modify unauthorised files before human review.

**Layer 3: Dependency Validation.** The same CI check analyses imports in changed files to ensure they only reference allowed dependencies. If you try to import from another service's implementation or use a prohibited shared library, the check fails automatically.

**Layer 4: Pre-flight Diff Guard.** Before the LLM can even apply a patch, a validator examines the proposed diff and rejects it if any file path fails the allowlist. This provides immediate feedback, not just at PR time.

**Layer 5: CODEOWNERS Integration.** Each service path has declared owners. If you use separate bot accounts per service, each bot is a member of only its service team and can only approve changes within its scope. Cross-service changes require human approval from both teams.

**Layer 6: Contract Testing.** Any change that touches a service contract must pass contract tests for both provider and consumers. These run automatically in CI, and all must pass before merge is allowed.

### The Full CI Enforcement Pipeline

The following is a working GitHub Actions workflow that implements Layers 2–6. Drop it in `.github/workflows/llm-boundary.yml` and it runs on every pull request targeting a service.

```yaml
name: LLM Boundary Enforcement

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  validate-scope:
    name: Validate Service Boundary
    runs-on: ubuntu-latest
    outputs:
      service: ${{ steps.service.outputs.service }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Identify target service
        id: service
        run: |
          # Convention: feature/SERVICE_NAME/description
          SERVICE=$(echo "${{ github.head_ref }}" | cut -d'/' -f2)
          echo "service=$SERVICE" >> $GITHUB_OUTPUT
          echo "Working on service: $SERVICE"

      - name: Validate modified file paths (Layer 2)
        run: |
          SERVICE="${{ steps.service.outputs.service }}"
          # Read allowed paths from manifest
          ALLOWED=$(jq -r ".services[\"$SERVICE\"].paths[]" monorepo.json)
          CHANGED=$(git diff --name-only origin/main...HEAD)
          VIOLATIONS=""
          for file in $CHANGED; do
            PERMITTED=false
            for pattern in $ALLOWED; do
              # Strip trailing ** glob for prefix matching
              PREFIX="${pattern%\*\*}"
              if echo "$file" | grep -q "^${PREFIX}"; then
                PERMITTED=true; break
              fi
            done
            if [ "$PERMITTED" = "false" ]; then
              VIOLATIONS="$VIOLATIONS\n  - $file"
            fi
          done
          if [ -n "$VIOLATIONS" ]; then
            echo "::error ::Files outside allowed scope for '$SERVICE':$VIOLATIONS"
            echo "Allowed paths: $ALLOWED"
            exit 1
          fi
          echo "All changed files within scope for $SERVICE ✅"

      - name: Validate imports (Layer 3)
        run: |
          SERVICE="${{ steps.service.outputs.service }}"
          ALLOWED_DEPS=$(jq -r ".services[\"$SERVICE\"].dependencies[]" monorepo.json)
          VIOLATIONS=""
          # Check changed Python files for forbidden internal imports
          git diff --name-only origin/main...HEAD | grep '\.py$' | while read -r pyfile; do
            grep -E "^from services\.|^import services\." "$pyfile" 2>/dev/null | while read -r imp; do
              PKG=$(echo "$imp" | awk '{print $2}' | cut -d'.' -f1-2)
              if ! echo "$ALLOWED_DEPS" | grep -qF "$PKG"; then
                echo "::error file=$pyfile::Forbidden internal import '$PKG' in $pyfile"
                exit 1
              fi
            done
          done

  contract-tests:
    name: Contract Tests
    runs-on: ubuntu-latest
    needs: validate-scope
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install pytest pytest-asyncio pact-python

      - name: Run provider contract tests
        run: |
          SERVICE="${{ needs.validate-scope.outputs.service }}"
          cd services/$SERVICE
          pytest tests/contract/provider/ -v --tb=short

      - name: Run consumer contract tests for all declared consumers
        run: |
          SERVICE="${{ needs.validate-scope.outputs.service }}"
          CONSUMERS=$(jq -r ".services[\"$SERVICE\"].contracts.consumers[]" monorepo.json)
          for consumer in $CONSUMERS; do
            echo "--- Consumer tests: $consumer verifying against $SERVICE ---"
            cd services/$consumer
            pytest tests/contract/consumer/ -v -k "$SERVICE" --tb=short
            cd -
          done

  domain-rule-tests:
    name: Domain Rule Tests
    runs-on: ubuntu-latest
    needs: validate-scope
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          SERVICE="${{ needs.validate-scope.outputs.service }}"
          pip install -r services/$SERVICE/requirements.txt

      - name: Run domain invariant tests
        run: |
          SERVICE="${{ needs.validate-scope.outputs.service }}"
          cd services/$SERVICE
          pytest tests/domain/ -v --tb=short
          echo "Domain rule tests passed ✅"
```

> **Adapting for your stack:** Replace `pytest` with your test runner of choice. The path and import validation steps are framework-agnostic and work identically for TypeScript, Java, or Go projects — adjust the `grep` patterns for your import syntax. The `monorepo.json` manifest is the single configuration source for all steps.

For the domain rule tests pattern (Given-When-Then BDD specs that this step runs), see **Paper 1, Part IV: Testing Strategy at Each Tier**.

Together, these layers create defence in depth. If RAG scoping somehow fails, path validation catches it. If path validation somehow fails, dependency checking catches it. If all automated checks somehow fail, CODEOWNERS requires human approval. Violations aren't just discouraged—they're systematically prevented.

### The LLM Workflow Step by Step

Now let's trace through exactly what happens when an LLM assists with a code change.

**Step 1: Service Identification.** Before any retrieval happens, the system must deterministically identify which service is being changed. This isn't guessed by the LLM—it's derived from explicit context like a service label on the request, a branch naming convention, or a routing step that maps features to services.

**Step 2: Semantic Search.** The LLM generates a query based on your request. For "add partial refund support," the query might be "partial refund settlement logic implementation." This query is run against only the payments service RAG index—other service indexes are never touched. The semantic search returns chunks ranked by relevance, typically 20-30 candidates.

**Step 3: Intelligent Re-ranking.** Raw semantic similarity isn't enough. The system re-ranks results using metadata: tests get highest priority (they encode expected behaviour as executable specs), then domain logic (business rules), then ADRs (explain "why"), then application layer (use cases), then infrastructure (only if needed). This prioritisation ensures the LLM sees the most authoritative information first.

**Step 4: Context Assembly.** The top 8-15 chunks are assembled into the LLM's context, totalling about 25,000 tokens instead of 400,000. Crucially, this context includes an explicit constraint: "You may not propose changes outside these paths." This is defence in depth—even though file access is already restricted.

**Step 5: Reasoning.** The LLM reasons over this focused context. It sees the refund test suite defining current behaviour, the Payment aggregate showing current implementation, the architectural decision record explaining regulatory requirements, and the value objects it needs to work with. It doesn't see 142 irrelevant files. It isn't distracted by other services.

**Step 6: Diff Proposal.** The LLM outputs a proposed change as a unified diff—just text, with no side effects yet.

**Step 7: Automated Validation.** Before any code is touched, the pre-flight diff guard validates that all files in the diff are within allowed paths. The path validator confirms scope. The dependency checker confirms no unauthorised imports. Only if all pass does the process continue.

**Step 8: Application and Testing.** The patch is applied by a deterministic tool (not by the LLM directly), service-specific tests are run for fast feedback, and a pull request is created by a bot account with the changes logged for audit. Humans review business logic, not architectural compliance—that's already been validated.

### Error Handling in the LLM Workflow

Things go wrong. Understanding the failure modes helps you build robust processes.

**LLM generates invalid code.** This is caught at Step 8 when tests fail. The system logs the failure, the generated code, and the context that was provided. The developer reviews the log, adjusts the prompt or provides additional context, and tries again. Don't manually fix generated code—this defeats the purpose of having reproducible, auditable generation.

**Path validation fails.** The LLM proposed changes to files outside the allowed scope. This is caught at Step 7. Review the diff to understand what the LLM was trying to do. Often this indicates a legitimate need that should be addressed differently—perhaps a shared library needs updating (by its owning team) or a contract needs amendment.

**Dependency validation fails.** The LLM imported something prohibited. Usually this means the LLM found a pattern in its training data that isn't appropriate for your architecture. Add explicit instructions in the context: "Do not import from package X. Use the Y interface instead."

**Semantic search returns nothing relevant.** The index may be empty, corrupted, or the query may be too abstract. Try rephrasing with specific terms that would appear in your code. If this happens consistently, rebuild the index from scratch.

**Context window exceeded.** The retrieved context plus conversation history exceeds the model's limit. Start a fresh conversation. If this happens frequently, your chunks may be too large, or you're having too many turns before starting fresh.

**Rate limiting or API errors.** Transient failures from the LLM provider. The system should retry with exponential backoff. If persistent, check your API quota or the provider's status page.

### Data Governance and LLM Assistance

The framework so far has focused on code boundaries, but data is equally critical—especially in government and regulated industries where PII, GDPR compliance, data retention, and audit trails are non-negotiable.

**Each Microservice Owns Its Data.** The foundational principle: each microservice has its own database that no other service accesses directly. This is database-per-service, not a shared database. Services communicate through APIs and events, never through direct database access. This ensures that data schemas can evolve independently without breaking consumers.

When an LLM assists with data-related changes, it operates within this boundary. It can modify the refund-management service's database schema, but it cannot touch the payment-processing service's database—even though they're in the same bounded context.

**RAG and Data: What LLMs See.** This is critical: **production data never goes into RAG indexes or LLM prompts**. Ever. The risks are too high—data leakage, PII exposure, compliance violations.

Instead:
- RAG indexes contain database *schemas* and migration scripts (the structure, not the data)
- Code examples use synthetic data or anonymised samples
- Test data is generated, not copied from production
- LLM assistance for data queries uses schema definitions, not actual records

When a developer asks "How do I query refunds for a specific payment?", the LLM sees the repository interface, the database schema, and example queries—but no actual payment or refund data.

**Compliance and Audit.** The framework logs all LLM-assisted changes that touch data: what question was asked, what files were changed, what schema migrations were generated, who approved the changes, and what tests were run. This audit trail is essential for compliance reviews. You can demonstrate that no LLM ever saw production PII, that all changes went through proper approval, and that all data access followed the principle of least privilege.

---

## Part Four: Practical Application

### Adding a Feature to an Existing Service

Let's trace through a real scenario: adding subscription-based refunds to the refund-management microservice—a moderately complex feature that touches domain logic, contracts, and potentially infrastructure.

**The journey begins with a domain question.** Before writing any code, you ask: "Does subscription refund logic belong in the payments bounded context?" The system queries the Domain Architecture tier RAG, retrieving the payments context definition, domain invariants, and the domain glossary. The LLM analyses this and confirms: yes, subscription refunds fit within the existing payments context. No new cross-domain dependencies are needed. A domain rule test is added to encode the new invariant. This takes 30 seconds and costs about £0.002.

**Next comes contract impact analysis.** You ask: "Will this change the refund API contract?" The system queries the Microservice Architecture tier RAG, retrieving the current OpenAPI specification, the list of consuming microservices, and the versioning policy. The LLM proposes adding an optional "refundType" field. It checks compatibility: the field is optional, existing consumers can ignore it, no breaking changes. Contract tests are automatically updated for all consumer microservices. This takes 45 seconds and costs about £0.0015.

**Now you implement the feature.** You ask: "Implement subscription refund logic with proration." Here's where Service-Scoped RAG shines. The system performs a semantic search for "subscription refund proration" across the refund-management microservice index. It retrieves exactly eight files totalling 25,000 tokens: the refund test suite, the Refund aggregate root, the RefundRequest entity, the ProcessRefund use case, an ADR about subscription payment rules, the Money value object, the repository interface, and the RefundCalculator domain service.

Critically, it does *not* retrieve the 142 other files in the microservice—controllers, middleware, database implementations, configuration files, deployment scripts. Those aren't relevant to this task. And it absolutely does not retrieve anything from the payment-processing or settlement microservices, even though they're in the same bounded context.

**You have a 12-turn conversation with the LLM.** It proposes domain model changes, adds proration calculation logic, updates the use case, adds tests for edge cases, refines the algorithm based on your feedback, adds validation, implements error handling, adds integration tests, updates API documentation, and helps you review everything. The conversation flows naturally because the context stays focused. Total cost: about £0.035. Total time: 25 minutes. All changes stay within the domain and application layers of the refund-management microservice. Zero changes to other microservices, shared libraries, or infrastructure.

**Infrastructure comes last.** You ask: "Do I need infrastructure changes?" The system queries the Infrastructure tier RAG, retrieving your current Helm chart, ConfigMap template, and feature flag patterns. The LLM suggests adding a feature flag and new monitoring metrics. An infrastructure contract test ensures the config is present. This takes 2 minutes and costs about £0.002.

**Finally, CI/CD validates everything.** You create a pull request and the automation kicks in. Path validation confirms all changes are within the allowed scope. Domain rule tests pass. Provider contract tests confirm your API is backwards compatible. Consumer contract tests run in the notification and finance microservices—all pass. Unit and integration tests pass. Architectural fitness functions detect zero layer violations. Total CI time: 75 seconds. All checks pass automatically.

A human tech lead reviews the pull request, but they're only reviewing business logic—the proration algorithm and edge case handling. They spend 10 minutes because they don't need to check architectural compliance, contract compatibility, or test coverage. That's already been validated.

From initial question to production-ready code: 40 minutes of developer time, 10 minutes of review time, about £0.041 in token costs, zero architectural violations, zero contract breaks.

### Integrating with a Shared Platform Service

Here's a different scenario that shows how the framework handles platform services: a microservice within one subdomain needs to integrate with a platform service owned by another team.

You're building a benefits-claim-service that needs to authenticate citizens. The Identity platform service is shared across multiple subdomains. It's owned by the Platform Identity Team, not your team.

**Discovering the Platform Contract.** You ask: "How do I authenticate a citizen in my service?" The LLM queries the domain RAG, which includes *contracts* for all platform dependencies (read-only). It retrieves the identity-service OpenAPI contract, authentication flow documentation, rate limiting and SLA information, and example integration code from other services.

Critically, the LLM does *not* see the identity-service's implementation code, database schema, or internal architecture. That's owned by a different team and protected by different RAG boundaries. You see what the Identity platform service does, not how it does it.

**Implementing the Anti-Corruption Layer.** The identity-service uses its own domain language—"Subject," "Credential," "AuthenticationContext." Your service uses different language—"Claimant," "Verification," "ClaimContext." You don't want Identity concepts leaking into your domain model.

You ask: "Create an anti-corruption layer for identity authentication." The LLM generates an adapter in your infrastructure layer that calls the Identity API, a service interface in your domain layer, translation logic that maps Identity concepts to your domain concepts, and tests that verify the translation works correctly.

Your domain code depends on your own interface (which you own). The infrastructure provides an adapter (which implements your interface by calling the identity-service). The dependency arrow points inward—your domain doesn't know the Identity platform service exists.

For more on anti-corruption layers and bounded context relationships, see **Paper 1, Section 1.4: Context Relationships and Communication**.

**Contract Testing Across Domain Boundaries.** You write consumer contract tests that verify your assumptions about the identity-service: "When I call authenticate with valid credentials, I get a token." "When I call validate-token, I get subject information." "When credentials are invalid, I get a 401 error."

These tests run against the identity-service's contract. If the Platform Identity Team makes a breaking change, your tests fail in CI before you deploy. They're forced to either maintain compatibility or coordinate a migration with all consuming teams.

**Governance and Communication.** Because this is a cross-domain dependency, the framework enforces additional governance. Your microservice is automatically registered as a consumer of the identity-service. When the Platform Identity Team proposes contract changes, your team is notified. Contract tests for all consumers must pass before identity-service changes deploy. Breaking changes require explicit migration plans and coordination windows.

You don't need to remember to check with the Identity team—the framework enforces the coordination automatically.

---

## Part Five: The Economics

### The Cost Comparison

The calculation starts with a formula that you can adapt to your own organisation:

```
annual_token_cost = services × changes_per_service_per_month × 12
                  × avg_turns_per_change × tokens_per_turn × price_per_token

hidden_cleanup_cost = annual_changes × violation_rate × avg_remediation_cost
```

Consider an organisation with 100 microservices, each undergoing about 10 changes per month. That's 1,200 changes per month, 14,400 per year. Each change involves an average of four LLM interactions as developers iterate toward a solution—57,600 interactions annually.

*Note: These figures are based on services averaging 150 files and 400K tokens each, LLM pricing of approximately $0.03 per 1K tokens for input, conversation lengths of 4-5 turns, and architectural violations occurring in approximately 15% of uncontrolled changes. Your actual results will vary based on service size, change frequency, team discipline, and organisational factors. The framework's value increases with scale.*

**Worked arithmetic:**

| Variable | Without framework | With framework |
|----------|-----------------|----------------|
| Services | 100 | 100 |
| Changes/service/month | 10 | 10 |
| Annual interactions (×4 turns avg) | 57,600 | 57,600 |
| Tokens per interaction | 400,000 (full service) | 25,000 (RAG-scoped) |
| Token price ($/1K) | $0.03 | $0.03 |
| **Annual token cost** | **£2,765** | **£173** |
| Violation rate | 15% of changes | ~0.5% (CI blocks most) |
| Avg remediation cost per violation | £1,000 | £1,000 |
| Annual violations | 1,728 | 58 |
| **Annual architectural cleanup** | **£150,000** | **£5,000** |
| Integration fixes (contract breaks) | £100,000 | £2,000 |
| Review overhead (manual checks) | £200,000 | £30,000 |
| **Total annual cost** | **£453,000** | **£37,000** |

Substitute your own numbers: realistic estimates for a 20-service team (2 changes/service/month, 2 turns/change): without framework ~£17,000/yr, with framework ~£3,000/yr. The ratio holds.

**Without this framework**, the costs are brutal. Loading full service context for each interaction costs about £0.048 in tokens. That's £2,765 annually in direct token costs alone. But that's the small number. The hidden costs dwarf it: around £150,000 per year in architectural cleanup when changes violate boundaries, £100,000 in integration fixes when contracts break, and £200,000 in review overhead because humans must catch everything automation should have caught. Total annual cost: approximately £453,000.

**With this framework**, the economics flip dramatically. Service RAG reduces token cost per interaction to about £0.008. The 57,600 interactions cost £461 per year—a 95% reduction. More importantly, the hidden costs evaporate. Architectural cleanup drops to about £5,000 because violations are prevented by design. Integration fixes drop to £2,000 because contract tests catch breaks automatically. Review overhead drops to £30,000 because humans only review business logic. Total annual cost: approximately £37,000.

The savings are £416,000 per year, every year, for as long as you operate these systems.

The setup cost is real but manageable: perhaps £70,000 for RAG infrastructure, manifest creation, CI pipeline setup, contract test framework, and team training. With £416,000 in annual savings, you break even in about two months. Over three years, you're looking at over £1.1 million in savings.

But here's what the numbers don't capture: the organisation with uncontrolled LLM usage is accumulating architectural debt that will eventually force a rewrite or major remediation. The organisation using this framework maintains architectural integrity while moving faster. That's not just a cost difference—it's a strategic advantage.

---

## Part Six: Implementation Roadmap

### Phase One: Foundation (Months 1-4)

The first phase establishes the machinery that makes everything else possible.

**Month 1: Architecture Documentation.** Before building anything, document what you have. Create a manifest that describes your current services, their boundaries, their dependencies, and their ownership. This forces architectural decisions to be explicit. Even if your current state is messy, documenting it honestly is the first step toward improvement.

**Month 2: RAG Infrastructure.** Set up the vector database, embedding pipeline, and semantic search infrastructure. Start with one pilot service—something important enough to matter but contained enough to be manageable. Build the index for that service, wire up the LLM integration, and prove the concept works.

**Month 3: CI/CD Integration.** Implement the enforcement layers: path validation, dependency checking, and the pre-flight diff guard. These run in CI for the pilot service. Every PR that touches the pilot service goes through automated boundary validation.

**Month 4: Contract Testing Foundation.** Establish the contract testing framework. Write provider and consumer tests for the pilot service's APIs. Integrate these into CI so contract violations block merge.

By the end of Phase One, you have one fully-protected service with Service-Scoped RAG, layered enforcement, and contract testing. This proves the approach works in your environment.

### Lightweight Adoption for Smaller Teams

Not every organisation has 100 services and platform engineering capacity. If you're a single developer or a small team, here's how to get 80% of the value with 20% of the infrastructure.

**Skip the vector database initially.** Use file-based context selection instead. Create a simple script that, given a service name and a task description, copies the relevant files into a single context file. Manual curation beats no curation.

**Use a manifest even with one service.** Write the `monorepo.json` even if you only have one service. It forces you to think about boundaries, and it's ready when you add more services.

**Implement path validation first.** A 20-line script that checks `git diff` against allowed paths catches most boundary violations. Run it as a pre-commit hook.

**Use conversation discipline instead of RAG.** Without RAG, enforce scope through conversation hygiene: start fresh conversations for each task, explicitly paste only relevant files into context, tell the LLM what it may and may not modify.

**Graduate to full infrastructure when you feel pain.** When manual context selection becomes tedious, build the RAG index. When you have multiple services, add dependency validation. When you have multiple teams, add CODEOWNERS. Let complexity grow with actual need.

### Phase Two: Expansion (Months 5-10)

With the foundation proven, expand to cover more of your architecture.

**Months 5-7: Core Services.** Roll out the framework to your most critical services—the ones where architectural violations would cause the most damage. Typically this is 10-20 services covering payment processing, identity, core business logic.

**Months 8-10: Broad Coverage.** Extend to remaining services. By now, the process is well-understood and teams can self-serve. New services are created with the framework from day one. Legacy services are migrated incrementally.

### Phase Three: Optimisation (Months 11-18)

With broad coverage established, focus on making the framework more effective.

**Cross-Service Intelligence.** Enhance RAG to understand cross-service patterns. When a developer asks about a feature that spans multiple services, the system can identify which services are involved and coordinate the work appropriately.

**Predictive Analysis.** Use historical data to predict which changes are likely to cause problems. Flag high-risk changes for additional review. Identify patterns that consistently lead to architectural drift.

**Self-Improving Guardrails.** The system learns from violations that slip through. When a boundary violation is caught in code review (meaning automated checks missed it), the check is enhanced to catch similar cases in the future.

---

## Conclusion

LLMs are not going away. They will become more capable, more accessible, and more embedded in development workflows. The question isn't whether to use them—it's how to use them without destroying the architectural integrity that makes systems maintainable.

This framework provides an answer: enforce boundaries through architecture, not instructions. Give LLMs focused context through Service-Scoped RAG. Validate changes automatically through layered CI/CD controls. Make violations impossible, not just discouraged.

The result is the best of both worlds: the productivity gains that LLMs promise, delivered without the architectural chaos that uncontrolled adoption creates. Developers move faster. Architects sleep better. Systems remain comprehensible.

The cost of implementing this framework is measured in months and thousands of pounds. The cost of not implementing it is measured in years and millions of pounds—plus the eventual forced rewrite when architectural debt becomes insurmountable.

The choice is clear.

---

## Related Papers

- **Paper 1: Domain-Driven Design & Clean Architecture for Enterprise Systems** — Comprehensive treatment of bounded contexts, aggregates, Clean Architecture layers, and the hierarchy of boundaries that this framework enforces.

- **Paper 2: Evidence-Based Identity** — Application of DDD principles to identity coordination, demonstrating how the bounded context patterns work in a specific, complex domain.

- **Paper 4: Automated LLM-Driven Development** — End-to-end automated pipeline that builds directly on this framework's guardrails to generate complete microservices from business requirements, closing the loop from manifest to running service.

- **Paper 5: Getting Started** — Practical implementation guide with technology recommendations, pseudocode examples, and troubleshooting for teams beginning adoption of this framework.

- **Paper 6: DWP Case Study** — Application of these guardrails and the underlying DDD architecture to a real government services programme, including the team topology and governance model that makes it work at scale.

---

*Paper 3 of 6 in the Architecting Modern Government Services series*

**Prerequisites:** Paper 1 — Domain-Driven Design & Clean Architecture · Paper 2 — Evidence-Based Identity  
**Next:** Paper 4 — Automated LLM-Driven Development  
**See also:** Paper 5 — Getting Started · Paper 6 — DWP Case Study
