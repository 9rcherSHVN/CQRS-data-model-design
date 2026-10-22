
## Common implementation rules - Aggregates - CQRS Write Model

1.	Shared infrastructure standardizes transaction handling, concurrency, idempotency, event metadata, and outbox persistence; it must not become a generic “save any entity” API.
   
3.	Each authoritative command procedure still contains or invokes the named domain policy for its own business operation and rules.

---
**This design gives developers one place to find the rules, one path to test, one path to secure, and one event contract to audit.**

For example:

- a Purchase Order aggregate may ensure that ordered quantities are positive and that a closed order cannot be changed; 
- a Receipt aggregate may ensure that a receipt references an eligible PO line; 
- an Invoice aggregate may prevent duplicate supplier invoice numbers.
  
On a successful command, the handler appends events such as PurchaseOrderCreated, GoodsReceived, or SupplierInvoiceRegistered. 

The append uses AggregateId + ExpectedVersion optimistic concurrency. 

- If another request has already advanced the aggregate version, the unique version constraint rejects the stale write;
- The application reloads the aggregate and decides whether to retry, merge, or return a business conflict. 

**Developers must not update an event row after insertion.**

---

## Shared aggregate infrastructure behind every command

- **cqrs.AggregateHead**: current version and concurrency gate per aggregate instance.
- **cqrs.ProcessedCommand**: unique CommandId, status, resulting aggregate version, and serialized command result for safe retries.
-	**cqrs.DomainEvent**: append-only events with a unique AggregateType + AggregateId + AggregateVersion constraint.
-	**cqrs.OutboxMessage**: messages committed with the state change and published asynchronously.
-	**Temporal domain tables**: current relational state plus automatic system-time history.
-	**Security roles**: application identities receive EXECUTE on command procedures but **no direct INSERT, UPDATE, or DELETE permission on aggregate tables**.

