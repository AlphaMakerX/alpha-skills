---
name: ddd
description: Use when user mentions "ddd" or "DDD" for software design, when modeling business domains, when needing to extract domain concepts from requirements, or when implementing features following Domain-Driven Design patterns (entities, value objects, primitive types, aggregates, domain services, domain events).
---

# Domain-Driven Design (DDD)

Build software that speaks business language. Use proper domain types. Isolate external interactions.

## Workflow

```
┌─────────────────────────────────────────────────────┐
│  1. UNDERSTAND: Business Discovery                  │
│     Ask ONE question at a time about goal/actors    │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│  2. RESEARCH: Technical Discovery                   │
│     Scan existing domain models; reuse first        │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│  3. EXTRACT: Strategic Modeling                     │
│     Bounded Contexts + Ubiquitous Language           │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│  4. DESIGN: Tactical Planning                       │
│     Classify → Entity / VO / Aggregate / Service    │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│  5. IMPLEMENT: Code Generation                      │
│     Brand Types, layered architecture, ACL          │
└─────────────────────────────────────────────────────┘
```

## Step 1: UNDERSTAND (Business Discovery)

Ask **ONE question at a time**. Prefer multiple-choice when possible.

- What is the **business goal** of this feature?
- Who are the **main actors/users**?
- What **triggers** this feature? (user action, external event, scheduled task)
- What are the **success criteria**?
- Are there **existing similar features** to reference?

## Step 2: RESEARCH (Technical Discovery)

Before designing new concepts, search the existing codebase:

1. Search for existing domain models:
   ```
   grep -r "class.*Entity" --include="*.ts"
   grep -r "interface.*ValueObject" --include="*.ts"
   ```
2. Check for existing bounded contexts (look for `src/domain/`, `src/modules/`)
3. Identify reusable concepts: ID types, value objects, domain events

**Priority: Reuse existing concepts over creating new ones.**

## Step 3: EXTRACT (Strategic Modeling)

### Ubiquitous Language (通用语言)

Create a glossary table. See [ubiquitous-language.md](references/ubiquitous-language.md)

| Term (English) | Term (中文) | Definition | Code Reference |
|----------------|-------------|------------|----------------|
| _Example_ | _示例_ | _描述_ | _`ClassName`_ |

### Bounded Context Identification

See [bounded-contexts.md](references/bounded-contexts.md) for full guide.

Key question: Does this concept have **different meanings** in different parts of the system?

## Step 4: DESIGN (Tactical Planning)

Present a planning table before writing code:

| Pattern | Name | Responsibility | Key Properties |
|---------|------|----------------|----------------|
| **Entity** | _Order_ | _Represents a customer order_ | _id, status, items_ |
| **Value Object** | _Money_ | _Immutable monetary amount_ | _amount, currency_ |
| **Aggregate Root** | _Order_ | _Consistency boundary_ | _contains LineItems_ |
| **Domain Service** | _PricingService_ | _Calculate order totals_ | _stateless_ |
| **Domain Event** | _OrderPlaced_ | _Published when order created_ | _orderId, timestamp_ |

### Decision Trees

```
Does it have a unique lifecycle identity?
├─ Yes → Entity          → See [entities.md](references/entities.md)
└─ No  → Value Object    → See [value-objects.md](references/value-objects.md)

Does it span multiple entities/aggregates?
├─ Yes → Domain Service  → See [domain-services.md](references/domain-services.md)
└─ No  → Put in Entity   → Avoid anemic models

Is the creation logic complex?
├─ Yes → Factory pattern  → See [aggregates.md](references/aggregates.md)
└─ No  → Simple constructor

Does it receive external data?
├─ Yes → Anti-Corruption Layer → See [anti-corruption-layer.md](references/anti-corruption-layer.md)
└─ No  → Pure domain logic
```

## Step 5: IMPLEMENT (Code Generation)

### Architecture

Follow the 4-layer model. See [layered-architecture.md](references/layered-architecture.md)

```
UI → Application → Domain ← Infrastructure
```

- **Domain layer has ZERO external dependencies**
- Use [application-services.md](references/application-services.md) for orchestration
- Use [repositories.md](references/repositories.md) for persistence abstraction

### Type Safety

Use **Brand Types** for all domain primitives. See [primitive-types.md](references/primitive-types.md)

```typescript
type OrderId = string & { readonly __brand: 'OrderId' };
```

### Aggregate boundaries

See [aggregates.md](references/aggregates.md) — keep small, reference by ID, one transaction = one aggregate.

### Domain Events

See [domain-events.md](references/domain-events.md) — past tense naming, collect-then-dispatch.

### Post-Implementation

Run through [verification-checklist.md](references/verification-checklist.md).

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Anemic domain model | Put behavior in entities, not services |
| Too many aggregates | Start with fewer, larger aggregates |
| Ignoring existing code | Always search project first |
| Technical naming | Use business language (Ubiquitous Language) |
| Primitives in domain | Use Brand Types for all domain IDs and values |
| Business logic in Application Service | Application Service orchestrates only; logic belongs in Domain |
| Domain layer imports infrastructure | Invert the dependency; Domain defines interfaces |

## Reference Index

| File | Content |
|------|---------|
| [primitive-types.md](references/primitive-types.md) | Brand Types, domain primitives |
| [value-objects.md](references/value-objects.md) | Immutable value comparison |
| [entities.md](references/entities.md) | Identity lifecycle, rich behavior |
| [aggregates.md](references/aggregates.md) | Consistency boundaries, Factory |
| [bounded-contexts.md](references/bounded-contexts.md) | Strategic boundary identification |
| [anti-corruption-layer.md](references/anti-corruption-layer.md) | External data, pure/impure separation |
| [domain-events.md](references/domain-events.md) | State change recording |
| [domain-services.md](references/domain-services.md) | Stateless cross-aggregate logic |
| [repositories.md](references/repositories.md) | Persistence abstraction |
| [application-services.md](references/application-services.md) | Use Case orchestration |
| [layered-architecture.md](references/layered-architecture.md) | 4-layer dependency model |
| [ubiquitous-language.md](references/ubiquitous-language.md) | Business terminology conventions |
| [verification-checklist.md](references/verification-checklist.md) | Post-implementation validation |
