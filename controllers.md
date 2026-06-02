using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Task9.Data;
using Task9.DTOs;
using Task9.Models;

namespace Task9.Controllers;

[ApiController]
[Route("api/submissions")]
public class SubmissionsController : ControllerBase
{
    private readonly UniversityTasksDbContext _context;

    public SubmissionsController(UniversityTasksDbContext context)
    {
        _context = context;
    }

    [HttpPost]
    public async Task<ActionResult<SubmissionDto>> CreateSubmission([FromBody] CreateSubmissionDto dto)
    {
        if (string.IsNullOrWhiteSpace(dto.RepositoryUrl) || !dto.RepositoryUrl.StartsWith("https://"))
            return BadRequest("RepositoryUrl must be non-empty and start with https://");

        var student = await _context.Students.FirstOrDefaultAsync(s => s.IdStudent == dto.StudentId);
        if (student is null)
            return NotFound($"Student {dto.StudentId} not found.");
        if (!student.IsActive)
            return BadRequest("Student is not active.");

        var assignment = await _context.Assignments
            .Include(a => a.Course)
            .FirstOrDefaultAsync(a => a.IdAssignment == dto.AssignmentId);
        if (assignment is null)
            return NotFound($"Assignment {dto.AssignmentId} not found.");
        if (!assignment.IsPublished)
            return BadRequest("Assignment is not published.");

        var isEnrolled = await _context.Enrollments.AnyAsync(e =>
            e.IdStudent == dto.StudentId &&
            e.IdCourse == assignment.IdCourse &&
            (e.Status == "Active" || e.Status == "Completed"));
        if (!isEnrolled)
            return BadRequest("Student is not enrolled in the course that owns this assignment.");

        var alreadySubmitted = await _context.Submissions.AnyAsync(s =>
            s.IdStudent == dto.StudentId &&
            s.IdAssignment == dto.AssignmentId);
        if (alreadySubmitted)
            return Conflict("Student has already submitted this assignment.");

        var now = DateTime.UtcNow;
        var submission = new Submission
        {
            IdAssignment = dto.AssignmentId,
            IdStudent = dto.StudentId,
            RepositoryUrl = dto.RepositoryUrl,
            Status = assignment.IsOverdue(now) ? "Late" : "Submitted",
            SubmittedAt = now
        };

        _context.Submissions.Add(submission);
        await _context.SaveChangesAsync();

        var result = new SubmissionDto
        {
            IdSubmission = submission.IdSubmission,
            StudentFullName = student.FullName,
            AssignmentTitle = assignment.Title,
            RepositoryUrl = submission.RepositoryUrl,
            Status = submission.Status,
            Score = submission.Score,
            Feedback = submission.Feedback,
            SubmittedAt = submission.SubmittedAt
        };

        return CreatedAtAction(nameof(GetSubmission), new { idSubmission = submission.IdSubmission }, result);
    }

    [HttpPut("{idSubmission}/grade")]
    public async Task<IActionResult> GradeSubmission(int idSubmission, [FromBody] GradeSubmissionDto dto)
    {
        var submission = await _context.Submissions
            .Include(s => s.Assignment)
            .FirstOrDefaultAsync(s => s.IdSubmission == idSubmission);

        if (submission is null)
            return NotFound($"Submission {idSubmission} not found.");
        if (dto.Score < 0)
            return BadRequest("Score cannot be negative.");
        if (dto.Score > submission.Assignment.MaxPoints)
            return BadRequest($"Score cannot exceed max points ({submission.Assignment.MaxPoints}).");

        // Change Tracker tracks the loaded entity; mutating properties marks them as modified
        submission.Score = dto.Score;
        submission.Feedback = dto.Feedback;
        submission.Status = "Graded";

        await _context.SaveChangesAsync();
        return NoContent();
    }

    [HttpDelete("{idSubmission}")]
    public async Task<IActionResult> DeleteSubmission(int idSubmission)
    {
        var submission = await _context.Submissions.FirstOrDefaultAsync(s => s.IdSubmission == idSubmission);

        if (submission is null)
            return NotFound($"Submission {idSubmission} not found.");
        if (submission.Status == "Graded")
            return BadRequest("Cannot delete a graded submission.");

        _context.Submissions.Remove(submission);
        await _context.SaveChangesAsync();
        return NoContent();
    }

    // Used by CreatedAtAction in CreateSubmission
    [HttpGet("{idSubmission}")]
    public async Task<ActionResult<SubmissionDto>> GetSubmission(int idSubmission)
    {
        var submission = await _context.Submissions
            .AsNoTracking()
            .Where(s => s.IdSubmission == idSubmission)
            .Select(s => new SubmissionDto
            {
                IdSubmission = s.IdSubmission,
                StudentFullName = s.Student.FirstName + " " + s.Student.LastName,
                AssignmentTitle = s.Assignment.Title,
                RepositoryUrl = s.RepositoryUrl,
                Status = s.Status,
                Score = s.Score,
                Feedback = s.Feedback,
                SubmittedAt = s.SubmittedAt
            })
            .FirstOrDefaultAsync();

        if (submission is null)
            return NotFound($"Submission {idSubmission} not found.");

        return Ok(submission);
    }
}




---


using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Task9.Data;
using Task9.DTOs;

namespace Task9.Controllers;

[ApiController]
[Route("api/students")]
public class StudentsController : ControllerBase
{
    private readonly UniversityTasksDbContext _context;

    public StudentsController(UniversityTasksDbContext context)
    {
        _context = context;
    }

    [HttpGet("{idStudent}/dashboard")]
    public async Task<ActionResult<StudentDashboardDto>> GetDashboard(int idStudent)
    {
        var dashboard = await _context.Students
            .AsNoTracking()
            .Where(s => s.IdStudent == idStudent)
            .Select(s => new StudentDashboardDto
            {
                IdStudent = s.IdStudent,
                IndexNumber = s.IndexNumber,
                FullName = s.FirstName + " " + s.LastName,
                IsActive = s.IsActive,
                Enrollments = s.Enrollments
                    .Select(e => new EnrollmentSummaryDto
                    {
                        IdEnrollment = e.IdEnrollment,
                        CourseName = e.Course.Name,
                        CourseCode = e.Course.Code,
                        Status = e.Status,
                        EnrollmentDate = e.EnrollmentDate
                    })
                    .ToList(),
                Submissions = s.Submissions
                    .Select(sub => new SubmissionSummaryDto
                    {
                        IdSubmission = sub.IdSubmission,
                        AssignmentTitle = sub.Assignment.Title,
                        RepositoryUrl = sub.RepositoryUrl,
                        Status = sub.Status,
                        Score = sub.Score,
                        SubmittedAt = sub.SubmittedAt
                    })
                    .ToList()
            })
            .FirstOrDefaultAsync();

        if (dashboard is null)
            return NotFound($"Student {idStudent} not found.");

        return Ok(dashboard);
    }
}

---

using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Task9.Data;
using Task9.DTOs;

namespace Task9.Controllers;

[ApiController]
[Route("api/courses")]
public class CoursesController : ControllerBase
{
    private readonly UniversityTasksDbContext _context;

    public CoursesController(UniversityTasksDbContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<CourseDto>>> GetCourses([FromQuery] bool activeOnly = false)
    {
        var query = _context.Courses.AsNoTracking();

        if (activeOnly)
            query = query.Where(c => c.Assignments.Any(a => a.IsPublished));

        var courses = await query
            .Select(c => new CourseDto
            {
                IdCourse = c.IdCourse,
                Code = c.Code,
                Name = c.Name,
                Credits = c.Credits,
                AssignmentCount = c.Assignments.Count()
            })
            .ToListAsync();

        return Ok(courses);
    }

    [HttpGet("{idCourse}/assignments")]
    public async Task<ActionResult<IEnumerable<AssignmentDto>>> GetAssignments(
        int idCourse,
        [FromQuery] bool publishedOnly = false)
    {
        var courseExists = await _context.Courses
            .AsNoTracking()
            .AnyAsync(c => c.IdCourse == idCourse);

        if (!courseExists)
            return NotFound($"Course {idCourse} not found.");

        var query = _context.Assignments
            .AsNoTracking()
            .Where(a => a.IdCourse == idCourse);

        if (publishedOnly)
            query = query.Where(a => a.IsPublished);

        var assignments = await query
            .Select(a => new AssignmentDto
            {
                IdAssignment = a.IdAssignment,
                Title = a.Title,
                DueDate = a.DueDate,
                MaxPoints = a.MaxPoints,
                IsPublished = a.IsPublished,
                SubmissionCount = a.Submissions.Count()
            })
            .ToListAsync();

        return Ok(assignments);
    }
}
