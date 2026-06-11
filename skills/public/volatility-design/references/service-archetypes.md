# Service Archetypes Reference
*Based on Righting Software by Juval Löwy*

This file is loaded when assigning archetypes or designing contracts in Step 4–6 of the skill.

---

## The Five Archetypes

### 1. Client

**Role**: The entry point into the system. Translates user or external system interactions into service calls.

**Encapsulates**: Presentation logic, user interaction patterns, UI state.

**Volatility**: Typically the most volatile layer — UI technologies, UX patterns, and client platforms change constantly.

**Contract characteristics:**
- Has no service operations of its own (it is the consumer, not the provider)
- Calls only Managers — never Engines, Resources, or other Clients directly
- Maintains no business logic — delegates everything downstream
- May maintain UI state (session, navigation) but not business state

**Examples**: Web app, mobile app, CLI tool, partner API gateway, webhook consumer

**Smell indicators:**
- A Client that calls a Resource directly (layer violation)
- A Client that contains conditional business logic
- Multiple Clients with duplicated orchestration logic (should be in a Manager)

---

### 2. Manager

**Role**: Orchestrates workflows. Sequences calls to Engines and Resources to fulfill a business activity. Holds the *flow*, not the *logic*.

**Encapsulates**: Business process sequences, workflow steps, coordination of downstream services.

**Volatility**: Medium — workflows change when the business process changes (not when business rules change).

**Contract characteristics:**
- Operations are coarse-grained and represent a full business activity (e.g., `ProcessOnboarding`, `HandleClaim`, `FulfillOrder`)
- Takes a rich input DTO and returns a rich output DTO
- Declares fault contracts for each step that can fail
- Does NOT contain business logic — delegates to Engines
- Does NOT persist data — delegates to Resources

**Call rules:**
- Called by: Clients, other Managers (rarely, and with care)
- Calls: Engines, Resources, Utilities
- Never calls another Manager in the same workflow chain (this creates hidden coupling)

**Examples**: OnboardingManager, PaymentWorkflowManager, AccountResolutionManager

**Smell indicators:**
- A Manager with conditional logic based on business rules (logic belongs in an Engine)
- A Manager that also persists data (persistence belongs in a Resource)
- A Manager with 20+ operations (likely two workflows conflated into one)

---

### 3. Engine

**Role**: Implements business logic, algorithms, transformations, and computations. The "brain" of the system.

**Encapsulates**: Business rules, scoring logic, calculation algorithms, ML models, decision logic.

**Volatility**: High — business rules evolve, regulations change, models retrain, formulas update.

**Contract characteristics:**
- Operations represent a computation or transformation (e.g., `ScoreRisk`, `CalculatePremium`, `ClassifyTransaction`)
- Input: data to process. Output: result of computation. No side effects (except via explicit Resource calls)
- Stateless — the same inputs produce the same outputs
- Fault contracts cover invalid inputs and computation failures

**Call rules:**
- Called by: Managers
- Calls: Resources (to fetch data needed for computation), Utilities
- Does NOT call other Engines directly (if needed, route through a Manager)

**Examples**: FraudScoringEngine, PremiumCalculationEngine, RiskClassificationEngine, AnomalyDetectionEngine

**Smell indicators:**
- An Engine that persists results directly (should return to Manager, which calls Resource)
- An Engine that contains workflow logic ("first call A, then if result > threshold, call B")
- An Engine whose logic is hardcoded and can't be reconfigured (suggests missing parameterization)

---

### 4. Resource

**Role**: Manages access to persistent state — databases, file systems, external data APIs, message queues.

**Encapsulates**: Storage technology, schema structure, external data provider details.

**Volatility**: Medium — storage technologies change (DB migrations, cloud moves), schemas evolve, external APIs get versioned.

**Contract characteristics:**
- Operations are data-centric: read/write/query patterns
- Abstracts the storage technology entirely — callers never know if it's Postgres, BigQuery, S3, or an API
- Returns typed DTOs — never raw DB rows or deserialized JSON blobs
- Fault contracts cover not-found, connection failures, and write conflicts

**Call rules:**
- Called by: Managers, Engines
- Calls: Utilities (for caching, encryption, audit logging)
- Does NOT contain business logic — never validates business rules, only data integrity

**Examples**: CustomerRepository, TransactionDataStore, MLFeatureStore, NotificationHistoryRepository

**Smell indicators:**
- A Resource that filters or transforms data based on business rules (logic belongs in an Engine)
- A Resource with an operation like `GetActiveCustomersWithHighRiskAndPendingPayments` (this is a query that should be expressed at the Manager/Engine layer)
- Callers that know the Resource's underlying storage technology

---

### 5. Utility

**Role**: Provides cross-cutting infrastructure capabilities used by multiple services across all layers.

**Encapsulates**: Infrastructure concerns that aren't specific to any one business domain.

**Volatility**: Low to Medium — driven by infrastructure changes (new logging platform, key rotation, caching strategy).

**Contract characteristics:**
- Operations are narrow, reusable, and domain-agnostic
- E.g., `Log(event)`, `Encrypt(data)`, `Cache(key, value, ttl)`, `Notify(recipient, message, channel)`
- No business logic — purely functional/infrastructural
- Fault contracts cover infrastructure failures

**Call rules:**
- Called by: any layer (Client, Manager, Engine, Resource)
- Calls: other Utilities (e.g., a Notification Utility might call an Encryption Utility)
- Should not call Managers, Engines, or Resources — that would invert the abstraction

**Examples**: AuditLogger, EncryptionService, NotificationDispatcher, CacheService, AuthService, FeatureFlagService

**Smell indicators:**
- A Utility that contains business logic (then it's an Engine, not a Utility)
- Business services calling infrastructure details directly instead of through a Utility

---

## Archetype Assignment Decision Tree

```
Does it present or receive input from a user/external system?
  → YES: Client

Does it sequence steps and orchestrate other services?
  → YES: Manager

Does it compute, score, classify, or apply business rules?
  → YES: Engine

Does it store, retrieve, or access persistent/external data?
  → YES: Resource

Is it a cross-cutting concern used by many services?
  → YES: Utility
```

---

## Contract Design Rules (All Archetypes)

### Operation Design
- **Coarse-grained**: one operation per business activity, not per field or step
- **Avoid chatty interfaces**: `UpdateProfile(ProfileDTO)` not `SetName()` + `SetEmail()` + `SetPhone()`
- **Client-agnostic**: contract shape should reflect the service's responsibility, not the caller's convenience

### DTO Design
- Dedicated DTOs per operation — never reuse internal domain objects in contracts
- DTOs are flat where possible; avoid deep object graphs
- DTOs must be serializable (JSON-safe types: strings, numbers, booleans, arrays, nested DTOs)
- Separate input DTOs from output DTOs (they evolve independently)

### Fault Contract Design
- Every operation declares all known failure modes
- Faults are typed (e.g., `CustomerNotFoundFault`, `ScoreUnavailableFault`) not generic
- Faults describe the *business condition*, not the *technical exception*
- Callers must handle all declared faults explicitly

### Versioning Rule
- Once a contract is published, it does not break backward compatibility
- Additive changes (new optional fields) are allowed
- Breaking changes require a new version: `V2.FraudScoringEngine.ScoreRisk()`
- Callers are never forced to upgrade unless they need new capabilities
