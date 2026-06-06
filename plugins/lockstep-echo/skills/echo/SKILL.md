---
name: echo
description: Enforce Echo v5 handler conventions for Go, focused on error handling. Use when editing .go handler/route/middleware files or when the user mentions Echo, HandlerFunc, c.Bind, middleware, HTTPError, HTTPErrorHandler, error wrap, or graceful shutdown. Forbids manual JSON error responses, err.Error() string comparison, swallowed errors, and e.Start without graceful shutdown.
paths:
  - "**/*.go"
allowed-tools:
  - Read
  - Grep
---

This skill enforces Echo v5 conventions for Go HTTP services. Apply only when the project imports `github.com/labstack/echo/v5` (v5.0+). If the project still imports `echo/v4` or earlier, **STOP** and ask the user—v5 import path is `v5/`.

## Why Echo

Echo's selling point is its error-handling model. Handler signature `func(c *echo.Context) error` lets errors bubble naturally through the middleware chain. `*echo.HTTPError` is a typed, status-aware error that pairs cleanly with `errors.Is` / `errors.As`. A single `HTTPErrorHandler` converts every error to the wire format, so handlers stay short and uniform.

The rule: **let errors flow as errors, never as JSON written inline**. Lose this and you lose Echo's core advantage.

## Core principles

- **Handler signature**: `func(c *echo.Context) error`. Return errors. Do not write headers or JSON on the error path.
- **Use `echo.NewHTTPError(status, msg)`** for all client-facing errors. Do not call `c.JSON(http.StatusBadRequest, ...)` directly.
- **Centralize error → response in a custom `HTTPErrorHandler`**. Every handler benefits without duplication.
- **Wrap errors with `%w`** so `errors.Is` / `errors.As` still work several layers up.
- **Use sentinel errors** (`var ErrFoo = errors.New("...")`) or typed errors for business cases. Never string-compare with `err.Error()`.
- **Bind + validate every body**: `c.Bind` then `c.Validate`. Both errors propagate.
- **Use route groups for shared middleware** (auth, rate limit), not per-route duplication.
- **Graceful shutdown is mandatory**: trap SIGINT/SIGTERM and call `e.Shutdown(ctx)`.

## Server skeleton

```go
e := echo.New()
e.HTTPErrorHandler = customErrorHandler
e.Validator = &customValidator{...}
e.Use(middleware.Logger(), middleware.Recover(), middleware.RequestID())

// routes ...

go func() {
    if err := e.Start(":8080"); err != nil && !errors.Is(err, http.ErrServerClosed) {
        e.Logger.Fatal(err)
    }
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := e.Shutdown(ctx); err != nil {
    e.Logger.Fatal(err)
}
```

## Error handling pattern (the core)

### 1. Errors bubble; handlers do not write error JSON

```go
func GetUser(c *echo.Context) error {
    user, err := svc.Get(c.Param("id"))
    if err != nil {
        return err   // don't c.JSON here
    }
    return c.JSON(http.StatusOK, user)
}
```

### 2. Map business errors to `HTTPError` at the handler boundary

```go
var (
    ErrUserNotFound = errors.New("user not found")
    ErrUserDisabled = errors.New("user disabled")
)

func GetUser(c *echo.Context) error {
    user, err := svc.Get(c.Param("id"))
    switch {
    case errors.Is(err, ErrUserNotFound):
        return echo.NewHTTPError(http.StatusNotFound, "user not found")
    case errors.Is(err, ErrUserDisabled):
        return echo.NewHTTPError(http.StatusForbidden, "user disabled")
    case err != nil:
        return err   // system error — HTTPErrorHandler will catch it
    }
    return c.JSON(http.StatusOK, user)
}
```

### 3. Wrap errors so the chain survives

```go
// service layer
func Get(id string) (*User, error) {
    user, err := repo.Find(id)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)   // %w preserves the chain
    }
    return user, nil
}
```

`errors.Is(err, ErrUserNotFound)` still matches even after wrapping. **Do not** use `%v` or `%s` to format errors—the chain breaks.

### 4. Custom `HTTPErrorHandler` — single place to format responses

```go
type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details any    `json:"details,omitempty"`
    TraceID string `json:"trace_id,omitempty"`
}

func customErrorHandler(err error, c *echo.Context) {
    if c.Response().Committed {
        return
    }
    traceID := c.Response().Header().Get(echo.HeaderXRequestID)

    var he *echo.HTTPError
    if errors.As(err, &he) {
        _ = c.JSON(he.Code, APIError{
            Code:    httpCodeName(he.Code),   // "BAD_REQUEST" / "NOT_FOUND" etc.
            Message: fmt.Sprint(he.Message),
            TraceID: traceID,
        })
        return
    }

    // system / unknown error: log the full stack, only expose trace_id externally
    slog.Error("internal error",
        "err", err,
        "trace_id", traceID,
        "path", c.Path(),
    )
    _ = c.JSON(http.StatusInternalServerError, APIError{
        Code:    "INTERNAL",
        Message: "internal server error",
        TraceID: traceID,
    })
}
```

## Forbidden patterns

- `c.JSON(http.StatusBadRequest, ...)` for error responses. Use `echo.NewHTTPError`.
- `if err.Error() == "user not found"` string compare. Use `errors.Is` with sentinel errors.
- `return nil` after writing an error response (swallows the error path).
- `return fmt.Errorf("foo: %v", err)` (loses chain). Use `%w`.
- `panic(err)` in handlers. Return errors. `Recover` middleware exists for unexpected panics, not control flow.
- Per-handler JSON envelope structs (`struct{ Code int; Data T; Error string }`). Return data on success, return error on failure—`HTTPErrorHandler` formats both ends.
- `c.JSON` repeated in every handler for the same error shape. Centralize in `HTTPErrorHandler`.
- Ignoring `c.Bind` / `c.Validate` errors.
- `e.Start` without graceful shutdown.
- Per-route auth check repeated in every handler. Use `e.Group("/admin", authMW)`.
- `import "github.com/labstack/echo/v4"`. Move to `v5`.
- `fmt.Println` / `log.Printf` for logging. Use `e.Logger` or a project-wide `slog` wrapper bound to the request context.
- Hand-rolled CORS / rate-limiter / logger / recover / request-id middleware. Use echo's built-in `middleware` package: `middleware.CORS()`, `middleware.RateLimiter(...)`, `middleware.Logger()`, `middleware.Recover()`, `middleware.RequestID()`.
- Hand-rolled JWT auth. In v5, JWT lives in the standalone package `github.com/labstack/echo-jwt`—install it instead of writing token parsing yourself.
- Hand-rolled router on top of `net/http` when echo is already in scope. Stick to `e.GET` / `e.POST` / `e.Group(...)`.
- Hand-written middleware / handler helper when the project already has one under `internal/middleware/`, `pkg/middleware/`, or similar. grep first; reuse if found.

## Common antipatterns

| ❌ | ✅ |
|---|---|
| `c.JSON(400, map[string]string{"err": "bad"})` | `return echo.NewHTTPError(http.StatusBadRequest, "bad")` |
| `if err.Error() == "..." { ... }` | `if errors.Is(err, ErrFoo) { ... }` |
| `return fmt.Errorf("get user: %v", err)` | `return fmt.Errorf("get user %s: %w", id, err)` |
| Hand-written auth check on every route | `g := e.Group("/api", AuthMiddleware()); g.GET(...)` |
| `e.Start(":8080")` with direct fatal exit | `go e.Start(...)` + signal trap + `e.Shutdown(ctx)` |
| `if err != nil { panic(err) }` | `if err != nil { return err }` |
| `import echo/v4` | `import echo/v5` |

## When you need behavior outside Echo

If you need a feature Echo does not provide (custom transport, h2c, websocket with non-standard upgrade), **STOP** and report. Do not bypass the framework with raw `http.Handler` mixed in.

## Verification (grep after every .go change)

```bash
grep -rnE 'echo/v4' --include='*.go' --include='go.mod' .
grep -rnE 'c\.JSON\([^)]*[Ss]tatus(BadRequest|NotFound|Unauthorized|Forbidden|InternalServerError)' --include='*.go' .
grep -rnE 'err\.Error\(\)\s*==' --include='*.go' .                   # string comparison
grep -rnE 'fmt\.Errorf\([^)]*%v[^)]*err\)' --include='*.go' .         # should be %w
grep -rnE 'panic\(err' --include='*.go' .
grep -rnE 'fmt\.Println|log\.Printf' --include='*.go' .
grep -rnE '^\s*e\.Start\(' --include='*.go' .                         # check graceful shutdown
```

Reference: https://echo.labstack.com/docs
