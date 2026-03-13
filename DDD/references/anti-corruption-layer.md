# Anti-Corruption Layer (ACL)

**Isolate your domain from external models. Translate foreign data into domain language at the boundary.**

## When to Use

```
Does this code receive data from outside your bounded context?
├─ External API (REST, GraphQL, gRPC)
├─ Third-party SDK or library
├─ Database raw records
├─ CLI tool output
├─ User input (forms, URL params)
├─ Message queue / Event bus
├─ File system (config, imports)
└─ If ANY of the above → Apply ACL

Is the external model under your control?
├─ No  → Mandatory ACL (you MUST translate)
└─ Yes → Optional, but recommended for large models
```

## The Core Pattern: Pure/Impure Separation

```
External World          Boundary              Domain
═══════════════    ════════════════    ════════════════
  External DTO  →  ACL (translate)  →  Domain Model
  (impure)         (conversion)        (pure logic)
```

### Step-by-Step

```typescript
// 1. EXTERNAL DTO — matches the external system's shape
//    Lives in: infrastructure layer
interface ExternalOrderDto {
  order_id: string;
  customer_email: string;
  line_items: Array<{
    sku: string;
    qty: number;
    price_cents: number;
  }>;
  created_at: string;
}

// 2. ACL TRANSLATOR — converts to domain model
//    Lives in: infrastructure layer (or dedicated acl/ directory)
class OrderAclTranslator {
  toDomain(dto: ExternalOrderDto): Order {
    const orderId = createOrderId(dto.order_id);
    const customerId = createCustomerId(dto.customer_email);

    const items = dto.line_items.map(item =>
      OrderItem.create(
        createProductId(item.sku),
        createQuantity(item.qty),
        Money.create(item.price_cents / 100, Currency.USD)
      )
    );

    return Order.reconstitute(orderId, customerId, items);
  }

  toExternal(order: Order): ExternalOrderDto {
    return {
      order_id: order.id,
      customer_email: order.customerId,
      line_items: order.items.map(item => ({
        sku: item.productId,
        qty: item.quantity,
        price_cents: item.unitPrice.amount * 100,
      })),
      created_at: order.createdAt.toISOString(),
    };
  }
}

// 3. PURE DOMAIN LOGIC — no knowledge of external format
//    Lives in: domain layer
function calculateOrderTotal(order: Order): Money {
  return order.items.reduce(
    (sum, item) => sum.add(item.subtotal),
    Money.zero(Currency.USD)
  );
}

// 4. ORCHESTRATION — wires impure and pure together
//    Lives in: application layer
class ProcessOrderUseCase {
  constructor(
    private readonly orderApi: OrderApiClient,
    private readonly translator: OrderAclTranslator,
    private readonly orderRepository: OrderRepository
  ) {}

  async execute(externalOrderId: string): Promise<OrderId> {
    // Impure: fetch external data
    const dto = await this.orderApi.fetchOrder(externalOrderId);

    // ACL: translate to domain
    const order = this.translator.toDomain(dto);

    // Pure: apply business logic
    const total = calculateOrderTotal(order);

    // Impure: persist
    await this.orderRepository.save(order);
    return order.id;
  }
}
```

## Function Classification

| Type | Description | Example |
|------|-------------|---------|
| **Pure** | No side effects. Same input → same output. | `calculateTotal(order)` |
| **Impure** | Has side effects (I/O, network, state). | `fetchOrder(id)`, `saveOrder(order)` |
| **Semi-Pure** | Orchestrates pure + impure via dependency injection. | `ProcessOrderUseCase.execute()` |

### Why Separate?

- **Pure functions** are easy to test (no mocks needed)
- **Impure functions** are isolated to boundaries
- **Semi-pure** can be tested by injecting fakes

```typescript
// Testing semi-pure function
const fakeApi = { fetchOrder: async () => mockDto };
const useCase = new ProcessOrderUseCase(fakeApi, translator, fakeRepo);
await useCase.execute('ext-123');
```

## State Change Entry Points

Identify every point where external data enters your system:

| Source | Example | ACL Action |
|--------|---------|------------|
| **HTTP API** | REST endpoints, webhooks | Parse DTO → domain model |
| **WebSocket** | Real-time updates | Validate message → domain event |
| **gRPC / RPC** | Service-to-service calls | Proto to domain conversion |
| **CLI** | User command, shell script | Parse args → domain command |
| **User Input** | Forms, URL params | Validate + sanitize → value objects |
| **File System** | Config files, CSV imports | Parse file → domain objects |
| **Message Queue** | Kafka, RabbitMQ events | Deserialize → domain event |
| **Database** | Raw query results | Map rows → aggregate reconstitution |

## Directory Structure

```
src/
├── domain/
│   └── order/
│       ├── Order.ts              # Pure domain model
│       └── OrderRepository.ts    # Interface (port)
├── application/
│   └── order/
│       └── ProcessOrderUseCase.ts  # Semi-pure orchestration
└── infrastructure/
    └── order/
        ├── acl/
        │   └── OrderAclTranslator.ts  # Translation layer
        ├── api/
        │   └── ExternalOrderApiClient.ts  # Impure I/O
        ├── dto/
        │   └── ExternalOrderDto.ts  # External data shape
        └── persistence/
            └── OrderRepositoryImpl.ts  # Repository implementation
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Domain model mirrors external DTO shape | Design domain model from business needs, not API shape |
| Translation logic in domain layer | Keep ACL in infrastructure layer |
| No validation during translation | Validate and throw domain errors during conversion |
| Leaking DTO types into domain | Domain layer never imports DTOs |
| Inline translation (no dedicated translator) | Create explicit translator class for each external system |
