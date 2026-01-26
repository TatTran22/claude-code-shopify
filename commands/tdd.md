---
description: Enforce test-driven development workflow. Scaffold interfaces, generate tests FIRST, then implement minimal code to pass. Ensure 80%+ coverage.
---

# TDD Command

This command invokes the **tdd-guide** agent to enforce test-driven development methodology.

## What This Command Does

1. **Scaffold Interfaces** - Define types/interfaces first
2. **Generate Tests First** - Write failing tests (RED)
3. **Implement Minimal Code** - Write just enough to pass (GREEN)
4. **Refactor** - Improve code while keeping tests green (REFACTOR)
5. **Verify Coverage** - Ensure 80%+ test coverage

## When to Use

Use `/tdd` when:
- Implementing new features
- Adding new functions/components
- Fixing bugs (write test that reproduces bug first)
- Refactoring existing code
- Building critical business logic

## How It Works

The tdd-guide agent will:

1. **Define interfaces** for inputs/outputs
2. **Write tests that will FAIL** (because code doesn't exist yet)
3. **Run tests** and verify they fail for the right reason
4. **Write minimal implementation** to make tests pass
5. **Run tests** and verify they pass
6. **Refactor** code while keeping tests green
7. **Check coverage** and add more tests if below 80%

## TDD Cycle

```
RED → GREEN → REFACTOR → REPEAT

RED:      Write a failing test
GREEN:    Write minimal code to pass
REFACTOR: Improve code, keep tests passing
REPEAT:   Next feature/scenario
```

## Example Usage

```
User: /tdd I need a function to calculate market liquidity score

Agent (tdd-guide):
# TDD Session: Market Liquidity Score Calculator

## Step 1: Define Interface (SCAFFOLD)

```go
// internal/market/liquidity.go
package market

import "time"

type MarketData struct {
	TotalVolume   float64
	BidAskSpread  float64
	ActiveTraders int
	LastTradeTime time.Time
}

func CalculateLiquidityScore(market MarketData) int {
	// TODO: Implementation
	panic("not implemented")
}
```

## Step 2: Write Failing Test (RED)

```go
// internal/market/liquidity_test.go
package market

import (
	"testing"
	"time"

	"github.com/stretchr/testify/assert"
)

func TestCalculateLiquidityScore(t *testing.T) {
	tests := []struct {
		name   string
		market MarketData
		want   int
	}{
		{
			name: "high score for liquid market",
			market: MarketData{
				TotalVolume:   100000,
				BidAskSpread:  0.01,
				ActiveTraders: 500,
				LastTradeTime: time.Now(),
			},
			want: 85, // Expected high score
		},
		{
			name: "low score for illiquid market",
			market: MarketData{
				TotalVolume:   100,
				BidAskSpread:  0.5,
				ActiveTraders: 2,
				LastTradeTime: time.Now().Add(-24 * time.Hour),
			},
			want: 20, // Expected low score
		},
		{
			name: "zero score for no volume",
			market: MarketData{
				TotalVolume:   0,
				BidAskSpread:  0,
				ActiveTraders: 0,
				LastTradeTime: time.Now(),
			},
			want: 0,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := CalculateLiquidityScore(tt.market)
			assert.InDelta(t, tt.want, got, 10, "score should be within ±10 of expected")
			assert.GreaterOrEqual(t, got, 0, "score must be >= 0")
			assert.LessOrEqual(t, got, 100, "score must be <= 100")
		})
	}
}
```

## Step 3: Run Tests - Verify FAIL

```bash
go test ./internal/market -v

=== RUN   TestCalculateLiquidityScore
=== RUN   TestCalculateLiquidityScore/high_score_for_liquid_market
panic: not implemented

--- FAIL: TestCalculateLiquidityScore (0.00s)
    --- FAIL: TestCalculateLiquidityScore/high_score_for_liquid_market (0.00s)

FAIL    github.com/example/project/internal/market    0.002s
```

✅ Tests fail as expected. Ready to implement.

## Step 4: Implement Minimal Code (GREEN)

```go
// internal/market/liquidity.go
package market

import (
	"math"
	"time"
)

func CalculateLiquidityScore(market MarketData) int {
	// Handle zero volume edge case
	if market.TotalVolume == 0 {
		return 0
	}

	// Calculate component scores (0-100 scale)
	volumeScore := math.Min(market.TotalVolume/1000, 100)
	spreadScore := math.Max(100-(market.BidAskSpread*1000), 0)
	traderScore := math.Min(float64(market.ActiveTraders)/10, 100)

	// Recent activity bonus
	hoursSinceLastTrade := time.Since(market.LastTradeTime).Hours()
	recencyScore := math.Max(100-(hoursSinceLastTrade*10), 0)

	// Weighted average
	score := volumeScore*0.4 +
		spreadScore*0.3 +
		traderScore*0.2 +
		recencyScore*0.1

	// Clamp to 0-100
	return int(math.Max(0, math.Min(score, 100)))
}
```

## Step 5: Run Tests - Verify PASS

```bash
go test ./internal/market -v

=== RUN   TestCalculateLiquidityScore
=== RUN   TestCalculateLiquidityScore/high_score_for_liquid_market
=== RUN   TestCalculateLiquidityScore/low_score_for_illiquid_market
=== RUN   TestCalculateLiquidityScore/zero_score_for_no_volume
--- PASS: TestCalculateLiquidityScore (0.00s)
    --- PASS: TestCalculateLiquidityScore/high_score_for_liquid_market (0.00s)
    --- PASS: TestCalculateLiquidityScore/low_score_for_illiquid_market (0.00s)
    --- PASS: TestCalculateLiquidityScore/zero_score_for_no_volume (0.00s)
PASS
ok      github.com/example/project/internal/market    0.003s
```

✅ All tests passing!

## Step 6: Refactor (IMPROVE)

```go
// internal/market/liquidity.go - Refactored with constants and better readability
package market

import (
	"math"
	"time"
)

// Liquidity score weights
const (
	volumeWeight  = 0.4
	spreadWeight  = 0.3
	traderWeight  = 0.2
	recencyWeight = 0.1
)

// Scale factors for normalizing inputs
const (
	volumeScaleFactor  = 1000.0
	spreadScaleFactor  = 1000.0
	traderScaleFactor  = 10.0
	recencyPenaltyRate = 10.0
)

// clamp restricts a value to the given range [min, max]
func clamp(value, min, max float64) float64 {
	return math.Max(min, math.Min(value, max))
}

// CalculateLiquidityScore computes a liquidity score (0-100) for a market
// based on volume, spread, active traders, and recency of last trade.
func CalculateLiquidityScore(market MarketData) int {
	if market.TotalVolume == 0 {
		return 0
	}

	volumeScore := math.Min(market.TotalVolume/volumeScaleFactor, 100)
	spreadScore := clamp(100-(market.BidAskSpread*spreadScaleFactor), 0, 100)
	traderScore := math.Min(float64(market.ActiveTraders)/traderScaleFactor, 100)

	hoursSinceLastTrade := time.Since(market.LastTradeTime).Hours()
	recencyScore := clamp(100-(hoursSinceLastTrade*recencyPenaltyRate), 0, 100)

	weightedScore := volumeScore*volumeWeight +
		spreadScore*spreadWeight +
		traderScore*traderWeight +
		recencyScore*recencyWeight

	return int(clamp(weightedScore, 0, 100))
}
```

## Step 7: Verify Tests Still Pass

```bash
go test ./internal/market -v

=== RUN   TestCalculateLiquidityScore
=== RUN   TestCalculateLiquidityScore/high_score_for_liquid_market
=== RUN   TestCalculateLiquidityScore/low_score_for_illiquid_market
=== RUN   TestCalculateLiquidityScore/zero_score_for_no_volume
--- PASS: TestCalculateLiquidityScore (0.00s)
    --- PASS: TestCalculateLiquidityScore/high_score_for_liquid_market (0.00s)
    --- PASS: TestCalculateLiquidityScore/low_score_for_illiquid_market (0.00s)
    --- PASS: TestCalculateLiquidityScore/zero_score_for_no_volume (0.00s)
PASS
ok      github.com/example/project/internal/market    0.002s
```

✅ Refactoring complete, tests still passing!

## Step 8: Check Coverage

```bash
go test ./internal/market -cover -coverprofile=coverage.out

PASS
coverage: 100.0% of statements
ok      github.com/example/project/internal/market    0.003s

go tool cover -func=coverage.out

github.com/example/project/internal/market/liquidity.go:25:    clamp                   100.0%
github.com/example/project/internal/market/liquidity.go:30:    CalculateLiquidityScore 100.0%
total:                                                         (statements)            100.0%

Coverage: 100% ✅ (Target: 80%)
```

✅ TDD session complete!
```

## TDD Best Practices

**DO:**
- ✅ Write the test FIRST, before any implementation
- ✅ Run tests and verify they FAIL before implementing
- ✅ Write minimal code to make tests pass
- ✅ Refactor only after tests are green
- ✅ Add edge cases and error scenarios
- ✅ Aim for 80%+ coverage (100% for critical code)
- ✅ Use table-driven tests (Go standard pattern)

**DON'T:**
- ❌ Write implementation before tests
- ❌ Skip running tests after each change
- ❌ Write too much code at once
- ❌ Ignore failing tests
- ❌ Test implementation details (test behavior)
- ❌ Mock everything (prefer integration tests with test containers)

## Test Types to Include

**Unit Tests** (Function-level):
- Happy path scenarios
- Edge cases (empty, nil, max values)
- Error conditions
- Boundary values
- Use table-driven tests

**Integration Tests** (Component-level):
- API endpoints (use httptest)
- Database operations (use testcontainers-go)
- Redis caching (use miniredis or test containers)
- RabbitMQ consumers (use test containers)
- Chi handlers with middleware

**E2E Tests** (use `/e2e` command):
- Critical user flows
- Multi-step processes
- Full stack integration

## Coverage Requirements

- **80% minimum** for all code
- **100% required** for:
  - Financial calculations
  - Authentication logic
  - Security-critical code
  - Core business logic
  - Shopify webhook handlers

## Go-Specific Testing Commands

```bash
# Run all tests
go test ./...

# Run tests with verbose output
go test ./... -v

# Run tests with coverage
go test ./... -cover

# Generate coverage report
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out

# Run tests with race detector
go test ./... -race

# Run specific package tests
go test ./internal/market

# Run specific test
go test ./internal/market -run TestCalculateLiquidityScore

# Run tests matching pattern
go test ./... -run ".*Liquidity.*"

# Run tests with timeout
go test ./... -timeout 30s

# Run tests in parallel
go test ./... -parallel 4
```

## Important Notes

**MANDATORY**: Tests must be written BEFORE implementation. The TDD cycle is:

1. **RED** - Write failing test
2. **GREEN** - Implement to pass
3. **REFACTOR** - Improve code

Never skip the RED phase. Never write code before tests.

**Go Testing Best Practices:**
- Always use table-driven tests for multiple scenarios
- Use testify/assert for readable assertions
- Use testify/require for fatal assertions
- Test error paths explicitly
- Use httptest for HTTP handler testing
- Use testcontainers-go for integration tests
- Run `go test -race` to detect race conditions

## Integration with Other Commands

- Use `/plan` first to understand what to build
- Use `/tdd` to implement with tests
- Use `/build-fix` if build errors occur
- Use `/code-review` to review implementation
- Use `/test-coverage` to verify coverage

## Related Agents

This command invokes the `tdd-guide` agent located at:
`~/.claude/agents/tdd-guide.md`

And can reference the `tdd-workflow` skill at:
`~/.claude/skills/tdd-workflow/`
