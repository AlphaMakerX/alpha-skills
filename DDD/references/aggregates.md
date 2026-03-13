# Aggregates

**A cluster of objects treated as a single unit for data changes. Has an Aggregate Root.**

## When to Use

```
Do these objects need to be consistent together?
├─ Yes → Same Aggregate
└─ No  → Separate Aggregates (reference by ID)
```

## Core Rules

1. **Aggregate Root** — Only the root has a global identity; external references point to the root only
2. **Consistency Boundary** — All invariants within the aggregate are always consistent
3. **Modify via Root Only** — All changes go through the root's methods
4. **One Transaction = One Aggregate** — Don't modify multiple aggregates in a single transaction
5. **Reference by ID** — Cross-aggregate references use IDs, not object references
6. **Keep Small** — Only include what needs strong (immediate) consistency

## Implementation Pattern

```typescript
// Order is the Aggregate Root
class Order extends Entity<OrderId> {
  private _items: OrderItem[] = [];    // Internal entity
  private _status: OrderStatus;

  // ALL modifications go through the root
  addItem(productId: ProductId, quantity: Quantity, price: Money): void {
    this.ensureCanModify();
    const item = OrderItem.create(productId, quantity, price);
    this._items.push(item);
    this.recalculateTotal();
  }

  removeItem(productId: ProductId): void {
    this.ensureCanModify();
    const index = this._items.findIndex(i => i.productId === productId);
    if (index === -1) throw new ItemNotFoundError(productId);
    this._items.splice(index, 1);
    this.recalculateTotal();
  }

  // Expose read-only view — never expose mutable internals
  get items(): readonly OrderItem[] {
    return [...this._items];
  }

  // Invariant enforced within aggregate boundary
  private recalculateTotal(): void {
    // Business logic stays in the aggregate
  }

  private ensureCanModify(): void {
    if (this._status !== OrderStatus.Draft) {
      throw new OrderNotModifiableError(this.id);
    }
  }
}

// OrderItem is INTERNAL to Order aggregate
// No global identity, no repository, no direct external access
class OrderItem {
  private constructor(
    readonly productId: ProductId,  // Reference by ID to Product aggregate
    private _quantity: Quantity,
    private _unitPrice: Money
  ) {}

  static create(productId: ProductId, quantity: Quantity, price: Money): OrderItem {
    return new OrderItem(productId, quantity, price);
  }

  get subtotal(): Money {
    return this._unitPrice.multiply(this._quantity);
  }
}
```

## Sizing Decision

```
Should X be in the same aggregate as Y?

Does X need to be immediately consistent with Y?
├─ Yes → Same aggregate
│        Example: OrderItem must be consistent with Order total
└─ No  → Separate aggregates, use eventual consistency
         Example: Order and Inventory — update inventory via domain event

Can they be updated independently?
├─ Yes → Separate aggregates
└─ No  → Same aggregate
```

### Too Large (Anti-Pattern)

```typescript
// ❌ BAD: Entire e-commerce system in one aggregate
class Shop extends Entity<ShopId> {
  orders: Order[];
  products: Product[];
  customers: Customer[];
  inventory: InventoryItem[];
  // Locks everything on any change!
}

// ✅ GOOD: Small, focused aggregates
class Order extends Entity<OrderId> {
  customerId: CustomerId;    // Reference by ID
  items: OrderItem[];
}
class Product extends Entity<ProductId> { ... }
class Customer extends Entity<CustomerId> { ... }
```

## Cross-Aggregate References

```typescript
// ❌ BAD: Direct object reference
class Order {
  customer: Customer;  // Holds entire Customer aggregate
}

// ✅ GOOD: Reference by ID
class Order {
  customerId: CustomerId;  // Only the ID
}
```

**Why?** Direct references:
- Create implicit coupling between aggregates
- Make it unclear which aggregate owns the data
- Prevent independent scaling and persistence

## Domain Events from Aggregates

Use **collect-then-dispatch** pattern:

```typescript
class Order extends Entity<OrderId> {
  private _events: DomainEvent[] = [];

  place(): void {
    this._status = OrderStatus.Placed;
    this._events.push(new OrderPlaced(this.id, this.total));
  }

  // Application layer calls this after saving
  pullEvents(): DomainEvent[] {
    const events = [...this._events];
    this._events = [];
    return events;
  }
}

// In Application Service:
async function placeOrder(orderId: OrderId): Promise<void> {
  const order = await orderRepository.findById(orderId);
  order.place();
  await orderRepository.save(order);

  // Dispatch events AFTER successful save
  const events = order.pullEvents();
  await eventBus.publishAll(events);
}
```

## Factory Pattern for Complex Aggregates

When creating an aggregate requires complex logic, use a **Factory**:

```typescript
class OrderFactory {
  // Factory encapsulates complex creation logic
  static createFromCart(
    orderId: OrderId,
    customerId: CustomerId,
    cart: CartItems[],
    pricingService: PricingService
  ): Order {
    const order = Order.create(orderId, customerId);

    for (const cartItem of cart) {
      const price = pricingService.getCurrentPrice(cartItem.productId);
      order.addItem(cartItem.productId, cartItem.quantity, price);
    }

    return order;
  }

  // Another factory method for a different creation path
  static reconstitute(
    id: OrderId,
    customerId: CustomerId,
    status: OrderStatus,
    items: OrderItem[]
  ): Order {
    // Used by Repository to rebuild from persistence
    return new Order(id, customerId, status, items);
  }
}
```

**When to use Factory:**
- Creation involves multiple steps or external data
- Creation requires other services (pricing, validation)
- Multiple creation paths exist (from cart, from import, from template)
- Reconstitution from persistence is non-trivial

**When NOT to use Factory:**
- Simple creation with `static create()` is sufficient
- No special logic needed

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Aggregate too large | Include only what needs immediate consistency |
| Direct object references | Use IDs for cross-aggregate references |
| Multiple aggregates in one transaction | Use domain events for eventual consistency |
| Exposing mutable internals | Return copies or readonly views |
| External code modifying internal entities | All changes go through the Aggregate Root |
| No factory for complex creation | Use Factory when creation logic is complex |
