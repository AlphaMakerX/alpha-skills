# Domain Events

**Records something significant that happened in the domain. Enables loose coupling between aggregates.**

## When to Use

```
Did something happen in the domain that...
├─ Other parts of the system need to know about?
├─ Needs an audit trail?
├─ Should trigger a side effect in another aggregate?
└─ If ANY → Publish a Domain Event
```

## Naming Convention

**Always use past tense** — events describe what already happened.

| ✅ Good (Past Tense) | ❌ Bad |
|----------------------|--------|
| `OrderPlaced` | `PlaceOrder` (that's a command) |
| `PaymentReceived` | `ProcessPayment` |
| `ItemShipped` | `ShipItem` |
| `CustomerRegistered` | `RegisterCustomer` |
| `InventoryAdjusted` | `AdjustInventory` |

## Event Structure

```typescript
interface DomainEvent {
  readonly eventType: string;        // Event name for routing
  readonly occurredAt: Date;         // When it happened
  readonly aggregateId: string;      // Which aggregate emitted it
}

// Concrete event
class OrderPlaced implements DomainEvent {
  readonly eventType = 'OrderPlaced';
  readonly occurredAt: Date;

  constructor(
    readonly aggregateId: string,
    readonly customerId: CustomerId,
    readonly totalAmount: Money,
    readonly itemCount: number
  ) {
    this.occurredAt = new Date();
  }
}

class OrderCancelled implements DomainEvent {
  readonly eventType = 'OrderCancelled';
  readonly occurredAt: Date;

  constructor(
    readonly aggregateId: string,
    readonly reason: CancellationReason
  ) {
    this.occurredAt = new Date();
  }
}
```

## Collect-Then-Dispatch Pattern

Aggregates **collect** events during business operations. Events are **dispatched** after persistence succeeds.

```typescript
// In the Aggregate Root
class Order extends Entity<OrderId> {
  private _events: DomainEvent[] = [];

  place(): void {
    if (this._items.length === 0) throw new EmptyOrderError(this.id);
    this._status = OrderStatus.Placed;
    // Collect — don't dispatch yet
    this._events.push(new OrderPlaced(this.id, this._customerId, this.total, this._items.length));
  }

  cancel(reason: CancellationReason): void {
    this._status = OrderStatus.Cancelled;
    this._events.push(new OrderCancelled(this.id, reason));
  }

  // Called by application layer AFTER save
  pullEvents(): DomainEvent[] {
    const events = [...this._events];
    this._events = [];
    return events;
  }
}
```

```typescript
// In Application Service
class PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly eventBus: EventBus
  ) {}

  async execute(orderId: OrderId): Promise<void> {
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw new OrderNotFoundError(orderId);

    order.place();

    // 1. Persist first
    await this.orderRepo.save(order);

    // 2. Dispatch events AFTER successful save
    const events = order.pullEvents();
    await this.eventBus.publishAll(events);
  }
}
```

**Why collect-then-dispatch?**
- Events should only be published if the state change succeeds
- If save fails, no misleading events are published
- Application layer controls the timing

## Event Handling

```typescript
// Event handler (in another bounded context or module)
class SendOrderConfirmationEmail implements EventHandler<OrderPlaced> {
  handles = 'OrderPlaced';

  async handle(event: OrderPlaced): Promise<void> {
    await this.emailService.sendConfirmation(event.customerId, event.aggregateId);
  }
}

class UpdateInventoryOnOrderPlaced implements EventHandler<OrderPlaced> {
  handles = 'OrderPlaced';

  async handle(event: OrderPlaced): Promise<void> {
    // Trigger inventory adjustment in Warehouse context
    await this.inventoryService.reserveStock(event.aggregateId);
  }
}
```

## Eventual Consistency

Domain events enable **eventual consistency** between aggregates:

```
Order Context                    Warehouse Context
═══════════════                  ═══════════════
  Order.place()
      │
      ▼
  OrderPlaced event ────────────► InventoryReserved
                                       │
                                       ▼
                                  Update stock levels
```

- **Immediate consistency**: Within one aggregate (same transaction)
- **Eventual consistency**: Across aggregates (via domain events)

## Event Bus Interface

```typescript
interface EventBus {
  publish(event: DomainEvent): Promise<void>;
  publishAll(events: DomainEvent[]): Promise<void>;
  subscribe<T extends DomainEvent>(
    eventType: string,
    handler: EventHandler<T>
  ): void;
}

interface EventHandler<T extends DomainEvent> {
  handles: string;
  handle(event: T): Promise<void>;
}
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Events named as commands (imperative) | Use past tense: `OrderPlaced`, not `PlaceOrder` |
| Dispatching before save succeeds | Always save first, then dispatch |
| Including mutable references in events | Events should contain only IDs and primitive data |
| Too much data in events | Include only what consumers need |
| Event handlers modifying the emitting aggregate | Events flow one-way; don't create cycles |
