# ASP.NET Core + EF Core Code First — Complete Cheat Sheet

---

## 1. Terminal: Create Project

```bash
# Create project
dotnet new webapi -n MyProject --use-controllers
cd MyProject

# Add EF Core packages (versioned for .NET 9)
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 9.0.5
dotnet add package Microsoft.EntityFrameworkCore.Design --version 9.0.5

git remote add origin https://github.com/StarKolya/APBD_ttPrep
git push -u origin main

# Git setup
git init
dotnet new gitignore
git add .
git commit -m "init"
```

> **Note:** Only two packages are needed. `Tools` and the base `EntityFrameworkCore` package are pulled in as dependencies automatically.

---

## 2. Folder Structure

```bash
mkdir Models DTOs Exceptions Services Data

touch Models/SomeEntity.cs
touch Models/AnotherEntity.cs
touch Models/JoinEntity.cs
touch Data/DatabaseContext.cs
touch DTOs/SomeDto.cs
touch DTOs/CreateSomeDto.cs
touch Exceptions/NotFoundException.cs
touch Exceptions/ConflictException.cs
touch Services/IDbService.cs
touch Services/DbService.cs
touch Controllers/SomeController.cs
```

---

## 3. appsettings.json

```json
{
  "ConnectionStrings": {
    "Default": "Data Source=localhost, 1433; User=SA; Password=yourStrong(!)Password; Initial Catalog=MyDb; Integrated Security=False;Connect Timeout=30;Encrypt=False;Trust Server Certificate=False"
  }
}
```

---

## 4. Program.cs

```csharp
using Microsoft.EntityFrameworkCore;
using MyProject.Data;
using MyProject.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services.AddDbContext<DatabaseContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<IDbService, DbService>();

var app = builder.Build();

app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

## 5. Entity Models

### Simple entity

```csharp
// Models/Client.cs
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace MyProject.Models;

[Table("Client")]
public class Client
{
    [Key]
    public int Id { get; set; }

    [MaxLength(50)]
    public string FirstName { get; set; } = null!;

    [MaxLength(100)]
    public string LastName { get; set; } = null!;

    public ICollection<Order> Orders { get; set; } = null!;
}
```

---

### One-to-Many (Order belongs to Client; Order has a Status)

```csharp
// Models/Order.cs
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace MyProject.Models;

[Table("Order")]
public class Order
{
    [Key]
    public int Id { get; set; }

    public DateTime CreatedAt { get; set; }
    public DateTime? FulfilledAt { get; set; }   // nullable — no value yet

    // FKs — [ForeignKey] points to the navigation property name
    [ForeignKey(nameof(Client))]
    public int ClientId { get; set; }

    [ForeignKey(nameof(Status))]
    public int StatusId { get; set; }

    public Client Client { get; set; } = null!;
    public Status Status { get; set; } = null!;

    public ICollection<ProductOrder> ProductOrders { get; set; } = null!;
}
```

---

### Many-to-Many join table with extra columns

```csharp
// Models/ProductOrder.cs
using System.ComponentModel.DataAnnotations.Schema;
using Microsoft.EntityFrameworkCore;

namespace MyProject.Models;

[Table("Product_Order")]
[PrimaryKey(nameof(ProductId), nameof(OrderId))]   // composite PK via attribute — no OnModelCreating needed
public class ProductOrder
{
    [ForeignKey(nameof(Product))]
    public int ProductId { get; set; }

    [ForeignKey(nameof(Order))]
    public int OrderId { get; set; }

    public int Amount { get; set; }

    public Product Product { get; set; } = null!;
    public Order Order { get; set; } = null!;
}
```

---

### Decimal/numeric precision

```csharp
// Models/Product.cs
using Microsoft.EntityFrameworkCore;

[Table("Product")]
public class Product
{
    [Key]
    public int Id { get; set; }

    [MaxLength(50)]
    public string Name { get; set; } = null!;

    [Column(TypeName = "numeric")]
    [Precision(10, 2)]                  // use [Precision] instead of Column(TypeName="decimal(10,2)")
    public double Price { get; set; }

    public ICollection<ProductOrder> ProductOrders { get; set; } = null!;
}
```

---

## 6. DatabaseContext

```csharp
// Data/DatabaseContext.cs
using Microsoft.EntityFrameworkCore;
using MyProject.Models;

namespace MyProject.Data;

public class DatabaseContext : DbContext
{
    // One DbSet per entity / table
    public DbSet<Client> Clients { get; set; }
    public DbSet<Status> Statuses { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<ProductOrder> ProductOrders { get; set; }

    protected DatabaseContext() { }   // required parameterless constructor

    public DatabaseContext(DbContextOptions options) : base(options) { }
    //    ↑ note: DbContextOptions (non-generic) — not DbContextOptions<DatabaseContext>

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Seed data — IDs must be hardcoded
        modelBuilder.Entity<Client>().HasData(
            new Client { Id = 1, FirstName = "John", LastName = "Doe" },
            new Client { Id = 2, FirstName = "Jane", LastName = "Doe" }
        );

        modelBuilder.Entity<Status>().HasData(
            new Status { Id = 1, Name = "Created" },
            new Status { Id = 2, Name = "Ongoing" },
            new Status { Id = 3, Name = "Completed" }
        );

        modelBuilder.Entity<Product>().HasData(
            new Product { Id = 1, Name = "Apple", Price = 3.45 },
            new Product { Id = 2, Name = "Bananas", Price = 5.55 }
        );

        modelBuilder.Entity<Order>().HasData(
            new Order { Id = 1, CreatedAt = DateTime.Parse("2025-05-01"), ClientId = 1, StatusId = 3 }
        );

        modelBuilder.Entity<ProductOrder>().HasData(
            new ProductOrder { ProductId = 1, OrderId = 1, Amount = 3 }
        );
    }
}
```

> **Key difference:** Constructor takes `DbContextOptions` (non-generic), not `DbContextOptions<DatabaseContext>`. Also has a protected parameterless constructor.

---

## 7. Migrations

```bash
# Create migration (run after every model/context change)
dotnet ef migrations add InitialCreate

# Apply migration to database
dotnet ef database update

# Undo last migration (before applying to DB)
dotnet ef migrations remove

# See all migrations
dotnet ef migrations list

# Drop database and recreate from scratch
dotnet ef database drop
dotnet ef database update
```

> **Rule:** Every time you change a Model or DbContext → new migration → update.

---

## 8. DTOs

```csharp
// DTOs/OrderDto.cs — can define multiple related DTOs in one file
namespace MyProject.DTOs;

public class OrderDto
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? FulfilledAt { get; set; }
    public string Status { get; set; } = null!;
    public ClientInfoDto Client { get; set; } = null!;
    public List<OrderLineItemDto> Products { get; set; } = null!;
}

public class ClientInfoDto
{
    public string FirstName { get; set; } = null!;
    public string LastName { get; set; } = null!;
}

public class OrderLineItemDto
{
    public string Name { get; set; } = null!;
    public double Price { get; set; }
    public int Amount { get; set; }
}

// DTOs/FulfillOrderDto.cs — simple request body
namespace MyProject.DTOs;

public class FulfillOrderDto
{
    public string StatusName { get; set; } = null!;
}
```

---

## 9. Exceptions

```csharp
// Exceptions/NotFoundException.cs
namespace MyProject.Exceptions;

public class NotFoundException : Exception
{
    public NotFoundException() { }
    public NotFoundException(string? message) : base(message) { }
    public NotFoundException(string? message, Exception? innerException) : base(message, innerException) { }
}

// Exceptions/ConflictException.cs
namespace MyProject.Exceptions;

public class ConflictException : Exception
{
    public ConflictException() { }
    public ConflictException(string? message) : base(message) { }
    public ConflictException(string? message, Exception? innerException) : base(message, innerException) { }
}
```

---

## 10. Service Layer

### Interface

```csharp
// Services/IDbService.cs
using MyProject.DTOs;

namespace MyProject.Services;

public interface IDbService
{
    Task<OrderDto> GetOrderById(int orderId);
    Task FulfillOrder(int orderId, FulfillOrderDto dto);
}
```

### Implementation

```csharp
// Services/DbService.cs
using ExampleTest2.Data;
using ExampleTest2.DTOs;
using ExampleTest2.Exceptions;
using Microsoft.EntityFrameworkCore;

namespace MyProject.Services;

public class DbService : IDbService
{
    private readonly DatabaseContext _context;

    public DbService(DatabaseContext context)
    {
        _context = context;
    }

    // ─── GET BY ID ──────────────────────────────────────────────────────────

    public async Task<OrderDto> GetOrderById(int orderId)
    {
        // Project directly to DTO inside the query — no Include() needed
        var order = await _context.Orders
            .Select(e => new OrderDto
            {
                Id          = e.Id,
                CreatedAt   = e.CreatedAt,
                FulfilledAt = e.FulfilledAt,
                Status      = e.Status.Name,               // navigate to related entity inside Select
                Client = new ClientInfoDto
                {
                    FirstName = e.Client.FirstName,
                    LastName  = e.Client.LastName,
                },
                Products = e.ProductOrders.Select(po => new OrderLineItemDto
                {
                    Name   = po.Product.Name,
                    Price  = po.Product.Price,
                    Amount = po.Amount
                }).ToList()
            })
            .FirstOrDefaultAsync(e => e.Id == orderId);   // filter AFTER Select

        if (order is null)
            throw new NotFoundException();

        return order;
    }

    // ─── UPDATE WITH TRANSACTION ─────────────────────────────────────────────

    public async Task FulfillOrder(int orderId, FulfillOrderDto dto)
    {
        using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            var order = await _context.Orders.FirstOrDefaultAsync(o => o.Id == orderId);
            if (order is null)
                throw new NotFoundException("Order not found.");

            var status = await _context.Statuses.FirstOrDefaultAsync(s => s.Name == dto.StatusName);
            if (status is null)
                throw new NotFoundException("Status not found.");

            if (order.FulfilledAt != null)
                throw new ConflictException("Order already fulfilled.");

            order.StatusId    = status.Id;
            order.FulfilledAt = DateTime.Now;

            // Remove related child rows
            var relatedProducts = _context.ProductOrders.Where(po => po.OrderId == orderId);
            _context.ProductOrders.RemoveRange(relatedProducts);

            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;   // re-throw so controller can catch and return correct HTTP status
        }
    }
}
```

---

## 11. Controller — Thin, Calls Service

```csharp
// Controllers/OrdersController.cs
using Microsoft.AspNetCore.Mvc;
using MyProject.DTOs;
using MyProject.Exceptions;
using MyProject.Services;

namespace MyProject.Controllers;

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IDbService _dbService;

    public OrdersController(IDbService dbService)
    {
        _dbService = dbService;
    }

    // GET /api/orders/5
    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(int id)
    {
        try
        {
            var order = await _dbService.GetOrderById(id);
            return Ok(order);
        }
        catch (NotFoundException e)
        {
            return NotFound(e.Message);
        }
    }

    // PUT /api/orders/5/fulfill
    [HttpPut("{orderId}/fulfill")]
    public async Task<IActionResult> FulfillOrder(int orderId, FulfillOrderDto dto)
    {
        try
        {
            await _dbService.FulfillOrder(orderId, dto);
            return Ok();
        }
        catch (NotFoundException e)
        {
            return NotFound(e.Message);
        }
        catch (ConflictException e)
        {
            return Conflict(e.Message);
        }
    }
}
```

---

## 12. EF Core Query Patterns — Quick Reference

```csharp
// ── Find by PK (fastest, uses cache) — no Include support
var entity = await _context.Orders.FindAsync(id);

// ── Find one with condition
var entity = await _context.Orders.FirstOrDefaultAsync(e => e.Id == id);

// ── Get list with filter
var list = await _context.Orders
    .Where(e => e.StatusId == 1)
    .ToListAsync();

// ── Check exists (no data loaded)
var exists = await _context.Orders.AnyAsync(e => e.Id == id);

// ── Include navigation (JOIN equivalent) — use when you need the full entity
var entity = await _context.Orders
    .Include(o => o.Client)
    .Include(o => o.Status)
    .Include(o => o.ProductOrders)
        .ThenInclude(po => po.Product)
    .FirstOrDefaultAsync(o => o.Id == id);

// ── Project to DTO directly (preferred — no Include needed, DB returns only needed columns)
var dto = await _context.Orders
    .Select(o => new OrderDto
    {
        Id         = o.Id,
        Status     = o.Status.Name,            // navigate inside Select freely
        ClientName = o.Client.FirstName + " " + o.Client.LastName,
        Products   = o.ProductOrders.Select(po => new OrderLineItemDto
        {
            Name  = po.Product.Name,
            Price = po.Product.Price
        }).ToList()
    })
    .FirstOrDefaultAsync(o => o.Id == id);    // filter after Select

// ── Add new entity
_context.Orders.Add(newOrder);
await _context.SaveChangesAsync();

// ── Update entity (just change properties on tracked object, then save)
order.StatusId    = newStatus.Id;
order.FulfilledAt = DateTime.Now;
await _context.SaveChangesAsync();

// ── Delete one entity
_context.Orders.Remove(order);
await _context.SaveChangesAsync();

// ── Delete a collection
_context.ProductOrders.RemoveRange(order.ProductOrders);
await _context.SaveChangesAsync();
```

---

## 13. Transactions

Use a transaction when you need multiple DB operations to succeed or fail together:

```csharp
using var transaction = await _context.Database.BeginTransactionAsync();

try
{
    // ... make changes ...
    await _context.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;   // always re-throw so the caller gets the exception
}
```

---

## 14. Data Annotations Reference

### On Models (controls DB schema)

| Annotation | Effect |
|---|---|
| `[Key]` | Primary key |
| `[PrimaryKey(nameof(A), nameof(B))]` | Composite PK (on class, replaces HasKey in DbContext) |
| `[Required]` | NOT NULL in DB |
| `[MaxLength(100)]` | VARCHAR(100) in DB |
| `[Column(TypeName = "numeric")]` + `[Precision(10, 2)]` | Numeric with precision |
| `[Table("TableName")]` | Override table name |
| `[ForeignKey(nameof(NavProp))]` | Explicit FK — points to navigation property name |
| `[NotMapped]` | Excluded from DB |

### On DTOs (controls API validation)

| Annotation | Effect |
|---|---|
| `[Required]` | Must be present in request |
| `[StringLength(100, MinimumLength = 1)]` | Min/max string length |
| `[Range(0.01, 9999)]` | Numeric range |
| `[EmailAddress]` | Email format |
| `[MinLength(1)]` | Collection must have ≥1 item |

---

## 15. HTTP Status Summary

| Method | Success | Not found | Conflict | Bad input |
|--------|---------|-----------|----------|-----------|
| GET all | 200 + list | — | — | — |
| GET by id | 200 + dto | 404 | — | — |
| POST | 201 | 404 (FK missing) | 409 | 400 |
| PUT | 200 | 404 | 409 | 400 |
| DELETE | 204 | 404 | — | — |

```csharp
return Ok(result);                    // 200
return Created($"api/orders/{id}", null);  // 201
return NoContent();                   // 204
return BadRequest("message");         // 400
return NotFound(e.Message);           // 404
return Conflict(e.Message);           // 409
```

---

## 16. Common Mistakes to Avoid

| Mistake | Fix |
|---|---|
| Only 2 packages needed, not 4 | `SqlServer` + `Design` — Tools and base EF are pulled in automatically |
| Wrong DbContext constructor | Use `DbContextOptions` (non-generic), add protected parameterless constructor |
| Forgot `Include()` → navigation is null | Add `.Include()` and `.ThenInclude()`, or use `.Select()` projection |
| `FindAsync` doesn't support `Include` | Use `FirstOrDefaultAsync` with `.Include()` |
| Using `Include` when projecting to DTO | Inside `.Select()` you can navigate freely — no `Include` needed |
| Changed model but didn't migrate | `dotnet ef migrations add ...` + `dotnet ef database update` |
| Decimal/numeric precision lost | Use `[Column(TypeName = "numeric")]` + `[Precision(10, 2)]` |
| Composite PK needs extra config | Use `[PrimaryKey(nameof(A), nameof(B))]` attribute on the class |
| Saving children before parent exists | `SaveChangesAsync()` after parent, then add children |
| Service not registered in Program.cs | `builder.Services.AddScoped<IDbService, DbService>()` |
| Transaction not rolled back on error | Wrap in try/catch, `RollbackAsync()` in catch, always re-throw |

---

## 17. EF Core Methods — Complete Reference

### Loading data

```csharp
// Get one by PK — returns null if not found, no Include support
await _context.Orders.FindAsync(id);

// Get first match by condition — returns null if not found
await _context.Orders.FirstOrDefaultAsync(o => o.Id == id);

// Get all rows as list
await _context.Orders.ToListAsync();

// Get filtered list
await _context.Orders.Where(o => o.StatusId == 1).ToListAsync();

// Check if any row matches — returns bool
await _context.Orders.AnyAsync(o => o.Id == id);
```

---

### Joins (use when you need the full entity, not projecting to DTO)

```csharp
.Include(o => o.Client)                     // one level
.Include(o => o.ProductOrders)              // collection
    .ThenInclude(po => po.Product)          // nested
.Include(o => o.Status)                     // multiple separate joins
```

---

### Filtering / shaping

```csharp
.Where(o => o.StatusId == 1)               // filter rows

// Project to DTO — navigate freely without Include
.Select(o => new OrderDto
{
    Id     = o.Id,
    Status = o.Status.Name,
    Client = o.Client.FirstName
})
```

---

### Writing data

```csharp
_context.Orders.Add(newOrder);                      // INSERT
_context.Orders.Remove(order);                      // DELETE one
_context.ProductOrders.RemoveRange(collection);     // DELETE many

order.StatusId    = newStatus.Id;                   // UPDATE — just mutate tracked object
order.FulfilledAt = DateTime.Now;

await _context.SaveChangesAsync();                  // COMMIT — nothing hits DB until this
```

---

### Query chain order

```csharp
await _context.Entity       // start with the table
    .Include(...)           // joins (skip if using Select)
    .ThenInclude(...)       // nested joins
    .Where(...)             // filter rows
    .Select(...)            // shape output / project to DTO
    .FirstOrDefaultAsync()  // execute → one result
    .ToListAsync()          // execute → list
    .AnyAsync();            // execute → bool
```

---

### Method summary table

| Method | SQL equivalent | Returns |
|---|---|---|
| `FindAsync(id)` | `SELECT * WHERE ID = @id` | `T?` |
| `FirstOrDefaultAsync(x => ...)` | `SELECT TOP 1 WHERE ...` | `T?` |
| `ToListAsync()` | `SELECT *` | `List<T>` |
| `Where(x => ...)` | `WHERE ...` | queryable |
| `AnyAsync(x => ...)` | `WHERE EXISTS (...)` | `bool` |
| `Include(x => x.Nav)` | `LEFT JOIN ...` | queryable |
| `ThenInclude(x => x.Nav)` | `LEFT JOIN ... (nested)` | queryable |
| `Select(x => new ...)` | `SELECT col1, col2 ...` | queryable |
| `Add(entity)` | `INSERT INTO ...` | void |
| `Remove(entity)` | `DELETE WHERE ID = @id` | void |
| `RemoveRange(list)` | `DELETE (bulk)` | void |
| `SaveChangesAsync()` | `COMMIT` | void |
