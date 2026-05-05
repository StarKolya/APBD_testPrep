# ASP.NET Core + ADO.NET — Complete Reference Guide

---

## 1. Terminal: Create Project

```bash
# Create project (webapi template with controllers)
dotnet new webapi -n Task6 --use-controllers

# Enter project folder
cd Task6

# Add required packages
dotnet add package Microsoft.Data.SqlClient


# Git setup
git init
dotnet new gitignore

# First commit
git add .
git commit -m "init"

# Push to GitHub (do this AFTER creating a repo on github.com)
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git branch -M main
git push -u origin main
```

### GitHub Auth
You'll need a **Personal Access Token (PAT)**:
1. Go to GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)
2. Click "Generate new token"
3. Give it `repo` scope
4. Copy the token — you'll use it as your password when git asks

Or use the GitHub CLI:
```bash
gh auth login   # follow the prompts
```

---

## 2. Create Folder Structure + Files

```bash
# Folders
mkdir DTOs Exceptions Services

# DTOs — one file per DTO (rename to match your domain)
touch DTOs/SomeListDto.cs
touch DTOs/SomeDetailsDto.cs
touch DTOs/CreateSomeRequestDto.cs
touch DTOs/UpdateSomeRequestDto.cs
touch DTOs/ErrorResponseDto.cs

# Exceptions
touch Exceptions/NotFoundException.cs
touch Exceptions/ConflictException.cs

# Services
touch Services/IDbService.cs
touch Services/DbService.cs

# Controller — rename to match your resource
touch Controllers/SomeController.cs
```

---

## 3. appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1433;Database=ClinicAdoNet;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=True"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

---

## 4. Program.cs

This is the default file — just add your service registration:

```csharp
using Task6.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddScoped<IDbService, DbService>();  // <-- add this

var app = builder.Build();

app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

## 5. C# Type Reference for DTOs

| Data | C# Type | Notes |
|------|---------|-------|
| Whole number | `int` | IDs, counts |
| Money / price | `decimal` | Never use `float`/`double` for money |
| Date + time | `DateTime` | Appointment dates |
| Date only | `DateOnly` | Birthdays etc. |
| True/false | `bool` | IsActive etc. |
| Text | `string` | Names, status, reason |
| Nullable text | `string?` | Optional fields |
| Nullable date | `DateTime?` | ReturnDate, optional dates |
| Nullable int | `int?` | Optional foreign keys |

### DTO Skeletons

```csharp
// SomeListDto.cs — short, for list endpoints
namespace YourProject.DTOs;
public class SomeListDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateTime Date { get; set; }
    public string Status { get; set; } = string.Empty;
}

// SomeDetailsDto.cs — full details, for get-by-id endpoints
namespace YourProject.DTOs;
public class SomeDetailsDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateTime Date { get; set; }
    public string Status { get; set; } = string.Empty;
    public string? OptionalField { get; set; }        // nullable — may be null
    public decimal Price { get; set; }
    public string RelatedEntityName { get; set; } = string.Empty;  // from JOIN
}

// CreateSomeRequestDto.cs — body of POST request
namespace YourProject.DTOs;
public class CreateSomeRequestDto
{
    public int RelatedEntityId { get; set; }           // foreign key
    public DateTime Date { get; set; }
    public string Reason { get; set; } = string.Empty;
    public decimal Price { get; set; }
}

// UpdateSomeRequestDto.cs — body of PUT request
namespace YourProject.DTOs;
public class UpdateSomeRequestDto
{
    public int RelatedEntityId { get; set; }
    public DateTime Date { get; set; }
    public string Status { get; set; } = string.Empty;
    public string Reason { get; set; } = string.Empty;
    public string? OptionalField { get; set; }
}

// ErrorResponseDto.cs — used in catch blocks
namespace YourProject.DTOs;
public class ErrorResponseDto
{
    public string Message { get; set; } = string.Empty;
}
```

---

## 6. Data Annotations — Validation on DTOs

Add these to **request DTOs only** (POST/PUT bodies). ASP.NET automatically returns `400 Bad Request` if validation fails — no extra code needed in the controller.

```csharp
using System.ComponentModel.DataAnnotations;

namespace YourProject.DTOs;

public class CreateSomeRequestDto
{
    [Required]
    public int RelatedEntityId { get; set; }        // must be present

    [Required]
    [StringLength(250, MinimumLength = 1)]           // 1–250 chars
    public string Reason { get; set; } = string.Empty;

    [Required]
    public DateTime Date { get; set; }

    [Range(0.01, 10000.00)]                          // numeric range
    public decimal Price { get; set; }

    [EmailAddress]                                   // must look like an email
    public string? Email { get; set; }

    [MinLength(1)]                                   // list must have at least 1 item
    public List<SomeChildDto> Items { get; set; } = [];
}
```

### All useful annotations

| Annotation | Use case |
|---|---|
| `[Required]` | Field cannot be null or missing |
| `[StringLength(100)]` | Max character length |
| `[StringLength(100, MinimumLength = 1)]` | Min and max length |
| `[Range(1, 999)]` | Numeric min/max |
| `[EmailAddress]` | Valid email format |
| `[MinLength(1)]` | String or collection min length |

### What annotations can't do — handle manually in controller

```csharp
// date in the past
if (dto.Date < DateTime.Now)
    return BadRequest("Date cannot be in the past.");

// value must be one of a specific set
var validStatuses = new[] { "Scheduled", "Completed", "Cancelled" };
if (!validStatuses.Contains(dto.Status))
    return BadRequest("Invalid status value.");

// cross-field rules
if (dto.EndDate < dto.StartDate)
    return BadRequest("End date must be after start date.");
```

---

## 7. Exceptions

```csharp
// Exceptions/NotFoundException.cs
namespace YourProject.Exceptions;
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

// Exceptions/ConflictException.cs
namespace YourProject.Exceptions;
public class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }
}
```

---

## 8. IDbService Skeleton

```csharp
using YourProject.DTOs;

namespace YourProject.Services;

public interface IDbService
{
    Task<IEnumerable<SomeListDto>> GetAllAsync(string? filterParam1, string? filterParam2);
    Task<SomeDetailsDto> GetByIdAsync(int id);
    Task CreateAsync(CreateSomeRequestDto dto);
    Task UpdateAsync(int id, UpdateSomeRequestDto dto);
    Task DeleteAsync(int id);
}
```

---

## 9. DbService — ADO.NET Patterns

### Constructor (always the same):

```csharp
using Microsoft.Data.SqlClient;
using YourProject.DTOs;
using YourProject.Exceptions;

namespace YourProject.Services;

public class DbService : IDbService
{
    private readonly string _connectionString;

    public DbService(IConfiguration config)
    {
        _connectionString = config.GetConnectionString("DefaultConnection") ?? string.Empty;
    }

    // methods go here
}
```

---

### How a DB operation works — step by step

Every method in DbService follows the same lifecycle. Here it is in order:

**Without a transaction (GET, simple DELETE):**
```
1. Create connection     → new SqlConnection(_connectionString)
2. Open connection       → await connection.OpenAsync()
3. Create command        → new SqlCommand(query, connection)
4. Add parameters        → command.Parameters.AddWithValue(...)
5. Execute               → ExecuteReaderAsync / ExecuteScalarAsync / ExecuteNonQueryAsync
6. Read results          → while / if reader.ReadAsync()
7. Return / throw        → return dto  OR  throw new NotFoundException(...)
8. Connection closes     → automatically via await using
```

**With a transaction (POST, PUT, complex DELETE):**
```
1. Create connection     → new SqlConnection(_connectionString)
2. Open connection       → await connection.OpenAsync()
3. Begin transaction     → await connection.BeginTransactionAsync()
4. Create command        → new SqlCommand()
                           command.Connection = connection
                           command.Transaction = transaction as SqlTransaction
5. try {
     For each operation:
       a. Clear params   → command.Parameters.Clear()
       b. Set query      → command.CommandText = ...
       c. Add params     → command.Parameters.AddWithValue(...)
       d. Execute        → ExecuteScalarAsync / ExecuteNonQueryAsync
       e. Check result   → if null → throw NotFoundException / ConflictException

6.   CommitAsync()       → saves all changes to DB
   }
7. catch {
     RollbackAsync()     → undoes everything if anything failed
     throw              → re-throws so controller returns correct HTTP status
   }
8. Connection closes     → automatically via await using
```

**The golden rule:** open connection → begin transaction → try/commit → catch/rollback. If anything throws inside the try, rollback runs and nothing is saved.

---

### Pattern 1: SELECT many rows → return list


```csharp
public async Task<IEnumerable<SomeListDto>> GetAllAsync(string? filterParam1, string? filterParam2)
{
    var query = """
        SELECT t.col1, t.col2, t.col3,
               r.col1 AS RelatedCol
        FROM dbo.SomeTable t
        JOIN dbo.RelatedTable r ON r.Id = t.RelatedId
        WHERE (@Filter1 IS NULL OR t.col1 = @Filter1)
          AND (@Filter2 IS NULL OR r.col2 = @Filter2)
        ORDER BY t.col1
        """;

    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand(query, connection);
    command.Parameters.AddWithValue("@Filter1", (object?)filterParam1 ?? DBNull.Value);
    command.Parameters.AddWithValue("@Filter2", (object?)filterParam2 ?? DBNull.Value);

    await using var reader = await command.ExecuteReaderAsync();

    // get column positions ONCE before the loop
    var ord1 = reader.GetOrdinal("col1");
    var ord2 = reader.GetOrdinal("col2");
    var ord3 = reader.GetOrdinal("col3");
    var ordRelated = reader.GetOrdinal("RelatedCol");

    var results = new List<SomeListDto>();
    while (await reader.ReadAsync())
    {
        results.Add(new SomeListDto
        {
            Field1 = reader.GetInt32(ord1),
            Field2 = reader.GetString(ord2),
            Field3 = reader.GetDateTime(ord3),
            RelatedField = reader.GetString(ordRelated)
        });
    }

    return results;
}
```

---

### Pattern 2: SELECT one row → return DTO or throw 404

```csharp
public async Task<SomeDetailsDto> GetByIdAsync(int id)
{
    var query = """
        SELECT t.col1, t.col2, t.col3, t.nullableCol,
               r.col1 AS RelatedCol
        FROM dbo.SomeTable t
        JOIN dbo.RelatedTable r ON r.Id = t.RelatedId
        WHERE t.Id = @Id
        """;

    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand(query, connection);
    command.Parameters.AddWithValue("@Id", id);

    await using var reader = await command.ExecuteReaderAsync();

    // use if not while — expecting one row
    if (!await reader.ReadAsync())
        throw new NotFoundException($"Record {id} not found.");

    return new SomeDetailsDto
    {
        Field1       = reader.GetInt32(reader.GetOrdinal("col1")),
        Field2       = reader.GetString(reader.GetOrdinal("col2")),
        Field3       = reader.GetDateTime(reader.GetOrdinal("col3")),
        // nullable column — always check IsDBNull first
        NullableField = reader.IsDBNull(reader.GetOrdinal("nullableCol"))
                            ? null
                            : reader.GetString(reader.GetOrdinal("nullableCol")),
        RelatedField = reader.GetString(reader.GetOrdinal("RelatedCol"))
    };
}
```

---

### Pattern 3: Check if a record EXISTS

```csharp
// inline — reuse the same command object inside a transaction
command.CommandText = "SELECT 1 FROM dbo.SomeTable WHERE Id = @Id";
command.Parameters.AddWithValue("@Id", someId);
if (await command.ExecuteScalarAsync() == null)
    throw new NotFoundException($"Record {someId} not found.");

// check with extra condition (e.g. must be active)
command.Parameters.Clear();
command.CommandText = "SELECT 1 FROM dbo.SomeTable WHERE Id = @Id AND IsActive = 1";
command.Parameters.AddWithValue("@Id", someId);
if (await command.ExecuteScalarAsync() == null)
    throw new NotFoundException($"Active record {someId} not found.");

// check for conflict (thing must NOT exist)
command.Parameters.Clear();
command.CommandText = """
    SELECT 1 FROM dbo.SomeTable
    WHERE ForeignKeyId = @FkId AND Date = @Date
    """;
command.Parameters.AddWithValue("@FkId", dto.ForeignKeyId);
command.Parameters.AddWithValue("@Date", dto.Date);
if (await command.ExecuteScalarAsync() != null)
    throw new ConflictException("A record already exists for this time.");
```

---

### Pattern 4: INSERT with transaction (parent + child records)

```csharp
public async Task CreateAsync(CreateSomeRequestDto dto)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var transaction = await connection.BeginTransactionAsync();
    await using var command = new SqlCommand();
    command.Connection = connection;
    command.Transaction = transaction as SqlTransaction;

    try
    {
        // Step 1: validate foreign key exists (repeat for each FK)
        command.CommandText = "SELECT 1 FROM dbo.RelatedTable WHERE Id = @Id";
        command.Parameters.AddWithValue("@Id", dto.RelatedId);
        if (await command.ExecuteScalarAsync() == null)
            throw new NotFoundException($"Related record {dto.RelatedId} not found.");

        // Step 2: check for conflicts if needed
        command.Parameters.Clear();
        command.CommandText = """
            SELECT 1 FROM dbo.SomeTable
            WHERE ForeignKeyId = @FkId AND Date = @Date
            """;
        command.Parameters.AddWithValue("@FkId", dto.RelatedId);
        command.Parameters.AddWithValue("@Date", dto.Date);
        if (await command.ExecuteScalarAsync() != null)
            throw new ConflictException("Conflict: record already exists.");

        // Step 3: insert parent — OUTPUT INSERTED gets the new auto-increment ID
        command.Parameters.Clear();
        command.CommandText = """
            INSERT INTO dbo.SomeTable (col1, col2, col3)
            OUTPUT INSERTED.Id
            VALUES (@Val1, @Val2, @Val3)
            """;
        command.Parameters.AddWithValue("@Val1", dto.Field1);
        command.Parameters.AddWithValue("@Val2", dto.Field2);
        command.Parameters.AddWithValue("@Val3", (object?)dto.NullableField ?? DBNull.Value);

        var newId = Convert.ToInt32(await command.ExecuteScalarAsync());

        // Step 4: insert child records using the new parent ID
        foreach (var item in dto.Items)
        {
            command.Parameters.Clear();
            command.CommandText = """
                INSERT INTO dbo.ChildTable (ParentId, col1, col2)
                VALUES (@ParentId, @Val1, @Val2)
                """;
            command.Parameters.AddWithValue("@ParentId", newId);
            command.Parameters.AddWithValue("@Val1", item.Field1);
            command.Parameters.AddWithValue("@Val2", item.Field2);
            await command.ExecuteNonQueryAsync();
        }

        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw; // re-throw so controller catches it
    }
}
```

---

### Pattern 5: UPDATE existing record

```csharp
public async Task UpdateAsync(int id, UpdateSomeRequestDto dto)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var transaction = await connection.BeginTransactionAsync();
    await using var command = new SqlCommand();
    command.Connection = connection;
    command.Transaction = transaction as SqlTransaction;

    try
    {
        // check record exists — also grab any field needed for business rules
        command.CommandText = "SELECT Status FROM dbo.SomeTable WHERE Id = @Id";
        command.Parameters.AddWithValue("@Id", id);
        var currentStatus = await command.ExecuteScalarAsync() as string;
        if (currentStatus == null)
            throw new NotFoundException($"Record {id} not found.");

        // optional business rule based on current state
        if (currentStatus == "Locked")
            throw new ConflictException("Cannot modify a locked record.");

        // do the update
        command.Parameters.Clear();
        command.CommandText = """
            UPDATE dbo.SomeTable
            SET col1 = @Val1,
                col2 = @Val2,
                col3 = @Val3,
                nullableCol = @NullableVal
            WHERE Id = @Id
            """;
        command.Parameters.AddWithValue("@Val1", dto.Field1);
        command.Parameters.AddWithValue("@Val2", dto.Field2);
        command.Parameters.AddWithValue("@Val3", dto.Field3);
        command.Parameters.AddWithValue("@NullableVal", (object?)dto.NullableField ?? DBNull.Value);
        command.Parameters.AddWithValue("@Id", id);

        await command.ExecuteNonQueryAsync();
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

---

### Pattern 6: DELETE

```csharp
public async Task DeleteAsync(int id)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand();
    command.Connection = connection;

    // check exists — grab a field for business rules if needed
    command.CommandText = "SELECT Status FROM dbo.SomeTable WHERE Id = @Id";
    command.Parameters.AddWithValue("@Id", id);
    var result = await command.ExecuteScalarAsync();

    if (result == null)
        throw new NotFoundException($"Record {id} not found.");

    // optional business rule
    if (result.ToString() == "Locked")
        throw new ConflictException("Cannot delete a locked record.");

    command.Parameters.Clear();
    command.CommandText = "DELETE FROM dbo.SomeTable WHERE Id = @Id";
    command.Parameters.AddWithValue("@Id", id);
    await command.ExecuteNonQueryAsync();
}
```

---

## 10. Controller Skeleton — All 5 Endpoints

```csharp
using Microsoft.AspNetCore.Mvc;
using YourProject.DTOs;
using YourProject.Exceptions;
using YourProject.Services;

namespace YourProject.Controllers;

[ApiController]
[Route("api/[controller]")]
public class SomeController : ControllerBase
{
    private readonly IDbService _dbService;

    public SomeController(IDbService dbService)
    {
        _dbService = dbService;
    }

    // GET /api/some?filter1=x&filter2=y
    [HttpGet]
    public async Task<IActionResult> GetAll([FromQuery] string? filter1, [FromQuery] string? filter2)
    {
        var result = await _dbService.GetAllAsync(filter1, filter2);
        return Ok(result);
    }

    // GET /api/some/5
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        try
        {
            var result = await _dbService.GetByIdAsync(id);
            return Ok(result);
        }
        catch (NotFoundException e)
        {
            return NotFound(new ErrorResponseDto { Message = e.Message });
        }
    }

    // POST /api/some
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateSomeRequestDto? dto)
    {
        if (dto is null) return BadRequest("Request body is required.");
        // add any simple input validation here (empty strings, past dates, etc.)

        try
        {
            await _dbService.CreateAsync(dto);
            return Created($"api/some", null);
        }
        catch (NotFoundException e) { return NotFound(new ErrorResponseDto { Message = e.Message }); }
        catch (ConflictException e) { return Conflict(new ErrorResponseDto { Message = e.Message }); }
    }

    // PUT /api/some/5
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateSomeRequestDto? dto)
    {
        if (dto is null) return BadRequest("Request body is required.");
        // add any simple input validation here

        try
        {
            await _dbService.UpdateAsync(id, dto);
            return Ok();
        }
        catch (NotFoundException e) { return NotFound(new ErrorResponseDto { Message = e.Message }); }
        catch (ConflictException e) { return Conflict(new ErrorResponseDto { Message = e.Message }); }
    }

    // DELETE /api/some/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        try
        {
            await _dbService.DeleteAsync(id);
            return NoContent(); // 204
        }
        catch (NotFoundException e) { return NotFound(new ErrorResponseDto { Message = e.Message }); }
        catch (ConflictException e) { return Conflict(new ErrorResponseDto { Message = e.Message }); }
    }
}
```

---

## 11. Pattern 7: Get new ID after INSERT — two ways

### Option A: `OUTPUT INSERTED` (preferred, more explicit)
```csharp
command.CommandText = """
    INSERT INTO dbo.SomeTable (col1, col2)
    OUTPUT INSERTED.Id
    VALUES (@Val1, @Val2)
    """;

var newId = Convert.ToInt32(await command.ExecuteScalarAsync());
```

### Option B: `SELECT @@IDENTITY` (used in older code / when OUTPUT isn't available)
```csharp
command.CommandText = """
    INSERT INTO dbo.SomeTable (col1, col2)
    VALUES (@Val1, @Val2)
    SELECT @@IDENTITY;
    """;

var newId = Convert.ToInt32(await command.ExecuteScalarAsync());
```

Both return the auto-generated ID of the row just inserted. `OUTPUT INSERTED` is safer when triggers are involved; `@@IDENTITY` is simpler but can be affected by triggers on other tables.

---

## 12. Pattern 8: Building nested DTOs from flat JOIN results

When your SQL JOINs produce **flat rows** but your DTO has **nested lists**, you need to assemble the structure manually in the while loop.

Example: one Customer → many Rentals → each Rental has many Movies. SQL returns one row per movie, so the same rental appears on multiple rows.

```
| CustomerId | FirstName | RentalId | RentalDate | MovieTitle  |
|------------|-----------|----------|------------|-------------|
| 1          | John      | 10       | 2024-01-01 | Inception   |
| 1          | John      | 10       | 2024-01-01 | Interstellar|
| 1          | John      | 11       | 2024-02-01 | Dune        |
```

```csharp
ParentDto? result = null;

while (await reader.ReadAsync())
{
    // Step 1: build the parent ONCE on the first row
    if (result is null)
    {
        result = new ParentDto
        {
            FirstName = reader.GetString(ordFirstName),
            LastName  = reader.GetString(ordLastName),
            Children  = new List<ChildDto>()
        };
    }

    // Step 2: find or create the child (e.g. Rental)
    var childId = reader.GetInt32(ordChildId);
    var child = result.Children.FirstOrDefault(c => c.Id == childId);

    if (child is null)
    {
        child = new ChildDto
        {
            Id   = childId,
            Date = reader.GetDateTime(ordDate),
            // nullable field:
            ReturnDate = reader.IsDBNull(ordReturnDate)
                             ? null
                             : reader.GetDateTime(ordReturnDate),
            GrandChildren = new List<GrandChildDto>()
        };
        result.Children.Add(child);
    }

    // Step 3: always add the grandchild (e.g. Movie) — new one on every row
    child.GrandChildren.Add(new GrandChildDto
    {
        Title = reader.GetString(ordTitle),
        Price = reader.GetDecimal(ordPrice)
    });
}

// if result is still null, no rows came back — throw 404
return result ?? throw new NotFoundException("Record not found.");
```

### The key rules:
- Parent is created **once** — check `if (result is null)`
- Child is found or created — use `FirstOrDefault(c => c.Id == childId)`
- Grandchild is **always added** — every row has a new one

---

## 13. What to Return from Each Endpoint

### GET — return data or 404
```csharp
// GET all — always 200, empty list is fine
var result = await _dbService.GetAllAsync(...);
return Ok(result);  // 200

// GET by id — 200 or 404
try
{
    var result = await _dbService.GetByIdAsync(id);
    return Ok(result);  // 200 + dto
}
catch (NotFoundException e) { return NotFound(new ErrorResponseDto { Message = e.Message }); }
```

### POST — 201 on success
```csharp
try
{
    await _dbService.CreateAsync(dto);
    return Created($"api/some", null);  // 201
}
catch (NotFoundException e) { return NotFound(...); }    // 404 — FK missing
catch (ConflictException e) { return Conflict(...); }   // 409 — duplicate/conflict
```

### PUT — 200 on success
```csharp
try
{
    await _dbService.UpdateAsync(id, dto);
    return Ok();  // 200, no body
}
catch (NotFoundException e) { return NotFound(...); }
catch (ConflictException e) { return Conflict(...); }
```

### DELETE — 204 on success
```csharp
try
{
    await _dbService.DeleteAsync(id);
    return NoContent();  // 204, no body
}
catch (NotFoundException e) { return NotFound(...); }
catch (ConflictException e) { return Conflict(...); }
```

### 400 Bad Request — always BEFORE calling the service
```csharp
if (dto is null) return BadRequest("Request body is required.");
if (string.IsNullOrWhiteSpace(dto.Reason)) return BadRequest("Reason is required.");
if (dto.Date < DateTime.Now) return BadRequest("Date cannot be in the past.");
// then call service...
```

### Full HTTP status summary

| Method | Success | Not found | Conflict | Bad input |
|--------|---------|-----------|----------|-----------|
| GET all | 200 + list | — | — | — |
| GET by id | 200 + dto | 404 | — | — |
| POST | 201 | 404 (FK missing) | 409 | 400 |
| PUT | 200 | 404 | 409 | 400 |
| DELETE | 204 | 404 | 409 | — |

---

## 14. Quick Reference Cheatsheet

| Goal | Method | Returns |
|------|--------|---------|
| Many rows | `ExecuteReaderAsync` + `while` | List |
| One row | `ExecuteReaderAsync` + `if` | DTO or throw 404 |
| Nested DTOs from flat JOIN | `while` + `FirstOrDefault` + conditional add | Parent DTO |
| Check exists | `ExecuteScalarAsync` → `!= null` | bool |
| Get new ID after INSERT | `OUTPUT INSERTED.Id` or `SELECT @@IDENTITY` + `ExecuteScalarAsync` | int |
| Insert/Update/Delete | `ExecuteNonQueryAsync` | void |
| Nullable column | `reader.IsDBNull(ord) ? null : reader.GetXxx(ord)` | T? |
| Null SQL param | `(object?)value ?? DBNull.Value` | — |

| HTTP Status | When |
|-------------|------|
| 200 OK | Successful GET or PUT |
| 201 Created | Successful POST |
| 204 No Content | Successful DELETE |
| 400 Bad Request | Invalid input / business rule on input |
| 404 Not Found | Record doesn't exist |
| 409 Conflict | Business rule violation (e.g. deleting Completed) |



Data Source=(localdb)\\MSSQLLocalDB;Initial Catalog=kolokwium;Integrated Security=True;
