# Verification Checklist

Run through this checklist after implementing DDD patterns. Every item should pass.

## Naming (Ubiquitous Language)

- [ ] All class names use business terms (not `DataProcessor`, `ItemHandler`)
- [ ] All method names describe business actions (not `process()`, `handle()`)
- [ ] No technical suffixes on domain classes (`VO`, `DTO`, `Model`, `Record`)
- [ ] Unknown terms have been asked about and recorded in glossary
- [ ] Consistent terminology — same concept uses same name everywhere

## Type Safety (No Primitive Obsession)

- [ ] All entity IDs use Brand Types (`OrderId`, not `string`)
- [ ] Domain quantities use typed values (`Quantity`, not `number`)
- [ ] All Brand Types have factory functions with validation
- [ ] No `as` casting outside of factory functions
- [ ] No `any` or `unknown` in domain layer

## Entities

- [ ] Each entity has a unique identity (Brand Type ID)
- [ ] Equality is identity-based (`equals()` compares ID)
- [ ] No public setters — use named behavior methods
- [ ] Rich behavior — entity owns its business rules (not anemic)
- [ ] State transitions are guarded by invariant checks
- [ ] Internal collections are not exposed mutably (return copies)

## Value Objects

- [ ] No identity — equality by value (all fields compared)
- [ ] Immutable — operations return new instances
- [ ] Self-validating — invalid state is impossible after construction
- [ ] Private constructor + `static create()` factory method
- [ ] No setters

## Aggregates

- [ ] Aggregate Root is the only entry point for modifications
- [ ] Internal entities are not directly accessible from outside
- [ ] Cross-aggregate references use IDs only (not object references)
- [ ] Aggregate is as small as possible (minimal consistency boundary)
- [ ] One transaction modifies one aggregate only
- [ ] Complex creation uses Factory pattern when needed

## Domain Events

- [ ] Named in past tense (`OrderPlaced`, not `PlaceOrder`)
- [ ] Events are immutable data objects
- [ ] Events contain only IDs and primitives (not domain object references)
- [ ] Collect-then-dispatch pattern used (publish after save)
- [ ] Cross-aggregate communication uses events (eventual consistency)

## Domain Services

- [ ] Stateless — no instance fields
- [ ] Contains business logic that spans multiple entities/aggregates
- [ ] Lives in domain layer — no infrastructure dependencies
- [ ] Named with business terms

## Repositories

- [ ] One repository per aggregate root only
- [ ] Interface defined in domain layer
- [ ] Implementation in infrastructure layer
- [ ] Returns fully reconstituted aggregates (not partial objects)
- [ ] No business logic in repository implementation

## Application Services

- [ ] Follow Load → Execute → Persist → Publish pattern
- [ ] Contains NO business logic — only orchestration
- [ ] Uses Commands for input
- [ ] Maps DTOs ↔ domain objects at this layer
- [ ] Transaction boundaries are defined here

## Layered Architecture

- [ ] Domain layer has ZERO external dependencies (no `import axios`, etc.)
- [ ] Infrastructure implements domain interfaces (Dependency Inversion)
- [ ] UI layer does not call repositories directly
- [ ] No business rules in controllers or infrastructure
- [ ] Clear directory structure per layer

## Anti-Corruption Layer

- [ ] All external data goes through ACL before entering domain
- [ ] External DTOs are separate from domain models
- [ ] Translation logic lives in infrastructure layer
- [ ] Domain model designed from business needs, not API shape
- [ ] Pure/impure separation is maintained
