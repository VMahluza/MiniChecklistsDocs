# Mini Learning Project: Construction Checklist API (Clean Architecture Sample)

A guided hands-on mini project mirroring the architectural style and tech stack of the WCS module you're reviewing. Focus: .NET 8 backend fundamentals (CRUD only) using Entity Framework Core, MediatR, and Clean Architecture principles.

## Architecture Overview

This project implements **Clean Architecture** with **Vertical Slice** organization, ensuring separation of concerns and maintainability.

### Layer Summary

| Layer | Purpose | Dependencies | Key Components |
|-------|---------|--------------|----------------|
| **Domain** | Core business entities and rules | None (innermost layer) | Entities, Value Objects, Abstractions |
| **Application** | Business logic and use cases | Domain only | Commands, Queries, Handlers, DTOs, Validators |
| **Infrastructure** | Data access and external concerns | Application + Domain | EF Core, Repositories, Database Context |
| **API (Presentation)** | HTTP endpoints and user interface | All layers | Controllers/Endpoints, Request/Response models |

### Dependency Flow
```
API → Infrastructure → Application → Domain
```
- Each layer can only depend on layers below it
- Domain has no dependencies (pure business logic)
- Infrastructure implements interfaces defined in Application

## 1. Tech Stack Recap

| Concern | Library / Approach |
|---------|-------------------|
| Runtime | .NET 8 |
| HTTP Layer | ASP.NET Core (Minimal APIs) |
| Persistence | Entity Framework Core 9 (SQL Server) |
| Data Access Style | DbContext + Repository abstraction |
| CQRS / Mediator | MediatR |
| Validation | FluentValidation |
| Object Mapping | AutoMapper |
| Raw SQL (Selective) | Dapper (when optimized reads needed) |
| Logging | Serilog (optional for this mini) |
| Pattern Flavor | Vertical slice + clean architecture |
| Cross-cutting | Auditing via base entity + SaveChanges override |

## 2. Goal of This Mini Project

Implement 3 entities with relationships:

- **Project** (1)  
- **ServiceProvider** (N per Project via Checklist)  
- **Checklist** (ProjectId + SupplierCode + DocumentName + IsChecked)  

### Features to Build:
- Create/Read/Update/Delete for Project
- Add checklist entries for a project + service provider
- Query a project with its checklists
- Basic validation + mapping

## 3. Solution Structure

```
MiniChecklistSolution/
├── README.md
└── src/
    ├── MiniChecklist.Api/                # 🌐 Presentation Layer
    │   ├── Program.cs                    # Entry point, DI setup, endpoints
    │   └── appsettings.json             # Configuration
    ├── MiniChecklist.Application/        # 🧠 Application Layer
    │   ├── DTOs/                        # Data Transfer Objects
    │   ├── Commands/                    # Write operations
    │   ├── Queries/                     # Read operations
    │   ├── Handlers/                    # MediatR request handlers
    │   ├── Validators/                  # FluentValidation rules
    │   └── Mapping/                     # AutoMapper profiles
    ├── MiniChecklist.Domain/            # 🏛️ Domain Layer
    │   ├── Entities/                    # Core business entities
    │   └── Abstractions/                # Base classes, interfaces
    └── MiniChecklist.Infrastructure/    # 🔧 Infrastructure Layer
        ├── Persistence/                 # EF Core context, configurations
        └── Migrations/                  # Database migrations
```

### Layer Responsibilities

#### 🏛️ **Domain Layer** (Core)
- **What it contains**: Business entities, domain rules, abstractions
- **Dependencies**: None
- **Examples**: `Project`, `Checklist`, `ServiceProvider` entities, `AuditableEntity` base class

#### 🧠 **Application Layer** (Use Cases)
- **What it contains**: Business logic, CQRS handlers, DTOs, validation
- **Dependencies**: Domain only
- **Examples**: `CreateProjectHandler`, `ProjectDto`, `CreateProjectValidator`

#### 🔧 **Infrastructure Layer** (Data & External Services)
- **What it contains**: Database access, external APIs, file system
- **Dependencies**: Application + Domain
- **Examples**: `AppDbContext`, EF Core configurations, migrations

#### 🌐 **API Layer** (Presentation)
- **What it contains**: HTTP endpoints, request/response handling, dependency injection
- **Dependencies**: All layers
- **Examples**: Minimal API endpoints, Program.cs setup

## 4. Getting Started

### Create the Solution & Projects

```bash
# Create solution
dotnet new sln -n MiniChecklistSolution
mkdir src && cd src

# Create projects
dotnet new web -n MiniChecklist.Api
dotnet new classlib -n MiniChecklist.Domain
dotnet new classlib -n MiniChecklist.Application
dotnet new classlib -n MiniChecklist.Infrastructure

# Add projects to solution
cd ..
dotnet sln add src/MiniChecklist.Api
dotnet sln add src/MiniChecklist.Domain
dotnet sln add src/MiniChecklist.Application
dotnet sln add src/MiniChecklist.Infrastructure

# Set up project references (following dependency rules)
cd src
dotnet add MiniChecklist.Application reference MiniChecklist.Domain
dotnet add MiniChecklist.Infrastructure reference MiniChecklist.Application
dotnet add MiniChecklist.Infrastructure reference MiniChecklist.Domain
dotnet add MiniChecklist.Api reference MiniChecklist.Application
dotnet add MiniChecklist.Api reference MiniChecklist.Infrastructure
```

## 5. Add NuGet Packages

```bash
# Application layer packages
dotnet add MiniChecklist.Application package MediatR
dotnet add MiniChecklist.Application package FluentValidation
dotnet add MiniChecklist.Application package AutoMapper

# Infrastructure layer packages
dotnet add MiniChecklist.Infrastructure package Microsoft.EntityFrameworkCore.SqlServer
dotnet add MiniChecklist.Infrastructure package Microsoft.EntityFrameworkCore.Tools
dotnet add MiniChecklist.Infrastructure package Microsoft.EntityFrameworkCore.Design

# API layer packages
dotnet add MiniChecklist.Api package Microsoft.EntityFrameworkCore.Design
dotnet add MiniChecklist.Api package Swashbuckle.AspNetCore
```

## 6. Domain Layer Implementation

### Base Entity
```csharp
// src/MiniChecklist.Domain/Abstractions/AuditableEntity.cs
namespace MiniChecklist.Domain.Abstractions;

public abstract class AuditableEntity
{
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; } = default!;
    public DateTime? LastUpdatedAt { get; set; }
    public string? LastUpdatedBy { get; set; }
}
```

### Domain Entities
```csharp
// src/MiniChecklist.Domain/Entities/Project.cs
using MiniChecklist.Domain.Abstractions;

namespace MiniChecklist.Domain.Entities;

public class Project : AuditableEntity
{
    public Guid ProjectId { get; set; }
    public string Name { get; set; } = default!;
    public string Region { get; set; } = default!;
    public virtual ICollection<Checklist> Checklists { get; set; } = new List<Checklist>();
}
```

```csharp
// src/MiniChecklist.Domain/Entities/ServiceProvider.cs
using MiniChecklist.Domain.Abstractions;

namespace MiniChecklist.Domain.Entities;

public class ServiceProvider : AuditableEntity
{
    public string SupplierCode { get; set; } = default!;
    public string SupplierName { get; set; } = default!;
    public virtual ICollection<Checklist> Checklists { get; set; } = new List<Checklist>();
}
```

```csharp
// src/MiniChecklist.Domain/Entities/Checklist.cs
using MiniChecklist.Domain.Abstractions;

namespace MiniChecklist.Domain.Entities;

public class Checklist : AuditableEntity
{
    public Guid ChecklistId { get; set; }
    public Guid ProjectId { get; set; }
    public string SupplierCode { get; set; } = default!;
    public string DocumentName { get; set; } = default!;
    public bool IsChecked { get; set; }
    
    // Navigation properties
    public virtual Project Project { get; set; } = default!;
    public virtual ServiceProvider ServiceProvider { get; set; } = default!;
}
```

## 7. Application Layer Implementation

### DTOs (Data Transfer Objects)
```csharp
// src/MiniChecklist.Application/DTOs/ProjectDto.cs
namespace MiniChecklist.Application.DTOs;

public class ProjectDto
{
    public Guid ProjectId { get; set; }
    public string Name { get; set; } = default!;
    public string Region { get; set; } = default!;
    public DateTime CreatedAt { get; set; }
    public List<ChecklistDto> Checklists { get; set; } = new();
}
```

### CQRS Commands and Queries
```csharp
// src/MiniChecklist.Application/Projects/Commands/CreateProject.cs
using MediatR;
using MiniChecklist.Application.DTOs;

namespace MiniChecklist.Application.Projects.Commands;

public record CreateProject(string Name, string Region) : IRequest<ProjectDto>;
```

```csharp
// src/MiniChecklist.Application/Projects/Queries/GetProjectById.cs
using MediatR;
using MiniChecklist.Application.DTOs;

namespace MiniChecklist.Application.Projects.Queries;

public record GetProjectById(Guid ProjectId) : IRequest<ProjectDto?>;
```

### Validation
```csharp
// src/MiniChecklist.Application/Projects/Validators/CreateProjectValidator.cs
using FluentValidation;
using MiniChecklist.Application.Projects.Commands;

namespace MiniChecklist.Application.Projects.Validators;

public class CreateProjectValidator : AbstractValidator<CreateProject>
{
    public CreateProjectValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(200);
            
        RuleFor(x => x.Region)
            .NotEmpty()
            .MaximumLength(100);
    }
}
```

## 8. Infrastructure Layer Implementation

### Database Context
```csharp
// src/MiniChecklist.Infrastructure/Persistence/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using MiniChecklist.Domain.Entities;
using MiniChecklist.Domain.Abstractions;

namespace MiniChecklist.Infrastructure.Persistence;

public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Project> Projects => Set<Project>();
    public DbSet<ServiceProvider> ServiceProviders => Set<ServiceProvider>();
    public DbSet<Checklist> Checklists => Set<Checklist>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configure entities
        modelBuilder.Entity<Project>().HasKey(p => p.ProjectId);
        modelBuilder.Entity<ServiceProvider>().HasKey(s => s.SupplierCode);
        modelBuilder.Entity<Checklist>().HasKey(c => c.ChecklistId);

        // Configure relationships
        modelBuilder.Entity<Checklist>()
            .HasOne(c => c.Project)
            .WithMany(p => p.Checklists)
            .HasForeignKey(c => c.ProjectId)
            .OnDelete(DeleteBehavior.Cascade);

        modelBuilder.Entity<Checklist>()
            .HasOne(c => c.ServiceProvider)
            .WithMany(s => s.Checklists)
            .HasForeignKey(c => c.SupplierCode);

        // Unique constraint
        modelBuilder.Entity<Checklist>()
            .HasIndex(c => new { c.ProjectId, c.SupplierCode, c.DocumentName })
            .IsUnique();

        base.OnModelCreating(modelBuilder);
    }

    // Automatic auditing
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
```

## 9. API Layer Implementation

### Program.cs (Dependency Injection & Endpoints)
```csharp
// src/MiniChecklist.Api/Program.cs
using AutoMapper;
using FluentValidation;
using MediatR;
using Microsoft.EntityFrameworkCore;
using MiniChecklist.Application.Mapping;
using MiniChecklist.Application.Projects.Commands;
using MiniChecklist.Application.Projects.Queries;
using MiniChecklist.Application.Projects.Validators;
using MiniChecklist.Infrastructure.Persistence;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default") 
        ?? "Server=(localdb)\\MSSQLLocalDB;Database=MiniChecklistDb;Trusted_Connection=true;"));

// Application services
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(CreateProject).Assembly));
builder.Services.AddAutoMapper(typeof(MappingProfile));
builder.Services.AddValidatorsFromAssemblyContaining<CreateProjectValidator>();

// API documentation
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Development tools
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Auto-migrate database (dev only)
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}

// API Endpoints
app.MapPost("/projects", async (CreateProject cmd, IMediator mediator) =>
{
    var result = await mediator.Send(cmd);
    return Results.Created($"/projects/{result.ProjectId}", result);
});

app.MapGet("/projects/{id:guid}", async (Guid id, IMediator mediator) =>
{
    var proj = await mediator.Send(new GetProjectById(id));
    return proj is null ? Results.NotFound() : Results.Ok(proj);
});

app.Run();

// Request models
record AddChecklistItemRequest(string SupplierCode, string DocumentName, bool IsChecked);
record UpdateChecklistRequest(bool IsChecked);
```

## 10. Database Setup

### Create Initial Migration
```bash
# Install EF Core tools globally
dotnet tool install --global dotnet-ef

# Create initial migration
dotnet ef migrations add InitialCreate \
    -p MiniChecklist.Infrastructure \
    -s MiniChecklist.Api \
    -o Persistence/Migrations

# Update database
dotnet ef database update \
    -p MiniChecklist.Infrastructure \
    -s MiniChecklist.Api
```

## 11. Testing the API

### Sample HTTP Requests

**Create a project:**
```http
POST /projects
Content-Type: application/json

{
    "name": "Head Office Refurbishment",
    "region": "North"
}
```

**Get a project:**
```http
GET /projects/{projectId}
```

**Add checklist item:**
```http
POST /projects/{projectId}/checklists
Content-Type: application/json

{
    "supplierCode": "SUP001",
    "documentName": "Contract Data",
    "isChecked": true
}
```

## 12. Key Benefits of This Architecture

### ✅ **Separation of Concerns**
- Each layer has a single responsibility
- Changes in one layer don't affect others
- Easy to test individual components

### ✅ **Dependency Inversion**
- Domain layer has no external dependencies
- Infrastructure implements interfaces defined in Application
- Enables easy mocking and testing

### ✅ **CQRS Pattern**
- Separate commands (writes) from queries (reads)
- Optimized operations for different scenarios
- Clear intent in code

### ✅ **Maintainability**
- Clear structure makes code easy to understand
- Consistent patterns across features
- Easy to add new features following same patterns

## 13. Extension Ideas

Once core CRUD works, consider adding:

- **Pagination** for large datasets
- **Logging** with Serilog
- **Caching** with Redis
- **API versioning**
- **Authentication & Authorization**
- **Background jobs** with Hangfire
- **Event sourcing** for audit trails
- **Docker containerization**

## 14. Key Parallels to Original Codebase

| Original Concept | Mini Version |
|------------------|-------------|
| WcsDbContext | AppDbContext |
| AuditableEntity timestamps | Same pattern |
| Checklist/Project/ServiceProvider | Same entities |
| MediatR handlers | CreateProjectHandler, etc. |
| FluentValidation | CreateProjectValidator |
| AutoMapper | MappingProfile |
| SaveChanges override for auditing | Implemented |
| Clean Architecture layers | Domain/Application/Infrastructure/API |

## 15. Troubleshooting

### Reset Database
```bash
dotnet ef database drop -p MiniChecklist.Infrastructure -s MiniChecklist.Api -f
dotnet ef database update -p MiniChecklist.Infrastructure -s MiniChecklist.Api
```

### Common Issues
- **Missing references**: Ensure project references follow dependency rules
- **Migration errors**: Check connection string and SQL Server availability
- **Validation failures**: Review FluentValidation rules
- **Mapping issues**: Verify AutoMapper profile configurations

---

🎯 **Goal**: Master Clean Architecture patterns through hands-on practice with a realistic business scenario. This foundation will make working with larger, more complex systems much easier.

Happy building! 🚀
````I'll help you fix the README.md file and provide a comprehensive layer summary. Let me create an improved version with better formatting and clear explanations of all the architectural layers.

````markdown name=README.md
# Mini Learning Project: Construction Checklist API (Clean Architecture Sample)

A guided hands-on mini project mirroring the architectural style and tech stack of a production-grade .NET application. This project focuses on .NET 8 backend fundamentals using Clean Architecture principles with CRUD operations, Entity Framework Core, and MediatR.

## Table of Contents
1. [Tech Stack Overview](#1-tech-stack-overview)
2. [Project Goals](#2-project-goals)
3. [Architecture Overview](#3-architecture-overview)
4. [Solution Structure](#4-solution-structure)
5. [Layer Explanations](#5-layer-explanations)
6. [Setup Instructions](#6-setup-instructions)
7. [Implementation Guide](#7-implementation-guide)
8. [API Testing](#8-api-testing)
9. [Extension Ideas](#9-extension-ideas)
10. [Key Architectural Patterns](#10-key-architectural-patterns)

## 1. Tech Stack Overview

| Component | Technology |
|-----------|------------|
| Runtime | .NET 8 |
| Web Framework | ASP.NET Core (Minimal APIs) |
| Database | SQL Server with Entity Framework Core 9 |
| Architecture Pattern | Clean Architecture with Vertical Slices |
| CQRS/Mediator | MediatR |
| Validation | FluentValidation |
| Object Mapping | AutoMapper |
| Raw SQL (Optional) | Dapper for optimized reads |
| Logging | Serilog (optional) |
| Cross-cutting Concerns | Auditing via base entity |

## 2. Project Goals

Build a construction checklist management system with three core entities:

- **Project**: Main construction project entity
- **ServiceProvider**: Suppliers/contractors working on projects  
- **Checklist**: Document tracking items for each project-provider combination

### Core Features
- Full CRUD operations for Projects
- Add/update checklist entries for project + service provider combinations
- Query projects with their associated checklists
- Input validation and data mapping
- Audit trail for all entities

## 3. Architecture Overview

This project implements **Clean Architecture** with the following principles:

- **Dependency Inversion**: Inner layers don't depend on outer layers
- **Separation of Concerns**: Each layer has a specific responsibility
- **Testability**: Business logic is isolated and easily testable
- **Framework Independence**: Core business logic doesn't depend on external frameworks

## 4. Solution Structure

```
MiniChecklistSolution/
└── src/
    ├── MiniChecklist.Api/                 # 🌐 Presentation Layer
    ├── MiniChecklist.Application/         # 🧠 Application Layer  
    ├── MiniChecklist.Domain/              # 🏗️ Domain Layer
    └── MiniChecklist.Infrastructure/      # 🔧 Infrastructure Layer
```

## 5. Layer Explanations

### 🏗️ Domain Layer (Core)
**Purpose**: Contains the business entities and core business rules.

**Responsibilities**:
- Define domain entities (Project, ServiceProvider, Checklist)
- Define domain abstractions and interfaces
- Implement business rules and domain logic
- Contains no dependencies on external frameworks

**Key Components**:
- `AuditableEntity`: Base class providing audit fields (CreatedAt, CreatedBy, etc.)
- Entity models with business logic and relationships
- Domain interfaces and abstractions

### 🧠 Application Layer (Business Logic)
**Purpose**: Orchestrates business operations and implements use cases.

**Responsibilities**:
- Define application services and use cases
- Handle commands and queries (CQRS pattern)
- Implement business workflows
- Data validation and transformation
- Define DTOs for data transfer

**Key Components**:
- **Commands**: Actions that modify data (CreateProject, AddChecklistItem)
- **Queries**: Read operations (GetProjectById, GetProjectChecklists)
- **Handlers**: Process commands and queries using MediatR
- **DTOs**: Data transfer objects for API communication
- **Validators**: FluentValidation rules for input validation
- **Mapping Profiles**: AutoMapper configurations

### 🔧 Infrastructure Layer (Data & External Services)
**Purpose**: Handles data persistence and external service integrations.

**Responsibilities**:
- Database context and entity configurations
- Repository implementations
- Database migrations
- External API integrations
- File system operations

**Key Components**:
- `AppDbContext`: Entity Framework DbContext
- Entity configurations and relationships
- Database migrations
- Repository pattern implementations
- External service clients

### 🌐 Presentation Layer (API)
**Purpose**: Handles HTTP requests and responses.

**Responsibilities**:
- Define API endpoints
- Handle HTTP requests/responses
- Route requests to appropriate handlers
- API documentation (Swagger)
- Authentication and authorization (if implemented)

**Key Components**:
- Minimal API endpoints
- Request/response models
- Middleware pipeline configuration
- Dependency injection setup
- API documentation

## 6. Setup Instructions

### Create Solution Structure
```bash
# Create solution
dotnet new sln -n MiniChecklistSolution
mkdir src && cd src

# Create projects
dotnet new web -n MiniChecklist.Api
dotnet new classlib -n MiniChecklist.Domain
dotnet new classlib -n MiniChecklist.Application
dotnet new classlib -n MiniChecklist.Infrastructure

# Add projects to solution
dotnet sln add MiniChecklist.Api/ MiniChecklist.Domain/ MiniChecklist.Application/ MiniChecklist.Infrastructure/

# Setup project references
dotnet add MiniChecklist.Application reference MiniChecklist.Domain
dotnet add MiniChecklist.Infrastructure reference MiniChecklist.Application MiniChecklist.Domain
dotnet add MiniChecklist.Api reference MiniChecklist.Application MiniChecklist.Infrastructure
```

### Install NuGet Packages
```bash
# Application layer
dotnet add MiniChecklist.Application package MediatR
dotnet add MiniChecklist.Application package FluentValidation
dotnet add MiniChecklist.Application package AutoMapper

# Infrastructure layer
dotnet add MiniChecklist.Infrastructure package Microsoft.EntityFrameworkCore.SqlServer
dotnet add MiniChecklist.Infrastructure package Microsoft.EntityFrameworkCore.Tools

# API layer
dotnet add MiniChecklist.Api package Microsoft.EntityFrameworkCore.Design
dotnet add MiniChecklist.Api package Swashbuckle.AspNetCore
```

## 7. Implementation Guide

### Domain Layer Implementation

```csharp
// src/MiniChecklist.Domain/Abstractions/AuditableEntity.cs
namespace MiniChecklist.Domain.Abstractions;

public abstract class AuditableEntity
{
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; } = default!;
    public DateTime? LastUpdatedAt { get; set; }
    public string? LastUpdatedBy { get; set; }
}
```

```csharp
// src/MiniChecklist.Domain/Entities/Project.cs
using MiniChecklist.Domain.Abstractions;

namespace MiniChecklist.Domain.Entities;

public class Project : AuditableEntity
{
    public Guid ProjectId { get; set; }
    public string Name { get; set; } = default!;
    public string Region { get; set; } = default!;
    public ICollection<Checklist> Checklists { get; set; } = new List<Checklist>();
}
```

### Application Layer Implementation

```csharp
// src/MiniChecklist.Application/DTOs/ProjectDto.cs
namespace MiniChecklist.Application.DTOs;

public class ProjectDto
{
    public Guid ProjectId { get; set; }
    public string Name { get; set; } = default!;
    public string Region { get; set; } = default!;
    public List<ChecklistDto> Checklists { get; set; } = new();
}
```

```csharp
// src/MiniChecklist.Application/Projects/Commands/CreateProject.cs
using MediatR;
using MiniChecklist.Application.DTOs;

namespace MiniChecklist.Application.Projects.Commands;

public record CreateProject(string Name, string Region) : IRequest<ProjectDto>;
```

### Infrastructure Layer Implementation

```csharp
// src/MiniChecklist.Infrastructure/Persistence/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using MiniChecklist.Domain.Entities;
using MiniChecklist.Domain.Abstractions;

namespace MiniChecklist.Infrastructure.Persistence;

public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Project> Projects => Set<Project>();
    public DbSet<ServiceProvider> ServiceProviders => Set<ServiceProvider>();
    public DbSet<Checklist> Checklists => Set<Checklist>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configure entity relationships and constraints
        modelBuilder.Entity<Checklist>()
            .HasIndex(c => new { c.ProjectId, c.SupplierCode, c.DocumentName })
            .IsUnique();
        
        base.OnModelCreating(modelBuilder);
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Implement audit trail logic
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
```

### API Layer Implementation

```csharp
// src/MiniChecklist.Api/Program.cs
using MediatR;
using Microsoft.EntityFrameworkCore;
using MiniChecklist.Infrastructure.Persistence;
using MiniChecklist.Application.Projects.Commands;

var builder = WebApplication.CreateBuilder(args);

// Configure services
builder.Services.AddDbContext<AppDbContext>(opt => 
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddMediatR(typeof(CreateProject));
builder.Services.AddAutoMapper(typeof(MappingProfile));
builder.Services.AddValidatorsFromAssemblyContaining<CreateProjectValidator>();

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Define API endpoints
app.MapPost("/projects", async (CreateProject cmd, IMediator mediator) =>
{
    var result = await mediator.Send(cmd);
    return Results.Created($"/projects/{result.ProjectId}", result);
});

app.MapGet("/projects/{id:guid}", async (Guid id, IMediator mediator) =>
{
    var project = await mediator.Send(new GetProjectById(id));
    return project is null ? Results.NotFound() : Results.Ok(project);
});

app.Run();
```

## 8. API Testing

### Create Project
```http
POST /projects
Content-Type: application/json

{
  "name": "Head Office Refurbishment",
  "region": "North"
}
```

### Add Checklist Item
```http
POST /projects/{projectId}/checklists
Content-Type: application/json

{
  "supplierCode": "SUP001",
  "documentName": "Contract Data",
  "isChecked": true
}
```

### Get Project with Checklists
```http
GET /projects/{projectId}
```

### Update Checklist Item
```http
PUT /checklists/{checklistId}
Content-Type: application/json

{
  "isChecked": false
}
```

## 9. Extension Ideas

After implementing the core CRUD functionality:

- **Advanced Querying**: Add Dapper for optimized read operations
- **Pagination**: Implement paged results for large datasets  
- **Logging**: Integrate Serilog for structured logging
- **Soft Delete**: Implement logical deletion with audit trails
- **Result Pattern**: Add Result/Error wrapper for better error handling
- **Authentication**: Add JWT-based authentication
- **Caching**: Implement Redis caching for frequently accessed data
- **Unit Tests**: Add comprehensive test coverage
- **Integration Tests**: Test API endpoints end-to-end

## 10. Key Architectural Patterns

| Pattern | Implementation | Benefit |
|---------|----------------|---------|
| **CQRS** | Commands for writes, Queries for reads | Clear separation of concerns |
| **Mediator** | MediatR for decoupled request handling | Loose coupling between components |
| **Repository** | EF Core DbContext as repository | Data access abstraction |
| **Unit of Work** | DbContext SaveChanges | Transaction management |
| **Dependency Injection** | Built-in .NET DI container | Testability and flexibility |
| **Clean Architecture** | Layered separation | Maintainable and testable code |
| **Audit Trail** | AuditableEntity base class | Change tracking |

## Database Setup

### Add Initial Migration
```bash
# Install EF Core tools
dotnet tool install --global dotnet-ef

# Create migration
dotnet ef migrations add InitialCreate -p MiniChecklist.Infrastructure -s MiniChecklist.Api -o Persistence/Migrations

# Update database
dotnet ef database update -p MiniChecklist.Infrastructure -s MiniChecklist.Api
```

### Reset Database (Development)
```bash
dotnet ef database drop -p MiniChecklist.Infrastructure -s MiniChecklist.Api -f
dotnet ef database update -p MiniChecklist.Infrastructure -s MiniChecklist.Api
```

---

## Summary

This mini project demonstrates modern .NET development practices using Clean Architecture principles. The layered approach ensures maintainable, testable, and scalable code that can serve as a foundation for larger enterprise applications.

Each layer has clear responsibilities and dependencies flow inward, making the system flexible and easy to extend. The use of established patterns like CQRS, Mediator, and Repository makes the codebase familiar to developers and aligned with industry best practices.

Happy coding! 🚀
