# ARD-001: Use Case Abstraction

## Date
2025-04-11

## Status
Proposed

## Context
Our mobile applications for the Cargo Tracking System leverage Clean Architecture and Domain-Driven Design principles across both iOS (Swift) and Android (Kotlin) platforms. Use cases represent the application layer in Clean Architecture and encapsulate the core business operations of our system.

We currently have guidelines for implementing use cases, but there are two key architectural questions that need resolution:

1. Should we define a formal shared abstraction (base protocol/interface) for use cases, or simply rely on guidelines without enforcing a specific structure?
2. Should all use cases be implemented as asynchronous operations, or should we allow synchronous use cases in certain scenarios?

These decisions impact development consistency, code organization, testing strategy, and the ability to maintain clean architectural boundaries as our codebase grows.

## Decision
We will implement a shared abstraction for use cases with the following characteristics:

1. A base `UseCase` protocol/interface that defines a standard structure for all use cases
2. Two specialized variants:
   - `AsyncUseCase`: For operations requiring external data or I/O
   - `SyncUseCase`: For pure in-memory operations with no external dependencies

## Consequences

### Positive Consequences
- **Standardized Implementation**: Consistent implementation pattern across all use cases
- **Clear Intent Communication**: The distinction between sync/async use cases communicates performance expectations
- **Improved Testability**: Standard interfaces enable consistent testing approaches
- **Enhanced Static Analysis**: Type system can enforce architectural constraints
- **Better Discoverability**: New developers can quickly understand the use case pattern
- **Reduced Boilerplate**: Base abstractions can be extended with common functionality
- **Strong Architectural Boundaries**: Guarantees proper encapsulation of business rules
- **Better Error Handling**: Forces consistent error handling patterns

### Negative Consequences
- **Learning Curve**: New developers must understand the abstractions
- **Increased Initial Complexity**: Adding abstract layers introduces some upfront complexity
- **Flexibility Constraints**: May be overly rigid for some simple operations
- **Interface Segregation Principle Tension**: A single interface for all use cases may include methods not needed by all implementers

## Alternatives Considered

### 1. Guidelines Without Abstraction
We considered providing only documentation guidelines without formal abstractions. While this offers more flexibility, it leads to inconsistent implementations and weaker architectural enforcement.

### 2. Single Asynchronous Interface Only
We considered mandating that all use cases be asynchronous. This would simplify the design but would be unnecessarily complex for purely computational operations that have no reason to be async.

### 3. Custom Interface Per Use Case
We considered having no shared abstraction and letting each use case define its own interface. This would maximize flexibility but sacrifice consistency and would make it harder to enforce architectural boundaries.

## Rationale
The decision to use a formal shared abstraction with both sync and async variants balances several factors:

1. **Domain-Centric Design**: Our use cases represent the application layer in Clean Architecture and directly implement business operations from our domain model. A formal abstraction ensures these operations are properly isolated.

2. **Performance and Developer Experience**: Not all operations need to be asynchronous. Pure computational operations are more naturally expressed synchronously and forcing them into an asynchronous paradigm introduces unnecessary complexity.

3. **Consistency**: Our codebase will be more maintainable with a consistent approach to implementing use cases. The shared abstraction provides this consistency while the sync/async distinction provides appropriate flexibility.

4. **Testing**: With a clear interface, we can create mock implementations for testing without intimate knowledge of the use case implementation details.

## Implications

### For Developers
- All business operations should be implemented as use cases implementing either the `AsyncUseCase` or `SyncUseCase` interface
- View Models and other UI components should never directly access repositories or domain services
- Testing should focus on the use case interface rather than implementation details

### For Architecture
- The use case abstraction forms the key boundary between the application layer and the presentation layer
- Use cases become the primary vehicle for crossing architectural boundaries

## Swift/Kotlin Examples

### Swift Implementation

```swift
// Base Use Case Protocol
protocol UseCase {
    associatedtype RequestType
    associatedtype ResponseType
    associatedtype ErrorType: Error
}

// Synchronous Use Case
protocol SyncUseCase: UseCase {
    func execute(request: RequestType) -> Result<ResponseType, ErrorType>
}

// Asynchronous Use Case
protocol AsyncUseCase: UseCase {
    func execute(request: RequestType) async -> Result<ResponseType, ErrorType>
}

// Example: Synchronous Use Case - Validating cargo routing rules
protocol ValidateRouteSpecificationUseCase: SyncUseCase where
    RequestType == RouteSpecificationValidationRequest,
    ResponseType == RouteSpecificationValidationResponse,
    ErrorType == ValidationError {
    
    func execute(request: RouteSpecificationValidationRequest) -> Result<RouteSpecificationValidationResponse, ValidationError>
}

// Example: Asynchronous Use Case - Booking cargo
protocol BookCargoUseCase: AsyncUseCase where
    RequestType == BookCargoRequest,
    ResponseType == BookCargoResponse,
    ErrorType == DomainError {
    
    func execute(request: BookCargoRequest) async -> Result<BookCargoResponse, DomainError>
}
```

### Kotlin Implementation

```kotlin
// Base Use Case Interface
interface UseCase<RequestT, ResponseT, ErrorT>

// Synchronous Use Case
interface SyncUseCase<RequestT, ResponseT, ErrorT> : UseCase<RequestT, ResponseT, ErrorT> {
    fun execute(request: RequestT): Result<ResponseT, ErrorT>
}

// Asynchronous Use Case
interface AsyncUseCase<RequestT, ResponseT, ErrorT> : UseCase<RequestT, ResponseT, ErrorT> {
    suspend fun execute(request: RequestT): Result<ResponseT, ErrorT>
}

// Example: Synchronous Use Case - Validating cargo routing rules
interface ValidateRouteSpecificationUseCase : 
    SyncUseCase<RouteSpecificationValidationRequest, RouteSpecificationValidationResponse, ValidationError>

// Example: Asynchronous Use Case - Booking cargo
interface BookCargoUseCase : 
    AsyncUseCase<BookCargoRequest, BookCargoResponse, DomainError>
```

## Best Practices and Anti-Patterns

### Best Practices
1. Use the appropriate use case type based on the operation:
   - Use `AsyncUseCase` for operations requiring network calls, database access, or other I/O
   - Use `SyncUseCase` for pure computational operations, validations, or transformations
2. Keep use cases focused on a single responsibility
3. Implement domain logic in domain entities or services, not in use cases
4. Use cases should orchestrate domain operations but not implement domain rules
5. Return rich domain errors in the Result type, not exceptions
6. Never expose domain entities directly in response types
7. Input validation should occur at the use case boundary

### Anti-Patterns
1. ❌ Bypassing the use case abstraction and accessing repositories directly from the UI layer
2. ❌ Implementing domain rules in use cases instead of in domain entities/services
3. ❌ Using synchronous use cases for operations requiring I/O
4. ❌ Exposing mutable domain entities in response types
5. ❌ Using exceptions for flow control instead of the Result type
6. ❌ Creating use cases with side effects but no return value
7. ❌ Implementing use cases that depend on UI state

## Related ADRs
- ADR-002: Repository Abstractions
- ADR-003: Error Handling Strategy

