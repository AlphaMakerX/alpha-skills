# Entities

**Has unique identity. Identity persists throughout lifecycle. Equality by ID.**

## When to Use

```
Does the concept...
├─ Have a unique lifecycle identity? (tracked by ID, not by attributes)
├─ Change state over time? (status transitions, attribute updates)
├─ Need to be distinguishable from others with same attributes?
└─ Have behavior that protects invariants?

If YES → Entity
If NO identity needed → Value Object (see value-objects.md)
```

## Core Properties

1. **Identity** — Defined by a unique ID (Brand Type), not by attributes
2. **Lifecycle** — Created, modified, possibly deleted over time
3. **Equality by ID** — Two entities with same ID are the same, regardless of attribute values
4. **Behavioral** — Contains business logic, protects its own invariants

## Base Entity Pattern

```typescript
abstract class Entity<TId> {
  protected constructor(protected readonly _id: TId) {}

  get id(): TId {
    return this._id;
  }

  // Identity-based equality
  equals(other: Entity<TId>): boolean {
    if (other === null || other === undefined) return false;
    return this._id === other._id;
  }
}
```

## Implementation Example

```typescript
class Order extends Entity<OrderId> {
  private _status: OrderStatus;
  private _items: OrderItem[];
  private _placedAt: Date | null;
  private _events: DomainEvent[] = [];

  private constructor(
    id: OrderId,
    private readonly _customerId: CustomerId,
    status: OrderStatus,
    items: OrderItem[]
  ) {
    super(id);
    this._status = status;
    this._items = items;
    this._placedAt = null;
  }

  // Factory method
  static create(id: OrderId, customerId: CustomerId): Order {
    const order = new Order(id, customerId, OrderStatus.Draft, []);
    order._events.push(new OrderCreated(id, customerId));
    return order;
  }

  // Business behavior — NOT a setter
  addItem(productId: ProductId, quantity: Quantity, price: Money): void {
    this.ensureCanModify();
    const existing = this._items.find(i => i.productId === productId);
    if (existing) {
      throw new DuplicateItemError(productId);
    }
    this._items.push(OrderItem.create(productId, quantity, price));
    this._events.push(new OrderItemAdded(this.id, productId, quantity));
  }

  // State transition with invariant protection
  place(): void {
    this.ensureCanModify();
    if (this._items.length === 0) {
      throw new EmptyOrderError(this.id);
    }
    this._status = OrderStatus.Placed;
    this._placedAt = new Date();
    this._events.push(new OrderPlaced(this.id, this.total));
  }

  cancel(reason: CancellationReason): void {
    if (this._status === OrderStatus.Shipped) {
      throw new OrderAlreadyShippedError(this.id);
    }
    this._status = OrderStatus.Cancelled;
    this._events.push(new OrderCancelled(this.id, reason));
  }

  // Computed property — NOT stored, derived from state
  get total(): Money {
    return this._items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero(Currency.USD)
    );
  }

  // Read-only access to internal state
  get items(): readonly OrderItem[] {
    return [...this._items];
  }

  get status(): OrderStatus { return this._status; }

  // Invariant guard
  private ensureCanModify(): void {
    if (this._status !== OrderStatus.Draft) {
      throw new OrderNotModifiableError(this.id, this._status);
    }
  }

  // Collect domain events for dispatching
  pullEvents(): DomainEvent[] {
    const events = [...this._events];
    this._events = [];
    return events;
  }
}
```

## Rich vs Anemic Model

```typescript
// ❌ ANEMIC — Entity is just a data bag
class Order {
  id: OrderId;
  status: OrderStatus;
  items: OrderItem[];
}

// Business logic lives in a service (BAD)
class OrderService {
  placeOrder(order: Order): void {
    if (order.items.length === 0) throw new Error('...');
    order.status = OrderStatus.Placed;  // Direct mutation!
  }
}

// ✅ RICH — Entity owns its behavior
class Order extends Entity<OrderId> {
  place(): void {
    if (this._items.length === 0) throw new EmptyOrderError(this.id);
    this._status = OrderStatus.Placed;
  }
}
```

**The domain model owns its rules. Services only orchestrate.**

## State Transitions

Use an enum or union type to model explicit states:

```typescript
enum OrderStatus {
  Draft = 'Draft',
  Placed = 'Placed',
  Confirmed = 'Confirmed',
  Shipped = 'Shipped',
  Delivered = 'Delivered',
  Cancelled = 'Cancelled',
}

// Valid transitions
// Draft → Placed → Confirmed → Shipped → Delivered
//   ↓         ↓         ↓
// Cancelled  Cancelled  Cancelled
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Public setters (`set status()`) | Use named methods: `place()`, `cancel()` |
| Anemic model (data bag) | Put behavior IN the entity |
| No invariant protection | Guard every state transition |
| Primitive ID (`id: string`) | Use Brand Type (`id: OrderId`) |
| Mutable internal collections | Return copies: `[...this._items]` |
| Constructor does too much | Use `static create()` factory + private constructor |
