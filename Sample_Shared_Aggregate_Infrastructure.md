
## Shared aggregate infrastructure: reusable mechanics behind every command ##

**Design intent**: Shared aggregate infrastructure is a small, stable platform that every command uses for cross-cutting correctness: 
-	command idempotency, 
-	aggregate-version concurrency, 
-	event sequencing, 
-	event metadata, 
-	outbox persistence, 
-	transaction conventions, 
-	security, and 
-	diagnostics. 

It standardizes how an accepted decision is committed; **"it does not decide whether ApprovePurchaseOrder, ReceiveGoods, or RegisterSupplierInvoice is allowed."**

It means **shared CQRS infrastructure provides execution mechanics, not business judgment**.
---
For example, the infrastructure can:

- enforce `CommandId` idempotency;
- verify `ExpectedVersion`;
- increment `AggregateVersion`;
- append an event;
- create an outbox message;
- commit everything atomically.

But it cannot determine whether a specific action is valid, such as:

- whether this user may approve the PO;
- whether the PO is in `Submitted` status;
- whether goods may be received against a closed PO;
- whether the receipt exceeds the allowed tolerance;
- whether an invoice number is acceptable.

**Those decisions belong to the named command handler—such as `ApprovePurchaseOrder`, `ReceiveGoods`, or `RegisterSupplierInvoice`—and its domain policy.**

It means: 
- **Shared infrastructure standardizes how an already validated business decision is committed.** 
- **The business command handler decides whether the requested action is authorized and valid.**

The execution order should be understood as:

1. Receive the command.
2. Check authorization, current state, and business rules.
3. Decide to accept or reject it.
4. If accepted, use shared infrastructure to persist the state change, event, command result, and outbox message atomically.
5. If rejected, do not advance the aggregate version or append a state-changing event.

**“Doesn’t decide whether it is allowed”** therefore refers to **domain authorization and business validity's responsibility**, not database permission to execute the procedure.

## Separation between domain policy and shared mechanics 

| Business-command responsibility | Shared-infrastructure responsibility |
|---|---|
| Interpret command intent and validate required business input.	| Validate standard envelope fields such as CommandId, AggregateId, ExpectedVersion, actor, source, and correlation identifiers.|
| Load the authoritative PO, receipt, or invoice facts needed for the decision. | Own standard tables, keys, constraints, metadata formats, error categories, and transaction conventions. |
| Enforce named invariants and calculate the resulting state transition. | Atomically claim command idempotency and advance the aggregate version. |
| Create the business event type and payload. | Append the event with sequence/version guarantees and create its outbox work item. |
| Return business-specific output and rejection details. | Persist a repeatable command result for safe retries and emit standard telemetry. |

### Core rule: 

Infrastructure may expose narrow primitives such as:
- “claim command,”
- “advance this aggregate from expected version 5 to 6,”
- “append these validated events,” and
- “enqueue this publication.”

It **must not expose** 
- “save arbitrary table,”
- “update any column,” or 
- “emit any caller-supplied event without an owning command.”

Business procedures compose approved primitives inside their own transaction.


## Shared data contracts 

``` TSQL
CREATE SCHEMA cqrs;
GO

CREATE TABLE cqrs.AggregateHead
(
 AggregateType sysname NOT NULL,
 AggregateId uniqueidentifier NOT NULL,
 CurrentVersion bigint NOT NULL,
 LastEventSequence bigint NULL,
 ModifiedAtUtc datetime2(7) NOT NULL,
 CONSTRAINT PK_AggregateHead
   PRIMARY KEY (AggregateType, AggregateId),
 CONSTRAINT CK_AggregateHead_Version CHECK (CurrentVersion >= 0)
);

CREATE TABLE cqrs.ProcessedCommand
(
 CommandId uniqueidentifier NOT NULL,
 CommandType sysname NOT NULL,
 AggregateType sysname NOT NULL,
 AggregateId uniqueidentifier NOT NULL,
 CommandPayloadHash varbinary(32) NOT NULL,
 StatusCode varchar(20) NOT NULL,
 ResultingVersion bigint NULL,
 ResultEventId uniqueidentifier NULL,
 ResultPayload nvarchar(max) NULL,
 StartedAtUtc datetime2(7) NOT NULL,
 CompletedAtUtc datetime2(7) NULL,
 CONSTRAINT PK_ProcessedCommand PRIMARY KEY (CommandId),
 CONSTRAINT CK_ProcessedCommand_Status
   CHECK (StatusCode IN ('Started','Completed','Rejected'))
);

CREATE TABLE cqrs.DomainEvent
(
 SequenceNumber bigint IDENTITY(1,1) NOT NULL,
 EventId uniqueidentifier NOT NULL,
 AggregateType sysname NOT NULL,
 AggregateId uniqueidentifier NOT NULL,
 AggregateVersion bigint NOT NULL,
 EventType sysname NOT NULL,
 EventSchemaVersion int NOT NULL,
 EventPayload nvarchar(max) NOT NULL,
 Metadata nvarchar(max) NOT NULL,
 CreatedAtUtc datetime2(7) NOT NULL,
 CONSTRAINT PK_DomainEvent PRIMARY KEY CLUSTERED (SequenceNumber),
 CONSTRAINT UQ_DomainEvent_Id UNIQUE (EventId),
 CONSTRAINT UQ_DomainEvent_AggregateVersion
   UNIQUE (AggregateType, AggregateId, AggregateVersion),
 CONSTRAINT CK_DomainEvent_Payload CHECK (ISJSON(EventPayload)=1),
 CONSTRAINT CK_DomainEvent_Metadata CHECK (ISJSON(Metadata)=1)
);

CREATE TABLE cqrs.OutboxMessage
(
 MessageId uniqueidentifier NOT NULL,
 EventId uniqueidentifier NOT NULL,
 Destination nvarchar(300) NOT NULL,
 MessageType sysname NOT NULL,
 MessagePayload nvarchar(max) NOT NULL,
 StatusCode varchar(20) NOT NULL,
 AttemptCount int NOT NULL,
 AvailableAtUtc datetime2(7) NOT NULL,
 LockedBy nvarchar(200) NULL,
 LockedUntilUtc datetime2(7) NULL,
 CreatedAtUtc datetime2(7) NOT NULL,
 DispatchedAtUtc datetime2(7) NULL,
 LastError nvarchar(2000) NULL,
 CONSTRAINT PK_OutboxMessage PRIMARY KEY (MessageId),
 CONSTRAINT UQ_OutboxMessage_EventDestination UNIQUE (EventId, Destination),
 CONSTRAINT FK_OutboxMessage_Event FOREIGN KEY (EventId)
   REFERENCES cqrs.DomainEvent(EventId),
 CONSTRAINT CK_OutboxMessage_Status
   CHECK (StatusCode IN ('Pending','Claimed','Dispatched','Failed'))
);
```

### Command-payload hash. 

**CommandId** identifies the logical request. 

Store a deterministic hash of **the normalized command payload** so, a retry with the same CommandId but different business data is rejected as misuse rather than mistaken for the original request.

```text 
One CommandId identifies one immutable logical command and must remain associated with one normalized command payload.
```
**One `CommandId` identifies one immutable logical command and must remain associated with one normalized command payload.**

The payload hash enforces that rule efficiently.

## What “normalized command payload” means

Before hashing, represent all **business inputs that define the command** in one canonical form:

- fixed field order;
- consistent casing where case is insignificant;
- normalized decimals and dates;
- trimmed identifiers;
- explicit null handling;
- stable collection ordering when order has no business meaning.

For example:

```text
CommandId: CMD-501
CommandType: ChangePurchaseOrderLineQuantity
PurchaseOrderId: PO-45001234
LineNumber: 10
NewQuantity: 120.0000
ExpectedVersion: 7
```

Equivalent representations such as `120`, `120.0`, and `120.0000` should produce the same normalized value and therefore the same hash.

Usually exclude transport metadata that does not define the business request, such as retry count or message-delivery timestamp.

## Retry outcomes

| Existing `CommandId` | Incoming hash | Result |
|---|---|---|
| Not found | Any valid hash | Execute and record command |
| Found | Same hash | Return the previously stored result |
| Found | Different hash | Reject as `CommandIdPayloadMismatch` |

The system does **not** rerun the matching command. It returns the original stored outcome, such as the same `EventId`, resulting version, or rejection result.

## Why different business data must be rejected

Suppose the original request was:

```text
CommandId: CMD-501
Change line 10 quantity to 120
```

It succeeds and produces version 8.

Later, the system receives:

```text
CommandId: CMD-501
Change line 10 quantity to 150
```

This is **not a retry**. It is a different business decision accidentally or improperly using the original identity.

If the system simply returns the original result, the caller may wrongly believe quantity `150` was applied. If it executes again, one `CommandId` would represent two decisions and **idempotency** would be broken. Therefore, rejection is the only safe result.

## Common misuse and rejection cases

1. **Client reuses one ID for multiple edits** 
   A UI creates the `CommandId` when the form opens and reuses it after the user changes the quantity. The second submission must use a new `CommandId`.

2. **Integration uses a stable business ID as `CommandId`** 
   For example, it uses the PO number for every PO update. Since a PO has many commands during its lifetime, later updates produce different hashes and are rejected.

3. **Retry code rebuilds data from current state** 
   The retry retains the original `CommandId` but refreshes `ExpectedVersion`, recalculates an amount, or changes defaults. That is a reevaluated command, not the original retry.

4. **Message transformation changes between deliveries** 
   A mapping deployment changes rounding, unit conversion, or field defaults while an old message is retried. The mismatch prevents the changed interpretation from silently masquerading as the original command.

5. **Programming or routing defect** 
   Two unrelated requests receive the same `CommandId` because of caching, object reuse, or incorrect ID generation.

6. **Partial or malicious payload substitution** 
   The identity remains the same, but an aggregate ID, quantity, account, supplier, or other business field changes. Rejection prevents an existing idempotency key from authorizing a different operation.

## Important distinction

- **Same `CommandId` + same normalized payload** → same logical command; return the stored result.
- **New `CommandId` + changed payload** → new command; validate it against the current state.
- **Same `CommandId` + changed payload** → identity conflict; reject it.

#### Changing `ExpectedVersion` is also normally a payload change. A command that was reevaluated after a concurrency conflict represents a new decision and should receive a **new `CommandId`**, while retaining the same `CorrelationId` if it remains part of the same workflow.



## Reusable command lifecycle 
1.	**Normalize and validate the envelope.** Require CommandId, CommandType, AggregateType, AggregateId, ExpectedVersion, ActorId, SourceSystem, CorrelationId, and a reason where policy requires it.
2.	**Begin a short local transaction.** Use SET XACT_ABORT ON and TRY/CATCH. Do not call remote services while the transaction is open.
3.	**Claim idempotency.** Insert ProcessedCommand with status Started. A duplicate CommandId loads and returns the previously completed result when the payload hash matches.
4.	**Acquire the concurrency right.** Create or conditionally update AggregateHead only when CurrentVersion equals ExpectedVersion. Failure is a concurrency conflict, not a generic SQL error.
5.	**Run business policy.** The named command reads authoritative aggregate rows, checks invariants, and computes new state and events.
6.	**Persist the accepted transition.** Update temporal current-state rows, append domain events, and create outbox rows.
7.	**Complete the command record.** Store resulting version, event identifier, and stable result payload.
8.	**Commit and return.** Return the same result contract to the original caller and every valid retry.

## Narrow reusable SQL primitives

A practical SQL implementation can centralize repetitive persistence code in tightly scoped internal procedures. Keep them in the cqrs schema, deny direct execution to application roles, and allow only signed business-command modules to invoke them.

``` TSQL
-- Called inside the business command's existing transaction.
CREATE OR ALTER PROCEDURE cqrs.AdvanceAggregateVersion
 @AggregateType sysname,
 @AggregateId uniqueidentifier,
 @ExpectedVersion bigint,
 @NewVersion bigint OUTPUT
AS
BEGIN
 SET NOCOUNT ON;
 SET @NewVersion = @ExpectedVersion + 1;

 UPDATE cqrs.AggregateHead
 SET CurrentVersion = @NewVersion,
     ModifiedAtUtc = SYSUTCDATETIME()
 WHERE AggregateType = @AggregateType
   AND AggregateId = @AggregateId
   AND CurrentVersion = @ExpectedVersion;

 IF @@ROWCOUNT <> 1
   THROW 51301, 'Aggregate version conflict.', 1;
END;

CREATE OR ALTER PROCEDURE cqrs.AppendEventAndOutbox
 @EventId uniqueidentifier,
 @AggregateType sysname,
 @AggregateId uniqueidentifier,
 @AggregateVersion bigint,
 @EventType sysname,
 @EventSchemaVersion int,
 @EventPayload nvarchar(max),
 @Metadata nvarchar(max),
 @Destination nvarchar(300)
AS
BEGIN
 SET NOCOUNT ON;

 IF ISJSON(@EventPayload) <> 1 OR ISJSON(@Metadata) <> 1
   THROW 51302, 'Event payload or metadata is not valid JSON.', 1;

 INSERT cqrs.DomainEvent
  (EventId, AggregateType, AggregateId, AggregateVersion,
   EventType, EventSchemaVersion, EventPayload, Metadata, CreatedAtUtc)
 VALUES
  (@EventId, @AggregateType, @AggregateId, @AggregateVersion,
   @EventType, @EventSchemaVersion, @EventPayload, @Metadata, SYSUTCDATETIME());

 INSERT cqrs.OutboxMessage
  (MessageId, EventId, Destination, MessageType, MessagePayload,
   StatusCode, AttemptCount, AvailableAtUtc, CreatedAtUtc)
 VALUES
  (NEWID(), @EventId, @Destination, @EventType, @EventPayload,
   'Pending', 0, SYSUTCDATETIME(), SYSUTCDATETIME());
END;
```

**Transaction ownership rule:** 
- The outer business command owns BEGIN TRANSACTION, COMMIT, and ROLLBACK.
- Shared primitives must not silently commit an outer transaction.
- They should assume an active transaction or document their behavior consistently, use THROW to preserve the original error, and avoid hidden side effects.

## Business command composing the shared infrastructure

``` TSQL
CREATE OR ALTER PROCEDURE purchasing.ApprovePurchaseOrder
 @CommandId uniqueidentifier,
 @PurchaseOrderId uniqueidentifier,
 @ExpectedVersion bigint,
 @ActorId nvarchar(200),
 @CorrelationId uniqueidentifier
AS
BEGIN
 SET NOCOUNT ON;
 SET XACT_ABORT ON;

 DECLARE @NewVersion bigint, @EventId uniqueidentifier = NEWID();
 DECLARE @EventPayload nvarchar(max), @Metadata nvarchar(max);

 BEGIN TRY
  BEGIN TRANSACTION;

  -- Shared idempotency primitive claims @CommandId or returns its prior result.
  EXEC cqrs.ClaimCommand
    @CommandId=@CommandId,
    @CommandType=N'ApprovePurchaseOrder',
    @AggregateType=N'PurchaseOrder',
    @AggregateId=@PurchaseOrderId;

  EXEC cqrs.AdvanceAggregateVersion
    @AggregateType=N'PurchaseOrder',
    @AggregateId=@PurchaseOrderId,
    @ExpectedVersion=@ExpectedVersion,
    @NewVersion=@NewVersion OUTPUT;

  -- Business-specific policy remains visible here.
  IF NOT EXISTS
   (SELECT 1 FROM purchasing.PurchaseOrder
    WHERE PurchaseOrderId=@PurchaseOrderId
      AND StatusCode='Submitted')
    THROW 51401, 'Only a submitted PO can be approved.', 1;

  IF EXISTS
   (SELECT 1 FROM purchasing.PurchaseOrderLine
    WHERE PurchaseOrderId=@PurchaseOrderId
      AND (OrderedQty <= 0 OR UnitPrice < 0))
    THROW 51402, 'PO contains an invalid line.', 1;

  UPDATE purchasing.PurchaseOrder
  SET StatusCode='Approved', ApprovedBy=@ActorId,
      ApprovalDateUtc=SYSUTCDATETIME(),
      AggregateVersion=@NewVersion, LastEventId=@EventId,
      LastCommandId=@CommandId, ModifiedAtUtc=SYSUTCDATETIME()
  WHERE PurchaseOrderId=@PurchaseOrderId;

  -- Build a typed PurchaseOrderApproved payload and standard metadata.
  EXEC cqrs.AppendEventAndOutbox
    @EventId=@EventId, @AggregateType=N'PurchaseOrder',
    @AggregateId=@PurchaseOrderId, @AggregateVersion=@NewVersion,
    @EventType=N'PurchaseOrderApproved', @EventSchemaVersion=1,
    @EventPayload=@EventPayload, @Metadata=@Metadata,
    @Destination=N'procurement-events';

  EXEC cqrs.CompleteCommand
    @CommandId=@CommandId, @ResultingVersion=@NewVersion,
    @ResultEventId=@EventId;

  COMMIT TRANSACTION;
 END TRY
 BEGIN CATCH
  IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
  THROW;
 END CATCH;
END;
```

The same infrastructure is reused by ReceiveGoods and RegisterSupplierInvoice, but their policy blocks differ. 
-	ReceiveGoods validates PO eligibility, tolerance, receipt identity, and reversal state, then emits GoodsReceived. 
-	RegisterSupplierInvoice validates the normalized supplier invoice key and invoice totals, then emits SupplierInvoiceRegistered.
  
Reuse occurs below the domain decision, not instead of it.
---
“**Reuse occurs below the domain decision, not instead of it**” means:

### Domain decision comes first

Each business command must decide whether the requested action is valid.

For example, `ApprovePurchaseOrder` checks:

- Is the PO currently submitted?
- Is the caller authorized?
- Are all required fields and lines valid?
- Does `ExpectedVersion` still match?

Only this command’s business logic can conclude: **approval is allowed**.

### Shared infrastructure executes an accepted decision

After approval is allowed, reusable infrastructure can:

- claim `CommandId` for idempotency;
- advance the aggregate version;
- append the validated event;
- create the outbox message;
- store the repeatable command result;
- commit or roll back the transaction.

These are generic mechanics. They do not understand what “approval” means.

### Why infrastructure cannot replace the decision

A generic operation such as:

```text
Advance version
Append event: PurchaseOrderApproved
Publish message
```

could successfully perform all its technical steps even when the PO is cancelled or the caller lacks approval authority. Technically valid persistence does not prove a legally valid business transition.

Therefore:

```text
Business command
    → checks policy and derives the event
    → reusable infrastructure safely persists that decision
```

Not:

```text
Caller chooses an event
    → generic infrastructure assumes it is valid
```

**In short:** reuse the transaction, concurrency, idempotency, event-storage, and outbox mechanics. Do not replace command-specific validation and event derivation with generic infrastructure.



## Creation commands and version zero

Creating an aggregate requires a separate race-safe primitive. Insert AggregateHead at the agreed initial version and rely on the primary key to reject concurrent creation of the same AggregateType + AggregateId. The command then inserts authoritative rows and appends the creation event. Do not overload “expected version = -1” without documenting exactly which event version and head version will result.

## Multiple events from one command

A command may emit more than one event—for example, ReceiveGoods may emit GoodsReceiptPosted and PurchaseOrderReceiptLimitReached. Pass a table-valued parameter containing events in deterministic ordinal order. The shared append primitive assigns contiguous aggregate versions and creates one outbox row per publishable event. Advance AggregateHead to the final assigned version only within the same transaction.

## Security and module boundaries

- Grant application roles EXECUTE only on business-command procedures such as purchasing.ApprovePurchaseOrder.
-	Deny direct INSERT, UPDATE, and DELETE on authoritative aggregate tables, DomainEvent, AggregateHead, ProcessedCommand, and OutboxMessage.
-	Deny direct application execution of internal cqrs primitives. Use ownership chaining or signed modules so approved command procedures can access them.
-	Give the outbox dispatcher least-privilege access only to claim, read, and mark outbox work; it must not change domain rows or events.
-	Expose read models and audit views through separate read-only roles.

## Error contract, observability, and testing

-	Error taxonomy: distinguish validation rejection, business-rule rejection, concurrency conflict, duplicate command, duplicate business key, and infrastructure failure. Map each to a stable application error contract.
-	Telemetry: record CommandId, AggregateId, command type, duration, outcome, retry status, resulting version, EventId, and SQL error number. Never log sensitive payload fields indiscriminately.
-	Contract tests: every command must prove that success commits state + events + outbox + completed command result, while any injected failure rolls all of them back.
-	Concurrency tests: simultaneous commands using the same ExpectedVersion must produce one winner and explicit conflicts for the others.
-	Idempotency tests: the same CommandId and payload returns the same result; the same CommandId with a different payload is rejected.
-	Security tests: application identities can execute business commands but cannot update domain or infrastructure tables directly.

**Effective implementation rule.** 
- Keep shared infrastructure small, deterministic, transaction-safe, and domain-agnostic.
- Keep business intent, invariant ownership, event naming, and state-transition logic in named command modules.
- That combination gives reuse without generic CRUD, centralized correctness without one monolithic stored procedure, and an auditable path from CommandId to AggregateVersion to EventId to OutboxMessage.
