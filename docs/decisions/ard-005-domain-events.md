# ARD-005: Domain Events

## Date
2025-04-11

## Status
Proposed

## Context
Our Cargo Tracking System implements Clean Architecture and Domain-Driven Design principles across both iOS (Swift) and Android (Kotlin) platforms. As our system grows in complexity, we face the challenge of maintaining proper aggregate boundaries while enabling communication between different parts of the system.

Key challenges we are addressing:

1. **Maintaining Aggregate Boundaries**: In DDD, aggregates should be consistent internal units with clear boundaries. Direct modification of one aggregate from another violates these boundaries and creates unwanted coupling.

2. **Implementing Cross-Aggregate Communication**: Business processes often span multiple aggregates, requiring a mechanism for communication that respects boundaries.

3. **Supporting Event-Driven Architecture**: As our system evolves, we need a foundation for event-driven workflows and eventual consistency patterns.

4. **Enabling System Integration**: Our domain needs to communicate significant state changes to external systems and services without tight coupling.

5. **Creating Audit Trails**: We need to track significant business events for compliance, debugging, and business intelligence purposes.

While we do not have an immediate implementation need, establishing a consistent approach to domain events now will provide a foundation for future development and ensure our architecture can evolve to support complex workflows.

## Decision
We will establish a domain events pattern with the following key characteristics:

1. **Event Definition**:
   - Domain events represent significant occurrences within the domain
   - Events are named in past tense, indicating something that has happened
   - Events are immutable value objects containing relevant data
   - Events include timestamp and correlation information

2. **Event Publication**:
   - Domain objects publish events when significant state changes occur
   - Events are published at the end of successful domain operations
   - Publication is handled by an event dispatcher infrastructure service
   
3. **Event Consumption**:
   - Event handlers subscribe to specific event types
   - Handlers perform follow-up actions based on event data
   - Handlers operate in separate transactions from event publishers

4. **Event Storage**:
   - Events are persisted for audit purposes
   - Events can be replayed for system recovery or analysis

## Consequences

### Positive Consequences
- **Decoupled Components**: Aggregates can affect each other without direct references
- **Better Audit Trails**: Events provide a record of significant domain activities
- **Improved Scalability**: Event-driven architecture enables easier scaling of components
- **Enhanced Testability**: Components can be tested in isolation with event mocks
- **Future Flexibility**: Establishes patterns for eventual consistency and CQRS
- **Better Business Alignment**: Events often map directly to significant business occurrences
- **Simplified Integration**: External systems can subscribe to events rather than using direct API calls

### Negative Consequences
- **Indirect Workflow**: Event-driven flows can be harder to trace than direct method calls
- **Eventual Consistency Complexity**: Developers need to think in terms of eventual consistency
- **Infrastructure Requirements**: Additional components needed for event dispatching and persistence
- **Learning Curve**: Event-driven thinking requires a shift in development mindset
- **Debugging Challenges**: Asynchronous event flows can be more difficult to debug

## Alternatives Considered

### 1. Direct Aggregate References
We considered allowing aggregates to directly reference and modify each other. This approach is simpler but violates DDD principles by creating tight coupling between aggregates.

### 2. Application Service Orchestration Only
We considered relying solely on application services to orchestrate all cross-aggregate operations. This works for simple cases but leads to bloated services and doesn't provide audit trails or integration points.

### 3. Message Queue Integration Only
We considered using only external message queues for all event handling. This works well for integration but adds unnecessary complexity for internal domain communication.

### 4. Event Sourcing
We considered a full event sourcing approach where the state is derived entirely from events. While powerful, this represents a significant architectural shift and adds substantial complexity.

## Rationale
Domain events provide a middle ground between tight coupling (direct references) and complete decoupling (separate bounded contexts). They allow our domain model to remain properly encapsulated while enabling the complex workflows required by our business domain.

Events are particularly relevant in shipping logistics, where many processes involve multiple steps with different responsible parties. For example:

1. When a cargo is booked (`CargoBooked` event), several follow-up actions might be needed:
   - The customer needs to be notified
   - The routing service needs to find potential routes
   - The cargo needs to be added to capacity planning systems

2. When a cargo is handled (`CargoHandled` event), multiple updates are required:
   - The tracking status needs to be updated
   - Delivery estimates may need adjustment
   - Customers may need notification
   - Connected voyages may need to update their manifests

Domain events allow these processes to occur without requiring the cargo aggregate to directly know about and call all these related systems.

## Implications

### For Domain Modeling
- Identify significant state changes in domain objects that should trigger events
- Name events in past tense (e.g., `CargoBooked`, not `BookCargo`)
- Include necessary information in events to enable handlers to act
- Consider which domain operations should be event-driven

### For Architecture
- Establish infrastructure for event dispatching and handling
- Determine event persistence strategy
- Consider event versioning for system evolution
- Define patterns for event correlation and causation tracking

## Domain Events Catalog

While we don't have an immediate implementation need, the following catalog outlines potential domain events relevant to our cargo shipping system:

### Cargo Aggregate Events
- **CargoBooked**: When a new cargo booking is confirmed
- **CargoRouteAssigned**: When an itinerary is assigned to cargo
- **CargoRouteChanged**: When a cargo's route specification or itinerary changes
- **CargoHandled**: When a handling event affects cargo (loaded, unloaded, etc.)
- **CargoDelivered**: When cargo reaches its final destination
- **CargoDeliveryEstimateChanged**: When the ETA for cargo delivery changes

### Location Aggregate Events
- **LocationAdded**: When a new port or waypoint is added to the system
- **LocationCapacityChanged**: When a location's handling capacity changes

### Voyage Aggregate Events
- **VoyageRegistered**: When a new voyage is added to the system
- **VoyageScheduleChanged**: When a voyage's schedule is updated
- **VoyageDeparted**: When a voyage leaves a location
- **VoyageArrived**: When a voyage arrives at a location
- **VoyageDelayed**: When a voyage encounters delays

### Customer Aggregate Events
- **CustomerRegistered**: When a new customer account is created
- **CustomerPreferencesChanged**: When a customer updates their preferences
- **CustomerStatusChanged**: When a customer's status changes (premium, restricted, etc.)

### Sales Aggregate Events
- **QuoteRequested**: When a customer requests a shipping quote
- **QuoteProvided**: When a price quote is generated
- **ContractCreated**: When a shipping contract is established
- **ContractAmended**: When a contract terms are modified

## Event Structure Guidelines

Although we're not implementing domain events immediately, the following structure guidelines will help establish a consistent approach when implementation begins:

### Base Event Structure
All domain events should include:
- **Event ID**: Unique identifier for the event
- **Event Type**: The type of event (can be derived from class name)
- **Timestamp**: When the event occurred
- **Version**: Schema version for evolution support
- **Correlation ID**: For tracking related events
- **Aggregate ID**: Identifier of the aggregate that produced the event
- **Aggregate Type**: Type of the aggregate that produced the event

### Example Event Schema (Conceptual)

```json
{
  "eventId": "e7f4480c-a08f-4c4a-83c7-5271ae26d761",
  "eventType": "CargoBooked",
  "timestamp": "2025-04-11T10:15:30Z",
  "version": "1.0",
  "correlationId": "cor-123456",
  "aggregateId": "ABC-123-XYZ",
  "aggregateType": "Cargo",
  "data": {
    "trackingId": "ABC-123-XYZ",
    "origin": "USNYC",
    "destination": "NLRTM",
    "arrivalDeadline": "2025-05-15T00:00:00Z",
    "customerId": "C-98765",
    "bookingDate": "2025-04-11T10:15:30Z"
  }
}
```

## Event Handling Guidelines

When implementing event handlers:

1. **Single Responsibility**: Each handler should have a single, focused purpose
2. **Idempotency**: Handlers should be idempotent to allow safe retries
3. **Error Handling**: Failed handling should not affect the publishing transaction
4. **Performance Consideration**: Critical path operations should not be blocked by handlers
5. **Transaction Boundaries**: Each handler operates in its own transaction

## Best Practices and Anti-Patterns

### Best Practices
1. **Name Events Meaningfully**: Use past tense verbs that describe what happened
2. **Include Sufficient Context**: Events should contain all information handlers need
3. **Design for Evolution**: Consider how events might change over time
4. **Use Events for Integration**: Leverage events as integration points with external systems
5. **Consider Event Order**: Some business processes may depend on event sequence
6. **Document Event Flow**: Maintain documentation about event producers and consumers
7. **Implement Audit Logging**: Use events to create comprehensive audit trails

### Anti-Patterns
1. ❌ **Command Events**: Events should represent past facts, not commands for future actions
2. ❌ **Bloated Events**: Including unnecessary data that no handler requires
3. ❌ **Event Dependency Cycles**: Circular dependencies between event handlers
4. ❌ **Transactional Coupling**: Assuming events and their handling occur in the same transaction
5. ❌ **Missing Event Documentation**: Failing to document the purpose and content of events
6. ❌ **Logic in Events**: Events should be data carriers, not contain business logic
7. ❌ **Temporal Coupling**: Building systems that depend on precise event timing or ordering

## Related ADRs
- ARD-001: Use Case Abstraction
- ARD-002: Repository Abstraction
- ARD-003: Error Handling Strategy
- ARD-004: Domain Model Design
