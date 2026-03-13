# Repositories

**Abstract persistence for aggregates. Interface in Domain, implementation in Infrastructure.**

## Core Rule

```
ONE REPOSITORY PER AGGREGATE ROOT
```

- Repositories operate on **whole aggregates**, not individual entities
- Domain layer defines the **interface** (port)
- Infrastructure layer provides the **implementation** (adapter)

## Interface Pattern (Domain Layer)

```typescript
// src/domain/order/OrderRepository.ts
interface OrderRepository {
  // Query
  findById(id: OrderId): Promise<Order | null>;
  findByCustomerId(customerId: CustomerId): Promise<Order[]>;

  // Persistence
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;

  // Identity generation
  nextId(): OrderId;
}
```

## Implementation Pattern (Infrastructure Layer)

```typescript
// src/infrastructure/order/persistence/OrderRepositoryImpl.ts
class OrderRepositoryImpl implements OrderRepository {
  constructor(private readonly db: Database) {}

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.db.query('SELECT * FROM orders WHERE id = ?', [id]);
    if (!row) return null;

    // Reconstitute full aggregate
    const items = await this.db.query('SELECT * FROM order_items WHERE order_id = ?', [id]);
    return OrderMapper.toDomain(row, items);
  }

  async save(order: Order): Promise<void> {
    // Persist entire aggregate
    await this.db.transaction(async (tx) => {
      await tx.upsert('orders', OrderMapper.toPersistence(order));
      await tx.delete('order_items', { order_id: order.id });
      for (const item of order.items) {
        await tx.insert('order_items', OrderItemMapper.toPersistence(order.id, item));
      }
    });
  }

  nextId(): OrderId {
    return createOrderId(generateUUID());
  }
}
```

## Mapper Pattern

Keep mapping logic separate from the repository:

```typescript
class OrderMapper {
  // Persistence → Domain
  static toDomain(row: OrderRow, itemRows: OrderItemRow[]): Order {
    const items = itemRows.map(OrderItemMapper.toDomain);
    return Order.reconstitute(
      createOrderId(row.id),
      createCustomerId(row.customer_id),
      row.status as OrderStatus,
      items
    );
  }

  // Domain → Persistence
  static toPersistence(order: Order): OrderRow {
    return {
      id: order.id,
      customer_id: order.customerId,
      status: order.status,
      total_amount: order.total.amount,
      total_currency: order.total.currency,
      created_at: order.createdAt,
    };
  }
}
```

## Collection-Oriented vs Persistence-Oriented

### Collection-Oriented

Repository acts like an in-memory collection. Changes are tracked and flushed.

```typescript
interface OrderRepository {
  add(order: Order): void;        // Like Array.push
  remove(order: Order): void;     // Like Array.splice
  findById(id: OrderId): Order | null;
  // Changes flushed via Unit of Work pattern
}
```

### Persistence-Oriented

Each call explicitly persists. More common in practice.

```typescript
interface OrderRepository {
  save(order: Order): Promise<void>;     // Explicit save
  delete(id: OrderId): Promise<void>;     // Explicit delete
  findById(id: OrderId): Promise<Order | null>;
}
```

**Recommendation:** Use persistence-oriented for most projects. Simpler to reason about.

## Query Methods

Keep query methods focused on aggregate root retrieval:

```typescript
interface OrderRepository {
  // ✅ Returns aggregates
  findById(id: OrderId): Promise<Order | null>;
  findByCustomerId(customerId: CustomerId): Promise<Order[]>;
  findPendingOrders(): Promise<Order[]>;

  // ❌ DON'T return internal entities
  findOrderItem(itemId: OrderItemId): Promise<OrderItem>;  // BAD

  // ❌ DON'T return raw data
  getOrderCount(): Promise<number>;  // BAD — use a read model/query service
}
```

For complex queries that don't need full aggregates, use a separate **Query Service** or **Read Model**:

```typescript
// Separate from the repository
interface OrderQueryService {
  getOrderSummaries(customerId: CustomerId): Promise<OrderSummaryDto[]>;
  getOrderCount(status: OrderStatus): Promise<number>;
}
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Repository for non-aggregate entity | Only aggregate roots get repositories |
| Returning partial aggregates | Always reconstitute the full aggregate |
| Query methods returning raw data | Use a separate Query Service for reporting |
| Domain layer depends on DB types | Domain defines interface; infrastructure implements |
| No mapper (mapping inline in repo) | Extract to dedicated Mapper class |
| Repository with business logic | Repositories are pure persistence; logic belongs in domain |
