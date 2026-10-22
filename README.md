# CQRS-data-model-design

## Scope: 
Business application system integration data layer: CQRS (Command Query Responsibility Segregation) architecture and Temporal SQL tables - for custom data model builds 

## Audience: 
Solution specialists and developers Questions: roadmap or runbook how to design and implement it. 

## Reference: 
Best design patterns and practices. 

## Requirements: 
- Focus on the custom data model design and build in Microsoft SQL server.
- This custom data model can catch all business entity change events (including historical transaction records).
- Show the most current state of business entities, such as balances, etc.
- Replay what had been changed in the past for traceability and auditability. 

## Recommendation (short): 

Build a hybrid solution that uses (in MS SQL Server):
- CQRS write-model
    1. An append-only Event Store
    2. Domain model applies commands, mutates states, invariant enforcement, and writes events
    3. Transactional Outbox to capture every domain change as events
- CQRS read-model projections that are rebuildable by replaying events
- System-versioned Temporal Tables for row-level history where you need native SQL time-travel
- **Snapshot store** (optional): store periodic snapshots of aggregates to speed rehydration for large event streams.


This hybrid design solustion gives strong auditability, a current-state read model, and deterministic replay for traceability.
