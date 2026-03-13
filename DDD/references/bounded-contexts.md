# Bounded Contexts

**An explicit boundary within which a domain model applies. Same term can have different meanings in different contexts.**

## When to Define a Bounded Context

```
Does the same concept mean different things in different parts of the system?
├─ Yes → Separate bounded contexts
│        Example: "Product" in Sales (price, discount) vs Warehouse (location, quantity)
└─ No  → Likely same context

Could different teams independently own and evolve this concept?
├─ Yes → Separate bounded contexts
└─ No  → Likely same context

Does it require different data models or storage?
├─ Yes → Separate bounded contexts
└─ No  → Likely same context
```

## Visualization

```
┌─────────────────────────┐  ┌─────────────────────────┐
│   Sales Context         │  │   Warehouse Context     │
│  ┌───────────────┐      │  │  ┌───────────────┐      │
│  │   Product     │      │  │  │   Product     │      │
│  │ - price       │      │  │  │ - location    │      │
│  │ - discount    │      │  │  │ - quantity    │      │
│  │ - margin      │      │  │  │ - shelfLife   │      │
│  └───────────────┘      │  │  └───────────────┘      │
│                         │  │                         │
│  ┌───────────────┐      │  │  ┌───────────────┐      │
│  │   Customer    │      │  │  │   Shipment    │      │
│  │ - creditLimit │      │  │  │ - tracking    │      │
│  │ - loyaltyTier │      │  │  │ - carrier     │      │
│  └───────────────┘      │  │  └───────────────┘      │
└─────────────────────────┘  └─────────────────────────┘
```

**Key insight:** `Product` in Sales and `Product` in Warehouse are **different classes** with different properties and behaviors, even though the business calls them both "Product."

## Context Mapping Patterns

Relationships between bounded contexts:

### Shared Kernel

Two contexts share a common subset of the model. Changes require agreement.

```
┌──────────┐   Shared   ┌──────────┐
│ Context A │◄──Kernel──►│ Context B │
└──────────┘             └──────────┘
```

- Use when: Two teams closely collaborate on shared concepts
- Risk: Tight coupling; changes affect both contexts
- Example: Shared `Money` value object used by Sales and Billing

### Customer-Supplier

Upstream context provides data; downstream context consumes it.

```
┌──────────┐             ┌──────────┐
│ Upstream  │────────────►│ Downstream│
│ (Supplier)│             │ (Customer)│
└──────────┘             └──────────┘
```

- Use when: One context depends on another's output
- Downstream has some influence on upstream's model
- Example: Order context (upstream) → Shipping context (downstream)

### Anti-Corruption Layer (ACL)

Downstream isolates itself from upstream's model via a translation layer.

```
┌──────────┐             ┌─────┐  ┌──────────┐
│ External  │────────────►│ ACL │──►│ My       │
│ System    │             │     │  │ Context  │
└──────────┘             └─────┘  └──────────┘
```

- Use when: Integrating with legacy systems, external APIs, or unreliable upstream
- The ACL translates external models into your domain language
- See [anti-corruption-layer.md](anti-corruption-layer.md) for implementation details

### Open Host Service

Context exposes a well-defined public API for others to consume.

```
                           ┌──────────┐
              ┌───────────►│ Consumer A│
┌──────────┐  │            └──────────┘
│ Open Host │──┤           ┌──────────┐
│ Service   │  ├──────────►│ Consumer B│
└──────────┘  │            └──────────┘
              └───────────►│ Consumer C│
                           └──────────┘
```

- Use when: Multiple consumers need access to your context
- Publish a stable API with versioning
- Example: Authentication service consumed by all other contexts

### Conformist

Downstream adopts upstream's model as-is. No translation.

- Use when: Upstream has no incentive to change, and translation cost is too high
- Risk: Downstream fully coupled to upstream's model decisions
- Example: Using a third-party SaaS product's data model directly

## Directory Structure per Context

```
src/
├── domain/
│   ├── sales/              # Sales bounded context
│   │   ├── entities/
│   │   ├── value-objects/
│   │   ├── aggregates/
│   │   ├── services/
│   │   ├── events/
│   │   └── repositories/
│   ├── warehouse/           # Warehouse bounded context
│   │   ├── entities/
│   │   ├── ...
│   └── shared/
│       └── kernel/          # Shared Kernel (if used)
│           ├── Money.ts
│           └── types.ts
├── application/
│   ├── sales/
│   └── warehouse/
└── infrastructure/
    ├── sales/
    └── warehouse/
```

## Identification Checklist

When analyzing a system to find bounded contexts, ask:

- [ ] Are there terms used differently by different stakeholders?
- [ ] Are there natural team boundaries?
- [ ] What concepts change together vs independently?
- [ ] Where do you see data model conflicts?
- [ ] What could be deployed independently?

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| One giant context for everything | Split when concepts have different meanings or lifecycles |
| Sharing entity classes across contexts | Each context has its own model, even for "same" concept |
| No explicit boundary | Use directory structure and module boundaries |
| Tight coupling between contexts | Use events or ACL for cross-context communication |
| Premature splitting | Start with larger contexts, refine as understanding grows |
