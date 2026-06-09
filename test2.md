# ASP.NET Core + EF Core Code First — Complete Cheat Sheet

---

## 1. Terminal: Create Project

```bash
# Create project
dotnet new webapi -n MyProject --use-controllers
cd MyProject

# Add EF Core packages
dotnet add package Microsoft.EntityFrameworkCore --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Design --version 9.0.0

# Git setup
git init
dotnet new gitignore
git add .
git commit -m "init"
```

---

## 2. Folder Structure

```bash
mkdir Models DTOs Exceptions
touch Models/SomeEntity.cs
touch Models/AnotherEntity.cs
touch Data/AppDbContext.cs
touch DTOs/SomeListDto.cs
touch DTOs/SomeDetailsDto.cs
touch DTOs/CreateSomeRequestDto.cs
touch DTOs/UpdateSomeRequestDto.cs
touch DTOs/ErrorResponseDto.cs
touch Exceptions/NotFoundException.cs
touch Exceptions/ConflictException.cs
touch Controllers/SomeController.cs
```

---

## 3. appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1433;Database=MyDb;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=True"
  }
}
```

> **LocalDB alternative:**
> `"Data Source=(localdb)\\MSSQLLocalDB;Initial Catalog=MyDb;Integrated Security=True;"`

---

## 4. Program.cs

```csharp
using Microsoft.EntityFrameworkCore;
using MyProject.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Register EF Core DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

## 5. Entity Models

### Simple entity (no relationships)

```csharp
// Models/Animal.cs
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace MyProject.Models;

[Table("Animal")]                          // optional — maps to specific table name
public class Animal
{
    [Key]
    public int Id { get; set; }            // PK — EF recognizes "Id" automatically

    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;

    public DateTime DateOfBirth { get; set; }

    public bool IsAdmitted { get; set; }

    [Column(TypeName = "decimal(10,2)")]   // for money/decimal precision
    public decimal Weight { get; set; }

    public string? Notes { get; set; }    // nullable — no [Required]
}
```

---

### One-to-Many (Animal belongs to one Owner; Owner has many Animals)

```csharp
// Models/Owner.cs
public class Owner
{
    [Key]
    public int Id { get; set; }

    [Required]
    [MaxLength(100)]
    public string FirstName { get; set; } = string.Empty;

    [Required]
    [MaxLength(100)]
    public string LastName { get; set; } = string.Empty;

    // Navigation: one Owner → many Animals
    public ICollection<Animal> Animals { get; set; } = new List<Animal>();
}

// Models/Animal.cs — add FK + navigation back to Owner
public class Animal
{
    [Key]
    public int Id { get; set; }

    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;

    // FK column
    public int OwnerId { get; set; }

    // Navigation back to parent
    [ForeignKey("OwnerId")]
    public Owner Owner { get; set; } = null!;
}
```

---

### Many-to-Many (Animal ↔ Procedure — join table with extra columns)

```csharp
// Models/Procedure.cs
public class Procedure
{
    [Key]
    public int Id { get; set; }

    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;

    [Column(TypeName = "decimal(10,2)")]
    public decimal Price { get; set; }

    // Navigation: one Procedure → many join records
    public ICollection<AnimalProcedure> AnimalProcedures { get; set; } = new List<AnimalProcedure>();
}

// Models/AnimalProcedure.cs — join table with extra data
public class AnimalProcedure
{
    // Composite PK — configured in DbContext
    public int AnimalId { get; set; }
    public int ProcedureId { get; set; }

    public DateTime Date { get; set; }
    public string? Notes { get; set; }

    // Navigation properties to both sides
    [ForeignKey("AnimalId")]
    public Animal Animal { get; set; } = null!;

    [ForeignKey("ProcedureId")]
    public Procedure Procedure { get; set; } = null!;
}
```

---

## 6. AppDbContext

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using MyProject.Models;

namespace MyProject.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // One DbSet per entity / table
    public DbSet<Animal> Animals { get; set; }
    public DbSet<Owner> Owners { get; set; }
    public DbSet<Procedure> Procedures { get; set; }
    public DbSet<AnimalProcedure> AnimalProcedures { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Composite PK for join table
        modelBuilder.Entity<AnimalProcedure>()
            .HasKey(ap => new { ap.AnimalId, ap.ProcedureId });

        // Many-to-many relationships (both sides)
        modelBuilder.Entity<AnimalProcedure>()
            .HasOne(ap => ap.Animal)
            .WithMany(a => a.AnimalProcedures)
            .HasForeignKey(ap => ap.AnimalId);

        modelBuilder.Entity<AnimalProcedure>()
            .HasOne(ap => ap.Procedure)
            .WithMany(p => p.AnimalProcedures)
            .HasForeignKey(ap => ap.ProcedureId);

        // Optional: seed data
        modelBuilder.Entity<Owner>().HasData(
            new Owner { Id = 1, FirstName = "John", LastName = "Doe" }
        );

        modelBuilder.Entity<Animal>().HasData(
            new Animal { Id = 1, Name = "Rex", OwnerId = 1, DateOfBirth = new DateTime(2020, 1, 1) }
        );
    }
}
```

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
// DTOs/SomeListDto.cs — short, for list endpoints
namespace MyProject.DTOs;
public class SomeListDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateTime Date { get; set; }
    public string Status { get; set; } = string.Empty;
}

// DTOs/SomeDetailsDto.cs — full, for get-by-id
namespace MyProject.DTOs;
public class SomeDetailsDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateTime Date { get; set; }
    public string? OptionalField { get; set; }
    public decimal Price { get; set; }
    public string RelatedEntityName { get; set; } = string.Empty;
    public List<ChildItemDto> Items { get; set; } = [];
}

// DTOs/ChildItemDto.cs — nested inside details DTO
namespace MyProject.DTOs;
public class ChildItemDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}

// DTOs/CreateSomeRequestDto.cs — POST body
using System.ComponentModel.DataAnnotations;
namespace MyProject.DTOs;
public class CreateSomeRequestDto
{
    [Required]
    public int RelatedEntityId { get; set; }

    [Required]
    [StringLength(250, MinimumLength = 1)]
    public string Name { get; set; } = string.Empty;

    [Required]
    public DateTime Date { get; set; }

    [Range(0.01, 100000.00)]
    public decimal Price { get; set; }

    [MinLength(1)]
    public List<ChildItemDto> Items { get; set; } = [];
}

// DTOs/UpdateSomeRequestDto.cs — PUT body
namespace MyProject.DTOs;
public class UpdateSomeRequestDto
{
    [Required]
    public int RelatedEntityId { get; set; }
    public DateTime Date { get; set; }
    public string Status { get; set; } = string.Empty;
    public string? OptionalField { get; set; }
}

// DTOs/ErrorResponseDto.cs
namespace MyProject.DTOs;
public class ErrorResponseDto
{
    public string Message { get; set; } = string.Empty;
}
```

---

## 9. Exceptions

```csharp
// Exceptions/NotFoundException.cs
namespace MyProject.Exceptions;
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

// Exceptions/ConflictException.cs
namespace MyProject.Exceptions;
public class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }
}
```

---

## 10. Controller — EF Core Patterns

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MyProject.Data;
using MyProject.DTOs;
using MyProject.Exceptions;
using MyProject.Models;

namespace MyProject.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AnimalsController : ControllerBase
{
    private readonly AppDbContext _context;

    public AnimalsController(AppDbContext context)
    {
        _context = context;
    }

    // ─── GET ALL ────────────────────────────────────────────────────────────

    // GET /api/animals?name=Rex
    [HttpGet]
    public async Task<IActionResult> GetAll([FromQuery] string? name)
    {
        var query = _context.Animals
            .Include(a => a.Owner)                          // JOIN to Owner
            .Include(a => a.AnimalProcedures)               // JOIN to join table
                .ThenInclude(ap => ap.Procedure)            // then JOIN to Procedure
            .AsQueryable();

        if (!string.IsNullOrWhiteSpace(name))
            query = query.Where(a => a.Name.Contains(name));

        var result = await query
            .Select(a => new SomeListDto
            {
                Id   = a.Id,
                Name = a.Name,
                // RelatedField = a.Owner.FirstName + " " + a.Owner.LastName
            })
            .ToListAsync();

        return Ok(result);
    }

    // ─── GET BY ID ──────────────────────────────────────────────────────────

    // GET /api/animals/5
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var animal = await _context.Animals
            .Include(a => a.Owner)
            .Include(a => a.AnimalProcedures)
                .ThenInclude(ap => ap.Procedure)
            .FirstOrDefaultAsync(a => a.Id == id);

        if (animal is null)
            return NotFound(new ErrorResponseDto { Message = $"Animal {id} not found." });

        var result = new SomeDetailsDto
        {
            Id    = animal.Id,
            Name  = animal.Name,
            // map other fields here
            Items = animal.AnimalProcedures.Select(ap => new ChildItemDto
            {
                Id    = ap.Procedure.Id,
                Name  = ap.Procedure.Name,
                Price = ap.Procedure.Price
            }).ToList()
        };

        return Ok(result);
    }

    // ─── CREATE (POST) ──────────────────────────────────────────────────────

    // POST /api/animals
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateSomeRequestDto? dto)
    {
        if (dto is null) return BadRequest("Request body is required.");

        // Validate FK exists
        var owner = await _context.Owners.FindAsync(dto.RelatedEntityId);
        if (owner is null)
            return NotFound(new ErrorResponseDto { Message = $"Owner {dto.RelatedEntityId} not found." });

        // Check for conflict
        var conflict = await _context.Animals
            .AnyAsync(a => a.OwnerId == dto.RelatedEntityId && a.Name == dto.Name);
        if (conflict)
            return Conflict(new ErrorResponseDto { Message = "Animal with this name already exists for owner." });

        // Create entity
        var animal = new Animal
        {
            Name    = dto.Name,
            OwnerId = dto.RelatedEntityId,
            Date    = dto.Date,
            Price   = dto.Price
        };

        _context.Animals.Add(animal);
        await _context.SaveChangesAsync();

        // Add child records (many-to-many join rows)
        foreach (var item in dto.Items)
        {
            var procedure = await _context.Procedures.FindAsync(item.Id);
            if (procedure is null)
                return NotFound(new ErrorResponseDto { Message = $"Procedure {item.Id} not found." });

            _context.AnimalProcedures.Add(new AnimalProcedure
            {
                AnimalId    = animal.Id,
                ProcedureId = item.Id,
                Date        = dto.Date
            });
        }

        await _context.SaveChangesAsync();

        return Created($"api/animals/{animal.Id}", null);   // 201
    }

    // ─── UPDATE (PUT) ───────────────────────────────────────────────────────

    // PUT /api/animals/5
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateSomeRequestDto? dto)
    {
        if (dto is null) return BadRequest("Request body is required.");

        var animal = await _context.Animals.FindAsync(id);
        if (animal is null)
            return NotFound(new ErrorResponseDto { Message = $"Animal {id} not found." });

        // Optional business rule
        // if (animal.Status == "Locked")
        //     return Conflict(new ErrorResponseDto { Message = "Cannot modify locked record." });

        // Update fields
        animal.OwnerId = dto.RelatedEntityId;
        animal.Date    = dto.Date;
        // animal.Status  = dto.Status;

        await _context.SaveChangesAsync();
        return Ok();   // 200
    }

    // ─── DELETE ─────────────────────────────────────────────────────────────

    // DELETE /api/animals/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var animal = await _context.Animals
            .Include(a => a.AnimalProcedures)   // include children to delete them too
            .FirstOrDefaultAsync(a => a.Id == id);

        if (animal is null)
            return NotFound(new ErrorResponseDto { Message = $"Animal {id} not found." });

        // Optional business rule
        // if (animal.Status == "Active")
        //     return Conflict(new ErrorResponseDto { Message = "Cannot delete active animal." });

        _context.Animals.Remove(animal);        // EF cascades to AnimalProcedures if configured
        await _context.SaveChangesAsync();
        return NoContent();   // 204
    }
}
```

---

## 11. EF Core Query Patterns — Quick Reference

```csharp
// ── Find by PK (fastest, uses cache)
var entity = await _context.SomeTable.FindAsync(id);

// ── Find one with condition
var entity = await _context.SomeTable.FirstOrDefaultAsync(e => e.Id == id);

// ── Get list with filter
var list = await _context.SomeTable
    .Where(e => e.Status == "Active")
    .ToListAsync();

// ── Check exists (no data loaded)
var exists = await _context.SomeTable.AnyAsync(e => e.Id == id);

// ── Include navigation (JOIN equivalent)
var entity = await _context.Animals
    .Include(a => a.Owner)                      // one level
    .Include(a => a.AnimalProcedures)           // collection
        .ThenInclude(ap => ap.Procedure)        // nested include
    .FirstOrDefaultAsync(a => a.Id == id);

// ── Projection to DTO (efficient — only fetches needed columns)
var dto = await _context.Animals
    .Where(a => a.Id == id)
    .Select(a => new SomeDetailsDto
    {
        Id   = a.Id,
        Name = a.Name,
        OwnerName = a.Owner.FirstName + " " + a.Owner.LastName
    })
    .FirstOrDefaultAsync();

// ── Add new entity
_context.SomeTable.Add(newEntity);
await _context.SaveChangesAsync();

// ── Update entity (just change properties, then save)
entity.Name = dto.Name;
await _context.SaveChangesAsync();

// ── Delete entity
_context.SomeTable.Remove(entity);
await _context.SaveChangesAsync();

// ── Null check pattern after FindAsync / FirstOrDefault
if (entity is null)
    return NotFound(new ErrorResponseDto { Message = $"Record {id} not found." });
```

---

## 12. Data Annotations Reference

### On Models (controls DB schema)

| Annotation | Effect |
|---|---|
| `[Key]` | Primary key |
| `[Required]` | NOT NULL in DB |
| `[MaxLength(100)]` | VARCHAR(100) in DB |
| `[Column(TypeName = "decimal(10,2)")]` | Exact SQL type |
| `[Table("TableName")]` | Override table name |
| `[ForeignKey("PropertyName")]` | Explicit FK mapping |
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

## 13. HTTP Status Summary

| Method | Success | Not found | Conflict | Bad input |
|--------|---------|-----------|----------|-----------|
| GET all | 200 + list | — | — | — |
| GET by id | 200 + dto | 404 | — | — |
| POST | 201 | 404 (FK missing) | 409 | 400 |
| PUT | 200 | 404 | 409 | 400 |
| DELETE | 204 | 404 | 409 | — |

```csharp
return Ok(result);                                          // 200
return Created($"api/resource/{id}", null);                 // 201
return NoContent();                                         // 204
return BadRequest("Message or object");                     // 400
return NotFound(new ErrorResponseDto { Message = "..." });  // 404
return Conflict(new ErrorResponseDto { Message = "..." });  // 409
```

---

## 14. Common Mistakes to Avoid

| Mistake | Fix |
|---|---|
| Forgot `Include()` → navigation is null | Add `.Include()` and `.ThenInclude()` |
| `FindAsync` doesn't support `Include` | Use `FirstOrDefaultAsync` with `.Include()` |
| Changed model but didn't migrate | `dotnet ef migrations add ...` + `dotnet ef database update` |
| `decimal` columns lose precision | Add `[Column(TypeName = "decimal(10,2)")]` on model |
| Many-to-many composite PK not set | Configure `HasKey(e => new { e.AId, e.BId })` in `OnModelCreating` |
| Saving children before parent exists | `SaveChangesAsync()` after parent, then add children |


---
 
## 15. EF Core Methods — Complete Reference
 
### Loading data
 
```csharp
// Get one by PK — returns null if not found
// SQL: SELECT * FROM X WHERE ID = @id
await _context.Orders.FindAsync(id);
 
// Get first match by condition — returns null if not found
// SQL: SELECT TOP 1 * FROM X WHERE ...
await _context.Orders.FirstOrDefaultAsync(o => o.Id == id);
 
// Get all rows as list
// SQL: SELECT * FROM X
await _context.Orders.ToListAsync();
 
// Get all rows matching condition as list
// SQL: SELECT * FROM X WHERE ...
await _context.Orders.Where(o => o.StatusId == 1).ToListAsync();
 
// Check if any row matches — returns bool, never null
// SQL: SELECT CASE WHEN EXISTS (SELECT 1 FROM X WHERE ...) THEN 1 ELSE 0
await _context.Orders.AnyAsync(o => o.Id == id);
```
 
---
 
### Joins (always before the final fetch)
 
```csharp
// Join one level
// SQL: LEFT JOIN ProductOrders ON ...
.Include(o => o.ProductOrders)
 
// Join two levels (through a join table)
// SQL: LEFT JOIN ProductOrders ON ... LEFT JOIN Products ON ...
.Include(o => o.ProductOrders)
    .ThenInclude(po => po.Product)
 
// Multiple separate joins
.Include(o => o.Client)
.Include(o => o.Status)
.Include(o => o.ProductOrders)
    .ThenInclude(po => po.Product)
```
 
---
 
### Filtering / shaping (always before the final fetch)
 
```csharp
// Filter rows
// SQL: WHERE ...
.Where(o => o.StatusId == 1)
 
// Map to DTO directly in the query (efficient — DB only returns needed columns)
// SQL: SELECT o.Id, o.CreatedAt, c.FirstName ...
.Select(o => new OrderDto
{
    Id         = o.Id,
    CreatedAt  = o.CreatedAt,
    ClientName = o.Client.FirstName   // works inside Select without Include
})
```
 
---
 
### Writing data
 
```csharp
// INSERT — add new entity
// SQL: INSERT INTO X VALUES (...)
_context.Orders.Add(newOrder);
 
// DELETE — remove one entity
// SQL: DELETE FROM X WHERE ID = @id
_context.Orders.Remove(order);
 
// DELETE — remove a whole collection at once
// SQL: DELETE FROM X WHERE ... (multiple rows)
_context.ProductOrders.RemoveRange(order.ProductOrders);
 
// UPDATE — just change properties on a tracked object, no extra call needed
// SQL: UPDATE X SET ... WHERE ID = @id
order.StatusId    = newStatus.Id;
order.FulfilledAt = DateTime.Now;
 
// Commit — NOTHING hits the DB until this is called
await _context.SaveChangesAsync();
```
 
---
 
### The order they go in
 
```csharp
await _context.Entity       // start with the table
    .Include(...)           // joins — add related tables
    .ThenInclude(...)       // nested joins
    .Where(...)             // filter rows
    .Select(...)            // shape output (optional)
    .FirstOrDefaultAsync()  // execute — pick ONE of these:
    .ToListAsync()          //   → list of results
    .AnyAsync();            //   → just true/false
```
 
> `SaveChangesAsync()` always stands alone at the end after all your changes.
 
---
 
### How change tracking works
 
EF watches every object loaded through `_context`. You never need to "send it back":
 
```csharp
// 1. Load — EF starts tracking this object
var order = await _context.Orders.FirstOrDefaultAsync(o => o.Id == id);
 
// 2. Mutate — EF notices the change internally
order.StatusId    = newStatus.Id;
order.FulfilledAt = DateTime.Now;
 
// 3. Commit — EF generates and runs the UPDATE automatically
await _context.SaveChangesAsync();
 
// Only needed if EF doesn't know about the object (e.g. created manually):
_context.Orders.Update(order);   // tell EF "track this"
await _context.SaveChangesAsync();
```
 
---
 
### Method summary table
 
| Method | SQL equivalent | Returns |
|---|---|---|
| `FindAsync(id)` | `SELECT * WHERE ID = @id` | `T?` |
| `FirstOrDefaultAsync(x => ...)` | `SELECT TOP 1 WHERE ...` | `T?` |
| `ToListAsync()` | `SELECT *` | `List<T>` |
| `Where(x => ...)` | `WHERE ...` | queryable (chain more) |
| `AnyAsync(x => ...)` | `WHERE EXISTS (...)` | `bool` |
| `Include(x => x.Nav)` | `LEFT JOIN ...` | queryable (chain more) |
| `ThenInclude(x => x.Nav)` | `LEFT JOIN ... (nested)` | queryable (chain more) |
| `Select(x => new ...)` | `SELECT col1, col2 ...` | queryable (chain more) |
| `Add(entity)` | `INSERT INTO ...` | void |
| `Remove(entity)` | `DELETE WHERE ID = @id` | void |
| `RemoveRange(list)` | `DELETE WHERE ... (bulk)` | void |
| `SaveChangesAsync()` | `COMMIT` (runs everything) | void |
 
