---
skill_id: queue-worker-patterns
name: RabbitMQ Queue & Worker Patterns
description: Comprehensive patterns for building reliable message queues and workers with RabbitMQ in Go - connection management, producers, consumers, DLQ, retries, and graceful shutdown
category: integration
tags: [rabbitmq, queue, worker, message-broker, async, dlq, retry]
applies_to: [go]
auto_trigger: ["rabbitmq", "queue", "worker", "consumer", "producer", "amqp"]
---

# RabbitMQ Queue & Worker Patterns

Production-ready patterns for building message queues and workers with RabbitMQ in Go. Covers connection management, reliable publishing, consumer patterns, dead letter queues, and graceful shutdown.

## Core Concepts

### RabbitMQ Architecture

```
Producer                Exchange              Queue               Consumer
  │                        │                    │                    │
  │  1. Publish message    │                    │                    │
  │ ──────────────────────>│                    │                    │
  │                        │  2. Route by key   │                    │
  │                        │ ──────────────────>│                    │
  │                        │                    │  3. Consume        │
  │                        │                    │ ──────────────────>│
  │                        │                    │                    │
  │                        │                    │  4. ACK/NACK       │
  │                        │                    │<───────────────────│
  │                        │                    │                    │
  │                        │  If NACK + retry   │                    │
  │                        │  exceeded          │                    │
  │                        │  ─────────────────>│ Dead Letter Queue  │
```

### Exchange Types

- **Direct**: Routes to queue with exact routing key match
- **Topic**: Routes to queues with pattern matching (e.g., `orders.*.created`)
- **Fanout**: Routes to all bound queues (broadcast)
- **Headers**: Routes based on message headers

---

## 1. Connection Management

### 1.1 Connection Pool with Auto-Reconnect

```go
// internal/queue/connection.go
package queue

import (
    "context"
    "fmt"
    "sync"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

type ConnectionPool struct {
    url        string
    conn       *amqp.Connection
    channels   chan *amqp.Channel
    maxChannels int
    mu         sync.RWMutex
    closed     bool
}

func NewConnectionPool(url string, maxChannels int) (*ConnectionPool, error) {
    pool := &ConnectionPool{
        url:         url,
        maxChannels: maxChannels,
        channels:    make(chan *amqp.Channel, maxChannels),
    }

    if err := pool.connect(); err != nil {
        return nil, err
    }

    // Monitor connection and reconnect if needed
    go pool.monitorConnection()

    return pool, nil
}

func (p *ConnectionPool) connect() error {
    p.mu.Lock()
    defer p.mu.Unlock()

    conn, err := amqp.Dial(p.url)
    if err != nil {
        return fmt.Errorf("failed to connect to RabbitMQ: %w", err)
    }

    p.conn = conn

    // Pre-create channels
    for i := 0; i < p.maxChannels; i++ {
        ch, err := conn.Channel()
        if err != nil {
            return fmt.Errorf("failed to create channel: %w", err)
        }
        p.channels <- ch
    }

    return nil
}

func (p *ConnectionPool) monitorConnection() {
    for {
        if p.closed {
            return
        }

        p.mu.RLock()
        conn := p.conn
        p.mu.RUnlock()

        // Wait for connection to close
        errChan := conn.NotifyClose(make(chan *amqp.Error))
        err := <-errChan

        if err != nil {
            log.Printf("RabbitMQ connection closed: %v. Reconnecting...", err)

            // Exponential backoff reconnection
            for attempt := 0; ; attempt++ {
                backoff := time.Duration(1<<uint(attempt)) * time.Second
                if backoff > 30*time.Second {
                    backoff = 30 * time.Second
                }
                time.Sleep(backoff)

                if err := p.connect(); err != nil {
                    log.Printf("Reconnection attempt %d failed: %v", attempt+1, err)
                    continue
                }

                log.Println("Successfully reconnected to RabbitMQ")
                break
            }
        }
    }
}

func (p *ConnectionPool) GetChannel() (*amqp.Channel, error) {
    p.mu.RLock()
    defer p.mu.RUnlock()

    if p.closed {
        return nil, fmt.Errorf("connection pool is closed")
    }

    select {
    case ch := <-p.channels:
        return ch, nil
    case <-time.After(5 * time.Second):
        return nil, fmt.Errorf("timeout waiting for channel")
    }
}

func (p *ConnectionPool) ReturnChannel(ch *amqp.Channel) {
    if ch == nil {
        return
    }

    select {
    case p.channels <- ch:
    default:
        ch.Close() // Channel pool full, close the channel
    }
}

func (p *ConnectionPool) Close() error {
    p.mu.Lock()
    defer p.mu.Unlock()

    if p.closed {
        return nil
    }

    p.closed = true
    close(p.channels)

    // Close all channels
    for ch := range p.channels {
        ch.Close()
    }

    return p.conn.Close()
}
```

---

## 2. Producer Patterns

### 2.1 Fire-and-Forget Producer

```go
// internal/queue/producer.go
package queue

import (
    "context"
    "encoding/json"
    "fmt"

    amqp "github.com/rabbitmq/amqp091-go"
)

type Producer struct {
    pool     *ConnectionPool
    exchange string
}

func NewProducer(pool *ConnectionPool, exchange string) *Producer {
    return &Producer{
        pool:     pool,
        exchange: exchange,
    }
}

// Publish publishes a message without confirmation
func (p *Producer) Publish(ctx context.Context, routingKey string, body interface{}) error {
    ch, err := p.pool.GetChannel()
    if err != nil {
        return fmt.Errorf("failed to get channel: %w", err)
    }
    defer p.pool.ReturnChannel(ch)

    // Declare exchange (idempotent)
    if err := ch.ExchangeDeclare(
        p.exchange, // name
        "topic",    // type
        true,       // durable
        false,      // auto-deleted
        false,      // internal
        false,      // no-wait
        nil,        // arguments
    ); err != nil {
        return fmt.Errorf("failed to declare exchange: %w", err)
    }

    // Marshal body to JSON
    bodyBytes, err := json.Marshal(body)
    if err != nil {
        return fmt.Errorf("failed to marshal body: %w", err)
    }

    // Publish message
    err = ch.PublishWithContext(
        ctx,
        p.exchange,  // exchange
        routingKey,  // routing key
        false,       // mandatory
        false,       // immediate
        amqp.Publishing{
            DeliveryMode: amqp.Persistent, // 2 = persistent
            ContentType:  "application/json",
            Body:         bodyBytes,
            Timestamp:    time.Now(),
        },
    )

    if err != nil {
        return fmt.Errorf("failed to publish message: %w", err)
    }

    return nil
}
```

### 2.2 Producer with Confirmation

```go
// internal/queue/confirmed_producer.go
package queue

import (
    "context"
    "fmt"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

type ConfirmedProducer struct {
    pool     *ConnectionPool
    exchange string
}

func NewConfirmedProducer(pool *ConnectionPool, exchange string) *ConfirmedProducer {
    return &ConfirmedProducer{
        pool:     pool,
        exchange: exchange,
    }
}

// PublishWithConfirmation publishes a message and waits for broker confirmation
func (p *ConfirmedProducer) PublishWithConfirmation(ctx context.Context, routingKey string, body interface{}) error {
    ch, err := p.pool.GetChannel()
    if err != nil {
        return fmt.Errorf("failed to get channel: %w", err)
    }
    defer p.pool.ReturnChannel(ch)

    // Enable publisher confirms
    if err := ch.Confirm(false); err != nil {
        return fmt.Errorf("failed to enable confirms: %w", err)
    }

    // Setup confirmation listener
    confirms := ch.NotifyPublish(make(chan amqp.Confirmation, 1))

    // Declare exchange
    if err := ch.ExchangeDeclare(
        p.exchange,
        "topic",
        true,
        false,
        false,
        false,
        nil,
    ); err != nil {
        return fmt.Errorf("failed to declare exchange: %w", err)
    }

    // Marshal body
    bodyBytes, err := json.Marshal(body)
    if err != nil {
        return fmt.Errorf("failed to marshal body: %w", err)
    }

    // Publish message
    err = ch.PublishWithContext(
        ctx,
        p.exchange,
        routingKey,
        false,
        false,
        amqp.Publishing{
            DeliveryMode: amqp.Persistent,
            ContentType:  "application/json",
            Body:         bodyBytes,
            Timestamp:    time.Now(),
        },
    )

    if err != nil {
        return fmt.Errorf("failed to publish message: %w", err)
    }

    // Wait for confirmation
    select {
    case confirm := <-confirms:
        if !confirm.Ack {
            return fmt.Errorf("message was not acknowledged by broker")
        }
        return nil
    case <-time.After(5 * time.Second):
        return fmt.Errorf("timeout waiting for confirmation")
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

---

## 3. Consumer Patterns

### 3.1 Basic Consumer with Manual ACK

```go
// internal/queue/consumer.go
package queue

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    amqp "github.com/rabbitmq/amqp091-go"
)

type MessageHandler func(ctx context.Context, body []byte) error

type Consumer struct {
    pool       *ConnectionPool
    queue      string
    exchange   string
    routingKey string
    handler    MessageHandler
}

func NewConsumer(pool *ConnectionPool, queue, exchange, routingKey string, handler MessageHandler) *Consumer {
    return &Consumer{
        pool:       pool,
        queue:      queue,
        exchange:   exchange,
        routingKey: routingKey,
        handler:    handler,
    }
}

func (c *Consumer) Start(ctx context.Context) error {
    ch, err := c.pool.GetChannel()
    if err != nil {
        return fmt.Errorf("failed to get channel: %w", err)
    }
    defer c.pool.ReturnChannel(ch)

    // Declare exchange
    if err := ch.ExchangeDeclare(
        c.exchange,
        "topic",
        true,
        false,
        false,
        false,
        nil,
    ); err != nil {
        return fmt.Errorf("failed to declare exchange: %w", err)
    }

    // Declare queue with DLX (Dead Letter Exchange)
    args := amqp.Table{
        "x-dead-letter-exchange":    c.exchange + ".dlx",
        "x-dead-letter-routing-key": c.routingKey + ".dead",
    }

    if _, err := ch.QueueDeclare(
        c.queue, // name
        true,    // durable
        false,   // auto-delete
        false,   // exclusive
        false,   // no-wait
        args,    // arguments
    ); err != nil {
        return fmt.Errorf("failed to declare queue: %w", err)
    }

    // Bind queue to exchange
    if err := ch.QueueBind(
        c.queue,
        c.routingKey,
        c.exchange,
        false,
        nil,
    ); err != nil {
        return fmt.Errorf("failed to bind queue: %w", err)
    }

    // Set prefetch count (QoS)
    if err := ch.Qos(
        10,    // prefetch count
        0,     // prefetch size
        false, // global
    ); err != nil {
        return fmt.Errorf("failed to set QoS: %w", err)
    }

    // Start consuming
    msgs, err := ch.Consume(
        c.queue,
        "",    // consumer tag
        false, // auto-ack (MUST BE FALSE for manual ack)
        false, // exclusive
        false, // no-local
        false, // no-wait
        nil,   // args
    )

    if err != nil {
        return fmt.Errorf("failed to start consuming: %w", err)
    }

    log.Printf("Consumer started for queue: %s", c.queue)

    // Process messages
    for {
        select {
        case <-ctx.Done():
            log.Println("Consumer shutting down...")
            return ctx.Err()
        case msg, ok := <-msgs:
            if !ok {
                return fmt.Errorf("message channel closed")
            }

            // Process message with timeout
            msgCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
            err := c.handler(msgCtx, msg.Body)
            cancel()

            if err != nil {
                log.Printf("Failed to process message: %v", err)

                // NACK and requeue (will go to DLX after max retries)
                if err := msg.Nack(false, false); err != nil {
                    log.Printf("Failed to NACK message: %v", err)
                }
            } else {
                // ACK message
                if err := msg.Ack(false); err != nil {
                    log.Printf("Failed to ACK message: %v", err)
                }
            }
        }
    }
}
```

### 3.2 Worker Pool Consumer

```go
// internal/queue/worker_pool_consumer.go
package queue

import (
    "context"
    "log"
    "sync"

    amqp "github.com/rabbitmq/amqp091-go"
)

type WorkerPoolConsumer struct {
    pool       *ConnectionPool
    queue      string
    exchange   string
    routingKey string
    handler    MessageHandler
    numWorkers int
}

func NewWorkerPoolConsumer(pool *ConnectionPool, queue, exchange, routingKey string, handler MessageHandler, numWorkers int) *WorkerPoolConsumer {
    return &WorkerPoolConsumer{
        pool:       pool,
        queue:      queue,
        exchange:   exchange,
        routingKey: routingKey,
        handler:    handler,
        numWorkers: numWorkers,
    }
}

func (c *WorkerPoolConsumer) Start(ctx context.Context) error {
    ch, err := c.pool.GetChannel()
    if err != nil {
        return fmt.Errorf("failed to get channel: %w", err)
    }
    defer c.pool.ReturnChannel(ch)

    // Declare exchange, queue, binding (same as basic consumer)
    if err := c.setup(ch); err != nil {
        return err
    }

    // Set prefetch count based on worker pool size
    if err := ch.Qos(c.numWorkers, 0, false); err != nil {
        return fmt.Errorf("failed to set QoS: %w", err)
    }

    // Start consuming
    msgs, err := ch.Consume(
        c.queue,
        "",
        false, // manual ACK
        false,
        false,
        false,
        nil,
    )

    if err != nil {
        return fmt.Errorf("failed to start consuming: %w", err)
    }

    log.Printf("Worker pool consumer started with %d workers for queue: %s", c.numWorkers, c.queue)

    // Start worker pool
    var wg sync.WaitGroup
    for i := 0; i < c.numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            c.worker(ctx, workerID, msgs)
        }(i)
    }

    // Wait for context cancellation
    <-ctx.Done()
    log.Println("Worker pool consumer shutting down...")

    // Wait for all workers to finish
    wg.Wait()
    return nil
}

func (c *WorkerPoolConsumer) worker(ctx context.Context, workerID int, msgs <-chan amqp.Delivery) {
    log.Printf("Worker %d started", workerID)

    for {
        select {
        case <-ctx.Done():
            log.Printf("Worker %d shutting down", workerID)
            return
        case msg, ok := <-msgs:
            if !ok {
                log.Printf("Worker %d: message channel closed", workerID)
                return
            }

            log.Printf("Worker %d processing message", workerID)

            // Process message with timeout
            msgCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
            err := c.handler(msgCtx, msg.Body)
            cancel()

            if err != nil {
                log.Printf("Worker %d: failed to process message: %v", workerID, err)
                msg.Nack(false, false)
            } else {
                msg.Ack(false)
            }
        }
    }
}

func (c *WorkerPoolConsumer) setup(ch *amqp.Channel) error {
    // Declare exchange
    if err := ch.ExchangeDeclare(
        c.exchange,
        "topic",
        true,
        false,
        false,
        false,
        nil,
    ); err != nil {
        return fmt.Errorf("failed to declare exchange: %w", err)
    }

    // Declare queue with DLX
    args := amqp.Table{
        "x-dead-letter-exchange":    c.exchange + ".dlx",
        "x-dead-letter-routing-key": c.routingKey + ".dead",
    }

    if _, err := ch.QueueDeclare(
        c.queue,
        true,
        false,
        false,
        false,
        args,
    ); err != nil {
        return fmt.Errorf("failed to declare queue: %w", err)
    }

    // Bind queue
    if err := ch.QueueBind(
        c.queue,
        c.routingKey,
        c.exchange,
        false,
        nil,
    ); err != nil {
        return fmt.Errorf("failed to bind queue: %w", err)
    }

    return nil
}
```

---

## 4. Dead Letter Queue (DLQ) Pattern

### 4.1 DLQ Setup

```go
// internal/queue/dlq.go
package queue

import (
    "fmt"

    amqp "github.com/rabbitmq/amqp091-go"
)

// SetupDLQ sets up a dead letter exchange and queue
func SetupDLQ(ch *amqp.Channel, exchange, queue string) error {
    dlxName := exchange + ".dlx"
    dlqName := queue + ".dead"

    // Declare DLX
    if err := ch.ExchangeDeclare(
        dlxName,
        "topic",
        true,
        false,
        false,
        false,
        nil,
    ); err != nil {
        return fmt.Errorf("failed to declare DLX: %w", err)
    }

    // Declare DLQ
    if _, err := ch.QueueDeclare(
        dlqName,
        true,
        false,
        false,
        false,
        nil,
    ); err != nil {
        return fmt.Errorf("failed to declare DLQ: %w", err)
    }

    // Bind DLQ to DLX
    if err := ch.QueueBind(
        dlqName,
        "#", // catch all
        dlxName,
        false,
        nil,
    ); err != nil {
        return fmt.Errorf("failed to bind DLQ: %w", err)
    }

    return nil
}
```

### 4.2 DLQ Consumer (for Manual Inspection)

```go
// internal/queue/dlq_consumer.go
package queue

import (
    "context"
    "log"

    amqp "github.com/rabbitmq/amqp091-go"
)

type DLQConsumer struct {
    pool  *ConnectionPool
    queue string
}

func NewDLQConsumer(pool *ConnectionPool, queue string) *DLQConsumer {
    return &DLQConsumer{
        pool:  pool,
        queue: queue + ".dead",
    }
}

func (c *DLQConsumer) Start(ctx context.Context) error {
    ch, err := c.pool.GetChannel()
    if err != nil {
        return fmt.Errorf("failed to get channel: %w", err)
    }
    defer c.pool.ReturnChannel(ch)

    msgs, err := ch.Consume(
        c.queue,
        "",
        false, // manual ACK
        false,
        false,
        false,
        nil,
    )

    if err != nil {
        return fmt.Errorf("failed to start DLQ consumer: %w", err)
    }

    log.Printf("DLQ consumer started for queue: %s", c.queue)

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case msg, ok := <-msgs:
            if !ok {
                return fmt.Errorf("DLQ channel closed")
            }

            // Log dead letter for investigation
            log.Printf("Dead letter received: %s", string(msg.Body))
            log.Printf("Death reason: %v", msg.Headers["x-death"])

            // ACK to remove from DLQ (or move to another system for analysis)
            msg.Ack(false)
        }
    }
}
```

---

## 5. Retry Pattern with Exponential Backoff

### 5.1 Retry Queue with Delay

```go
// internal/queue/retry.go
package queue

import (
    "fmt"

    amqp "github.com/rabbitmq/amqp091-go"
)

// SetupRetryQueue creates a retry queue with delay
func SetupRetryQueue(ch *amqp.Channel, exchange, queue string, delayMs int) error {
    retryQueueName := fmt.Sprintf("%s.retry.%dms", queue, delayMs)

    // Declare retry queue with TTL and DLX back to main exchange
    args := amqp.Table{
        "x-message-ttl":             int32(delayMs),
        "x-dead-letter-exchange":    exchange,
        "x-dead-letter-routing-key": queue,
    }

    if _, err := ch.QueueDeclare(
        retryQueueName,
        true,
        false,
        false,
        false,
        args,
    ); err != nil {
        return fmt.Errorf("failed to declare retry queue: %w", err)
    }

    return nil
}

// PublishToRetry publishes a message to retry queue
func PublishToRetry(ch *amqp.Channel, exchange, queue string, body []byte, retryCount int) error {
    // Calculate delay based on retry count (exponential backoff)
    delays := []int{1000, 5000, 15000, 60000} // 1s, 5s, 15s, 60s
    delayIndex := retryCount
    if delayIndex >= len(delays) {
        // Max retries exceeded, send to DLQ
        return fmt.Errorf("max retries exceeded")
    }

    delay := delays[delayIndex]
    retryQueueName := fmt.Sprintf("%s.retry.%dms", queue, delay)

    // Ensure retry queue exists
    if err := SetupRetryQueue(ch, exchange, queue, delay); err != nil {
        return err
    }

    // Publish to retry queue
    err := ch.Publish(
        "",              // exchange (use default for direct queue publish)
        retryQueueName,  // routing key
        false,
        false,
        amqp.Publishing{
            DeliveryMode: amqp.Persistent,
            ContentType:  "application/json",
            Body:         body,
            Headers: amqp.Table{
                "x-retry-count": retryCount + 1,
            },
        },
    )

    return err
}
```

### 5.2 Consumer with Retry Logic

```go
// internal/queue/retry_consumer.go
package queue

import (
    "context"
    "log"

    amqp "github.com/rabbitmq/amqp091-go"
)

type RetryConsumer struct {
    pool       *ConnectionPool
    queue      string
    exchange   string
    routingKey string
    handler    MessageHandler
    maxRetries int
}

func (c *RetryConsumer) Start(ctx context.Context) error {
    ch, err := c.pool.GetChannel()
    if err != nil {
        return fmt.Errorf("failed to get channel: %w", err)
    }
    defer c.pool.ReturnChannel(ch)

    // Setup queue, exchange, DLQ
    if err := c.setup(ch); err != nil {
        return err
    }

    msgs, err := ch.Consume(
        c.queue,
        "",
        false,
        false,
        false,
        false,
        nil,
    )

    if err != nil {
        return fmt.Errorf("failed to start consuming: %w", err)
    }

    log.Printf("Retry consumer started for queue: %s", c.queue)

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case msg, ok := <-msgs:
            if !ok {
                return fmt.Errorf("message channel closed")
            }

            // Get retry count from headers
            retryCount := 0
            if count, ok := msg.Headers["x-retry-count"].(int32); ok {
                retryCount = int(count)
            }

            // Process message
            err := c.handler(ctx, msg.Body)

            if err != nil {
                log.Printf("Failed to process message (retry %d): %v", retryCount, err)

                if retryCount >= c.maxRetries {
                    log.Printf("Max retries exceeded, sending to DLQ")
                    msg.Nack(false, false) // Send to DLQ
                } else {
                    // Publish to retry queue
                    if err := PublishToRetry(ch, c.exchange, c.queue, msg.Body, retryCount); err != nil {
                        log.Printf("Failed to publish to retry queue: %v", err)
                        msg.Nack(false, false)
                    } else {
                        msg.Ack(false)
                    }
                }
            } else {
                msg.Ack(false)
            }
        }
    }
}

func (c *RetryConsumer) setup(ch *amqp.Channel) error {
    // Declare exchange
    if err := ch.ExchangeDeclare(
        c.exchange,
        "topic",
        true,
        false,
        false,
        false,
        nil,
    ); err != nil {
        return err
    }

    // Setup DLQ
    if err := SetupDLQ(ch, c.exchange, c.queue); err != nil {
        return err
    }

    // Declare main queue with DLX
    args := amqp.Table{
        "x-dead-letter-exchange":    c.exchange + ".dlx",
        "x-dead-letter-routing-key": c.routingKey + ".dead",
    }

    if _, err := ch.QueueDeclare(
        c.queue,
        true,
        false,
        false,
        false,
        args,
    ); err != nil {
        return err
    }

    // Bind queue
    if err := ch.QueueBind(
        c.queue,
        c.routingKey,
        c.exchange,
        false,
        nil,
    ); err != nil {
        return err
    }

    return nil
}
```

---

## 6. Graceful Shutdown

```go
// cmd/worker/main.go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "yourapp/internal/queue"
)

func main() {
    // Create connection pool
    pool, err := queue.NewConnectionPool(os.Getenv("RABBITMQ_URL"), 10)
    if err != nil {
        log.Fatalf("Failed to create connection pool: %v", err)
    }
    defer pool.Close()

    // Create consumer
    handler := func(ctx context.Context, body []byte) error {
        // Process message
        log.Printf("Processing message: %s", string(body))
        return nil
    }

    consumer := queue.NewWorkerPoolConsumer(
        pool,
        "my-queue",
        "my-exchange",
        "my.routing.key",
        handler,
        5, // 5 workers
    )

    // Create context with cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Start consumer in goroutine
    go func() {
        if err := consumer.Start(ctx); err != nil {
            log.Printf("Consumer error: %v", err)
        }
    }()

    // Wait for interrupt signal
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    <-sigChan

    log.Println("Received shutdown signal")

    // Cancel context to stop consumer
    cancel()

    // Give workers time to finish current messages
    time.Sleep(5 * time.Second)

    log.Println("Shutdown complete")
}
```

---

## 7. Monitoring & Health Checks

### 7.1 Health Check Endpoint

```go
// internal/queue/health.go
package queue

import (
    "context"
    "encoding/json"
    "net/http"
    "time"
)

type HealthChecker struct {
    pool *ConnectionPool
}

func NewHealthChecker(pool *ConnectionPool) *HealthChecker {
    return &HealthChecker{pool: pool}
}

func (h *HealthChecker) HandleHealth(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    // Try to get a channel (tests connection)
    ch, err := h.pool.GetChannel()
    if err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "unhealthy",
            "error":  err.Error(),
        })
        return
    }
    h.pool.ReturnChannel(ch)

    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{
        "status": "healthy",
    })
}
```

### 7.2 Queue Metrics

```go
// internal/queue/metrics.go
package queue

import (
    "fmt"

    amqp "github.com/rabbitmq/amqp091-go"
)

type QueueStats struct {
    Messages      int
    Consumers     int
    MessagesReady int
    MessagesUnack int
}

func GetQueueStats(ch *amqp.Channel, queueName string) (*QueueStats, error) {
    queue, err := ch.QueueInspect(queueName)
    if err != nil {
        return nil, fmt.Errorf("failed to inspect queue: %w", err)
    }

    return &QueueStats{
        Messages:      queue.Messages,
        Consumers:     queue.Consumers,
        MessagesReady: queue.Messages,
        MessagesUnack: 0, // Not available in QueueInspect
    }, nil
}
```

---

## 8. Best Practices

### ✅ DO

```go
// ✅ Always use manual ACK for reliability
msgs, err := ch.Consume(
    queue,
    "",
    false, // auto-ack = false
    false,
    false,
    false,
    nil,
)

// ✅ Set prefetch count (QoS) to limit concurrent messages
ch.Qos(10, 0, false)

// ✅ Use persistent messages
amqp.Publishing{
    DeliveryMode: amqp.Persistent, // 2 = persistent
    Body:         body,
}

// ✅ Use durable queues and exchanges
ch.QueueDeclare(
    queue,
    true, // durable
    false,
    false,
    false,
    nil,
)

// ✅ Setup DLQ for failed messages
args := amqp.Table{
    "x-dead-letter-exchange":    "my-exchange.dlx",
    "x-dead-letter-routing-key": "my-key.dead",
}

// ✅ Implement reconnection logic
go pool.monitorConnection()

// ✅ Use context for cancellation
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()

// ✅ Graceful shutdown with signal handling
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
<-sigChan
cancel() // Cancel context
time.Sleep(5 * time.Second) // Wait for workers

// ✅ Use worker pools for parallel processing
consumer := queue.NewWorkerPoolConsumer(pool, queue, exchange, key, handler, 5)
```

### ❌ DON'T

```go
// ❌ Don't use auto-ack (messages can be lost)
ch.Consume(queue, "", true, false, false, false, nil)

// ❌ Don't ignore connection errors
conn, _ := amqp.Dial(url) // WRONG

// ❌ Don't process messages synchronously without timeout
err := handler(ctx, msg.Body) // No timeout

// ❌ Don't forget to ACK/NACK messages
// WRONG: msg is processed but never ACK'd or NACK'd

// ❌ Don't publish without error handling
ch.Publish(exchange, key, false, false, msg) // Ignoring error

// ❌ Don't use non-durable queues in production
ch.QueueDeclare(queue, false, false, false, false, nil) // WRONG

// ❌ Don't skip DLQ setup
// Messages with repeated failures have nowhere to go

// ❌ Don't block shutdown
// WRONG: No graceful shutdown, workers killed mid-processing
```

---

## 9. Complete Example

### 9.1 Order Processing System

```go
// cmd/order-worker/main.go
package main

import (
    "context"
    "encoding/json"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "yourapp/internal/queue"
)

type Order struct {
    ID       string  `json:"id"`
    Total    float64 `json:"total"`
    ShopID   string  `json:"shop_id"`
}

func main() {
    // Create connection pool
    pool, err := queue.NewConnectionPool(os.Getenv("RABBITMQ_URL"), 20)
    if err != nil {
        log.Fatalf("Failed to create connection pool: %v", err)
    }
    defer pool.Close()

    // Define message handler
    handler := func(ctx context.Context, body []byte) error {
        var order Order
        if err := json.Unmarshal(body, &order); err != nil {
            return fmt.Errorf("failed to unmarshal order: %w", err)
        }

        log.Printf("Processing order %s for shop %s (total: $%.2f)", order.ID, order.ShopID, order.Total)

        // Simulate processing
        time.Sleep(2 * time.Second)

        // Process order logic here...

        return nil
    }

    // Create retry consumer with worker pool
    consumer := queue.NewWorkerPoolConsumer(
        pool,
        "orders.process",
        "shopify",
        "orders.created",
        handler,
        10, // 10 workers
    )

    // Setup graceful shutdown
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Start consumer
    go func() {
        if err := consumer.Start(ctx); err != nil {
            log.Printf("Consumer stopped: %v", err)
        }
    }()

    // Wait for interrupt
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    <-sigChan

    log.Println("Shutting down...")
    cancel()
    time.Sleep(5 * time.Second)
    log.Println("Shutdown complete")
}
```

---

## Quick Reference

### Connection URL Format
```
amqp://user:password@localhost:5672/vhost
```

### Exchange Types
- **direct**: Exact routing key match
- **topic**: Pattern matching (e.g., `orders.*.created`)
- **fanout**: Broadcast to all queues
- **headers**: Route by message headers

### QoS Settings
```go
ch.Qos(
    10,    // prefetch count (max unacked messages)
    0,     // prefetch size (bytes)
    false, // global (false = per consumer)
)
```

### Common Retry Delays
- 1st retry: 1 second
- 2nd retry: 5 seconds
- 3rd retry: 15 seconds
- 4th retry: 60 seconds
- 5+ retries: DLQ

---

## Resources

- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [amqp091-go Library](https://github.com/rabbitmq/amqp091-go)
- [RabbitMQ Tutorials (Go)](https://www.rabbitmq.com/getstarted.html)
- [Reliability Guide](https://www.rabbitmq.com/reliability.html)
