---
name: Logging
description: Use when setting up logging in a new ASP.NET Core project or reviewing existing logging setup. Covers structured logging, correlation IDs, source generators, global exception handling, and OpenTelemetry.
version: 0.1.0
---

# Logging in ASP.NET Core

Source: https://dometrain.com/blog/logging-in-dotnet-best-practices/ + tech lead review additions.

## Baseline — must be in every project

### Structured logging with message templates

```csharp
logger.LogInformation("Order {OrderId} created by {UserId}", order.Id, userId);
```

Never string interpolation. Never `$"Order {order.Id}"`. Templates enable structured search in aggregators.

### [LoggerMessage] source generator

```csharp
public static partial class LogMessages
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} created by {UserId}")]
    public static partial void OrderCreated(this ILogger logger, int orderId, string userId);
}
```

Compile-time validated, zero-alloc on hot paths. Prefer over `LoggerMessage.Define` (older API).

### Correlation ID middleware

Every request must carry a correlation ID through the entire pipeline. Log it, pass it to downstream services.

### appsettings.json — suppress framework noise

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  }
}
```

### AddHttpLogging

```csharp
builder.Services.AddHttpLogging(options =>
{
    options.LoggingFields = HttpLoggingFields.RequestPath | HttpLoggingFields.ResponseStatusCode;
});
```

### Global exception handling

Not in the source article, but mandatory. Every prod project must have `UseExceptionHandler` or custom middleware that catches unhandled exceptions and logs them with request context.

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogError(exception, "Unhandled exception on {Method} {Path}", context.Request.Method, context.Request.Path);

        context.Response.StatusCode = 500;
        await context.Response.WriteAsJsonAsync(new { error = "Internal server error" });
    });
});
```

## Prod requirements

### Serilog

For any project with a log aggregator (Seq, OpenSearch, Datadog) — Serilog is practically required. Built-in JSON console formatter works but is less capable.

### OpenTelemetry

In 2025+ for new ASP.NET Core projects — OpenTelemetry is the standard for distributed tracing, not optional.
