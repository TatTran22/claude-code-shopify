---
description: Incrementally fix Go compilation and build errors. Invoke build-error-resolver agent to systematically resolve issues one at a time.
---

# Build and Fix

Incrementally fix Go compilation and build errors:

## Build Process

1. **Run build commands**:
   ```bash
   # Compile all packages
   go build ./...

   # Run go vet (static analysis)
   go vet ./...

   # Run golangci-lint (comprehensive linting)
   golangci-lint run

   # Run gofmt check
   gofmt -l .

   # Frontend build (if applicable)
   cd frontend && npm run build
   ```

2. **Parse error output**:
   - Group by file
   - Sort by severity (compilation errors → vet warnings → lint issues)
   - Prioritize: syntax errors → type errors → unused imports → style issues

3. **For each error**:
   - Show error context (5 lines before/after)
   - Explain the issue
   - Propose fix following Go conventions
   - Apply fix
   - Re-run build
   - Verify error resolved

4. **Stop if**:
   - Fix introduces new errors
   - Same error persists after 3 attempts
   - User requests pause
   - Build is clean (all checks pass)

5. **Show summary**:
   - Errors fixed
   - Errors remaining
   - New errors introduced
   - Commands to verify: `go build ./... && go vet ./... && golangci-lint run`

## Common Go Build Errors Fixed

**Compilation Errors:**
- Missing return statements
- Undefined variables or functions
- Type mismatches
- Import cycle errors
- Unused imports (automatic removal)

**Go Vet Warnings:**
- Printf format string mismatches
- Composite literal issues
- Unreachable code
- Possible nil pointer dereferences

**Golangci-lint Issues:**
- Error handling (errcheck)
- Unused variables
- Code complexity (gocyclo)
- Security issues (gosec)
- Inefficient assignments

**Formatting Issues:**
- Not gofmt'd (auto-fix with `gofmt -w .`)
- Not goimports'd (auto-fix with `goimports -w .`)

## Example Workflow

```
User: /build-fix

Agent (build-error-resolver):
Running build checks...

$ go build ./...
# github.com/example/project/internal/market
internal/market/liquidity.go:15:2: undefined: calculateScore

Found 1 error. Fixing...

File: internal/market/liquidity.go:15
Error: undefined: calculateScore

Fix: Add missing function or import

Applied fix:
+ func calculateScore(data MarketData) int {
+     return 0 // TODO: implement
+ }

Re-running build...
$ go build ./...

✅ Build successful!

Summary:
- Errors fixed: 1
- Errors remaining: 0
- Build status: CLEAN
```

## Safety First

**Fix one error at a time** to ensure:
- Each fix is verified
- No cascading failures
- Rollback is possible if needed
- Clear cause-and-effect relationship

## Integration with Other Commands

- Use `/plan` before major refactoring
- Use `/tdd` to write tests alongside fixes
- Use `/code-review` after fixes complete
- Use `/security-review` if security issues found

## Related Agents

This command invokes the `build-error-resolver` agent located at:
`~/.claude/agents/build-error-resolver.md`

## Go-Specific Build Commands

```bash
# Full build verification
go build ./...
go vet ./...
go test ./...
golangci-lint run
gofmt -l . | grep . && echo "Files not formatted" || echo "✓ All files formatted"

# Fix formatting automatically
gofmt -w .
goimports -w .

# Run with race detector
go test -race ./...

# Check for dependency issues
go mod tidy
go mod verify

# Build specific binary
go build -o bin/api cmd/api/main.go
```
