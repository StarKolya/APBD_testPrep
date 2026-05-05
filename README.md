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
dotnet add package Swashbuckle.AspNetCore   # Swagger UI

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
mkdir Controllers DTOs Exceptions Services

# DTOs
touch DTOs/AppointmentListDto.cs
touch DTOs/AppointmentDetailsDto.cs
touch DTOs/CreateAppointmentRequestDto.cs
touch DTOs/UpdateAppointmentRequestDto.cs
touch DTOs/ErrorResponseDto.cs

# Exceptions
touch Exceptions/NotFoundException.cs
touch Exceptions/ConflictException.cs

# Services
touch Services/IDbService.cs
touch Services/DbService.cs

# Controller
touch Controllers/AppointmentsController.cs
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
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

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
// AppointmentListDto.cs
namespace Task6.DTOs;
public class AppointmentListDto
{
    public int IdAppointment { get; set; }
    public DateTime AppointmentDate { get; set; }
    public string Status { get; set; } = string.Empty;
    public string Reason { get; set; } = string.Empty;
    public string PatientFullName { get; set; } = string.Empty;
    public string PatientEmail { get; set; } = string.Empty;
}

// AppointmentDetailsDto.cs
namespace Task6.DTOs;
public class AppointmentDetailsDto
{
    public int IdAppointment { get; set; }
    public DateTime AppointmentDate { get; set; }
    public string Status { get; set; } = string.Empty;
    public string Reason { get; set; } = string.Empty;
    public string? InternalNotes { get; set; }
    public string PatientFullName { get; set; } = string.Empty;
    public string PatientEmail { get; set; } = string.Empty;
    public string PatientPhone { get; set; } = string.Empty;
    public string DoctorLicenseNumber { get; set; } = string.Empty;
}

// CreateAppointmentRequestDto.cs
namespace Task6.DTOs;
public class CreateAppointmentRequestDto
{
    public int IdPatient { get; set; }
    public int IdDoctor { get; set; }
    public DateTime AppointmentDate { get; set; }
    public string Reason { get; set; } = string.Empty;
}

// UpdateAppointmentRequestDto.cs
namespace Task6.DTOs;
public class UpdateAppointmentRequestDto
{
    public int IdPatient { get; set; }
    public int IdDoctor { get; set; }
    public DateTime AppointmentDate { get; set; }
    public string Status { get; set; } = string.Empty;
    public string Reason { get; set; } = string.Empty;
    public string? InternalNotes { get; set; }
}

// ErrorResponseDto.cs
namespace Task6.DTOs;
public class ErrorResponseDto
{
    public string Message { get; set; } = string.Empty;
}
```

---

## 6. Exceptions

```csharp
// Exceptions/NotFoundException.cs
namespace Task6.Exceptions;
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

// Exceptions/ConflictException.cs
namespace Task6.Exceptions;
public class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }
}
```

---

## 7. IDbService Skeleton

```csharp
using Task6.DTOs;

namespace Task6.Services;

public interface IDbService
{
    Task<IEnumerable<AppointmentListDto>> GetAllAppointmentsAsync(string? status, string? patientLastName);
    Task<AppointmentDetailsDto> GetAppointmentByIdAsync(int id);
    Task CreateAppointmentAsync(CreateAppointmentRequestDto dto);
    Task UpdateAppointmentAsync(int id, UpdateAppointmentRequestDto dto);
    Task DeleteAppointmentAsync(int id);
}
```

---

## 8. DbService — ADO.NET Patterns

### Full skeleton with constructor:

```csharp
using Microsoft.Data.SqlClient;
using Task6.DTOs;
using Task6.Exceptions;

namespace Task6.Services;

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

### Pattern 1: SELECT many rows → return list

```csharp
public async Task<IEnumerable<AppointmentListDto>> GetAllAppointmentsAsync(string? status, string? patientLastName)
{
    var query = """
        SELECT a.IdAppointment, a.AppointmentDate, a.Status, a.Reason,
               p.FirstName + ' ' + p.LastName AS PatientFullName,
               p.Email AS PatientEmail
        FROM dbo.Appointments a
        JOIN dbo.Patients p ON p.IdPatient = a.IdPatient
        WHERE (@Status IS NULL OR a.Status = @Status)
          AND (@PatientLastName IS NULL OR p.LastName = @PatientLastName)
        ORDER BY a.AppointmentDate
        """;

    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand(query, connection);
    command.Parameters.AddWithValue("@Status", (object?)status ?? DBNull.Value);
    command.Parameters.AddWithValue("@PatientLastName", (object?)patientLastName ?? DBNull.Value);

    await using var reader = await command.ExecuteReaderAsync();

    // get column positions ONCE before loop
    var ordId       = reader.GetOrdinal("IdAppointment");
    var ordDate     = reader.GetOrdinal("AppointmentDate");
    var ordStatus   = reader.GetOrdinal("Status");
    var ordReason   = reader.GetOrdinal("Reason");
    var ordFullName = reader.GetOrdinal("PatientFullName");
    var ordEmail    = reader.GetOrdinal("PatientEmail");

    var results = new List<AppointmentListDto>();
    while (await reader.ReadAsync())
    {
        results.Add(new AppointmentListDto
        {
            IdAppointment   = reader.GetInt32(ordId),
            AppointmentDate = reader.GetDateTime(ordDate),
            Status          = reader.GetString(ordStatus),
            Reason          = reader.GetString(ordReason),
            PatientFullName = reader.GetString(ordFullName),
            PatientEmail    = reader.GetString(ordEmail)
        });
    }

    return results;
}
```

---

### Pattern 2: SELECT one row → return DTO or throw 404

```csharp
public async Task<AppointmentDetailsDto> GetAppointmentByIdAsync(int id)
{
    var query = """
        SELECT a.IdAppointment, a.AppointmentDate, a.Status, a.Reason, a.InternalNotes,
               p.FirstName + ' ' + p.LastName AS PatientFullName,
               p.Email, p.PhoneNumber,
               d.LicenseNumber
        FROM dbo.Appointments a
        JOIN dbo.Patients p ON p.IdPatient = a.IdPatient
        JOIN dbo.Doctors d ON d.IdDoctor = a.IdDoctor
        WHERE a.IdAppointment = @Id
        """;

    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand(query, connection);
    command.Parameters.AddWithValue("@Id", id);

    await using var reader = await command.ExecuteReaderAsync();

    // use if, not while — expecting one row
    if (!await reader.ReadAsync())
        throw new NotFoundException($"Appointment {id} not found.");

    return new AppointmentDetailsDto
    {
        IdAppointment    = reader.GetInt32(reader.GetOrdinal("IdAppointment")),
        AppointmentDate  = reader.GetDateTime(reader.GetOrdinal("AppointmentDate")),
        Status           = reader.GetString(reader.GetOrdinal("Status")),
        Reason           = reader.GetString(reader.GetOrdinal("Reason")),
        // nullable column — check IsDBNull first
        InternalNotes    = reader.IsDBNull(reader.GetOrdinal("InternalNotes"))
                               ? null
                               : reader.GetString(reader.GetOrdinal("InternalNotes")),
        PatientFullName  = reader.GetString(reader.GetOrdinal("PatientFullName")),
        PatientEmail     = reader.GetString(reader.GetOrdinal("Email")),
        PatientPhone     = reader.GetString(reader.GetOrdinal("PhoneNumber")),
        DoctorLicenseNumber = reader.GetString(reader.GetOrdinal("LicenseNumber"))
    };
}
```

---

### Pattern 3: Check if a record EXISTS

```csharp
// returns true/false — use before INSERT/DELETE to validate
private async Task<bool> ExistsAsync(SqlCommand command, string query, string paramName, object value)
{
    command.CommandText = query;
    command.Parameters.Clear();
    command.Parameters.AddWithValue(paramName, value);
    var result = await command.ExecuteScalarAsync();
    return result != null;
}

// example usage:
// bool patientExists = await ExistsAsync(command,
//     "SELECT 1 FROM dbo.Patients WHERE IdPatient = @Id AND IsActive = 1",
//     "@Id", dto.IdPatient);
// if (!patientExists) throw new NotFoundException("Patient not found.");
```

---

### Pattern 4: INSERT with transaction (parent + child records)

```csharp
public async Task CreateAppointmentAsync(CreateAppointmentRequestDto dto)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var transaction = await connection.BeginTransactionAsync();
    await using var command = new SqlCommand();
    command.Connection = connection;
    command.Transaction = transaction as SqlTransaction;

    try
    {
        // Step 1: validate patient exists and is active
        command.CommandText = "SELECT 1 FROM dbo.Patients WHERE IdPatient = @Id AND IsActive = 1";
        command.Parameters.AddWithValue("@Id", dto.IdPatient);
        if (await command.ExecuteScalarAsync() == null)
            throw new NotFoundException($"Patient {dto.IdPatient} not found.");

        // Step 2: validate doctor exists and is active
        command.Parameters.Clear();
        command.CommandText = "SELECT 1 FROM dbo.Doctors WHERE IdDoctor = @Id AND IsActive = 1";
        command.Parameters.AddWithValue("@Id", dto.IdDoctor);
        if (await command.ExecuteScalarAsync() == null)
            throw new NotFoundException($"Doctor {dto.IdDoctor} not found.");

        // Step 3: check for scheduling conflict (409)
        command.Parameters.Clear();
        command.CommandText = """
            SELECT 1 FROM dbo.Appointments
            WHERE IdDoctor = @IdDoctor
              AND AppointmentDate = @Date
              AND Status = 'Scheduled'
            """;
        command.Parameters.AddWithValue("@IdDoctor", dto.IdDoctor);
        command.Parameters.AddWithValue("@Date", dto.AppointmentDate);
        if (await command.ExecuteScalarAsync() != null)
            throw new ConflictException("Doctor already has an appointment at this time.");

        // Step 4: insert — use OUTPUT INSERTED to get the new auto-increment ID back
        command.Parameters.Clear();
        command.CommandText = """
            INSERT INTO dbo.Appointments (IdPatient, IdDoctor, AppointmentDate, Reason, Status)
            OUTPUT INSERTED.IdAppointment
            VALUES (@IdPatient, @IdDoctor, @AppointmentDate, @Reason, 'Scheduled')
            """;
        command.Parameters.AddWithValue("@IdPatient", dto.IdPatient);
        command.Parameters.AddWithValue("@IdDoctor", dto.IdDoctor);
        command.Parameters.AddWithValue("@AppointmentDate", dto.AppointmentDate);
        command.Parameters.AddWithValue("@Reason", dto.Reason);

        var newId = Convert.ToInt32(await command.ExecuteScalarAsync());
        // newId is the auto-generated IdAppointment — use it for child records if needed

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
public async Task UpdateAppointmentAsync(int id, UpdateAppointmentRequestDto dto)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var transaction = await connection.BeginTransactionAsync();
    await using var command = new SqlCommand();
    command.Connection = connection;
    command.Transaction = transaction as SqlTransaction;

    try
    {
        // check appointment exists
        command.CommandText = "SELECT Status FROM dbo.Appointments WHERE IdAppointment = @Id";
        command.Parameters.AddWithValue("@Id", id);
        var currentStatus = await command.ExecuteScalarAsync() as string;
        if (currentStatus == null)
            throw new NotFoundException($"Appointment {id} not found.");

        // business rule: can't change date of a completed appointment
        if (currentStatus == "Completed")
            throw new ConflictException("Cannot modify a completed appointment.");

        // do the update
        command.Parameters.Clear();
        command.CommandText = """
            UPDATE dbo.Appointments
            SET AppointmentDate = @Date,
                Status = @Status,
                Reason = @Reason,
                InternalNotes = @Notes
            WHERE IdAppointment = @Id
            """;
        command.Parameters.AddWithValue("@Date", dto.AppointmentDate);
        command.Parameters.AddWithValue("@Status", dto.Status);
        command.Parameters.AddWithValue("@Reason", dto.Reason);
        command.Parameters.AddWithValue("@Notes", (object?)dto.InternalNotes ?? DBNull.Value);
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
public async Task DeleteAppointmentAsync(int id)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand();
    command.Connection = connection;

    // check exists and get status
    command.CommandText = "SELECT Status FROM dbo.Appointments WHERE IdAppointment = @Id";
    command.Parameters.AddWithValue("@Id", id);
    var status = await command.ExecuteScalarAsync() as string;

    if (status == null)
        throw new NotFoundException($"Appointment {id} not found.");
    if (status == "Completed")
        throw new ConflictException("Cannot delete a completed appointment.");

    command.Parameters.Clear();
    command.CommandText = "DELETE FROM dbo.Appointments WHERE IdAppointment = @Id";
    command.Parameters.AddWithValue("@Id", id);
    await command.ExecuteNonQueryAsync();
}
```

---

## 9. Controller Skeleton — All 5 Endpoints

```csharp
using Microsoft.AspNetCore.Mvc;
using Task6.DTOs;
using Task6.Exceptions;
using Task6.Services;

namespace Task6.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AppointmentsController : ControllerBase
{
    private readonly IDbService _dbService;

    public AppointmentsController(IDbService dbService)
    {
        _dbService = dbService;
    }

    // GET /api/appointments?status=Scheduled&patientLastName=Kowalska
    [HttpGet]
    public async Task<IActionResult> GetAll([FromQuery] string? status, [FromQuery] string? patientLastName)
    {
        var result = await _dbService.GetAllAppointmentsAsync(status, patientLastName);
        return Ok(result);
    }

    // GET /api/appointments/5
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        try
        {
            var result = await _dbService.GetAppointmentByIdAsync(id);
            return Ok(result);
        }
        catch (NotFoundException e)
        {
            return NotFound(new ErrorResponseDto { Message = e.Message });
        }
    }

    // POST /api/appointments
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateAppointmentRequestDto? dto)
    {
        if (dto is null) return BadRequest("Request body is required.");
        if (string.IsNullOrWhiteSpace(dto.Reason)) return BadRequest("Reason is required.");
        if (dto.AppointmentDate < DateTime.Now) return BadRequest("Appointment date cannot be in the past.");

        try
        {
            await _dbService.CreateAppointmentAsync(dto);
            return Created($"api/appointments", null);
        }
        catch (NotFoundException e) { return NotFound(new ErrorResponseDto { Message = e.Message }); }
        catch (ConflictException e) { return Conflict(new ErrorResponseDto { Message = e.Message }); }
    }

    // PUT /api/appointments/5
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateAppointmentRequestDto? dto)
    {
        if (dto is null) return BadRequest("Request body is required.");

        var validStatuses = new[] { "Scheduled", "Completed", "Cancelled" };
        if (!validStatuses.Contains(dto.Status))
            return BadRequest("Status must be Scheduled, Completed, or Cancelled.");

        try
        {
            await _dbService.UpdateAppointmentAsync(id, dto);
            return Ok();
        }
        catch (NotFoundException e) { return NotFound(new ErrorResponseDto { Message = e.Message }); }
        catch (ConflictException e) { return Conflict(new ErrorResponseDto { Message = e.Message }); }
    }

    // DELETE /api/appointments/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        try
        {
            await _dbService.DeleteAppointmentAsync(id);
            return NoContent(); // 204
        }
        catch (NotFoundException e) { return NotFound(new ErrorResponseDto { Message = e.Message }); }
        catch (ConflictException e) { return Conflict(new ErrorResponseDto { Message = e.Message }); }
    }
}
```

---

## 10. Quick Reference Cheatsheet

| Goal | Method | Returns |
|------|--------|---------|
| Many rows | `ExecuteReaderAsync` + `while` | List |
| One row | `ExecuteReaderAsync` + `if` | DTO or throw 404 |
| Check exists | `ExecuteScalarAsync` → `!= null` | bool |
| Get new ID after INSERT | `OUTPUT INSERTED.Id` + `ExecuteScalarAsync` | int |
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
