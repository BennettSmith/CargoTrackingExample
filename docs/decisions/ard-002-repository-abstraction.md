# ARD-002: Repository Abstraction

## Date
2025-04-11

## Status
Proposed

## Context
Our Cargo Tracking System uses Clean Architecture and Domain-Driven Design principles across both iOS (Swift) and Android (Kotlin) platforms. The repository pattern serves as a critical abstraction that mediates between the domain layer and data sources (remote APIs, local databases, etc.).

While the repository pattern is commonly used in Android development, it's less familiar to many iOS developers. Additionally, our implementation of repositories has some specific requirements to align with our DDD approach that may differ from typical Android implementations.

We need to establish clear guidelines for implementing repositories in a way that:
1. Provides a consistent pattern across both platforms
2. Clearly defines the relationship between repositories and domain aggregates
3. Ensures proper encapsulation of data access logic
4. Handles error cases using our established Result type pattern
5. Maintains domain integrity when crossing the persistence boundary

## Decision
We will adopt a repository pattern with the following characteristics:

1. Each repository interface corresponds to a single domain aggregate root
2. Repositories accept and return domain objects, not data transfer objects (DTOs)
3. All repository methods will return a Result type rather than throwing exceptions
4. All repository operations will be asynchronous
5. Repository interfaces will be defined in the domain layer, with implementations in the infrastructure layer
6. Repositories will enforce domain invariants when reconstituting domain objects

## Consequences

### Positive Consequences
- **Clean Domain Boundary**: Domain models remain pure, with no persistence concerns
- **Simplified Testing**: Domain logic can be tested without real data sources
- **Consistent Error Handling**: All data access errors are handled through the Result type
- **Aggregate Integrity**: Repositories respect aggregate boundaries, preventing anemic domain models
- **Flexibility**: Data sources can be changed without affecting domain logic
- **Separation of Concerns**: Data mapping and persistence logic is isolated from business logic
- **Transaction Management**: Repositories provide a natural boundary for transactions

### Negative Consequences
- **Learning Curve**: The pattern may be unfamiliar to some iOS developers
- **Additional Abstraction**: Adds another layer between domain logic and data sources
- **Mapping Overhead**: Requires mapping between domain objects and data source models
- **Potential Over-Engineering**: For simple CRUD operations, the pattern may seem like overkill

## Alternatives Considered

### 1. Data Access Objects (DAOs) with No Domain Mapping
We considered using DAOs that would work directly with database-centric models without mapping to domain objects. This approach would be simpler but would leak persistence concerns into the domain layer.

### 2. Active Record Pattern
We considered having domain models implement their own persistence logic (Active Record pattern). This would simplify usage but would violate Clean Architecture by mixing domain and persistence concerns.

### 3. Repository Per Use Case
We considered defining repositories around use cases rather than aggregates. This would optimize for specific flows but would lead to duplication and potential inconsistencies.

### 4. Generic Repository Interface
We considered using a generic repository interface for all entity types. This would reduce code duplication but would make it harder to express domain-specific persistence rules.

## Rationale
The repository pattern creates a clear boundary between the domain and data layers, allowing each to evolve independently. By aligning repositories with aggregate roots, we enforce proper domain modeling and ensure that persistence operations respect domain boundaries.

Our implementation differs from some common Android approaches in a few key ways:

1. **Aggregate Focus**: Unlike some Android implementations that create repositories around features or screens, our repositories are strictly aligned with domain aggregates.

2. **No LiveData or Flows in Repository Interfaces**: We keep our repository interfaces pure and platform-agnostic, returning simple Result types rather than platform-specific reactive types.

3. **Domain Objects Only**: Some Android implementations pass DTOs across the repository boundary, but we insist on mapping to domain objects within the repository to maintain the integrity of our domain model.

4. **Error Handling**: Rather than throwing exceptions (common in Android) or using nullable returns (common in iOS), we use the Result type to handle errors explicitly.

## Implications

### For Developers
- Repositories must be implemented for each aggregate root in the domain model
- Data mapping logic should be contained within repositories or dedicated mappers
- Repository methods should be asynchronous and return Result types
- Repositories should enforce domain invariants when reconstituting objects

### For Architecture
- Repository interfaces belong in the domain layer
- Repository implementations belong in the infrastructure layer
- Repositories should be injected into use cases, never accessed directly from the UI layer

## Swift/Kotlin Examples

### Swift Repository Interface Example

```swift
// Located in Domain Layer
protocol CargoRepository {
    // Find a Cargo by its tracking ID
    func findByTrackingId(_ trackingId: TrackingId) async -> Result<Cargo, DomainError>
    
    // Save a Cargo aggregate
    func save(_ cargo: Cargo) async -> Result<Void, DomainError>
    
    // Generate a new tracking ID
    func nextTrackingId() async -> Result<TrackingId, DomainError>
    
    // Find all cargos for a specific customer
    func findForCustomer(_ customerId: CustomerId) async -> Result<[Cargo], DomainError>
}
```

### Swift Repository Implementation Example

```swift
// Located in Infrastructure Layer
final class CargoRepositoryImpl: CargoRepository {
    private let apiClient: APIClient
    private let database: CoreDataStack
    private let mapper: CargoMapper
    
    init(apiClient: APIClient, database: CoreDataStack, mapper: CargoMapper) {
        self.apiClient = apiClient
        self.database = database
        self.mapper = mapper
    }
    
    func findByTrackingId(_ trackingId: TrackingId) async -> Result<Cargo, DomainError> {
        do {
            // Try to get from local cache first
            if let localEntity = try database.fetchCargoEntity(withTrackingId: trackingId.id) {
                return .success(mapper.toDomain(localEntity))
            }
            
            // If not in cache, fetch from API
            let response = try await apiClient.fetchCargo(trackingId: trackingId.id)
            
            // Save to local cache
            let entity = mapper.toEntity(response)
            try database.save(entity)
            
            // Return domain object
            return .success(mapper.toDomain(entity))
        } catch let error as APIError {
            return .failure(.repositoryError(error))
        } catch {
            return .failure(.repositoryError(error))
        }
    }
    
    func save(_ cargo: Cargo) async -> Result<Void, DomainError> {
        do {
            // Save to local database
            let entity = mapper.toEntity(cargo)
            try database.save(entity)
            
            // Sync with remote API
            let dto = mapper.toDTO(cargo)
            try await apiClient.saveCargo(dto)
            
            return .success(())
        } catch let error as APIError {
            return .failure(.repositoryError(error))
        } catch {
            return .failure(.repositoryError(error))
        }
    }
    
    func nextTrackingId() async -> Result<TrackingId, DomainError> {
        do {
            let response = try await apiClient.generateTrackingId()
            return .success(TrackingId(id: response.trackingId))
        } catch let error as APIError {
            return .failure(.repositoryError(error))
        } catch {
            return .failure(.repositoryError(error))
        }
    }
    
    func findForCustomer(_ customerId: CustomerId) async -> Result<[Cargo], DomainError> {
        do {
            let response = try await apiClient.fetchCargosForCustomer(customerId: customerId.id)
            let entities = response.map { mapper.toEntity($0) }
            
            // Save to local cache
            try database.save(entities)
            
            // Return domain objects
            return .success(entities.map { mapper.toDomain($0) })
        } catch let error as APIError {
            return .failure(.repositoryError(error))
        } catch {
            return .failure(.repositoryError(error))
        }
    }
}
```

### Kotlin Repository Interface Example

```kotlin
// Located in Domain Layer
interface CargoRepository {
    // Find a Cargo by its tracking ID
    suspend fun findByTrackingId(trackingId: TrackingId): Result<Cargo, DomainError>
    
    // Save a Cargo aggregate
    suspend fun save(cargo: Cargo): Result<Unit, DomainError>
    
    // Generate a new tracking ID
    suspend fun nextTrackingId(): Result<TrackingId, DomainError>
    
    // Find all cargos for a specific customer
    suspend fun findForCustomer(customerId: CustomerId): Result<List<Cargo>, DomainError>
}
```

### Kotlin Repository Implementation Example

```kotlin
// Located in Infrastructure Layer
class CargoRepositoryImpl(
    private val apiService: ApiService,
    private val database: AppDatabase,
    private val mapper: CargoMapper
) : CargoRepository {
    
    override suspend fun findByTrackingId(trackingId: TrackingId): Result<Cargo, DomainError> {
        return try {
            // Try to get from local cache first
            val localEntity = database.cargoDao().findByTrackingId(trackingId.id)
            
            if (localEntity != null) {
                Result.Success(mapper.toDomain(localEntity))
            } else {
                // If not in cache, fetch from API
                val response = apiService.fetchCargo(trackingId.id)
                
                // Save to local cache
                val entity = mapper.toEntity(response)
                database.cargoDao().insert(entity)
                
                // Return domain object
                Result.Success(mapper.toDomain(entity))
            }
        } catch (e: Exception) {
            Result.Failure(DomainError.RepositoryError(e))
        }
    }
    
    override suspend fun save(cargo: Cargo): Result<Unit, DomainError> {
        return try {
            // Save to local database
            val entity = mapper.toEntity(cargo)
            database.cargoDao().insert(entity)
            
            // Sync with remote API
            val dto = mapper.toDTO(cargo)
            apiService.saveCargo(dto)
            
            Result.Success(Unit)
        } catch (e: Exception) {
            Result.Failure(DomainError.RepositoryError(e))
        }
    }
    
    override suspend fun nextTrackingId(): Result<TrackingId, DomainError> {
        return try {
            val response = apiService.generateTrackingId()
            Result.Success(TrackingId(response.trackingId))
        } catch (e: Exception) {
            Result.Failure(DomainError.RepositoryError(e))
        }
    }
    
    override suspend fun findForCustomer(customerId: CustomerId): Result<List<Cargo>, DomainError> {
        return try {
            val response = apiService.fetchCargosForCustomer(customerId.id)
            val entities = response.map { mapper.toEntity(it) }
            
            // Save to local cache
            database.cargoDao().insertAll(entities)
            
            // Return domain objects
            Result.Success(entities.map { mapper.toDomain(it) })
        } catch (e: Exception) {
            Result.Failure(DomainError.RepositoryError(e))
        }
    }
}
```

## Comparison with Common Android Repository Pattern

Our repository pattern differs from common Android implementations in several key ways:

| Aspect | Common Android Pattern | Our DDD Repository Pattern |
|--------|------------------------|----------------------------|
| **Purpose** | Data access abstraction for UI components | Domain boundary enforcing aggregate persistence rules |
| **Return Types** | LiveData, Flow, or Observable for reactivity | Result type with explicit error handling |
| **Organization** | Often organized by features or screens | Strictly organized by domain aggregates |
| **Exposure** | Often exposes DTOs directly | Always maps to domain objects |
| **Scope** | May handle data for multiple entity types | Handles exactly one aggregate root type |
| **Error Handling** | Often uses exceptions | Uses Result type for explicit error handling |
| **Caching Strategy** | Often implemented within repository | May delegate to separate caching service |
| **Usage** | Directly consumed by ViewModels | Consumed by use cases |

## Best Practices and Anti-Patterns

### Best Practices
1. Align repositories with aggregate roots in the domain model
2. Map between persistence models and domain objects within the repository
3. Encapsulate all data access logic within repositories
4. Use the Result type for error handling rather than exceptions
5. Keep domain objects ignorant of persistence details
6. Use repository interfaces in the domain layer to define the contract
7. Cache data appropriately to minimize network/database access

### Anti-Patterns
1. ❌ Creating repositories around UI features rather than domain aggregates
2. ❌ Exposing persistence models (DTOs, entities) to the domain or UI layers
3. ❌ Implementing repositories that handle multiple unrelated domain concepts
4. ❌ Direct access to repositories from the UI layer, bypassing use cases
5. ❌ Embedding business logic in repositories
6. ❌ Including UI-specific concerns (e.g., pagination UI logic) in repositories
7. ❌ Using platform-specific types (LiveData, Flow) in repository interfaces

## Related ADRs
- ARD-001: Use Case Abstraction
- ARD-003: Error Handling Strategy
- ARD-004: Domain Model Design
