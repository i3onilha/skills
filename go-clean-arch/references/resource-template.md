# Resource Template Reference

This file provides templates and patterns for adding a new resource to an existing Clean Architecture Go project. Use this when the user says something like "Create a PRODUCT resource" or "Add a Category entity."

> **⚠️ Module name substitution:** Replace `go-clean-example` with your actual module name from `go.mod` in all import paths throughout this file. Copy-pasting these templates without substitution will produce uncompilable code.

---

## Step-by-step workflow

### 1. Design the schema

Add tables to `internal/repository/mysql/schema.sql`. Follow this pattern:

```sql
-- ⚠️ WARNING: DROP TABLE IF EXISTS destroys all existing data in the table.
-- This pattern is for initial setup only. For incremental schema changes to an
-- existing database, use ALTER TABLE instead to avoid data loss.
DROP TABLE IF EXISTS `<resource_plural>`;
CREATE TABLE `<resource_plural>` (
  `<resource_id>_id` int NOT NULL AUTO_INCREMENT,
  -- columns based on fields
  PRIMARY KEY (`<resource_id>_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

Include seed data if helpful for testing.

### 2. Write SQL queries

Create `internal/repository/mysql/<resource>/` directory (lowercase, singular).

Create `<resource>.sql` with named queries:

```sql
-- name: Get<Resource>ByID :one
SELECT <columns> FROM <table>
WHERE <resource_id>_id = ?;

-- name: Get<Resources>By<Condition> :many
SELECT <columns> FROM <table>
WHERE <condition>
LIMIT ? OFFSET ?;
```

> **Pagination:** All `:many` queries must include `LIMIT ? OFFSET ?` to prevent unbounded result sets that can cause memory exhaustion, slow responses, or denial-of-service.

sqlc directives:
- `:one` — returns a single row
- `:many` — returns multiple rows
- `:exec` — execute, no return
- `:execrows` — execute, return rows affected

Then update `sqlc.yaml` to include the new `.sql` file. Add a new entry to the `sql` array:

```yaml
  - engine: "mysql"                                    # ← Append this to the existing sql: array
    queries: "internal/repository/mysql/<resource>/<resource>.sql"
    schema: "internal/repository/mysql/schema.sql"
    gen:
      go:
        package: "<resource>"
        out: "internal/repository/mysql/<resource>"
        emit_json_tags: true
        overrides:
          - db_type: "decimal"
            go_type:
              import: "github.com/shopspring/decimal"
              type: "Decimal"
```

Run `sqlc generate` to produce the Go files.

### 3. Domain layer

`internal/domain/<resource>_domain.go`:

```go
package domain

import "errors"

// Sentinel errors for clean architecture separation.
// The repository maps sql.ErrNoRows → ErrNotFound so controllers
// never depend on database-specific errors.
var ErrNotFound = errors.New("not found")

// Core business entity — decoupled from API shapes and DB models
type <Resource> struct {
    <Resource>ID int32   `json:"<resource_id>_id,omitempty"`
    // ... fields based on what the user described
}

// Business logic methods
func (d *<Resource>) <CalculateSomething>(...) <ReturnType> {
    // business rule implementation
}

// Response shapes returned from repository
type Get<Resource>Response struct {
    // fields for a single-resource response
}

type Get<Resource>sResponse struct {
    Items []*<Resource> `json:"items,omitempty"`
}
```

Key rules:
- Domain entities contain business logic (validations, calculations)
- Use `omitempty` on JSON tags for optional fields
- Response structs mirror what the repository returns after mapping from sqlc models
- Use Go convention `ID` suffix (e.g., `UserID`, `OrderID`), not `Id`

### 4. DTO layer

`internal/dto/<resource>_dto.go`:

```go
package dto

// API response shape — what the HTTP endpoint returns
type <Resource>Response struct {
    // flattened/combined fields for the API consumer
}

type <Resource>Item struct {
    // individual item in a list response
}
```

Key rules:
- DTOs are for API serialization — decoupled from domain models
- Use DTOs when the API response shape differs from the domain entity (e.g., computed fields, nested data flattened)
- If the response shape matches the domain exactly, you can skip DTOs and return domain directly

### 5. Repository interface

Add to `internal/repository/repository.go` (or create a new file like `internal/repository/<resource>_repository.go`):

```go
package repository

import (
    "context"
    "go-clean-example/internal/domain"
)

type <Resource>Repository interface {
    Get<Resource>ByID(ctx context.Context, id int32) (*domain.Get<Resource>Response, error)
    Get<Resource>sBy<Condition>(ctx context.Context, param <Type>) (*domain.Get<Resource>sResponse, error)
}
```

Key rules:
- Interfaces accept `context.Context` as first param
- Return domain types, not sqlc types or DTOs
- Name methods consistently: use `Get<Resource>ByID` (matching the usecase layer naming)
- Do **not** add context timeouts here — set the timeout once at the usecase level

### 6. Repository implementation

`internal/repository/mysql/<resource>_repository.go`:

```go
package mysql

import (
    "context"
    "database/sql"
    "errors"
    "go-clean-example/internal/domain"
    "go-clean-example/internal/repository"
    "go-clean-example/internal/repository/mysql/<resource>"
)

type <resource>Repository struct {
    query <resource>.Queries
}

func New<Resource>Repository(db *sql.DB) repository.<Resource>Repository {
    return &<resource>Repository{query: *<resource>.New(db)}
}

func (r *<resource>Repository) Get<Resource>ByID(ctx context.Context, id int32) (*domain.Get<Resource>Response, error) {
    result, err := r.query.Get<Resource>ByID(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound
        }
        return nil, err
    }
    return &domain.Get<Resource>Response{
        // map sqlc result to domain
        <Resource>ID: result.<Resource>ID,
        // ...
    }, nil
}

func (r *<resource>Repository) Get<Resource>sBy<Condition>(ctx context.Context, param <Type>) (*domain.Get<Resource>sResponse, error) {
    results, err := r.query.Get<Resource>sBy<Condition>(ctx, param)
    if err != nil {
        return nil, err
    }
    items := make([]*domain.<Resource>, 0, len(results))
    for _, row := range results {
        items = append(items, &domain.<Resource>{
            // map each row
            // For sql.NullString fields, check .Valid before accessing .String:
            // Field: func() string { if row.Field.Valid { return row.Field.String }; return "" }(),
        })
    }
    return &domain.Get<Resource>sResponse{Items: items}, nil
}
```

Key rules:
- Do **not** add context timeouts here — set the timeout once at the usecase (entry) level
- Map sqlc types to domain types (handle `sql.NullString` → check `.Valid` before `.String`)
- The struct is unexported; the constructor returns the interface

### 7. Usecase interface

Add to `internal/usecase/usecase.go` (or create `internal/usecase/<resource>_usecase.go` with the interface):

```go
package usecase

import (
    "context"
    "go-clean-example/internal/dto"  // or domain if no DTO
)

type <Resource>Usecase interface {
    Get<Resource>ByID(ctx context.Context, id int32) (*dto.<Resource>Response, error)
    // other methods the user needs
}
```

Key rules:
- Interfaces return DTOs (or domain if no DTO layer)
- Accept context + business-level parameters (not HTTP-specific types)

### 8. Usecase implementation

`internal/usecase/<resource>_usecase.go`:

```go
package usecase

import (
    "context"
    "time"
    "go-clean-example/internal/dto"
    "go-clean-example/internal/repository"
    "github.com/adityaeka26/go-pkg/logger"
)

type <resource>Usecase struct {
    logger          *logger.Logger
    <resource>Repo  repository.<Resource>Repository
}

func New<Resource>Usecase(logger *logger.Logger, repo repository.<Resource>Repository) <Resource>Usecase {
    return &<resource>Usecase{
        logger:         logger,
        <resource>Repo: repo,
    }
}

func (u *<resource>Usecase) Get<Resource>ByID(ctx context.Context, id int32) (*dto.<Resource>Response, error) {
    // Set timeout once at the entry point.
    // When making multiple sequential repository calls, budget the timeout
    // based on the number of calls. For example, with 3 sequential calls,
    // a 5s total timeout gives ~1.6s per call. If one call is slow, the
    // remaining calls inherit a shrinking deadline and may fail prematurely.
    // Consider using errgroup for parallel fan-out when calls are independent.
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // 1. Fetch from repository
    info, err := u.<resource>Repo.Get<Resource>ByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // 2. Fetch related data if needed
    // NOTE: This requires a Get<Related> method defined on your repository interface
    // and implemented in the repository layer. Add it if your usecase needs it.
    // related, err := u.<resource>Repo.Get<Related>(ctx, id)
    // if err != nil {
    //     return nil, err
    // }

    // 3. Apply business logic (calculations, transformations)
    // Example: if your domain entity has a CalculateFinalPrice method:
    // items := make([]dto.<Resource>Item, len(related.Items))
    // for i, item := range related.Items {
    //     computed := item.CalculateFinalPrice()
    //     items[i] = dto.<Resource>Item{
    //         // map domain → DTO with business logic applied
    //     }
    // }

    // 4. Build response
    return &dto.<Resource>Response{
        // ...
    }, nil
}
```

Key rules:
- Constructor accepts interfaces (not concrete types) — enables testing
- The logger package (`github.com/adityaeka26/go-pkg/logger`) must be added to your project via `go get github.com/adityaeka26/go-pkg`. Ensure it's in your `go.mod` before using these templates.
- Set context timeout **once** at the usecase entry point; do not add nested timeouts in repository methods. When making multiple sequential calls, budget the total timeout across calls or use `errgroup` for parallel fan-out.
- Orchestrate repository calls, apply business logic, build DTO responses
- Any "fetch related data" step requires methods that you must define on your repository interface

### 9. Controller

`internal/controller/<resource>_controller.go`:

```go
package controller

import (
    "errors"
    "math"
    "net/http"
    "strconv"
    "go-clean-example/internal/domain"
    "go-clean-example/internal/usecase"
    "github.com/gin-gonic/gin"
)

type <Resource>Controller struct {
    logger  Logger
    usecase usecase.<Resource>Usecase
}

// Logger is a minimal logging interface to avoid hard-depending on a specific package.
// In practice, this is satisfied by most logger implementations (e.g. slog, zap, go-pkg/logger).
type Logger interface {
    Error(msg string, keysAndValues ...any)
}

func New<Resource>Controller(logger Logger, usecase usecase.<Resource>Usecase) *<Resource>Controller {
    return &<Resource>Controller{logger: logger, usecase: usecase}
}

func (h *<Resource>Controller) Get<Resource>ByID(c *gin.Context) {
    idParam := c.Param("id")
    id, err := strconv.Atoi(idParam)
    if err != nil || id <= 0 {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid ID"})
        return
    }
    if id > math.MaxInt32 {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid ID"})
        return
    }

    resp, err := h.usecase.Get<Resource>ByID(c.Request.Context(), int32(id))
    if errors.Is(err, domain.ErrNotFound) {
        c.JSON(http.StatusNotFound, gin.H{"error": "<resource> not found"})
        return
    }
    if err != nil {
        h.logger.Error("failed to retrieve <resource>", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to retrieve <resource>"})
        return
    }
    c.JSON(http.StatusOK, resp)
}
```

Key rules:
- Parse and validate path/query params (return 400 on invalid input)
- Map `domain.ErrNotFound` → 404, other errors → 500
- Delegate all business logic to the usecase — controllers are thin
- Log unexpected errors at the error level before returning 500 responses
- **Integer IDs:** The template casts to `int32` with a bounds check (`math.MaxInt32`). If your IDs exceed 2,147,483,647, use string IDs instead.
- **String IDs (UUIDs, slugs):** Use `c.Param("id")` directly without `strconv.Atoi`. Pass the string to your usecase without conversion.

### 10. Wire into server.go

Update `cmd/server/server.go`:

In `fx.Provide(...)`, add:
```go
mysql.New<Resource>Repository,
usecase.New<Resource>Usecase,
controller.New<Resource>Controller,
```

For route registration, avoid unbounded parameter growth on `registerRoutes` by defining a `RegisterRoutes` method on each controller. This keeps the function signature stable as you add resources.

Define a `Registrable` interface:
```go
type Registrable interface {
    RegisterRoutes(group *gin.RouterGroup)
}
```

Then each controller implements it:
```go
func (h *<Resource>Controller) RegisterRoutes(group *gin.RouterGroup) {
    res := group.Group("/<resource>s")
    {
        res.GET("/:id", h.Get<Resource>ByID)
    }
}
```

And `registerRoutes` becomes:
```go
func registerRoutes(
    router *gin.Engine,
    controllers ...Registrable,
) {
    api := router.Group("/api")
    for _, c := range controllers {
        c.RegisterRoutes(api)
    }
}
```

Invoke in `fx.Invoke`:
```go
fx.Invoke(registerRoutes,
    controller.NewUserController,
    controller.New<Resource>Controller,
),
```

If you prefer the simpler approach (acceptable for small projects with few resources), you can pass controllers directly:
```go
func registerRoutes(
    router *gin.Engine,
    userController *controller.UserController,
    <resource>Controller *controller.<Resource>Controller,
) {
    user := router.Group("/api/users")
    {
        user.GET("/:id", userController.GetUser)
    }

    <resource>s := router.Group("/api/<resource>s")
    {
        <resource>s.GET("/:id", <resource>Controller.Get<Resource>ByID)
    }
}
```

> **Recommendation:** Use the `Registrable` interface pattern when you expect more than ~5 resources to keep the codebase maintainable.

### 11. Verify

1. `go mod tidy` — resolve any new imports
2. `sqlc generate` — produce sqlc Go files from `.sql`
3. `go build ./cmd/server/server.go` — must compile cleanly

---

## Example: The USER resource (with orders + discounts)

The existing project demonstrates this pattern with a USER resource that:
- Fetches user profile info (`GetUserByID :one`)
- Fetches user orders with discount data via LEFT JOIN with aggregation (`GetOrdersByUserID :many`)
- Calculates `final_price = quantity * price * (1 - discount_percent/100)` in the domain layer
- Returns a combined response with user info + order items with computed prices
- Supports pagination via `LIMIT ? OFFSET ?` parameters

Use this as a reference when creating resources that involve joins and calculated fields.
