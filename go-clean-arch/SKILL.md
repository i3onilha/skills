---
name: go-clean-arch
description: >
  Scaffold a Go REST API project following Clean Architecture with sqlc, Gin, Uber FX (DI),
  Viper config, Docker, and Air hot-reload. Supports all sqlc engines: PostgreSQL, MySQL, and SQLite.
  Use this skill whenever the user wants to create a new Go Clean Architecture project from scratch,
  bootstrap a go-clean-arch-style repo, or add a new domain resource (e.g. "Create a USER resource",
  "add a Product entity", "scaffold an Order service", "set up a Go API with sqlc and clean layers").
  Also use when the user mentions clean architecture in Go, layered Go projects,
  sqlc code generation for Go, or wants Go boilerplate with domain/dto/controller/usecase/repository layers.
---

# Go Clean Architecture Skill

This skill handles two scenarios:
1. **Full boilerplate** — the user wants a brand-new Clean Architecture Go project from scratch.
2. **Add a resource** — the user wants to create a new domain resource (e.g. "Create USER resource to get orders with discounts") inside an existing Clean Architecture project.

Determine which scenario applies, then follow the appropriate workflow below.

---

## Workflow A: Full Boilerplate

### Step 0: Ask for the app name and database engine

Before scaffolding, ask the user two questions:
1. **"What should the app/module be named?"** (e.g. `my-api`, `go-sales`, `inventory-service`). Use their answer as the module name throughout all files (`go.mod`, imports, etc.). If they don't have a preference, suggest a reasonable default based on their project context.
2. **"Which database engine should I use?"** Offer the three sqlc-supported options:
   - **PostgreSQL** (stable, recommended for production) — uses `github.com/lib/pq` or `github.com/jackc/pgx/v5` driver
   - **MySQL** (stable) — uses `github.com/go-sql-driver/mysql` driver
   - **SQLite** (beta, good for local/dev or embedded apps) — uses `modernc.org/sqlite` driver

If the user doesn't have a preference, default to **PostgreSQL**.

### Step 1: Scaffold the project

Follow the detailed prompt in `references/full-boilerplate.md`, adapting all database-specific content for the chosen engine. This will scaffold:

- Go module (`go.mod`), `cmd/server/server.go` with Uber FX DI
- All layer directories: `internal/controller/`, `internal/usecase/`, `internal/domain/`, `internal/dto/`, `internal/repository/`
- Database connection pool (`internal/repository/<engine>/conn.go`) — engine-specific setup
- Database schema with seed data (`schema.sql`) — engine-specific SQL dialect
- sqlc config (`sqlc.yaml`) with the correct engine and initial `.sql` query file
- Config loading via Viper (`config/config.go`)
- Docker Compose, Dockerfile, `.env.example`, `.air.toml`, `.gitignore`
- A working `USER` resource as the example (with orders + discounts)

After creating all files:
1. Run `go mod tidy`
2. Run `sqlc generate`
3. Run `gofmt -s -w .` to simplify and format all Go files
4. Verify the build: `go build ./cmd/server/server.go`

**Then ask the user**: "The boilerplate is ready. Would you like me to generate mock repository implementations for testing? This will create mock repositories with embedded JSON data files, enabling fast, database-free unit tests."

If the user confirms, follow the mock generation workflow in **Workflow C: Generate Mocks** below.

After the boilerplate (and optional mocks) are ready, ask: "Would you like me to add any additional resources on top of it?"

---

## Workflow B: Add a Resource

When the user already has the boilerplate and wants to add a new resource (e.g. "Create a PRODUCT resource"), follow these steps:

### Step 1: Understand the resource

Extract from the user's request:
- **Resource name** (e.g. `Product`, `Order`, `Category`) — singular, PascalCase
- **Fields** and their types — infer from the description or ask for clarification
- **Endpoints** the user wants (e.g. `GET /products/:id`, `POST /products`)
- **Business logic** — any calculations, validations, or joins with other tables
- **Database relationships** — does it reference other entities? Foreign keys?

If the user's request is vague (e.g. "Create a USER resource to get orders with discounts"), infer reasonable fields from the description. The example implies:
- A `users` table with profile info
- An `orders` table linked to users
- A `discounts` table linked to orders
- An endpoint that returns user info + orders with calculated final prices

### Step 2: Design the schema

Write the SQL schema additions in `internal/repository/mysql/schema.sql`:
- Add the new table(s) with appropriate columns, constraints, and foreign keys
- Include seed data for testing

### Step 3: Write the SQL queries

Create `internal/repository/mysql/<resource>/` (e.g. `internal/repository/mysql/product/`) with:
- `<resource>.sql` — all SQL queries the resource needs (use sqlc directives: `:one`, `:many`, `:exec`)
- Use `COALESCE`, `JOIN`, etc. as needed for calculated fields

> **Warning:** The `<resource>/` directory is reserved for sqlc-generated code (`models.go`, `db.go`, `<resource>.sql.go`). Do **not** place your `<resource>_repository.go` implementation file inside this directory — it belongs in `internal/repository/mysql/` as a sibling, not a child.

> **Note:** The `.sql` file will be picked up after you update `sqlc.yaml` in Step 6. The `sqlc generate` command runs in the final verify step (Step 7).

### Step 4: Create all layer files

Follow the pattern in `references/resource-template.md` for each file. The key files are:

| File | Purpose |
|------|---------|
| `internal/domain/<resource>_domain.go` | Core entities + business logic methods |
| `internal/dto/<resource>_dto.go` | API response/request shapes |
| `internal/repository/repository.go` | Add new interface methods (or create a new interface file) |
| `internal/repository/mysql/<resource>_repository.go` | Implements the repository interface using sqlc |
| `internal/usecase/usecase.go` | Add new usecase interface (or extend existing) |
| `internal/usecase/<resource>_usecase.go` | Orchestrates repository calls + business logic |
| `internal/controller/<resource>_controller.go` | HTTP handler — parses params, delegates to usecase, returns JSON |

### Step 5: Wire into server.go

Update `cmd/server/server.go`:
- Add the new repository constructor, usecase, and controller to `fx.Provide(...)`
- Add the new controller to `fx.Invoke(registerRoutes, ...)`
- Update `registerRoutes` to register the new endpoints

### Step 6: Update sqlc.yaml

Add the new `.sql` file path to `sqlc.yaml` under the `sql` array so sqlc picks it up on the next run.

### Step 7: Verify

1. Run `go mod tidy`
2. Run `sqlc generate` (if you added/modified `.sql` files)
3. Run `gofmt -s -w .` to simplify and format all Go files
4. Build: `go build ./cmd/server/server.go`

**Then ask the user**: "Would you like me to generate mock repository implementations for the <Resource> resource alongside the real implementation? This will create mock repositories with embedded JSON data files, enabling fast, database-free unit tests."

If the user confirms, follow the mock generation workflow in **Workflow C: Generate Mocks** below.

---

## Workflow C: Generate Mocks

When the user confirms they want mock repository implementations (either after boilerplate creation or after adding a resource), follow the workflow defined in the **mock** skill. Specifically:

### Step 1: Identify resources to mock

- **After boilerplate**: Generate mocks for all scaffolded resources (e.g., `User` with related `Order` data)
- **After adding a resource**: Generate mocks for the newly added resource

### Step 2: Create the mock directory structure

Create mocks following this pattern:

```
internal/repository/mock/
├── <resource>_repository.go          # Mock repository wrapper implementing real interface
└── <resource>/                       # Resource-specific mock directory
    ├── <resource>.go                 # Mock repository with embedded JSON
    ├── <resource>s.json              # Sample data for primary entity
    └── <related_entity>.json         # Sample data for related entities (if any)
```

### Step 3: Generate JSON data files

Create JSON files with realistic sample data using the entity-keyed format (matching `references/mock-template.md`):

```json
{
  "<resources>": [
    {
      "<resource_id>_id": 1,
      "name": "Alice Johnson",
      "email": "alice@example.com"
    },
    {
      "<resource_id>_id": 2,
      "name": "Bob Smith",
      "email": "bob@example.com"
    }
  ]
}
```

Name the file `data.json` (referenced by the `//go:embed data.json` directive in Step 4).

**Rules:**
- Include **at least 4-6 records** for meaningful test scenarios
- Use realistic, varied data (different names, emails, locations, etc.)
- Include edge cases if relevant (null values, empty strings, special characters)
- IDs should be sequential integers starting from 1
- For nullable fields, use JSON null or omit the field
- Decimal/monetary values should be strings to preserve precision (e.g., `"1200.50"`)
- Dates should use ISO 8601 format (e.g., `"2026-04-15T00:00:00Z"`)
- Foreign keys should reference valid IDs from the primary entity

### Step 4: Generate the mock repository Go file

Create `<resource>/<resource>.go` with:

```go
package <resource_lowercase>

import (
	"context"
	"database/sql"
	_ "embed"
	"encoding/json"
	"fmt"

	"<app-name>/internal/domain"
)

// Models — these mirror sqlc-generated types for the <resource> resource.
type <Resource> struct {
	<ResourceID> int32          `json:"<resource_id>_id"`
	Name         string         `json:"name"`
	Email        string         `json:"email"`
	Location     sql.NullString `json:"location"`
	// For monetary/precise values, add: "github.com/shopspring/decimal" import and use decimal.Decimal fields
}

//go:embed data.json
var dataJSON string

// Parsed mock data — loaded once at init time
type mockData struct {
	<Resources> []<Resource> `json:"<resources>"`
}

var data mockData

func init() {
	if err := json.Unmarshal([]byte(dataJSON), &data); err != nil {
		panic(fmt.Sprintf("mock: failed to unmarshal data.json: %v", err))
	}
}

// Mock repository implementation — satisfies repository.<Resource>Repository
type Mock<Resource>Repository struct{}

func New<Resource>Repository() repository.<Resource>Repository {
	return &Mock<Resource>Repository{}
}

func (m *Mock<Resource>Repository) Get<Resource>ByID(ctx context.Context, id int32) (*domain.Get<Resource>Response, error) {
	for _, item := range data.<Resources> {
		if item.<ResourceID> == id {
			return &domain.Get<Resource>Response{
				<ResourceID>: item.<ResourceID>,
				Name:        item.Name,
				Email:       item.Email,
				// map remaining fields; for sql.NullString: Location: func() string { if item.Location.Valid { return item.Location.String }; return "" }(),
			}, nil
		}
	}
	return nil, domain.ErrNotFound
}
```

**Key patterns:**
- Use `//go:embed` directives for JSON files
- Parse JSON **once in `init()`** — do NOT unmarshal on every method call
- Replicate sqlc-generated struct shapes exactly (including `sql.NullString`, `sql.NullTime`)
- For monetary fields, add `"github.com/shopspring/decimal"` import and use `decimal.Decimal` types
- Implement all repository interface methods
- Return `*domain.Get<Resource>Response` (pointer to domain response), NOT the raw mock struct
- **CRITICAL BUG TO AVOID:** Return the found item (`return &domain.Get<Resource>Response{...}, nil`), NOT empty struct (`return &domain.Get<Resource>Response{}, nil`)
- Return `nil, domain.ErrNotFound` for missing records (not `fmt.Errorf`) — controllers map this to HTTP 404
- Filter queries appropriately if the method accepts params (e.g., limit/offset/userID)
- Use receiver variable name `m` (conventional for mock implementations)

### Step 5: Generate the mock repository wrapper

Create `<resource>_repository.go` in the `mock/` root:

```go
package mock

import (
	"context"

	"<app-name>/internal/domain"
	"<app-name>/internal/repository"
	"<app-name>/internal/repository/mock/<resource>"
)

type <resource>Repository struct {
	query <resource>.Mock<Resource>Repository
}

func New<Resource>Repository() repository.<Resource>Repository {
	return &<resource>Repository{query: <resource>.Mock<Resource>Repository{}}
}

func (r *<resource>Repository) Get<Resource>ByID(ctx context.Context, id int32) (*domain.Get<Resource>Response, error) {
	item, err := r.query.Get<Resource>ByID(ctx, id)
	if err != nil {
		return nil, err
	}
	return &domain.Get<Resource>Response{
		<ResourceID>: item.<ResourceID>,
		Name:         item.Name,
		Email:        item.Email,
		// map remaining fields; handle sql.Null* conversions as needed
	}, nil
}
```

**Key patterns:**
- Wrapper implements the actual `repository.<Resource>Repository` interface
- Delegates to the embedded mock implementation
- Maps mock types → domain types (handling `sql.Null*`, type conversions)
- Response shapes match what the real repository would return

### Step 6: Verify integration

After generating mocks:

1. **Check interface compliance**:
   ```bash
   go build ./...
   ```

2. **Optionally generate a basic unit test** demonstrating mock usage:
   ```go
   func TestMockUserRepositoryGetUserByID(t *testing.T) {
       repo := mock.NewUserRepository()
       ctx := context.Background()

       user, err := repo.GetUserInfo(ctx, 1)
       if err != nil {
           t.Fatalf("unexpected error: %v", err)
       }
       if user.UserID != 1 {
           t.Errorf("expected user ID 1, got %d", user.UserID)
       }
   }
   ```

3. Run `gofmt -s -w .` to format

---

## Key Conventions (Always Follow)

### Clean Architecture Dependency Rule — Business Logic in Domain Layer Only

**Business logic MUST reside ONLY in the domain layer (`internal/domain/`) as methods on domain entities.**

The data flow is strictly:
```
Repository → Domain (business logic) → Usecase (orchestration) → Controller
```

**Never:**
```
Repository → Usecase (with business logic) → Controller  ← WRONG
```

**Rules:**
- **Domain layer** (`internal/domain/`): Contains ALL business logic — validations, calculations, transformations, business rules. Implemented as methods on domain entities (e.g., `order.CalculateFinalPrice()`, `user.IsValidEmail()`).
- **Usecase layer**: ONLY orchestrates repository calls and delegates to domain methods. It should NOT contain business logic. It coordinates data flow: fetch from repository → apply domain methods → build DTO response.
- **Repository layer**: ONLY maps database/sqlc types to domain types and handles data retrieval/persistence. No business logic.
- **Controller layer**: ONLY parses HTTP params, delegates to usecase, and returns JSON responses. No business logic.

**Example of correct separation:**
```go
// ✅ Domain layer — business logic lives here
func (o *Order) CalculateFinalPrice() decimal.Decimal {
    discountFactor := decimal.NewFromInt(100).Sub(o.DiscountPercent).Div(decimal.NewFromInt(100))
    return decimal.NewFromInt(int64(o.Quantity)).Mul(o.Price).Mul(discountFactor)
}

// ✅ Usecase layer — orchestration only
func (u *userUsecase) GetUserWithOrders(ctx context.Context, userID int32) (*dto.UserResponse, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    orders, err := u.userRepo.GetOrdersByUserID(ctx, userID)
    if err != nil {
        return nil, err
    }

    // Delegate to domain method — do NOT calculate here
    items := make([]dto.Order, len(orders.Orders))
    for i, order := range orders.Orders {
        finalPrice := order.CalculateFinalPrice()  // ← domain method
        items[i] = dto.Order{FinalPrice: finalPrice}
    }
    return &dto.UserResponse{Orders: items}, nil
}
```

### Dependency Direction
Interfaces live in inner layers (`usecase/`, `repository/`), implementations in outer layers. The flow is always: **Controller → Usecase → Repository → Database**.

### Timeouts
Set a context timeout **once** at the usecase entry point (e.g., 5s). Do not add additional timeouts in repository methods:
```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

### Error handling in controllers
- Invalid params → `400 Bad Request`
- `domain.ErrNotFound` → `404 Not Found` (define this sentinel error in the domain layer; the repository maps `sql.ErrNoRows` → `domain.ErrNotFound`)
- Any other error → `500 Internal Server Error`

### sqlc rules
- Never edit `models.go`, `db.go`, or `*.sql.go` — they're auto-generated
- Only edit `.sql` files and `schema.sql`, then run `sqlc generate`
- Use `overrides` in `sqlc.yaml` to map `decimal` → `github.com/shopspring/decimal.Decimal`

### Config pattern
Use `sync.Once` + `atomic.Pointer` for thread-safe lazy init in `config/config.go`.

### DB connection
Ping at startup with a 5s timeout (fail fast). Set pool tuning: `MaxOpenConns=25`, `MaxIdleConns=10`, `ConnMaxLifetime=5min`, `ConnMaxIdleTime=5min`.

### Security — Authentication & Authorization
- **Every endpoint should be behind authentication.** Never ship publicly accessible endpoints without auth.
- **Recommended strategy:** JWT-based authentication via a Gin middleware. Validate the token, extract the user identity, and attach it to the request context.
- **Middleware placement:** Insert auth middleware on the router group level in `registerRoutes`:
  ```go
  protected := router.Group("/api")
  protected.Use(AuthMiddleware())  // validates JWT, sets user in context
  {
      protected.GET("/users/:id", userController.GetUser)
  }
  ```
- **Authorization:** For resource-level access control (e.g., users can only edit their own data), add checks in the usecase layer after fetching the resource owner.
- **Public endpoints:** If certain endpoints must be public (e.g., `/api/auth/login`), register them on a separate group without the auth middleware.

---

## Reference Files

- **`references/full-boilerplate.md`** — Complete instructions for creating the entire project from scratch (all files, config, schema, seed data)
- **`references/resource-template.md`** — Template and patterns for adding a single new resource (domain, dto, controller, usecase, repository, sqlc)

Read the appropriate reference file when you need detailed code examples and file contents.
