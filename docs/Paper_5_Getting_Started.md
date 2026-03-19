# Getting Started: A Practical Guide

**Building Your First LLM-Assisted Microservice System**

*Part 5 of 6 of the Architecting Modern Government Services Series*

*Version 1.0 | February 2026*

---

## Version History

| Version | Date | Changes |
|---------|------|--------|
| 1.0 | Feb 2026 | Initial release |

---

## Who This Paper Is For

You've read Papers 1-4. You understand the concepts. Now you want to build something. This paper bridges the gap between understanding and doing.

We'll cover:
- The technology stack you need
- How to set up RAG infrastructure
- How to build the enforcement pipeline
- How to wire up LLM integration
- A lightweight approach for prototyping
- Common problems and how to solve them

This is the practical companion to the conceptual papers. Less "why," more "how."

> **How to Read This Paper**
> **Prototype in a day?** Jump to Part Five (A Lightweight Approach), then return to Parts Two–Four for depth.
> **Setting up from scratch?** Follow the parts in sequence — each one builds on the previous.
> **Debugging a problem?** Jump directly to Part Six (Common Problems).
> **Already running and want to scale?** Part Seven covers production readiness and tool graduation.
> **New to the series?** Papers 1, 3, and 4 establish the *why* behind everything in this paper.

---

## Week 0: Before You Start

The most common reason teams stall at Week 1 is that Week 0 prerequisites weren't in place. These aren't optional preparation steps — they're hard dependencies. Skip them, and the framework will fight your organisation's existing habits rather than enhance them.

**A monorepo, or a committed plan to move to one.** The manifest pattern, path validation, and service scoping all assume code lives in one repository. If you're currently multi-repo, you need a migration plan. You don't need to complete it before piloting — but you need agreement on the destination. Without it, enforcement becomes politically contentious: why should my repo follow your rules?

**A champion team.** Pick one team willing to be your pilot. They need to understand the deal: slightly slower initial setup, a week or two learning the new workflow, and willingness to report problems rather than work around them. The pilot team's experience becomes the evidence base for persuading everyone else.

**Technical sponsorship.** Someone with authority to say "this is how we write services now" needs to be publicly behind this. The enforcement pipeline will catch a change that a developer considers valid and block it. Without sponsorship, that block gets overridden and the framework loses credibility on day one.

**An understood pilot service.** The first service you index should be one your team knows deeply — ideally the one you work on daily. Piloting on an unfamiliar codebase makes it impossible to tell whether poor LLM suggestions are a framework problem or a knowledge gap.

**Basic git discipline.** The enforcement pipeline runs on pull requests. If your team currently commits directly to main, establish PR-based workflow first — not for governance reasons, but because every validation script assumes this flow.

None of these take weeks to establish. Most teams can address them in a day of honest conversations. But they are blockers, and skipping them produces a pilot that "didn't work" for reasons unrelated to the framework itself.

---

## Part One: The Technology Stack

### What You Actually Need

The framework described in Papers 3-4 sounds like it requires significant infrastructure. For a production system at scale, it does. But for prototyping and learning, you can start much smaller. Here's the minimum viable stack:

| Component | Production Choice | Prototype Choice | Purpose |
|-----------|------------------|------------------|---------|
| Vector Database | Pinecone, Weaviate | ChromaDB (local) | Store and search code embeddings |
| Embedding Model | OpenAI ada-002, Cohere | OpenAI ada-002 | Convert code to vectors |
| LLM | GPT-4, Claude | GPT-4 or Claude API | Generate and reason about code |
| Orchestration | Custom framework | LangChain or direct API | Wire RAG to LLM |
| CI/CD | GitHub Actions, GitLab CI | GitHub Actions | Enforce boundaries |
| Contract Testing | Pact, Spring Cloud Contract | Pact | Verify service contracts |
| Repository | Monorepo (Nx, Turborepo) | Single Git repo | Store all code and artefacts |

**For a prototype, you need:**
1. An OpenAI API key (or Anthropic for Claude)
2. Python 3.10+ installed
3. A Git repository
4. About £20-50 in API credits to experiment

That's it. Everything else can be local or free-tier.

### Why These Choices?

**ChromaDB for prototyping** because it runs locally, requires no account setup, and has a simple API. You can literally `pip install chromadb` and start indexing in five minutes. For production, you'd move to a managed service like Pinecone (scalable, reliable) or Weaviate (open source, self-hostable).

**OpenAI ada-002 for embeddings** because it's the most widely used, well-documented, and cost-effective embedding model. At $0.0001 per 1K tokens, indexing a 100-file service costs about $0.10. Alternatives like Cohere or open-source models (sentence-transformers) work too, but ada-002 has the best documentation and tooling.

**GPT-4 or Claude for generation** because they're the most capable models for code generation and reasoning. GPT-4 is slightly better at following complex instructions; Claude is better at longer contexts and more nuanced reasoning. Either works. For prototyping, use whichever you have access to.

**LangChain for orchestration** because it handles the RAG workflow out of the box: load documents, chunk them, embed them, store in vector DB, retrieve on query, inject into prompt, call LLM. You could build this yourself, but LangChain saves weeks of plumbing. For production, you might replace it with a custom solution optimised for your needs.

**GitHub Actions for CI/CD** because it's free for public repos, cheap for private, and most developers already know it. The enforcement scripts we'll build are just Python—they'd work in any CI system.

**Pact for contract testing** because it's the industry standard, supports multiple languages, and has excellent documentation. It's slightly complex to set up, but worth it.

---

## Part Two: Setting Up RAG Infrastructure

### Step 1: Install Dependencies

Create a new Python project with these dependencies:

```
# requirements.txt
chromadb>=0.4.0
openai>=1.0.0
langchain>=0.1.0
langchain-openai>=0.0.5
tiktoken>=0.5.0
```

Set your API key as an environment variable:

```
export OPENAI_API_KEY=sk-your-key-here
```

### Step 2: Understand the Indexing Process

Before writing code, understand what we're building. The RAG index is the service-scoped context architecture described in Paper 3, Part Two. Filtering the index to a single service is what prevents context contamination between bounded contexts: when the LLM only retrieves code from the `eligibility-decision` service, it cannot accidentally suggest patterns from the `claims-intake` service. Service isolation at the retrieval layer enforces boundary isolation at the reasoning layer.

The index is created through this process:

```
Source Code Files
       ↓
   [Chunking]
   Split files into semantic units (functions, classes, ~500 tokens each)
       ↓
   [Metadata Extraction]
   Tag each chunk: file path, type (domain/application/infrastructure/test), service name
       ↓
   [Embedding]
   Convert each chunk to a vector using ada-002
       ↓
   [Storage]
   Store vectors + metadata in ChromaDB
       ↓
   Service RAG Index
```

When a query comes in:

```
Developer Question
       ↓
   [Query Embedding]
   Convert question to vector using same embedding model
       ↓
   [Similarity Search]
   Find 10-20 most similar chunks in the index
       ↓
   [Re-ranking]
   Prioritise: tests first, then domain, then application, then infrastructure
       ↓
   [Context Assembly]
   Combine top 8-12 chunks into prompt context
       ↓
   LLM Prompt
```

### Step 3: Build the Indexer

Here's the structure of a service indexer. This isn't production code—it's a clear illustration of the process:

```
ServiceIndexer
├── __init__(service_name, service_paths, vector_store)
│
├── index_service()
│   ├── load_files(paths)           # Read all code files
│   ├── chunk_files(files)          # Split into semantic units
│   ├── extract_metadata(chunks)    # Tag with file type, service, etc.
│   ├── embed_chunks(chunks)        # Convert to vectors
│   └── store_chunks(embeddings)    # Save to vector DB
│
├── chunk_file(file_content, file_path)
│   ├── Detect language (Python, Java, etc.)
│   ├── Parse into AST if possible
│   ├── Extract functions, classes, methods as chunks
│   ├── Fall back to sliding window for non-parseable files
│   └── Return list of chunks with boundaries
│
└── classify_chunk(chunk, file_path)
    ├── If path contains "test" → type = "test"
    ├── If path contains "domain" → type = "domain"
    ├── If path contains "application" → type = "application"
    ├── If path contains "infrastructure" → type = "infrastructure"
    └── Else → type = "other"
```

**Key decisions in chunking:**

- **Chunk size:** Aim for 300-800 tokens per chunk. Too small and you lose context; too large and you waste tokens on irrelevant content. Functions and classes are natural boundaries.

- **Overlap:** Include 50-100 tokens of overlap between chunks so context isn't lost at boundaries.

- **Metadata:** Always include file path, chunk type, service name, and line numbers. This metadata enables filtering and re-ranking.

**Pseudocode for the core indexing loop:**

```
function index_service(service_name, paths):
    chunks = []
    
    for each file in find_files(paths):
        content = read_file(file)
        file_chunks = chunk_file(content, file.path)
        
        for chunk in file_chunks:
            chunk.metadata = {
                "service": service_name,
                "file_path": file.path,
                "type": classify_chunk(chunk, file.path),
                "start_line": chunk.start_line,
                "end_line": chunk.end_line
            }
            chunks.append(chunk)
    
    embeddings = embedding_model.embed(chunks)
    
    vector_store.add(
        documents=[c.content for c in chunks],
        metadatas=[c.metadata for c in chunks],
        embeddings=embeddings,
        ids=[generate_id(c) for c in chunks]
    )
    
    return len(chunks)
```

### Step 4: Build the Retriever

The retriever finds relevant chunks for a query. The key insight is that raw similarity isn't enough—you need to re-rank by chunk type to prioritise tests and domain logic.

```
function retrieve_context(query, service_name, top_k=12):
    # Step 1: Embed the query
    query_embedding = embedding_model.embed(query)
    
    # Step 2: Search with service filter
    results = vector_store.query(
        query_embedding=query_embedding,
        filter={"service": service_name},
        top_k=top_k * 2  # Get more than we need for re-ranking
    )
    
    # Step 3: Re-rank by type priority
    priority = {"test": 0, "domain": 1, "application": 2, "infrastructure": 3, "other": 4}
    
    results.sort(key=lambda r: (
        priority.get(r.metadata["type"], 5),  # Primary: type priority
        -r.similarity_score                    # Secondary: similarity (descending)
    ))
    
    # Step 4: Take top_k after re-ranking
    return results[:top_k]
```

**Why tests first?** Tests encode expected behaviour. When an LLM sees tests before implementation, it understands what the code *should do*, not just what it currently does. This dramatically improves generation accuracy. The prioritisation order mirrors Paper 3's principle that test artefacts carry higher signal density than implementation code.

### Step 5: Keep the Index Current

The index must update when code changes. The simplest approach:

```
function update_index_for_changes(service_name, changed_files):
    # Delete old chunks for changed files
    for file_path in changed_files:
        vector_store.delete(
            filter={"service": service_name, "file_path": file_path}
        )
    
    # Re-index only the changed files
    for file_path in changed_files:
        content = read_file(file_path)
        chunks = chunk_file(content, file_path)
        # ... embed and store as before
```

In CI/CD, trigger this on every merge to main:

```
# In your CI pipeline (e.g., GitHub Actions)
on:
  push:
    branches: [main]

jobs:
  update-rag:
    steps:
      - name: Get changed files
        id: changes
        run: |
          echo "files=$(git diff --name-only HEAD~1)" >> $GITHUB_OUTPUT
      
      - name: Update RAG index
        run: python scripts/update_rag.py --files "${{ steps.changes.outputs.files }}"
```

---

## Part Three: LLM Integration

With the RAG index in place, the LLM has eyes. Now it needs rules. The prompt structure determines what the LLM is allowed to do and how it should behave within those constraints. This is where the service manifest — filled in during Part Four — begins to earn its value: the allowed paths and dependencies in the manifest become the guardrails encoded in the system prompt.

### The Prompt Structure

The LLM needs three things in its prompt:
1. **System instructions:** What it should do and what constraints apply
2. **Retrieved context:** The relevant code chunks from RAG
3. **User query:** What the developer is asking

Here's the structure:

```
SYSTEM:
You are an expert developer working on the {service_name} microservice.
You must follow Clean Architecture principles.
You may only modify files within these paths: {allowed_paths}
You must not import from these forbidden modules: {forbidden_imports}

When proposing changes, output a unified diff format.
Always include or update tests for any logic changes.

CONTEXT:
The following code snippets are relevant to your task:

--- File: tests/domain/test_refund.py (lines 45-89) ---
{chunk_content}

--- File: domain/entities/refund.py (lines 1-67) ---
{chunk_content}

... more chunks ...

USER:
{user_query}
```

### Wiring It Together

The orchestration flow connects RAG retrieval to LLM generation:

```
function assist_with_code(user_query, service_name):
    # Step 1: Load service configuration from manifest
    service_config = load_manifest()["services"][service_name]
    allowed_paths = service_config["paths"]
    dependencies = service_config["dependencies"]
    
    # Step 2: Retrieve relevant context
    context_chunks = retrieve_context(user_query, service_name)
    
    # Step 3: Format context for prompt
    formatted_context = ""
    for chunk in context_chunks:
        formatted_context += f"""
--- File: {chunk.metadata["file_path"]} (lines {chunk.metadata["start_line"]}-{chunk.metadata["end_line"]}) ---
{chunk.content}

"""
    
    # Step 4: Build the prompt
    system_prompt = f"""
You are an expert developer working on the {service_name} microservice.
Follow Clean Architecture: domain has no external dependencies, 
application depends only on domain, infrastructure implements interfaces.

You may ONLY modify files matching these patterns: {allowed_paths}
You may ONLY import from: {dependencies}

Output proposed changes as unified diffs.
Include tests for any logic changes.
"""
    
    # Step 5: Call the LLM
    response = llm.chat([
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": f"CONTEXT:\n{formatted_context}\n\nQUERY:\n{user_query}"}
    ])
    
    # Step 6: Parse and validate the response
    proposed_changes = parse_diff(response.content)
    validation_errors = validate_changes(proposed_changes, allowed_paths, dependencies)
    
    if validation_errors:
        # Ask LLM to fix violations
        return assist_with_code(
            f"Your changes had these violations: {validation_errors}. Please fix them.\n\nOriginal query: {user_query}",
            service_name
        )
    
    return proposed_changes
```

### Handling Conversations

For multi-turn conversations, maintain conversation history but be aggressive about pruning irrelevant context:

```
function continue_conversation(conversation_history, new_query, service_name):
    # Retrieve fresh context for the new query
    new_context = retrieve_context(new_query, service_name)
    
    # Summarise old conversation if it's getting long
    if count_tokens(conversation_history) > 4000:
        summary = llm.chat([
            {"role": "system", "content": "Summarise this conversation, keeping key decisions and context."},
            {"role": "user", "content": format_conversation(conversation_history)}
        ])
        conversation_history = [{"role": "assistant", "content": f"Previous discussion summary: {summary}"}]
    
    # Add new context and query
    conversation_history.append({
        "role": "user", 
        "content": f"UPDATED CONTEXT:\n{format_context(new_context)}\n\nNEW QUERY:\n{new_query}"
    })
    
    response = llm.chat(conversation_history)
    conversation_history.append({"role": "assistant", "content": response.content})
    
    return response, conversation_history
```

**Key insight:** Don't let conversations grow unbounded. Fresh RAG retrieval for each query is better than accumulating stale context.

---

## Part Four: The Manifest and Enforcement

The RAG index and LLM integration together give you a capable coding assistant. The manifest and enforcement layer is what makes it a *trustworthy* one. Without enforcement, the LLM can produce code that violates service boundaries — and the violation will only be discovered in code review, or worse, in production. The manifest is the machine-readable form of the bounded context declarations described in Paper 1, Section 1.4. The enforcement scripts and CI pipeline are the guardrails described in Paper 3, Part Three. Together, they make boundary violations impossible to merge without explicit override.

### The Manifest Schema

The manifest is the single source of truth for your architecture. Here's the complete schema:

```
{
  "$schema": "monorepo-manifest-v1",
  
  "boundedContexts": {
    "<context-name>": {
      "description": "string - what this context is responsible for",
      "glossary": "string - path to ubiquitous language definitions",
      "owner": "string - team responsible for this context",
      "microservices": ["array of microservice names in this context"]
    }
  },
  
  "microservices": {
    "<service-name>": {
      "boundedContext": "string - which BC this service belongs to",
      "aggregate": "string - which aggregate this service owns",
      "paths": ["array of glob patterns for allowed file paths"],
      "dependencies": ["array of allowed import paths"],
      "owner": "string - team or individual owner",
      "contacts": ["array of contact emails"],
      
      "contracts": {
        "provides": ["array of contract IDs this service implements"],
        "consumes": ["array of contract IDs this service depends on"],
        "consumers": ["array of service names that consume this service"]
      },
      
      "database": {
        "type": "postgres|mysql|mongodb|dynamodb",
        "schema": "string - database schema name",
        "migrations": "string - path to migration files",
        "pii": ["array of field names containing PII"]
      }
    }
  },
  
  "contracts": {
    "<contract-id>": {
      "type": "rest|grpc|event",
      "version": "string - semantic version",
      "schema": "string - path to OpenAPI/AsyncAPI/protobuf file",
      "owner": "string - service that owns this contract"
    }
  },
  
  "sharedLibraries": {
    "<library-name>": {
      "paths": ["array of glob patterns"],
      "consumers": ["array of services that may use this library"],
      "owner": "string - team responsible"
    }
  }
}
```

### Enforcement Scripts

These scripts run in CI to enforce boundaries. They're simple—the power comes from the manifest they read.

**Path Validator:**

```
function validate_paths(changed_files, service_name, manifest):
    service = manifest["microservices"][service_name]
    allowed_patterns = service["paths"]
    
    violations = []
    for file_path in changed_files:
        if not matches_any_pattern(file_path, allowed_patterns):
            violations.append({
                "file": file_path,
                "error": f"File not in allowed paths for {service_name}",
                "allowed": allowed_patterns
            })
    
    return violations  # Empty list means pass
```

**Dependency Validator:**

```
function validate_dependencies(changed_files, service_name, manifest):
    service = manifest["microservices"][service_name]
    allowed_deps = service["dependencies"] + service["paths"]  # Can import from self
    
    violations = []
    for file_path in changed_files:
        imports = extract_imports(file_path)
        for imp in imports:
            if not matches_any_pattern(imp, allowed_deps):
                violations.append({
                    "file": file_path,
                    "import": imp,
                    "error": f"Import not in allowed dependencies for {service_name}",
                    "allowed": allowed_deps
                })
    
    return violations
```

**Architecture Layer Validator:**

```
function validate_layers(changed_files, service_name, manifest):
    violations = []
    
    for file_path in changed_files:
        layer = detect_layer(file_path)  # domain, application, infrastructure, api
        imports = extract_imports(file_path)
        
        for imp in imports:
            imp_layer = detect_layer(imp)
            
            # Domain can't import from any other layer
            if layer == "domain" and imp_layer in ["application", "infrastructure", "api"]:
                violations.append({
                    "file": file_path,
                    "import": imp,
                    "error": f"Domain layer cannot import from {imp_layer}"
                })
            
            # Application can't import from infrastructure (only interfaces)
            if layer == "application" and imp_layer == "infrastructure":
                if "ports" not in imp and "interfaces" not in imp:
                    violations.append({
                        "file": file_path,
                        "import": imp,
                        "error": "Application layer can only import infrastructure interfaces, not implementations"
                    })
    
    return violations
```

**Pre-flight Diff Validator:**

This runs BEFORE the LLM's proposed changes are applied:

```
function validate_proposed_diff(diff, service_name, manifest):
    # Parse the diff to extract affected files
    affected_files = extract_files_from_diff(diff)
    
    # Run all validators
    path_violations = validate_paths(affected_files, service_name, manifest)
    dep_violations = validate_dependencies(affected_files, service_name, manifest)
    layer_violations = validate_layers(affected_files, service_name, manifest)
    
    all_violations = path_violations + dep_violations + layer_violations
    
    if all_violations:
        # Don't apply the diff - return errors to the LLM
        return {
            "status": "rejected",
            "violations": all_violations,
            "message": "Proposed changes violate architectural constraints"
        }
    
    return {"status": "approved", "violations": []}
```

### CI Pipeline Integration

This is the enforcement mechanism from Paper 3, Part Three — the pipeline that makes boundary violations impossible to merge without an explicit override decision. The CI checks are not advisory; they block the PR. Every validator that passes is a guarantee, not a hope.

Here's how these validators integrate into a GitHub Actions workflow:

```
# .github/workflows/validate.yaml

name: Architectural Validation

on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for diff
      
      - name: Determine affected service
        id: service
        run: |
          # Find which service the changes belong to
          changed_files=$(git diff --name-only origin/main...HEAD)
          service=$(python scripts/detect_service.py --files "$changed_files")
          echo "name=$service" >> $GITHUB_OUTPUT
      
      - name: Validate paths
        run: |
          python scripts/validate_paths.py \
            --service "${{ steps.service.outputs.name }}" \
            --manifest monorepo.json
      
      - name: Validate dependencies
        run: |
          python scripts/validate_deps.py \
            --service "${{ steps.service.outputs.name }}" \
            --manifest monorepo.json
      
      - name: Validate architecture layers
        run: |
          python scripts/validate_layers.py \
            --service "${{ steps.service.outputs.name }}" \
            --manifest monorepo.json
      
      - name: Run contract tests
        run: |
          cd services/${{ steps.service.outputs.name }}
          pytest tests/contract -v
```

---

## Part Five: A Lightweight Approach for Prototyping

### The Single-Developer Version

If you're prototyping alone, much of the cross-team coordination doesn't apply. Here's what you actually need:

**Keep:**
- RAG indexing (essential for focused LLM context)
- The manifest (even for one service, it clarifies boundaries)
- Path and dependency validation (catches mistakes early)
- Basic contract definitions (even if you're the only consumer)

**Skip for now:**
- Consumer contract tests (no consumers yet)
- Cross-service impact analysis (only one service)
- CODEOWNERS and approval workflows (you're the only owner)
- Team coordination processes

**Simplify:**
- One bounded context to start
- One or two microservices maximum
- Local ChromaDB instead of hosted vector DB
- Manual RAG updates instead of CI-triggered

### Minimum Viable Setup

Here's what you can build in a day:

**Hour 1-2: Project structure**

```
my-prototype/
├── monorepo.json                 # Manifest
├── services/
│   └── eligibility-decision/     # One service to start
│       ├── domain/
│       ├── application/
│       ├── infrastructure/
│       ├── api/
│       └── tests/
├── contracts/
│   └── eligibility-decision-api-v1.yaml
├── scripts/
│   ├── index_service.py          # RAG indexer
│   ├── validate_paths.py         # Path validator
│   └── assist.py                 # LLM integration
└── .github/
    └── workflows/
        └── validate.yaml         # CI pipeline
```

**Hour 3-4: RAG setup**

Write a simple indexer that:
- Reads all Python files in `services/eligibility-decision/`
- Chunks them by function/class
- Embeds with OpenAI
- Stores in local ChromaDB

**Hour 5-6: LLM integration**

Write a simple CLI that:
- Takes a query as input
- Retrieves context from ChromaDB
- Sends to GPT-4 with system prompt
- Prints the response

**Hour 7-8: Basic validation**

Write path validation script and add to GitHub Actions.

You now have a working prototype of the framework. It's not production-ready, but it demonstrates the core concepts and gives you something to iterate on.

### What to Add Next

Once the basics work, add in order of value:

1. **Dependency validation** - Catches architectural violations
2. **Layer validation** - Ensures Clean Architecture compliance
3. **Contract definitions** - Even simple OpenAPI specs help
4. **Automated RAG updates** - Trigger on git push
5. **Multi-turn conversations** - Better developer experience
6. **Second service** - Test cross-service boundaries
7. **Contract tests** - When you have consumers

---

## Part Six: Common Problems and Solutions

The problems below are drawn from teams at different stages of implementation. They're ordered from most to least frequently encountered.

### RAG Retrieval Problems

**Problem: RAG index gets stale between code changes**

Symptoms: The LLM suggests patterns that no longer exist, references functions that have been renamed, or seems unaware of recent refactors. Developers notice the LLM working from an outdated version of the code.

Causes and solutions:
- **Missing CI trigger:** Ensure the RAG update job runs on every merge to main, not just nightly or manually.
- **Update script fails silently:** A failed index update should fail the CI run. Add explicit success/failure reporting to the RAG update step.
- **Partial update bug:** New chunks are added without deleting old ones for changed files. Verify your update logic deletes existing chunks for changed files before re-indexing.

Re-indexing strategy by scale:

| Scenario | Strategy | Expected time |
|----------|----------|--------------|
| < 10 services, < 50 files each | Full re-index on every merge | < 5 minutes |
| 10–50 services | Incremental update: changed files only | < 2 minutes |
| 50+ services | Per-service re-index, triggered by path filter | < 1 minute per service |

**Problem: False-negative retrieval (relevant code doesn't appear)**

Symptoms: The LLM is unaware of code you know exists. It suggests solutions that duplicate existing logic, misses important constraints, or contradicts code that wasn't retrieved.

Causes and solutions:
- **File not indexed:** Verify the file is in the index with `list_service_chunks(service_name)` filtered by file path.
- **Query doesn't match vocabulary:** The code uses `AbcPolicy` but the query says `pricing rule`. Use the service's ubiquitous language glossary (defined in Paper 1) as prompt context, or add synonym mapping to your embedding step.
- **Chunk boundary splits a function:** The function spans two chunks and neither half alone has enough signal to surface it. Adjust chunking to respect function/class boundaries. Increase overlap from 50 to 150 tokens.
- **Embedding model mismatch:** Code was indexed with `text-embedding-ada-002` but a different model is used at query time. Indexing and retrieval must use identical models.

Diagnostic command for false-negative investigation:

```python
# Query directly with your search term to see what ranks
results = vector_store.query(
    "pricing rule",
    filter={"service": "my-service"},
    top_k=20
)
print([(r.metadata["file_path"], r.similarity_score) for r in results])
```

**Problem: LLM gets irrelevant context**

Symptoms: The LLM makes changes to the wrong files or misunderstands the codebase.

Causes and solutions:
- **Chunk size too large:** Reduce to 300–500 tokens. Large chunks dilute relevance.
- **Poor metadata:** Ensure file type classification is accurate. If tests aren't tagged as tests, they won't be prioritised.
- **Query too vague:** "Fix the bug" retrieves poorly; "Fix the refund amount validation in ProcessRefund use case" retrieves well.

### LLM Generation Problems

**Problem: LLM ignores retrieved context**

Symptoms: The LLM generates code from scratch rather than extending existing patterns, or contradicts conventions visible in the retrieved chunks. The prompt includes relevant examples but the output doesn't reflect them.

Causes and solutions:
- **Context is buried too deep in the prompt:** Move CONTEXT to immediately precede the USER message. LLMs attend to recent content more than distant content.
- **Context is too long and diluted:** If retrieved chunks total more than 4,000 tokens, the signal is diluted. Reduce top_k from 12 to 6, ensuring only the most relevant chunks are included.
- **Generic system prompt overrides examples:** If your system prompt prescribes how to write code very specifically, the model follows that over the provided examples. Reference the context explicitly: "Use the patterns shown in the CONTEXT section."
- **Wrong retrieval filter:** The query retrieved code from a different layer than needed. Filter by `chunk_type=domain` when asking about business rules; `chunk_type=application` for use case changes.

**Problem: LLM proposes changes outside allowed paths**

Symptoms: Validation rejects the diff.

Solutions:
- Make the system prompt more explicit about allowed paths
- Include a concrete example of a valid change in the prompt
- If it persists, the LLM may not understand the boundary — simplify the constraint description and test with a simpler query first

**Problem: LLM generates code that doesn't compile**

Symptoms: Syntax errors, import errors, type mismatches.

Solutions:
- Include more context (increase top_k) — missing type definitions are a common cause
- Add a validation step that attempts to parse/compile proposed code before showing it to the developer. For Python: `ast.parse()`. For TypeScript: `tsc --noEmit`.
- Ensure retrieved context includes imports and type definitions from the same file as the function being modified

**Problem: LLM ignores Clean Architecture**

Symptoms: Domain code imports from infrastructure, business logic in controllers.

Solutions:
- Make layer rules explicit in the system prompt with examples of what's forbidden and why
- Include architecture tests in the RAG context — the LLM will see what patterns are expected and what the tests catch
- Add post-generation validation that checks layer dependencies before accepting changes

### Contract Testing Problems

**Problem: Contract tests are too brittle**

Symptoms: Contract tests fail after changes that shouldn't have broken them. Every minor refactor triggers failures across consumer tests.

Causes and solutions:
- **Consumer tests assert too much:** Consumer tests should assert only what the consumer actually uses. If a consumer only uses `id` and `status`, assert only those fields. Don't copy the full provider response schema into consumer test fixtures.
- **Provider tests are too loose:** Provider tests must verify every field in the contract. Generate test data that exercises all contract fields, including optional ones.
- **Contract not versioned:** When the contract changes, the version must change. If consumers pin to `v1`, changes to `v1` contents break them. Create `v2` and migrate consumers explicitly.

**Problem: Contract tests fail but code is correct**

Symptoms: Provider tests pass but consumer tests fail.

Causes and solutions:
- Ensure provider and consumer use the same contract version
- Verify test fixtures match actual contract schema (check for optional field ordering issues, date format representation differences)
- For breaking changes, follow the versioning workflow — create a new contract version rather than modifying an existing one

### CI/CD Problems

**Problem: Validation is too slow**

Symptoms: CI takes 10+ minutes; developers bypass it.

Solutions:
- Only validate changed files, not the entire codebase
- Cache dependencies and the manifest parse result
- Run validators in parallel
- Run lightweight validation (path/dep check) on PRs; full contract testing on merge to main

**Problem: False positives in validation**

Symptoms: Valid code is rejected.

Solutions:
- Review the manifest — are allowed paths correctly specified?
- Check for edge cases in glob patterns (`services/payments/**` vs `services/payments/*`)
- Add escape hatches for legitimate exceptions, but require them to be logged and reviewed. An escape hatch that's never reviewed is a silent vulnerability.

### Organisational Problems

**Problem: Teams resist the manifest**

Symptoms: Teams treat the manifest as bureaucratic overhead. They delay updating it, request exceptions, or submit manifest-only PRs long after the code changes they describe.

Causes and solutions:
- **Reframe it as ownership, not compliance:** The manifest is how a team declares what they own. Owning something in the manifest means making decisions about it without asking permission. Teams resistant to the manifest often haven't internalised that declaration cuts both ways.
- **Remove the central committee bottleneck:** Teams should be able to update their own manifest entries without external approval. Reserve central review only for new bounded contexts or cross-context dependency additions. A central approval process kills motivation.
- **Make enforcement visible:** If the CI pipeline doesn't actually fail on boundary violations, the manifest becomes documentation that drifts. Make violations red in CI so teams see the connection between the manifest and real consequences.
- **Transfer ownership to service teams:** If the platform team currently manages the manifest, transfer each service's entry to the owning team. Platform owns the schema and tooling; service teams own their data.

**Problem: Developers bypass the framework**

Symptoms: Direct commits to main, manual deployments, ignored validation failures.

Solutions:
- Make the framework faster and less intrusive first — friction is the primary motivation for bypass
- Ensure the framework provides visible value (show developers the violations it caught before they reached review)
- Get management buy-in to require enforcement at the process level
- Make bypass require explicit approval and logging — not to punish, but to create a paper trail that surfaces patterns

---

## Part Seven: From Prototype to Production

### What Good Looks Like

After your first few weeks running the framework, use these benchmarks to calibrate your progress. They aren't hard targets — but if your numbers are far outside these ranges, they signal a specific problem to investigate.

**Token cost per engineer-day:**
| Stage | Interactions/day | Tokens/interaction | Daily cost (GPT-4) |
|-------|-----------------|-------------------|-------------------|
| Prototype (8 chunks, short context) | 10–20 | ~3,000 | £0.30–0.60 |
| Mature (12 chunks, full system prompt) | 20–40 | ~6,000 | £1.20–2.40 |
| High-volume (20+ changes, async workflows) | 50+ | ~8,000 | £4.00+ |

If costs are significantly above these figures, check chunk size and top_k first — those are the main levers.

**CI validation pass rate:**

| Maturity level | Expected pass rate | What failure signals |
|----------------|-------------------|---------------------|
| First 4 weeks (teams learning boundaries) | 85–90% | Normal learning curve |
| After 3 months | 95%+ | Systematic violation suggests manifest needs revision |
| Mature system | 99%+ | Each failure is a near-miss worth investigating |

If pass rate stays below 90% after month two, the most common causes are: manifest paths too narrow, teams working across service boundaries legitimately (possible missed bounded context split), or LLM system prompt not explicit enough about constraints.

**Boundary violation rate:**
- Transition period (first 3 months): up to 5–10 violations per week per service is not unusual
- After stabilisation: fewer than 1 violation per service per month is a healthy target
- If violations cluster around the same file boundary, that boundary may be wrong — investigate whether the manifest needs adjusting rather than suppressing the violations

**RAG retrieval quality check (weekly):** Ask the LLM to fix a bug you already fixed. If it proposes the same fix you implemented, retrieval is working. If it proposes something different, check whether the relevant file is indexed and whether your query matches the code's vocabulary.

**Time-to-first-guardrail:** Target running path validation in CI within 2 days of initial setup. A 5-day delay usually means a blocker (monorepo setup, API key provisioning, test scaffold missing) — surface it early rather than grinding through it.

### When to Scale Up

Your prototype works. Developers like it. Now what? Here are the signals that you need to scale up:

**Scale RAG infrastructure when:**
- Index size exceeds local ChromaDB comfort (>100K chunks)
- Multiple developers need simultaneous access
- You need persistent, reliable storage
- Index update speed becomes a bottleneck

**Move to hosted vector DB:** Pinecone, Weaviate Cloud, or similar.

**Scale CI/CD when:**
- Validation time exceeds 5 minutes
- You have more than 10 services
- Multiple teams need different pipelines
- You need sophisticated approval workflows

**Action:** Invest in CI infrastructure, parallel validation, caching.

**Scale governance when:**
- Multiple teams own different bounded contexts
- Contract changes affect multiple consumers
- You need audit trails for compliance
- Cross-team coordination becomes a bottleneck

**Action:** Implement formal governance workflows, contract review processes, architecture decision records.

### When to Change Your Tools

The prototype stack (ChromaDB, LangChain, GitHub Actions scripts) is intentionally simple. Here are the specific triggers that signal it's time to upgrade each component:

**Move from ChromaDB to a hosted vector database (Pinecone, Weaviate) when:**
- Index size exceeds 500,000 chunks (roughly 50 services × 200 files × 50 chunks/file)
- Multiple developers need simultaneous access and are experiencing contention
- Your RAG update job takes more than 10 minutes
- You need disaster recovery guarantees for the index

Don't move early. ChromaDB running locally or on a single server is surprisingly capable. The main failure mode is trying to share it across a team — that's the trigger, not the size.

**Move from LangChain to custom orchestration when:**
- You need retrieval behaviours LangChain doesn't support out of the box (custom re-ranking, multi-stage retrieval, graph-augmented RAG)
- LangChain's abstraction is hiding errors that are difficult to diagnose
- Upgrade costs: LangChain has broken backwards compatibility multiple times — if you're spending time on library upgrades, custom code may be cheaper to maintain

Don't move early. LangChain's value is speed of iteration in the prototype stage. Switch when the abstractions genuinely constrain you, not before.

**Move from GitHub Actions scripts to a platform pipeline when:**
- You have more than 20 services across multiple teams
- Different services need genuinely different pipeline configurations that are hard to manage in YAML
- You need cross-service orchestration (e.g., run consumers' contract tests when a provider changes) that requires a workflow engine
- Your compliance requirements demand audit trails that CI logs don't satisfy

The pattern: stay on the simpler tool until a specific pain point makes the cost of staying higher than the cost of migrating.

### The Production Checklist

Before going to production, ensure:

**RAG Infrastructure:**
- [ ] Hosted vector database with backups
- [ ] Automated index updates on every commit
- [ ] Monitoring for index health and retrieval quality
- [ ] Separate indexes per service (no cross-contamination)

**LLM Integration:**
- [ ] Rate limiting to control costs
- [ ] Logging of all queries and responses (for audit)
- [ ] Fallback behaviour when LLM is unavailable
- [ ] PII filtering to prevent sensitive data in prompts

**CI/CD:**
- [ ] All validators running on every PR
- [ ] Contract tests for all service interactions
- [ ] Automated RAG updates
- [ ] Deployment pipelines per service

**Governance:**
- [ ] Manifest is complete and accurate
- [ ] CODEOWNERS reflects actual ownership
- [ ] Contract change process is documented
- [ ] Escalation paths are clear

**Monitoring:**
- [ ] Token usage tracking (costs)
- [ ] Validation pass/fail rates
- [ ] RAG retrieval quality metrics
- [ ] Developer satisfaction surveys

---

## Quick Reference

### Commands You'll Use Daily

```
# Index a service
python scripts/index_service.py --service eligibility-decision

# Update index for changed files
python scripts/update_rag.py --files "domain/entities/decision.py,application/use_cases/create.py"

# Get LLM assistance
python scripts/assist.py --service eligibility-decision --query "Add partial refund support"

# Validate changes locally
python scripts/validate_all.py --service eligibility-decision

# Run contract tests
cd services/eligibility-decision && pytest tests/contract -v
```

### Key Files

| File | Purpose |
|------|---------|
| `monorepo.json` | Manifest defining all boundaries |
| `scripts/index_service.py` | Build RAG index for a service |
| `scripts/validate_paths.py` | Check files are in allowed paths |
| `scripts/validate_deps.py` | Check imports are allowed |
| `scripts/validate_layers.py` | Check Clean Architecture compliance |
| `scripts/assist.py` | LLM integration CLI |
| `.github/workflows/validate.yaml` | CI pipeline |

### Troubleshooting Quick Reference

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Irrelevant RAG results | Poor chunking or stale index | Re-chunk, update index |
| LLM violates boundaries | Weak system prompt | Make constraints more explicit |
| CI too slow | Validating everything | Validate only changed files |
| Contract test failures | Version mismatch | Sync contract versions |
| Import errors in generated code | Missing context | Increase top-k, check index |

---

## Conclusion

This paper has translated the conceptual framework of Papers 1-4 into practical implementation guidance. You now know:

- What tools to use and why
- How to build RAG indexes
- How to integrate LLMs
- How to enforce boundaries
- How to start small and scale up
- How to solve common problems

The framework is substantial, but you don't have to build it all at once. Start with a single service, basic RAG, and simple validation. Add sophistication as you learn what works in your context.

The papers that preceded this one explain why these patterns matter. This paper explains how to make them real. The rest is iteration, refinement, and learning from your specific challenges.

Build something. See what breaks. Fix it. Repeat.

---

## Related Papers

- **Paper 1: Domain-Driven Design & Clean Architecture** — The conceptual foundation for the boundaries this framework enforces. The manifest bounds context declarations you'll write in Part Four.

- **Paper 2: Evidence-Based Identity** — A domain-specific application of the DDD patterns, showing how the bounded context model works in practice.

- **Paper 3: LLM-Assisted Development** — The guardrails and RAG architecture in depth. Part Two and Part Four of this paper implement the concepts from Paper 3.

- **Paper 4: Automated Development** — The end-to-end pipeline from requirements to deployment. This paper covers the manual and semi-automated implementation; Paper 4 covers full automation.

- **Paper 6: Case Study** — End-to-end walkthrough applying the framework to a DWP benefits administration system. Useful as a reference for what a mature implementation looks like across multiple services.

---

*End of Paper 5*
