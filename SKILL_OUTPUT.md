---
name: agentic-master-backend
description: ASP.NET Core 8 patterns for controllers, MediatR CQRS, dependency injection, Result<T> error handling, FluentValidation, and MassTransit background services. Use when writing HTTP endpoints, registering services, handling errors, validating requests, or building message consumers.
---

# ASP.NET Core 8 Backend Patterns

This skill covers essential patterns for building scalable backends in ASP.NET Core 8 with MediatR, MassTransit, and Serilog.

## Controllers and HTTP Endpoints

Controllers in this codebase inherit from `ControllerBase` and return `IActionResult`. Each controller method calls a MediatR handler via the mediator and maps the `Result<T>` response to an HTTP status code.

**Pattern:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    private readonly IMediator _mediator;

    public EmployeesController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Create(CreateEmployeeCommand cmd)
    {
        var result = await _mediator.Send(cmd);
        return result.IsSuccess
            ? Created(nameof(GetById), result.Value)
            : BadRequest(result.Error);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var result = await _mediator.Send(new GetEmployeeQuery(id));
        return result.IsSuccess
            ? Ok(result.Value)
            : NotFound();
    }
}
```

**Key conventions:**
- Never add business logic in controllers—delegate to handlers via `IMediator.Send()` or `IMediator.CreateStream()`
- Always map `Result<T>.IsSuccess` to `Ok()` or `Created()`; map failure cases to `BadRequest()` or `NotFound()`
- Constructor-inject `IMediator` only; let DI handle everything else
- Keep parameter names simple; MediatR uses property-matching from route data, query strings, and body

## MediatR Commands and Queries

All requests funnel through MediatR. Commands return `Result<T>` (or `Result` for void operations); queries use `AsNoTracking()` to prevent tracking overhead.

**Command with validation:**
```csharp
public record CreateEmployeeCommand(string FirstName, string LastName, string Email)
    : IRequest<Result<EmployeeDto>>;

public class CreateEmployeeHandler : IRequestHandler<CreateEmployeeCommand, Result<EmployeeDto>>
{
    private readonly IApplicationDbContext _context;

    public CreateEmployeeHandler(IApplicationDbContext context) => _context = context;

    public async Task<Result<EmployeeDto>> Handle(CreateEmployeeCommand req, CancellationToken ct)
    {
        var employee = new Employee { FirstName = req.FirstName, LastName = req.LastName, Email = req.Email };
        _context.Employees.Add(employee);
        await _context.SaveChangesAsync(ct);
        return Result.Success(new EmployeeDto(employee.Id, employee.FirstName, employee.LastName, employee.Email));
    }
}

public class CreateEmployeeValidator : AbstractValidator<CreateEmployeeCommand>
{
    public CreateEmployeeValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
    }
}
```

**Query with projection:**
```csharp
public record GetEmployeeQuery(int Id) : IRequest<Result<EmployeeDto>>;

public class GetEmployeeHandler : IRequestHandler<GetEmployeeQuery, Result<EmployeeDto>>
{
    private readonly IApplicationDbContext _context;

    public async Task<Result<EmployeeDto>> Handle(GetEmployeeQuery req, CancellationToken ct)
    {
        var dto = await _context.Employees
            .AsNoTracking()
            .Where(e => e.Id == req.Id)
            .Select(e => new EmployeeDto(e.Id, e.FirstName, e.LastName, e.Email))
            .FirstOrDefaultAsync(ct);

        return dto != null
            ? Result.Success(dto)
            : Result.Failure<EmployeeDto>("Employee not found");
    }
}
```

**Why:**
- Commands modify state; queries retrieve it
- `AsNoTracking()` avoids change tracking overhead for read-only queries
- `Result<T>` encapsulates success/failure without exceptions for flow control
- Validators run automatically via MediatR pipeline behaviors

## Dependency Injection

Each layer registers its services via an extension method on `IServiceCollection`. This keeps DI configuration organized and composable.

**Application layer registration:**
```csharp
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(DependencyInjection).Assembly));
    services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);
    return services;
}
```

**Persistence layer registration:**
```csharp
public static IServiceCollection AddPersistence(this IServiceCollection services, IConfiguration config)
{
    var connectionString = config.GetConnectionString("DefaultConnection");
    services.AddDbContext<ApplicationDbContext>(opts =>
        opts.UseSqlServer(connectionString, x => x.MigrationsAssembly("StaffManagement.Persistence")));
    services.AddScoped<IApplicationDbContext>(sp => sp.GetRequiredService<ApplicationDbContext>());
    return services;
}
```

**In Program.cs:**
```csharp
builder.Services
    .AddApplication()
    .AddPersistence(builder.Configuration);
```

**Lifetimes:**
- **Transient**: Stateless utilities, validators, mappers
- **Scoped**: DbContext, UnitOfWork, request-scoped services
- **Singleton**: Configuration, logging, caches

## Error Handling with Result<T>

Use a `Result<T>` type (or similar discriminated union) to represent success/failure without exception throwing.

```csharp
public abstract record Result
{
    public sealed record Success : Result;
    public sealed record Failure(string Error) : Result;
}

public abstract record Result<T> : Result
{
    public sealed record Success(T Value) : Result<T>;
    public sealed record Failure(string Error) : Result<T>;

    public bool IsSuccess => this is Success;
    public string? Error => (this as Failure)?.Error;
}
```

**HTTP mapping logic:**
```csharp
public static IActionResult ToHttpResult<T>(this Result<T> result) => result switch
{
    Result<T>.Success success => new OkObjectResult(success.Value),
    Result<T>.Failure failure => new BadRequestObjectResult(new { error = failure.Error }),
};
```

**Why:**
- Exceptions should signal truly exceptional bugs, not expected validation failures
- `Result<T>` composes cleanly in pipelines and async chains
- Callers always see whether an operation succeeded without unwrapping exceptions

## Background Services with MassTransit

Use MassTransit + RabbitMQ for async messaging. Define message contracts, register consumers, and handle idempotency at the consumer level.

**Message contract:**
```csharp
public record EmployeeCreatedEvent
{
    public int EmployeeId { get; init; }
    public string FirstName { get; init; }
    public string LastName { get; init; }
}
```

**Publishing from a handler:**
```csharp
public class CreateEmployeeHandler : IRequestHandler<CreateEmployeeCommand, Result<EmployeeDto>>
{
    private readonly IPublishEndpoint _publishEndpoint;

    public async Task<Result<EmployeeDto>> Handle(CreateEmployeeCommand req, CancellationToken ct)
    {
        // ... create employee ...
        await _publishEndpoint.Publish(new EmployeeCreatedEvent { EmployeeId = employee.Id, ... }, ct);
        return Result.Success(dto);
    }
}
```

**Consumer:**
```csharp
public class EmployeeCreatedConsumer : IConsumer<EmployeeCreatedEvent>
{
    private readonly ILogger<EmployeeCreatedConsumer> _logger;

    public async Task Consume(ConsumeContext<EmployeeCreatedEvent> context)
    {
        _logger.LogInformation("Employee {EmployeeId} created: {FirstName} {LastName}",
            context.Message.EmployeeId, context.Message.FirstName, context.Message.LastName);
        await Task.CompletedTask;
    }
}
```

**MassTransit registration:**
```csharp
services.AddMassTransit(x =>
{
    x.AddConsumer<EmployeeCreatedConsumer>();
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMq:Host"], h => { /* auth */ });
        cfg.ConfigureEndpoints(ctx);
    });
});
```

## Structured Logging with Serilog

Configure Serilog in Program.cs with a structured sink (e.g., Seq or console JSON output). Log contextual data as properties, never string interpolation.

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.Seq(builder.Configuration["Seq:Url"])
    .Enrich.FromLogContext()
    .Enrich.WithCorrelationId()
    .CreateLogger();

builder.Host.UseSerilog();
```

**In handlers:**
```csharp
_logger.LogInformation("Employee created: {@Employee}", new { employee.Id, employee.Email });
_logger.LogError(ex, "Failed to process employee {EmployeeId}", employeeId);
```

## Validation

Use FluentValidation validators attached to command/query records. MediatR pipelines run validators automatically before handlers.

**Key rules:**
- One validator per request type
- Define validators in the Application layer near handlers
- Register with `AddValidatorsFromAssembly()` in DI setup
- Return validation errors via `Result.Failure()` in a fallback handler, or let MediatR pipeline behaviors convert them

```csharp
public class CreateEmployeeValidator : AbstractValidator<CreateEmployeeCommand>
{
    public CreateEmployeeValidator(IApplicationDbContext context)
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Email must be valid")
            .MustAsync(async (email, ct) => !await context.Employees.AnyAsync(e => e.Email == email, ct))
            .WithMessage("Email already in use");

        RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.LastName).NotEmpty().MaximumLength(100);
    }
}
```

## Summary

- **Controllers**: Thin wrappers that call MediatR and map `Result<T>` to HTTP
- **MediatR**: Use commands for writes, queries for reads; always return `Result<T>`
- **DI**: Register services via layer extension methods; respect lifetimes
- **Error handling**: Use `Result<T>` for expected failures; exceptions for bugs
- **Background work**: MassTransit consumers for async workflows
- **Logging**: Serilog with structured properties, no string interpolation
- **Validation**: FluentValidation on all requests, automatic pipeline integration
