# Domain Services

**Stateless operations that don't naturally belong to any single entity or value object.**

## When to Use

```
Does this logic...
├─ Span multiple entities or aggregates?
│   └─ Yes → Domain Service
├─ Not naturally belong to any entity?
│   └─ Yes → Domain Service
├─ Need no state of its own? (pure calculation/coordination)
│   └─ Yes → Domain Service
└─ Otherwise → Put it IN the entity (avoid anemic models)
```

## Domain Service vs Entity Behavior

```typescript
// ❌ BAD — Logic belongs in Order entity
class OrderService {
  calculateTotal(order: Order): Money {
    return order.items.reduce((sum, item) => sum.add(item.subtotal), Money.zero());
  }
}

// ✅ GOOD — Logic in the entity
class Order extends Entity<OrderId> {
  get total(): Money {
    return this._items.reduce((sum, item) => sum.add(item.subtotal), Money.zero());
  }
}
```

```typescript
// ✅ GOOD — Logic spans multiple aggregates → Domain Service
class PricingService {
  calculateDiscount(order: Order, customer: Customer): Money {
    const loyaltyDiscount = this.getLoyaltyDiscount(customer.loyaltyTier);
    const volumeDiscount = this.getVolumeDiscount(order.total);
    return Money.max(loyaltyDiscount, volumeDiscount);
  }

  private getLoyaltyDiscount(tier: LoyaltyTier): Money { ... }
  private getVolumeDiscount(total: Money): Money { ... }
}
```

## Domain Service vs Application Service

| | Domain Service | Application Service |
|---|---|---|
| **Layer** | Domain | Application |
| **Contains** | Business rules | Orchestration (workflow) |
| **Dependencies** | Domain objects only | Repositories, event bus, external services |
| **State** | Stateless | Stateless |
| **Knowledge** | Business logic | Use case coordination |
| **Example** | `PricingService.calculateDiscount()` | `PlaceOrderUseCase.execute()` |

```typescript
// DOMAIN Service — pure business logic
class TransferService {
  transfer(from: Account, to: Account, amount: Money): void {
    from.debit(amount);
    to.credit(amount);
  }
}

// APPLICATION Service — orchestration
class TransferMoneyUseCase {
  constructor(
    private readonly accountRepo: AccountRepository,
    private readonly transferService: TransferService,
    private readonly eventBus: EventBus
  ) {}

  async execute(fromId: AccountId, toId: AccountId, amount: Money): Promise<void> {
    const from = await this.accountRepo.findById(fromId);
    const to = await this.accountRepo.findById(toId);

    // Delegate business logic to domain service
    this.transferService.transfer(from, to, amount);

    await this.accountRepo.save(from);
    await this.accountRepo.save(to);
    await this.eventBus.publish(new MoneyTransferred(fromId, toId, amount));
  }
}
```

## Properties of a Good Domain Service

1. **Stateless** — No internal state; all data comes from parameters
2. **Named in business terms** — `PricingService`, `TransferService`, not `DataProcessor`
3. **Domain layer only** — No infrastructure dependencies (no HTTP, no DB)
4. **Pure functions** — Easy to test without mocks
5. **Clear responsibility** — One focused area of business logic

## Implementation Pattern

```typescript
// Domain Service
class ShippingCostCalculator {
  calculate(
    order: Order,
    destination: Address,
    shippingRules: ShippingRule[]
  ): Money {
    const weight = order.totalWeight;
    const zone = this.determineZone(destination);
    const applicableRule = shippingRules.find(r => r.appliesTo(zone, weight));

    if (!applicableRule) {
      throw new NoShippingRuleError(zone, weight);
    }

    return applicableRule.calculateCost(weight);
  }

  private determineZone(destination: Address): ShippingZone {
    // Business logic for determining shipping zone
    ...
  }
}
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Logic belongs in an entity | If it operates on one entity's data, put it in the entity |
| Infrastructure dependencies | Domain Services must NOT depend on repositories or APIs |
| Stateful service | Domain Services are stateless — no instance fields |
| Named with technical jargon | Use business terms: `PricingService`, not `OrderProcessor` |
| Confused with Application Service | Domain Service = business logic; Application Service = orchestration |
