# Common Patterns (Go)

## API Response Format

```go
// Generic API response structure
type APIResponse[T any] struct {
    Success bool   `json:"success"`
    Data    *T     `json:"data,omitempty"`
    Error   string `json:"error,omitempty"`
    Meta    *Meta  `json:"meta,omitempty"`
}

type Meta struct {
    Total  int `json:"total"`
    Page   int `json:"page"`
    Limit  int `json:"limit"`
}

// Usage
func respondSuccess(w http.ResponseWriter, data interface{}) {
    response := APIResponse[interface{}]{
        Success: true,
        Data:    &data,
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(response)
}

func respondError(w http.ResponseWriter, statusCode int, message string) {
    response := APIResponse[interface{}]{
        Success: false,
        Error:   message,
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(response)
}
```

---

## Repository Pattern

```go
// Repository interface for data access
type MarketRepository interface {
    FindAll(ctx context.Context, filters MarketFilters) ([]Market, error)
    FindByID(ctx context.Context, id string) (*Market, error)
    Create(ctx context.Context, market *Market) error
    Update(ctx context.Context, id string, market *Market) error
    Delete(ctx context.Context, id string) error
}

// Implementation with PostgreSQL
type PostgresMarketRepository struct {
    db *pgxpool.Pool
}

func NewPostgresMarketRepository(db *pgxpool.Pool) *PostgresMarketRepository {
    return &PostgresMarketRepository{db: db}
}

func (r *PostgresMarketRepository) FindByID(ctx context.Context, id string) (*Market, error) {
    query := `SELECT id, name, status FROM markets WHERE id = $1`

    var market Market
    err := r.db.QueryRow(ctx, query, id).Scan(&market.ID, &market.Name, &market.Status)
    if err == pgx.ErrNoRows {
        return nil, fmt.Errorf("market not found: %w", err)
    }
    if err != nil {
        return nil, fmt.Errorf("failed to query market: %w", err)
    }

    return &market, nil
}
```

---

## Service Layer Pattern

```go
// Service for business logic
type MarketService struct {
    repo MarketRepository
}

func NewMarketService(repo MarketRepository) *MarketService {
    return &MarketService{repo: repo}
}

func (s *MarketService) GetMarket(ctx context.Context, id string) (*Market, error) {
    market, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get market: %w", err)
    }

    return market, nil
}

func (s *MarketService) CreateMarket(ctx context.Context, market *Market) error {
    // Business logic validation
    if market.Name == "" {
        return errors.New("market name is required")
    }

    if err := s.repo.Create(ctx, market); err != nil {
        return fmt.Errorf("failed to create market: %w", err)
    }

    return nil
}
```

---

## Middleware Pattern (Chi)

```go
// Middleware function signature: func(http.Handler) http.Handler
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Call next handler
        next.ServeHTTP(w, r)

        // Log after request
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// Auth middleware
func AuthMiddleware(jwtSecret string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := r.Header.Get("Authorization")
            if token == "" {
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return
            }

            // Validate token...
            next.ServeHTTP(w, r)
        })
    }
}
```

---

## Context Pattern

```go
// Always pass context as first parameter
func FetchData(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}

// Check for context cancellation
func LongRunningTask(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // Continue processing
        }
    }
}
```

---

## Error Wrapping Pattern

```go
// Always wrap errors with context
func ProcessOrder(ctx context.Context, orderID string) error {
    order, err := fetchOrder(ctx, orderID)
    if err != nil {
        return fmt.Errorf("failed to fetch order %s: %w", orderID, err)
    }

    if err := validateOrder(order); err != nil {
        return fmt.Errorf("order %s validation failed: %w", orderID, err)
    }

    if err := saveOrder(ctx, order); err != nil {
        return fmt.Errorf("failed to save order %s: %w", orderID, err)
    }

    return nil
}

// Custom error types
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s - %s", e.Field, e.Message)
}
```

---

## Concurrency Pattern

```go
// Worker pool pattern
func ProcessItems(ctx context.Context, items []Item) error {
    var wg sync.WaitGroup
    errChan := make(chan error, len(items))

    // Limit concurrent workers
    sem := make(chan struct{}, 10)

    for _, item := range items {
        wg.Add(1)
        go func(i Item) {
            defer wg.Done()

            sem <- struct{}{}        // Acquire
            defer func() { <-sem }() // Release

            if err := processItem(ctx, i); err != nil {
                errChan <- err
            }
        }(item)
    }

    wg.Wait()
    close(errChan)

    // Check for errors
    for err := range errChan {
        if err != nil {
            return err
        }
    }

    return nil
}
```

---

## Dependency Injection Pattern

```go
// Define dependencies as interfaces
type Handler struct {
    marketService  MarketService
    orderService   OrderService
    logger         Logger
}

func NewHandler(
    marketService MarketService,
    orderService OrderService,
    logger Logger,
) *Handler {
    return &Handler{
        marketService: marketService,
        orderService:  orderService,
        logger:        logger,
    }
}

// Wire up in main.go
func main() {
    // Setup dependencies
    db := setupDatabase()
    cache := setupRedis()
    logger := setupLogger()

    // Create repositories
    marketRepo := repository.NewMarketRepository(db)

    // Create services
    marketService := service.NewMarketService(marketRepo, cache)

    // Create handlers
    handler := handler.NewHandler(marketService, logger)

    // Setup routes
    r := chi.NewRouter()
    r.Get("/markets", handler.HandleGetMarkets)
}
```

---

## Skeleton Projects

When implementing new functionality:

1. **Search for battle-tested skeleton projects**
   - Look for Go projects with similar architecture
   - Check GitHub stars, last update, test coverage
   - Verify it follows Go conventions

2. **Use parallel agents to evaluate options:**
   - Security assessment (dependencies, vulnerabilities)
   - Extensibility analysis (clean architecture, interfaces)
   - Relevance scoring (matches your tech stack)
   - Implementation planning (migration path)

3. **Clone best match as foundation**
   ```bash
   git clone https://github.com/user/go-chi-example.git
   cd go-chi-example
   # Review structure, adapt to your needs
   ```

4. **Iterate within proven structure**
   - Keep project organization
   - Replace specific business logic
   - Maintain testing patterns
   - Follow established conventions

---

## Configuration Pattern

```go
// Config struct
type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Redis    RedisConfig
}

type ServerConfig struct {
    Port int
    Host string
}

type DatabaseConfig struct {
    Host     string
    Port     int
    User     string
    Password string
    DBName   string
}

// Load from environment
func LoadConfig() (*Config, error) {
    return &Config{
        Server: ServerConfig{
            Port: getEnvInt("PORT", 8080),
            Host: getEnv("HOST", "0.0.0.0"),
        },
        Database: DatabaseConfig{
            Host:     getEnv("DB_HOST", "localhost"),
            Port:     getEnvInt("DB_PORT", 5432),
            User:     getEnv("DB_USER", "postgres"),
            Password: getEnv("DB_PASSWORD", ""),
            DBName:   getEnv("DB_NAME", "myapp"),
        },
    }, nil
}

func getEnv(key, defaultVal string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return defaultVal
}

func getEnvInt(key string, defaultVal int) int {
    if val := os.Getenv(key); val != "" {
        if i, err := strconv.Atoi(val); err == nil {
            return i
        }
    }
    return defaultVal
}
```

---

**Remember**: Go has established patterns. Don't reinvent the wheel—follow community conventions.
