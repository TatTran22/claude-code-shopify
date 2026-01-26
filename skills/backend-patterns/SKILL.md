---
skill_id: backend-patterns
name: Go Backend Patterns with Chi Router
description: Backend architecture patterns for Go with Chi router - API design, repository pattern, PostgreSQL/Redis integration, middleware, context propagation, and error handling
category: backend
tags: [go, chi, postgresql, redis, api, middleware, repository]
applies_to: [go]
auto_trigger: ["chi", "router", "api", "middleware", "repository", "pgx", "sqlc"]
---

# Go Backend Patterns with Chi Router

Production-ready backend patterns for Go with Chi router, PostgreSQL (pgx/sqlc), Redis caching, structured logging, and error handling.

## Core Architecture

```
HTTP Request
     │
     ▼
Chi Router (Middleware Chain)
     │
     ├─> Logging Middleware
     ├─> Auth Middleware
     ├─> Rate Limit Middleware
     │
     ▼
HTTP Handler
     │
     ├─> Validator (validate request)
     ├─> Service (business logic)
     │     │
     │     ├─> Repository (data access)
     │     │     ├─> PostgreSQL (pgx/sqlc)
     │     │     └─> Redis (caching)
     │     │
     │     └─> External APIs
     │
     └─> Response (JSON)
```

---

## 1. Chi Router Patterns

### 1.1 Basic Router Setup

```go
// cmd/api/main.go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
    "github.com/go-chi/cors"
)

func main() {
    r := chi.NewRouter()

    // Core middleware (order matters!)
    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(middleware.Timeout(60 * time.Second))

    // CORS middleware
    r.Use(cors.Handler(cors.Options{
        AllowedOrigins:   []string{"https://*", "http://*"},
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"},
        AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
        ExposedHeaders:   []string{"Link"},
        AllowCredentials: true,
        MaxAge:           300,
    }))

    // Routes
    r.Get("/health", handleHealth)
    r.Route("/api/v1", func(r chi.Router) {
        // Public routes
        r.Post("/auth/login", handleLogin)
        r.Post("/auth/register", handleRegister)

        // Protected routes
        r.Group(func(r chi.Router) {
            r.Use(authMiddleware)

            r.Route("/markets", func(r chi.Router) {
                r.Get("/", handleListMarkets)
                r.Post("/", handleCreateMarket)
                r.Get("/{id}", handleGetMarket)
                r.Put("/{id}", handleUpdateMarket)
                r.Delete("/{id}", handleDeleteMarket)
            })

            r.Route("/orders", func(r chi.Router) {
                r.Get("/", handleListOrders)
                r.Post("/", handleCreateOrder)
            })
        })
    })

    // Start server
    srv := &http.Server{
        Addr:         ":8080",
        Handler:      r,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Graceful shutdown
    go func() {
        log.Printf("Server starting on %s", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server failed: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Server forced to shutdown: %v", err)
    }

    log.Println("Server exited")
}

func handleHealth(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(`{"status":"healthy"}`))
}
```

### 1.2 Chi Middleware Patterns

```go
// internal/middleware/auth.go
package middleware

import (
    "context"
    "net/http"
    "strings"

    "github.com/golang-jwt/jwt/v5"
)

type contextKey string

const UserContextKey contextKey = "user"

type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func Auth(jwtSecret string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract token from Authorization header
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                respondError(w, http.StatusUnauthorized, "missing authorization header")
                return
            }

            tokenString := strings.TrimPrefix(authHeader, "Bearer ")

            // Parse and validate JWT
            token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
                if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                    return nil, fmt.Errorf("unexpected signing method")
                }
                return []byte(jwtSecret), nil
            })

            if err != nil || !token.Valid {
                respondError(w, http.StatusUnauthorized, "invalid token")
                return
            }

            claims, ok := token.Claims.(*Claims)
            if !ok {
                respondError(w, http.StatusUnauthorized, "invalid token claims")
                return
            }

            // Add user to context
            ctx := context.WithValue(r.Context(), UserContextKey, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// RequireRole checks if user has required role
func RequireRole(role string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims, ok := r.Context().Value(UserContextKey).(*Claims)
            if !ok {
                respondError(w, http.StatusUnauthorized, "unauthorized")
                return
            }

            if claims.Role != role && claims.Role != "admin" {
                respondError(w, http.StatusForbidden, "insufficient permissions")
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

func respondError(w http.ResponseWriter, code int, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(map[string]string{"error": message})
}
```

### 1.3 Rate Limiting Middleware

```go
// internal/middleware/ratelimit.go
package middleware

import (
    "context"
    "net/http"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

type RateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(rps int, burst int) *RateLimiter {
    return &RateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     rate.Limit(rps),
        burst:    burst,
    }
}

func (rl *RateLimiter) getLimiter(identifier string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    limiter, exists := rl.limiters[identifier]
    if !exists {
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.limiters[identifier] = limiter
    }

    return limiter
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Use IP address as identifier
        ip := r.RemoteAddr

        limiter := rl.getLimiter(ip)

        if !limiter.Allow() {
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusTooManyRequests)
            json.NewEncoder(w).Encode(map[string]string{
                "error": "rate limit exceeded",
            })
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

### 1.4 Structured Logging Middleware

```go
// internal/middleware/logger.go
package middleware

import (
    "log/slog"
    "net/http"
    "time"

    "github.com/go-chi/chi/v5/middleware"
)

func StructuredLogger(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ww := middleware.NewWrapResponseWriter(w, r.ProtoMajor)

            start := time.Now()

            defer func() {
                logger.Info("http request",
                    slog.String("method", r.Method),
                    slog.String("path", r.URL.Path),
                    slog.Int("status", ww.Status()),
                    slog.Int("bytes", ww.BytesWritten()),
                    slog.Duration("duration", time.Since(start)),
                    slog.String("request_id", middleware.GetReqID(r.Context())),
                )
            }()

            next.ServeHTTP(ww, r)
        })
    }
}
```

---

## 2. Repository Pattern with PostgreSQL

### 2.1 Repository Interface

```go
// internal/domain/market.go
package domain

import (
    "context"
    "time"
)

type Market struct {
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Status      string    `json:"status"`
    Volume      float64   `json:"volume"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}

type MarketFilters struct {
    Status *string
    Limit  int
    Offset int
}

// MarketRepository defines the interface for market data access
type MarketRepository interface {
    FindAll(ctx context.Context, filters MarketFilters) ([]Market, error)
    FindByID(ctx context.Context, id string) (*Market, error)
    Create(ctx context.Context, market *Market) error
    Update(ctx context.Context, id string, market *Market) error
    Delete(ctx context.Context, id string) error
}
```

### 2.2 PostgreSQL Repository with pgx

```go
// internal/repository/market_postgres.go
package repository

import (
    "context"
    "fmt"

    "github.com/google/uuid"
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"

    "yourapp/internal/domain"
)

type PostgresMarketRepository struct {
    db *pgxpool.Pool
}

func NewPostgresMarketRepository(db *pgxpool.Pool) *PostgresMarketRepository {
    return &PostgresMarketRepository{db: db}
}

func (r *PostgresMarketRepository) FindAll(ctx context.Context, filters domain.MarketFilters) ([]domain.Market, error) {
    query := `
        SELECT id, name, description, status, volume, created_at, updated_at
        FROM markets
        WHERE ($1::text IS NULL OR status = $1)
        ORDER BY created_at DESC
        LIMIT $2 OFFSET $3
    `

    var status *string
    if filters.Status != nil {
        status = filters.Status
    }

    rows, err := r.db.Query(ctx, query, status, filters.Limit, filters.Offset)
    if err != nil {
        return nil, fmt.Errorf("query failed: %w", err)
    }
    defer rows.Close()

    var markets []domain.Market
    for rows.Next() {
        var m domain.Market
        if err := rows.Scan(
            &m.ID,
            &m.Name,
            &m.Description,
            &m.Status,
            &m.Volume,
            &m.CreatedAt,
            &m.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan failed: %w", err)
        }
        markets = append(markets, m)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("rows error: %w", err)
    }

    return markets, nil
}

func (r *PostgresMarketRepository) FindByID(ctx context.Context, id string) (*domain.Market, error) {
    query := `
        SELECT id, name, description, status, volume, created_at, updated_at
        FROM markets
        WHERE id = $1
    `

    var m domain.Market
    err := r.db.QueryRow(ctx, query, id).Scan(
        &m.ID,
        &m.Name,
        &m.Description,
        &m.Status,
        &m.Volume,
        &m.CreatedAt,
        &m.UpdatedAt,
    )

    if err == pgx.ErrNoRows {
        return nil, fmt.Errorf("market not found")
    }

    if err != nil {
        return nil, fmt.Errorf("query failed: %w", err)
    }

    return &m, nil
}

func (r *PostgresMarketRepository) Create(ctx context.Context, market *domain.Market) error {
    query := `
        INSERT INTO markets (id, name, description, status, volume, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
    `

    market.ID = uuid.NewString()
    market.CreatedAt = time.Now()
    market.UpdatedAt = time.Now()

    _, err := r.db.Exec(ctx, query,
        market.ID,
        market.Name,
        market.Description,
        market.Status,
        market.Volume,
        market.CreatedAt,
        market.UpdatedAt,
    )

    if err != nil {
        return fmt.Errorf("insert failed: %w", err)
    }

    return nil
}

func (r *PostgresMarketRepository) Update(ctx context.Context, id string, market *domain.Market) error {
    query := `
        UPDATE markets
        SET name = $2, description = $3, status = $4, volume = $5, updated_at = $6
        WHERE id = $1
    `

    market.UpdatedAt = time.Now()

    result, err := r.db.Exec(ctx, query,
        id,
        market.Name,
        market.Description,
        market.Status,
        market.Volume,
        market.UpdatedAt,
    )

    if err != nil {
        return fmt.Errorf("update failed: %w", err)
    }

    if result.RowsAffected() == 0 {
        return fmt.Errorf("market not found")
    }

    return nil
}

func (r *PostgresMarketRepository) Delete(ctx context.Context, id string) error {
    query := `DELETE FROM markets WHERE id = $1`

    result, err := r.db.Exec(ctx, query, id)
    if err != nil {
        return fmt.Errorf("delete failed: %w", err)
    }

    if result.RowsAffected() == 0 {
        return fmt.Errorf("market not found")
    }

    return nil
}
```

### 2.3 Database Connection Pool

```go
// internal/database/postgres.go
package database

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

type Config struct {
    Host     string
    Port     int
    User     string
    Password string
    DBName   string
    SSLMode  string
}

func NewPostgresPool(ctx context.Context, cfg Config) (*pgxpool.Pool, error) {
    dsn := fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.DBName, cfg.SSLMode,
    )

    config, err := pgxpool.ParseConfig(dsn)
    if err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }

    // Connection pool settings
    config.MaxConns = 25
    config.MinConns = 5
    config.MaxConnLifetime = time.Hour
    config.MaxConnIdleTime = 30 * time.Minute
    config.HealthCheckPeriod = time.Minute

    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        return nil, fmt.Errorf("failed to create pool: %w", err)
    }

    // Test connection
    if err := pool.Ping(ctx); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    return pool, nil
}
```

---

## 3. Redis Caching Pattern

### 3.1 Cached Repository (Cache-Aside Pattern)

```go
// internal/repository/market_cached.go
package repository

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"

    "yourapp/internal/domain"
)

type CachedMarketRepository struct {
    base  domain.MarketRepository
    redis *redis.Client
    ttl   time.Duration
}

func NewCachedMarketRepository(base domain.MarketRepository, redis *redis.Client, ttl time.Duration) *CachedMarketRepository {
    return &CachedMarketRepository{
        base:  base,
        redis: redis,
        ttl:   ttl,
    }
}

func (r *CachedMarketRepository) FindByID(ctx context.Context, id string) (*domain.Market, error) {
    cacheKey := fmt.Sprintf("market:%s", id)

    // Try cache first
    cached, err := r.redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var market domain.Market
        if err := json.Unmarshal([]byte(cached), &market); err == nil {
            return &market, nil
        }
    }

    // Cache miss - fetch from database
    market, err := r.base.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // Update cache (fire and forget)
    go func() {
        data, err := json.Marshal(market)
        if err == nil {
            r.redis.Set(context.Background(), cacheKey, data, r.ttl)
        }
    }()

    return market, nil
}

func (r *CachedMarketRepository) Create(ctx context.Context, market *domain.Market) error {
    if err := r.base.Create(ctx, market); err != nil {
        return err
    }

    // Invalidate list cache
    r.redis.Del(ctx, "markets:list:*")

    return nil
}

func (r *CachedMarketRepository) Update(ctx context.Context, id string, market *domain.Market) error {
    if err := r.base.Update(ctx, id, market); err != nil {
        return err
    }

    // Invalidate cache
    cacheKey := fmt.Sprintf("market:%s", id)
    r.redis.Del(ctx, cacheKey)

    return nil
}

func (r *CachedMarketRepository) Delete(ctx context.Context, id string) error {
    if err := r.base.Delete(ctx, id); err != nil {
        return err
    }

    // Invalidate cache
    cacheKey := fmt.Sprintf("market:%s", id)
    r.redis.Del(ctx, cacheKey)

    return nil
}

func (r *CachedMarketRepository) FindAll(ctx context.Context, filters domain.MarketFilters) ([]domain.Market, error) {
    // For simplicity, bypass cache for list queries
    // In production, consider caching with filters as key
    return r.base.FindAll(ctx, filters)
}
```

### 3.2 Redis Client Setup

```go
// internal/cache/redis.go
package cache

import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

func NewRedisClient(addr, password string, db int) (*redis.Client, error) {
    client := redis.NewClient(&redis.Options{
        Addr:         addr,
        Password:     password,
        DB:           db,
        PoolSize:     10,
        MinIdleConns: 5,
    })

    // Test connection
    if err := client.Ping(context.Background()).Err(); err != nil {
        return nil, fmt.Errorf("failed to connect to Redis: %w", err)
    }

    return client, nil
}
```

---

## 4. Service Layer Pattern

### 4.1 Service with Business Logic

```go
// internal/service/market_service.go
package service

import (
    "context"
    "fmt"

    "yourapp/internal/domain"
)

type MarketService struct {
    repo domain.MarketRepository
}

func NewMarketService(repo domain.MarketRepository) *MarketService {
    return &MarketService{repo: repo}
}

func (s *MarketService) ListMarkets(ctx context.Context, filters domain.MarketFilters) ([]domain.Market, error) {
    // Apply business rules
    if filters.Limit <= 0 {
        filters.Limit = 20 // Default limit
    }

    if filters.Limit > 100 {
        filters.Limit = 100 // Max limit
    }

    return s.repo.FindAll(ctx, filters)
}

func (s *MarketService) GetMarket(ctx context.Context, id string) (*domain.Market, error) {
    market, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get market: %w", err)
    }

    return market, nil
}

func (s *MarketService) CreateMarket(ctx context.Context, market *domain.Market) error {
    // Validate business rules
    if market.Name == "" {
        return fmt.Errorf("market name is required")
    }

    if market.Status == "" {
        market.Status = "draft" // Default status
    }

    if err := s.repo.Create(ctx, market); err != nil {
        return fmt.Errorf("failed to create market: %w", err)
    }

    // Could trigger events here (e.g., publish to queue)

    return nil
}

func (s *MarketService) UpdateMarket(ctx context.Context, id string, market *domain.Market) error {
    // Check if market exists
    existing, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return fmt.Errorf("market not found: %w", err)
    }

    // Business logic: prevent status change from 'closed' to 'active'
    if existing.Status == "closed" && market.Status == "active" {
        return fmt.Errorf("cannot reopen closed market")
    }

    if err := s.repo.Update(ctx, id, market); err != nil {
        return fmt.Errorf("failed to update market: %w", err)
    }

    return nil
}

func (s *MarketService) DeleteMarket(ctx context.Context, id string) error {
    if err := s.repo.Delete(ctx, id); err != nil {
        return fmt.Errorf("failed to delete market: %w", err)
    }

    return nil
}
```

---

## 5. HTTP Handlers

### 5.1 RESTful Handler

```go
// internal/handler/market_handler.go
package handler

import (
    "encoding/json"
    "net/http"

    "github.com/go-chi/chi/v5"

    "yourapp/internal/domain"
    "yourapp/internal/service"
)

type MarketHandler struct {
    service *service.MarketService
}

func NewMarketHandler(service *service.MarketService) *MarketHandler {
    return &MarketHandler{service: service}
}

func (h *MarketHandler) HandleListMarkets(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Parse query parameters
    filters := domain.MarketFilters{
        Limit:  20,
        Offset: 0,
    }

    if status := r.URL.Query().Get("status"); status != "" {
        filters.Status = &status
    }

    markets, err := h.service.ListMarkets(ctx, filters)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "failed to list markets")
        return
    }

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "success": true,
        "data":    markets,
    })
}

func (h *MarketHandler) HandleGetMarket(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    id := chi.URLParam(r, "id")

    market, err := h.service.GetMarket(ctx, id)
    if err != nil {
        respondError(w, http.StatusNotFound, "market not found")
        return
    }

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "success": true,
        "data":    market,
    })
}

func (h *MarketHandler) HandleCreateMarket(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    var market domain.Market
    if err := json.NewDecoder(r.Body).Decode(&market); err != nil {
        respondError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    if err := h.service.CreateMarket(ctx, &market); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    respondJSON(w, http.StatusCreated, map[string]interface{}{
        "success": true,
        "data":    market,
    })
}

func (h *MarketHandler) HandleUpdateMarket(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    id := chi.URLParam(r, "id")

    var market domain.Market
    if err := json.NewDecoder(r.Body).Decode(&market); err != nil {
        respondError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    if err := h.service.UpdateMarket(ctx, id, &market); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "success": true,
        "data":    market,
    })
}

func (h *MarketHandler) HandleDeleteMarket(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    id := chi.URLParam(r, "id")

    if err := h.service.DeleteMarket(ctx, id); err != nil {
        respondError(w, http.StatusInternalServerError, err.Error())
        return
    }

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "success": true,
        "message": "market deleted",
    })
}

// Helper functions
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}
```

---

## 6. Request Validation

### 6.1 Validator with go-playground/validator

```go
// internal/validator/validator.go
package validator

import (
    "fmt"

    "github.com/go-playground/validator/v10"
)

type Validator struct {
    validate *validator.Validate
}

func New() *Validator {
    return &Validator{
        validate: validator.New(),
    }
}

func (v *Validator) Validate(data interface{}) error {
    if err := v.validate.Struct(data); err != nil {
        if validationErrors, ok := err.(validator.ValidationErrors); ok {
            return fmt.Errorf("validation failed: %v", validationErrors)
        }
        return err
    }
    return nil
}

// Example usage
type CreateMarketRequest struct {
    Name        string  `json:"name" validate:"required,min=3,max=100"`
    Description string  `json:"description" validate:"required,max=500"`
    Volume      float64 `json:"volume" validate:"gte=0"`
}

func (h *MarketHandler) HandleCreateMarketWithValidation(w http.ResponseWriter, r *http.Request) {
    var req CreateMarketRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    // Validate
    v := validator.New()
    if err := v.Validate(req); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Process request...
}
```

---

## 7. Error Handling Patterns

### 7.1 Custom Error Types

```go
// internal/errors/errors.go
package errors

import (
    "fmt"
    "net/http"
)

type AppError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Status  int    `json:"-"`
}

func (e *AppError) Error() string {
    return e.Message
}

// Predefined errors
var (
    ErrNotFound = &AppError{
        Code:    "NOT_FOUND",
        Message: "resource not found",
        Status:  http.StatusNotFound,
    }

    ErrUnauthorized = &AppError{
        Code:    "UNAUTHORIZED",
        Message: "unauthorized",
        Status:  http.StatusUnauthorized,
    }

    ErrValidation = &AppError{
        Code:    "VALIDATION_ERROR",
        Message: "validation failed",
        Status:  http.StatusBadRequest,
    }

    ErrInternal = &AppError{
        Code:    "INTERNAL_ERROR",
        Message: "internal server error",
        Status:  http.StatusInternalServerError,
    }
)

func NewError(code, message string, status int) *AppError {
    return &AppError{
        Code:    code,
        Message: message,
        Status:  status,
    }
}
```

### 7.2 Error Wrapping

```go
// Always wrap errors with context
func (r *PostgresMarketRepository) FindByID(ctx context.Context, id string) (*domain.Market, error) {
    var m domain.Market
    err := r.db.QueryRow(ctx, query, id).Scan(&m.ID, &m.Name, ...)

    if err == pgx.ErrNoRows {
        return nil, fmt.Errorf("market not found: %w", err)
    }

    if err != nil {
        return nil, fmt.Errorf("failed to query market: %w", err)
    }

    return &m, nil
}
```

---

## 8. Context Propagation

### 8.1 Context with Timeout

```go
func (h *MarketHandler) HandleCreateMarket(w http.ResponseWriter, r *http.Request) {
    // Add timeout to context
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    var market domain.Market
    if err := json.NewDecoder(r.Body).Decode(&market); err != nil {
        respondError(w, http.StatusBadRequest, "invalid request")
        return
    }

    if err := h.service.CreateMarket(ctx, &market); err != nil {
        respondError(w, http.StatusInternalServerError, err.Error())
        return
    }

    respondJSON(w, http.StatusCreated, market)
}
```

### 8.2 Context Values

```go
// Add request ID to context (in middleware)
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.NewString()
        ctx := context.WithValue(r.Context(), "request_id", requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Access request ID in handler
func (h *Handler) SomeHandler(w http.ResponseWriter, r *http.Request) {
    requestID, _ := r.Context().Value("request_id").(string)
    log.Printf("Request ID: %s", requestID)
}
```

---

## 9. Best Practices

### ✅ DO

```go
// ✅ Always use context for cancellation
func (s *Service) FetchData(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        // Continue processing
    }
}

// ✅ Always check errors (NEVER ignore)
data, err := json.Marshal(obj)
if err != nil {
    return fmt.Errorf("failed to marshal: %w", err)
}

// ✅ Use pointer receivers for methods that modify state
func (r *Repository) Update(ctx context.Context, id string) error {
    r.cache.Invalidate(id)
    return nil
}

// ✅ Use value receivers for read-only methods on small types
func (m Market) IsActive() bool {
    return m.Status == "active"
}

// ✅ Use parameterized queries (pgx handles this)
query := "SELECT * FROM markets WHERE id = $1"
err := db.QueryRow(ctx, query, id).Scan(...)

// ✅ Close rows after use
rows, err := db.Query(ctx, query)
if err != nil {
    return err
}
defer rows.Close()

// ✅ Use structured logging
slog.Info("market created",
    slog.String("market_id", market.ID),
    slog.String("name", market.Name),
)

// ✅ Graceful shutdown
quit := make(chan os.Signal, 1)
signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
<-quit
```

### ❌ DON'T

```go
// ❌ Never ignore errors
data, _ := json.Marshal(obj) // WRONG

// ❌ Don't use SELECT * in production
query := "SELECT * FROM markets" // WRONG
query := "SELECT id, name, status FROM markets" // CORRECT

// ❌ Don't forget to defer Close()
rows, _ := db.Query(ctx, query)
// WRONG: Forgot defer rows.Close()

// ❌ Don't use string concatenation for SQL (SQL injection risk)
query := "SELECT * FROM markets WHERE id = '" + id + "'" // DANGEROUS
query := "SELECT * FROM markets WHERE id = $1" // CORRECT

// ❌ Don't block with fmt.Println in production
fmt.Println("Debug info") // WRONG
slog.Info("debug info") // CORRECT

// ❌ Don't pass database connections without context
func (r *Repo) Query() error {
    return r.db.Query(query) // WRONG (no context)
}

// ❌ Don't create new connections per request
func handler(w http.ResponseWriter, r *http.Request) {
    db, _ := sql.Open(...) // WRONG
}
```

---

## 10. Project Structure

```
project-root/
├── cmd/
│   ├── api/              # API server entry point
│   │   └── main.go
│   └── worker/           # Background worker entry point
│       └── main.go
├── internal/
│   ├── domain/           # Business entities
│   │   ├── market.go
│   │   └── order.go
│   ├── repository/       # Data access layer
│   │   ├── market_postgres.go
│   │   ├── market_cached.go
│   │   └── order_postgres.go
│   ├── service/          # Business logic
│   │   ├── market_service.go
│   │   └── order_service.go
│   ├── handler/          # HTTP handlers
│   │   ├── market_handler.go
│   │   └── order_handler.go
│   ├── middleware/       # Chi middleware
│   │   ├── auth.go
│   │   ├── logger.go
│   │   └── ratelimit.go
│   ├── database/         # Database setup
│   │   └── postgres.go
│   ├── cache/            # Cache setup
│   │   └── redis.go
│   └── errors/           # Custom errors
│       └── errors.go
├── migrations/           # Database migrations
│   ├── 001_create_markets.up.sql
│   └── 001_create_markets.down.sql
├── pkg/                  # Public libraries (optional)
├── config/               # Configuration
│   └── config.go
├── go.mod
└── go.sum
```

---

## Quick Reference

### Chi Router
```go
r := chi.NewRouter()
r.Use(middleware.Logger)
r.Get("/path", handler)
r.Route("/api", func(r chi.Router) {
    r.Get("/users", getUsers)
})
```

### PostgreSQL with pgx
```go
pool, _ := pgxpool.New(ctx, dsn)
rows, _ := pool.Query(ctx, "SELECT * FROM markets WHERE id = $1", id)
defer rows.Close()
```

### Redis Caching
```go
client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
client.Set(ctx, "key", "value", 5*time.Minute)
val, _ := client.Get(ctx, "key").Result()
```

### Error Wrapping
```go
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}
```

---

## Resources

- [Chi Router Documentation](https://github.com/go-chi/chi)
- [pgx PostgreSQL Driver](https://github.com/jackc/pgx)
- [go-redis](https://github.com/redis/go-redis)
- [go-playground/validator](https://github.com/go-playground/validator)
- [Effective Go](https://go.dev/doc/effective_go)
