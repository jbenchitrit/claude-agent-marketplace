---
name: volatility-design
description: >
  Apply the Righting Software (Juval Löwy) volatility-based design method to decompose a system
  into well-bounded services. Use this skill whenever the user wants to design a new system,
  decompose a monolith, identify service boundaries, design microservices, or asks about
  system architecture. Also trigger when the user mentions terms like "how to break this apart",
  "service decomposition", "where to draw boundaries", "what should be a service", or
  "volatility-based design". Always use this skill instead of ad-hoc architecture advice when
  the goal involves identifying components, services, or subsystem boundaries.
---

# Volatility-Based System Design

> Grounded in *Righting Software* by Juval Löwy.
> Reference files:
> - `references/service-archetypes.md` — Read when assigning archetypes or designing contracts
> - `references/volatility-identification.md` — Read when helping the user identify volatility axes

The core idea: **design to what changes, not to what exists today.** Every service encapsulates exactly one axis of volatility. If a business rule changes, exactly one service changes. If a data source changes, exactly one service changes. If more than one service would need to change for a single business event, the design is wrong.

---

## Workflow

### Step 1 — Understand the System Context

Ask the user to describe the system. Gather:

- **Domain**: What business problem does this solve?
- **Users**: Who are the actors/clients?
- **Key operations**: What are the top 5–10 things the system needs to *do*?
- **Known pain points**: What has broken or changed repeatedly in the past?
- **Scale**: Is this greenfield, a monolith being split, or a redesign?

Do not ask all at once — be conversational. Start with domain and key operations.

---

### Step 2 — Elicit Axes of Volatility

This is the most important step. Read `references/volatility-identification.md` before running it.

**Prompt the user with this framing:**

> "Let's think about what changes in this system — not what it does today, but what is most likely to change next month, next year, or under different business conditions. Don't think in layers (UI, DB, API). Think in *what changes together*."

Systematically probe these volatility categories:

| Category | Probing Question |
|---|---|
| **Business rules** | Which rules or calculations are likely to evolve? |
| **External integrations** | Which 3rd-party APIs, feeds, or vendors could change or be swapped? |
| **Data sources** | Which databases, storage systems, or schemas are volatile? |
| **Algorithms** | Are there computations that will be tuned, replaced, or A/B tested? |
| **Notification/delivery** | How do results get delivered — and could that mechanism change? |
| **Identity / auth** | Could the auth provider, scheme, or user model change? |
| **Pricing / billing** | Are business models, tiers, or payment providers likely to evolve? |
| **Regulatory / compliance** | Are there rules imposed by external bodies that could change? |
| **UI / presentation** | How many surfaces (web, mobile, API) exist, and how independently do they evolve? |
| **Orchestration logic** | Is there workflow that sequences steps, and how stable is that sequence? |

For each area, ask: **"How likely is this to change? What triggers the change?"**

Produce a **Volatility Inventory** — a list of named volatility axes with a brief description and a rough volatility score (High / Medium / Low).

**Example Volatility Inventory:**

```
Axis                     | Description                              | Volatility
-------------------------|------------------------------------------|-----------
Fraud detection rules    | Scoring logic for risk assessment        | High
Notification channels    | Email, SMS, push — mechanism can change  | High
Identity provider        | Auth0 today, may swap                    | Medium
Onboarding workflow      | Steps and sequencing of sign-up flow     | Medium
Customer data storage    | Postgres schema, may add graph layer     | Medium
Reporting / analytics    | Dashboards, KPIs, export formats         | Low
```

---

### Step 3 — Group and Prune Axes

Review the inventory with the user:

- **Merge** axes that change for the same reason (they are one axis)
- **Split** axes that change independently (they are two axes)
- **Eliminate** axes that are truly stable — no service needed
- **Elevate** axes with the highest volatility — they are the most critical service boundaries

**The test for a correct axis:** If this axis changes, would *any other axis* need to change too? If yes, they should be merged or one is a dependency, not a separate axis.

Aim for 5–15 axes for a typical system. More than 20 suggests over-splitting.

---

### Step 4 — Map Axes to Services

Each volatility axis becomes **exactly one service**. Name services after their axis, not their technology.

Read `references/service-archetypes.md` now and assign each service an archetype:

| Archetype | What it encapsulates | Typical volatility |
|---|---|---|
| **Client** | Entry point, user-facing interaction | High (UI changes constantly) |
| **Manager** | Workflow orchestration, step sequencing | Medium (business process changes) |
| **Engine** | Business logic, algorithms, transformations | High (rules change, models retrain) |
| **Resource** | Data persistence, external data access | Medium (schemas, sources change) |
| **Utility** | Cross-cutting: logging, auth, caching, encryption | Low-Medium (infrastructure-driven) |

**Service Map Template:**

```
Service Name            | Archetype | Encapsulated Volatility Axis
------------------------|-----------|-----------------------------
FraudScoringEngine      | Engine    | Fraud detection rules & models
NotificationService     | Utility   | Notification channels & delivery
IdentityService         | Utility   | Auth provider & user identity
OnboardingManager       | Manager   | Onboarding workflow & sequencing
CustomerRepository      | Resource  | Customer data storage & schema
AnalyticsService        | Resource  | Reporting data & export formats
OnboardingClient        | Client    | Web/mobile UI for onboarding
```

---

### Step 5 — Define Layering and Dependencies

Services must follow Löwy's layering rules. **Violations are design smells.**

```
Clients
  ↓ call
Managers
  ↓ call
Engines  ←→ Resources   (Engines may call Resources for data)
  ↓             ↓
Utilities  ←  Utilities  (called by all layers)
```

**Rules:**
- Clients ONLY call Managers (never Engines or Resources directly)
- Managers orchestrate Engines and Resources — they hold no logic themselves
- Engines implement business logic — they do not persist data (call Resources for that)
- Resources handle storage and external data — no business logic
- Utilities are horizontal — any layer can call a Utility
- **No circular dependencies between services**
- **No same-archetype calls** (Manager → Manager, Engine → Engine) — this almost always signals a missing abstraction

Produce a layered service diagram in ASCII or describe it clearly.

---

### Step 6 — Draft Service Contracts

For each service, define its contract. Read `references/service-archetypes.md` for contract rules by archetype.

For each service, output:

```
Service: [Name]
Archetype: [Type]
Volatility Axis: [What it encapsulates]

Operations:
  - OperationName(InputDTO) → OutputDTO
  - ...

Data Contracts (key DTOs):
  - InputDTO: { field: type, ... }
  - OutputDTO: { field: type, ... }

Fault Contracts:
  - FaultName: When [condition]
  - ...

Dependencies (calls):
  - Calls: [ServiceA, ServiceB]
  - Called by: [ServiceX]
```

**Contract rules (non-negotiable):**
- Operations are coarse-grained — one operation per meaningful business activity, never per field
- DTOs are dedicated transfer objects — never expose internal domain objects
- Faults are explicit and typed — not generic exceptions
- No state across calls — contracts are stateless
- Once published, contracts do not change — versioning is required for evolution

---

### Step 7 — Validate the Design

Run the **Change Test** on the design. For each likely future change, ask:

> "If [X changes], which services need to be modified?"

The answer should always be **exactly one service**. If the answer is two or more, the design has a boundary problem — go back to Step 3.

Run these validations:
- [ ] Each axis of volatility is encapsulated in exactly one service
- [ ] No service has more than one volatility axis (god service smell)
- [ ] No axis is split across multiple services (fragmentation smell)
- [ ] Layering rules are respected — no illegal call directions
- [ ] All contracts are coarse-grained — no chatty interfaces
- [ ] Fault contracts are explicit for all known failure modes

---

### Step 8 — Produce the Design Document

Output a structured design document with these sections:

1. **System Overview** — 2–3 sentence description of the system and its boundaries
2. **Volatility Matrix** — The full inventory table from Step 2
3. **Service Map** — Table of services, archetypes, and encapsulated axes
4. **Dependency Diagram** — Layered view of how services relate
5. **Service Contracts** — One contract block per service (Step 6 format)
6. **Validation Results** — The change test results from Step 7
7. **Open Design Questions** — Axes or boundaries that remain uncertain

---

## Anti-Patterns to Catch and Correct

| Anti-Pattern | Symptom | Fix |
|---|---|---|
| **Functional decomposition** | Services named after nouns/features (UserService, OrderService) rather than change axes | Reframe around volatility axes |
| **God service** | One service with 10+ operations spanning unrelated concerns | Split into multiple single-axis services |
| **Chatty interface** | Many fine-grained operations (SetName, SetEmail, SetPhone separately) | Merge into coarse operations (UpdateProfile) |
| **Layer violation** | Client calling a Resource directly | Insert a Manager or route through correct layer |
| **Leaky contract** | DTOs that expose internal DB columns or domain objects | Introduce proper DTOs |
| **Missing fault contracts** | Operations with no declared error conditions | Enumerate all known failure modes explicitly |
| **Volatility in the contract** | A contract that changes when a business rule changes | Move the volatile piece into implementation |

---

## Quick Reference: Design in One Page

```
1. What are the axes of volatility? (what changes)
2. One service per axis
3. Assign archetype: Client / Manager / Engine / Resource / Utility
4. Layer: Client → Manager → Engine ↔ Resource → Utility
5. Contract: coarse operations + typed DTOs + explicit faults
6. Validate: each change touches exactly one service
```
