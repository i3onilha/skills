---
name: go-clean-arch
description: >
  Scaffold a Go REST API project following Clean Architecture with sqlc, Gin, Uber FX (DI),
  Viper config, Docker, and Air hot-reload. Use this skill whenever the user wants to create
  a new Go Clean Architecture project from scratch, bootstrap a go-clean-arch-style repo,
  or add a new domain resource (e.g. "Create a USER resource", "add a Product entity",
  "scaffold an Order service", "set up a Go API with sqlc and clean layers").
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

When the user asks to create the entire project from scratch, follow the detailed prompt in `references/full-boilerplate.md`. This will scaffold:

- Go module (`go.mod`), `cmd/server/server.go` with Uber FX DI
- All layer directories: `internal/controller/`, `internal/usecase/`, `internal/domain/`, `internal/dto/`, `internal/repository/`
- MySQL connection pool (`internal/repository/mysql/conn.go`)
- Database schema with seed data (`schema.sql`)
- sqlc config (`sqlc.yaml`) and initial `.sql` query file
- Config loading via Viper (`config/config.go`)
- Docker Compose, Dockerfile, `.env.example`, `.air.toml`, `.gitignore`
- A working `USER` resource as the example (with orders + discounts)

After creating all files:
1. Run `go mod tidy`
2. Run `sqlc generate`
3. Verify the build: `go build ./cmd/server/server.go`

**Then ask the user**: "The boilerplate is ready. Would you like me to add any additional resources on top of it?"

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
3. Build: `go build ./cmd/server/server.go`

---

## Key Conventions (Always Follow)

### Dependency Rule
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
