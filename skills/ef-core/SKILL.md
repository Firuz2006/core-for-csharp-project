---
name: EF Core
description: Use when setting up Entity Framework Core in a new ASP.NET Core project or reviewing existing EF Core setup. Covers IApplicationDbContext, migrations, seeding, and anti-patterns.
version: 0.1.0
---

# Entity Framework Core in ASP.NET Core

## IApplicationDbContext — single entry point

### Clean Architecture

Application layer defines the interface with all DbSets:

```csharp
public interface IApplicationDbContext
{
    DbSet<User> Users { get; }
    DbSet<Product> Products { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

Infrastructure layer implements it:

```csharp
public class ApplicationDbContext : DbContext, IApplicationDbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<User> Users => Set<User>();
    public DbSet<Product> Products => Set<Product>();
}
```

### Without Clean Architecture

Same rule — one `IApplicationDbContext`, one implementation. Never scatter DbContext usage without an interface.

## Anti-pattern — never use EnsureCreated

```csharp
// NEVER
await context.Database.EnsureCreatedAsync();
```

Always use migrations. `EnsureCreated` skips migration history, breaks `MigrateAsync`, cannot evolve schema.

## Database initialization — MigrateAsync + seeders

### Seeder interface

One interface for all seeders:

```csharp
public interface IDataSeeder
{
    Task SeedAsync(CancellationToken cancellationToken);
}
```

### Seeder implementations

```csharp
public class UserSeeder(IApplicationDbContext context) : IDataSeeder
{
    public async Task SeedAsync(CancellationToken cancellationToken)
    {
        // seed logic
    }
}
```

```csharp
public class ProductSeeder(IApplicationDbContext context) : IDataSeeder
{
    public async Task SeedAsync(CancellationToken cancellationToken)
    {
        // seed logic
    }
}
```

### Database initializer

```csharp
public class DatabaseInitializer(
    ApplicationDbContext context,
    IEnumerable<IDataSeeder> seeders)
{
    public async Task InitializeAsync(CancellationToken cancellationToken)
    {
        await context.Database.MigrateAsync(cancellationToken);

        foreach (IDataSeeder seeder in seeders)
        {
            await seeder.SeedAsync(cancellationToken);
        }
    }
}
```

### Registration

```csharp
builder.Services.AddScoped<IDataSeeder, UserSeeder>();
builder.Services.AddScoped<IDataSeeder, ProductSeeder>();
builder.Services.AddScoped<DatabaseInitializer>();
```

### Startup call

```csharp
using IServiceScope scope = app.Services.CreateScope();
DatabaseInitializer initializer = scope.ServiceProvider.GetRequiredService<DatabaseInitializer>();
await initializer.InitializeAsync(CancellationToken.None);
```
