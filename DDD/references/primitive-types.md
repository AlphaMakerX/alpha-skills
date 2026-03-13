# Primitive Types (Brand Types)

## The Rule

```
PRIMITIVES ARE FORBIDDEN IN DOMAIN LOGIC
```

Raw `string`, `number`, `boolean` in domain code = **primitive obsession** anti-pattern. It allows:
- Passing `userId` where `orderId` is expected (both are `string`)
- Negative values for quantities
- Empty strings for required fields

## Brand Type Pattern

```typescript
// Core Brand utility
type Brand<T, B extends string> = T & { readonly __brand: B };

// Domain IDs
type CustomerId = Brand<string, 'CustomerId'>;
type OrderId = Brand<string, 'OrderId'>;
type ProductId = Brand<string, 'ProductId'>;

// Domain quantities
type Quantity = Brand<number, 'Quantity'>;
type Percentage = Brand<number, 'Percentage'>;
type EmailAddress = Brand<string, 'EmailAddress'>;
```

## Factory Functions with Validation

Every Brand Type needs a **factory function** that validates before casting:

```typescript
function createCustomerId(value: string): CustomerId {
  if (!value || value.trim().length === 0) {
    throw new InvalidCustomerIdError(value);
  }
  return value as CustomerId;
}

function createQuantity(value: number): Quantity {
  if (!Number.isInteger(value) || value < 0) {
    throw new InvalidQuantityError(value);
  }
  return value as Quantity;
}

function createEmailAddress(value: string): EmailAddress {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(value)) {
    throw new InvalidEmailError(value);
  }
  return value as EmailAddress;
}
```

## Type Safety in Practice

```typescript
// ❌ Primitive obsession — compiles but wrong
function findOrder(id: string): Order { ... }
findOrder(customerId);  // No compile error!

// ✅ Brand Types — compile-time safety
function findOrder(id: OrderId): Order { ... }
findOrder(customerId);  // Compile error: CustomerId ≠ OrderId
```

## Common Domain Primitives

| Concept | Base Type | Example |
|---------|-----------|---------|
| Entity IDs | `string` | `OrderId`, `UserId`, `ProductId` |
| Money amounts | `number` | `Amount` (use with currency in Value Object) |
| Quantities | `number` | `Quantity`, `StockLevel` |
| Percentages | `number` | `DiscountRate`, `TaxRate` |
| Identifiers | `string` | `EmailAddress`, `PhoneNumber`, `SKU` |
| Timestamps | `Date` | `CreatedAt`, `ExpiresAt` |

## When to Use Brand Types vs Value Objects

```
Is it a single value with validation?
├─ Yes → Brand Type (lightweight)
│        Example: OrderId, EmailAddress, Quantity
└─ No  → Value Object (multi-field, behavior)
         Example: Money(amount + currency), Address(street + city + ...)
```

## Organizing Brand Types

```typescript
// src/domain/shared/kernel/types.ts
// Central location for cross-context Brand Types

type Brand<T, B extends string> = T & { readonly __brand: B };

// IDs
export type CustomerId = Brand<string, 'CustomerId'>;
export type OrderId = Brand<string, 'OrderId'>;

// Factories
export function createCustomerId(value: string): CustomerId { ... }
export function createOrderId(value: string): OrderId { ... }
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| No validation in factory | Always validate before casting |
| Factory returns `undefined` on invalid | Throw domain-specific error instead |
| Brand Type for multi-field concept | Use Value Object for `Money`, `Address`, etc. |
| Skipping factory, casting directly | Never `as OrderId` outside factory functions |
