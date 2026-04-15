---
name: HttpClient Usage
description: Use when adding HTTP calls to an ASP.NET Core project, registering HttpClient in DI, or reviewing existing HttpClient usage. Covers IHttpClientFactory, typed clients, resilience, and common pitfalls.
version: 0.1.0
---

# HttpClient in ASP.NET Core

Source: https://www.milanjovanovic.tech/blog/the-right-way-to-use-httpclient-in-dotnet + tech lead review additions.

## Anti-pattern — never do this

```csharp
// Port exhaustion + DNS caching issues
using var client = new HttpClient();
var response = await client.GetAsync("https://api.example.com");
```

`new HttpClient()` in `using` = port exhaustion under load. Never.

## Correct approach — IHttpClientFactory

Progression: IHttpClientFactory → named clients → typed clients.

### Basic factory

```csharp
builder.Services.AddHttpClient();
```

```csharp
public class MyService(IHttpClientFactory httpClientFactory)
{
    public async Task CallApiAsync(CancellationToken cancellationToken)
    {
        HttpClient client = httpClientFactory.CreateClient();
        HttpResponseMessage response = await client.GetAsync("https://api.example.com", cancellationToken);
    }
}
```

### Named clients

```csharp
builder.Services.AddHttpClient("GitHub", client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
    client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json");
});
```

### Typed clients

```csharp
builder.Services.AddHttpClient<GitHubClient>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
});
```

## Pitfall — typed client in singleton

Typed clients are registered as transient by default. Injecting a typed client into a singleton service = the HttpClient instance lives forever = DNS changes are never picked up.

### Fix — PooledConnectionLifetime

```csharp
builder.Services.AddHttpClient<GitHubClient>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2)
});
```

## Resilience — .NET 8+

Source article is 2023, doesn't cover this. `Microsoft.Extensions.Http.Resilience` gives retry + circuit breaker + timeout in one line:

```csharp
builder.Services.AddHttpClient<GitHubClient>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
})
.AddStandardResilienceHandler();
```

No manual Polly config needed.

## CancellationToken — mandatory

Not in source article, but required. Always propagate `CancellationToken` through all async HTTP methods:

```csharp
await client.GetAsync(url, cancellationToken);
await client.PostAsJsonAsync(url, payload, cancellationToken);
```
