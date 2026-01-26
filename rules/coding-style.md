# Coding Style (Go)

## gofmt is Law (CRITICAL)

ALWAYS run `gofmt` and `goimports` before committing:

```bash
gofmt -w .
goimports -w .
```

**No exceptions**. Formatting is automatic, not a choice.

---

## Error Handling (CRITICAL)

ALWAYS check errors. NEVER use `_` to ignore:

```go
// WRONG: Ignoring errors
data, _ := json.Marshal(obj)  // NEVER DO THIS

// CORRECT: Always check
data, err := json.Marshal(obj)
if err != nil {
    return fmt.Errorf("failed to marshal: %w", err)
}
```

**ALWAYS wrap errors with context using `%w`:**

```go
// WRONG: No context
if err != nil {
    return err
}

// CORRECT: Wrap with context
if err != nil {
    return fmt.Errorf("failed to fetch market %s: %w", id, err)
}
```

---

## Naming Conventions (CRITICAL)

Use **MixedCaps**, NOT snake_case:

```go
// CORRECT
var marketCount int
var isAuthenticated bool
var userID string           // ID, not Id
const MaxRetries = 3        // NOT MAX_RETRIES

func GetMarket(id string) (*Market, error) {}

// WRONG
var market_count int        // snake_case
const MAX_RETRIES = 3       // SCREAMING_SNAKE_CASE
type IReader interface {}   // Interface prefix
```

---

## Project Structure

```
project-root/
├── cmd/
│   ├── api/              # API server entry point
│   │   └── main.go
│   └── worker/           # Worker entry point
│       └── main.go
├── internal/
│   ├── domain/           # Business entities
│   ├── repository/       # Data access
│   ├── service/          # Business logic
│   ├── handler/          # HTTP handlers
│   └── middleware/       # Chi middleware
├── migrations/           # Database migrations
├── go.mod
└── go.sum
```

---

## File Organization

MANY SMALL FILES > FEW LARGE FILES:
- High cohesion, low coupling
- 200-400 lines typical, 800 max
- Extract logic from large functions
- Organize by feature/domain, not by type

---

## Function Guidelines

Keep functions **small and focused** (<30 lines):

```go
// CORRECT: Small, focused
func (s *Service) CreateMarket(ctx context.Context, name string) error {
    if err := s.validate(name); err != nil {
        return err
    }

    market := &Market{ID: uuid.NewString(), Name: name}

    if err := s.repo.Create(ctx, market); err != nil {
        return fmt.Errorf("failed to create market: %w", err)
    }

    return nil
}

// WRONG: Too long
func CreateMarket() {
    // 100+ lines of mixed logic
}
```

**Always use `context.Context` as first parameter:**

```go
// CORRECT
func FetchData(ctx context.Context, url string) error {}

// WRONG
func FetchData(url string) error {}  // No context
```

---

## Pointer vs Value Receivers

**Use pointer receivers if:**
1. Method modifies the receiver
2. Receiver is large (avoid copying)
3. All other methods use pointer receivers (consistency)

```go
// CORRECT: Pointer receiver for modification
func (m *Market) Save() error {
    m.UpdatedAt = time.Now()
    return nil
}

// CORRECT: Value receiver for read-only
func (m Market) IsActive() bool {
    return m.Status == "active"
}

// WRONG: Mixed receivers (inconsistent)
func (m *Market) Save() error {}   // pointer
func (m Market) Delete() error {}  // value - INCONSISTENT
```

---

## Interface Design

**Keep interfaces small** (1-2 methods):

```go
// CORRECT: Small interface
type Reader interface {
    Read(p []byte) (n int, err error)
}

// WRONG: Large interface
type Repository interface {
    FindAll() ([]Market, error)
    FindByID(id string) (*Market, error)
    Create(m *Market) error
    Update(m *Market) error
    Delete(id string) error
    // Too many methods
}
```

**Accept interfaces, return structs:**

```go
// CORRECT
func ProcessData(r io.Reader) error {}
func NewRepo(db *pgxpool.Pool) *Repo {}

// WRONG
func NewRepo(db *pgxpool.Pool) RepoInterface {}
```

---

## Defer for Cleanup

ALWAYS use defer for cleanup:

```go
// CORRECT
file, err := os.Open("file.txt")
if err != nil {
    return err
}
defer file.Close()

// Database rows
rows, err := db.Query(ctx, query)
if err != nil {
    return err
}
defer rows.Close()

// Mutex unlock
mu.Lock()
defer mu.Unlock()
```

---

## Code Quality Checklist

Before marking work complete:
- [ ] `gofmt` and `goimports` run (CRITICAL)
- [ ] All errors checked (no `_` for errors)
- [ ] Errors wrapped with context (`%w`)
- [ ] Functions are small (<30 lines)
- [ ] Files are focused (<800 lines)
- [ ] No deep nesting (>4 levels) - use early returns
- [ ] MixedCaps naming (no snake_case)
- [ ] Constants use MixedCaps (no SCREAMING_SNAKE_CASE)
- [ ] Context passed to long-running operations
- [ ] Defer used for cleanup
- [ ] No `fmt.Println` in production code (use structured logging)
- [ ] Exported functions have godoc comments
- [ ] Pointer receivers used consistently

---

## Quick Reference

### Error Handling
```go
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}
```

### Naming
- Package: `user` (lowercase, no underscores)
- Variable: `marketCount` (MixedCaps)
- Constant: `MaxRetries` (MixedCaps, NOT SCREAMING)
- Function: `GetMarket` (exported), `parseQuery` (unexported)

### Formatting
```bash
gofmt -w .
goimports -w .
```

---

**Remember**: Go conventions are not suggestions—they're how Go code is written. Follow them.
