# C# Microservices Monorepo Blueprint

> A production-ready template for building .NET 9 microservices with Clean Architecture, CQRS, and event-driven communication. Adapt this blueprint with your own entity-relationship diagram.

---

## How to Use This Blueprint

1. **Replace placeholders** — Substitute `{ServiceName}`, `{EntityName}`, etc. with your actual names
2. **Map your ERD** — For each entity in your diagram, follow the Domain Layer patterns
3. **Define relationships** — Use the aggregate/entity patterns based on your ERD cardinality
4. **Create contracts** — Define integration events for cross-service communication

---

## Repository Structure

```
your-monorepo/
├── src/
│   ├── BuildingBlocks/                    # Shared libraries
│   │   ├── BuildingBlocks.Domain/         # DDD base classes
│   │   ├── BuildingBlocks.Application/    # CQRS, behaviors
│   │   ├── BuildingBlocks.Infrastructure/ # EF Core, messaging
│   │   └── BuildingBlocks.Contracts/      # Integration events
│   └── Services/
│       └── {ServiceName}/
│           ├── {ServiceName}.Domain/
│           ├── {ServiceName}.Application/
│           ├── {ServiceName}.Infrastructure/
│           └── {ServiceName}.Api/
├── tests/
│   ├── {ServiceName}.UnitTests/
│   ├── {ServiceName}.IntegrationTests/
│   └── Architecture.Tests/
├── docs/
├── deploy/
├── Directory.Build.props
├── Directory.Packages.props
├── docker-compose.yml
├── global.json
└── {SolutionName}.sln
```

---

## Configuration Files

### global.json
```json
{
  "sdk": {
    "version": "9.0.100",
    "rollForward": "latestMinor"
  }
}
```

### Directory.Build.props
```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors Condition="'$(CI)' == 'true'">true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
</Project>
```

### Directory.Packages.props (Central Package Management)
```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  
  <ItemGroup Label="Core">
    <PackageVersion Include="MediatR" Version="12.4.1" />
    <PackageVersion Include="FluentValidation" Version="11.11.0" />
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="11.11.0" />
    <PackageVersion Include="Ardalis.GuardClauses" Version="5.0.0" />
  </ItemGroup>
  
  <ItemGroup Label="Infrastructure">
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.2" />
    <PackageVersion Include="EFCore.NamingConventions" Version="9.0.0" />
    <PackageVersion Include="MassTransit" Version="8.3.5" />
    <PackageVersion Include="MassTransit.RabbitMQ" Version="8.3.5" />
    <PackageVersion Include="Microsoft.Extensions.Caching.StackExchangeRedis" Version="9.0.0" />
  </ItemGroup>
  
  <ItemGroup Label="API">
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="9.0.0" />
    <PackageVersion Include="Serilog.AspNetCore" Version="8.0.3" />
    <PackageVersion Include="Serilog.Sinks.Seq" Version="8.0.0" />
  </ItemGroup>
  
  <ItemGroup Label="Testing">
    <PackageVersion Include="xunit" Version="2.9.2" />
    <PackageVersion Include="FluentAssertions" Version="7.0.0" />
    <PackageVersion Include="NSubstitute" Version="5.3.0" />
    <PackageVersion Include="Testcontainers.PostgreSql" Version="4.1.0" />
    <PackageVersion Include="Testcontainers.RabbitMq" Version="4.1.0" />
    <PackageVersion Include="Bogus" Version="35.6.1" />
    <PackageVersion Include="NetArchTest.Rules" Version="1.3.2" />
  </ItemGroup>
</Project>
```

---

## BuildingBlocks — Shared Foundation

### Domain Abstractions

#### Entity Base Class
```csharp
namespace BuildingBlocks.Domain.Abstractions;

public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : notnull
{
    public TId Id { get; protected init; } = default!;

    protected Entity() { }
    protected Entity(TId id) => Id = id;

    public bool Equals(Entity<TId>? other) =>
        other is not null && Id.Equals(other.Id);

    public override bool Equals(object? obj) => Equals(obj as Entity<TId>);
    public override int GetHashCode() => Id.GetHashCode();
}
```

#### Aggregate Root with Domain Events
```csharp
public abstract class AggregateRoot<TId> : Entity<TId>
    where TId : notnull
{
    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void RaiseDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

#### Strongly-Typed ID (Prevents Primitive Obsession)
```csharp
public abstract record StronglyTypedId(Guid Value)
{
    public override string ToString() => Value.ToString();
}

// Usage in your service:
public sealed record {EntityName}Id(Guid Value) : StronglyTypedId(Value)
{
    public static {EntityName}Id New() => new(Guid.NewGuid());
    public static {EntityName}Id From(Guid value) => new(value);
}
```

#### Value Object Base
```csharp
public abstract class ValueObject : IEquatable<ValueObject>
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public bool Equals(ValueObject? other) =>
        other is not null && GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());

    public override int GetHashCode() =>
        GetEqualityComponents().Aggregate(0, (hash, obj) => HashCode.Combine(hash, obj));
}
```

#### Result Pattern (Explicit Error Handling)
```csharp
public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    protected Result(bool isSuccess, Error error) { IsSuccess = isSuccess; Error = error; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
    public static Result<T> Success<T>(T value) => new(value, true, Error.None);
    public static Result<T> Failure<T>(Error error) => new(default, false, error);
}

public class Result<T> : Result
{
    private readonly T? _value;
    public T Value => IsSuccess ? _value! : throw new InvalidOperationException();
    
    protected internal Result(T? value, bool isSuccess, Error error) : base(isSuccess, error) 
        => _value = value;
}

public sealed record Error(string Code, string Description)
{
    public static readonly Error None = new(string.Empty, string.Empty);
    public static readonly Error NotFound = new("Error.NotFound", "Resource not found");
}
```

### Application Abstractions

#### CQRS Interfaces
```csharp
using MediatR;

// Commands (mutate state)
public interface ICommand : IRequest<Result> { }
public interface ICommand<TResponse> : IRequest<Result<TResponse>> { }
public interface ICommandHandler<TCommand> : IRequestHandler<TCommand, Result> where TCommand : ICommand { }
public interface ICommandHandler<TCommand, TResponse> : IRequestHandler<TCommand, Result<TResponse>> 
    where TCommand : ICommand<TResponse> { }

// Queries (read-only)
public interface IQuery<TResponse> : IRequest<Result<TResponse>> { }
public interface IQueryHandler<TQuery, TResponse> : IRequestHandler<TQuery, Result<TResponse>> 
    where TQuery : IQuery<TResponse> { }
```

#### Validation Behavior (Auto-validates all requests)
```csharp
public sealed class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
    where TResponse : Result
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators) => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var failures = (await Task.WhenAll(_validators.Select(v => v.ValidateAsync(request, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .Select(f => new Error(f.PropertyName, f.ErrorMessage))
            .ToArray();

        return failures.Length != 0 
            ? (TResponse)(object)Result.Failure(new ValidationError(failures)) 
            : await next();
    }
}
```

---

## Mapping Your ERD to Domain Layer

### Step 1: Identify Aggregates vs Entities

From your ERD, determine:
- **Aggregate Roots**: Entities that are entry points (have their own repository)
- **Child Entities**: Entities that belong to an aggregate (loaded with their parent)

**Rule of thumb**: If an entity can exist independently and is queried directly → Aggregate Root

```
ERD Example:
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Project   │1─────*│ Repository  │       │   Release   │
│ (Aggregate) │       │  (Entity)   │       │  (Entity)   │
└─────────────┘       └─────────────┘       └─────────────┘
       │1                                         │*
       └──────────────────────────────────────────┘
```

### Step 2: Create Strongly-Typed IDs

For each entity in your ERD:

```csharp
// File: {ServiceName}.Domain/ValueObjects/Identifiers.cs
namespace {ServiceName}.Domain.ValueObjects;

public sealed record {Entity1}Id(Guid Value) : StronglyTypedId(Value)
{
    public static {Entity1}Id New() => new(Guid.NewGuid());
    public static {Entity1}Id From(Guid value) => new(value);
}

public sealed record {Entity2}Id(Guid Value) : StronglyTypedId(Value)
{
    public static {Entity2}Id New() => new(Guid.NewGuid());
    public static {Entity2}Id From(Guid value) => new(value);
}
```

### Step 3: Create Aggregate Root

```csharp
// File: {ServiceName}.Domain/Entities/{AggregateRoot}.cs
namespace {ServiceName}.Domain.Entities;

public sealed class {AggregateRoot} : AggregateRoot<{AggregateRoot}Id>, IAuditable
{
    // === Scalar Properties (from ERD columns) ===
    public string Name { get; private set; } = default!;
    public string? Description { get; private set; }
    public {Status}Status Status { get; private set; }

    // === Child Collections (from ERD relationships) ===
    private readonly List<{ChildEntity}> _{childEntities} = [];
    public IReadOnlyCollection<{ChildEntity}> {ChildEntities} => _{childEntities}.AsReadOnly();

    // === Audit Fields ===
    public DateTimeOffset CreatedAt { get; private set; }
    public string? CreatedBy { get; private set; }
    public DateTimeOffset? ModifiedAt { get; private set; }
    public string? ModifiedBy { get; private set; }

    // === Constructors ===
    private {AggregateRoot}() { }  // EF Core

    private {AggregateRoot}({AggregateRoot}Id id, string name, string? description) : base(id)
    {
        Name = name;
        Description = description;
        Status = {Status}Status.Active;
    }

    // === Factory Method ===
    public static {AggregateRoot} Create(string name, string? description)
    {
        Guard.Against.NullOrWhiteSpace(name, nameof(name));

        var entity = new {AggregateRoot}({AggregateRoot}Id.New(), name, description);
        entity.RaiseDomainEvent(new {AggregateRoot}CreatedDomainEvent(entity.Id, name));
        return entity;
    }

    // === Business Methods (encapsulate behavior) ===
    public Result<{ChildEntity}> Add{ChildEntity}(/* params from ERD */)
    {
        // Invariant checking
        if (_{childEntities}.Any(x => x.SomeProperty == someValue))
            return Result.Failure<{ChildEntity}>({AggregateRoot}Errors.{ChildEntity}AlreadyExists);

        var child = {ChildEntity}.Create(Id, /* params */);
        _{childEntities}.Add(child);

        RaiseDomainEvent(new {ChildEntity}AddedDomainEvent(child.Id, Id));
        return Result.Success(child);
    }

    public void Update(string name, string? description)
    {
        Name = Guard.Against.NullOrWhiteSpace(name);
        Description = description;
        RaiseDomainEvent(new {AggregateRoot}UpdatedDomainEvent(Id, Name));
    }
}
```

### Step 4: Create Child Entities

```csharp
// File: {ServiceName}.Domain/Entities/{ChildEntity}.cs
namespace {ServiceName}.Domain.Entities;

public sealed class {ChildEntity} : Entity<{ChildEntity}Id>
{
    // Foreign key to parent aggregate
    public {AggregateRoot}Id {AggregateRoot}Id { get; private set; } = default!;

    // Properties from ERD
    public string Property1 { get; private set; } = default!;
    public string Property2 { get; private set; } = default!;

    private {ChildEntity}() { }

    private {ChildEntity}({ChildEntity}Id id, {AggregateRoot}Id parentId, string prop1, string prop2) : base(id)
    {
        {AggregateRoot}Id = parentId;
        Property1 = prop1;
        Property2 = prop2;
    }

    // Internal factory (only parent aggregate can create)
    internal static {ChildEntity} Create({AggregateRoot}Id parentId, string prop1, string prop2)
    {
        return new {ChildEntity}({ChildEntity}Id.New(), parentId, prop1, prop2);
    }
}
```

### Step 5: Define Domain Errors

```csharp
// File: {ServiceName}.Domain/Errors/{AggregateRoot}Errors.cs
namespace {ServiceName}.Domain.Errors;

public static class {AggregateRoot}Errors
{
    public static Error NotFound({AggregateRoot}Id id) =>
        new($"{AggregateRoot}.NotFound", $"{AggregateRoot} with ID {id} was not found");

    public static Error {ChildEntity}AlreadyExists =>
        new($"{AggregateRoot}.{ChildEntity}AlreadyExists", "{ChildEntity} already exists");

    public static Error InvalidState(string reason) =>
        new($"{AggregateRoot}.InvalidState", reason);
}
```

### Step 6: Define Domain Events

```csharp
// File: {ServiceName}.Domain/Events/{AggregateRoot}Events.cs
namespace {ServiceName}.Domain.Events;

public sealed record {AggregateRoot}CreatedDomainEvent(
    {AggregateRoot}Id {AggregateRoot}Id,
    string Name) : DomainEvent;

public sealed record {AggregateRoot}UpdatedDomainEvent(
    {AggregateRoot}Id {AggregateRoot}Id,
    string Name) : DomainEvent;

public sealed record {ChildEntity}AddedDomainEvent(
    {ChildEntity}Id {ChildEntity}Id,
    {AggregateRoot}Id {AggregateRoot}Id) : DomainEvent;
```

---

## Application Layer Patterns

### Command Example

```csharp
// File: {ServiceName}.Application/{Feature}/Commands/Create{Entity}Command.cs

// Command
public sealed record Create{Entity}Command(
    string Name,
    string? Description) : ICommand<{Entity}Id>;

// Validator
public sealed class Create{Entity}CommandValidator : AbstractValidator<Create{Entity}Command>
{
    public Create{Entity}CommandValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Description).MaximumLength(2000).When(x => x.Description is not null);
    }
}

// Handler
public sealed class Create{Entity}CommandHandler : ICommandHandler<Create{Entity}Command, {Entity}Id>
{
    private readonly I{Entity}Repository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public Create{Entity}CommandHandler(I{Entity}Repository repository, IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Result<{Entity}Id>> Handle(Create{Entity}Command command, CancellationToken ct)
    {
        var entity = {Entity}.Create(command.Name, command.Description);
        _repository.Add(entity);
        await _unitOfWork.SaveChangesAsync(ct);
        return entity.Id;
    }
}
```

### Query Example

```csharp
// File: {ServiceName}.Application/{Feature}/Queries/Get{Entity}ByIdQuery.cs

public sealed record Get{Entity}ByIdQuery({Entity}Id Id) : IQuery<{Entity}Dto>;

public sealed record {Entity}Dto
{
    public required Guid Id { get; init; }
    public required string Name { get; init; }
    public string? Description { get; init; }
    public required IReadOnlyList<{ChildEntity}Dto> {ChildEntities} { get; init; }
}

public sealed class Get{Entity}ByIdQueryHandler : IQueryHandler<Get{Entity}ByIdQuery, {Entity}Dto>
{
    private readonly {ServiceName}DbContext _context;

    public async Task<Result<{Entity}Dto>> Handle(Get{Entity}ByIdQuery query, CancellationToken ct)
    {
        var dto = await _context.{Entities}
            .AsNoTracking()
            .Where(e => e.Id == query.Id)
            .Select(e => new {Entity}Dto { /* projection */ })
            .FirstOrDefaultAsync(ct);

        return dto is null ? Result.Failure<{Entity}Dto>(Error.NotFound) : dto;
    }
}
```

---

## Integration Events (Cross-Service Communication)

```csharp
// File: BuildingBlocks.Contracts/{ServiceName}/{Entity}Events.cs
namespace BuildingBlocks.Contracts.{ServiceName};

public interface IIntegrationEvent
{
    Guid EventId { get; }
    DateTimeOffset OccurredOn { get; }
}

public sealed record {Entity}CreatedEvent : IIntegrationEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTimeOffset OccurredOn { get; init; } = DateTimeOffset.UtcNow;
    
    public required Guid {Entity}Id { get; init; }
    public required string Name { get; init; }
    // Add fields that other services need
}

public sealed record {Entity}UpdatedEvent : IIntegrationEvent { /* ... */ }
public sealed record {Entity}DeletedEvent : IIntegrationEvent { /* ... */ }
```

---

## Infrastructure Layer

### EF Core Configuration

```csharp
// File: {ServiceName}.Infrastructure/Persistence/Configurations/{Entity}Configuration.cs
public class {Entity}Configuration : IEntityTypeConfiguration<{Entity}>
{
    public void Configure(EntityTypeBuilder<{Entity}> builder)
    {
        builder.ToTable("{entities}");  // snake_case table name

        builder.HasKey(e => e.Id);

        // Strongly-typed ID conversion
        builder.Property(e => e.Id)
            .HasConversion(id => id.Value, value => {Entity}Id.From(value));

        // Columns from ERD
        builder.Property(e => e.Name).HasMaxLength(200).IsRequired();
        builder.Property(e => e.Description).HasMaxLength(2000);

        // Relationships from ERD
        builder.HasMany(e => e.{ChildEntities})
            .WithOne()
            .HasForeignKey(c => c.{Entity}Id)
            .OnDelete(DeleteBehavior.Cascade);

        // Value objects (owned types)
        builder.OwnsOne(e => e.SomeValueObject, vo =>
        {
            vo.Property(v => v.Value).HasColumnName("some_value");
        });

        // Soft delete filter
        builder.HasQueryFilter(e => !e.IsDeleted);
    }
}
```

### Repository Implementation

```csharp
// File: {ServiceName}.Infrastructure/Persistence/Repositories/{Entity}Repository.cs
public sealed class {Entity}Repository : Repository<{Entity}, {Entity}Id, {ServiceName}DbContext>, I{Entity}Repository
{
    public {Entity}Repository({ServiceName}DbContext context) : base(context) { }

    public async Task<{Entity}?> GetWithChildrenAsync({Entity}Id id, CancellationToken ct)
    {
        return await DbSet
            .Include(e => e.{ChildEntities})
            .FirstOrDefaultAsync(e => e.Id == id, ct);
    }
}
```

---

## Docker Compose Template

```yaml
version: '3.8'

services:
  postgres-{service1}:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: {service1}_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-{service1}-data:/var/lib/postgresql/data

  postgres-{service2}:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: {service2}_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5433:5432"
    volumes:
      - postgres-{service2}-data:/var/lib/postgresql/data

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  seq:
    image: datalust/seq:latest
    environment:
      ACCEPT_EULA: "Y"
    ports:
      - "5341:80"

volumes:
  postgres-{service1}-data:
  postgres-{service2}-data:
  rabbitmq-data:
```

---

## Checklist: From ERD to Running Microservices

### Per Entity in Your ERD:

- [ ] Create `{Entity}Id` strongly-typed identifier
- [ ] Create entity class (AggregateRoot or Entity)
- [ ] Define domain errors for the entity
- [ ] Define domain events for state changes
- [ ] Create EF Core configuration
- [ ] Add DbSet to DbContext

### Per Aggregate Root:

- [ ] Create repository interface in Application layer
- [ ] Create repository implementation in Infrastructure layer
- [ ] Create CRUD commands (Create, Update, Delete)
- [ ] Create queries (GetById, List, Search)
- [ ] Create validators for commands
- [ ] Create endpoint group in API layer

### Per Service Boundary:

- [ ] Define integration events in BuildingBlocks.Contracts
- [ ] Create domain event handlers that publish integration events
- [ ] Create MassTransit consumers for events from other services
- [ ] Configure MassTransit in DependencyInjection

### Infrastructure:

- [ ] Add service to docker-compose.yml
- [ ] Configure connection strings in appsettings.json
- [ ] Create initial EF Core migration
- [ ] Add health checks for dependencies

---

## Quick Reference: Clean Architecture Rules

| Layer | Can Reference | Cannot Reference |
|-------|---------------|------------------|
| Domain | Nothing (maybe BuildingBlocks.Domain) | Application, Infrastructure, API |
| Application | Domain | Infrastructure, API |
| Infrastructure | Domain, Application | API |
| API | All layers | - |

**Key Principles:**
1. Dependencies point inward (toward Domain)
2. Domain has no dependencies on frameworks
3. Application defines interfaces, Infrastructure implements
4. Services communicate via message contracts only

---

*Adapt this blueprint to your ERD. Replace all `{placeholders}` with your actual entity names and properties.*