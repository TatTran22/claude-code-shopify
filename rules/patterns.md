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

// Usage with Fiber
func respondSuccess(c fiber.Ctx, data interface{}) error {
    return c.JSON(APIResponse[interface{}]{
        Success: true,
        Data:    &data,
    })
}

func respondError(c fiber.Ctx, statusCode int, message string) error {
    return c.Status(statusCode).JSON(APIResponse[interface{}]{
        Success: false,
        Error:   message,
    })
}

// Or use fiber.Map for simpler cases
func handleExample(c fiber.Ctx) error {
    return c.JSON(fiber.Map{
        "success": true,
        "data":    someData,
    })
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

## Middleware Pattern (Fiber)

```go
// Middleware function signature: func(c fiber.Ctx) error
func LoggingMiddleware(c fiber.Ctx) error {
    start := time.Now()

    // Call next handler
    err := c.Next()

    // Log after request
    log.Printf("%s %s %v", c.Method(), c.Path(), time.Since(start))

    return err
}

// Auth middleware with configuration
func AuthMiddleware(jwtSecret string) fiber.Handler {
    return func(c fiber.Ctx) error {
        token := c.Get("Authorization")
        if token == "" {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "unauthorized",
            })
        }

        // Validate token...
        // Store user in context: c.Locals("user", claims)

        return c.Next()
    }
}

// Usage
app := fiber.New()
app.Use(LoggingMiddleware)
app.Use(AuthMiddleware(os.Getenv("JWT_SECRET")))
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
    h := handler.NewHandler(marketService, logger)

    // Setup Fiber app
    app := fiber.New(fiber.Config{
        Prefork: true,
    })

    // Setup routes
    api := app.Group("/api/v1")
    api.Get("/markets", h.HandleGetMarkets)
    api.Get("/markets/:id", h.HandleGetMarket)

    app.Listen(":8080")
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
   git clone https://github.com/user/go-fiber-example.git
   cd go-fiber-example
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
