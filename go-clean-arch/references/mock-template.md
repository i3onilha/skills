# Mock Repository Template Reference

This file provides templates for adding a **mock repository** implementation to an existing Clean Architecture Go project. Use this when the user says something like "Create a PRODUCT resource with mock data" or "Add a Category entity for testing without a database."

> **When to use mocks:** Use this mock template when you want to test or demo the application **without a real database**. The mock replaces the real `repository/<engine>/<resource>_repository.go` and satisfies the same `repository.<Resource>Repository` interface using in-memory JSON data.
>
> **Wiring into DI:** In `cmd/server/server.go`, replace the real repository constructor (e.g., `<engine>.New<Resource>Repository`) with the mock constructor (`mock.New<Resource>Repository`). No other layers change — the interface contract is identical.

> **⚠️ Module name substitution:** Replace `<app-name>` with your actual module name from `go.mod` in all import paths throughout this file. Copy-pasting these templates without substitution will produce uncompilable code.

---

## Step-by-step workflow

### 1. Write Mock Repository Implementation

Create `internal/repository/mock/<resource>/<resource>.mock.go` file (lowercase, singular).

This file is **self-contained** — it embeds mock data and implements all repository methods in-memory. No database or sqlc code generation is needed.

```go
package <resource>

import (
	"context"
	_ "embed"
	"encoding/json"
	"fmt"

	"<app-name>/internal/domain"
	"<app-name>/internal/repository"
)

// Models — these mirror sqlc-generated types for the <resource> resource.
type Get<Resource>sBy<Condition>Params struct {
	<Condition>ID int32 `json:"<condition>_id"`
	Limit         int32 `json:"limit"`
	Offset        int32 `json:"offset"`
}

type Get<Resource>sBy<Condition>Row struct {
	<ResourceID>  int32  `json:"<resource_id>_id"`
	<Condition>ID int32  `json:"<condition>_id"`
	// ... additional fields based on your resource
}

type <Resource> struct {
	<ResourceID> int32  `json:"<resource_id>_id"`
	Name         string `json:"name"`
	Email        string `json:"email"`
	// ... additional fields; for nullable columns use sql.NullString:
	// Location sql.NullString `json:"location"`
}

//go:embed data.json
var dataJSON string

// Mock data set
type Data struct {
	<Resources>  []<Resource>                 `json:"<resources>"`
	<Related> []Get<Resource>sBy<Condition>Row `json:"<related>"`
}

var data Data

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

func (r *Mock<Resource>Repository) Get<Resource>ByID(ctx context.Context, id int32) (*domain.Get<Resource>Response, error) {
	for _, item := range data.<Resources> {
		if item.<ResourceID> == id {
			return &domain.Get<Resource>Response{
				<ResourceID>: item.<ResourceID>,
				Name:        item.Name,
				Email:       item.Email,
				// map remaining fields from mock type to domain type
			}, nil
		}
	}
	return nil, domain.ErrNotFound
}

func (r *Mock<Resource>Repository) Get<Resource>sBy<Condition>(ctx context.Context, arg Get<Resource>sBy<Condition>Params) (*domain.Get<Resource>sResponse, error) {
	rows := []Get<Resource>sBy<Condition>Row{}
	for _, row := range data.<Related> {
		if row.<Condition>ID == arg.<Condition>ID {
			rows = append(rows, row)
		}
	}

	// Apply pagination
	total := int32(len(rows))
	start := arg.Offset
	if start > total {
		start = total
	}
	end := start + arg.Limit
	if end > total {
		end = total
	}
	rows = rows[start:end]

	items := make([]*domain.<Resource>, 0, len(rows))
	for _, row := range rows {
		items = append(items, &domain.<Resource>{
			<ResourceID>: row.<ResourceID>,
			// map each row to domain type
			// For sql.NullString fields, check .Valid before accessing .String:
			// Field: func() string { if row.Field.Valid { return row.Field.String }; return "" }(),
		})
	}
	return &domain.Get<Resource>sResponse{Items: items}, nil
}

```

Create `internal/repository/mock/<resource>/data.json` file (lowercase, singular).

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
    },
    {
      "<resource_id>_id": 3,
      "name": "Charlie Brown",
      "email": "charlie@example.com"
    },
    {
      "<resource_id>_id": 4,
      "name": "Diana Prince",
      "email": "diana@example.com"
    }
  ],
  "<related>": [
    {
      "<resource_id>_id": 1,
      "<condition>_id": 1
    },
    {
      "<resource_id>_id": 2,
      "<condition>_id": 1
    },
    {
      "<resource_id>_id": 3,
      "<condition>_id": 2
    },
    {
      "<resource_id>_id": 4,
      "<condition>_id": 3
    }
  ]
}

```

> **Only one file needed:** Unlike the real sqlc-based repository (which requires `.sql` files, `sqlc.yaml` updates, and `sqlc generate`), the mock is a single `.mock.go` file + a `data.json` data file. No code generation step is required.

### 2. Domain layer

`internal/domain/<resource>_domain.go`:

```go
package domain

import (
	"errors"

	"github.com/shopspring/decimal"
)

// Sentinel errors for clean architecture separation.
// The repository maps "not found" → ErrNotFound so controllers
// never depend on database-specific errors.
var ErrNotFound = errors.New("not found")

// Core business entity — decoupled from API shapes and DB models
type <Resource> struct {
	<ResourceID> int32 `json:"<resource_id>_id"`
	Name         string `json:"name"`
	Email        string `json:"email"`
	// ... additional fields
}

type Get<Resource>Response struct {
	<ResourceID> int32  `json:"<resource_id>_id"`
	Name         string `json:"name"`
	Email        string `json:"email"`
	// ... fields matching what Get<Resource>ByID returns
}

type Get<Resource>sResponse struct {
	Items []*<Resource> `json:"items,omitempty"`
}

```

Key rules:
- Define `var ErrNotFound = errors.New("not found")` — the repository returns this when a record is missing, and the controller maps it to HTTP 404
- **ALL business logic goes here** — validations, calculations, transformations, business rules
- Implement as methods on domain entities (e.g., `order.CalculateFinalPrice()`, `user.CanAccessResource()`)
- Usecases should NEVER contain business logic — they only orchestrate and delegate to these domain methods
- Use `omitempty` on JSON tags for optional fields
- Response structs mirror what the repository returns after mapping from mock/sqlc models
- Use Go convention `ID` suffix (e.g., `UserID`, `OrderID`), not `Id`

---

## Example: The USER resource (with orders + discounts)

The existing project demonstrates this pattern with a USER resource that:
- Fetches user profile info (`GetUserByID :one`)
- Fetches user orders with discount data via LEFT JOIN with aggregation (`GetOrdersByUserID :many`)
- Calculates `final_price = quantity * price * (1 - discount_percent/100)` in the domain layer
- Returns a combined response with user info + order items with computed prices
- Supports pagination via `LIMIT ? OFFSET ?` parameters

Use this as a reference when creating resources that involve joins and calculated fields.
