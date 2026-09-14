---
name: echo
description: Enforce Echo v5 handler conventions for Go, focused on error handling. Use when editing .go handler/route/middleware files or when the user mentions Echo, HandlerFunc, c.Bind, middleware, HTTPError, HTTPErrorHandler, error wrap, or graceful shutdown. Routes every failure through HTTPError and one central HTTPErrorHandler, matches errors with errors.Is/As, wraps and returns them, and starts the server with graceful shutdown.
paths:
  - "**/*.go"
---

> Targets Echo v5 · verified 2026-09 (latest v5.3.1, released 2026-07-21).

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
- **Graceful shutdown is mandatory**: start with `echo.StartConfig{GracefulTimeout: ...}.Start(ctx, e)` on a `signal.NotifyContext` context. `e.Start(addr)` does the same with a 10s default timeout; v5 has no `e.Shutdown`.

## Server skeleton

```go
e := echo.New()
e.HTTPErrorHandler = customErrorHandler
e.Validator = &customValidator{...}
e.Use(middleware.RequestLogger(), middleware.Recover(), middleware.RequestID())

// routes ...

ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

sc := echo.StartConfig{Address: ":8080", GracefulTimeout: 10 * time.Second}
if err := sc.Start(ctx, e); err != nil {   // http.ErrServerClosed is already filtered out
    e.Logger.Error("server stopped", "error", err)
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

func customErrorHandler(c *echo.Context, err error) {
    if resp, uErr := echo.UnwrapResponse(c.Response()); uErr == nil && resp.Committed {
        return
    }
    traceID := c.Response().Header().Get(echo.HeaderXRequestID)

    // HTTPStatusCoder covers *HTTPError, *BindingError and router errors (ErrNotFound, ErrMethodNotAllowed)
    var sc echo.HTTPStatusCoder
    if errors.As(err, &sc) && sc.StatusCode() != 0 {
        code := sc.StatusCode()
        msg := http.StatusText(code)
        switch t := sc.(type) {
        case *echo.HTTPError:
            if t.Message != "" {
                msg = t.Message
            }
        case *echo.BindingError:
            if t.Message != "" {
                msg = t.Message
            }
        }
        _ = c.JSON(code, APIError{
            Code:    httpCodeName(code),   // "BAD_REQUEST" / "NOT_FOUND" etc.
            Message: msg,
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
- `e.Shutdown(ctx)` / `e.Logger.Fatal(err)` copied from v4. Neither exists in v5 (`e.Logger` is `*slog.Logger`); use `echo.StartConfig{...}.Start(ctx, e)`.
- Per-route auth check repeated in every handler. Use `e.Group("/admin", authMW)`.
- `import "github.com/labstack/echo/v4"`. Move to `v5`.
- `fmt.Println` / `log.Printf` for logging. Use `e.Logger` or a project-wide `slog` wrapper bound to the request context.
- Hand-rolled CORS / rate-limiter / logger / recover / request-id middleware. Use echo's built-in `middleware` package: `middleware.CORS()`, `middleware.RateLimiter(...)`, `middleware.RequestLogger()`, `middleware.Recover()`, `middleware.RequestID()`.
- Hand-rolled JWT auth. In v5, JWT lives in the standalone package `github.com/labstack/echo-jwt/v5`—install it instead of writing token parsing yourself.
- Hand-rolled router on top of `net/http` when echo is already in scope. Stick to `e.GET` / `e.POST` / `e.Group(...)`.
- Hand-written middleware / handler helper when the project already has one under `internal/middleware/`, `pkg/middleware/`, or similar. grep first; reuse if found.

## Common antipatterns

| ❌ | ✅ |
|---|---|
| `c.JSON(400, map[string]string{"err": "bad"})` | `return echo.NewHTTPError(http.StatusBadRequest, "bad")` |
| `if err.Error() == "..." { ... }` | `if errors.Is(err, ErrFoo) { ... }` |
| `return fmt.Errorf("get user: %v", err)` | `return fmt.Errorf("get user %s: %w", id, err)` |
| Hand-written auth check on every route | `g := e.Group("/api", AuthMiddleware()); g.GET(...)` |
| `go e.Start(...)` + signal trap + `e.Shutdown(ctx)` (v4) | `echo.StartConfig{Address: ":8080", GracefulTimeout: 10 * time.Second}.Start(ctx, e)` |
| `if err != nil { panic(err) }` | `if err != nil { return err }` |
| `import echo/v4` | `import echo/v5` |

## When you need behavior outside Echo

If you need a feature Echo does not provide (custom transport, h2c, websocket with non-standard upgrade), **STOP** and report. Do not bypass the framework with raw `http.Handler` mixed in.

## Verification (grep after every .go change)

```bash
grep -rnE 'echo/v4' --include='*.go' --include='go.mod' .
grep -rnE 'c\.JSON\([^)]*[Ss]tatus(BadRequest|NotFound|Unauthorized|Forbidden|InternalServerError)' --include='*.go' .
grep -rnE 'err\.Error\(\)\s*==' --include='*.go' .                   # string comparison
grep -rnE 'fmt\.Errorf\(.*%[vs].*,[[:space:]]*err\)' --include='*.go' . | grep -v '%w'   # should be %w
grep -rnE 'panic\(err' --include='*.go' .
grep -rnE 'fmt\.Println|log\.Printf' --include='*.go' .
```

Reference: https://echo.labstack.com/docs
