# Ubiquitous Language (通用语言)

**All names MUST be business terms, NEVER technical jargon. When in doubt, ASK the user.**

## The Iron Law

```
CODE SPEAKS BUSINESS, NOT TECH
```

Every class name, method name, variable name, and module name should be understandable by a domain expert who doesn't write code.

## Bad vs Good Examples

| ❌ Bad (Technical) | ✅ Good (Business) |
|-------------------|--------------------|
| `userId: string` | `customerId: CustomerId` |
| `getData()` | `findOrdersByCustomer()` |
| `processItem()` | `fulfillOrder()` |
| `handleEvent()` | `confirmPayment()` |
| `validateInput()` | `verifyShippingAddress()` |
| `updateRecord()` | `changeDeliveryDate()` |
| `ItemProcessor` | `OrderFulfillmentService` |
| `DataManager` | `InventoryTracker` |
| `EventHandler` | `OrderConfirmationNotifier` |
| `StringHelper` | `PhoneNumberFormatter` |

## Unknown Terms? ASK.

**Don't guess. Don't invent.** When unsure about a business term:

1. **Stop and ask the user:**
   - "What does [term] mean in your business?"
   - "What's the difference between [termA] and [termB]?"
   - "Should I call this [optionA] or [optionB]?"

2. **Record the decision** in `.agent/ddd/glossary.md`

## Glossary Format

Maintain a glossary file for each bounded context:

```markdown
# Domain Glossary

## Order Context

| Term (English) | Term (中文) | Definition | Notes |
|----------------|-------------|------------|-------|
| Order | 订单 | A customer's request to purchase items | Not "shopping cart" |
| Line Item | 订单行项 | A single product line within an Order | Has quantity and unit price |
| Fulfillment | 履约 | The process of shipping an Order | Separate from payment |
| Cancellation | 取消 | Voiding an order before shipment | Has a required reason |

## Payment Context

| Term (English) | Term (中文) | Definition | Notes |
|----------------|-------------|------------|-------|
| Payment | 支付 | Transfer of money for an Order | Can be partial |
| Refund | 退款 | Return of payment to customer | Requires approval |
```

## Storage Location

```
.agent/ddd/glossary.md
```

This file is:
- **Shared** across conversations — future AI sessions read it
- **Versioned** with the project
- **Living** — updated as understanding grows

## Rules for Naming

1. **Method names describe business actions**, not technical operations:
   - `placeOrder()` not `insertOrder()`
   - `cancelShipment()` not `deleteShipment()`
   - `applyDiscount()` not `setDiscount()`

2. **Class names use domain nouns**:
   - `Order`, `Customer`, `Shipment` not `OrderData`, `CustomerModel`, `ShipmentRecord`

3. **Events use past tense** (see [domain-events.md](domain-events.md)):
   - `OrderPlaced`, `PaymentReceived` not `PlaceOrder`, `ProcessPayment`

4. **Commands use imperative mood**:
   - `PlaceOrderCommand`, `CancelShipmentCommand`

5. **Value Objects describe the concept**:
   - `Money`, `Address`, `DateRange` not `MoneyVO`, `AddressData`

6. **Bilingual projects**: Include both English and Chinese terms in the glossary. Code uses English; documentation may use Chinese.

## Refactoring Existing Code

When you find technical naming in existing code:

```typescript
// Before (technical)
class DataProcessor {
  processItems(items: any[]): ProcessResult { ... }
}

// After (business)
class OrderFulfillmentService {
  fulfillOrders(orders: Order[]): FulfillmentResult { ... }
}
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Guessing business terms | Stop and ask the user |
| Not recording decisions | Add to `.agent/ddd/glossary.md` |
| Technical suffixes (`VO`, `DTO`, `Model`) on domain classes | Use clean business names |
| Different terms for same concept | Pick one and add to glossary |
| Same term for different concepts | Split into bounded contexts |
