Here’s a polished, drop-in `README.md` you can paste at the root of your repo. Tweak names/ports as you wish.

```markdown
# ERestaurant API

Multi-tenant restaurant API for **Materials**, **Additional Materials**, **Combos**, and **Orders** built with **ASP.NET Core (.NET 8)** and **EF Core**.  
It enforces tenant isolation (headers-based), provides rich seed data, DB-level constraints, auto-included navigations, and consistent error payloads.

---

## Table of Contents
- [Features](#features)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Configuration](#configuration)
- [Run](#run)
- [Database & Seeding](#database--seeding)
- [Headers & Localization](#headers--localization)
- [API Overview](#api-overview)
- [Examples](#examples)
- [Error Responses](#error-responses)
- [Troubleshooting](#troubleshooting)
- [Notes](#notes)
- [License](#license)

---

## Features
- **Multi-tenancy isolation**
  - Middleware reads `X-Tenant-Id` and applies a global query filter (`TenantId == header && IsDeleted == false`).
  - Composite foreign keys `{Id, TenantId}` prevent cross-tenant references at the DB level.
- **Two-language responses**
  - `X-Accept-Language: en | ar` affects mapper output for names/text.
- **Seed data (per tenant)**
  - 5 tenants out-of-the-box: Burgers, Pizza, Shawarma, Coffee&Tea, Desserts.
  - Deterministic GUIDs for repeatable seeding.
- **AutoInclude navigations**
  - Core navigation properties are eager-loaded by default where helpful.
- **DB constraints**
  - Prices > 0, Quantities > 0, and Tax ∈ [0,1] enforced at the database.
- **ProblemDetails errors**
  - Consistent error envelopes for validation, FK conflicts, and server errors.

---

## Project Structure
```

ERestaurant.sln
│
├─ ERestaurant.API/                 # Web API (controllers, middleware, swagger)
│  └─ Controllers/
│
├─ ERestaurant.Application/         # Use cases, DTOs, services
│  ├─ DTOs/
│  └─ Services/
│
├─ ERestaurant.Domain/              # Entities, enums, base classes
│  ├─ Entity/
│  └─ Enums/
│
└─ ERestaurant.Infrastructure/      # DbContext, Repositories, Seed, Auditing
├─ DatabaseContext/
├─ HelperClass/
└─ Repositories/

````

---

## Requirements
- **.NET 8 SDK**
- **SQL Server** (LocalDB / Express / Developer / Azure SQL)
- (Optional) EF Core CLI tools:
  ```bash
  dotnet tool install --global dotnet-ef
````

---

## Configuration

Update `appsettings.Development.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\MSSQLLocalDB;Database=ERestaurant;Trusted_Connection=True;MultipleActiveResultSets=true;TrustServerCertificate=True"
  },
  "App": {
    "RecreateDatabaseOnStart": false
  },
  "Logging": { "LogLevel": { "Default": "Information" } }
}
```

**Port / URLs**
Check `Properties/launchSettings.json`:

```json
"applicationUrl": "https://localhost:7034;http://localhost:5034"
```

Use the same port in your requests (examples below use `7034`).

---

## Run

```bash
dotnet restore
dotnet build
dotnet run --project ERestaurant.API
```

Swagger UI:
`https://localhost:7034/swagger`

---

## Database & Seeding

There are two dev flows:

### A) Run migrations (recommended)

```bash
dotnet ef database update --project ERestaurant.Infrastructure --startup-project ERestaurant.API
```

This creates the schema and applies **SeedData** automatically from `OnModelCreating`.

### B) Recreate DB each run (dev-only)

Set `"App:RecreateDatabaseOnStart": true` and call a dev helper on startup (example):

```csharp
// Program.cs (Development only)
if (app.Environment.IsDevelopment() && builder.Configuration.GetValue<bool>("App:RecreateDatabaseOnStart"))
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<ERestaurantDbContext>();
    await db.Database.EnsureDeletedAsync();
    await db.Database.MigrateAsync();
}
```

> **Warning**: Use only in development; it **deletes** the DB on each run.

---

## Headers & Localization

Every request **must** include:

```
X-Tenant-Id: 1 | 2 | 3 | 4 | 5
X-Accept-Language: en | ar
```

* Tenant middleware rejects missing/invalid `X-Tenant-Id`.
* The mapper uses `X-Accept-Language` to choose Arabic vs English fields where applicable.

---

## API Overview

Base URL (dev): `https://localhost:7034`

### Material

* `GET    /API/Material`
* `GET    /API/Material/{id}`
* `POST   /API/Material`
* `PUT    /API/Material/{id}`
* `DELETE /API/Material/{id}`

### AdditionalMaterial

* `GET    /API/AdditionalMaterial`
* `GET    /API/AdditionalMaterial/{id}`
* `POST   /API/AdditionalMaterial`
* `PUT    /API/AdditionalMaterial/{id}`
* `DELETE /API/AdditionalMaterial/{id}`

### Combo

* `GET    /API/Combo`
* `GET    /API/Combo/WithAddition`
* `GET    /API/Combo/{id}`
* `POST   /API/Combo`
* `PUT    /API/Combo/{id}`
* `DELETE /API/Combo/{id}`

**Combo Materials (within a Combo)**

* `POST   /API/Combo/{comboId}/Material/{materialId}?quantity=1`
* `PUT    /API/Combo/{comboId}/Material/{materialId}?quantity=2`
* `DELETE /API/Combo/{comboId}/Material/{materialId}`

### Order

* `GET    /API/Order`
* `GET    /API/Order/{id}`
* `POST   /API/Order`
* `PUT    /API/Order/{id}`
* `DELETE /API/Order/{id}`

**Order Items (within an Order)**

* `POST   /API/Order/{orderId}/Item/Material/{materialId}?quantity=1`
* `POST   /API/Order/{orderId}/Item/Combo/{comboId}?quantity=1`
* `POST   /API/Order/{orderId}/Item/Additional/{additionalMaterialId}?quantity=1`
* `DELETE /API/Order/{orderId}/Item/OrderItem/{referenceId}`

---

## Examples

> Replace `{{PORT}}` with your dev port (e.g., `7034`).
> Replace `{{TENANT}}` with a valid tenant (1–5).
> Use `search` endpoints to fetch IDs from seed data (e.g., search “Brioche Bun” under tenant 1).

### List materials (tenant 1, English)

```bash
curl -k -H "X-Tenant-Id: 1" -H "X-Accept-Language: en" \
"https://localhost:{{PORT}}/API/Material?pageNumber=1&pageSize=10"
```

### Get material by name (search like)

```bash
curl -k -H "X-Tenant-Id: 1" -H "X-Accept-Language: en" \
"https://localhost:{{PORT}}/API/Material?searchNameQuery=Brioche"
```

### Create combo (valid, tenant 1)

```bash
curl -k -X POST "https://localhost:{{PORT}}/API/Combo" \
-H "Content-Type: application/json" \
-H "X-Tenant-Id: 1" -H "X-Accept-Language: en" \
-d '{
  "nameEn": "Classic Burger Meal",
  "nameAr": "وجبة برجر كلاسيك",
  "price": 5.250,
  "tax": 0.160,
  "imageUrl": "https://www.example.com/1/combo/classic-burger-meal",
  "isActive": true,
  "material": [
    { "materialId": "<BriocheBunId_T1>",   "quantity": 1 },
    { "materialId": "<BeefPattyId_T1>",    "quantity": 1 },
    { "materialId": "<CheddarSliceId_T1>", "quantity": 1 },
    { "materialId": "<LettuceId_T1>",      "quantity": 25 },
    { "materialId": "<TomatoId_T1>",       "quantity": 30 },
    { "materialId": "<FriesId_T1>",        "quantity": 120 }
  ]
}'
```

### Create combo (error: material is from another tenant)

```bash
curl -k -X POST "https://localhost:{{PORT}}/API/Combo" \
-H "Content-Type: application/json" \
-H "X-Tenant-Id: 1" -H "X-Accept-Language: en" \
-d '{
  "nameEn": "Wrong Tenant Combo",
  "nameAr": "كومبو مستأجر آخر",
  "price": 3.000,
  "tax": 0.160,
  "imageUrl": "https://www.example.com/1/combo/wrong-tenant",
  "isActive": true,
  "material": [
    { "materialId": "<MozzarellaId_T2>", "quantity": 1 }  // Tenant 2 ID under Tenant 1
  ]
}'
```

### Add material to order (valid)

```bash
curl -k -X POST \
"https://localhost:{{PORT}}/API/Order/<OrderId_T1>/Item/Material/<MaterialId_T1>?quantity=2" \
-H "X-Tenant-Id: 1" -H "X-Accept-Language: en"
```

### Add material to order (error: negative quantity)

```bash
curl -k -X POST \
"https://localhost:{{PORT}}/API/Order/<OrderId_T1>/Item/Material/<MaterialId_T1>?quantity=-5" \
-H "X-Tenant-Id: 1" -H "X-Accept-Language: en"
```

### Add combo/additional from **different** tenant (expect FK/validation error)

```bash
# Combo from tenant 2 into tenant 1 order → error
curl -k -X POST \
"https://localhost:{{PORT}}/API/Order/<OrderId_T1>/Item/Combo/<ComboId_T2>?quantity=1" \
-H "X-Tenant-Id: 1" -H "X-Accept-Language: en"

# Additional material from tenant 3 into tenant 1 order → error
curl -k -X POST \
"https://localhost:{{PORT}}/API/Order/<OrderId_T1>/Item/Additional/<AdditionalId_T3>?quantity=1" \
-H "X-Tenant-Id: 1" -H "X-Accept-Language: en"
```

---

## Error Responses

The API returns **ProblemDetails** style payloads. Typical examples:

**Validation (400)**

```json
{
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "quantity": ["Quantity must be greater than 0."]
  },
  "traceId": "..."
}
```

**Foreign key conflict (409)**

```json
{
  "title": "Database Update Error",
  "status": 409,
  "detail": "The INSERT statement conflicted with the FOREIGN KEY constraint 'FK_ComboMaterial_Material_MaterialId_TenantId'...",
  "traceId": "..."
}
```

**Not found (404)**

```json
{
  "title": "Not Found",
  "status": 404,
  "detail": "material not found",
  "traceId": "..."
}
```

---

## Troubleshooting

### “Cannot be tracked because another instance with the same key is already being tracked”

* Don’t attach full graphs with tracked duplicates. Update only the **aggregate root** (e.g., `Order`) and **set only the FK IDs** for children.
* If you assembled an in-memory graph for updating, **null** navigation properties and keep only `...Id` values before calling `Update`.
* Prefer `Query(asNoTracking: true)` for reads; then attach only what you modify.

### Cross-tenant errors (409)

* Ensure `X-Tenant-Id` matches all referenced entities (composite FKs enforce `{Id, TenantId}`).
* Don’t mix IDs from other tenants in create/update calls.

### Negative or zero values rejected

* DB **CHECK CONSTRAINTS** deny negative or zero quantities/prices, and tax outside `[0,1]`.

### Port mismatch

* Align requests with `launchSettings.json` `applicationUrl` or run with:

  ```bash
  dotnet run --project ERestaurant.API --urls "https://localhost:7034"
  ```

---

## Notes

* AutoInclude navigations are configured for common reads (e.g., `Order -> OrderItem`, `Combo -> ComboMaterial`).
* Seed data themes per tenant:

  1. Burgers
  2. Pizza
  3. Shawarma
  4. Coffee&Tea
  5. Desserts

Use name searches to fetch IDs you want to reference in payloads.

---

## License

Add your license here (MIT, Apache-2.0, etc.).

```

want me to include exact sample IDs from your seed (e.g., “Brioche Bun” under tenant 1) directly in the examples? I can add a “Known IDs” section to the README.
```
