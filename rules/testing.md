# Testing Requirements (Go)

## Minimum Test Coverage: 80%

**Command to check coverage:**
```bash
go test ./... -cover
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out  # View in browser
```

---

## Test Types (ALL required)

1. **Unit Tests** - Individual functions, methods, utilities
   - Test file: `file_test.go` (same package)
   - Run: `go test ./...`

2. **Integration Tests** - API endpoints, database operations, external services
   - Use test containers for PostgreSQL, Redis, RabbitMQ
   - Run: `go test -tags=integration ./...`

3. **E2E Tests** - Critical user flows (Frontend: Playwright)
   - Test Shopify embedded app flows
   - Run: `cd frontend && npm run test:e2e`

---

## Test-Driven Development (TDD)

MANDATORY workflow for new features:

1. **Write test first (RED)**
   ```bash
   # Create *_test.go file
   # Write failing test
   go test ./...  # Should FAIL
   ```

2. **Run test - it should FAIL**
   - Verify test actually tests the feature
   - Check test output for correct failure

3. **Write minimal implementation (GREEN)**
   - Write just enough code to pass
   - Don't over-engineer

4. **Run test - it should PASS**
   ```bash
   go test ./...  # Should PASS
   ```

5. **Refactor (IMPROVE)**
   - Clean up code
   - Extract functions
   - Improve naming

6. **Verify coverage (80%+)**
   ```bash
   go test ./... -cover
   # Check that coverage is >= 80%
   ```

---

## Table-Driven Tests (REQUIRED for Go)

ALWAYS use table-driven tests:

```go
func TestGetMarket(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    *Market
        wantErr bool
    }{
        {
            name:  "valid market ID",
            input: "market-123",
            want:  &Market{ID: "market-123", Name: "Election 2024"},
            wantErr: false,
        },
        {
            name:    "empty ID",
            input:   "",
            want:    nil,
            wantErr: true,
        },
        {
            name:    "not found",
            input:   "invalid-id",
            want:    nil,
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := GetMarket(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("GetMarket() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if !reflect.DeepEqual(got, tt.want) {
                t.Errorf("GetMarket() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

---

## Testing Best Practices

### Use testify for assertions (optional but recommended)

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestCreateMarket(t *testing.T) {
    market, err := CreateMarket("Election 2024")

    require.NoError(t, err)  // Fail immediately if error
    assert.NotNil(t, market)
    assert.Equal(t, "Election 2024", market.Name)
}
```

### Test HTTP Handlers with httptest

```go
import (
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestHandleGetMarket(t *testing.T) {
    req := httptest.NewRequest("GET", "/api/markets/123", nil)
    w := httptest.NewRecorder()

    handler := NewMarketHandler(mockService)
    handler.HandleGetMarket(w, req)

    assert.Equal(t, http.StatusOK, w.Code)
}
```

### Use Test Containers for Integration Tests

```go
import (
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func setupTestDB(t *testing.T) *pgxpool.Pool {
    ctx := context.Background()

    container, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15"),
    )
    require.NoError(t, err)

    t.Cleanup(func() {
        container.Terminate(ctx)
    })

    connStr, err := container.ConnectionString(ctx)
    require.NoError(t, err)

    pool, err := pgxpool.New(ctx, connStr)
    require.NoError(t, err)

    return pool
}

func TestMarketRepository_Create(t *testing.T) {
    pool := setupTestDB(t)
    defer pool.Close()

    repo := NewMarketRepository(pool)

    market := &Market{Name: "Test Market"}
    err := repo.Create(context.Background(), market)

    assert.NoError(t, err)
    assert.NotEmpty(t, market.ID)
}
```

---

## Mocking

### Use interfaces for mocking

```go
// Define interface
type MarketRepository interface {
    FindByID(ctx context.Context, id string) (*Market, error)
}

// Mock implementation
type MockMarketRepository struct {
    FindByIDFunc func(ctx context.Context, id string) (*Market, error)
}

func (m *MockMarketRepository) FindByID(ctx context.Context, id string) (*Market, error) {
    if m.FindByIDFunc != nil {
        return m.FindByIDFunc(ctx, id)
    }
    return nil, nil
}

// Use in test
func TestServiceGetMarket(t *testing.T) {
    mockRepo := &MockMarketRepository{
        FindByIDFunc: func(ctx context.Context, id string) (*Market, error) {
            return &Market{ID: id, Name: "Test"}, nil
        },
    }

    service := NewMarketService(mockRepo)
    market, err := service.GetMarket(context.Background(), "123")

    assert.NoError(t, err)
    assert.Equal(t, "123", market.ID)
}
```

### Use testify/mock for complex mocks

```go
import "github.com/stretchr/testify/mock"

type MockRepository struct {
    mock.Mock
}

func (m *MockRepository) FindByID(ctx context.Context, id string) (*Market, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*Market), args.Error(1)
}

// In test
func TestService(t *testing.T) {
    mockRepo := new(MockRepository)
    mockRepo.On("FindByID", mock.Anything, "123").Return(&Market{ID: "123"}, nil)

    service := NewService(mockRepo)
    // Test service...

    mockRepo.AssertExpectations(t)
}
```

---

## Benchmark Tests

```go
func BenchmarkGetMarket(b *testing.B) {
    // Setup
    repo := setupRepo()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        repo.GetMarket(context.Background(), "123")
    }
}

// Run benchmarks:
// go test -bench=. -benchmem
```

---

## Troubleshooting Test Failures

1. **Use tdd-guide agent** for TDD workflow guidance
2. **Check test isolation** - Each test should be independent
3. **Verify mocks are correct** - Ensure mocks return expected values
4. **Check error messages** - Go test output shows exact failure location
5. **Run with -v flag** - `go test -v ./...` for verbose output
6. **Fix implementation, not tests** (unless tests are genuinely wrong)

---

## Test Commands

```bash
# Run all tests
go test ./...

# Run with coverage
go test ./... -cover

# Run specific package
go test ./internal/service

# Run specific test
go test -run TestGetMarket ./internal/service

# Run with verbose output
go test -v ./...

# Run integration tests only
go test -tags=integration ./...

# Run benchmarks
go test -bench=. -benchmem

# Generate coverage report
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out
```

---

## Agent Support

- **tdd-guide** - Use PROACTIVELY for new features, enforces write-tests-first (Go testing)
- **e2e-runner** - Playwright E2E testing specialist (for frontend)

---

## Web Component Testing (Shopify Polaris)

When testing Shopify Polaris Web Components (`s-*` prefix):

```typescript
// Use tag + data-testid selectors
page.locator('s-text-field[data-testid="email"]')
page.locator('s-button[variant="primary"]')

// Get attribute values
const error = await page.locator('s-text-field').getAttribute('error')
```

See `e2e-runner` agent for comprehensive Web Component testing patterns.

---

## Coverage Requirements

**80% minimum coverage** for:
- All packages in `internal/`
- All business logic
- All HTTP handlers
- All repository methods

**Exceptions** (can have lower coverage):
- `cmd/` packages (main functions)
- Simple getters/setters
- Auto-generated code

---

## Quick Reference

### Test File Structure
```
market.go       # Implementation
market_test.go  # Tests (same package)
```

### Test Function Naming
```go
func TestFunctionName(t *testing.T) {}          // Unit test
func TestFunctionName_Scenario(t *testing.T) {} // Specific scenario
func BenchmarkFunctionName(b *testing.B) {}     // Benchmark
```

### Running Tests
```bash
go test ./...              # All tests
go test -cover ./...       # With coverage
go test -v ./...           # Verbose
go test -run TestName ./... # Specific test
```

---

**Remember**: Tests are documentation. Write clear test names that describe what is being tested.
