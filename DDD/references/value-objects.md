# Value Objects

**No identity. Equality by attributes. Immutable.**

## When to Use

```
Does the concept...
├─ Have NO lifecycle identity? (compared by value, not by ID)
├─ Consist of multiple related fields?
├─ Have behavior attached to it? (add, subtract, validate)
└─ Need immutability guarantees?

If YES to most → Value Object
If single value with validation → Brand Type (see primitive-types.md)
```

## Core Properties

1. **No Identity** — Two `Money(100, "USD")` are the same, regardless of which instance
2. **Immutable** — Never mutate; return new instances
3. **Self-Validating** — Invalid state is impossible after construction
4. **Equality by Value** — Compare all fields, not reference

## Implementation Pattern

```typescript
class Money {
  private constructor(
    private readonly _amount: number,
    private readonly _currency: Currency
  ) {
    if (_amount < 0) throw new NegativeAmountError(_amount);
  }

  // Factory method enforces validation
  static create(amount: number, currency: Currency): Money {
    return new Money(amount, currency);
  }

  // Zero instance for safe defaults
  static zero(currency: Currency): Money {
    return new Money(0, currency);
  }

  // Getters (no setters — immutable)
  get amount(): number { return this._amount; }
  get currency(): Currency { return this._currency; }

  // Operations return NEW instances
  add(other: Money): Money {
    this.ensureSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  subtract(other: Money): Money {
    this.ensureSameCurrency(other);
    return new Money(this._amount - other._amount, this._currency);
  }

  multiply(factor: number): Money {
    return new Money(this._amount * factor, this._currency);
  }

  // Value-based equality
  equals(other: Money): boolean {
    return this._amount === other._amount
        && this._currency === other._currency;
  }

  private ensureSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new CurrencyMismatchError(this._currency, other._currency);
    }
  }
}
```

## More Examples

### Address

```typescript
class Address {
  private constructor(
    private readonly _street: string,
    private readonly _city: string,
    private readonly _postalCode: PostalCode,
    private readonly _country: CountryCode
  ) {}

  static create(street: string, city: string, postalCode: PostalCode, country: CountryCode): Address {
    if (!street.trim()) throw new InvalidAddressError('Street is required');
    if (!city.trim()) throw new InvalidAddressError('City is required');
    return new Address(street, city, postalCode, country);
  }

  equals(other: Address): boolean {
    return this._street === other._street
        && this._city === other._city
        && this._postalCode === other._postalCode
        && this._country === other._country;
  }
}
```

### DateRange

```typescript
class DateRange {
  private constructor(
    private readonly _start: Date,
    private readonly _end: Date
  ) {
    if (_start >= _end) throw new InvalidDateRangeError(_start, _end);
  }

  static create(start: Date, end: Date): DateRange {
    return new DateRange(start, end);
  }

  contains(date: Date): boolean {
    return date >= this._start && date <= this._end;
  }

  overlaps(other: DateRange): boolean {
    return this._start < other._end && this._end > other._start;
  }

  get durationInDays(): number {
    return Math.ceil((this._end.getTime() - this._start.getTime()) / (1000 * 60 * 60 * 24));
  }

  equals(other: DateRange): boolean {
    return this._start.getTime() === other._start.getTime()
        && this._end.getTime() === other._end.getTime();
  }
}
```

## Composing Value Objects

Value Objects can contain other Value Objects:

```typescript
class OrderTotal {
  private constructor(
    private readonly _subtotal: Money,
    private readonly _tax: Money,
    private readonly _discount: Money
  ) {}

  get total(): Money {
    return this._subtotal.add(this._tax).subtract(this._discount);
  }

  equals(other: OrderTotal): boolean {
    return this._subtotal.equals(other._subtotal)
        && this._tax.equals(other._tax)
        && this._discount.equals(other._discount);
  }
}
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Public constructor | Use `private constructor` + `static create()` |
| Setter methods | Return new instance from operations |
| Missing validation | Validate in constructor; invalid state = impossible |
| Identity-based equality (`===`) | Override `equals()` to compare all fields |
| Using Value Object where Brand Type suffices | Single value + validation → Brand Type |
