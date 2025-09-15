# Mini Learning Project: Construction Checklist API (Clean(ish) Modular Sample)

A guided hands-on mini project mirroring the architectural style and tech stack of the WCS module you’re reviewing. Focus: .NET 8 backend fundamentals (CRUD only) using Entity Framework Core, MediatR, FluentValidation, AutoMapper, layered architecture, and Minimal APIs.

## 1. Tech Stack Recap (What the original project uses)

| Concern            | Library / Approach |
|--------------------|--------------------|
| Runtime            | .NET 8 |
| HTTP Layer         | ASP.NET Core (Minimal APIs / Endpoints) |
| Persistence        | Entity Framework Core 9 (SQL Server) |
| Data Access Style  | DbContext + Repository abstraction |
| CQRS / Mediator    | MediatR |
| Validation         | FluentValidation |
| Object Mapping     | AutoMapper |
| Raw SQL (Selective)| Dapper (when optimized reads needed) |
| Logging (orig)     | Serilog (optional for this mini) |
| Pattern Flavor     | Vertical slice / modular + clean architecture influences |
| Cross-cutting      | Auditing via base entity + SaveChanges override |

## 2. Goal of This Mini Project

Implement 3 entities with relationships:

- Project (1)  
- ServiceProvider (N per Project via Checklist)  
- Checklist (ProjectId + SupplierCode + DocumentName + IsChecked)  

We’ll build:
- Create/Read/Update/Delete for Project
- Add checklist entries for a project + service provider
- Query a project with its checklists
- Basic validation + mapping

## 3. Final Solution Layout

MiniChecklistSolution/ src/ MiniChecklist.Api/                (Presentation) MiniChecklist.Application/        (MediatR handlers, DTOs, Validators) MiniChecklist.Domain/             (Entities, Abstractions) MiniChecklist.Infrastructure/     (EF Core, Repositories, Migrations) README.md
### MiniChecklistSolution Directory Structure
MiniChecklistSolution/ 
  └── src/ 
      ├── MiniChecklist.Api/                
      # Presentation layer (Minimal API endpoints) 
      ├── MiniChecklist.Application/        
      # Application layer (MediatR handlers, DTOs, Validators) 
      ├── MiniChecklist.Domain/             
      # Domain layer (Entities, Abstractions) 
      └── MiniChecklist.Infrastructure/     
      # Infrastructure layer (EF Core, Repositories, Migrations) README.md

**Legend:**
- `MiniChecklist.Api/` – ASP.NET Core Minimal API project (entry point, HTTP endpoints)
- `MiniChecklist.Application/` – Business logic, CQRS handlers, DTOs, validation
- `MiniChecklist.Domain/` – Core domain models and interfaces
- `MiniChecklist.Infrastructure/` – Data access, EF Core context, repository implementations, migrations
- `README.md` – This documentation file

This structure supports clean separation of concerns and modular development.
             


## 4. Create the Solution & Projects
dotnet new sln -n MiniChecklistSolution mkdir src && cd src
dotnet new web -n MiniChecklist.Api dotnet new classlib -n MiniChecklist.Domain dotnet new classlib -n MiniChecklist.Application dotnet new classlib -n MiniChecklist.Infrastructure
dotnet sln add MiniChecklist./ dotnet add MiniChecklist.Application reference ../MiniChecklist.Domain dotnet add MiniChecklist.Infrastructure reference ../MiniChecklist.Application ../MiniChecklist.Domain dotnet add MiniChecklist.Api reference MiniChecklist.Application MiniChecklist.Infrastructure

## 5. Add NuGet Packages
dotnet add MiniChecklist.Application package MediatR dotnet add MiniChecklist.Application package FluentValidation dotnet add MiniChecklist.Application package AutoMapper dotnet add MiniChecklist.Infrastructure package Microsoft.EntityFrameworkCore dotnet add MiniChecklist.Infrastructure package Microsoft.EntityFrameworkCore.SqlServer dotnet add MiniChecklist.Infrastructure package Microsoft.EntityFrameworkCore.Design dotnet add MiniChecklist.Infrastructure package Dapper dotnet add MiniChecklist.Api package Microsoft.EntityFrameworkCore.Tools

(Optional) Serilog later if desired.
## 6. Domain Layer
src/MiniChecklist.Domain/Abstractions/AuditableEntity.cs namespace MiniChecklist.Domain.Abstractions; public abstract class AuditableEntity { public DateTime CreatedAt { get; set; } public string CreatedBy { get; set; } = "system"; public DateTime? LastUpdatedAt { get; set; } public string? LastUpdatedBy { get; set; } }

rc/MiniChecklist.Domain/Entities/Project.cs using MiniChecklist.Domain.Abstractions; namespace MiniChecklist.Domain.Entities; public class Project : AuditableEntity { public Guid ProjectId { get; set; } = Guid.NewGuid(); public string Name { get; set; } = default!; public string Region { get; set; } = default!; public string Status { get; set; } = "New"; public ICollection<Checklist> Checklists { get; set; } = new List<Checklist>(); }

src/MiniChecklist.Domain/Entities/Checklist.cs using MiniChecklist.Domain.Abstractions; namespace MiniChecklist.Domain.Entities; public class Checklist : AuditableEntity { public Guid ChecklistId { get; set; } = Guid.NewGuid(); public Guid ProjectId { get; set; } public string SupplierCode { get; set; } = default!; public string DocumentName { get; set; } = default!; public bool IsChecked { get; set; } public Project? Project { get; set; } public ServiceProvider? ServiceProvider { get; set; } }


## 7. Application Layer (DTOs, Commands, Queries)
src/MiniChecklist.Application/DTOs/ProjectDto.cs namespace MiniChecklist.Application.DTOs; public class ProjectDto { public Guid ProjectId { get; set; } public string Name { get; set; } = default!; public string Region { get; set; } = default!; public string Status { get; set; } = default!; public List<ChecklistDto> Checklists { get; set; } = []; }

src/MiniChecklist.Application/DTOs/ChecklistDto.cs namespace MiniChecklist.Application.DTOs; public class ChecklistDto { public Guid ChecklistId { get; set; } public string DocumentName { get; set; } = default!; public bool IsChecked { get; set; } public string SupplierCode { get; set; } = default!; }

src/MiniChecklist.Application/DTOs/ServiceProviderDto.cs namespace MiniChecklist.Application.DTOs; public class ServiceProviderDto { public string SupplierCode { get; set; } = default!; public string SupplierName { get; set; } = default!; }


### Mapping Profile
src/MiniChecklist.Application/Mapping/MappingProfile.cs using AutoMapper; using MiniChecklist.Domain.Entities; using MiniChecklist.Application.DTOs;
namespace MiniChecklist.Application.Mapping; public class MappingProfile : Profile { public MappingProfile() { CreateMap<Project, ProjectDto>().ReverseMap(); CreateMap<Checklist, ChecklistDto>().ReverseMap(); CreateMap<ServiceProvider, ServiceProviderDto>().ReverseMap(); } }


### MediatR Requests
src/MiniChecklist.Application/Projects/Commands/CreateProject.cs using MediatR; using MiniChecklist.Application.DTOs;
namespace MiniChecklist.Application.Projects.Commands; public record CreateProject(string Name, string Region) : IRequest<ProjectDto>;

src/MiniChecklist.Application/Projects/Queries/GetProjectById.cs using MediatR; using MiniChecklist.Application.DTOs; namespace MiniChecklist.Application.Projects.Queries; public record GetProjectById(Guid ProjectId) : IRequest<ProjectDto?>;

src/MiniChecklist.Application/Checklists/Commands/AddChecklistItem.cs using MediatR; using MiniChecklist.Application.DTOs; namespace MiniChecklist.Application.Checklists.Commands; public record AddChecklistItem(Guid ProjectId, string SupplierCode, string DocumentName, bool IsChecked) : IRequest<ChecklistDto>;


### Validators
src/MiniChecklist.Application/Projects/Validators/CreateProjectValidator.cs using FluentValidation; using MiniChecklist.Application.Projects.Commands;
namespace MiniChecklist.Application.Projects.Validators; public class CreateProjectValidator : AbstractValidator<CreateProject> { public CreateProjectValidator() { RuleFor(x => x.Name).NotEmpty().MaximumLength(150); RuleFor(x => x.Region).NotEmpty(); } }

### Handlers
src/MiniChecklist.Application/Projects/Handlers/CreateProjectHandler.cs using AutoMapper; using MediatR; using MiniChecklist.Application.DTOs; using MiniChecklist.Application.Projects.Commands; using MiniChecklist.Infrastructure.Persistence; using MiniChecklist.Domain.Entities; using FluentValidation;
namespace MiniChecklist.Application.Projects.Handlers; public class CreateProjectHandler(IValidator<CreateProject> validator, AppDbContext db, IMapper mapper) : IRequestHandler<CreateProject, ProjectDto> { public async Task<ProjectDto> Handle(CreateProject request, CancellationToken ct) { await validator.ValidateAndThrowAsync(request, ct); var entity = new Project { Name = request.Name, Region = request.Region }; db.Projects.Add(entity); await db.SaveChangesAsync(ct); return mapper.Map<ProjectDto>(entity); } }

src/MiniChecklist.Application/Projects/Handlers/GetProjectByIdHandler.cs using AutoMapper; using MediatR; using Microsoft.EntityFrameworkCore; using MiniChecklist.Application.DTOs; using MiniChecklist.Application.Projects.Queries; using MiniChecklist.Infrastructure.Persistence;
namespace MiniChecklist.Application.Projects.Handlers; public class GetProjectByIdHandler(AppDbContext db, IMapper mapper) : IRequestHandler<GetProjectById, ProjectDto?> { public async Task<ProjectDto?> Handle(GetProjectById request, CancellationToken ct) { var project = await db.Projects .Include(p => p.Checklists) .FirstOrDefaultAsync(p => p.ProjectId == request.ProjectId, ct); return project is null ? null : mapper.Map<ProjectDto>(project); } }


src/MiniChecklist.Application/Checklists/Handlers/AddChecklistItemHandler.cs using AutoMapper; using MediatR; using Microsoft.EntityFrameworkCore; using MiniChecklist.Application.Checklists.Commands; using MiniChecklist.Application.DTOs; using MiniChecklist.Domain.Entities; using MiniChecklist.Infrastructure.Persistence;
namespace MiniChecklist.Application.Checklists.Handlers; public class AddChecklistItemHandler(AppDbContext db, IMapper mapper) : IRequestHandler<AddChecklistItem, ChecklistDto> { public async Task<ChecklistDto> Handle(AddChecklistItem request, CancellationToken ct) { // ensure project exists var projectExists = await db.Projects.AnyAsync(p => p.ProjectId == request.ProjectId, ct); if (!projectExists) throw new InvalidOperationException("Project not found.");


  // upsert service provider (simplified)
    var sp = await db.ServiceProviders.FindAsync([request.SupplierCode], ct);
    if (sp is null)
    {
        sp = new ServiceProvider { SupplierCode = request.SupplierCode, SupplierName = $"Supplier {request.SupplierCode}" };
        db.ServiceProviders.Add(sp);
    }

    var checklist = new Checklist
    {
        ProjectId = request.ProjectId,
        SupplierCode = request.SupplierCode,
        DocumentName = request.DocumentName,
        IsChecked = request.IsChecked,
        CreatedAt = DateTime.UtcNow,
        CreatedBy = "demo"
    };
    db.Checklists.Add(checklist);
    await db.SaveChangesAsync(ct);
    return mapper.Map<ChecklistDto>(checklist);
}
}

## 8. Infrastructure Layer (EF Core)

src/MiniChecklist.Infrastructure/Persistence/AppDbContext.cs using Microsoft.EntityFrameworkCore; using MiniChecklist.Domain.Entities; using MiniChecklist.Domain.Abstractions;
namespace MiniChecklist.Infrastructure.Persistence; public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options) { public DbSet<Project> Projects => Set<Project>(); public DbSet<Checklist> Checklists => Set<Checklist>(); public DbSet<ServiceProvider> ServiceProviders => Set<ServiceProvider>();

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Project>().HasKey(p => p.ProjectId);
    modelBuilder.Entity<ServiceProvider>().HasKey(s => s.SupplierCode);
    modelBuilder.Entity<Checklist>().HasKey(c => c.ChecklistId);

    modelBuilder.Entity<Checklist>()
        .HasOne(c => c.Project)
        .WithMany(p => p.Checklists)
        .HasForeignKey(c => c.ProjectId)
        .OnDelete(DeleteBehavior.Cascade);

    modelBuilder.Entity<Checklist>()
        .HasOne(c => c.ServiceProvider)
        .WithMany(s => s.Checklists)
        .HasForeignKey(c => c.SupplierCode);

    modelBuilder.Entity<Checklist>()
        .HasIndex(c => new { c.ProjectId, c.SupplierCode, c.DocumentName })
        .IsUnique();

    base.OnModelCreating(modelBuilder);
}

public override Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var now = DateTime.UtcNow;
    foreach (var entry in ChangeTracker.Entries<AuditableEntity>())
    {
        if (entry.State == EntityState.Added)
        {
            entry.Entity.CreatedAt = now;
            entry.Entity.CreatedBy ??= "system";
        }
        else if (entry.State == EntityState.Modified)
        {
            entry.Entity.LastUpdatedAt = now;
            entry.Entity.LastUpdatedBy = "system";
        }
    }
    return base.SaveChangesAsync(ct);
}
}

## 9. API Project (Minimal APIs + Wiring)

src/MiniChecklist.Api/Program.cs using AutoMapper; using FluentValidation; using MediatR; using Microsoft.EntityFrameworkCore; using MiniChecklist.Application.Mapping; using MiniChecklist.Application.Projects.Commands; using MiniChecklist.Application.Projects.Queries; using MiniChecklist.Application.Projects.Validators; using MiniChecklist.Application.Checklists.Commands; using MiniChecklist.Infrastructure.Persistence;
var builder = WebApplication.CreateBuilder(args);
// Db (local dev) builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(builder.Configuration.GetConnectionString("Default") ?? "Server=(localdb)\MSSQLLocalDB;Database=MiniChecklistDb;Trusted_Connection=True;TrustServerCertificate=True"));
// MediatR & AutoMapper & Validators builder.Services.AddMediatR(typeof(CreateProject)); builder.Services.AddAutoMapper(typeof(MappingProfile)); builder.Services.AddScoped<IValidator<CreateProject>, CreateProjectValidator>();
builder.Services.AddEndpointsApiExplorer(); builder.Services.AddSwaggerGen();
var app = builder.Build(); if (app.Environment.IsDevelopment()) { app.UseSwagger(); app.UseSwaggerUI(); }
// Migrate automatically (dev only) using (var scope = app.Services.CreateScope()) { var db = scope.ServiceProvider.GetRequiredService<AppDbContext>(); db.Database.Migrate(); }
// Routes app.MapPost("/projects", async (CreateProject cmd, IMediator mediator) => { var result = await mediator.Send(cmd); return Results.Created($"/projects/{result.ProjectId}", result); });
app.MapGet("/projects/{id:guid}", async (Guid id, IMediator mediator) => { var proj = await mediator.Send(new GetProjectById(id)); return proj is null ? Results.NotFound() : Results.Ok(proj); });
app.MapPost("/projects/{id:guid}/checklists", async (Guid id, AddChecklistItemRequest body, IMediator mediator) => { var dto = await mediator.Send(new AddChecklistItem(id, body.SupplierCode, body.DocumentName, body.IsChecked)); return Results.Created($"/checklists/{dto.ChecklistId}", dto); });
app.MapGet("/projects/{id:guid}/checklists", async (Guid id, AppDbContext db) => { var data = await db.Checklists .Where(c => c.ProjectId == id) .Select(c => new { c.ChecklistId, c.DocumentName, c.IsChecked, c.SupplierCode }) .ToListAsync(); return Results.Ok(data); });
app.MapPut("/checklists/{checklistId:guid}", async (Guid checklistId, UpdateChecklistRequest req, AppDbContext db) => { var entity = await db.Checklists.FindAsync(checklistId); if (entity is null) return Results.NotFound(); entity.IsChecked = req.IsChecked; await db.SaveChangesAsync(); return Results.NoContent(); });
app.MapDelete("/checklists/{checklistId:guid}", async (Guid checklistId, AppDbContext db) => { var entity = await db.Checklists.FindAsync(checklistId); if (entity is null) return Results.NotFound(); db.Checklists.Remove(entity); await db.SaveChangesAsync(); return Results.NoContent(); });
app.Run();
record AddChecklistItemRequest(string SupplierCode, string DocumentName, bool IsChecked); record UpdateChecklistRequest(bool IsChecked);

## 10. Add Initial Migration
dotnet tool install --global dotnet-ef dotnet ef migrations add InitialCreate -p MiniChecklist.Infrastructure -s MiniChecklist.Api -o Persistence/Migrations dotnet ef database update -p MiniChecklist.Infrastructure -s MiniChecklist.Api

Ensure the Infrastructure project has the design package and a reference chain to Domain/Application.

## 11. Test With HTTP (Examples)

Create project:
POST /projects { "name": "Head Office Refurb", "region": "North" }
Add checklist item:
POST /projects/{projectId}/checklists { "supplierCode": "SUP001", "documentName": "Contract Data", "isChecked": true }
Get project:
GET /projects/{projectId}
Update checklist:
PUT /checklists/{checklistId} { "isChecked": false }

Delete checklist:
DELETE /checklists/{checklistId}


## 12. Extension Ideas (After Core CRUD Works)

- Add query with Dapper (projection-only read)
- Add pagination
- Add Serilog
- Add soft delete
- Introduce Result/Error wrapper similar to original

## 13. Key Parallels to Original Codebase

| Original Concept                     | Mini Version |
|-------------------------------------|--------------|
| WcsDbContext                        | AppDbContext |
| AuditableEntity timestamps          | Same pattern |
| Checklist / Project / ServiceProvider | Same trio |
| MediatR handlers                    | CreateProjectHandler, etc. |
| FluentValidation                    | CreateProjectValidator |
| AutoMapper                          | MappingProfile |
| SaveChanges override for auditing   | Implemented |
| Layered separation                  | Domain / Application / Infrastructure / Api |

## 14. Cleanup / Reset

To reset DB during experimentation:
dotnet ef database drop -p MiniChecklist.Infrastructure -s MiniChecklist.Api -f dotnet ef database update -p MiniChecklist.Infrastructure -s MiniChecklist.Api


---

You now have a concise, faithful reproduction of the architectural style: Entities → EF Core → MediatR handlers → Validation → Minimal API endpoints.

Focus next: repetition (add update & delete for Project, service provider listing, etc.). This will solidify the patterns in the main project.

Happy building.



