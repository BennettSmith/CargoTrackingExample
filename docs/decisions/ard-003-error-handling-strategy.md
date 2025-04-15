# ARD-003: Error Handling Strategy

## Date
2025-04-11

## Status
Proposed

## Context
Our Cargo Tracking System implements Clean Architecture and Domain-Driven Design principles across iOS (Swift) and Android (Kotlin) platforms. A consistent and comprehensive error handling strategy is essential to maintain architectural boundaries, provide clear feedback to users, and ensure system reliability.

We need to address several concerns in our error handling approach:

1. Distinguishing between different types of errors (input validation, domain invariant violations, technical failures)
2. Maintaining clean separation between validation at the application boundary and enforcement of domain rules
3. Ensuring errors are properly propagated through architectural layers without breaking encapsulation
4. Providing actionable error information for both users and developers
5. Handling errors consistently across Swift and Kotlin codebases

Currently, we lack standardized patterns for error handling, leading to inconsistencies and potential architectural violations. Some components use exceptions, others use nullable returns, and others use custom error types, making it difficult to reason about error flows.

## Decision
We will implement a comprehensive error handling strategy with the following key components:

1. **Result Type Pattern**: All operations that may fail will return a Result type rather than throwing exceptions or using nullable returns
2. **Tiered Error Classification**: We will distinguish between:
   - **Validation Errors**: Errors in input data format or structure
   - **Domain Errors**: Violations of domain rules or invariants
   - **Technical Errors**: System failures, network issues, etc.
3. **Value Objects for Input Validation**: Use value objects in request models to encapsulate and reuse validation logic
4. **Factory Methods for Validation**: Input validation will be performed in factory methods for request models and value objects
5. **Domain Invariants in Domain Layer**: Domain rule validation will be enforced within domain entities and aggregates
6. **Error Translation at Boundaries**: Errors will be translated when crossing architectural boundaries
7. **Error Response Models**: Standardized models for presenting errors to users

## Consequences

### Positive Consequences
- **Clear Error Separation**: Distinct handling of validation vs. domain errors
- **Explicit Error Paths**: No hidden error flows via exceptions
- **Improved Testability**: Error scenarios can be explicitly tested
- **Better User Experience**: More precise error messages for different failure types
- **Maintains Encapsulation**: Errors are properly transformed when crossing boundaries
- **Consistency**: Uniform approach across platforms and components

### Negative Consequences
- **Additional Boilerplate**: Result type and error handling add some verbosity
- **Learning Curve**: Developers must understand the different error categories
- **Mapping Overhead**: Requires mapping between error types at boundaries
- **Potential Duplication**: Some validation logic may need to be repeated in UI and request models

## Alternatives Considered

### 1. Exception-Based Approach
We considered using exceptions for all error handling. This would be more familiar to some developers but would make error flows implicit and harder to trace, especially in Kotlin where exceptions are unchecked.

### 2. Nullable Return Values
We considered using nullable return values (null indicating error) with separate error reporting. This would be simpler but would not provide contextual information about the nature of errors.

### 3. Unified Error Type
We considered using a single error type hierarchy for all errors. This would simplify error handling but would blur the distinction between different kinds of errors.

### 4. Reactive Error Handling
We considered using reactive programming patterns (RxSwift/RxJava) for error propagation. This would work well with asynchronous flows but would add significant complexity.

## Rationale
Our error handling strategy is designed to maintain clean architectural boundaries while providing detailed error information. By separating validation errors from domain errors, we ensure that:

1. Input validation occurs at the system boundary before domain logic is invoked
2. Domain rules are encapsulated within the domain layer
3. Technical errors are handled appropriately at infrastructure boundaries

The Result type pattern makes error handling explicit and ensures developers consider error cases. 

Using value objects within request models strengthens our design by:
1. Encapsulating validation logic where it belongs (in the value objects)
2. Reusing the same validation rules consistently across the application
3. Providing richer type information for request models
4. Creating a clear progression from raw input to validated domain values

By using factory methods for validation, we guarantee that invalid inputs cannot enter the system, while domain entities enforce their own invariants.

## Implications

### For Developers
- Use Result types for all operations that may fail
- Implement factory methods with validation for request models and value objects
- Never throw exceptions across architectural boundaries
- Translate errors when crossing layer boundaries

### For Architecture
- Repository interfaces will return domain errors, not technical errors
- Use cases will propagate domain errors but translate validation errors
- Presenters will transform all errors into user-friendly messages

## Swift/Kotlin Examples

### Error Types

#### Swift Error Types

```swift
// Base error protocol
protocol AppError: Error {
    var message: String { get }
    var code: String { get }
}

// Validation errors
struct ValidationError: AppError {
    let fieldErrors: [String: String]
    var message: String {
        return fieldErrors.map { "\($0.key): \($0.value)" }.joined(separator: ", ")
    }
    var code: String { return "validation_error" }
}

// Domain errors
enum DomainError: AppError {
    case entityNotFound(id: String, type: String)
    case invalidOperation(reason: String)
    case businessRuleViolation(rule: String, details: String)
    case concurrencyConflict(entity: String)
    case unauthorized
    case repositoryError(Error)
    
    var message: String {
        switch self {
        case .entityNotFound(let id, let type):
            return "Could not find \(type) with ID \(id)"
        case .invalidOperation(let reason):
            return "Operation cannot be performed: \(reason)"
        case .businessRuleViolation(let rule, let details):
            return "Business rule '\(rule)' violated: \(details)"
        case .concurrencyConflict(let entity):
            return "Concurrency conflict detected for \(entity)"
        case .unauthorized:
            return "Not authorized to perform this operation"
        case .repositoryError(let error):
            return "Repository error: \(error.localizedDescription)"
        }
    }
    
    var code: String {
        switch self {
        case .entityNotFound: return "entity_not_found"
        case .invalidOperation: return "invalid_operation"
        case .businessRuleViolation: return "business_rule_violation"
        case .concurrencyConflict: return "concurrency_conflict"
        case .unauthorized: return "unauthorized"
        case .repositoryError: return "repository_error"
        }
    }
}

// Technical errors
enum TechnicalError: AppError {
    case networkError(Error)
    case databaseError(Error)
    case serializationError(Error)
    case unexpectedError(Error)
    
    var message: String {
        switch self {
        case .networkError(let error):
            return "Network error: \(error.localizedDescription)"
        case .databaseError(let error):
            return "Database error: \(error.localizedDescription)"
        case .serializationError(let error):
            return "Serialization error: \(error.localizedDescription)"
        case .unexpectedError(let error):
            return "Unexpected error: \(error.localizedDescription)"
        }
    }
    
    var code: String {
        switch self {
        case .networkError: return "network_error"
        case .databaseError: return "database_error"
        case .serializationError: return "serialization_error"
        case .unexpectedError: return "unexpected_error"
        }
    }
}
```

#### Kotlin Error Types

```kotlin
// Base error interface
interface AppError {
    val message: String
    val code: String
}

// Validation errors
data class ValidationError(
    val fieldErrors: Map<String, String>
) : AppError {
    override val message: String
        get() = fieldErrors.entries.joinToString(", ") { "${it.key}: ${it.value}" }
    override val code: String = "validation_error"
}

// Domain errors
sealed class DomainError : AppError {
    data class EntityNotFound(val id: String, val type: String) : DomainError()
    data class InvalidOperation(val reason: String) : DomainError()
    data class BusinessRuleViolation(val rule: String, val details: String) : DomainError()
    data class ConcurrencyConflict(val entity: String) : DomainError()
    object Unauthorized : DomainError()
    data class RepositoryError(val underlyingError: Throwable) : DomainError()
    
    override val message: String
        get() = when (this) {
            is EntityNotFound -> "Could not find $type with ID $id"
            is InvalidOperation -> "Operation cannot be performed: $reason"
            is BusinessRuleViolation -> "Business rule '$rule' violated: $details"
            is ConcurrencyConflict -> "Concurrency conflict detected for $entity"
            is Unauthorized -> "Not authorized to perform this operation"
            is RepositoryError -> "Repository error: ${underlyingError.message}"
        }
    
    override val code: String
        get() = when (this) {
            is EntityNotFound -> "entity_not_found"
            is InvalidOperation -> "invalid_operation"
            is BusinessRuleViolation -> "business_rule_violation"
            is ConcurrencyConflict -> "concurrency_conflict"
            is Unauthorized -> "unauthorized"
            is RepositoryError -> "repository_error"
        }
}

// Technical errors
sealed class TechnicalError : AppError {
    data class NetworkError(val underlyingError: Throwable) : TechnicalError()
    data class DatabaseError(val underlyingError: Throwable) : TechnicalError()
    data class SerializationError(val underlyingError: Throwable) : TechnicalError()
    data class UnexpectedError(val underlyingError: Throwable) : TechnicalError()
    
    override val message: String
        get() = when (this) {
            is NetworkError -> "Network error: ${underlyingError.message}"
            is DatabaseError -> "Database error: ${underlyingError.message}"
            is SerializationError -> "Serialization error: ${underlyingError.message}"
            is UnexpectedError -> "Unexpected error: ${underlyingError.message}"
        }
    
    override val code: String
        get() = when (this) {
            is NetworkError -> "network_error"
            is DatabaseError -> "database_error"
            is SerializationError -> "serialization_error"
            is UnexpectedError -> "unexpected_error"
        }
}
```

### Result Type

#### Swift Result Type
Swift has a built-in `Result` type that we'll use:

```swift
// Built-in Result<Success, Failure> where Failure: Error
```

#### Kotlin Result Type
Kotlin needs a custom Result type:

```kotlin
sealed class Result<out T, out E> {
    data class Success<out T>(val value: T) : Result<T, Nothing>()
    data class Failure<out E>(val error: E) : Result<Nothing, E>()
    
    fun isSuccess(): Boolean = this is Success
    fun isFailure(): Boolean = this is Failure
    
    fun getOrNull(): T? = when (this) {
        is Success -> value
        is Failure -> null
    }
    
    fun errorOrNull(): E? = when (this) {
        is Success -> null
        is Failure -> error
    }
    
    fun <R> map(transform: (T) -> R): Result<R, E> = when (this) {
        is Success -> Success(transform(value))
        is Failure -> this
    }
    
    fun <R> mapError(transform: (E) -> R): Result<T, R> = when (this) {
        is Success -> this
        is Failure -> Failure(transform(error))
    }
}
```

### Input Validation with Value Objects

#### Swift Value Object with Validation

```swift
struct TrackingId {
    let id: String
    
    private init(id: String) {
        self.id = id
    }
    
    // Factory method with validation
    static func create(id: String) -> Result<TrackingId, ValidationError> {
        // Validate format (e.g., ABC-123-XYZ)
        let pattern = "^[A-Z]{3}-\\d{3}-[A-Z]{3}$"
        guard let regex = try? NSRegularExpression(pattern: pattern) else {
            return .failure(ValidationError(fieldErrors: ["trackingId": "Invalid format"]))
        }
        
        let matches = regex.matches(in: id, range: NSRange(id.startIndex..., in: id))
        guard !matches.isEmpty else {
            return .failure(ValidationError(fieldErrors: ["trackingId": "Must be in format ABC-123-XYZ"]))
        }
        
        return .success(TrackingId(id: id))
    }
}
```

#### Kotlin Value Object with Validation

```kotlin
data class TrackingId private constructor(val id: String) {
    companion object {
        // Factory method with validation
        fun create(id: String): Result<TrackingId, ValidationError> {
            // Validate format (e.g., ABC-123-XYZ)
            val pattern = "^[A-Z]{3}-\\d{3}-[A-Z]{3}$".toRegex()
            
            if (!pattern.matches(id)) {
                return Result.Failure(ValidationError(
                    mapOf("trackingId" to "Must be in format ABC-123-XYZ")
                ))
            }
            
            return Result.Success(TrackingId(id))
        }
    }
}
```

### Use Case Request Model with Value Objects

#### Swift Request Model Using Value Objects

```swift
struct BookCargoRequest {
    // Using value objects for properties instead of primitive types
    let origin: LocationCode
    let destination: LocationCode
    let arrivalDeadline: ArrivalDate
    let customer: CustomerId
    
    // Private constructor prevents creating invalid instances
    private init(
        origin: LocationCode,
        destination: LocationCode,
        arrivalDeadline: ArrivalDate,
        customer: CustomerId
    ) {
        self.origin = origin
        self.destination = destination
        self.arrivalDeadline = arrivalDeadline
        self.customer = customer
    }
    
    // Factory method with validation
    static func create(
        originCode: String,
        destinationCode: String,
        deadlineString: String,
        customerId: String
    ) -> Result<BookCargoRequest, ValidationError> {
        // Collect all validation errors
        var fieldErrors = [String: String]()
        var origin: LocationCode? = nil
        var destination: LocationCode? = nil
        var deadline: ArrivalDate? = nil
        var customer: CustomerId? = nil
        
        // Validate origin using LocationCode value object
        let originResult = LocationCode.create(code: originCode)
        if case .failure(let error) = originResult {
            fieldErrors["origin"] = error.fieldErrors["locationCode"]
        } else if case .success(let locationCode) = originResult {
            origin = locationCode
        }
        
        // Validate destination using LocationCode value object
        let destinationResult = LocationCode.create(code: destinationCode)
        if case .failure(let error) = destinationResult {
            fieldErrors["destination"] = error.fieldErrors["locationCode"]
        } else if case .success(let locationCode) = destinationResult {
            destination = locationCode
        }
        
        // Validate deadline using ArrivalDate value object
        let deadlineResult = ArrivalDate.create(dateString: deadlineString)
        if case .failure(let error) = deadlineResult {
            fieldErrors["arrivalDeadline"] = error.fieldErrors["date"]
        } else if case .success(let date) = deadlineResult {
            deadline = date
        }
        
        // Validate customer ID using CustomerId value object
        let customerResult = CustomerId.create(id: customerId)
        if case .failure(let error) = customerResult {
            fieldErrors["customer"] = error.fieldErrors["customerId"]
        } else if case .success(let id) = customerResult {
            customer = id
        }
        
        // Cross-field validation (only if individual validations passed)
        if origin != nil && destination != nil && origin?.code == destination?.code {
            fieldErrors["destination"] = "Destination must be different from origin"
        }
        
        // Return validation error if any field failed validation
        if !fieldErrors.isEmpty {
            return .failure(ValidationError(fieldErrors: fieldErrors))
        }
        
        // All validations passed, create the request
        return .success(BookCargoRequest(
            origin: origin!,
            destination: destination!,
            arrivalDeadline: deadline!,
            customer: customer!
        ))
    }
}
```

#### Kotlin Request Model Using Value Objects

```kotlin
data class BookCargoRequest private constructor(
    // Using value objects for properties instead of primitive types
    val origin: LocationCode,
    val destination: LocationCode,
    val arrivalDeadline: ArrivalDate,
    val customer: CustomerId
) {
    companion object {
        // Factory method with validation
        fun create(
            originCode: String,
            destinationCode: String,
            deadlineString: String,
            customerId: String
        ): Result<BookCargoRequest, ValidationError> {
            val fieldErrors = mutableMapOf<String, String>()
            var origin: LocationCode? = null
            var destination: LocationCode? = null
            var deadline: ArrivalDate? = null
            var customer: CustomerId? = null
            
            // Validate origin using LocationCode value object
            when (val originResult = LocationCode.create(originCode)) {
                is Result.Success -> origin = originResult.value
                is Result.Failure -> fieldErrors["origin"] = originResult.error.fieldErrors["locationCode"] ?: "Invalid location code"
            }
            
            // Validate destination using LocationCode value object
            when (val destinationResult = LocationCode.create(destinationCode)) {
                is Result.Success -> destination = destinationResult.value
                is Result.Failure -> fieldErrors["destination"] = destinationResult.error.fieldErrors["locationCode"] ?: "Invalid location code"
            }
            
            // Validate deadline using ArrivalDate value object
            when (val deadlineResult = ArrivalDate.create(deadlineString)) {
                is Result.Success -> deadline = deadlineResult.value
                is Result.Failure -> fieldErrors["arrivalDeadline"] = deadlineResult.error.fieldErrors["date"] ?: "Invalid date format"
            }
            
            // Validate customer ID using CustomerId value object
            when (val customerResult = CustomerId.create(customerId)) {
                is Result.Success -> customer = customerResult.value
                is Result.Failure -> fieldErrors["customer"] = customerResult.error.fieldErrors["customerId"] ?: "Invalid customer ID"
            }
            
            // Cross-field validation (only if individual validations passed)
            if (origin != null && destination != null && origin.code == destination.code) {
                fieldErrors["destination"] = "Destination must be different from origin"
            }
            
            if (fieldErrors.isNotEmpty()) {
                return Result.Failure(ValidationError(fieldErrors))
            }
            
            return Result.Success(
                BookCargoRequest(
                    origin = origin!!,
                    destination = destination!!,
                    arrivalDeadline = deadline!!,
                    customer = customer!!
                )
            )
        }
    }
}
```

### Domain Invariant Validation Example

#### Swift Domain Entity with Invariants

```swift
class Cargo {
    private(set) var trackingId: TrackingId
    private(set) var origin: Location
    private(set) var routeSpecification: RouteSpecification
    private(set) var itinerary: Itinerary?
    private(set) var deliveryProgress: DeliveryProgress
    
    init(
        trackingId: TrackingId,
        origin: Location,
        routeSpecification: RouteSpecification,
        itinerary: Itinerary?,
        deliveryProgress: DeliveryProgress
    ) {
        self.trackingId = trackingId
        self.origin = origin
        self.routeSpecification = routeSpecification
        self.itinerary = itinerary
        self.deliveryProgress = deliveryProgress
    }
    
    // Domain method that enforces invariants
    func assignToRoute(itinerary: Itinerary) -> Result<Void, DomainError> {
        // Domain rule: Itinerary must satisfy route specification
        if !routeSpecification.isSatisfiedBy(itinerary) {
            return .failure(.businessRuleViolation(
                rule: "RouteSpecificationSatisfaction",
                details: "The provided itinerary does not satisfy the route specification"
            ))
        }
        
        // Domain rule: Cannot assign itinerary if cargo is already claimed
        if deliveryProgress.transportStatus == .CLAIMED {
            return .failure(.invalidOperation(reason: "Cannot assign itinerary to claimed cargo"))
        }
        
        // Apply changes if all rules pass
        self.itinerary = itinerary
        
        // Update delivery progress based on new itinerary
        self.deliveryProgress = DeliveryProgress(
            transportStatus: .NOT_RECEIVED,
            lastKnownLocation: origin,
            currentVoyage: nil,
            isOnTrack: true,
            estimatedTimeOfArrival: itinerary.finalArrivalDate
        )
        
        return .success(())
    }
    
    // More domain methods with invariant checks...
}
```

#### Kotlin Domain Entity with Invariants

```kotlin
class Cargo(
    val trackingId: TrackingId,
    val origin: Location,
    var routeSpecification: RouteSpecification,
    private var _itinerary: Itinerary? = null,
    private var _deliveryProgress: DeliveryProgress
) {
    // Expose as immutable properties
    val itinerary: Itinerary? get() = _itinerary
    val deliveryProgress: DeliveryProgress get() = _deliveryProgress
    
    // Domain method that enforces invariants
    fun assignToRoute(itinerary: Itinerary): Result<Unit, DomainError> {
        // Domain rule: Itinerary must satisfy route specification
        if (!routeSpecification.isSatisfiedBy(itinerary)) {
            return Result.Failure(DomainError.BusinessRuleViolation(
                rule = "RouteSpecificationSatisfaction",
                details = "The provided itinerary does not satisfy the route specification"
            ))
        }
        
        // Domain rule: Cannot assign itinerary if cargo is already claimed
        if (_deliveryProgress.transportStatus == TransportStatus.CLAIMED) {
            return Result.Failure(DomainError.InvalidOperation(
                reason = "Cannot assign itinerary to claimed cargo"
            ))
        }
        
        // Apply changes if all rules pass
        _itinerary = itinerary
        
        // Update delivery progress based on new itinerary
        _deliveryProgress = DeliveryProgress(
            transportStatus = TransportStatus.NOT_RECEIVED,
            lastKnownLocation = origin,
            currentVoyage = null,
            isOnTrack = true,
            estimatedTimeOfArrival = itinerary.finalArrivalDate
        )
        
        return Result.Success(Unit)
    }
    
    // More domain methods with invariant checks...
}
```

### Error Translation in Use Case

```swift
final class BookCargoInteractor: BookCargoUseCase {
    private let cargoRepository: CargoRepository
    private let locationRepository: LocationRepository
    private let customerRepository: CustomerRepository
    
    init(
        cargoRepository: CargoRepository,
        locationRepository: LocationRepository,
        customerRepository: CustomerRepository
    ) {
        self.cargoRepository = cargoRepository
        self.locationRepository = locationRepository
        self.customerRepository = customerRepository
    }
    
    func execute(request: BookCargoRequest) async -> Result<BookCargoResponse, DomainError> {
        // Use case contains orchestration logic that may involve multiple domain objects
        // and enforces transactional consistency across aggregate boundaries
        
        // If the operation fails due to domain rule violations, return domain error
        // If it fails due to repository issues, translate to domain error
        // Note: Input validation happens before this use case is called, and request contains valid value objects
        
        // Example domain error translation:
        // Notice we can directly use the LocationCode value object from the request
        let originResult = await locationRepository.findByCode(request.origin)
        guard case .success(let origin) = originResult else {
            if case .failure(let error) = originResult {
                // Translate repository error to domain error
                switch error {
                case .entityNotFound:
                    return .failure(.entityNotFound(id: request.origin.code, type: "Location"))
                default:
                    return .failure(error)
                }
            }
            return .failure(.entityNotFound(id: request.origin.code, type: "Location"))
        }
        
        // ... rest of use case implementation
    }
}
```

### Presenter Error Translation

```swift
struct BookCargoPresenter {
    func present(response: BookCargoResponse) -> BookCargoUIModel {
        return BookCargoUIModel.success(trackingId: response.trackingId)
    }
    
    func presentError(error: Error) -> BookCargoUIModel {
        // Translate errors to user-friendly messages
        
        if let validationError = error as? ValidationError {
            // Transform validation errors to field-specific UI messages
            return BookCargoUIModel.validationFailure(
                fieldErrors: validationError.fieldErrors
            )
        }
        
        if let domainError = error as? DomainError {
            // Transform domain errors to user-friendly messages
            switch domainError {
            case .entityNotFound(let id, let type):
                if type == "Location" {
                    return BookCargoUIModel.error(message: "The specified port could not be found.")
                } else if type == "Customer" {
                    return BookCargoUIModel.error(message: "Customer account not found.")
                } else {
                    return BookCargoUIModel.error(message: "Required information not found.")
                }
                
            case .businessRuleViolation(let rule, _):
                if rule == "RouteSpecificationSatisfaction" {
                    return BookCargoUIModel.error(message: "We cannot find a valid route for your shipment.")
                } else {
                    return BookCargoUIModel.error(message: "Your booking cannot be completed due to a business rule violation.")
                }
                
            default:
                return BookCargoUIModel.error(message: "An error occurred while processing your booking.")
            }
        }
        
        // For all other errors, provide a generic message
        return BookCargoUIModel.error(message: "An unexpected error occurred. Please try again later.")
    }
}
```

## Best Practices and Anti-Patterns

### Best Practices

1. **Separate Validation Types**: 
   - Use validation errors for input format/data structure issues
   - Use domain errors for business rule violations
   - Use technical errors for infrastructure failures

2. **Factory Methods for Value Objects**:
   - Implement factory methods with validation for all value objects
   - Make constructors private to prevent creating invalid instances
   - Return Result type with appropriate validation errors

3. **Value Objects in Request Models**:
   - Use value objects instead of primitive types in request models
   - Leverage value object validation logic in request model factory methods
   - Collect and consolidate validation errors from multiple value objects
   - Perform cross-field validations at the request model level

4. **Request Model Validation**:
   - Validate all inputs at the application boundary
   - Use factory methods for request models
   - Group related validation errors for better user feedback

5. **Domain Invariant Protection**:
   - Validate domain rules within domain methods
   - Return domain errors when invariants are violated
   - Make setters private and expose operations that maintain invariants

6. **Error Translation**:
   - Translate technical errors to domain errors at repository boundaries
   - Translate all errors to user-friendly messages at presentation layer
   - Preserve detailed error information for logging

7. **Result Type Usage**:
   - Use Result type for all operations that may fail
   - Handle both success and failure cases explicitly
   - Chain operations with proper error propagation

8. **Consistent Error Format**:
   - Use standardized error codes and messages
   - Include necessary context in error details
   - Structure errors to support both user feedback and troubleshooting

### Anti-Patterns

1. ❌ **Mixed Validation Types**:
   - Mixing input validation with domain validation
   - Validating domain rules outside the domain layer
   - Validating input format in domain entities

2. ❌ **Primitive Obsession in Request Models**:
   - Using strings, integers, and other primitives directly in request models
   - Duplicating validation logic across multiple request models
   - Missing the opportunity to leverage type safety with value objects

3. ❌ **Exceptions for Control Flow**:
   - Using exceptions for expected failure cases
   - Throwing exceptions across architectural boundaries
   - Catching exceptions to convert to return values

4. ❌ **Anemic Error Information**:
   - Using boolean return values without error context
   - Returning null without error explanation
   - Using generic error messages that don't help users

5. ❌ **Leaky Error Abstractions**:
   - Exposing technical errors to the UI layer
   - Leaking infrastructure details in error messages
   - Passing persistence exceptions through the domain layer

6. ❌ **Inconsistent Error Handling**:
   - Using different patterns in different components
   - Mixing nullable returns, exceptions, and result types
   - Handling errors differently on iOS and Android

7. ❌ **UI-dependent Validation**:
   - Implementing validation logic only in UI components
   - Relying on UI frameworks for validation
   - Bypassing validation in programmatic use cases

8. ❌ **Hidden Validation Logic**:
   - Scattering validation rules across multiple layers
   - Embedding validation logic in constructors without clear documentation
   - Implementing implicit validation with side effects

## Related ADRs
- ARD-001: Use Case Abstraction
- ARD-002: Repository Abstraction
- ARD-004: Domain Model Design
