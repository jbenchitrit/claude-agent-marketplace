# Volatility Identification Guide
*Based on Righting Software by Juval Löwy*

This file is loaded during Step 2 (Elicit Axes of Volatility) of the skill.

---

## What Is Volatility?

Volatility is the *likelihood and frequency that a piece of a system will need to change*, independent of why or how it's built today.

**Critical distinction:**
- **Functional decomposition** (wrong): "We have customers, orders, and payments — let's make CustomerService, OrderService, PaymentService."
- **Volatility decomposition** (correct): "The fraud scoring rules change monthly, the payment provider could be swapped, and the notification channels are expanding — those are three axes of volatility."

Functional decomposition couples services to today's structure. Volatility decomposition designs for tomorrow's changes.

---

## The Two Dimensions of Volatility

When assessing each axis, score it on:

**1. Likelihood of change** (How probable is it this will change?)
- High: Changes multiple times per year
- Medium: Changes every 1–2 years
- Low: Rarely changes, but could

**2. Impact of change** (If it changes, how disruptive is it?)
- High: Would require touching many places in the codebase
- Medium: Localized but non-trivial
- Low: A configuration change or small code update

**Volatility Score = Likelihood × Impact**

Axes with High × High are your most critical service boundaries.

---

## Common Axes of Volatility by Domain

### Business Logic / Rules
- Pricing rules, discount calculations, promotion logic
- Risk scoring / fraud detection thresholds
- Eligibility criteria (who qualifies for what)
- Approval workflows and escalation rules
- SLA definitions and breach conditions

**Signal**: Subject matter experts own these rules and change them frequently.

---

### External Integrations
- Payment processors (Stripe, Braintree, Adyen — can be swapped)
- Identity providers (Auth0, Okta, Azure AD — can migrate)
- Communication providers (Twilio, SendGrid, Firebase — can change)
- Data enrichment APIs (credit bureaus, address validation)
- Partner APIs and webhooks

**Signal**: "We use X today, but if they raise prices / go down / we outgrow them, we'd move."

---

### Data Storage
- Primary database schema (relational → document, column additions)
- Caching strategy (in-memory, Redis, CDN)
- File/blob storage (local, S3, GCS — can migrate)
- Data warehouse / analytics layer (BigQuery, Snowflake)
- Event/message queue (Kafka, Pub/Sub, SQS)

**Signal**: The storage technology or schema is likely to evolve as scale grows or cloud strategy changes.

---

### Algorithms & Models
- ML model serving (model versions, retraining, A/B testing)
- Scoring functions that are tuned or replaced
- Optimization algorithms (routing, scheduling, matching)
- Text processing / NLP pipelines
- Feature engineering logic

**Signal**: "This logic will get better over time" or "we'll experiment with different approaches."

---

### Notification & Delivery
- Channels: email, SMS, push, in-app, Slack, webhook
- Templates and rendering logic
- Delivery preferences (user-controlled)
- Retry and failure handling policies

**Signal**: New channels are added, templates are A/B tested, or delivery logic is business-driven.

---

### Workflow & Orchestration
- Step sequences (what happens first, what happens next)
- Branching logic based on outcomes
- Compensating transactions (what to undo if a step fails)
- State machine transitions

**Signal**: "The process steps have changed 3 times in the last year" or "we're planning to add more steps."

---

### Presentation / UI
- Web, mobile, desktop surfaces
- Localization and internationalization
- Accessibility requirements
- Design system changes
- A/B test variants

**Signal**: Multiple surfaces or frequent redesigns.

---

### Identity & Authorization
- Who can do what (RBAC, ABAC)
- Auth provider (can be swapped)
- Token format and validation
- MFA policies
- SSO integrations

**Signal**: "We're moving to SSO" or "our permission model is expanding."

---

### Compliance & Regulatory
- GDPR / data privacy rules
- Financial regulations (PCI-DSS, SOX)
- Geographic rules (different rules by country)
- Audit logging requirements
- Data retention and deletion policies

**Signal**: Different rules by jurisdiction, or rules that change when regulations update.

---

### Reporting & Analytics
- KPI definitions and metrics calculations
- Dashboard layouts and export formats
- Data aggregation schedules
- Audience-specific views (ops vs. executive vs. customer)

**Signal**: Reports are frequently requested in new formats, or metric definitions change.

---

## Axis Identification Techniques

### The "What Changes?" Interview
Ask stakeholders:
1. "What part of the system has changed most in the past year?"
2. "What are you worried might change next year?"
3. "If the business pivots or a competitor emerges, what would need to be rebuilt?"
4. "What's the part of the code that everyone is afraid to touch?"

Answers to these questions map directly to high-volatility axes.

### The "Why Would This Change?" Test
For each candidate axis, ask: *What business or technical event would cause this to change?*

If the answer is "it wouldn't change unless the entire business changes," it's not a meaningful axis — it's infrastructure.

If the answer is "the business team changes this every quarter," it's a real axis.

### The "Change Together" Test
Two things belong in the same axis if and only if they always change together, for the same reason.

**Example**: The fraud score threshold and the fraud scoring algorithm are both fraud-domain volatile, but:
- If the threshold changes (business policy), the algorithm doesn't need to change.
- If the algorithm changes (new ML model), the threshold may not change.
→ These are two axes, not one.

### The "Stable Interface" Test
A well-identified axis has a stable interface to the outside world, even as its internals change.

**Example**: A NotificationService contract says `Send(recipient, message, channel)`. Internally, it could switch from SendGrid to Twilio tomorrow — the contract doesn't change. That's a correctly identified axis.

If the contract *would* have to change when the axis internals change, the boundary is drawn wrong.

---

## Common Mistakes in Axis Identification

### Mistaking technology layers for axes
```
WRONG: UILayer, ServiceLayer, DataLayer
RIGHT: FraudRulesEngine, CustomerRepository, NotificationDispatcher
```

Technology layers always change together — they are not meaningful axes.

### Mistaking entities for axes
```
WRONG: CustomerService, OrderService, ProductService
RIGHT: CustomerEligibilityEngine, OrderFulfillmentManager, ProductCatalogRepository
```

An entity like "Customer" has many concerns — some are algorithmic (eligibility), some are storage (repository), some are workflow (onboarding). Those are separate axes.

### Over-granularizing
```
WRONG: EmailService, SMSService, PushNotificationService (three services)
RIGHT: NotificationDispatcherService (one axis: notification channel mechanism)
```

If these all change for the same reason (notification channel expansion), they are one axis.

### Under-granularizing
```
WRONG: CoreBusinessLogicService (one giant engine)
RIGHT: FraudScoringEngine + EligibilityEngine + PricingEngine (three separate axes)
```

If fraud rules, eligibility criteria, and pricing all change independently, they are three axes.

---

## Volatility Matrix Template

| Axis Name | Description | Likelihood | Impact | Score | Service |
|---|---|---|---|---|---|
| FraudDetectionRules | Scoring model & threshold logic | High | High | ★★★ | FraudScoringEngine |
| NotificationChannels | Email/SMS/push delivery mechanism | High | Medium | ★★ | NotificationDispatcher |
| CustomerDataSchema | Customer PII schema & storage layer | Medium | High | ★★ | CustomerRepository |
| AuthProvider | Identity provider & token scheme | Medium | Medium | ★ | IdentityService |
| OnboardingWorkflow | Step sequence for sign-up flow | Medium | Medium | ★ | OnboardingManager |
| AuditLogging | Audit trail format & destination | Low | Low | — | AuditUtility |

**Score key**: ★★★ = critical boundary, ★★ = important, ★ = standard, — = low priority
