# Inventory Backend API

A production-grade multi-tenant inventory management REST API built with .NET 10, following Clean Architecture principles.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Authentication](#authentication)
- [Key Concepts](#key-concepts)
- [Testing](#testing)
- [Database](#database)

---

## Overview

This API provides inventory management capabilities including product tracking, stock movements, purchases, sales, branch and warehouse management, audit history, and multi-tenant business isolation — all secured with JWT Bearer authentication.

**Tech stack:**

| Layer | Technologies |
|---|---|
| Runtime | .NET 10 |
| Database | PostgreSQL 16+ via Npgsql EF Core 10 |
| Auth | JWT Bearer + BCrypt.Net |
| Mapping | AutoMapper 16 |
| Validation | FluentValidation 12 |
| Excel | ClosedXML |
| Docs | Swagger / Swashbuckle |
| Tests | xUnit + Moq |

---

## Architecture

The solution uses **Clean Architecture** with four layers plus a test project:

```
Inventory.Domain          ← Pure entities, enums, no external deps
Inventory.Application     ← Services, DTOs, validators, repository interfaces
Inventory.Infrastructure  ← EF Core, repositories, JWT, Excel reader, seeding
Inventory.API             ← Controllers, middleware, DI wiring (entry point)
Inventory.Tests           ← xUnit + Moq unit tests
```

**Request flow:**

```
Controller → Service → Repository → InventoryDbContext (PostgreSQL)
```

Each layer exposes a static `DependencyInjection` extension that registers its own services. `Program.cs` calls all of them.

---

## Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0)
- [PostgreSQL 16+](https://www.postgresql.org/download/)
- [EF Core CLI tools](https://learn.microsoft.com/en-us/ef/core/cli/dotnet): `dotnet tool install --global dotnet-ef`

---

## Getting Started

### 1. Clone and restore

```bash
git clone <repository-url>
cd inventory-backend
dotnet restore Inventory.sln
```

### 2. Configure the database

Update the connection string in `Inventory.API/appsettings.json` (see [Configuration](#configuration)).

Create the database and apply migrations:

```bash
dotnet ef database update --project Inventory.Infrastructure --startup-project Inventory.API
```

### 3. Run

```bash
dotnet run --project Inventory.API
```

The API starts on `https://localhost:5001` / `http://localhost:5000`. Swagger UI is available at `/swagger` in development.

On startup, `DatabaseSeeder` runs automatically (idempotent) and seeds:

| Entity | Count |
|---|---|
| Business | 1 |
| Roles | 2 (Admin, Manager) |
| Measures | 6 |
| Categories | 4 |
| Products | 6 |
| Warehouses | 2 |
| Branches | 2 |
| Users | 2 (admin / manager) |

### 4. Build and format

```bash
# Build
dotnet build Inventory.sln

# Check formatting (required before merging)
dotnet format Inventory.sln --verify-no-changes --severity info

# Fix formatting
dotnet format Inventory.sln --severity info
```

---

## Configuration

`Inventory.API/appsettings.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=inventory;Username=postgres;Password=mysecretpassword"
  },
  "JwtSettings": {
    "SecretKey": "YourSuperSecretKeyThatIsAtLeast32CharactersLong!",
    "Issuer": "InventoryAPI",
    "Audience": "InventoryAPI",
    "ExpirationInMinutes": 60
  },
  "Cors": {
    "AllowedOrigins": ["https://yourfrontend.com"]
  }
}
```

> In development, CORS allows any origin. In production, set `Cors:AllowedOrigins` in configuration.

> **Important:** Replace `SecretKey` with a strong secret before deploying. Never commit real credentials.

---

## API Reference

All protected endpoints require a JWT Bearer token in the `Authorization` header and a `businessId` GUID in the request header for tenant scoping.

### Authentication

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/login` | None | Obtain JWT + refresh token |
| POST | `/api/auth/refresh` | None | Rotate refresh token |

### Products

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/products` | Any | 200 `PaginatedList<ProductResponse>` |
| GET | `/api/products/{id}` | Any | 200 `ProductResponse` |
| POST | `/api/products` | Admin | 200 `ProductResponse` |
| PUT | `/api/products/{id}` | Admin | 204 |
| DELETE | `/api/products/{id}` | Admin | 204 |
| POST | `/api/products/bulk-upload` | Admin | 200 (Excel import) |
| GET | `/api/products/template` | Any | 200 (Excel template file) |

### Categories

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/categories` | Admin | 200 `PaginatedList<CategoryResponse>` |
| GET | `/api/categories/{id}` | Any | 200 `CategoryResponse` |
| POST | `/api/categories` | Admin | 200 `CategoryResponse` |
| PUT | `/api/categories/{id}` | Admin | 204 |
| DELETE | `/api/categories/{id}` | Admin | 204 |

### Branches

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/branches` | Any | 200 `PaginatedList<BranchResponse>` |
| GET | `/api/branches/{id}` | Any | 200 `BranchResponse` |
| POST | `/api/branches` | Admin | 200 `BranchResponse` |
| PUT | `/api/branches/{id}` | Admin | 204 |
| DELETE | `/api/branches/{id}` | Admin | 204 |
| GET | `/api/branches/{id}/products` | Any | 200 `PaginatedList<ProductResponse>` |
| POST | `/api/branches/{id}/products` | Any | 204 |
| PUT | `/api/branches/{id}/products` | Admin | 204 |
| DELETE | `/api/branches/{id}/products` | Admin | 204 |
| GET | `/api/branches/{id}/products/doesnt-exist` | Any | 200 (products not yet in branch) |
| POST | `/api/branches/{id}/sales` | Any | 204 (transactional) |
| GET | `/api/branches/{id}/sales` | Any | 200 `PaginatedList<SaleResponse>` |

### Warehouses

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/warehouses` | Any | 200 `PaginatedList<WarehouseResponse>` |
| GET | `/api/warehouses/{id}` | Any | 200 `WarehouseResponse` |
| POST | `/api/warehouses` | Admin | 200 `WarehouseResponse` |
| PUT | `/api/warehouses/{id}` | Admin | 204 |
| DELETE | `/api/warehouses/{id}` | Admin | 204 |
| GET | `/api/warehouses/{id}/products` | Any | 200 `PaginatedList<WarehouseProductResponse>` |
| POST | `/api/warehouses/{id}/products` | Any | 204 |
| DELETE | `/api/warehouses/{id}/products` | Admin | 204 |
| GET | `/api/warehouses/{id}/products/doesnt-exist` | Any | 200 |

### Purchases

| Method | Endpoint | Role | Response |
|---|---|---|---|
| POST | `/api/purchases` | Any | 204 (transactional) |
| GET | `/api/purchases` | Any | 200 `PaginatedList<PurchaseResponse>` |

### Inventory Movements

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/inventorymovements` | Any | 200 `PaginatedList<InventoryMovementResponse>` |
| POST | `/api/inventorymovements` | Any | 200 `InventoryMovementResponse` |

Movement types: `Entry`, `Exit`, `Transfer` (resolved via Strategy pattern).

### Users

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/users` | Any | 200 `PaginatedList<UserResponse>` |
| GET | `/api/users/{id}` | Any | 200 `UserResponse` |
| POST | `/api/users` | Admin | 200 `UserResponse` |
| PUT | `/api/users/{id}` | Admin | 204 |
| DELETE | `/api/users/{id}` | Admin | 204 |

### Customers

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/customers` | Any | 200 `PaginatedList<CustomerResponse>` |
| POST | `/api/customers` | Any | 200 `CustomerResponse` |
| PUT | `/api/customers/{id}` | Any | 204 |

### Providers

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/providers` | Admin | 200 `PaginatedList<ProviderResponse>` |
| GET | `/api/providers/{id}` | Any | 200 `ProviderResponse` |
| POST | `/api/providers` | Admin | 200 `ProviderResponse` |
| PUT | `/api/providers/{id}` | Admin | 204 |
| DELETE | `/api/providers/{id}` | Admin | 204 |

### Other Endpoints

| Method | Endpoint | Role | Response |
|---|---|---|---|
| GET | `/api/businesses` | Admin | 200 `PaginatedList<BusinessResponse>` |
| POST | `/api/businesses` | Admin | 200 `BusinessResponse` |
| GET | `/api/dashboard/today` | Any | 200 `DashboardResponse` |
| GET | `/api/roles` | Any | 200 `List<RoleResponse>` |
| GET | `/api/measures` | Any | 200 `List<MeasureResponse>` |
| GET | `/api/audithistory` | Any | 200 `PaginatedList<AuditHistoryResponse>` |

---

## Authentication

The API uses JWT Bearer tokens with refresh token rotation.

### Login

```http
POST /api/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "yourpassword"
}
```

Response:

```json
{
  "accessToken": "<jwt>",
  "refreshToken": "<token>",
  "expiresIn": 3600
}
```

### Using the token

Include the token in subsequent requests:

```http
Authorization: Bearer <accessToken>
businessId: <your-business-guid>
```

### Refresh token rotation

Access tokens expire after 60 minutes. Refresh tokens rotate every 7 days:

```http
POST /api/auth/refresh
Content-Type: application/json

{
  "refreshToken": "<token>"
}
```

Returns a new `accessToken` + `refreshToken` pair. The old refresh token is revoked immediately.

---

## Key Concepts

### Multi-tenancy

Every major entity is scoped to a `BusinessId`. Controllers accept `businessId` as a required request header (`[FromHeader][BindRequired]`). Every repository query filters by `businessId` — failing to do so leaks cross-tenant data.

### Soft deletes

Records are never hard-deleted. Setting `IsDeleted = true` hides them from all queries via a global EF Core query filter.

### Pagination

List endpoints return `PaginatedList<T>`. Pass `pageIndex` / `pageSize` (1-based) as query parameters.

### Inventory movement strategies

Stock changes are handled via the Strategy pattern (`MovementStrategyResolver`):

- **Entry** — adds stock to a warehouse or branch
- **Exit** — removes stock from a warehouse or branch
- **Transfer** — moves stock between locations

### Stock management

`BranchProduct` and `WarehouseProduct` expose `AddStock(int)` and `ReduceStock(int)` domain methods with quantity validation. Always use these instead of assigning `Stock` directly.

### Builder pattern

All entities are constructed via builders in `Inventory.Domain/Entities/Builders/`. Never use object initializers directly.

### Audit history

All write operations (purchases, sales, inventory movements) are audited automatically via `AuditHistoryService` and stored with the acting user's ID and a timestamp.

### Bulk product import

`POST /api/products/bulk-upload` accepts an `.xlsx` file. Use `GET /api/products/template` to download the expected column format before uploading.

### Error responses

All errors are handled by `ExceptionHandlingMiddleware`:

| Exception | HTTP Status |
|---|---|
| `ValidationException` | 400 (field errors in `extensions.errors`) |
| `KeyNotFoundException` | 404 |
| `UnauthorizedAccessException` | 401 |
| `ArgumentException` / `InvalidOperationException` | 400 |
| Anything else | 500 |

---

## Testing

```bash
# Run all tests
dotnet test

# Run a specific test class
dotnet test --filter "FullyQualifiedName~ProviderServiceTests"
```

Tests use xUnit with Moq. Each service has unit tests covering the happy path and `KeyNotFoundException` scenarios. Dependencies (`IRepository`, `IMapper`, `IValidator`) are mocked.

Test naming convention: `MethodName_Condition_ExpectedResult`.

---

## Database

### Migrations

```bash
# Add a new migration
dotnet ef migrations add <MigrationName> --project Inventory.Infrastructure --startup-project Inventory.API

# Apply pending migrations
dotnet ef database update --project Inventory.Infrastructure --startup-project Inventory.API
```

### ID strategy

| Entity | PK type |
|---|---|
| Category, Product, Measure, Role | `int` (auto-increment) |
| Everything else | `Guid` (`uuid_generate_v4()`) |

### Key entities

| Entity | Description |
|---|---|
| `Business` | Tenant root |
| `Branch` / `Warehouse` | Physical locations with stock |
| `BranchProduct` / `WarehouseProduct` | Junction tables tracking stock per location |
| `Purchase` / `Sale` | Transactional records with detail lines |
| `InventoryMovement` | Tracks every stock change |
| `AuditHistory` | Immutable audit log |
| `RefreshToken` | Rotation-based auth tokens |
| `BusinessSaleCounter` | Per-business sale totals (raw SQL, no soft-delete) |
