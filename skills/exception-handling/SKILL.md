---
name: Exception Handling
description: Use when setting up global error handling in a new ASP.NET Core project or reviewing existing exception handling. Covers IExceptionHandler, ProblemDetails, StatusCodePages, and common anti-patterns.
version: 0.1.0
---

# Exception Handling in ASP.NET Core

Sources:
- https://www.milanjovanovic.tech/blog/global-error-handling-in-aspnetcore-from-middleware-to-modern-handlers
- https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling?view=aspnetcore-10.0

## Baseline — must be in every project

### IExceptionHandler (.NET 8+)

One handler per exception type. Returns `true` if handled, `false` to pass to next handler. Registration order = execution priority — specific handlers before catch-all.

```csharp
public class NotFoundExceptionHandler(ILogger<NotFoundExceptionHandler> logger) : IExceptionHandler
{
    public ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not NotFoundException notFound)
            return ValueTask.FromResult(false);

        logger.LogWarning(exception, "Resource not found: {Message}", notFound.Message);

        httpContext.Response.StatusCode = StatusCodes.Status404NotFound;
        return ValueTask.FromResult(true);
    }
}
```

Registration:

```csharp
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>(); // catch-all last
builder.Services.AddProblemDetails();

app.UseExceptionHandler();
```

### ProblemDetails (RFC 9457)

Standard error response format. Never return custom JSON error objects.

```csharp
builder.Services.AddProblemDetails();
```

Response:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.6.1",
  "title": "An error occurred while processing your request.",
  "status": 500,
  "traceId": "00-b644..."
}
```

Customize:

```csharp
builder.Services.AddProblemDetails(options =>
    options.CustomizeProblemDetails = ctx =>
        ctx.ProblemDetails.Extensions.Add("traceId", Activity.Current?.Id));
```

### UseStatusCodePages

Without this, 404/403 return empty body. With it — ProblemDetails format.

```csharp
app.UseStatusCodePages();
```

### Logging exceptions with request context

Always log the exception with request context, not just `ex.Message`:

```csharp
logger.LogError(exception, "Unhandled exception on {Method} {Path}",
    httpContext.Request.Method, httpContext.Request.Path);
```

## .NET 10 — SuppressDiagnosticsCallback

By default .NET 10 suppresses diagnostics (logs, metrics) for handled exceptions (when `TryHandleAsync` returns `true`). To revert to .NET 8/9 behavior:

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = context => false
});
```

## StatusCodeSelector

Map exception types to HTTP status codes directly:

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    StatusCodeSelector = ex => ex is TimeoutException
        ? StatusCodes.Status503ServiceUnavailable
        : StatusCodes.Status500InternalServerError
});
```

## Anti-patterns — never do this

### Exposing stack trace to client in production

```csharp
// NEVER
return Problem(detail: ex.ToString());
```

### Giant middleware with if/else chains

Don't write one middleware with growing if/else per exception type. Use separate `IExceptionHandler` implementations.

### Catching exceptions in every controller

Global handler exists for this. Don't repeat try/catch in every action method.

### Ignoring response-already-started

If response headers are already sent, status code cannot be changed. Exception handlers won't run. Connection will be closed.
