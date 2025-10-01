# E‑Restaurant Web API

A multi‑tenant restaurant backend for menus, combos, orders, and add‑ons — with built‑in localization (Arabic/English), clean error handling, and a developer‑friendly architecture (.NET + EF Core).

> This single README serves **two audiences**:
>
> 1) **Clients / Stakeholders** — overview, capabilities, and how to use the API at a high level.  
> 2) **Developers** — detailed setup, architecture, and conventions (the **Developer Guide** is further down).

---

## Table of Contents

- [Client Guide (Overview)](#client-guide-overview)
  - [What is this project?](#what-is-this-project)
  - [Key Features](#key-features)
  - [How to Access](#how-to-access)
  - [Important Headers](#important-headers)
  - [Typical Flows](#typical-flows)
  - [Error Format](#error-format)
  - [Service Levels \& Notes](#service-levels--notes)
  - [Contact \& Support](#contact--support)
- [Developer Guide (Detailed)](#developer-guide-detailed)
  - [Tech Stack](#tech-stack)
  - [Solution Structure](#solution-structure)
  - [Prerequisites](#prerequisites)
  - [Configuration](#configuration)
  - [Run the API](#run-the-api)
  - [Database \& Migrations](#database--migrations)
  - [Seeding](#seeding)
  - [Tenancy Model](#tenancy-model)
  - [Auto‑Loading (AutoInclude)](#auto-loading-autoinclude)
  - [Localization](#localization)
  - [Error Handling](#error-handling)
  - [Pagination \& Sorting](#pagination--sorting)
  - [Testing](#testing)
  - [Coding Standards \& Conventions](#coding-standards--conventions)
  - [CI/CD (Optional)](#cicd-optional)
  - [Troubleshooting](#troubleshooting)
  - [Roadmap](#roadmap)
  - [License](#license)

---

## Client Guide (Overview)

### What is this project?
**E‑Restaurant Web API** is a backend service for restaurant operations. It powers mobile apps or web dashboards to:
- Browse materials (ingredients/items), combos (meals made of materials), and optional add‑ons (e.g., sauces).
- Place and track orders.
- Work across **multiple tenants (restaurants/brands)** cleanly and safely.

The API is **language‑aware**:
- Content (like item names) responds in **Arabic** when `Accept-Language: ar` is sent, and **English** when `Accept-Language: en` is sent.

### Key Features
- **Multi‑tenant** isolation with per‑tenant data guards.
- **Menu & Combos**: materials, combos, and add‑ons.
- **Orders** with order items and add‑ons on each item.
- **Localization** at the DTO level (Arabic/English names).
- **Consistent errors** via RFC‑7807 `ProblemDetails`.
- **Swagger** for live API documentation.
- **Seed demo data** for quick evaluation.

### How to Access
- Run the API locally (see **Developer Guide**).  
- Once running, open: `http://localhost:5000/swagger` (or the port printed in the console) to explore endpoints interactively.

> If you received a hosted/staging URL from us, use that instead of localhost.

### Important Headers
| Header | Required | Purpose | Example |
|-------|---------|---------|---------|
| `X-Tenant-Id` | ✅ Yes | Selects which tenant (restaurant) your requests target. | `X-Tenant-Id: 1` |
| `Accept-Language` | Optional | Localizes names in responses. Supports `ar` or `en`. | `Accept-Language: ar` |

### Typical Flows
1) **List materials** (ingredients/items):  
   `GET /api/materials`
2) **View combos** (meals composed of materials):  
   `GET /api/combos`
3) **Place an order**:  
   `POST /api/orders` with items and optional add‑ons.
4) **Get order details**:  
   `GET /api/orders/{id}`

> Use **Swagger** to try these with your `X-Tenant-Id` and optional `Accept-Language` headers.

### Error Format
Errors follow **ProblemDetails** (RFC‑7807). Example:
```json
{
  "type": "https://http.dev/errors/validation",
  "title": "Validation Failed",
  "status": 400,
  "detail": "One or more validation errors occurred.",
  "errors": {
    "Items[0].Quantity": ["Quantity must be at least 1"]
  }
}
```

### Service Levels & Notes
- **Performance**: Optimized queries and auto‑loading for common navigation properties.
- **Isolation**: One tenant cannot read/modify another tenant’s data.
- **Localization**: Names switch language based on `Accept-Language`.
- **Security**: See Developer Guide for authentication/authorization options (varies by deployment).

### Contact & Support
- For access, demos, or questions, contact your project representative.

---

## Developer Guide (Detailed)

> Everything below is for engineers who will build/extend/deploy the API.

### Tech Stack
- **.NET 8** (or later), ASP.NET Core Web API
- **Entity Framework Core** (SQL Server)
- **AutoMapper** for mapping and localization shaping
- **Hellang.Middleware.ProblemDetails** for consistent error responses
- **Swagger / Swashbuckle** for API docs
- Optional: **JWT/Identity** if authentication is enabled in your deployment

### Solution Structure
Typical layout (adjust to your repo):
```
ERestaurant.sln
src/
  Api/                # Controllers, DI, middlewares, Swagger
  Application/        # DTOs, services, AutoMapper profiles
  Domain/             # Entities, enums, domain logic
  Infrastructure/     # DbContext, EF configs, repositories/UoW, seed
  Testing/            # Unit/integration tests
docs/                 # Diagrams, API docs, etc.
```

### Prerequisites
- .NET 8 SDK or later
- SQL Server (Express/Developer/LocalDB). Ensure **SQL Server Browser** is running for named instances.
- EF Core CLI tools:  
  ```bash
  dotnet tool install --global dotnet-ef
  ```

### Configuration
Create **`appsettings.Development.json`** in `src/Api` (or use env vars):

```json
{
  "ConnectionStrings": {
    "ERestaurantDb": "Server=localhost;Database=ERestaurant;Trusted_Connection=True;TrustServerCertificate=True"
  },
  "ErrorHandling": {
    "ExposeExceptionDetails": true
  },
  "Localization": {
    "SupportedCultures": [ "en", "ar" ],
    "DefaultCulture": "en"
  },
  "Tenancy": {
    "Header": "X-Tenant-Id",
    "DefaultTenantId": 1
  },
  "Swagger": {
    "Enable": true
  }
}
```

> **Production**: set a strong SQL connection string, disable `ExposeExceptionDetails`, and restrict Swagger as needed.

### Run the API
From the repo root (adjust paths if different):
```bash
dotnet restore
dotnet build

# Apply migrations then run
dotnet ef database update --project src/Infrastructure --startup-project src/Api
dotnet run --project src/Api
```
Open the printed URL (e.g., `https://localhost:7043/swagger`).

### Database & Migrations
Create a new migration when changing entities/config:
```bash
dotnet ef migrations add <Name> \
  --project src/Infrastructure \
  --startup-project src/Api
dotnet ef database update --project src/Infrastructure --startup-project src/Api
```

**Reset DB (Dev only)** — choose one:
- CLI reset:
  ```bash
  dotnet ef database drop -f --project src/Infrastructure --startup-project src/Api
  dotnet ef database update --project src/Infrastructure --startup-project src/Api
  ```
- Or guarded code path (Development only):
  ```csharp
  // in Program.cs during Development
  if (app.Environment.IsDevelopment() && builder.Configuration.GetValue<bool>("DevOptions:ResetDatabase"))
  {
      using var scope = app.Services.CreateScope();
      var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
      db.Database.EnsureDeleted();
      db.Database.EnsureCreated();
      SeedData.Apply(scope.ServiceProvider); // if you separate seeding logic
  }
  ```

### Seeding
A seed routine creates demo tenants and data, e.g.:
```csharp
Tenants: (1, "Burgers"), (2, "Pizza"), (3, "Shawarma"), (4, "Coffee&Tea"), (5, "Desserts")
```
Materials, combos, and add‑ons are generated per tenant with audit fields (`CreatedBy`, `CreatedDate`, etc.).  
Enable/disable seeding via config or environment.

### Tenancy Model
- **Header**: `X-Tenant-Id` identifies the active tenant.
- **Isolation**: Entities use `TenantId` and (optionally) alternate keys per entity, e.g.:
  ```csharp
  modelBuilder.Entity<Material>().HasAlternateKey(e => new { e.TenantId, e.Id });
  modelBuilder.Entity<Combo>().HasAlternateKey(e => new { e.TenantId, e.Id });
  modelBuilder.Entity<AdditionalMaterial>().HasAlternateKey(e => new { e.TenantId, e.Id });
  ```
- **Global query filter** applies the current tenant to all queries. A `ITenantProvider` reads the header and stores the `TenantId` for the current request scope.
- **Repository pattern** may stack an extra guard to ensure `TenantId` is always set on inserts/updates.

### Auto‑Loading (AutoInclude)
Common navigations are auto‑included through a specialized DbContext (or model config):
```csharp
modelBuilder.Entity<Combo>().Navigation(x => x.ComboMaterial).AutoInclude();
modelBuilder.Entity<ComboMaterial>().Navigation(x => x.Material).AutoInclude();

modelBuilder.Entity<Order>().Navigation(x => x.OrderItem).AutoInclude();
modelBuilder.Entity<OrderItem>().Navigation(x => x.Material).AutoInclude();
modelBuilder.Entity<OrderItem>().Navigation(x => x.Combo).AutoInclude();
modelBuilder.Entity<OrderItem>().Navigation(x => x.AdditionalMaterial).AutoInclude();
```
Use this judiciously to balance convenience and performance.

### Localization
- Accepts `Accept-Language: ar` or `en`.
- **AutoMapper** projects entities to DTOs using the language flag, e.g. passing `Items["isArabic"]` into mapping:
  ```csharp
  var dtos = _mapper.ProjectTo<ComboReturnDTO>(query, new { isArabic = lang.IsArabic });
  ```
- Profiles map `Name` from `NameAr` if Arabic, or `NameEn` if English.
- Register `RequestLocalizationOptions` in `Program.cs` with `SupportedCultures` = `[ "en", "ar" ]`.

### Error Handling
- Uses **Hellang.Middleware.ProblemDetails** to map exceptions to standard JSON:
  - `NotFoundException` → 404
  - `ValidationException` → 400 with `errors` dictionary
  - Others → 500 (hide details outside Development)
- Customize in `ProblemDetailsConfig` (maps, links, titles).

### Pagination & Sorting
- Endpoints return a `X-Pagination` header (JSON) for page/size/total when applicable.
- Query params: `page`, `pageSize`, `sort`, `filter` (pattern varies by endpoint).
- Keep responses lean and consistent.

### Testing
- **Unit tests** for services/mappers.
- **Integration tests** boot the API with in‑memory or test SQL DB.
- Seed a minimal dataset per test to avoid cross‑test coupling.

### Coding Standards & Conventions
- **C#/.NET Naming**: PascalCase for types/methods, camelCase for locals, `I` prefix for interfaces, `EnumMember` in PascalCase, DTOs suffixed `Dto` (or `DTO` consistently).
- **Projects**:
  - `ERestaurant.Api` — web layer only (controllers, DI, middlewares)
  - `ERestaurant.Application` — DTOs, services, AutoMapper
  - `ERestaurant.Domain` — entities/enums/rules
  - `ERestaurant.Infrastructure` — EF Core, repos/UoW, Seed
- **Exception flow**: throw domain/app exceptions; middleware maps to ProblemDetails.
- **Git**: feature branches, PR reviews, conventional commits (optional).

### CI/CD (Optional)
- Build, test, and publish Docker image per commit.
- Run EF migrations at startup via an init container or migration job.
- Promote through DEV → UAT → PROD with environment‑specific config.

### Troubleshooting
- **Swagger not loading**: ensure `Swagger.Enable` is true for the environment, check HTTPS URLs.
- **SQL connectivity**: verify instance name, `TrustServerCertificate=True` for dev, ensure SQL Browser service is running.
- **`AutoMapperMappingException`**: a profile may be missing; confirm the profile is registered and any `Items["isArabic"]` flag is passed during `ProjectTo`.
- **Tenant data looks mixed**: confirm `X-Tenant-Id` header is set on every request; verify global query filter is active.

### Roadmap
- Authentication/Authorization (JWT + roles).
- Audit logs.
- Rate limiting.
- Caching for menu endpoints.
- Webhooks for order events.

### License
Proprietary — internal demo and client evaluation. Contact the owner for licensing terms.

---
