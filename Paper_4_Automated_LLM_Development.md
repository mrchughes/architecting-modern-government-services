# From Requirements to Running Systems

**Automated LLM-Driven Development for Microservice Architectures**

*Part 4 of 6 of the Architecting Modern Government Services Series*

*Version 1.0 | February 2026*

---

## Version History

| Version | Date | Changes |
|---------|------|--------|
| 1.0 | Feb 2026 | Initial release |

---

## Executive Summary

Papers 1-3 established the conceptual foundations: Domain-Driven Design provides the boundaries, Clean Architecture provides the internal structure, and Service-Scoped RAG with CI/CD guardrails makes LLM assistance safe. But how do you actually build systems this way?

This paper answers that question. It presents an automated pipeline where LLMs don't just assist developers—they drive the entire process from business requirements to deployed, tested microservices. The human role shifts from writing code to providing input, validating output, and making strategic decisions. Everything else is automated.

This isn't pair programming with an AI assistant. This is a factory that takes requirements as input and produces production-ready systems as output.

---

## The Vision: End-to-End Automation

Consider what happens today when an organisation decides to build a new capability. Business analysts write requirements documents. Architects translate requirements into domain models. Developers implement those models as code. DevOps engineers create deployment pipelines. QA teams write and execute tests. Release managers coordinate deployments. Each handoff introduces delay, miscommunication, and the opportunity for drift between intent and implementation.

Now consider the alternative. A business analyst describes what's needed in natural language. An automated pipeline takes that description and produces: a domain model with bounded contexts, aggregates, and ubiquitous language; microservice scaffolds with Clean Architecture layers; API contracts and event schemas; database schemas and migrations; CI/CD pipelines configured for the new services; deployment manifests ready for Kubernetes; comprehensive test suites at every level; and documentation that stays in sync with the code.

The human reviews the output, provides feedback, and approves. The system deploys. Changes flow through the same pipeline—describe what needs to change, review the proposed modifications, approve, deploy.

This is the vision. The rest of this paper shows how to build it.

---

## Part One: The Two-Phase Model

### Design Phase vs. Evolution Phase

The key insight that makes automation possible is recognising that system development has two fundamentally different modes, each requiring different approaches to LLM assistance.

**The Design Phase** is when boundaries don't exist yet. You're discovering the domain, identifying bounded contexts, defining aggregates, establishing contracts. The LLM operates with broad context and proposes options. Human judgment is essential because these decisions shape everything that follows. The output is frozen artefacts: manifests, contracts, schemas, and test suites that define the boundaries for all future work.

**The Evolution Phase** is when boundaries are established. You're implementing within constraints, adding features, fixing bugs, refactoring. The LLM operates with narrow, service-scoped context and must respect existing boundaries. Automation can be more aggressive because guardrails prevent violations. The output is code that conforms to existing artefacts.

The transition between phases is explicit. A bounded context moves from Design to Evolution when its contracts are defined, its aggregates are identified, its manifest entry is complete, and its RAG index is built. From that point, the system enforces those boundaries rather than helping discover them.

This two-phase model is what makes end-to-end automation possible. In the Design Phase, automation proposes and humans decide. In the Evolution Phase, automation executes and humans verify. Neither phase requires humans to write code—but the Design Phase requires humans to make strategic choices that no amount of automation can replace.

### Human Oversight: The Five Checkpoints

Before describing the pipeline stages, one structural point matters: this automation does not run from requirements to deployment in a single unattended pass. It stops at five explicit checkpoints — one per stage — where a person reviews the generated output and either approves it or requests revisions. No stage advances until the human clears its checkpoint.

"Human checkpoint" does not mean "human writes code." It means "human validates that the automated output captures intent correctly." The LLM proposes; humans decide. Automation handles the mechanical work between decision points, but every meaningful gate is human-held.

**Domain Model Checkpoint:** Read the proposed bounded contexts and aggregates. Ask: Do these boundaries match how the business actually thinks? Is the ubiquitous language correct? Are there concepts the LLM missed? The human doesn't write the domain model — they approve or request revisions to the generated one.

**Service Architecture Checkpoint:** Review the proposed microservice split. Ask: Are there too many services? Too few? Are ownership assignments realistic given team structure? Do dependencies make sense? The human doesn't draw architecture diagrams — they validate the generated architecture.

**Contracts Checkpoint:** Examine the generated API specs and event schemas. Ask: Do the endpoints make sense? Are the data structures complete? Will consumers be able to use these contracts? The human doesn't write OpenAPI specs — they review and approve the generated ones.

**Scaffolds Checkpoint:** Spot-check the generated code structure. Ask: Is the folder structure correct? Are the layers properly separated? Does the boilerplate look reasonable? The human doesn't write scaffold code — they verify the generation templates produced something sensible.

**Deployment Checkpoint:** Verify that services come up, health checks pass, and basic smoke tests succeed. The human doesn't write Helm charts — they monitor the deployment.

The time spent at each checkpoint should be measured in minutes to hours, not days. If humans are spending days at a checkpoint, either the automation output is poor quality (fix the templates) or the humans are overstepping into implementation (trust the automation).

### The Artefact Pipeline

Everything flows through artefacts. An artefact is a machine-readable definition that captures a decision and enables enforcement. The pipeline produces artefacts at each stage, and each artefact becomes input to the next stage.

The pipeline runs through five stages in sequence. **Stage 1** takes business requirements — user stories, process descriptions, policy documents — and produces a structured domain model: bounded contexts, aggregates, and a ubiquitous language glossary. **Stage 2** takes that domain model and generates a service manifest defining microservices, ownership, and paths. **Stage 3** generates contracts from the manifest: OpenAPI specifications, event schemas, and shared type definitions. **Stage 4** takes those contracts and generates implementation scaffolds — the full Clean Architecture directory structure, interfaces, and boilerplate for each service. **Stage 5** takes the scaffolds and produces a deployed, tested system: Helm charts, CI/CD pipelines, database migration scripts, and health check configurations.

Each stage is gated by one of the five human checkpoints described above. The pipeline pauses after each stage until the checkpoint is cleared. If a checkpoint identifies problems, the stage re-runs with corrections before advancing.

Each artefact is versioned, stored in the repository, and becomes the source of truth for subsequent stages. When requirements change, the pipeline re-runs from the affected stage, propagating changes through all downstream artefacts automatically.

### When Human Review Triggers a Revision

The pipeline above might suggest that revisions are rare exceptions. In practice, particularly during the Design Phase, partial revision is the norm. Understanding the revision loop matters because it determines whether the pipeline reduces friction or adds it.

When a human reviewer identifies a problem at a checkpoint, the workflow is:

1. **Annotate the output.** The reviewer marks specific issues on the generated artefact: "This bounded context conflates two distinct responsibilities — order creation and order fulfilment are driven by different actors and different change rates." Annotations are precise; they identify the problem rather than prescribing the fix.

2. **Re-run the affected stage.** The pipeline re-runs Stage N with the original inputs plus the reviewer's annotations as additional context. The LLM incorporates the feedback and produces a revised output.

3. **Review the revision.** The reviewer checks whether the revision addresses the annotated issues. If yes, the checkpoint is cleared. If not, the cycle repeats.

4. **Advance when cleared.** Only after the checkpoint is cleared does the pipeline advance to Stage N+1.

The key principle: revisions always target the artefact, not the implementation. If Stage 3 generates a contract with missing error response codes, the contract is revised and Stage 4 regenerates the scaffold from the corrected contract. Developers never patch the scaffold by hand — that would create drift between the artefact and the implementation, defeating the purpose of generation.

In practice, Stage 1 (domain model) requires the most revision cycles, particularly early in a project when the requirements corpus is thin. Stage 4 (scaffolds) requires the fewest — once contracts are clean, scaffold generation is highly reliable.

---

## Part Two: Automated Domain Discovery

Domain discovery is the most analytically intensive step in microservice design. A human architect doing this manually spends days in workshops, drawing context maps, debating language boundaries, and negotiating ownership — then produces a document that drifts from reality the moment implementation begins. Automated discovery produces a first-draft domain model in minutes, applies consistent analysis across every requirement, and outputs machine-readable artefacts the pipeline can act on immediately. The human role shifts from creating the model to validating one — a faster task with fewer opportunities for inconsistency or personal bias.

### From Requirements to Bounded Contexts

The first automation challenge is the hardest: taking unstructured business requirements and producing a coherent domain model. This is where most "AI-assisted development" tools give up and leave the work to humans. But with the right approach, LLMs can do the heavy lifting while humans make the final calls.

The process starts with requirements ingestion. Business requirements arrive in many forms: user stories in Jira, process diagrams in Visio, policy documents in Word, stakeholder interview transcripts, legacy system documentation. The first step is to consolidate these into a requirements corpus—a structured collection that the LLM can analyse.

The ingestion pipeline normalises different formats into a common structure:

```json
{
  "requirements": [
    {
      "id": "REQ-001",
      "source": "stakeholder-interview-2026-01-15.md",
      "type": "capability",
      "description": "Citizens must be able to submit benefit claims online",
      "actors": ["citizen", "case-worker"],
      "entities_mentioned": ["claim", "citizen", "benefit", "evidence"]
    }
  ]
}
```

With the requirements corpus assembled, the LLM performs domain analysis. It identifies candidate bounded contexts by looking for clusters of related concepts, language boundaries (where the same word means different things), different business authorities (who owns the truth), and different change drivers (why would this logic change).

The output is a proposed domain model:

```json
{
  "domain": "benefits-administration",
  "bounded_contexts": [
    {
      "name": "claims-intake",
      "description": "Handling citizen claim submissions and initial processing",
      "ubiquitous_language": {
        "claim": "A citizen's request for benefit entitlement",
        "submission": "The act of providing a claim with supporting evidence",
        "evidence": "Documents supporting claim assertions"
      },
      "aggregates": ["Claim", "Submission", "EvidencePackage"],
      "authority": "intake-team",
      "change_drivers": ["submission-channels", "evidence-requirements"]
    },
    {
      "name": "eligibility-determination",
      "description": "Applying policy rules to determine benefit entitlement",
      "ubiquitous_language": {
        "claim": "A case under assessment for eligibility",
        "assessment": "The process of applying rules to determine eligibility",
        "decision": "The outcome of an eligibility assessment"
      },
      "aggregates": ["EligibilityCase", "Assessment", "Decision"],
      "authority": "policy-team",
      "change_drivers": ["legislation", "policy-updates"]
    }
  ],
  "context_relationships": [
    {
      "upstream": "claims-intake",
      "downstream": "eligibility-determination",
      "relationship": "customer-supplier",
      "translation": "Claim submission becomes EligibilityCase"
    }
  ]
}
```

Notice how the LLM has identified that "claim" means different things in different contexts—a signal that these are genuinely separate bounded contexts, not arbitrary divisions. This is the kind of analysis that would take a human architect days of workshops; the LLM produces a first draft in minutes.

But it's a draft. The human checkpoint is essential. Domain experts review the proposed contexts: Are the boundaries in the right places? Does the ubiquitous language match how the business actually talks? Are aggregates correctly identified? The LLM proposes; humans validate and refine.

### Generating the Ubiquitous Language Glossary

One of the most valuable outputs of domain discovery is the ubiquitous language glossary—a precise dictionary of terms used within each bounded context. This glossary becomes a RAG artefact that guides all future LLM assistance within that context.

The glossary generation process analyses requirements for term usage, identifies terms that appear across multiple contexts, flags terms with ambiguous or conflicting definitions, proposes precise definitions for each term within each context, and generates cross-context translation mappings.

The output is a machine-readable glossary:

```yaml
bounded_context: eligibility-determination
terms:
  - term: "claimant"
    definition: "A person whose eligibility for benefits is being assessed"
    distinguished_from:
      - context: "claims-intake"
        term: "applicant"
        relationship: "same person, different lifecycle stage"
    attributes:
      - "identity_reference"
      - "household_composition"
      - "income_details"
    invariants:
      - "A claimant must have a verified identity"
      - "Claimant attributes must be evidence-backed"
    
  - term: "assessment"
    definition: "The process of applying policy rules to determine eligibility"
    distinguished_from:
      - context: "evidence-validation"
        term: "verification"
        relationship: "verification produces inputs to assessment"
    lifecycle:
      - "initiated"
      - "evidence-gathering"
      - "rules-applied"
      - "decision-pending"
      - "complete"
```

This glossary does three things. First, it ensures everyone uses the same language—developers, business analysts, domain experts. Second, it provides context for LLM assistance: when an LLM helps with code in the eligibility-determination context, it uses "claimant" not "applicant," "assessment" not "verification." Third, it documents the boundaries: when terms differ across contexts, that's a boundary the system must respect.

### Identifying Aggregates and Invariants

Within each bounded context, the LLM identifies aggregates—clusters of entities that must change together to maintain consistency. This is pattern recognition at scale: the LLM analyses requirements for transactional boundaries, consistency requirements, and business rules that span multiple entities.

For each proposed aggregate, the LLM generates:

```yaml
aggregate: EligibilityDecision
root_entity: Decision
child_entities:
  - RuleApplication
  - EvidenceReference
  - ConditionOutcome
value_objects:
  - PolicyVersion
  - ConfidenceScore
  - DecisionRationale

invariants:
  - name: "evidence-minimum"
    description: "A decision cannot be approved without minimum evidence confidence"
    rule: "sum(evidence.confidence) >= policy.minimum_confidence"
    enforcement: "validate on state transition to 'approved'"
    
  - name: "rule-completeness"
    description: "All applicable rules must be evaluated before decision"
    rule: "applied_rules.covers(policy.required_rules)"
    enforcement: "validate on state transition to 'complete'"

state_transitions:
  - from: "initiated"
    to: "evidence-gathering"
    trigger: "startAssessment()"
    
  - from: "evidence-gathering"
    to: "rules-applied"
    trigger: "applyRules()"
    guards: ["sufficient-evidence"]
    
  - from: "rules-applied"
    to: "complete"
    trigger: "finaliseDecision()"
    guards: ["all-rules-evaluated"]
    emits: ["DecisionCompleted"]
```

This aggregate specification becomes input to code generation. The invariants become validation logic. The state transitions become the aggregate's public interface. The emitted events become the contracts with other services.

---

## Part Three: Automated Service Generation

Domain discovery (Part Two) established *what* the system contains: bounded contexts, aggregates, and a shared vocabulary. Service generation begins here — translating those domain definitions into deployable technical artefacts. This is the stage where automation delivers its most tangible efficiency gain: the mechanical work of writing boilerplate, wiring Clean Architecture layers together, and maintaining consistency across dozens of services is overwhelming for humans but routine for a template-driven pipeline. What remains for humans is the judgment layer — verifying that the generated artefacts make architectural sense before they become the foundation for everything downstream.

### From Domain Model to Service Manifest

With the domain model defined, the next stage generates the service architecture. The LLM takes the bounded contexts and aggregates and produces a service manifest that defines how they'll be implemented as microservices.

The mapping follows clear rules: each aggregate typically becomes one microservice (for reasons explained in Paper 1). Multiple microservices within a bounded context share the same ubiquitous language but own different data. Cross-context communication requires explicit contracts.

The generated manifest:

```json
{
  "boundedContexts": {
    "eligibility-determination": {
      "description": "Applying policy rules to determine benefit entitlement",
      "glossary": "docs/eligibility/glossary.yaml",
      "microservices": ["eligibility-decision-service", "policy-rules-service"]
    }
  },
  
  "microservices": {
    "eligibility-decision-service": {
      "boundedContext": "eligibility-determination",
      "aggregate": "EligibilityDecision",
      "paths": ["services/eligibility-decision/**"],
      "dependencies": [
        "libs/eligibility-types",
        "libs/events-common"
      ],
      "owner": "eligibility-team",
      "contracts": {
        "provides": ["eligibility-decision-api-v1", "decision-events-v1"],
        "consumes": ["customer-profile-api-v1", "evidence-api-v1"]
      },
      "database": {
        "type": "postgres",
        "schema": "eligibility_decisions",
        "migrations": "services/eligibility-decision/migrations"
      }
    }
  }
}
```

The manifest is the single source of truth for the service architecture. It's committed to the repository, versioned, and becomes the foundation for all enforcement. CI/CD reads the manifest to validate changes. RAG indexes are built based on manifest paths. Ownership and approval workflows follow manifest declarations.

### Generating Contracts First

Before generating any implementation code, the pipeline generates contracts. This is contract-first design enforced by automation: you cannot have implementation without a contract, because the contract is what the implementation implements.

For each service's provided contracts, the LLM generates OpenAPI specifications for REST APIs:

```yaml
openapi: 3.0.0
info:
  title: Eligibility Decision API
  version: 1.0.0
  description: API for managing eligibility decisions

paths:
  /decisions:
    post:
      operationId: createDecision
      summary: Initiate a new eligibility decision
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateDecisionRequest'
      responses:
        '201':
          description: Decision initiated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Decision'
        '400':
          description: Invalid request
          
  /decisions/{decisionId}:
    get:
      operationId: getDecision
      summary: Retrieve a decision by ID
      parameters:
        - name: decisionId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Decision found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Decision'
        '404':
          description: Decision not found

components:
  schemas:
    Decision:
      type: object
      required:
        - id
        - claimantId
        - status
        - createdAt
      properties:
        id:
          type: string
          format: uuid
        claimantId:
          type: string
          format: uuid
        status:
          type: string
          enum: [initiated, evidence_gathering, rules_applied, complete]
        outcome:
          type: string
          enum: [eligible, ineligible, pending_review]
        confidenceScore:
          type: number
          minimum: 0
          maximum: 1
        createdAt:
          type: string
          format: date-time
```

For events, the LLM generates AsyncAPI specifications or JSON Schema:

```yaml
asyncapi: 2.6.0
info:
  title: Eligibility Decision Events
  version: 1.0.0

channels:
  eligibility/decisions:
    publish:
      message:
        oneOf:
          - $ref: '#/components/messages/DecisionInitiated'
          - $ref: '#/components/messages/DecisionCompleted'

components:
  messages:
    DecisionCompleted:
      name: DecisionCompleted
      contentType: application/json
      payload:
        type: object
        required:
          - decisionId
          - claimantId
          - outcome
          - completedAt
        properties:
          decisionId:
            type: string
            format: uuid
          claimantId:
            type: string
            format: uuid
          outcome:
            type: string
            enum: [eligible, ineligible, pending_review]
          confidenceScore:
            type: number
          completedAt:
            type: string
            format: date-time
```

These contracts are frozen artefacts. Once approved, they can only change through a formal contract evolution process that ensures backward compatibility and coordinates with all consumers.

### Generating Clean Architecture Scaffolds

With contracts defined, the pipeline generates implementation scaffolds. This is where Clean Architecture templates meet the specific domain model to produce working code structure.

The generation process creates each layer:

**Domain Layer** — Pure business logic with no external dependencies:

```python
# services/eligibility-decision/domain/entities/decision.py

from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional
from uuid import UUID

from .value_objects import ConfidenceScore, PolicyVersion, DecisionRationale
from .events import DecisionCompleted, DecisionInitiated


@dataclass
class RuleApplication:
    """A single rule evaluation within a decision."""
    rule_id: str
    rule_version: PolicyVersion
    outcome: bool
    confidence: ConfidenceScore
    rationale: str


@dataclass  
class EvidenceReference:
    """Reference to evidence used in this decision."""
    evidence_id: UUID
    evidence_type: str
    confidence: ConfidenceScore


class EligibilityDecision:
    """
    Aggregate root for eligibility decisions.
    
    Invariants:
    - Cannot approve without minimum evidence confidence
    - All applicable rules must be evaluated before completion
    """
    
    def __init__(
        self,
        decision_id: UUID,
        claimant_id: UUID,
        policy_version: PolicyVersion
    ):
        self._id = decision_id
        self._claimant_id = claimant_id
        self._policy_version = policy_version
        self._status = "initiated"
        self._rule_applications: List[RuleApplication] = []
        self._evidence_references: List[EvidenceReference] = []
        self._outcome: Optional[str] = None
        self._confidence_score: Optional[ConfidenceScore] = None
        self._created_at = datetime.utcnow()
        self._events: List = []
        
        self._events.append(DecisionInitiated(
            decision_id=self._id,
            claimant_id=self._claimant_id,
            initiated_at=self._created_at
        ))
    
    @property
    def id(self) -> UUID:
        return self._id
    
    @property
    def status(self) -> str:
        return self._status
    
    def add_evidence(self, evidence: EvidenceReference) -> None:
        """Add evidence reference to this decision."""
        if self._status not in ["initiated", "evidence_gathering"]:
            raise InvalidStateError("Cannot add evidence after rules applied")
        
        self._evidence_references.append(evidence)
        self._status = "evidence_gathering"
    
    def apply_rule(self, application: RuleApplication) -> None:
        """Record a rule application."""
        if self._status not in ["evidence_gathering", "rules_applied"]:
            raise InvalidStateError("Must gather evidence before applying rules")
        
        self._rule_applications.append(application)
        self._status = "rules_applied"
    
    def finalise(self, minimum_confidence: float, required_rules: List[str]) -> None:
        """
        Finalise the decision.
        
        Enforces invariants:
        - Minimum evidence confidence must be met
        - All required rules must be evaluated
        """
        # Invariant: minimum evidence confidence
        total_confidence = sum(e.confidence.value for e in self._evidence_references)
        if total_confidence < minimum_confidence:
            raise InvariantViolation(
                f"Evidence confidence {total_confidence} below minimum {minimum_confidence}"
            )
        
        # Invariant: all required rules evaluated
        applied_rule_ids = {r.rule_id for r in self._rule_applications}
        missing_rules = set(required_rules) - applied_rule_ids
        if missing_rules:
            raise InvariantViolation(f"Missing required rules: {missing_rules}")
        
        # Calculate outcome
        all_rules_pass = all(r.outcome for r in self._rule_applications)
        self._outcome = "eligible" if all_rules_pass else "ineligible"
        self._confidence_score = ConfidenceScore(
            sum(r.confidence.value for r in self._rule_applications) / len(self._rule_applications)
        )
        self._status = "complete"
        
        self._events.append(DecisionCompleted(
            decision_id=self._id,
            claimant_id=self._claimant_id,
            outcome=self._outcome,
            confidence_score=self._confidence_score.value,
            completed_at=datetime.utcnow()
        ))
    
    def collect_events(self) -> List:
        """Collect and clear domain events."""
        events = self._events.copy()
        self._events.clear()
        return events
```

**Application Layer** — Use cases that orchestrate domain logic:

```python
# services/eligibility-decision/application/use_cases/create_decision.py

from dataclasses import dataclass
from uuid import UUID, uuid4

from ..ports import DecisionRepository, EventPublisher, PolicyService
from ...domain.entities import EligibilityDecision
from ...domain.value_objects import PolicyVersion


@dataclass
class CreateDecisionRequest:
    claimant_id: UUID


@dataclass
class CreateDecisionResponse:
    decision_id: UUID
    status: str


class CreateDecisionUseCase:
    """
    Use case: Initiate a new eligibility decision.
    
    Orchestrates:
    1. Get current policy version
    2. Create decision aggregate
    3. Persist decision
    4. Publish domain events
    """
    
    def __init__(
        self,
        decision_repo: DecisionRepository,
        event_publisher: EventPublisher,
        policy_service: PolicyService
    ):
        self._decision_repo = decision_repo
        self._event_publisher = event_publisher
        self._policy_service = policy_service
    
    def execute(self, request: CreateDecisionRequest) -> CreateDecisionResponse:
        # Get current policy version
        policy_version = self._policy_service.get_current_version()
        
        # Create aggregate
        decision = EligibilityDecision(
            decision_id=uuid4(),
            claimant_id=request.claimant_id,
            policy_version=policy_version
        )
        
        # Persist
        self._decision_repo.save(decision)
        
        # Publish events
        for event in decision.collect_events():
            self._event_publisher.publish(event)
        
        return CreateDecisionResponse(
            decision_id=decision.id,
            status=decision.status
        )
```

**Infrastructure Layer** — Concrete implementations of ports:

```python
# services/eligibility-decision/infrastructure/repositories/postgres_decision_repo.py

from uuid import UUID
from sqlalchemy.orm import Session

from ...application.ports import DecisionRepository
from ...domain.entities import EligibilityDecision
from ..models import DecisionModel


class PostgresDecisionRepository(DecisionRepository):
    """PostgreSQL implementation of DecisionRepository."""
    
    def __init__(self, session: Session):
        self._session = session
    
    def save(self, decision: EligibilityDecision) -> None:
        model = DecisionModel(
            id=decision.id,
            claimant_id=decision._claimant_id,
            status=decision.status,
            policy_version=str(decision._policy_version),
            created_at=decision._created_at
        )
        self._session.add(model)
        self._session.commit()
    
    def get_by_id(self, decision_id: UUID) -> EligibilityDecision:
        model = self._session.query(DecisionModel).filter_by(id=decision_id).first()
        if not model:
            raise DecisionNotFound(decision_id)
        return self._to_domain(model)
    
    def _to_domain(self, model: DecisionModel) -> EligibilityDecision:
        # Reconstitute aggregate from persistence model
        decision = EligibilityDecision.__new__(EligibilityDecision)
        decision._id = model.id
        decision._claimant_id = model.claimant_id
        decision._status = model.status
        # ... reconstitute full state
        return decision
```

**API Layer** — Controllers that translate HTTP to use cases:

```python
# services/eligibility-decision/api/controllers/decision_controller.py

from uuid import UUID
from fastapi import APIRouter, HTTPException

from ...application.use_cases import CreateDecisionUseCase, GetDecisionUseCase
from ..schemas import CreateDecisionRequestDTO, DecisionResponseDTO


router = APIRouter(prefix="/decisions", tags=["decisions"])


@router.post("/", response_model=DecisionResponseDTO, status_code=201)
def create_decision(
    request: CreateDecisionRequestDTO,
    use_case: CreateDecisionUseCase = Depends(get_create_decision_use_case)
):
    """Initiate a new eligibility decision."""
    result = use_case.execute(CreateDecisionRequest(
        claimant_id=request.claimant_id
    ))
    return DecisionResponseDTO(
        id=result.decision_id,
        status=result.status
    )


@router.get("/{decision_id}", response_model=DecisionResponseDTO)
def get_decision(
    decision_id: UUID,
    use_case: GetDecisionUseCase = Depends(get_get_decision_use_case)
):
    """Retrieve a decision by ID."""
    try:
        result = use_case.execute(GetDecisionRequest(decision_id=decision_id))
        return DecisionResponseDTO.from_domain(result.decision)
    except DecisionNotFound:
        raise HTTPException(status_code=404, detail="Decision not found")
```

The scaffold generation produces the complete folder structure, all layers populated with generated code, interfaces connecting the layers, and dependency injection wiring. It compiles, runs, and passes basic smoke tests out of the box.

---

## Part Four: Automated Testing Generation

Test generation is where most automated development proposals fall down. Generating code without generating tests produces an untestable system — and an untestable generated system is the worst possible outcome, because its failures are invisible until production. This paper's approach treats tests as first-class artefacts generated alongside the code they verify. The test suite grows in lockstep with the codebase, invariants are encoded before implementation rather than discovered after, and the CI pipeline validates correctness at every change. The result is that the human reviewer at each checkpoint has evidence of correctness to inspect, not just code to read and hope is right.

### Test Generation at Every Level

The pipeline doesn't just generate code—it generates comprehensive tests. Each artefact produces corresponding test artefacts, creating a safety net that validates correctness and catches regressions.

**Domain Rule Tests** encode business invariants as executable specifications:

```python
# tests/domain/test_eligibility_decision_invariants.py

import pytest
from uuid import uuid4

from domain.entities import EligibilityDecision
from domain.value_objects import PolicyVersion, ConfidenceScore
from domain.exceptions import InvariantViolation


class TestEligibilityDecisionInvariants:
    """Tests for EligibilityDecision aggregate invariants."""
    
    def test_cannot_finalise_without_minimum_evidence_confidence(self):
        """
        Business rule: A decision cannot be approved without minimum evidence confidence.
        """
        decision = EligibilityDecision(
            decision_id=uuid4(),
            claimant_id=uuid4(),
            policy_version=PolicyVersion("2026-01")
        )
        
        # Add evidence with low confidence
        decision.add_evidence(EvidenceReference(
            evidence_id=uuid4(),
            evidence_type="utility_bill",
            confidence=ConfidenceScore(0.3)
        ))
        
        # Apply a rule
        decision.apply_rule(RuleApplication(
            rule_id="residency-check",
            rule_version=PolicyVersion("2026-01"),
            outcome=True,
            confidence=ConfidenceScore(0.8),
            rationale="Address verified"
        ))
        
        # Attempt to finalise should fail
        with pytest.raises(InvariantViolation) as exc:
            decision.finalise(
                minimum_confidence=0.7,
                required_rules=["residency-check"]
            )
        
        assert "Evidence confidence" in str(exc.value)
    
    def test_cannot_finalise_without_all_required_rules(self):
        """
        Business rule: All applicable rules must be evaluated before completion.
        """
        decision = EligibilityDecision(
            decision_id=uuid4(),
            claimant_id=uuid4(),
            policy_version=PolicyVersion("2026-01")
        )
        
        # Add sufficient evidence
        decision.add_evidence(EvidenceReference(
            evidence_id=uuid4(),
            evidence_type="passport",
            confidence=ConfidenceScore(0.95)
        ))
        
        # Apply only one of two required rules
        decision.apply_rule(RuleApplication(
            rule_id="identity-check",
            rule_version=PolicyVersion("2026-01"),
            outcome=True,
            confidence=ConfidenceScore(0.9),
            rationale="Identity verified"
        ))
        
        # Attempt to finalise without residency check
        with pytest.raises(InvariantViolation) as exc:
            decision.finalise(
                minimum_confidence=0.7,
                required_rules=["identity-check", "residency-check"]
            )
        
        assert "Missing required rules" in str(exc.value)
```

**Contract Tests** verify that the service honours its published contracts:

```python
# tests/contract/test_eligibility_decision_api_provider.py

import pytest
from pact import Verifier

from api.app import create_app


class TestEligibilityDecisionAPIProvider:
    """Provider contract tests - verify we honour our published contract."""
    
    def test_provider_honours_contract(self):
        """
        Verify the eligibility-decision-service honours the
        eligibility-decision-api-v1 contract.
        """
        verifier = Verifier(
            provider="eligibility-decision-service",
            provider_base_url="http://localhost:8000"
        )
        
        # Verify against the published contract
        success, logs = verifier.verify_pacts(
            "./contracts/eligibility-decision-api-v1.json",
            provider_states_setup_url="http://localhost:8000/_pact/setup"
        )
        
        assert success, f"Contract verification failed: {logs}"
```

**Consumer Contract Tests** verify assumptions about consumed services:

```python
# tests/contract/test_customer_profile_consumer.py

import pytest
from pact import Consumer, Provider

from infrastructure.clients import CustomerProfileClient


class TestCustomerProfileConsumer:
    """Consumer contract tests - document our assumptions about customer-profile-service."""
    
    @pytest.fixture
    def pact(self):
        return Consumer("eligibility-decision-service").has_pact_with(
            Provider("customer-profile-service"),
            pact_dir="./pacts"
        )
    
    def test_get_customer_profile(self, pact):
        """
        We expect customer-profile-service to return profile data
        when we request by customer ID.
        """
        expected_profile = {
            "id": "123e4567-e89b-12d3-a456-426614174000",
            "verified_name": "Jane Smith",
            "verification_level": "HIGH",
            "place_relationships": [
                {
                    "type": "registered_address",
                    "address_line_1": "123 Main Street",
                    "postcode": "SW1A 1AA",
                    "confidence": 0.92
                }
            ]
        }
        
        pact.given(
            "a customer with ID 123e4567-e89b-12d3-a456-426614174000 exists"
        ).upon_receiving(
            "a request for customer profile"
        ).with_request(
            method="GET",
            path="/profiles/123e4567-e89b-12d3-a456-426614174000"
        ).will_respond_with(
            status=200,
            body=expected_profile
        )
        
        with pact:
            client = CustomerProfileClient(base_url=pact.uri)
            profile = client.get_profile("123e4567-e89b-12d3-a456-426614174000")
            
            assert profile.verified_name == "Jane Smith"
            assert profile.verification_level == "HIGH"
```

**Architectural Fitness Tests** verify that the code structure follows Clean Architecture:

```python
# tests/architecture/test_clean_architecture.py

import pytest
from importlib import import_module
import ast
import os


class TestCleanArchitectureLayers:
    """Verify Clean Architecture dependency rules are followed."""
    
    def test_domain_has_no_infrastructure_imports(self):
        """Domain layer must not import from infrastructure."""
        domain_files = self._get_python_files("domain/")
        
        for filepath in domain_files:
            imports = self._extract_imports(filepath)
            infrastructure_imports = [i for i in imports if "infrastructure" in i]
            
            assert not infrastructure_imports, (
                f"{filepath} imports infrastructure: {infrastructure_imports}"
            )
    
    def test_domain_has_no_framework_imports(self):
        """Domain layer must not import web frameworks."""
        domain_files = self._get_python_files("domain/")
        forbidden = ["fastapi", "flask", "django", "sqlalchemy", "boto3"]
        
        for filepath in domain_files:
            imports = self._extract_imports(filepath)
            framework_imports = [i for i in imports if any(f in i for f in forbidden)]
            
            assert not framework_imports, (
                f"{filepath} imports frameworks: {framework_imports}"
            )
    
    def test_application_does_not_import_infrastructure_implementations(self):
        """Application layer may import infrastructure interfaces, not implementations."""
        application_files = self._get_python_files("application/")
        
        for filepath in application_files:
            imports = self._extract_imports(filepath)
            # Allow: from infrastructure.ports import X
            # Forbid: from infrastructure.repositories import X
            impl_imports = [
                i for i in imports 
                if "infrastructure" in i and "ports" not in i
            ]
            
            assert not impl_imports, (
                f"{filepath} imports infrastructure implementations: {impl_imports}"
            )
    
    def _get_python_files(self, directory):
        """Get all Python files in directory."""
        files = []
        for root, _, filenames in os.walk(directory):
            for filename in filenames:
                if filename.endswith(".py"):
                    files.append(os.path.join(root, filename))
        return files
    
    def _extract_imports(self, filepath):
        """Extract import statements from a Python file."""
        with open(filepath) as f:
            tree = ast.parse(f.read())
        
        imports = []
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    imports.append(alias.name)
            elif isinstance(node, ast.ImportFrom):
                if node.module:
                    imports.append(node.module)
        return imports
```

The test suite is generated alongside the code, ensuring that every service starts with comprehensive test coverage. Tests are not an afterthought—they're a first-class output of the pipeline.

---

## Part Five: Automated Deployment

Deployment automation closes the loop. A pipeline that automates requirements-to-code but leaves deployment as manual coordination has addressed the intellectually interesting parts while leaving the operationally painful parts untouched. This stage takes generated code and delivers running services — with infrastructure configuration generated from the same service manifests that drove code generation. The same artefact that defined the service boundary also defines its resource requirements, environment variables, and health check endpoints. Nothing is configured twice; nothing drifts.

### From Code to Running Services

The final stage of the pipeline takes generated code and deploys it to a running environment. This is where infrastructure automation meets the service generation.

**Infrastructure Generation** creates deployment artefacts for each service:

```yaml
# services/eligibility-decision/helm/values.yaml
# Generated from service manifest and infrastructure templates

replicaCount: 2

image:
  repository: registry.example.com/eligibility-decision-service
  tag: "${GIT_SHA}"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8000

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: eligibility-decision-secrets
        key: database-url
  - name: EVENT_BUS_URL
    valueFrom:
      configMapKeyRef:
        name: shared-config
        key: event-bus-url

livenessProbe:
  httpGet:
    path: /health/live
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5

# Database migration job runs before deployment
migrations:
  enabled: true
  image: registry.example.com/eligibility-decision-migrations:${GIT_SHA}
```

**Pipeline Generation** creates CI/CD configuration:

```yaml
# .github/workflows/eligibility-decision-service.yaml
# Generated from service manifest and pipeline templates

name: eligibility-decision-service

on:
  push:
    paths:
      - 'services/eligibility-decision/**'
      - 'libs/eligibility-types/**'
      - 'libs/events-common/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate service scope
        run: |
          python scripts/validate_paths.py \
            --service eligibility-decision-service \
            --manifest monorepo.json
      
      - name: Validate dependencies
        run: |
          python scripts/validate_imports.py \
            --service eligibility-decision-service \
            --manifest monorepo.json

  test:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run unit tests
        run: |
          cd services/eligibility-decision
          pytest tests/unit -v --cov=domain --cov=application
      
      - name: Run integration tests
        run: |
          cd services/eligibility-decision
          docker-compose -f docker-compose.test.yaml up -d
          pytest tests/integration -v
          docker-compose -f docker-compose.test.yaml down
      
      - name: Run architecture tests
        run: |
          cd services/eligibility-decision
          pytest tests/architecture -v
      
      - name: Run provider contract tests
        run: |
          cd services/eligibility-decision
          pytest tests/contract/test_*_provider.py -v

  contract-compatibility:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run consumer contract tests for all consumers
        run: |
          # Get list of consumers from manifest
          consumers=$(jq -r '.microservices["eligibility-decision-service"].contracts.consumers[]' monorepo.json)
          for consumer in $consumers; do
            echo "Running contract tests for $consumer"
            cd services/$consumer
            pytest tests/contract/test_eligibility_decision_consumer.py -v
            cd ../..
          done

  deploy:
    needs: contract-compatibility
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and push image
        run: |
          docker build -t registry.example.com/eligibility-decision-service:${{ github.sha }} \
            services/eligibility-decision
          docker push registry.example.com/eligibility-decision-service:${{ github.sha }}
      
      - name: Deploy to staging
        run: |
          helm upgrade --install eligibility-decision-service \
            services/eligibility-decision/helm \
            --set image.tag=${{ github.sha }} \
            --namespace staging
      
      - name: Run smoke tests
        run: |
          kubectl wait --for=condition=ready pod \
            -l app=eligibility-decision-service \
            --timeout=300s \
            --namespace staging
          pytest tests/smoke -v --base-url=https://staging.example.com
      
      - name: Deploy to production
        run: |
          helm upgrade --install eligibility-decision-service \
            services/eligibility-decision/helm \
            --set image.tag=${{ github.sha }} \
            --namespace production
```

**Database Migrations** are generated from the domain model:

```python
# services/eligibility-decision/migrations/001_initial.py
# Generated from aggregate specification

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import UUID, JSONB


def upgrade():
    op.create_table(
        'eligibility_decisions',
        sa.Column('id', UUID(as_uuid=True), primary_key=True),
        sa.Column('claimant_id', UUID(as_uuid=True), nullable=False, index=True),
        sa.Column('policy_version', sa.String(20), nullable=False),
        sa.Column('status', sa.String(50), nullable=False),
        sa.Column('outcome', sa.String(50), nullable=True),
        sa.Column('confidence_score', sa.Numeric(3, 2), nullable=True),
        sa.Column('rule_applications', JSONB, nullable=False, default=[]),
        sa.Column('evidence_references', JSONB, nullable=False, default=[]),
        sa.Column('created_at', sa.DateTime, nullable=False),
        sa.Column('updated_at', sa.DateTime, nullable=False),
    )
    
    op.create_index(
        'ix_decisions_claimant_status',
        'eligibility_decisions',
        ['claimant_id', 'status']
    )


def downgrade():
    op.drop_table('eligibility_decisions')
```

The deployment pipeline is fully automated. When code is pushed, it's validated, tested, contract-checked, and deployed without human intervention—as long as all checks pass. Humans only get involved when something fails or when strategic decisions are needed.

### When Generation Fails

Automated generation isn't magic. It fails. Understanding the failure modes helps you build resilience.

**Domain model generation produces nonsense.** The requirements were ambiguous, incomplete, or contradictory. Don't try to fix the generated output—fix the input. Work with stakeholders to clarify requirements, then regenerate.

**Contract generation misses edge cases.** The domain model didn't specify error conditions or boundary cases. Add explicit requirements like "What happens when X is null?" or "What are the valid ranges for Y?" then regenerate.

**Scaffold generation produces code that doesn't compile.** The templates have a bug, or the contract used a data type the templates don't handle. Fix the template, not the generated code. Regenerate after the template is fixed.

**Tests fail after generation.** Either the business logic is wrong (rare—review the domain model) or the generated test doesn't match the generated implementation (more common—usually a template coordination bug). Diagnose which it is before fixing.

**Deployment fails.** Infrastructure configuration doesn't match the generated service's requirements. Check resource limits, environment variables, database connectivity. This is usually an infrastructure template issue.

**The Golden Rule of Generation Failures:** Never patch generated code by hand. If the output is wrong, fix the input (requirements, domain model, contracts) or the process (templates, prompts), then regenerate. Manual patches create drift between your artefacts and your code, defeating the purpose of generation.

---

## Part Six: Processing Changes

### The Evolution Workflow

Once a system is deployed, changes flow through the Evolution Phase workflow. This is where the day-to-day development happens, and it's designed to be as automated as possible.

A change request arrives—perhaps a new business requirement, a bug report, or a performance issue. The request is analysed to determine scope: which bounded contexts are affected? Which services need to change? What contracts might be impacted?

**Scope Analysis** happens automatically:

```
Change Request: "Add support for partial eligibility decisions where some 
rules pass but others are pending further evidence"

Analysis:
- Affected bounded context: eligibility-determination
- Affected service: eligibility-decision-service
- Contract impact: YES - new status value needed in API response
- Cross-service impact: YES - consumers must handle new status
- Recommended approach: 
  1. Add 'partial' to Decision status enum (backward compatible)
  2. Extend aggregate to track per-rule outcomes
  3. Update finalise() logic to allow partial completion
  4. Notify consumers of new status value
```

**Implementation** is generated within service boundaries. The LLM, operating with Service-Scoped RAG, proposes changes:

```python
# Proposed change to domain/entities/decision.py

class EligibilityDecision:
    # ... existing code ...
    
    def finalise_partial(
        self, 
        minimum_confidence: float,
        required_rules: List[str]
    ) -> None:
        """
        Finalise with partial outcome when some rules are pending.
        
        Unlike finalise(), allows completion when not all required rules
        are evaluated, resulting in 'partial' outcome.
        """
        # Invariant: minimum evidence confidence still required
        total_confidence = sum(e.confidence.value for e in self._evidence_references)
        if total_confidence < minimum_confidence:
            raise InvariantViolation(
                f"Evidence confidence {total_confidence} below minimum {minimum_confidence}"
            )
        
        # Check which rules are complete
        applied_rule_ids = {r.rule_id for r in self._rule_applications}
        pending_rules = set(required_rules) - applied_rule_ids
        
        if not pending_rules:
            # All rules complete - use standard finalise
            return self.finalise(minimum_confidence, required_rules)
        
        # Partial completion
        completed_rules_pass = all(
            r.outcome for r in self._rule_applications 
            if r.rule_id in applied_rule_ids
        )
        
        self._outcome = "partial" if completed_rules_pass else "ineligible"
        self._pending_rules = list(pending_rules)
        self._status = "partial"
        
        self._events.append(DecisionPartiallyCompleted(
            decision_id=self._id,
            claimant_id=self._claimant_id,
            outcome=self._outcome,
            pending_rules=self._pending_rules,
            completed_at=datetime.utcnow()
        ))
```

**Contract Evolution** is handled through a versioning workflow:

```yaml
# Contract change proposal
change_type: backward_compatible_addition
contract: eligibility-decision-api-v1

changes:
  - path: /components/schemas/Decision/properties/status/enum
    action: add_value
    value: "partial"
    
  - path: /components/schemas/Decision/properties
    action: add_property
    property:
      pendingRules:
        type: array
        items:
          type: string
        description: "Rules awaiting additional evidence (only present when status=partial)"

impact_analysis:
  consumers:
    - notifications-service: 
        impact: low
        action: "Should handle new status gracefully (existing code ignores unknown statuses)"
    - reporting-service:
        impact: medium  
        action: "May want to track partial decisions separately"
        
  recommendation: "Proceed with addition. Consumers can adopt new status handling at their own pace."
```

**Testing** validates the changes:

```
Running test suite for eligibility-decision-service...

Domain tests:
  ✓ test_cannot_finalise_without_minimum_evidence_confidence
  ✓ test_cannot_finalise_without_all_required_rules
  ✓ test_can_finalise_partial_with_pending_rules (NEW)
  ✓ test_partial_finalise_still_requires_minimum_evidence (NEW)

Contract tests:
  ✓ Provider honours eligibility-decision-api-v1 contract
  ✓ New 'partial' status included in valid responses

Consumer compatibility:
  ✓ notifications-service consumer tests pass
  ✓ reporting-service consumer tests pass

Architecture tests:
  ✓ Domain layer has no infrastructure imports
  ✓ Application layer uses only interface imports

All tests pass. Change is safe to deploy.
```

**Deployment** happens automatically when all checks pass. The change flows through staging, smoke tests run, and if everything passes, it goes to production. No human approval is needed for changes that stay within boundaries and pass all tests.

### Handling Breaking Changes

Not all changes are backward compatible. When a change would break consumers, the workflow shifts to coordination mode.

```
Change Request: "Rename 'claimant_id' to 'subject_id' across all APIs 
to align with new identity model"

Analysis:
- Contract impact: BREAKING - field rename breaks all consumers
- Affected consumers: 5 services across 3 bounded contexts
- Recommended approach:
  1. Add 'subject_id' as new field (keep 'claimant_id')
  2. Mark 'claimant_id' as deprecated in v1.1
  3. Release v2 with only 'subject_id'
  4. Coordinate consumer migration
  5. Remove 'claimant_id' in v2.1 after all consumers migrated

Migration timeline:
  Week 1: Release v1.1 with both fields
  Week 2-4: Consumers migrate to 'subject_id'
  Week 5: Release v2 (subject_id only)
  Week 6: Verify no v1 traffic
  Week 7: Deprecate v1
```

The pipeline generates migration artefacts: API version v1.1 with both fields, consumer migration guides, compatibility shims for the transition period, monitoring for v1 vs v2 traffic, and automated deprecation warnings.

Human approval is required for the migration plan, but execution is automated. Each consumer team receives generated migration code, tests verify compatibility with both versions, and the pipeline tracks migration progress across the organisation.

---

## Part Seven: Organisational Alignment

### Team Structure for Automated Development

The automation changes how teams work, but it doesn't eliminate the need for teams. It shifts what teams do.

**Domain Teams** own bounded contexts and the business logic within them. They don't write boilerplate—the pipeline generates it. They focus on business rules, invariants, and domain model refinement. They review generated code for correctness, not structure. They work with business stakeholders to refine requirements.

**Platform Teams** own the automation pipeline itself. They maintain RAG infrastructure, improve code generation templates, enhance CI/CD guardrails, and optimise the development experience. They're building the factory, not the products.

**Architecture Teams** make strategic decisions that the automation can't make. They define bounded context boundaries, approve cross-context contracts, govern the domain model, and resolve conflicts when automation surfaces ambiguities.

### Governance in an Automated World

Traditional governance reviews code. Automated governance reviews artefacts.

**What humans review:**
- Domain model proposals (Are the boundaries right?)
- Contract changes (Do they make business sense?)
- Breaking change migration plans (Is the timeline realistic?)
- New bounded context requests (Is this genuinely a new domain area?)

**What automation reviews:**
- Code structure (Does it follow Clean Architecture?)
- Dependency direction (Do imports point inward?)
- Contract compliance (Does the implementation match the spec?)
- Test coverage (Are invariants tested?)
- Path boundaries (Are changes within scope?)

The result is faster throughput with stronger guarantees. Humans focus on judgment calls; automation handles compliance. Neither is sufficient alone, but together they provide both speed and safety.

### Minimum Viable Automation: Where to Start

Teams starting from scratch face a genuine question: which automation stage delivers the most value if you can only implement one? The answer is **Stage 3: contract generation and validation**.

Contracts are the most durable artefacts in the pipeline. A contract specifies what a service promises to its consumers. Once generated, it governs every subsequent change: the scaffold must implement it, the tests must verify it, the consumers must rely on it. Automating contract generation ensures consistency from the start — no two developers independently draft slightly incompatible API specs for the same service.

More importantly, contracts are the artefact most likely to be neglected without automation. Domain models get documented (eventually). Scaffolds get written (manually, but they get written). Deployment pipelines get configured (painfully, but they get configured). Contracts are the thing that falls between the cracks — either they don't exist, or they exist as informal assumptions in developers' heads, or they exist as documentation that drifts from reality within weeks.

If you can only automate one thing: automate contract generation from your service manifest, and wire contract tests into CI so violations are caught immediately. The rest of the pipeline can follow as capacity allows.

### Lightweight Adoption: Starting Small

The full pipeline described in this paper requires significant platform engineering investment. If you're a smaller team or want to prove the concept before committing, here's a pragmatic path.

**Phase 0: Manual Pipeline with LLM Assistance**

Before building any automation, practice the workflow manually:

1. Use an LLM to help draft your domain model from requirements. Save the output as a markdown file—this is your domain model artefact.

2. Use an LLM to propose a service manifest based on the domain model. Save it as `monorepo.json`.

3. Use an LLM to generate OpenAPI specs for your contracts. Save them in a `contracts/` folder.

4. Use an LLM to generate scaffold code from templates. Copy the output into your repository structure.

5. Use standard CI/CD to deploy.

The checkpoints are the same—you validate at each stage. The only difference is that you're invoking the LLM manually rather than through automated pipeline stages.

**What You Learn from Phase 0:**

- Whether LLM-generated domain models capture your business correctly
- How much human iteration is needed at each stage
- What prompt templates work for your codebase
- Where the LLM struggles and needs better context

This learning costs almost nothing and takes weeks, not months. Only after you've validated that the workflow produces good results should you invest in automating it.

**Graduating to Full Automation**

Add automation where you feel the most pain:

- If generating scaffolds manually is tedious: automate Stage 4 first
- If contract consistency is a problem: automate contract generation and validation
- If deployment is error-prone: automate Stage 5
- If domain model drift is an issue: automate the domain model refresh

Each piece of automation can stand alone. You don't need the full pipeline to get value—each automated stage reduces manual work and increases consistency.

### Metrics and Continuous Improvement

The pipeline generates extensive telemetry:

**Generation Metrics:**
- Time from requirements to deployed service
- Code generation accuracy (human corrections needed)
- Test generation completeness (gaps identified post-generation)
- Contract compatibility rate (breaking changes detected)

**Operational Metrics:**
- Deployment frequency per service
- Change failure rate (deployments that require rollback)
- Mean time to recovery (when failures occur)
- Consumer impact from contract changes

**Quality Metrics:**
- Architectural violation rate (should be near zero)
- Test coverage by layer
- Domain model stability (frequency of aggregate changes)
- RAG retrieval accuracy (relevant context delivered)

These metrics drive continuous improvement. When generation accuracy drops, templates need refinement. When architectural violations increase, guardrails need strengthening. When deployment frequency varies widely across services, some teams may need additional support.

---

## Conclusion

This paper has presented a vision for end-to-end automated development: from business requirements to deployed microservices, with LLMs driving the generation and humans providing strategic guidance.

The key elements are:

**Two-Phase Model** — Design Phase for discovering boundaries with human judgment; Evolution Phase for implementing within boundaries with maximum automation.

**Artefact Pipeline** — Requirements become domain models become service manifests become contracts become code become deployments. Each stage produces machine-readable artefacts that drive the next stage.

**Generated Everything** — Domain entities, use cases, infrastructure adapters, tests, deployment manifests, CI/CD pipelines. Humans don't write boilerplate; they review generated output and refine business logic.

**Automated Enforcement** — Path validation, dependency checking, contract testing, architectural fitness functions. Violations are caught automatically, not through manual review.

**Coordinated Evolution** — Breaking changes trigger migration workflows. The pipeline generates compatibility shims, tracks migration progress, and automates deprecation.

This is not science fiction. Each component exists today. RAG infrastructure is mature. Code generation with LLMs is production-ready. CI/CD automation is well-understood. Contract testing is established practice. What this paper provides is the architecture that combines these components into a coherent, automated development pipeline.

The investment is significant: platform engineering, template development, organisational change. But the return is transformative: systems that maintain architectural integrity by construction, development velocity that scales with automation rather than headcount, and the ability to evolve complex systems safely.

Papers 1-3 established the foundations. This paper shows how to build on those foundations to create something genuinely new: a development process where humans provide intent and judgment, and machines handle everything else.

---

## Related Papers

- **Paper 1: Domain-Driven Design & Clean Architecture** — The conceptual foundations for bounded contexts, aggregates, and layer separation that this pipeline generates.

- **Paper 2: Evidence-Based Identity** — Domain-specific application demonstrating how the patterns work in a complex, real-world context.

- **Paper 3: LLM-Assisted Development** — The guardrails and RAG architecture that make automated generation safe at scale.

- **Paper 5: Getting Started** — Practical guide with technology recommendations, pseudocode examples, and troubleshooting for teams beginning implementation. See especially the "Minimum Viable" path and the Common Problems section.

- **Paper 6: Case Study** — End-to-end walkthrough applying the full pipeline to a DWP benefits administration system, with retrospective analysis of what the automation produced well and where human revision was needed.

---

*End of Paper 4*
