<!-- protonyx/game-manager:.agents/skills/dotnet-best-practices/SKILL.md | https://github.com/protonyx/game-manager/blob/HEAD/.agents/skills/dotnet-best-practices/SKILL.md -->

---
name: dotnet-best-practices
description: 'Ensure .NET/C# code meets best practices for the solution/project.'
---

# .NET/C# Best Practices

Your task is to ensure .NET/C# code in ${selection} meets the best practices specific to this solution/project. This includes:

## Documentation & Structure

- Create comprehensive XML documentation comments for all public classes, interfaces, methods, and properties
- Include parameter descriptions and return value descriptions in XML comments
- Follow the established namespace structure: {Core|Console|App|Service}.{Feature}

## Design Patterns & Architecture

- Use primary constructor syntax for dependency injection (e.g., `public class MyClass(IDependency dependency)`)
- Use interface segregation with clear naming conventions (prefix interfaces with 'I')
- Follow the Factory pattern for complex object creation.

## CQRS / MediatR Pattern

This project uses MediatR with a thin wrapper layer defined in `GameManager.Application/Contracts/`.

### Contracts (base interfaces)

| Interface | Implements | Return type |
|---|---|---|
| `ICommand` | `IRequest<UnitResult<ApplicationError>>` | void-style result |
| `ICommand<TResponse>` | `IRequest<Result<TResponse, ApplicationError>>` | typed result |
| `IQuery<TResponse>` | `IRequest<Result<TResponse, ApplicationError>>` | typed result |
| `ICommandHandler<TCommand>` | `IRequestHandler<…>` | handler for `ICommand` |
| `ICommandHandler<TCommand, TResponse>` | `IRequestHandler<…>` | handler for `ICommand<TResponse>` |
| `IQueryHandler<TQuery, TResponse>` | `IRequestHandler<…>` | handler for `IQuery<TResponse>` |

Always use these contracts — never implement `IRequest` / `IRequestHandler` directly.

### Naming & file layout

Each operation lives in its own subfolder under `GameManager.Application/Features/{Aggregate}/{Commands|Queries}/{OperationName}/`:

```
Features/Games/Commands/CreateGame/
    CreateGameCommand.cs
    CreateGameCommandHandler.cs
    CreateGameCommandValidator.cs        ← optional, only if validation needed
    CreateGameCommandResponse.cs         ← only if the command returns data
```

- Commands: `{Action}{Aggregate}Command` — e.g., `CreateGameCommand`, `EndGameCommand`
- Queries: `Get{Aggregate}Query` / `Get{Aggregate}SummaryQuery`
- Responses: `{CommandOrQuery}Response` — e.g., `CreateGameCommandResponse`, `GetGameQueryResponse`

### Result & error handling

Use `CSharpFunctionalExtensions` — **never throw exceptions for expected failures**.

```csharp
// Return an error
return GameErrors.GameNotFound(gameId);

// Return success (typed)
return new CreateGameCommandResponse(dto, game.ETag);

// Return success (void)
return UnitResult.Success<ApplicationError>();
```

Domain-specific error factories live in `GameManager.Application/Features/{Aggregate}/{Aggregate}Errors.cs` (e.g., `GameErrors`). Add new error codes there instead of creating inline `ApplicationError` calls in handlers.

`ApplicationError` factory methods and their HTTP mappings:

| Factory | HTTP status |
|---|---|
| `ApplicationError.Failure(reason)` | 400 |
| `ApplicationError.Validation(…)` | 400 (with field errors) |
| `ApplicationError.NotFound<T>(id)` | 404 |
| `ApplicationError.Authorization(reason)` | 403 |

### Validation

Add a `{Command}Validator : AbstractValidator<{Command}>` in the same folder. It is auto-registered and runs via `ValidationPipelineBehavior` before the handler is called — **do not repeat these checks inside the handler**.

```csharp
public class CreateGameCommandValidator : AbstractValidator<CreateGameCommand>
{
    public CreateGameCommandValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(GameName.MaximumLength);
        RuleForEach(x => x.Trackers).SetValidator(new CreateTrackerValidator());
    }
}
```

### Dispatching from endpoints

Endpoints use FastEndpoints. Inject `IMediator`, call `_mediator.Send(request, ct)`, then map the result:

```csharp
var result = await _mediator.Send(req, ct);
return result.IsSuccess
    ? TypedResults.CreatedAtRoute(result.Value.Game, nameof(GetGameEndpoint), new { id = result.Value.Game.Id })
    : result.Error.ToProblemDetails();   // extension in GameManager.Server/ApplicationExtensions.cs
```

### Registration

MediatR, FluentValidation, and pipeline behaviors are registered in `GameManager.Application/ApplicationServiceRegistration.cs`. No changes are needed there when adding a new command/query — handlers and validators are discovered automatically from the assembly.

## Dependency Injection & Services

- Use constructor dependency injection with null checks via `ArgumentNullException`
- Register services with appropriate lifetimes (`Singleton`, `Scoped`, `Transient`)
- Use `Microsoft.Extensions.DependencyInjection` patterns
- Implement service interfaces for testability

## Async/Await Patterns

- Use async/await for all I/O operations and long-running tasks
- Return `Task` or `Task<T>` from async methods
- Use `ConfigureAwait(false)` where appropriate
- Handle async exceptions properly

## Configuration & Settings

- Use strongly-typed configuration classes with data annotations (Options pattern)
- Implement validation attributes (`[Required]`, `[NotEmptyOrWhitespace]`)
- Use `IConfiguration` binding for settings
- Support `appsettings.json` configuration files

## Error Handling & Logging

- Use structured logging with `Microsoft.Extensions.Logging` and `ILogger<T>`
- Include scoped logging with meaningful context
- Throw specific exceptions with descriptive messages
- Use try-catch blocks for expected failure scenarios

## Performance & Security

- Use C# 14+ features and .NET 10 optimizations where applicable
- Implement proper input validation and sanitization
- Use parameterized queries for database operations
- Follow secure coding practices for AI/ML operations

## Code Quality

- Ensure SOLID principles compliance
- Avoid code duplication through base classes and utilities
- Use meaningful names that reflect domain concepts
- Keep methods focused and cohesive
- Implement proper disposal patterns for resources
