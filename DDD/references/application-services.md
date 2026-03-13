# Application Services (Use Cases)

**Orchestrate domain objects to fulfill a use case. No business logic here — only coordination.**

## When to Use

```
Does this code...
├─ Load aggregates from repositories?
├─ Call domain methods to execute business logic?
├─ Save results back to repositories?
├─ Dispatch domain events?
├─ Coordinate transactions?
└─ If YES to these → Application Service

Does this code contain business RULES?
└─ Yes → It belongs in Entity or Domain Service, NOT here
```

## The Standard Pattern

```
Load → Execute → Persist → Publish
```

```typescript
class PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly customerRepo: CustomerRepository,
    private readonly pricingService: PricingService,  // Domain Service
    private readonly eventBus: EventBus
  ) {}

  async execute(command: PlaceOrderCommand): Promise<OrderId> {
    // 1. LOAD — Get aggregates from repositories
    const order = await this.orderRepo.findById(command.orderId);
    if (!order) throw new OrderNotFoundError(command.orderId);

    const customer = await this.customerRepo.findById(command.customerId);
    if (!customer) throw new CustomerNotFoundError(command.customerId);

    // 2. EXECUTE — Delegate to domain objects (NO business logic here)
    const discount = this.pricingService.calculateDiscount(order, customer);
    order.applyDiscount(discount);
    order.place();

    // 3. PERSIST — Save modified aggregates
    await this.orderRepo.save(order);

    // 4. PUBLISH — Dispatch domain events
    const events = order.pullEvents();
    await this.eventBus.publishAll(events);

    return order.id;
  }
}
```

## Application Service vs Domain Service

This is the **#1 most confusing distinction** in DDD:

| | Application Service | Domain Service |
|---|---|---|
| **Purpose** | Orchestrate a use case | Encapsulate business rules |
| **Contains** | Loading, saving, event dispatch | Calculations, validations, rules |
| **Dependencies** | Repos, Event Bus, external services | Domain objects only |
| **Layer** | Application layer | Domain layer |
| **Testability** | Needs mocks for repos/event bus | Pure functions, no mocks needed |

### The Litmus Test

> **"If I remove all infrastructure (DB, HTTP, events), does this logic still make business sense?"**
> - **Yes** → Domain Service
> - **No** → Application Service

```typescript
// DOMAIN Service — pure business logic
// "Calculate a discount" makes sense without any infrastructure
class PricingService {
  calculateDiscount(order: Order, customer: Customer): Money { ... }
}

// APPLICATION Service — orchestration
// "Load order, apply discount, save, publish" requires infrastructure
class ApplyDiscountUseCase {
  async execute(orderId: OrderId): Promise<void> {
    const order = await this.orderRepo.findById(orderId);
    const customer = await this.customerRepo.findById(order.customerId);
    const discount = this.pricingService.calculateDiscount(order, customer);
    order.applyDiscount(discount);
    await this.orderRepo.save(order);
  }
}
```

## Command Pattern (Input)

Use explicit command objects for use case inputs:

```typescript
// Command — describes the intent
interface PlaceOrderCommand {
  readonly orderId: OrderId;
  readonly customerId: CustomerId;
}

interface CancelOrderCommand {
  readonly orderId: OrderId;
  readonly reason: CancellationReason;
}

// Use cases accept commands
class PlaceOrderUseCase {
  async execute(command: PlaceOrderCommand): Promise<OrderId> { ... }
}

class CancelOrderUseCase {
  async execute(command: CancelOrderCommand): Promise<void> { ... }
}
```

## DTO Mapping at This Layer

Application Services are responsible for converting between DTOs and domain objects:

```typescript
class CreateOrderUseCase {
  async execute(dto: CreateOrderDto): Promise<OrderResponseDto> {
    // DTO → Domain
    const customerId = createCustomerId(dto.customerId);
    const orderId = this.orderRepo.nextId();
    const order = Order.create(orderId, customerId);

    for (const item of dto.items) {
      order.addItem(
        createProductId(item.productId),
        createQuantity(item.quantity),
        Money.create(item.price, Currency.USD)
      );
    }

    await this.orderRepo.save(order);

    // Domain → DTO
    return OrderResponseDto.fromDomain(order);
  }
}
```

## Transaction Management

Application Services define transaction boundaries:

```typescript
class TransferMoneyUseCase {
  async execute(command: TransferMoneyCommand): Promise<void> {
    // Transaction wraps the entire use case
    await this.unitOfWork.execute(async () => {
      const from = await this.accountRepo.findById(command.fromAccountId);
      const to = await this.accountRepo.findById(command.toAccountId);

      // Domain logic
      this.transferService.transfer(from, to, command.amount);

      // Both saves within one transaction
      await this.accountRepo.save(from);
      await this.accountRepo.save(to);
    });
  }
}
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Business logic in Application Service | Move to Entity or Domain Service |
| Directly manipulating entity fields | Call entity's business methods |
| No explicit command/input object | Use Command pattern for clarity |
| Returning domain objects to UI layer | Map to DTO before returning |
| Application Service depends on domain service that needs infrastructure | Domain Services must be pure; inject via constructor |
