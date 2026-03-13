# Layered Architecture

**Separate concerns into distinct layers with strict dependency rules.**

## The Four Layers

```
┌─────────────────────────────────────────────────────┐
│                UI Layer (Presentation)              │
│     Controllers, API Endpoints, CLI, Views          │
└───────────────────────┬─────────────────────────────┘
                        │ DTOs, View Models
                        ▼
┌─────────────────────────────────────────────────────┐
│                Application Layer                     │
│     Use Cases, Orchestration, Transactions           │
└───────────────────────┬─────────────────────────────┘
                        │ Domain Objects
                        ▼
┌─────────────────────────────────────────────────────┐
│                 Domain Layer                         │
│  Entities, Value Objects, Aggregates, Domain Svc     │
└───────────────────────┬─────────────────────────────┘
                        │ Repository Interfaces
                        ▼
┌─────────────────────────────────────────────────────┐
│               Infrastructure Layer                   │
│   Persistence, Messaging, External APIs, File I/O    │
└─────────────────────────────────────────────────────┘
```

## The Dependency Rule

```
UI → Application → Domain ← Infrastructure
```

- Higher layers depend on lower layers
- **Domain layer has ZERO external dependencies**
- Infrastructure **implements** domain interfaces (Dependency Inversion)
- Domain never imports from Application, UI, or Infrastructure

## What Goes Where?

### UI Layer (Presentation)

**Purpose:** Translate user intent into application commands; format responses for display.

```typescript
// REST Controller
class OrderController {
  constructor(private readonly placeOrder: PlaceOrderUseCase) {}

  async handlePlaceOrder(req: Request, res: Response): Promise<void> {
    // Parse request → Command
    const command: PlaceOrderCommand = {
      orderId: createOrderId(req.params.orderId),
      customerId: createCustomerId(req.body.customerId),
    };

    // Delegate to Application Layer
    const result = await this.placeOrder.execute(command);

    // Format response
    res.json({ orderId: result });
  }
}
```

**Contains:** Controllers, API routes, CLI handlers, view models, request/response DTOs
**Does NOT contain:** Business logic, persistence calls, domain objects in responses

### Application Layer

**Purpose:** Orchestrate use cases. Load → Execute domain logic → Persist → Publish events.

See [application-services.md](application-services.md) for full details.

**Contains:** Use Cases, Application Services, Commands, DTO mapping, transaction boundaries
**Does NOT contain:** Business rules, HTML rendering, SQL queries

### Domain Layer

**Purpose:** The heart of the system. All business rules and invariants live here.

**Contains:**
- Entities (see [entities.md](entities.md))
- Value Objects (see [value-objects.md](value-objects.md))
- Aggregates (see [aggregates.md](aggregates.md))
- Domain Services (see [domain-services.md](domain-services.md))
- Domain Events (see [domain-events.md](domain-events.md))
- Repository Interfaces (see [repositories.md](repositories.md))
- Brand Types (see [primitive-types.md](primitive-types.md))

**Does NOT contain:** Persistence logic, HTTP clients, framework code, logging

```
Domain layer has ZERO external dependencies.
It defines interfaces. Others implement them.
```

### Infrastructure Layer

**Purpose:** Implement technical capabilities that the domain needs.

```typescript
// Implements the domain-defined interface
class OrderRepositoryImpl implements OrderRepository {
  constructor(private readonly db: Database) {}

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.db.query('SELECT * FROM orders WHERE id = ?', [id]);
    return row ? OrderMapper.toDomain(row) : null;
  }

  async save(order: Order): Promise<void> {
    await this.db.upsert('orders', OrderMapper.toPersistence(order));
  }
}
```

**Contains:** Repository implementations, API clients, message queue adapters, file I/O, ORM configs
**Does NOT contain:** Business rules, use case orchestration

## Dependency Inversion in Practice

The Domain layer defines an interface; Infrastructure implements it.

```typescript
// DOMAIN defines the contract (port)
// src/domain/order/OrderRepository.ts
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// INFRASTRUCTURE implements the contract (adapter)
// src/infrastructure/order/OrderRepositoryImpl.ts
class OrderRepositoryImpl implements OrderRepository {
  constructor(private readonly db: Database) {}
  async findById(id: OrderId): Promise<Order | null> { ... }
  async save(order: Order): Promise<void> { ... }
}

// APPLICATION wires them together
// src/application/order/PlaceOrderUseCase.ts
class PlaceOrderUseCase {
  constructor(private readonly orderRepo: OrderRepository) {}  // Interface, not impl!
  async execute(command: PlaceOrderCommand): Promise<void> { ... }
}
```

**The domain NEVER knows about the database.** It only knows there's an `OrderRepository` interface.

## Directory Structure

```
src/
├── ui/                          # UI Layer
│   ├── api/
│   │   ├── OrderController.ts
│   │   └── routes.ts
│   ├── cli/
│   │   └── commands.ts
│   └── dto/
│       ├── CreateOrderRequest.ts
│       └── OrderResponse.ts
├── application/                 # Application Layer
│   ├── order/
│   │   ├── PlaceOrderUseCase.ts
│   │   ├── CancelOrderUseCase.ts
│   │   └── commands/
│   │       ├── PlaceOrderCommand.ts
│   │       └── CancelOrderCommand.ts
│   └── shared/
│       └── EventBus.ts          # Interface
├── domain/                      # Domain Layer
│   ├── order/
│   │   ├── Order.ts             # Aggregate Root
│   │   ├── OrderItem.ts         # Internal Entity
│   │   ├── OrderStatus.ts       # Value Object / Enum
│   │   ├── OrderRepository.ts   # Interface (port)
│   │   └── events/
│   │       ├── OrderPlaced.ts
│   │       └── OrderCancelled.ts
│   └── shared/
│       └── kernel/
│           ├── types.ts         # Brand Types
│           ├── Money.ts         # Shared Value Object
│           └── DomainEvent.ts   # Base interface
└── infrastructure/              # Infrastructure Layer
    ├── order/
    │   ├── OrderRepositoryImpl.ts
    │   └── OrderMapper.ts
    ├── messaging/
    │   └── EventBusImpl.ts
    └── config/
        └── database.ts
```

## Layer Violation Detection

Quick checks for violations:

```
Domain layer imports from infrastructure?
→ VIOLATION. Invert the dependency.

Application layer renders HTML or formats API responses?
→ VIOLATION. Move to UI layer.

UI layer calls repository directly?
→ VIOLATION. Go through Application layer.

Infrastructure contains business rules?
→ VIOLATION. Move to Domain layer.

Domain layer has `import axios` or `import pg`?
→ VIOLATION. Domain has zero framework dependencies.
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Domain imports infrastructure | Define interfaces in Domain; implement in Infrastructure |
| Business logic in Controller | Controllers only parse/format; delegate to Application layer |
| Application layer returns domain objects | Map to DTOs before leaving Application layer |
| "God" Application Service | One use case = one Application Service class |
| Infrastructure logic mixed with domain | Strict separation via interfaces |
