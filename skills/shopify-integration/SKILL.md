---
skill_id: shopify-integration
name: Shopify Integration Patterns
description: Comprehensive patterns for building Shopify apps with Go backend - OAuth, webhooks, GraphQL/REST APIs, App Bridge, and GDPR compliance
category: integration
tags: [shopify, oauth, webhooks, graphql, app-bridge, gdpr]
applies_to: [go]
auto_trigger: ["shopify", "app-bridge", "oauth", "webhook", "admin-api", "metafield"]
---

# Shopify Integration Patterns

Comprehensive guide for building Shopify apps with Go backend. Covers authentication, API integration, webhook handling, and compliance.

## Core Concepts

### Shopify App Architecture

```
+-------------------------------------------------------------+
|                     Shopify Admin                           |
|  +------------------------------------------------------+   |
|  |         Embedded App (iframe)                        |   |
|  |  React + Vite + Polaris Web Components + App Bridge  |   |
|  +-------------------------+----------------------------+   |
|                            | API Calls                      |
+----------------------------+--------------------------------+
                             |
                             v
               +---------------------------+
               |   Go Backend (Chi)        |
               |  +---------------------+  |
               |  | OAuth Handler       |  |
               |  | Session Manager     |  |
               |  | API Middleware      |  |
               |  | Webhook Receivers   |  |
               |  +---------------------+  |
               +-----------+---------------+
                           |
             +-------------+-------------+
             v             v             v
        PostgreSQL      Redis       RabbitMQ
        (Sessions)     (Cache)     (Webhooks)
```

---

## 1. OAuth Authentication

### 1.1 OAuth Flow Implementation

```go
// internal/shopify/oauth.go
package shopify

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "net/http"
    "net/url"
    "strings"
)

type OAuthConfig struct {
    ClientID     string
    ClientSecret string
    RedirectURI  string
    Scopes       []string
}

// VerifyHMAC verifies the HMAC signature from Shopify
func (c *OAuthConfig) VerifyHMAC(queryParams url.Values) bool {
    receivedHMAC := queryParams.Get("hmac")
    queryParams.Del("hmac")

    // Build message from query params (sorted alphabetically)
    var params []string
    for key, values := range queryParams {
        for _, value := range values {
            params = append(params, fmt.Sprintf("%s=%s", key, value))
        }
    }
    sort.Strings(params)
    message := strings.Join(params, "&")

    // Compute HMAC
    mac := hmac.New(sha256.New, []byte(c.ClientSecret))
    mac.Write([]byte(message))
    expectedHMAC := hex.EncodeToString(mac.Sum(nil))

    return hmac.Equal([]byte(receivedHMAC), []byte(expectedHMAC))
}

// GetAuthorizationURL generates the OAuth authorization URL
func (c *OAuthConfig) GetAuthorizationURL(shop, state, nonce string) string {
    params := url.Values{}
    params.Set("client_id", c.ClientID)
    params.Set("scope", strings.Join(c.Scopes, ","))
    params.Set("redirect_uri", c.RedirectURI)
    params.Set("state", state)
    params.Set("grant_options[]", "per-user")

    return fmt.Sprintf("https://%s/admin/oauth/authorize?%s", shop, params.Encode())
}

// ExchangeCodeForToken exchanges authorization code for access token
func (c *OAuthConfig) ExchangeCodeForToken(shop, code string) (*TokenResponse, error) {
    data := url.Values{}
    data.Set("client_id", c.ClientID)
    data.Set("client_secret", c.ClientSecret)
    data.Set("code", code)

    tokenURL := fmt.Sprintf("https://%s/admin/oauth/access_token", shop)

    resp, err := http.PostForm(tokenURL, data)
    if err != nil {
        return nil, fmt.Errorf("token exchange failed: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("token exchange returned %d", resp.StatusCode)
    }

    var tokenResp TokenResponse
    if err := json.NewDecoder(resp.Body).Decode(&tokenResp); err != nil {
        return nil, fmt.Errorf("failed to decode token response: %w", err)
    }

    return &tokenResp, nil
}

type TokenResponse struct {
    AccessToken string `json:"access_token"`
    Scope       string `json:"scope"`
}
```

### 1.2 OAuth Handler (Chi Router)

```go
// internal/handler/oauth_handler.go
package handler

import (
    "crypto/rand"
    "encoding/base64"
    "net/http"

    "github.com/go-chi/chi/v5"
)

type OAuthHandler struct {
    config      *shopify.OAuthConfig
    sessionRepo SessionRepository
}

// HandleInstall initiates OAuth flow
func (h *OAuthHandler) HandleInstall(w http.ResponseWriter, r *http.Request) {
    shop := r.URL.Query().Get("shop")
    if shop == "" {
        http.Error(w, "missing shop parameter", http.StatusBadRequest)
        return
    }

    // Validate shop domain
    if !isValidShopDomain(shop) {
        http.Error(w, "invalid shop domain", http.StatusBadRequest)
        return
    }

    // Generate state for CSRF protection
    state, err := generateRandomString(32)
    if err != nil {
        http.Error(w, "failed to generate state", http.StatusInternalServerError)
        return
    }

    // Store state in session (with expiry)
    if err := h.sessionRepo.StoreOAuthState(r.Context(), shop, state); err != nil {
        http.Error(w, "failed to store state", http.StatusInternalServerError)
        return
    }

    // Redirect to Shopify OAuth
    authURL := h.config.GetAuthorizationURL(shop, state, "")
    http.Redirect(w, r, authURL, http.StatusFound)
}

// HandleCallback handles OAuth callback
func (h *OAuthHandler) HandleCallback(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query()

    // Verify HMAC
    if !h.config.VerifyHMAC(query) {
        http.Error(w, "invalid HMAC signature", http.StatusUnauthorized)
        return
    }

    shop := query.Get("shop")
    code := query.Get("code")
    state := query.Get("state")

    // Verify state (CSRF protection)
    storedState, err := h.sessionRepo.GetOAuthState(r.Context(), shop)
    if err != nil || storedState != state {
        http.Error(w, "invalid state parameter", http.StatusUnauthorized)
        return
    }

    // Exchange code for token
    tokenResp, err := h.config.ExchangeCodeForToken(shop, code)
    if err != nil {
        http.Error(w, "failed to exchange token", http.StatusInternalServerError)
        return
    }

    // Store access token
    session := &Session{
        Shop:        shop,
        AccessToken: tokenResp.AccessToken,
        Scope:       tokenResp.Scope,
    }

    if err := h.sessionRepo.SaveSession(r.Context(), session); err != nil {
        http.Error(w, "failed to save session", http.StatusInternalServerError)
        return
    }

    // Redirect to app
    http.Redirect(w, r, fmt.Sprintf("https://%s/admin/apps/%s", shop, os.Getenv("APP_HANDLE")), http.StatusFound)
}

func generateRandomString(length int) (string, error) {
    bytes := make([]byte, length)
    if _, err := rand.Read(bytes); err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(bytes), nil
}

func isValidShopDomain(shop string) bool {
    return strings.HasSuffix(shop, ".myshopify.com")
}
```

### 1.3 Session Token Authentication (Embedded Apps)

For embedded apps using App Bridge, use session tokens instead of OAuth for API requests:

```go
// internal/middleware/session_token.go
package middleware

import (
    "context"
    "fmt"
    "net/http"
    "strings"

    "github.com/golang-jwt/jwt/v5"
)

type SessionTokenMiddleware struct {
    clientSecret string
}

func NewSessionTokenMiddleware(clientSecret string) *SessionTokenMiddleware {
    return &SessionTokenMiddleware{clientSecret: clientSecret}
}

func (m *SessionTokenMiddleware) Verify(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract token from Authorization header
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "missing authorization header", http.StatusUnauthorized)
            return
        }

        tokenString := strings.TrimPrefix(authHeader, "Bearer ")

        // Parse and validate JWT
        token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
            }
            return []byte(m.clientSecret), nil
        })

        if err != nil || !token.Valid {
            http.Error(w, "invalid session token", http.StatusUnauthorized)
            return
        }

        claims, ok := token.Claims.(jwt.MapClaims)
        if !ok {
            http.Error(w, "invalid token claims", http.StatusUnauthorized)
            return
        }

        // Extract shop domain (dest field)
        dest, ok := claims["dest"].(string)
        if !ok {
            http.Error(w, "missing dest claim", http.StatusUnauthorized)
            return
        }

        shop := strings.TrimPrefix(dest, "https://")

        // Add shop to context
        ctx := context.WithValue(r.Context(), "shop", shop)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## 2. GraphQL Admin API

### 2.1 GraphQL Client

```go
// internal/shopify/graphql_client.go
package shopify

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
)

type GraphQLClient struct {
    shop        string
    accessToken string
    httpClient  *http.Client
}

func NewGraphQLClient(shop, accessToken string) *GraphQLClient {
    return &GraphQLClient{
        shop:        shop,
        accessToken: accessToken,
        httpClient:  &http.Client{},
    }
}

type GraphQLRequest struct {
    Query     string                 `json:"query"`
    Variables map[string]interface{} `json:"variables,omitempty"`
}

type GraphQLResponse struct {
    Data   json.RawMessage `json:"data"`
    Errors []GraphQLError  `json:"errors,omitempty"`
}

type GraphQLError struct {
    Message string `json:"message"`
    Path    []any  `json:"path,omitempty"`
}

func (c *GraphQLClient) Query(ctx context.Context, query string, variables map[string]interface{}, result interface{}) error {
    req := GraphQLRequest{
        Query:     query,
        Variables: variables,
    }

    body, err := json.Marshal(req)
    if err != nil {
        return fmt.Errorf("failed to marshal request: %w", err)
    }

    url := fmt.Sprintf("https://%s/admin/api/2024-01/graphql.json", c.shop)
    httpReq, err := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
    if err != nil {
        return fmt.Errorf("failed to create request: %w", err)
    }

    httpReq.Header.Set("Content-Type", "application/json")
    httpReq.Header.Set("X-Shopify-Access-Token", c.accessToken)

    resp, err := c.httpClient.Do(httpReq)
    if err != nil {
        return fmt.Errorf("request failed: %w", err)
    }
    defer resp.Body.Close()

    // Check for rate limiting
    if resp.StatusCode == http.StatusTooManyRequests {
        return fmt.Errorf("rate limited: %s", resp.Header.Get("Retry-After"))
    }

    if resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("request failed with status %d: %s", resp.StatusCode, string(body))
    }

    var graphQLResp GraphQLResponse
    if err := json.NewDecoder(resp.Body).Decode(&graphQLResp); err != nil {
        return fmt.Errorf("failed to decode response: %w", err)
    }

    if len(graphQLResp.Errors) > 0 {
        return fmt.Errorf("graphql errors: %v", graphQLResp.Errors)
    }

    if err := json.Unmarshal(graphQLResp.Data, result); err != nil {
        return fmt.Errorf("failed to unmarshal data: %w", err)
    }

    return nil
}
```

### 2.2 Common GraphQL Queries

```go
// internal/shopify/queries.go
package shopify

const (
    // Get shop information
    QueryShopInfo = `
        query {
            shop {
                id
                name
                email
                myshopifyDomain
                plan {
                    displayName
                }
            }
        }
    `

    // Get products with pagination
    QueryProducts = `
        query GetProducts($first: Int!, $after: String) {
            products(first: $first, after: $after) {
                edges {
                    node {
                        id
                        title
                        handle
                        status
                        variants(first: 10) {
                            edges {
                                node {
                                    id
                                    title
                                    price
                                    sku
                                }
                            }
                        }
                    }
                    cursor
                }
                pageInfo {
                    hasNextPage
                    endCursor
                }
            }
        }
    `

    // Create product mutation
    MutationCreateProduct = `
        mutation CreateProduct($input: ProductInput!) {
            productCreate(input: $input) {
                product {
                    id
                    title
                    handle
                }
                userErrors {
                    field
                    message
                }
            }
        }
    `

    // Update metafield
    MutationUpdateMetafield = `
        mutation UpdateMetafield($metafields: [MetafieldsSetInput!]!) {
            metafieldsSet(metafields: $metafields) {
                metafields {
                    id
                    namespace
                    key
                    value
                }
                userErrors {
                    field
                    message
                }
            }
        }
    `
)

// Example usage
func (s *ShopifyService) GetProducts(ctx context.Context, shop string, first int, after *string) (*ProductConnection, error) {
    client := s.getClient(shop)

    variables := map[string]interface{}{
        "first": first,
    }
    if after != nil {
        variables["after"] = *after
    }

    var result struct {
        Products ProductConnection `json:"products"`
    }

    if err := client.Query(ctx, QueryProducts, variables, &result); err != nil {
        return nil, err
    }

    return &result.Products, nil
}
```

### 2.3 Rate Limiting with Retries

```go
// internal/shopify/rate_limiter.go
package shopify

import (
    "context"
    "fmt"
    "strconv"
    "time"
)

type RateLimitedClient struct {
    client     *GraphQLClient
    maxRetries int
}

func NewRateLimitedClient(client *GraphQLClient, maxRetries int) *RateLimitedClient {
    return &RateLimitedClient{
        client:     client,
        maxRetries: maxRetries,
    }
}

func (c *RateLimitedClient) QueryWithRetry(ctx context.Context, query string, variables map[string]interface{}, result interface{}) error {
    var lastErr error

    for attempt := 0; attempt <= c.maxRetries; attempt++ {
        err := c.client.Query(ctx, query, variables, result)
        if err == nil {
            return nil
        }

        // Check if rate limited
        if isRateLimitError(err) {
            retryAfter := extractRetryAfter(err)
            if retryAfter > 0 {
                select {
                case <-time.After(retryAfter):
                    continue
                case <-ctx.Done():
                    return ctx.Err()
                }
            }

            // Exponential backoff if no retry-after header
            backoff := time.Duration(1<<uint(attempt)) * time.Second
            select {
            case <-time.After(backoff):
                continue
            case <-ctx.Done():
                return ctx.Err()
            }
        }

        lastErr = err
        break
    }

    return lastErr
}

func isRateLimitError(err error) bool {
    return err != nil && (
        strings.Contains(err.Error(), "rate limited") ||
        strings.Contains(err.Error(), "429") ||
        strings.Contains(err.Error(), "THROTTLED"))
}

func extractRetryAfter(err error) time.Duration {
    // Parse retry-after from error message
    // This is simplified - adjust based on actual error format
    return 2 * time.Second
}
```

---

## 3. Webhook Handling

### 3.1 Webhook Verification

```go
// internal/shopify/webhook.go
package shopify

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "io"
    "net/http"
)

type WebhookVerifier struct {
    secret string
}

func NewWebhookVerifier(secret string) *WebhookVerifier {
    return &WebhookVerifier{secret: secret}
}

// VerifyWebhook verifies the HMAC signature of a webhook request
func (v *WebhookVerifier) VerifyWebhook(r *http.Request, body []byte) bool {
    hmacHeader := r.Header.Get("X-Shopify-Hmac-Sha256")
    if hmacHeader == "" {
        return false
    }

    mac := hmac.New(sha256.New, []byte(v.secret))
    mac.Write(body)
    expectedMAC := base64.StdEncoding.EncodeToString(mac.Sum(nil))

    return hmac.Equal([]byte(hmacHeader), []byte(expectedMAC))
}

// WebhookMiddleware verifies webhook authenticity
func (v *WebhookVerifier) WebhookMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Read body
        body, err := io.ReadAll(r.Body)
        if err != nil {
            http.Error(w, "failed to read body", http.StatusBadRequest)
            return
        }
        defer r.Body.Close()

        // Verify HMAC
        if !v.VerifyWebhook(r, body) {
            http.Error(w, "invalid webhook signature", http.StatusUnauthorized)
            return
        }

        // Store body in context for handler
        ctx := context.WithValue(r.Context(), "webhook_body", body)

        // Add webhook metadata to context
        ctx = context.WithValue(ctx, "shop_domain", r.Header.Get("X-Shopify-Shop-Domain"))
        ctx = context.WithValue(ctx, "webhook_topic", r.Header.Get("X-Shopify-Topic"))
        ctx = context.WithValue(ctx, "webhook_id", r.Header.Get("X-Shopify-Webhook-Id"))

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 3.2 Webhook Handlers

```go
// internal/handler/webhook_handler.go
package handler

import (
    "context"
    "encoding/json"
    "net/http"
)

type WebhookHandler struct {
    queue WebhookQueue
}

// HandleOrderCreate handles orders/create webhook
func (h *WebhookHandler) HandleOrderCreate(w http.ResponseWriter, r *http.Request) {
    body := r.Context().Value("webhook_body").([]byte)
    shop := r.Context().Value("shop_domain").(string)

    var order Order
    if err := json.Unmarshal(body, &order); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    // Queue for async processing
    if err := h.queue.EnqueueOrderCreated(r.Context(), shop, &order); err != nil {
        http.Error(w, "failed to queue webhook", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusOK)
}

// HandleProductUpdate handles products/update webhook
func (h *WebhookHandler) HandleProductUpdate(w http.ResponseWriter, r *http.Request) {
    body := r.Context().Value("webhook_body").([]byte)
    shop := r.Context().Value("shop_domain").(string)

    var product Product
    if err := json.Unmarshal(body, &product); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    // Queue for async processing
    if err := h.queue.EnqueueProductUpdated(r.Context(), shop, &product); err != nil {
        http.Error(w, "failed to queue webhook", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusOK)
}

// HandleAppUninstalled handles app/uninstalled webhook (CRITICAL)
func (h *WebhookHandler) HandleAppUninstalled(w http.ResponseWriter, r *http.Request) {
    shop := r.Context().Value("shop_domain").(string)

    // Delete shop data immediately (or queue for cleanup)
    if err := h.queue.EnqueueAppUninstalled(r.Context(), shop); err != nil {
        http.Error(w, "failed to process uninstall", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusOK)
}
```

### 3.3 GDPR Webhooks (MANDATORY)

```go
// internal/handler/gdpr_webhook_handler.go
package handler

import (
    "encoding/json"
    "net/http"
)

type GDPRWebhookHandler struct {
    service GDPRService
}

// HandleCustomersDataRequest handles customers/data_request webhook
// MUST respond within 30 days with customer data
func (h *GDPRWebhookHandler) HandleCustomersDataRequest(w http.ResponseWriter, r *http.Request) {
    body := r.Context().Value("webhook_body").([]byte)

    var request CustomerDataRequest
    if err := json.Unmarshal(body, &request); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    // Queue for processing (respond to merchant within 30 days)
    if err := h.service.QueueDataRequest(r.Context(), &request); err != nil {
        http.Error(w, "failed to queue request", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusOK)
}

// HandleCustomersRedact handles customers/redact webhook
// MUST delete customer data within 30 days (if no other orders)
func (h *GDPRWebhookHandler) HandleCustomersRedact(w http.ResponseWriter, r *http.Request) {
    body := r.Context().Value("webhook_body").([]byte)

    var request CustomerRedactRequest
    if err := json.Unmarshal(body, &request); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    // Queue for deletion
    if err := h.service.QueueCustomerRedaction(r.Context(), &request); err != nil {
        http.Error(w, "failed to queue redaction", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusOK)
}

// HandleShopRedact handles shop/redact webhook
// MUST delete all shop data within 48 hours
func (h *GDPRWebhookHandler) HandleShopRedact(w http.ResponseWriter, r *http.Request) {
    body := r.Context().Value("webhook_body").([]byte)

    var request ShopRedactRequest
    if err := json.Unmarshal(body, &request); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    // Queue for immediate deletion (48-hour deadline)
    if err := h.service.QueueShopRedaction(r.Context(), &request); err != nil {
        http.Error(w, "failed to queue redaction", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusOK)
}

type CustomerDataRequest struct {
    ShopID       int64  `json:"shop_id"`
    ShopDomain   string `json:"shop_domain"`
    CustomerID   int64  `json:"customer_id"`
    OrdersToRedact []int64 `json:"orders_to_redact"`
}

type CustomerRedactRequest struct {
    ShopID       int64  `json:"shop_id"`
    ShopDomain   string `json:"shop_domain"`
    CustomerID   int64  `json:"customer_id"`
    OrdersToRedact []int64 `json:"orders_to_redact"`
}

type ShopRedactRequest struct {
    ShopID     int64  `json:"shop_id"`
    ShopDomain string `json:"shop_domain"`
}
```

### 3.4 Webhook Registration

```go
// internal/shopify/webhook_registration.go
package shopify

const MutationRegisterWebhook = `
    mutation webhookSubscriptionCreate($topic: WebhookSubscriptionTopic!, $webhookSubscription: WebhookSubscriptionInput!) {
        webhookSubscriptionCreate(
            topic: $topic
            webhookSubscription: $webhookSubscription
        ) {
            webhookSubscription {
                id
                topic
                endpoint {
                    __typename
                    ... on WebhookHttpEndpoint {
                        callbackUrl
                    }
                }
            }
            userErrors {
                field
                message
            }
        }
    }
`

func (s *ShopifyService) RegisterWebhook(ctx context.Context, shop, topic, callbackURL string) error {
    client := s.getClient(shop)

    variables := map[string]interface{}{
        "topic": topic,
        "webhookSubscription": map[string]interface{}{
            "callbackUrl": callbackURL,
            "format":      "JSON",
        },
    }

    var result struct {
        WebhookSubscriptionCreate struct {
            WebhookSubscription *WebhookSubscription `json:"webhookSubscription"`
            UserErrors          []UserError          `json:"userErrors"`
        } `json:"webhookSubscriptionCreate"`
    }

    if err := client.Query(ctx, MutationRegisterWebhook, variables, &result); err != nil {
        return fmt.Errorf("failed to register webhook: %w", err)
    }

    if len(result.WebhookSubscriptionCreate.UserErrors) > 0 {
        return fmt.Errorf("webhook registration errors: %v", result.WebhookSubscriptionCreate.UserErrors)
    }

    return nil
}

// Register all required webhooks after OAuth
func (s *ShopifyService) RegisterAllWebhooks(ctx context.Context, shop string) error {
    baseURL := os.Getenv("APP_URL")

    webhooks := []struct {
        topic string
        path  string
    }{
        {"ORDERS_CREATE", "/webhooks/orders/create"},
        {"ORDERS_UPDATED", "/webhooks/orders/update"},
        {"PRODUCTS_UPDATE", "/webhooks/products/update"},
        {"APP_UNINSTALLED", "/webhooks/app/uninstalled"},
        {"CUSTOMERS_DATA_REQUEST", "/webhooks/gdpr/customers_data_request"},
        {"CUSTOMERS_REDACT", "/webhooks/gdpr/customers_redact"},
        {"SHOP_REDACT", "/webhooks/gdpr/shop_redact"},
    }

    for _, wh := range webhooks {
        callbackURL := fmt.Sprintf("%s%s", baseURL, wh.path)
        if err := s.RegisterWebhook(ctx, shop, wh.topic, callbackURL); err != nil {
            return fmt.Errorf("failed to register %s: %w", wh.topic, err)
        }
    }

    return nil
}
```

---

## 4. App Bridge (Frontend Integration)

With Polaris Web Components, App Bridge uses direct initialization instead of React Providers.

### 4.1 App Bridge Setup (Direct Initialization)

```typescript
// frontend/src/lib/app-bridge.ts
import { createApp, type ClientApplication } from '@shopify/app-bridge';
import { getSessionToken } from '@shopify/app-bridge/utilities';

let app: ClientApplication | null = null;

/**
 * Get or create the App Bridge instance.
 * Uses singleton pattern for consistent access across the app.
 */
export function getAppBridge(): ClientApplication {
  if (!app) {
    const host = new URLSearchParams(window.location.search).get('host');

    if (!host) {
      throw new Error('Missing host parameter. Access app from Shopify admin.');
    }

    app = createApp({
      apiKey: import.meta.env.VITE_SHOPIFY_API_KEY,
      host,
    });
  }
  return app;
}

/**
 * Get the current session token for API authentication.
 */
export async function getAuthToken(): Promise<string> {
  const appBridge = getAppBridge();
  return getSessionToken(appBridge);
}

/**
 * Check if the app is running in Shopify context.
 */
export function isShopifyContext(): boolean {
  return !!new URLSearchParams(window.location.search).get('host');
}
```

### 4.2 App Frame with Error Handling

```typescript
// frontend/src/components/AppFrame.tsx
import { isShopifyContext } from '@/lib/app-bridge';

interface AppFrameProps {
  children: React.ReactNode;
}

export function AppFrame({ children }: AppFrameProps) {
  if (!isShopifyContext()) {
    return (
      <s-page title="Error">
        <s-section>
          <s-banner status="critical" heading="Access Error">
            <s-text>
              Missing host parameter. Please access this app from the Shopify admin.
            </s-text>
          </s-banner>
        </s-section>
      </s-page>
    );
  }

  return <>{children}</>;
}
```

### 4.3 Authenticated Fetch with Session Token

```typescript
// frontend/src/hooks/useAuthenticatedFetch.ts
import { useCallback } from 'react';
import { getAuthToken } from '@/lib/app-bridge';

const API_URL = import.meta.env.VITE_API_URL;

export function useAuthenticatedFetch() {
  return useCallback(async (uri: string, options?: RequestInit) => {
    const sessionToken = await getAuthToken();
    const url = uri.startsWith('http') ? uri : `${API_URL}${uri}`;

    const response = await fetch(url, {
      ...options,
      headers: {
        ...options?.headers,
        'Authorization': `Bearer ${sessionToken}`,
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      throw new Error(`Request failed: ${response.statusText}`);
    }

    return response;
  }, []);
}
```

### 4.4 Usage with TanStack Query

```typescript
// frontend/src/hooks/useProducts.ts
import { useQuery } from '@tanstack/react-query';
import { useAuthenticatedFetch } from './useAuthenticatedFetch';

// Query key factory for consistent cache management
export const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (filters: Record<string, string>) => [...productKeys.lists(), filters] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: string) => [...productKeys.details(), id] as const,
};

export function useProducts(filters?: Record<string, string>) {
  const fetch = useAuthenticatedFetch();

  return useQuery({
    queryKey: productKeys.list(filters ?? {}),
    queryFn: async () => {
      const params = new URLSearchParams(filters);
      const response = await fetch(`/api/products?${params}`);
      return response.json();
    },
  });
}
```

### 4.5 App Bridge Actions (Toast, Navigation)

```typescript
// frontend/src/hooks/useToast.ts
import { Toast } from '@shopify/app-bridge/actions';
import { getAppBridge } from '@/lib/app-bridge';

export function useToast() {
  const showToast = (message: string, options?: { isError?: boolean; duration?: number }) => {
    const app = getAppBridge();
    const toast = Toast.create(app, {
      message,
      duration: options?.duration ?? 3000,
      isError: options?.isError ?? false,
    });
    toast.dispatch(Toast.Action.SHOW);
  };

  return { showToast };
}

// frontend/src/hooks/useNavigation.ts
import { Redirect } from '@shopify/app-bridge/actions';
import { getAppBridge } from '@/lib/app-bridge';

export function useNavigation() {
  const navigateTo = (path: string) => {
    const app = getAppBridge();
    const redirect = Redirect.create(app);
    redirect.dispatch(Redirect.Action.APP, path);
  };

  const navigateToAdmin = (path: string) => {
    const app = getAppBridge();
    const redirect = Redirect.create(app);
    redirect.dispatch(Redirect.Action.ADMIN_PATH, path);
  };

  return { navigateTo, navigateToAdmin };
}
```

---

## 5. Metafields Management

### 5.1 Set Metafields

```go
// internal/shopify/metafield.go
package shopify

func (s *ShopifyService) SetProductMetafield(ctx context.Context, shop, productID, namespace, key, value, valueType string) error {
    client := s.getClient(shop)

    variables := map[string]interface{}{
        "metafields": []map[string]interface{}{
            {
                "ownerId":   productID,
                "namespace": namespace,
                "key":       key,
                "value":     value,
                "type":      valueType, // e.g., "single_line_text_field", "json", "number_integer"
            },
        },
    }

    var result struct {
        MetafieldsSet struct {
            Metafields []Metafield `json:"metafields"`
            UserErrors []UserError `json:"userErrors"`
        } `json:"metafieldsSet"`
    }

    if err := client.Query(ctx, MutationUpdateMetafield, variables, &result); err != nil {
        return fmt.Errorf("failed to set metafield: %w", err)
    }

    if len(result.MetafieldsSet.UserErrors) > 0 {
        return fmt.Errorf("metafield errors: %v", result.MetafieldsSet.UserErrors)
    }

    return nil
}
```

---

## 6. Best Practices

### ✅ DO

```go
// ✅ Always verify webhook HMAC signatures
func (v *WebhookVerifier) VerifyWebhook(r *http.Request, body []byte) bool {
    hmacHeader := r.Header.Get("X-Shopify-Hmac-Sha256")
    mac := hmac.New(sha256.New, []byte(v.secret))
    mac.Write(body)
    expectedMAC := base64.StdEncoding.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(hmacHeader), []byte(expectedMAC))
}

// ✅ Handle rate limiting with exponential backoff
func (c *RateLimitedClient) QueryWithRetry(ctx context.Context, query string) error {
    for attempt := 0; attempt <= c.maxRetries; attempt++ {
        err := c.client.Query(ctx, query, nil, nil)
        if err == nil {
            return nil
        }
        if isRateLimitError(err) {
            backoff := time.Duration(1<<uint(attempt)) * time.Second
            time.Sleep(backoff)
            continue
        }
        return err
    }
    return fmt.Errorf("max retries exceeded")
}

// ✅ Use context for cancellation
func (s *ShopifyService) FetchProducts(ctx context.Context, shop string) error {
    client := s.getClient(shop)
    return client.Query(ctx, QueryProducts, nil, nil)
}

// ✅ Queue webhooks for async processing
func (h *WebhookHandler) HandleOrderCreate(w http.ResponseWriter, r *http.Request) {
    h.queue.Enqueue(r.Context(), order)
    w.WriteHeader(http.StatusOK) // Respond immediately
}

// ✅ Register GDPR webhooks (MANDATORY for public apps)
webhooks := []string{
    "CUSTOMERS_DATA_REQUEST",
    "CUSTOMERS_REDACT",
    "SHOP_REDACT",
}

// ✅ Validate shop domain before OAuth
func isValidShopDomain(shop string) bool {
    return strings.HasSuffix(shop, ".myshopify.com")
}
```

### ❌ DON'T

```go
// ❌ Never skip webhook verification
func HandleWebhook(w http.ResponseWriter, r *http.Request) {
    // DANGEROUS: Processing unverified webhooks
    body, _ := io.ReadAll(r.Body)
    processWebhook(body)
}

// ❌ Don't expose client secret in frontend
// Store in backend environment variables only
const ClientSecret = "shpss_abc123" // WRONG

// ❌ Don't process webhooks synchronously
func HandleOrderCreate(w http.ResponseWriter, r *http.Request) {
    // SLOW: Blocks webhook response
    processOrder(order)
    time.Sleep(5 * time.Second)
    w.WriteHeader(http.StatusOK)
}

// ❌ Don't ignore rate limiting
func FetchAllProducts() {
    // DANGEROUS: No rate limit handling
    for {
        client.Query(ctx, query, nil, nil)
    }
}

// ❌ Don't forget to handle app/uninstalled webhook
// MUST clean up shop data when app is uninstalled

// ❌ Don't store access tokens in plain text
// Encrypt sensitive data in database
```

---

## 7. Testing Patterns

### 7.1 Mock Shopify Client

```go
// internal/shopify/mock_client.go
package shopify

type MockGraphQLClient struct {
    QueryFunc func(ctx context.Context, query string, variables map[string]interface{}, result interface{}) error
}

func (m *MockGraphQLClient) Query(ctx context.Context, query string, variables map[string]interface{}, result interface{}) error {
    if m.QueryFunc != nil {
        return m.QueryFunc(ctx, query, variables, result)
    }
    return nil
}
```

### 7.2 Test Webhook Verification

```go
// internal/shopify/webhook_test.go
package shopify_test

import (
    "bytes"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/stretchr/testify/assert"
)

func TestWebhookVerification(t *testing.T) {
    secret := "test-secret"
    verifier := shopify.NewWebhookVerifier(secret)

    tests := []struct {
        name       string
        body       []byte
        hmac       string
        wantValid  bool
    }{
        {
            name:      "valid webhook",
            body:      []byte(`{"id": 123}`),
            hmac:      computeHMAC(t, secret, []byte(`{"id": 123}`)),
            wantValid: true,
        },
        {
            name:      "invalid hmac",
            body:      []byte(`{"id": 123}`),
            hmac:      "invalid-hmac",
            wantValid: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req := httptest.NewRequest("POST", "/webhook", bytes.NewReader(tt.body))
            req.Header.Set("X-Shopify-Hmac-Sha256", tt.hmac)

            valid := verifier.VerifyWebhook(req, tt.body)
            assert.Equal(t, tt.wantValid, valid)
        })
    }
}
```

---

## 8. Common Patterns

### Pattern 1: Pagination

```go
func (s *ShopifyService) FetchAllProducts(ctx context.Context, shop string) ([]Product, error) {
    var allProducts []Product
    var cursor *string

    for {
        conn, err := s.GetProducts(ctx, shop, 50, cursor)
        if err != nil {
            return nil, err
        }

        for _, edge := range conn.Edges {
            allProducts = append(allProducts, edge.Node)
        }

        if !conn.PageInfo.HasNextPage {
            break
        }

        cursor = &conn.PageInfo.EndCursor
    }

    return allProducts, nil
}
```

### Pattern 2: Bulk Operations

```go
const MutationBulkOperationRunQuery = `
    mutation {
        bulkOperationRunQuery(
            query: """
                {
                    products {
                        edges {
                            node {
                                id
                                title
                            }
                        }
                    }
                }
            """
        ) {
            bulkOperation {
                id
                status
            }
            userErrors {
                field
                message
            }
        }
    }
`
```

---

## Quick Reference

### Required Webhooks
- `APP_UNINSTALLED` - Clean up shop data
- `CUSTOMERS_DATA_REQUEST` - GDPR data request (30 days)
- `CUSTOMERS_REDACT` - GDPR customer deletion (30 days)
- `SHOP_REDACT` - GDPR shop deletion (48 hours)

### OAuth Scopes
```go
scopes := []string{
    "read_products",
    "write_products",
    "read_orders",
    "read_customers",
}
```

### API Versions
- Use versioned API: `2024-01` or `2024-04`
- Check supported versions: https://shopify.dev/docs/api/usage/versioning

### Rate Limits
- REST API: 2 requests/second (bucket-based)
- GraphQL: Points-based (max 1000 points/second)
- Webhook delivery: 1 per second per endpoint

---

## Resources

- [Shopify App Development Docs](https://shopify.dev/docs/apps)
- [Admin API Reference](https://shopify.dev/docs/api/admin-graphql)
- [App Bridge Documentation](https://shopify.dev/docs/api/app-bridge)
- [Webhook Topics](https://shopify.dev/docs/api/webhooks)
- [GDPR Compliance](https://shopify.dev/docs/apps/store/data-protection)
