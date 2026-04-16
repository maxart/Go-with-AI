# The Go Backend Best Practices Guide (2025-2026)

A long-form companion to `CLAUDE.md.FULL` / `CLAUDE.md.SHORT`. This guide explains the **why** behind each rule, alternatives we considered, and the gotchas you'll hit in production. It targets Go 1.24+ on AWS (Lambda, ECS Fargate, EKS, EC2). Read top-to-bottom or jump by section.

If you're looking for the rules themselves — short, punchy, copy-pasteable into Claude Code — use `CLAUDE.md.SHORT` (no code blocks) or `CLAUDE.md.FULL` (with canonical examples). This file is for humans who want the reasoning.

---

## 1. Why Go for Backend Services on AWS

Go is not the fastest language. It is not the most expressive. It does not have the largest ecosystem. What it has is **a very small distance between "I read this file" and "I understand exactly what it does"**, and that property — combined with single-binary deploys, fast compiles, a sane stdlib, first-class concurrency, and a runtime that fits in tens of megabytes of RAM — makes it the best general-purpose backend language for cloud services in 2025-2026.

Concrete benefits on AWS:

- **Single static binary, no runtime.** A Go service builds to one file you copy into a `FROM scratch` Docker image. The final image is often under 20 MB. Smaller images mean faster ECS/Fargate cold starts, faster autoscaling, less S3/ECR bandwidth, less attack surface.
- **Cold-start measured in tens of milliseconds.** On AWS Lambda's `provided.al2023` runtime with `arm64` (Graviton2), Go cold starts routinely land under 60 ms. Compare to JVM (300-1500 ms) or Python with heavy imports (200-800 ms). For synchronous user-facing APIs this matters.
- **Memory footprint that compounds into cost.** Independent benchmarks consistently show Go using around a quarter of the memory of equivalent Java services at the same throughput. On a c6g.xlarge (8 GB RAM, ARM Graviton2) you fit roughly 18 typical Go services vs ~5 typical Java services. Multiply across hundreds of pods and the AWS bill diverges.
- **Concurrency that maps onto how cloud services actually work.** Most backend services are I/O-bound — waiting on Postgres, S3, DynamoDB, downstream APIs. Go's goroutines + channels + `context.Context` make fan-out / fan-in / cancellation tractable in a way that Node's callback / promise model and Python's `asyncio` make awkward, and that the JVM's threads make expensive.
- **Boring, stable, production-grade stdlib.** `net/http`, `database/sql`, `encoding/json`, `crypto/tls`, `log/slog`, `context`, `sync`, `errors` — these are battle-tested, generally do the right thing, and rarely break across versions. You can build a serious service with stdlib + 5-10 well-chosen modules. You don't need a 200-dependency `node_modules`-equivalent.
- **Compile-checked errors.** Go's `error` returns + `errors.Is`/`errors.As` give you typed error-handling without exception spaghetti. Combined with the race detector (`go test -race`) and `govulncheck`, the language pulls a lot of bug categories forward to compile / CI time.

Where Go is **not** the right answer: heavy data-processing pipelines (Python + pandas/Polars or Spark/Scala usually win), complex domain modeling with deep inheritance hierarchies (Java/Kotlin/C# may fit better), latency-critical systems below 100 µs (Rust/C++), large data-science / ML serving stacks (Python around the model, Go can wrap it). For most CRUD APIs, async workers, AWS glue, gRPC services, ETL controllers, real-time event handlers — Go is the right call.

The rest of this guide assumes you're building those kinds of services. If you're building a Kubernetes operator, AWS Lambda extension, or CLI tool, almost everything still applies.

---

## 2. Go Language Foundations

### 2.1 Versions and modules

The latest stable Go release is what you should target for new code. As of Q1 2026, that's Go 1.24+ (1.24 shipped February 2025; 1.25 shipped August 2025; 1.26 around February 2026). Pin the **minor** in `go.mod`'s `go` directive to whatever your CI runs:

```go
module github.com/maxart/example-service

go 1.24

toolchain go1.24.3
```

The `toolchain` directive lets you pin a patch version (helpful when reproducibility across developer laptops + CI matters). Use it sparingly — most teams pin only the `go` directive.

`go.mod` and `go.sum` are both committed. Run `go mod tidy` before every commit; CI must fail if it would change anything (`git diff --exit-code go.mod go.sum`). Never edit the `require` block by hand — `go get pkg@version` is correct, idempotent, and updates `go.sum`. `go get -u ./...` upgrades transitively and should only be used deliberately, ideally one major bump per PR so regressions are easy to bisect.

### 2.2 Idioms: small interfaces, concrete returns

The Go community's "accept interfaces, return concrete types" advice is widely repeated and widely misunderstood. The actual point is:

1. **Interfaces are best when defined where they're consumed.** If `package billing` needs to read invoices, it defines `type InvoiceReader interface { GetInvoice(ctx, id) (Invoice, error) }`. The producer (`package invoice`) doesn't need to know about it. This means consumers can swap implementations (real, fake, mock) without coordinating with producers.
2. **Tiny interfaces compose better than large ones.** `io.Reader` (one method) is everywhere. A 12-method `UserService` interface is not. If you find yourself defining big interfaces, the consumer is doing too much.
3. **Returning a concrete struct lets callers see the real shape** — fields, additional methods, the `String()` representation. Returning an interface forces every caller through the lowest-common-denominator surface.

When this advice doesn't apply: **constructors that need to support multiple implementations** (e.g., a logger factory that returns either a console or JSON logger) reasonably return an interface. Use judgment.

### 2.3 Generics

Generics shipped in Go 1.18 and have been refined steadily through 1.24. The `comparable` and `Ordered` constraints (from `cmp.Ordered`) cover most needs. Use generics when:

- You'd otherwise duplicate code that differs only in element type (collection helpers, set operations, retry wrappers).
- You'd otherwise use `any` (formerly `interface{}`) and pay for runtime type assertions.

Don't use generics when:

- You think it might be useful "later". Concrete code refactors fine; speculative generics multiply complexity.
- A method or constraint set would be longer than the code it parameterizes.

`any` is an alias for `interface{}` and should be the default in new code. Most linters now flag `interface{}` usage.

### 2.4 Iterators (range over function)

Go 1.23 introduced range over function — iterators that look like `func(yield func(T) bool)`. Useful for streaming, lazy evaluation, paginated traversal. Many stdlib packages (`bytes.Lines`, `strings.SplitSeq`, `maps.Keys`, `slices.All`) now expose iterator forms.

When to use:

- A producer naturally yields a sequence (file lines, paginated API results, tree traversal).
- Consumers want to break early without loading everything.

When not to use:

- A plain slice would do. `for i, v := range slice` is more familiar and lets the consumer index, re-iterate, etc.

### 2.5 Errors

Errors are values. Return them; don't throw them. The most common patterns:

**Wrap with context.** `fmt.Errorf("user.GetByID id=%s: %w", id, err)`. The `%w` verb (Go 1.13+) preserves the error chain so callers can use `errors.Is` and `errors.As`. Never use `%v` for the inner error — it discards the chain.

**Sentinel errors for known modes.** `var ErrNotFound = errors.New("not found")`. Callers do `if errors.Is(err, ErrNotFound) { ... }`. This is the standard for lookup-style APIs (`pgx.ErrNoRows`, `io.EOF`, `os.ErrNotExist`).

**Custom error types when callers need fields.** A `ValidationError` carrying field-level messages, a `RetryableError` carrying a `RetryAfter`, a `RateLimitError` carrying tier info. Define as a struct implementing `Error() string`; check with `errors.As`.

**Don't compare strings.** `if err.Error() == "not found"` is a bug waiting to happen. Use `errors.Is`.

**Don't blindly bubble.** `if err != nil { return err }` without context loses information. Add at least the function name and key inputs unless your immediate caller is going to wrap it anyway. Excessive wrapping is also a smell — the goal is one informative chain, not three layers of "wrap: wrap: wrap: real error".

**`errors.Join` for aggregation.** Returning multiple errors (e.g., from validating several fields) can use `errors.Join(err1, err2, err3)`. Callers can `errors.Is` against any of them.

**Don't swallow.** `_ = file.Close()` is almost always wrong. At minimum log it; in many cases it should be wrapped and returned (use `defer func() { if cerr := f.Close(); cerr != nil && err == nil { err = cerr } }()`).

### 2.6 Panic and recover

`panic` is for programmer errors (nil pointer, index out of range, "this should never happen"). It is **not** for control flow. The only places you legitimately call `panic` in production code:

- A library detecting a misuse that can't be expressed in the type system (e.g., a regex that won't compile in a `MustCompile`-style helper at startup).
- `main`, where you've decided that the process can't usefully run (missing config that's required).

`recover` is appropriate at process / request boundaries. The standard production pattern is a top-level HTTP middleware that wraps the next handler in `defer func() { if r := recover(); r != nil { logger.Error(...); writeError(w, 500, ...) } }()`. Without this, one bad goroutine takes down the process — and on ECS that means a task restart and a few seconds of dropped requests.

### 2.7 Pointers, slices, maps

**Pointer or value?** The default is value. Pass pointers when:

- The struct is large (rough heuristic: > 16 bytes / 2 machine words). Profile first.
- You need to mutate.
- The type contains a `sync.Mutex` (mutexes must not be copied).
- A nil value is meaningful (config that may be absent).

Otherwise, value semantics are safer (immutable from the caller's perspective) and Go's escape analysis often promotes locals to the stack anyway.

**Slices.** `nil` is a valid slice for read operations (`len`, `range`, `append`). `make([]T, 0, cap)` is the right preallocation when you know the capacity. Subslicing aliases the backing array — `s2 := s1[2:5]` shares memory with `s1`. To detach (avoid leaking the rest of `s1`), `slices.Clone(s2)`. Be especially careful when returning a subslice that the caller will hold onto for a long time.

**Maps.** Zero-value (`map[K]V`) is `nil` and panics on write — always initialize with `make(map[K]V)`. Concurrent writes are a data race — guard with `sync.Mutex`, or use `sync.Map` if reads dominate and keys are stable (don't reach for `sync.Map` by default; it's slower than a guarded map for general use). Iteration order is randomized on every range loop; never depend on it.

**Defer.** Defers run in LIFO order at function return. Acquire the resource, then `defer cleanup()` immediately. Defers in loops accumulate (each iteration adds one, all run at function end) — for per-iteration cleanup, factor the body into a function or close inline.

---

## 3. Project Structure

There is no official Go project layout. The widely-cited [`golang-standards/project-layout`](https://github.com/golang-standards/project-layout) is a **community convention**, not an Authority — its README itself disclaims that. But the conventions it documents are real, and they reduce friction for new team members.

Recommended layout for a backend service:

```
service/
  cmd/
    api/main.go            entry point for the HTTP server
    worker/main.go         entry point for the background worker
    migrate/main.go        entry point for schema migrations
  internal/
    config/                loads + validates env vars at startup
    http/                  router, middleware, handlers
    user/                  domain package: types, store iface, service
    order/                 another domain package
    platform/
      database/            *pgxpool.Pool init
      awsx/                aws-sdk-go-v2 config helper
      logger/              slog setup
      otelx/               OpenTelemetry init
  pkg/                     public, importable by other modules (rare)
  api/                     OpenAPI specs, .proto files
  migrations/              SQL migrations (golang-migrate format)
  scripts/                 one-off shell/Go scripts
  Dockerfile, Makefile, go.mod, go.sum, .golangci.yml, .env.example
```

### 3.1 `cmd/`

`cmd/<binary>/main.go` is **wiring only**. Load config, build dependency graph, start the server, install signal handlers, return on shutdown. No business logic. The reason: this is the one file every operator looks at first. Keeping it small means anyone can answer "what runs when this process starts?" in 60 seconds.

A typical `cmd/api/main.go` is 50-150 lines. If yours is longer, factor.

One subdirectory per binary. A repo can have `cmd/api`, `cmd/worker`, `cmd/migrate`, `cmd/seed`, etc. — each builds to a separate binary via `go build ./cmd/<name>`.

### 3.2 `internal/`

The Go compiler enforces that packages under `internal/` can only be imported by code rooted at the parent of `internal/`. This is the only language-level access control in Go and you should use it aggressively. Default to `internal/`; promote to `pkg/` only when something is genuinely meant to be imported externally.

The harder question is **how to organize `internal/`**. Two common approaches:

**By technical layer** (`handlers/`, `services/`, `repos/`, `models/`): familiar to people coming from Spring, Rails, Django. Tends to produce "shotgun surgery" — adding a feature requires editing files in four packages, and packages accumulate unrelated code.

**By domain** (`user/`, `order/`, `billing/`, `notification/`): each package owns its types, service logic, and store implementation. Adding a feature usually touches one package. Package boundaries reflect the business, not the framework. **This is the recommended approach.**

A domain package typically contains:

- `user.go` — the core type and any domain logic on it
- `service.go` — the service struct with constructor and methods (depends on the store interface)
- `store.go` — the store interface (consumer-defined; tells the service what it needs)
- `store_postgres.go` — the Postgres implementation (depends on the generated sqlc `Queries`)
- `store_memory.go` — an in-memory fake for tests (optional but useful)
- `service_test.go` — table-driven tests against the service using the in-memory store

When two domains need to interact, the coordination logic goes in a third package (e.g., `internal/checkout` orchestrating `internal/cart` and `internal/payment`), in the HTTP handler, or in `cmd/` itself if it's truly top-level wiring.

### 3.3 `pkg/`

`pkg/` signals "this is reusable library code intended for external consumers". For most internal services, `pkg/` should be empty. Only put code here if other modules genuinely import it. Public-facing SDKs, shared utilities used across multiple services in a monorepo, etc.

### 3.4 What not to do

Avoid packages named `util`, `common`, `helpers`, `models`. They become magnets for unrelated functions and grow into 5,000-line catch-alls. Name packages by what they provide: `humanduration`, `addrvalidator`, `retrybackoff`. If you can't name it, you probably haven't found the right boundary yet.

Don't put everything in one package "for simplicity". Single-package services can't enforce layering, can't mock cleanly, and create import cycles the moment they grow.

---

## 4. HTTP Server

### 4.1 Why stdlib `net/http` is enough now

For most of Go's history, the stdlib router (`http.ServeMux`) was too primitive — no method matching, no path variables, no middleware chaining. Teams reached for `gorilla/mux` (now archived), then `chi`, `gin`, `echo`, `fiber`. By 2022 the convention was "always use `chi` or `gin`".

Go 1.22 (February 2024) changed that. The enhanced `ServeMux` supports:

- Method matching: `mux.HandleFunc("GET /users/{id}", getUser)`.
- Path wildcards: `r.PathValue("id")` extracts the value.
- Greedy wildcards: `{path...}` matches the rest of the URL.
- Exact path matching: `/users/{$}` matches only `/users/`.
- Automatic 405 with `Allow` header on method mismatch.

For most backend services, this is all you need. The benefits of stopping at stdlib:

- One less dependency to keep current and audit.
- Standard middleware shape (`func(http.Handler) http.Handler`) — every external middleware works.
- New team members already know the API.

When to reach for a router library:

- **`chi`** — large middleware ecosystem, route groups with shared middleware, cleaner sub-router composition. Stays close to stdlib semantics. Reasonable choice for medium-large APIs.
- **`echo` / `gin`** — bigger frameworks with their own context types, validators, binding. Reach for them only if your team is already invested or you genuinely use the integrated features.
- **`fiber`** — built on `fasthttp`, not `net/http`. Faster on synthetic benchmarks; loses compatibility with the standard middleware ecosystem and standard `context.Context` semantics. Niche.

### 4.2 Server configuration

Default `http.Server` values are unsafe in production. Always set:

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       120 * time.Second,
    MaxHeaderBytes:    1 << 20, // 1 MiB
}
```

`ReadHeaderTimeout` is the most important — it's the only one that protects against Slowloris (slow-header attacks). The others bound how long requests can stall before the server gives up.

Choose timeouts based on the workload. A streaming endpoint may need `WriteTimeout: 0` (no limit) and explicit timeouts via `http.NewResponseController` per request. Most APIs are fine with 30s.

### 4.3 Handler shape

Handlers are `http.HandlerFunc` or types implementing `http.Handler`. Inject dependencies via a struct:

```go
type API struct {
    users  *user.Service
    orders *order.Service
    logger *slog.Logger
}

func (a *API) routes() http.Handler {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", a.handleGetUser)
    mux.HandleFunc("POST /users",     a.handleCreateUser)
    mux.HandleFunc("GET /orders/{id}", a.handleGetOrder)
    return a.middleware(mux) // recover, request_id, logger, otel, auth
}
```

This pattern keeps test setup trivial (build the struct with fakes, call `ServeHTTP`) and makes the dependency surface explicit. Avoid global state.

### 4.4 Middleware

Middleware = `func(http.Handler) http.Handler`. Compose with a tiny helper or inline:

```go
func chain(h http.Handler, mws ...func(http.Handler) http.Handler) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}
```

You don't need a framework for this. Standard chain order, outermost to innermost:

1. **Panic recovery** — turns panics into 500s, logs the stack. Outermost so it catches everything.
2. **Request ID** — generate UUID, store in context, echo in `X-Request-ID`. Logs and traces should include it.
3. **Logger** — log request line + duration + status on response. Use `slog`.
4. **Tracing** — `otelhttp.NewHandler(next, "api")` extracts/creates the OTel span.
5. **Auth** — verify token, attach principal to context. Skip for `/healthz` and `/readyz`.
6. **CORS** (if applicable) — preflight handling, allowed origins from config.
7. **Rate limit** (if applicable) — token bucket per IP / per principal.
8. **Handler.**

### 4.5 Decoding requests

```go
func decodeJSON[T any](r *http.Request, v *T) error {
    r.Body = http.MaxBytesReader(nil, r.Body, 1<<20) // 1 MiB
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()
    if err := dec.Decode(v); err != nil { return err }
    if dec.More() { return errors.New("request body has trailing data") }
    return nil
}
```

`MaxBytesReader` caps request size at the I/O level — important for DoS resistance. `DisallowUnknownFields` rejects requests with unexpected keys — useful for strict APIs (catches client bugs early); skip if you want to allow forward-compatible clients to send extra fields.

### 4.6 DTOs and response shaping

**Never return raw domain or DB structs.** Always define a DTO per endpoint:

```go
type userDTO struct {
    ID    uuid.UUID `json:"id"`
    Email string    `json:"email"`
    Name  string    `json:"name"`
}

func toUserDTO(u user.User) userDTO {
    return userDTO{ID: u.ID, Email: u.Email, Name: u.Name}
}
```

Three reasons:

1. **Avoid leaking internals.** A `User` struct may have `PasswordHash`, `InternalID`, `DeletedAt`. Forgetting to strip them is a vulnerability.
2. **Avoid accidental exposure on refactor.** Add a field to `User` and it would silently appear in the API. The DTO catches this — you must consciously add it.
3. **Decouple wire format from storage.** Renaming a DB column shouldn't break clients.

The cost is a few lines of code per endpoint. Worth it.

### 4.7 Error responses

A stable error shape lets clients build robust error handling:

```json
{ "error": { "code": "user.not_found", "message": "User does not exist" } }
```

Include `code` (machine-readable, stable, namespaced) and `message` (human-readable, may change). Optionally `details` for field-level validation errors. Never include stack traces, file paths, or internal error strings — log the full chain server-side keyed by the request ID.

Use proper HTTP status codes. **Never 200 with `"success": false`.** Returning 200 for errors breaks every standard client library, every monitoring tool, every retry policy. Use 4xx for client errors, 5xx for server errors.

### 4.8 Graceful shutdown

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()

go func() {
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        logger.Error("server", "err", err)
    }
}()

<-ctx.Done()
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
if err := srv.Shutdown(shutdownCtx); err != nil {
    logger.Error("shutdown", "err", err)
}
pool.Close()
otelShutdown(shutdownCtx)
```

`Shutdown` stops accepting new connections and waits for in-flight requests to finish (up to the deadline). On ECS this means the task drains cleanly when the orchestrator sends SIGTERM during a deploy or scale-down. Without it, in-flight requests are aborted abruptly — users see 502s.

Set ECS task `stopTimeout` to 30-60s to give `Shutdown` time to drain.

---

## 5. Context, Concurrency, Cancellation

### 5.1 The context contract

`context.Context` is Go's standard mechanism for cancellation, deadlines, and request-scoped values. The convention:

- Every function that does I/O or other blocking work takes `ctx context.Context` as its **first** parameter.
- Pass it through. Don't store it in a struct field (lint rule: `containedctx`).
- In HTTP handlers, use `r.Context()`. In Lambda handlers, use the context passed in. Never `context.Background()` from request-handling code.
- `context.TODO()` is for "I haven't decided yet" — leave a marker that this should become a real context.

The point of the contract: a single `cancel()` at the top of a request tree propagates immediately to every goroutine, every DB query, every outbound HTTP call. The DB driver releases its connection, the HTTP client closes the socket, and your service stops doing work the user no longer cares about.

### 5.2 Deadlines and timeouts

Use `context.WithTimeout` or `context.WithDeadline` at the boundary where you decide the budget:

- HTTP handler entry: bound the total time you'll spend on the request.
- Outbound RPC call: bound how long you'll wait for the downstream.
- Database query: bound how long you'll wait for the row.

Always `defer cancel()`. Forgetting it leaks the context — the timer fires later than you wanted but technically nothing breaks, except over time you run out of resources.

Don't add timeouts inside library code. Let the caller decide. A library function that imposes its own 10-second timeout breaks every user who needs 30s, and silently shortens every user who set 5s. Take the context, honor the context, return when it cancels.

### 5.3 `context.WithValue`

Use for request-scoped data that crosses API boundaries: request ID, authenticated principal, trace span. Do not use for optional function arguments — that's what arguments are for. Use unexported key types to avoid collisions:

```go
type ctxKey int
const (
    ctxKeyRequestID ctxKey = iota
    ctxKeyPrincipal
)
func WithPrincipal(ctx context.Context, p Principal) context.Context {
    return context.WithValue(ctx, ctxKeyPrincipal, p)
}
func PrincipalFrom(ctx context.Context) (Principal, bool) {
    p, ok := ctx.Value(ctxKeyPrincipal).(Principal)
    return p, ok
}
```

### 5.4 Goroutines

Every goroutine you start in a long-running service must have a defined termination path. The two acceptable patterns:

1. **Bounded by the caller's context** — the goroutine runs `select { case <-ctx.Done(): return; case x := <-ch: ... }` and exits when the context is cancelled.
2. **Bounded by an explicit channel close or done signal** — the goroutine ranges over a channel that the caller closes.

Naked `go fn()` without one of these leaks. You'll discover it at 3 AM when memory grows linearly with traffic.

### 5.5 errgroup

For parallel work that should fail-fast, `golang.org/x/sync/errgroup` is the right tool:

```go
g, gctx := errgroup.WithContext(ctx)
var user user.User
var orders []order.Order
g.Go(func() error {
    var err error
    user, err = userSvc.GetByID(gctx, id)
    return err
})
g.Go(func() error {
    var err error
    orders, err = orderSvc.ListByUser(gctx, id)
    return err
})
if err := g.Wait(); err != nil { return Profile{}, err }
```

The first non-nil error returned by any goroutine cancels `gctx`, and `Wait()` returns that error. Other goroutines should be honoring `gctx` and exit promptly.

Since `errgroup` v0.3.0 uses `context.WithCancelCause` internally, `context.Cause(gctx)` returns the actual goroutine error — log this for debugging.

`sync.WaitGroup` is fine when you don't need error aggregation (e.g., kicking off non-critical background work).

### 5.6 Channels and synchronization

Channels for hand-off and synchronization. Mutexes for shared state. Use the right tool:

- **Unbuffered channel**: a synchronization point — sender blocks until receiver is ready.
- **Buffered channel**: a queue — useful for bursting, decoupling rates.
- **`sync.Mutex`**: protect a small piece of shared state. Hold for the shortest possible window.
- **`sync.RWMutex`**: only when reads vastly dominate (10x or more) — measure first; the bookkeeping overhead means RWMutex can be slower than Mutex in many workloads.
- **`sync.Once`**: idempotent initialization (rarely needed; package-level vars and init() usually cover it).
- **`sync.Pool`**: object reuse for short-lived allocations in hot paths. Don't use as a cache.

Channel rules people forget:

- The **sender** closes. Closing a closed channel panics.
- Receiving from a closed channel returns the zero value immediately. Use `v, ok := <-ch` to detect close.
- A nil channel in `select` is never selected — useful for disabling a case dynamically.
- Buffered channels with capacity > 1 add latency variability — measure.

### 5.7 The race detector

`go test -race ./...` instruments memory accesses and detects data races at runtime. Run it in CI on every PR. Any `WARNING: DATA RACE` is a release blocker — races in Go can cause memory corruption, panics, or just subtly wrong results, and they're often only caught by the detector (not by reading code).

The race detector slows tests by 5-15x and uses 5-10x more memory. That's why it runs in CI rather than dev iteration. Some teams also run a fraction of production load with `-race` periodically (weekly canaries) to catch races that only appear under real concurrency.

---

## 6. Database Access (PostgreSQL)

### 6.1 Driver: pgx

For new code targeting Postgres, use `github.com/jackc/pgx/v5`. It's the de-facto standard:

- Faster than `database/sql` + `lib/pq` for most workloads.
- Native support for Postgres types (jsonb, arrays, hstore, listen/notify, COPY, prepared statements).
- Two APIs: a low-level pgx API (preferred for performance) and a `database/sql` driver (`pgx/v5/stdlib`) when you need driver portability.
- `lib/pq` is in maintenance mode — don't pick it for new services.

Pool initialization at startup:

```go
poolCfg, err := pgxpool.ParseConfig(dsn)
if err != nil { return nil, fmt.Errorf("parse dsn: %w", err) }
poolCfg.MaxConns = 20
poolCfg.MinConns = 2
poolCfg.MaxConnLifetime = 30 * time.Minute
poolCfg.MaxConnIdleTime = 5 * time.Minute
poolCfg.HealthCheckPeriod = 1 * time.Minute

pool, err := pgxpool.NewWithConfig(ctx, poolCfg)
if err != nil { return nil, fmt.Errorf("new pool: %w", err) }
if err := pool.Ping(ctx); err != nil { return nil, fmt.Errorf("ping: %w", err) }
return pool, nil
```

`MaxConns` is a per-pod cap, not a global one. If you run 10 pods with `MaxConns=20`, you're holding up to 200 Postgres connections. Postgres is sensitive to high connection counts — RDS Postgres typically maxes around 5,000 connections; for pods running thousands of replicas, use a connection pooler (PgBouncer, RDS Proxy) and keep per-pod `MaxConns` low (5-10).

`MaxConnLifetime` (~30 min) lets the pool periodically rotate connections — important when running through load balancers (RDS, RDS Proxy) that may need to rebalance.

### 6.2 Type-safe queries with sqlc

sqlc generates Go code from SQL queries. You write SQL by hand, sqlc parses it, infers types from your schema, and generates a `Queries` struct with type-safe methods.

Workflow:

1. Define schema in `db/schema.sql` (or by reading from Postgres directly).
2. Define queries in `db/queries/*.sql` with `-- name: GetUserByID :one` annotations.
3. Run `sqlc generate` to produce `internal/db/queries.sql.go` (or wherever your `sqlc.yaml` points).
4. Use the generated code from your store implementations.

Why sqlc over alternatives:

- **vs ORM (GORM, ent)**: ORMs hide SQL, generate inefficient queries, and break down on anything non-trivial (window functions, CTEs, partial indexes). sqlc keeps you in SQL.
- **vs hand-written `database/sql`**: sqlc eliminates the entire category of "wrong type passed to Scan" bugs and "forgot a column" bugs.
- **vs query builders (squirrel, goqu)**: query builders are clever but produce code that's harder to read than the equivalent SQL. sqlc gives you the generated Go interface and keeps the SQL itself canonical.

Configure sqlc to use pgx:

```yaml
# sqlc.yaml
version: "2"
sql:
  - engine: postgresql
    queries: db/queries
    schema: db/schema.sql
    gen:
      go:
        package: db
        out: internal/db
        sql_package: pgx/v5
```

### 6.3 Transactions

```go
func (s *Service) TransferFunds(ctx context.Context, fromID, toID uuid.UUID, amt int64) error {
    tx, err := s.pool.BeginTx(ctx, pgx.TxOptions{IsoLevel: pgx.Serializable})
    if err != nil { return fmt.Errorf("begin: %w", err) }
    defer tx.Rollback(ctx) // no-op after commit

    qtx := s.queries.WithTx(tx)
    if err := qtx.DebitAccount(ctx, db.DebitAccountParams{ID: fromID, Amount: amt}); err != nil {
        return fmt.Errorf("debit: %w", err)
    }
    if err := qtx.CreditAccount(ctx, db.CreditAccountParams{ID: toID, Amount: amt}); err != nil {
        return fmt.Errorf("credit: %w", err)
    }
    return tx.Commit(ctx)
}
```

Keep transactions **short**. They hold row locks. A 30-second transaction can block every other writer to the affected rows. If you find yourself doing HTTP calls or long computations inside a transaction, restructure — do the slow part outside, then a quick transaction at the end.

Choose isolation level explicitly. Postgres defaults to Read Committed. `Serializable` is the safest default for financial / inventory work; `Repeatable Read` is a middle ground.

### 6.4 No string concatenation

This is non-negotiable: never build SQL with `fmt.Sprintf` and user input. Always use parameter placeholders (`$1`, `$2`, ...). pgx and sqlc both make this the easy path.

Dynamic `IN (...)`: use `pgx.NamedArgs` or `=ANY($1)` with an array:

```go
rows, err := tx.Query(ctx, "SELECT * FROM users WHERE id = ANY($1)", ids)
```

Dynamic columns / table names (rare, almost always wrong): if you truly need to, validate against a hard-coded allow-list, never the raw input.

`gosec` rules `G201` and `G202` will flag string-built SQL. Treat any G201/G202 finding as a release blocker.

### 6.5 Store interface in the domain package

The store interface lives in the **consumer** package (the domain package), not the producer. This is the "accept interfaces, return concrete types" rule applied:

```go
// internal/user/store.go
type Store interface {
    Create(ctx context.Context, u User) error
    GetByID(ctx context.Context, id uuid.UUID) (User, error)
    Update(ctx context.Context, u User) error
}
```

The Postgres implementation (`store_postgres.go`) and the in-memory fake (`store_memory.go`) both implement this interface. The service depends on `Store`, not on either impl. Tests use the in-memory fake; production uses Postgres. No mock framework required.

### 6.6 Migrations

Use a real migration tool: `golang-migrate/migrate`, `pressly/goose`, or `ariga/atlas`. Versioned SQL files in `migrations/`. Each migration is **append-only** in production — you don't edit a migration that's been applied; you write a new one to fix it.

Run migrations as a separate step before deploy, not at app startup. Reasons:

- Multiple replicas all trying to migrate concurrently is a recipe for partial state.
- A migration failure should block the deploy, not crash-loop the new app version.
- You may need to migrate from a privileged DB user different from the runtime user.

Common pattern: a `cmd/migrate/main.go` that runs in a separate ECS task / Lambda / CI step before the rolling deploy of the API. Or use the migration tool's CLI in the deploy pipeline.

---

## 7. AWS SDK for Go v2

### 7.1 v2, not v1

AWS SDK for Go v1 reached end-of-support on **July 31, 2025**. New code must use v2 (`github.com/aws/aws-sdk-go-v2/...`). Existing v1 code should be on a migration plan — v1 will continue to function but receive no updates, including no security patches.

v2 differs from v1 in important ways:

- Modular: `aws-sdk-go-v2/config`, `aws-sdk-go-v2/service/s3`, etc. — you only pull in what you use, smaller binaries.
- Context-first: every API call takes `ctx` as the first arg.
- Functional options for client construction: `s3.NewFromConfig(cfg, func(o *s3.Options) { ... })`.
- `aws.String` / `aws.Int` etc. helpers for required pointer fields (still annoying but unavoidable).

### 7.2 Configuration

Load AWS config once at startup. Reuse the `aws.Config` to create per-service clients:

```go
cfg, err := config.LoadDefaultConfig(ctx,
    config.WithRegion(region),
    config.WithRetryMaxAttempts(5),
    config.WithRetryMode(aws.RetryModeAdaptive),
)
if err != nil { return nil, fmt.Errorf("aws config: %w", err) }

s3Client := s3.NewFromConfig(cfg)
ddbClient := dynamodb.NewFromConfig(cfg)
```

Service clients are safe for concurrent use — keep them as singletons. Creating a new client per request burns CPU on TLS handshakes and underlying connection setup that the SDK would otherwise reuse.

### 7.3 Credentials

**Never put AWS access keys in environment variables, config files in containers, or anywhere else in your service.** Use IAM roles:

- **Lambda**: execution role attached to the function. SDK picks it up automatically.
- **ECS Task** (Fargate or EC2): task role. SDK picks it up via the ECS metadata endpoint.
- **EKS pod**: IRSA (IAM Roles for Service Accounts) via the projected service account token. SDK picks it up via the `AWS_WEB_IDENTITY_TOKEN_FILE` env var that EKS injects.
- **EC2**: instance profile. SDK picks it up via IMDSv2.
- **Local dev**: `aws sso login` → SDK picks up via `~/.aws/credentials` and `AWS_PROFILE`.

The default credential chain handles all of these. Don't override unless you have a very specific reason.

### 7.4 Region

The SDK reads `AWS_REGION` (and `AWS_DEFAULT_REGION` as fallback). Lambda and ECS set this automatically. For local dev, set it via env or `aws configure`. Fail-fast at startup if the region is unset and you're in an environment that requires it.

### 7.5 Retries

The SDK has built-in retry. Choose the mode:

- **Standard** (default): retries common transient errors with exponential backoff. Good for most use cases.
- **Adaptive**: standard plus client-side throttling that backs off when the service is overloaded. Use for high-throughput workloads where you might overwhelm the service (DynamoDB hot partitions, S3 burst-then-throttle, etc.).
- **Off**: rare; only when you're handling retries in application code.

`config.WithRetryMaxAttempts(5)` is a reasonable upper bound. Pair with sensible per-call timeouts via `ctx`.

### 7.6 Paginators

For any list operation, use the SDK's paginator types:

```go
p := s3.NewListObjectsV2Paginator(client, &s3.ListObjectsV2Input{Bucket: aws.String(bucket)})
for p.HasMorePages() {
    page, err := p.NextPage(ctx)
    if err != nil { return fmt.Errorf("list page: %w", err) }
    for _, obj := range page.Contents {
        // process
    }
}
```

The paginator handles `NextToken`, retries, and context cancellation correctly. Hand-rolled pagination loops have a long history of bugs (off-by-one, missed pages, infinite loops on stale tokens).

### 7.7 Endpoint resolution

Don't override `BaseEndpoint` outside tests / LocalStack. The SDK's default endpoint resolution picks the right regional endpoint, FIPS endpoint where required, dual-stack endpoint where appropriate.

For LocalStack or Minio in tests:

```go
s3Client := s3.NewFromConfig(cfg, func(o *s3.Options) {
    o.BaseEndpoint = aws.String("http://localhost:4566")
    o.UsePathStyle = true // required for LocalStack S3
})
```

### 7.8 What not to do

- Don't bake AWS account IDs or full ARNs into source code. Pass them via env (or fetch from the metadata service / STS at runtime).
- Don't log raw API responses without filtering — they contain headers (sometimes credentials), object metadata, and PII.
- Don't share clients across regions — create one per region. Cross-region calls in a single client are not optimized.
- Don't use `aws-sdk-go-v2/aws/session` (that's v1). v2 doesn't have sessions; it has `aws.Config`.

---

## 8. AWS Lambda

### 8.1 Runtime: `provided.al2023`

The `go1.x` runtime was deprecated December 31, 2023 and is no longer accepting new function deployments. Use `provided.al2023` (the AL2023-based custom runtime) for all new Go Lambda functions. The previous `provided.al2` (AL2-based) also works but `provided.al2023` is the current default — smaller base image (under 40 MB vs 100+ MB for AL2-based), newer libc, longer support window.

Build:

```bash
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build \
  -tags lambda.norpc \
  -ldflags "-s -w" \
  -trimpath \
  -o bootstrap .
zip function.zip bootstrap
```

The binary **must** be named `bootstrap`. The `lambda.norpc` build tag drops the legacy RPC support code (smaller binary). `-s -w` strip debug info; `-trimpath` removes filesystem paths.

### 8.2 Architecture: arm64

Lambda on arm64 (Graviton2) is **20% cheaper for duration** and roughly **19% faster** than x86_64 for typical workloads. Cross-compiling Go for arm64 is trivial (`GOARCH=arm64`); no Docker or emulation needed. Default to arm64 for new functions.

The only reason to fall back to x86_64 is a CGO dependency that doesn't have arm64 wheels. With `CGO_ENABLED=0`, you can build a static binary that runs on either architecture (just rebuild for the target). Most pure-Go services have no CGO deps.

### 8.3 Handler

```go
package main

import (
    "context"
    "log/slog"
    "os"

    "github.com/aws/aws-lambda-go/events"
    "github.com/aws/aws-lambda-go/lambda"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb"
)

var (
    logger    *slog.Logger
    ddbClient *dynamodb.Client
)

func init() {
    logger = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
    cfg, err := config.LoadDefaultConfig(context.Background())
    if err != nil { panic(err) } // can't usefully run
    ddbClient = dynamodb.NewFromConfig(cfg)
}

func handle(ctx context.Context, evt events.APIGatewayV2HTTPRequest) (events.APIGatewayV2HTTPResponse, error) {
    // ... business logic using ddbClient and logger
    return events.APIGatewayV2HTTPResponse{StatusCode: 200, Body: `{"ok":true}`}, nil
}

func main() { lambda.Start(handle) }
```

Initialize clients, config, and loggers **outside** the handler — at package level via `init()` or as package vars. The Lambda execution environment is reused across invocations; per-invocation init is wasted billed time and adds latency.

### 8.4 Cold starts

Cold start on arm64 + AL2023 is achievable under 60 ms for a small Go binary. Steps that help:

- **Tiny binary**: `-s -w -trimpath`, `lambda.norpc` tag, `CGO_ENABLED=0`. A typical handler binary is 5-15 MB.
- **Minimal package-level init**: avoid expensive operations in `init()` — they run on every cold start.
- **Lazy init for rarely-used clients**: if a client is only used by 1 in 100 invocations, lazy-init it on first use.

For p99 cold start in user-facing APIs:

- **Provisioned Concurrency**: keeps `N` execution environments warm. Costs about as much as a small EC2 instance per provisioned concurrent invocation but eliminates cold starts for that capacity.
- **SnapStart** for Java was the headline feature; SnapStart for other languages is rolling out — check current support.
- For async (SQS, EventBridge, S3), cold starts usually don't matter.

### 8.5 HTTP in Lambda

Don't run a long-lived HTTP server. Use API Gateway, ALB, or Function URLs to invoke the function. The event types for each are in `github.com/aws/aws-lambda-go/events`:

- `APIGatewayProxyRequest` (REST API)
- `APIGatewayV2HTTPRequest` (HTTP API)
- `ALBTargetGroupRequest` (ALB target)
- `LambdaFunctionURLRequest` (Function URL)

If you're migrating an existing `net/http` service to Lambda, `aws-lambda-go-api-proxy` adapts `http.Handler` to handle these events. Acceptable for migration; native handlers are faster (no adapter overhead, no need to rebuild a `Request` and `ResponseWriter`).

### 8.6 Logging

Write structured JSON to stdout. CloudWatch Logs picks it up. Use `slog.JSONHandler`:

```go
logger.InfoContext(ctx, "processed",
    "lambda_request_id", lambdacontext.FromContext(ctx).AwsRequestID,
    "event_type", "sqs",
    "messages", len(evt.Records),
)
```

Querying via CloudWatch Logs Insights:

```
fields @timestamp, level, msg, lambda_request_id
| filter level = "ERROR"
| sort @timestamp desc
| limit 100
```

Plain `log.Println` lines are unstructured and effectively unqueryable at scale.

---

## 9. AWS ECS / Fargate

### 9.1 Container shape

One process per container. Use a multi-stage Dockerfile to keep the runtime image tiny:

```dockerfile
FROM golang:1.24-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=arm64 \
    go build -trimpath -ldflags "-s -w -X main.version=${VERSION}" \
    -o /out/api ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/api /api
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/api"]
```

Distroless `static-debian12:nonroot` includes:

- A non-root user.
- `/etc/ssl/certs/ca-certificates.crt` (so HTTPS to AWS APIs works).
- `/etc/passwd`, `/etc/group`, `/etc/nsswitch.conf` for nonroot user.
- No shell, no package manager, no extra tools.

Final image is typically under 20 MB.

For arm64, build with `GOARCH=arm64` and run on Graviton-backed Fargate (`runtimePlatform: { cpuArchitecture: ARM64, operatingSystemFamily: LINUX }`). Same 20% cost benefit as Lambda.

### 9.2 Lifecycle and signals

ECS sends SIGTERM, then waits `stopTimeout` (default 30s; max 120s) before SIGKILL. Your service must:

1. Trap SIGTERM.
2. Stop accepting new requests (close the listener via `srv.Shutdown(ctx)`).
3. Drain in-flight requests up to a deadline.
4. Close DB pools, flush observability buffers.
5. Return.

Set `stopTimeout` explicitly (30-60s for most services). The default 30s is often too short for HTTP services with longer-running requests.

### 9.3 Health checks

Two different concerns, two different endpoints:

- `/healthz` — **liveness**. Returns 200 if the process is alive. No DB hit. Should fail only if the process is hung (deadlock, infinite loop). The container orchestrator restarts the container on liveness failure.
- `/readyz` — **readiness**. Returns 200 if the service can serve requests. Pings the DB, checks critical downstream deps. Returns 503 if not ready (during startup, after DB outage, etc.). The load balancer stops sending traffic on readiness failure.

Map the ALB target group health check to `/readyz`. Map the orchestrator liveness probe (Kubernetes) to `/healthz`. ECS Fargate doesn't distinguish; just point its check at `/readyz` with appropriate intervals.

### 9.4 IAM: task role vs execution role

**Execution role**: what ECS itself uses to do its job — pull the image from ECR, write logs to CloudWatch, fetch secrets from Secrets Manager (if referenced via the task definition's `secrets:` block), pull container insights data.

**Task role**: what your code uses for AWS API calls (S3 puts, DynamoDB queries, SQS sends, etc.).

Distinct roles. Lock both to least privilege:

- Execution role: only the `AmazonECSTaskExecutionRolePolicy` plus permission to read the specific secrets your task needs.
- Task role: only the specific actions on specific resources your code touches.

Reusing one role for both is a common anti-pattern that leads to over-privileged tasks.

### 9.5 Secrets

Reference Secrets Manager / SSM Parameter Store from the task definition:

```json
{
  "secrets": [
    { "name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db/url-AbCdEf" },
    { "name": "STRIPE_KEY",  "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/prod/stripe/key" }
  ]
}
```

ECS injects them as environment variables at task start. Your code reads them via `os.Getenv`. **Don't** fetch from Secrets Manager from Go code on every request — that's slow and runs up the cost.

If your secret rotates and you need to pick up the new version without restarting, you can poll Secrets Manager periodically from Go (cache in memory, refresh every 5 minutes). Most services don't need this — rotate secrets via a deploy.

### 9.6 Logs

Default: write JSON to stdout, use the `awslogs` log driver to ship to CloudWatch:

```json
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-group": "/ecs/my-service",
    "awslogs-region": "us-east-1",
    "awslogs-stream-prefix": "api"
  }
}
```

For high-volume services, CloudWatch ingestion costs add up fast (~$0.50/GB ingested, ~$0.03/GB stored, plus query costs). Switch to FireLens with Fluent Bit and ship to S3 (for archival), OpenSearch (for queries), or Kinesis Firehose (for downstream pipelines). FireLens is a sidecar container in the same task definition.

### 9.7 Capacity model

**Fargate** (serverless, AWS manages the host):

- Easier ops. No instances to patch, scale, or capacity-plan.
- Slightly more expensive per CPU-hour than EC2.
- **Fargate Spot**: 70% cheaper. Two-minute interruption warning. Use for batch jobs, async workers, anything tolerant of restart. Don't use for synchronous user-facing services.

**EC2** (you provide the instances):

- More control. Cheaper per CPU at scale.
- You manage the AMI, patches, scaling, capacity reservations.
- Right when scale > Fargate cost crossover (typically a few hundred CPU-hours/month per service).

For most teams, default to Fargate; switch to EC2 when cost or scale demands.

### 9.8 Auto-scaling

Scale on the right signal:

- **CPU-bound services**: target tracking on CPU utilization (e.g., 70%).
- **Memory-bound services**: target tracking on memory utilization.
- **HTTP services**: ALB request count per target, or target response time.
- **Worker services consuming SQS**: SQS queue depth (`ApproximateNumberOfMessagesVisible`).

Always set min capacity ≥ 2 across at least 2 AZs for HA. Set sensible max capacity to bound runaway costs.

Scale-out cooldowns shorter than scale-in cooldowns — easier to add capacity than to interrupt active requests. Typical: 60s cooldown out, 300s cooldown in.

---

## 10. Configuration and Secrets

Read configuration from environment variables at startup. Validate. Fail fast.

```go
type Config struct {
    Port            int           `env:"PORT" envDefault:"8080"`
    DatabaseURL     string        `env:"DATABASE_URL,required"`
    AWSRegion       string        `env:"AWS_REGION" envDefault:"us-east-1"`
    LogLevel        string        `env:"LOG_LEVEL" envDefault:"info"`
    OtelEndpoint    string        `env:"OTEL_EXPORTER_OTLP_ENDPOINT"`
    ShutdownTimeout time.Duration `env:"SHUTDOWN_TIMEOUT" envDefault:"30s"`
}

func Load() (Config, error) {
    var c Config
    if err := env.Parse(&c); err != nil {
        return Config{}, fmt.Errorf("env: %w", err)
    }
    if c.Port < 1 || c.Port > 65535 {
        return Config{}, fmt.Errorf("invalid port %d", c.Port)
    }
    return c, nil
}
```

Use a typed loader: `caarlos0/env`, `kelseyhightower/envconfig`, or `sethvargo/go-envconfig`. They handle defaults, required fields, and type conversion (durations, URLs, slices). Avoid hand-rolled `os.Getenv` + manual parsing in scattered places.

**Don't read `os.Getenv` outside `internal/config`.** Scattered env reads make config dependencies invisible and untestable. One source of truth.

Secrets stay in AWS Secrets Manager or SSM Parameter Store SecureString. Reference them from the platform (Lambda env mapping, ECS task definition `secrets`). Inject them as env vars. Your Go code doesn't know whether a value came from a secret store or a plain env var.

**Don't log config.** `log.Printf("config: %+v", cfg)` is a recurring CVE pattern. Implement `String()` or `LogValue()` to redact:

```go
func (c Config) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Int("port", c.Port),
        slog.String("aws_region", c.AWSRegion),
        slog.String("log_level", c.LogLevel),
        slog.String("database_url", "<redacted>"),
    )
}
```

Commit `.env.example` with placeholder values. Never commit `.env`.

---

## 11. Logging with `slog`

`log/slog` is in the standard library since Go 1.21 and is the right default for new services. Three core types:

- `*slog.Logger` — the API you call (`Info`, `Error`, `InfoContext`, etc.).
- `slog.Handler` — does the actual formatting / output (`TextHandler` for dev, `JSONHandler` for prod).
- `slog.Record` — the structured log entry passed between them.

### 11.1 JSON in production

```go
opts := &slog.HandlerOptions{
    Level:     slog.LevelInfo,
    AddSource: false,
    ReplaceAttr: redactSecrets,
}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
```

JSON because every log aggregator (CloudWatch Logs Insights, Datadog, Loki, Splunk, OpenSearch) expects structured input. Plain text means every dashboard query is a regex.

### 11.2 Pass the logger, don't use the global

In `main`, set the package logger and pass it down:

```go
type Service struct {
    store  Store
    logger *slog.Logger
}

func NewService(store Store, logger *slog.Logger) *Service {
    return &Service{store: store, logger: logger.With("component", "user")}
}
```

`slog.SetDefault(logger)` is fine for `main` and tests. Don't reach for `slog.Default()` inside libraries — it makes test setup harder and surprising for callers.

### 11.3 Use the `*Context` variants

```go
logger.InfoContext(ctx, "user fetched", "user_id", id, "duration_ms", ms)
```

`InfoContext` lets a custom `Handler` extract values from `ctx` (request ID, trace ID, principal). The OpenTelemetry slog bridge uses this to attach trace / span IDs to every log line — invaluable for jumping from a log to the corresponding trace.

### 11.4 Key-value pairs, not formatted strings

```go
slog.Info("user fetched", "user_id", id, "email", email)         // good
slog.Info(fmt.Sprintf("user %s fetched (%s)", id, email))         // bad
```

The JSON handler indexes the attributes; queries can filter on `user_id`. Formatted strings can't be queried structurally.

For a fixed group of attributes you'll use repeatedly:

```go
log := logger.With("user_id", id, "tenant", tenant)
log.Info("fetched")
log.Info("updated", "field", "email")
```

### 11.5 Levels

- `Debug`: dev-only detail. Off in production.
- `Info`: normal operations — request started/finished, lifecycle events.
- `Warn`: recoverable anomaly — retry succeeded after retry, fallback used, deprecated path hit.
- `Error`: genuine failure — request 500'd, downstream call failed after retries, data corrupted.

**Don't log everything as Error.** If your error log volume is 1% of total log volume, alerting on errors is meaningful. If it's 50%, no one looks at the alerts.

### 11.6 Redact secrets

Three approaches, in order of preference:

1. **`LogValue()` on the type**: types containing secrets implement `slog.LogValuer` and return a redacted representation.
2. **`ReplaceAttr` in `HandlerOptions`**: the handler intercepts every attribute and redacts based on key (`password`, `secret`, `authorization`, etc.).
3. **Code review**: never sufficient on its own — humans miss things.

Never log `Authorization` headers, request bodies for auth endpoints, full credit card numbers, full SSNs, or anything else regulated.

---

## 12. Observability: OpenTelemetry and ADOT

### 12.1 Why OpenTelemetry

OpenTelemetry is the CNCF-blessed, vendor-neutral standard for traces, metrics, and (increasingly) logs. It replaced a fragmented landscape of vendor-specific SDKs (Jaeger client, Zipkin client, Prometheus client, Datadog client, ...) with a single instrumentation API and a wire protocol (OTLP).

For Go, the SDK is `go.opentelemetry.io/otel` plus per-signal sub-modules. Auto-instrumentation libraries exist for HTTP (`otelhttp`), gRPC (`otelgrpc`), pgx (`otelpgx`), AWS SDK v2 (`otelaws`), Kafka, Redis, etc.

AWS X-Ray transitioned to ingest OTLP. AWS Distro for OpenTelemetry (ADOT) is the AWS-supported collector that exports to X-Ray, CloudWatch Metrics, Prometheus Workspace, and any OTLP-compatible backend (Datadog, Honeycomb, Grafana Cloud, New Relic, Splunk Observability).

For new code, **use OpenTelemetry, not the X-Ray SDK**. The X-Ray SDK is on a deprecation trajectory.

### 12.2 Initialization

```go
func initOTel(ctx context.Context, cfg Config) (func(context.Context) error, error) {
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(cfg.ServiceName),
            semconv.ServiceVersion(cfg.Version),
            semconv.DeploymentEnvironmentName(cfg.Env),
        ),
    )
    if err != nil { return nil, err }

    traceExp, err := otlptracegrpc.New(ctx, otlptracegrpc.WithEndpoint(cfg.OtelEndpoint))
    if err != nil { return nil, err }
    tp := trace.NewTracerProvider(
        trace.WithBatcher(traceExp),
        trace.WithResource(res),
        trace.WithSampler(trace.ParentBased(trace.TraceIDRatioBased(0.05))),
    )
    otel.SetTracerProvider(tp)

    metricExp, err := otlpmetricgrpc.New(ctx, otlpmetricgrpc.WithEndpoint(cfg.OtelEndpoint))
    if err != nil { return nil, err }
    mp := metric.NewMeterProvider(
        metric.WithReader(metric.NewPeriodicReader(metricExp)),
        metric.WithResource(res),
    )
    otel.SetMeterProvider(mp)

    return func(ctx context.Context) error {
        return errors.Join(tp.Shutdown(ctx), mp.Shutdown(ctx))
    }, nil
}
```

`defer otelShutdown(shutdownCtx)` in `main` to flush before exit.

### 12.3 Auto-instrumentation

Wrap the HTTP server, the HTTP client, the database driver, the AWS clients:

```go
mux := http.NewServeMux()
// ... register handlers ...
handler := otelhttp.NewHandler(mux, "api")

httpClient := &http.Client{Transport: otelhttp.NewTransport(http.DefaultTransport)}

// pgx
pool, _ := pgxpool.NewWithConfig(ctx, poolCfg)
poolCfg.ConnConfig.Tracer = otelpgx.NewTracer()

// aws
awsv2.AppendMiddlewares(&cfg.APIOptions) // from go.opentelemetry.io/contrib/instrumentation/github.com/aws/aws-sdk-go-v2/otelaws
```

These add spans, propagate trace context across the wire, and emit standard semantic-convention attributes (`http.method`, `http.route`, `db.statement`, `aws.service`, etc.).

### 12.4 Manual spans for business operations

Auto-instrumentation gets you HTTP / DB / AWS calls. For business operations that span multiple of those calls, add a span:

```go
ctx, span := tracer.Start(ctx, "checkout.complete")
defer span.End()
span.SetAttributes(
    attribute.String("user.id", user.ID.String()),
    attribute.Int64("cart.value_cents", cart.TotalCents),
)

if err := svc.charge(ctx, user, cart); err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
    return err
}
```

Keep span attribute cardinality reasonable. Including raw request bodies or unbounded strings as attributes blows up your trace storage.

### 12.5 Metrics

Don't paper over missing logs with metrics or vice versa — they answer different questions. Logs answer "what happened on this request?"; metrics answer "what's the trend across all requests?".

A baseline set of metrics every HTTP service should emit:

- `http.server.request.duration` — histogram, by route + method + status.
- `http.server.active_requests` — gauge.
- `db.client.operation.duration` — histogram, by operation + table.
- `aws.client.call.duration` — histogram, by service + operation.
- Business counters: orders processed, signups, payments, errors by kind.

Auto-instrumentation libraries emit the first four. Business counters you add yourself.

### 12.6 ADOT deployment

**ECS Fargate**: the ADOT collector is a sidecar container in the same task definition. Both containers share localhost networking, so your app exports OTLP to `localhost:4317` (gRPC) or `:4318` (HTTP). The collector forwards to X-Ray, CloudWatch Metrics, Prometheus Workspace, etc.

**Lambda**: ADOT is a Lambda layer. Add it to your function, set `OPENTELEMETRY_COLLECTOR_CONFIG_FILE` env var to point at a config file in the layer, and the collector runs in-process during your function's execution.

**EKS**: ADOT runs as a DaemonSet (one per node) or sidecar. Apps export OTLP to the local collector endpoint.

### 12.7 Sampling

Tracing every request at high traffic is expensive and rarely necessary. Use parent-based sampling at 1-10% in production for high-traffic services; 100% for low-traffic services where every trace is useful. Always-trace error paths separately if your backend supports tail sampling.

---

## 13. Validation and Input

Validate at the **edge**: HTTP handler, Lambda handler, SQS consumer. Reject early, return 400 with a clear shape. Bad data doesn't reach the service or store layer.

For HTTP, define request structs adjacent to the handler:

```go
type createUserRequest struct {
    Email string `json:"email" validate:"required,email,max=255"`
    Name  string `json:"name"  validate:"required,min=1,max=100"`
    Age   int    `json:"age"   validate:"gte=0,lte=150"`
}
```

Validate with `go-playground/validator`:

```go
var validate = validator.New(validator.WithRequiredStructEnabled())

if err := validate.Struct(req); err != nil {
    writeValidationError(w, err)
    return
}
```

For complex domain validation that doesn't fit struct tags, expose a method:

```go
func (r createUserRequest) Validate() error {
    if r.Email == "admin@example.com" {
        return errors.New("reserved email")
    }
    return nil
}
```

Constrain sizes everywhere:

- `http.MaxBytesReader` for the request body (1 MiB typical).
- Max string length checked via `len(s)` (in bytes) or `utf8.RuneCountInString(s)` (in runes).
- Cap slice / map sizes from user input.

Untrusted input into other contexts — escape:

- SQL: parameter placeholders only (covered above).
- Shell: never `exec.Command("sh", "-c", input)`; use separate args.
- HTML: `html/template` (auto-escapes); never `text/template` for HTML output.
- URLs: `net/url.QueryEscape` for query params, full URL parse + validate for user-supplied URLs.

Treat **all** external input as untrusted: HTTP bodies, query / path params, headers, cookies, env vars in pipelines, S3 object contents, SQS message bodies, even file paths from clients.

---

## 14. Error Handling at Service Boundaries

Errors travel through the service in layers. Each layer adds context relevant to that layer:

- **Repository / store**: "select user id=42: pgx: no rows".
- **Service**: "user.GetByID id=42: select user id=42: pgx: no rows".
- **Handler**: maps the error to an HTTP response.

The handler is the one that knows about HTTP. Service and store layers return wrapped or sentinel errors; the handler inspects with `errors.Is` / `errors.As` and produces the response:

```go
func (a *API) handleGetUser(w http.ResponseWriter, r *http.Request) {
    u, err := a.users.GetByID(r.Context(), id)
    switch {
    case errors.Is(err, user.ErrNotFound):
        writeError(w, 404, "user.not_found", "user does not exist")
    case errors.Is(err, user.ErrForbidden):
        writeError(w, 403, "user.forbidden", "not allowed")
    case err != nil:
        a.logger.ErrorContext(r.Context(), "get user", "err", err, "user_id", id)
        writeError(w, 500, "internal", "an internal error occurred")
    default:
        writeJSON(w, 200, toUserDTO(u))
    }
}
```

Public error messages must not leak internal information. The handler logs the full error chain (with the request ID) and returns a sanitized message. Operators investigating a production issue match the request ID from the client back to the log.

`panic` recovery in middleware turns one panic into a 500, not a process crash. Always include this middleware in production:

```go
func recoverMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if rec := recover(); rec != nil {
                    logger.ErrorContext(r.Context(), "panic",
                        "recover", fmt.Sprintf("%v", rec),
                        "stack", string(debug.Stack()))
                    writeError(w, 500, "internal", "an internal error occurred")
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}
```

---

## 15. Security

A non-exhaustive list. Treat each one as non-negotiable.

### 15.1 Injection

- **SQL**: parameter placeholders. sqlc + pgx make injection impossible if you don't reach for `fmt.Sprintf`. `gosec G201`/`G202` flag string-built SQL.
- **Command**: never `exec.Command("sh", "-c", input)`. Always separate args (`exec.Command(cmd, "-flag", value)`); validate `cmd` against an allow-list. `gosec G204`.
- **Template injection**: `html/template`, never `text/template` for HTML output.
- **LDAP, NoSQL, log injection**: same principle — escape per the target context.

### 15.2 Path traversal

```go
func resolveSafe(base, rel string) (string, error) {
    full := filepath.Join(base, rel)
    cleaned := filepath.Clean(full)
    rel2, err := filepath.Rel(base, cleaned)
    if err != nil || strings.HasPrefix(rel2, "..") {
        return "", errors.New("path escapes base")
    }
    return cleaned, nil
}
```

Never `os.Open(userInput)`. Never `os.ReadFile(filepath.Join(base, userInput))` without the check above.

### 15.3 SSRF

User-supplied URLs that your backend fetches are SSRF risk. Validate:

- Scheme must be `https` (or `http` only for explicitly local environments).
- Host must be in an allow-list, OR resolved IP must be public (not RFC 1918 / link-local / loopback).

For the latter, use a custom `http.Client` with a `DialContext` that rejects private IPs:

```go
var safeDialer = &net.Dialer{
    Control: func(network, address string, c syscall.RawConn) error {
        host, _, _ := net.SplitHostPort(address)
        ip := net.ParseIP(host)
        if ip == nil { return errors.New("non-IP address") }
        if ip.IsPrivate() || ip.IsLoopback() || ip.IsLinkLocalUnicast() {
            return errors.New("address is private")
        }
        return nil
    },
}
```

### 15.4 TLS

`crypto/tls` defaults are sane in modern Go but specify `MinVersion: tls.VersionTLS12` (or `13` where you control both ends). Never `InsecureSkipVerify: true` outside tests — it disables certificate validation entirely.

For mTLS to internal services, use `tls.Config.Certificates` and `tls.Config.RootCAs` with your internal CA bundle.

### 15.5 Cryptography

- **Random tokens / IDs / nonces**: `crypto/rand`, never `math/rand` (predictable). `crypto/rand.Read(buf)` for raw bytes; `uuid.NewRandom()` for UUIDs.
- **Password hashing**: `argon2id` (`golang.org/x/crypto/argon2`) is the modern best practice. `bcrypt` (cost ≥ 12) is acceptable. `scrypt` is fine. Never SHA / MD5 / PBKDF2-SHA1 for passwords.
- **Symmetric encryption**: AES-GCM via `crypto/aes` + `crypto/cipher`. Never CBC mode. Never reuse a nonce.
- **Asymmetric**: Ed25519 for signing (faster + safer than RSA / ECDSA); RSA-2048 minimum for legacy interop. Use `crypto/ecdsa` only when interoperating with X.509 ECDSA chains.

### 15.6 Sessions and tokens

Default to **opaque session IDs in httpOnly Secure SameSite=Strict cookies** backed by a server-side store (Redis, DynamoDB). This is the simplest, safest pattern: revocation is trivial (delete the row), the cookie can't be read by JS (httpOnly), can't leak over HTTP (Secure), can't be sent in cross-site requests (SameSite=Strict).

If you need stateless tokens (federated services, mobile clients without cookies), use JWT with care:

- **Asymmetric algorithms only**: RS256 or Ed25519 (EdDSA). Avoid HS256 — symmetric secrets get leaked, and a leaked HS256 secret means anyone can mint tokens.
- **Force the expected algorithm** when verifying. Never trust the `alg` from the token header — that's the alg-confusion attack.
- **Verify all standard claims**: `aud`, `iss`, `exp`, `nbf` (and `iat` for staleness), plus your own claims.
- **Short expiry** (15 min - 1 hour) with refresh tokens for longer sessions. Revocation requires a server-side blocklist or short token lifetime; pick one.

### 15.7 Secrets in source / logs

- Pre-commit hook: `gitleaks` or `trufflehog`. If it lands in git history, treat it as compromised — rotate, even if you delete the commit.
- `gosec G101` flags hard-coded credentials.
- Never log `Authorization` headers, request bodies for auth endpoints, full credit card numbers, full SSNs, OAuth tokens, API keys.

### 15.8 Dependencies

- **`govulncheck`** in CI per PR. Unlike a generic SCA tool, it walks your call graph and only flags vulnerabilities in code paths you actually call. Critical/high are release blockers.
- **`gosec`** for SAST. Integrate in CI.
- **Dependabot / Renovate** for dependency updates. One major bump per PR.
- Image scan with **Trivy** or **Grype** on every container build. Scan deployed images on a schedule — new CVEs land daily.

### 15.9 Race conditions

`go test -race ./...` in CI on every PR. A data race is a security concern: time-of-check / time-of-use bugs, partially-constructed objects passed across goroutines, panics from concurrent map writes — all start as data races and end as exploitable bugs.

---

## 16. Testing

### 16.1 stdlib `testing` is enough

Go's standard testing package handles unit tests, subtests, parallel tests, benchmarks, examples, and (since 1.18) fuzzing. You don't need a framework. `stretchr/testify` (`require`, `assert`) is the most popular addition and is fine — it makes assertions terser. Many teams prefer plain `if got != want { t.Fatalf(...) }` for the explicitness.

Avoid BDD frameworks (Ginkgo, Convey). They add an extra DSL for no benefit on top of subtests + table-driven tests.

### 16.2 Table-driven tests

The idiomatic Go testing pattern:

```go
func TestUserService_GetByID(t *testing.T) {
    t.Parallel()
    cases := []struct {
        name    string
        seed    map[uuid.UUID]User
        id      uuid.UUID
        want    User
        wantErr error
    }{
        {"found", map[uuid.UUID]User{id1: {ID: id1, Email: "a@b.c"}}, id1, User{ID: id1, Email: "a@b.c"}, nil},
        {"not found", nil, id1, User{}, ErrNotFound},
    }
    for _, tc := range cases {
        tc := tc
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            svc := NewService(&fakeStore{users: tc.seed})
            got, err := svc.GetByID(context.Background(), tc.id)
            if !errors.Is(err, tc.wantErr) { t.Fatalf("err: got %v, want %v", err, tc.wantErr) }
            if got != tc.want { t.Errorf("user: got %+v, want %+v", got, tc.want) }
        })
    }
}
```

Always use `t.Run` for subtests. It gives:

- Named test output (clearer failures).
- Selective execution (`go test -run TestUserService_GetByID/found`).
- Parallel control per subtest (`t.Parallel()` inside each subtest).

`t.Parallel()` everywhere safe. It surfaces hidden ordering dependencies and cuts CI time. The `tc := tc` capture was needed pre-Go 1.22 because of loop variable scoping; from 1.22 onward, `for` loops give each iteration its own variable, so the capture is no longer required (but harmless).

### 16.3 Behavior, not implementation

Test through the public interface. Resist the temptation to test internal helpers that aren't part of the public contract — they change with refactors and break tests that don't reflect actual behavior change.

Two file conventions for tests:

- **`pkg_test` package** (black-box): test file declares `package pkg_test`, can only import the public API. Reflects what real consumers see.
- **same `pkg` package** (white-box): test file declares `package pkg`. Useful when you genuinely need to test a private helper (e.g., a complex algorithm with no public surface).

Use white-box sparingly.

### 16.4 Fakes vs mocks

For most cases, **hand-written fakes** are better than generated mocks. A fake is a real implementation of the interface, just simpler:

```go
type fakeUserStore struct{ mu sync.Mutex; users map[uuid.UUID]User }

func (s *fakeUserStore) GetByID(_ context.Context, id uuid.UUID) (User, error) {
    s.mu.Lock(); defer s.mu.Unlock()
    u, ok := s.users[id]
    if !ok { return User{}, ErrNotFound }
    return u, nil
}
```

Versus a generated mock that records calls and returns canned values. Reasons fakes win:

- Tests assert on outcomes ("after Create + GetByID, the user has these fields"), not on call sequences ("Create was called once with these args").
- Refactors that change the call sequence don't break tests if the outcome is unchanged.
- Fakes catch interaction bugs the implementation actually has, not just the ones the test author thought to mock.

Generated mocks (`mockery`, `gomock`) are appropriate when:

- The interface is large and stable, and writing a fake means many no-op methods.
- You need to assert on call sequences (rare; usually a sign of over-coupling).

**Never mock what you don't own.** Don't mock `*pgxpool.Pool` directly — wrap it in your `Store` interface and mock that. Wrappers around third-party types let you migrate to a different library without rewriting tests.

### 16.5 Integration tests

Unit tests cover service logic with fakes. Integration tests cover the wiring with real dependencies — Postgres, Redis, S3 (via LocalStack), etc.

`testcontainers-go` spins up Docker containers from inside a test:

```go
//go:build integration

package user_test

func TestUserStore_Postgres(t *testing.T) {
    ctx := context.Background()
    pgC, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("test"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(wait.ForLog("ready to accept connections")),
    )
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { _ = pgC.Terminate(ctx) })

    dsn, _ := pgC.ConnectionString(ctx, "sslmode=disable")
    pool, _ := pgxpool.New(ctx, dsn)
    // run migrations, create store, assert behavior
}
```

Gate integration tests with the `//go:build integration` build tag so they're skipped from `go test ./...` by default. Run them in CI separately: `go test -tags=integration ./...`. Use `TestMain` if you want one container shared across many tests in a package.

### 16.6 HTTP handler tests

```go
func TestAPI_GetUser(t *testing.T) {
    api := &API{users: NewService(&fakeStore{...}), logger: discardLogger}
    req := httptest.NewRequest("GET", "/users/"+id1.String(), nil)
    req.SetPathValue("id", id1.String()) // for ServeMux path values
    rr := httptest.NewRecorder()
    api.handleGetUser(rr, req)

    if rr.Code != 200 { t.Fatalf("status: got %d, want 200", rr.Code) }
    // assert on rr.Body
}
```

`httptest.NewRecorder` captures the response; `httptest.NewServer` wraps a handler in a real listener for end-to-end-style tests.

### 16.7 Coverage and other commands

```bash
go test -race -coverprofile=cov.out -covermode=atomic ./...
go tool cover -func=cov.out
go tool cover -html=cov.out -o cov.html
```

Don't chase 100%. Chase critical paths and edge cases. Coverage on `cmd/` wiring is usually low and that's fine. Coverage on `internal/<domain>/` should be high (70%+ is a reasonable gate).

`-race` is non-negotiable in CI. `-shuffle=on` randomizes test order — surfaces shared-state bugs.

### 16.8 Fuzzing

Built-in since Go 1.18. Use for parsers, validators, anything taking untrusted bytes:

```go
func FuzzParseEmail(f *testing.F) {
    f.Add("user@example.com")
    f.Fuzz(func(t *testing.T, in string) {
        e, err := parseEmail(in)
        if err == nil && e.Domain == "" {
            t.Errorf("accepted email with empty domain: %q", in)
        }
    })
}
```

Run locally during dev (`go test -fuzz=FuzzParseEmail ./...`); in CI, fuzz tests run as regular tests (against the seed corpus) — actual fuzzing usually happens on dedicated infrastructure.

---

## 17. Build, Cross-Compile, Deploy

### 17.1 Build

`go build` is enough. A `Makefile` is a fine entry point for `lint`, `test`, `build`, `docker`, `deploy` — CI replays Makefile targets so dev and CI commands stay aligned.

Typical build command for production:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 \
go build \
  -trimpath \
  -ldflags "-s -w -X main.version=${VERSION} -X main.commit=${COMMIT} -X main.buildTime=${BUILD_TIME}" \
  -o bin/api ./cmd/api
```

- `-trimpath`: remove filesystem paths (reproducibility, smaller binary, no leaked dev paths).
- `-ldflags "-s -w"`: strip symbol table and DWARF debug info (smaller binary; can't `delve` against it).
- `-ldflags "-X main.version=..."`: inject build-time strings into package vars. Enables `myapp version` to print the actual build info.
- `CGO_ENABLED=0`: produces a fully static binary — no libc dep, runs on `FROM scratch`.

### 17.2 Cross-compile

Go's cross-compilation is built in. From a macOS arm64 laptop or x86 Linux CI runner, build for Linux arm64:

```bash
GOOS=linux GOARCH=arm64 go build ...
```

No emulation, no Docker required. This is faster than building inside a Docker container on the target arch via QEMU.

### 17.3 Docker

Multi-stage build, distroless final image:

```dockerfile
FROM golang:1.24-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
ARG VERSION=dev
ARG COMMIT=unknown
RUN CGO_ENABLED=0 GOOS=linux GOARCH=arm64 \
    go build -trimpath \
      -ldflags "-s -w -X main.version=${VERSION} -X main.commit=${COMMIT}" \
      -o /out/api ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/api /api
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/api"]
```

Why distroless `static-debian12:nonroot`:

- Includes CA certificates, `/etc/passwd` for the nonroot user, basic timezone data — enough for most Go services to make outbound HTTPS calls.
- No shell, no package manager — minimal attack surface.
- Maintained by Google with regular CVE patching.

`FROM scratch` is even smaller but you lose the CA bundle (you'd need to `COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`) and the user setup. Distroless gives you both for ~10 MB extra.

### 17.4 Image scanning

Trivy (`aquasecurity/trivy`) or Grype (`anchore/grype`) in CI. Scan on every build:

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE}
```

Critical CVEs in the **final image** (not the build stage) are blockers. Schedule re-scans of deployed images — a CVE disclosed today applies to images you built last week.

### 17.5 Deploy

Container images for ECS / EKS / EC2; zip for Lambda. Always tag with the Git SHA:

```bash
IMAGE=${ECR_URL}/api:${GIT_SHA}
docker build -t ${IMAGE} .
docker push ${IMAGE}
aws ecs update-service --cluster prod --service api --force-new-deployment
```

Never deploy `latest` to production. The SHA tag is your one path to "what version is actually running?".

---

## 18. Tooling

### 18.1 Required

- **`gofmt` / `goimports`**: formatting + import grouping. Run on save. CI fails on drift.
- **`go vet`**: built-in static analysis. Catches printf format errors, unreachable code, etc.
- **`golangci-lint`**: meta-linter. Pin the version in CI. Recommended config (`.golangci.yml`):

```yaml
linters:
  enable:
    - errcheck       # unchecked errors
    - govet
    - staticcheck    # bug-finder
    - unused         # dead code
    - ineffassign
    - gosimple
    - revive         # replaces golint
    - gosec          # security
    - gocyclo        # cyclomatic complexity
    - misspell
    - gocritic       # pattern checks
    - errorlint      # %w wrapping, errors.Is/As
    - nilerr         # returning nil error when err != nil
    - bodyclose      # http response bodies closed
    - containedctx   # context in struct fields
    - contextcheck   # context propagation
    - noctx          # http request without context
    - sqlclosecheck  # rows/stmt closed
    - tparallel      # t.Parallel misuse
    - paralleltest   # missing t.Parallel
linters-settings:
  gocyclo: { min-complexity: 15 }
  gosec:
    excludes: [G104] # too noisy in tests
```

### 18.2 Security tooling

- **`govulncheck ./...`**: native vuln scanner. Walks the call graph — only flags vulns you actually call. Run in CI per PR; treat critical/high as blockers.
- **`gosec ./...`**: SAST for Go. Catches SQL injection patterns, hard-coded credentials, weak crypto, missing TLS verification, etc.

### 18.3 Profiling

- **`pprof`**: Go's built-in profiler. Expose via `net/http/pprof` on a separate admin listener (never the public port):

```go
adminMux := http.NewServeMux()
adminMux.Handle("/debug/pprof/", http.DefaultServeMux)
go func() { _ = http.ListenAndServe("localhost:6060", adminMux) }()
```

Capture profiles in production (under load):

```bash
go tool pprof -http=:9000 http://service-host:6060/debug/pprof/profile?seconds=30
go tool pprof -http=:9000 http://service-host:6060/debug/pprof/heap
```

The flame graph is usually all you need. Optimize the top 1-2 boxes; ignore the rest.

### 18.4 Benchmarks

```bash
go test -bench=. -benchmem -count=10 -run=^$ ./internal/foo > old.txt
# make change
go test -bench=. -benchmem -count=10 -run=^$ ./internal/foo > new.txt
benchstat old.txt new.txt
```

`benchstat` (from `golang.org/x/perf`) computes statistical significance. A single-run benchmark comparison is meaningless. `-count=10` and `benchstat` give you the answer.

### 18.5 IDE and dev loop

- **`gopls`**: official LSP. Works with VS Code, Neovim, Emacs, JetBrains. Enable `gopls.staticcheck: true` for inline staticcheck warnings.
- **`delve`**: debugger. Step-through, breakpoints, goroutine inspection.
- **`air`** / **`wgo`** / **`reflex`**: hot-reload for dev — rebuild and restart on file change. Optional; many devs prefer just `go run`.
- **`task`** (Taskfile): Make alternative with cleaner YAML syntax. Optional.
- **`goreleaser`**: produces multi-arch / multi-OS release artifacts (binaries, Docker images, Homebrew taps, etc.). Useful for CLIs and self-hosted services.

---

## 19. Performance and Resource Tuning

### 19.1 Profile first

Optimize from data, not intuition. The full loop:

1. Identify a hot path (slow endpoint, expensive batch job).
2. Reproduce locally with a benchmark or load generator.
3. Profile (`pprof` cpu + heap + alloc).
4. Make one change.
5. Re-measure with `benchstat`.
6. Commit if better; revert otherwise.

Most "obvious" optimizations don't help. Most help comes from removing allocations or reducing algorithmic complexity, not from micro-optimizations.

### 19.2 GOMAXPROCS in containers

`GOMAXPROCS` defaults to the number of CPU cores **the host** sees. On Kubernetes / ECS where your container has a CPU limit (say, 1 core) but the host has 96 cores, `GOMAXPROCS` defaults to 96 — Go's scheduler thinks it has 96 OS threads available, runs into the cgroup CPU limit constantly, gets throttled, exhibits high tail latency.

The fix: `go.uber.org/automaxprocs`. Import in `main`:

```go
import _ "go.uber.org/automaxprocs"
```

It reads the cgroup CPU quota at startup and sets `GOMAXPROCS` accordingly. Single line; massive impact on tail latency in containerized environments.

### 19.3 GOMEMLIMIT

Go 1.19+ added `GOMEMLIMIT`, a soft memory limit the GC respects. Set it to ~80% of the container memory limit. The runtime will GC harder as memory approaches the limit — dramatically reducing OOMKills under load spikes.

Set via env in the task / pod definition:

```
GOMEMLIMIT=400MiB     # for a container with memory limit ~512MiB
```

You can use suffixes: `B`, `KiB`, `MiB`, `GiB`. The runtime treats this as a target, not a hard cap (it can exceed it transiently).

### 19.4 GOGC

The classic GC pacer setting: GC triggers when heap doubles (`GOGC=100`). Tuning:

- `GOGC=50` — GC more often, lower memory ceiling, slightly more CPU on GC.
- `GOGC=200` — GC less often, higher memory ceiling, slightly less CPU on GC.

With `GOMEMLIMIT` set, you usually don't need to tune `GOGC` manually. Only do it if profiling shows GC dominating CPU.

### 19.5 Allocations dominate

In most Go services, allocations dominate latency more than raw CPU. `pprof -alloc_objects` finds the hot allocators. Common wins:

- **Pre-size slices and maps** when capacity is known: `make([]T, 0, n)` instead of `[]T{}` then growing.
- **`sync.Pool`** for short-lived objects in hot paths — request buffers, scratch slices, transient structs. Don't use as a cache.
- **`strings.Builder`** instead of `s += "..."` in loops.
- **Avoid `[]byte`↔`string` conversions** in tight loops — they copy. `unsafe.String` and `unsafe.Slice` (Go 1.20+) avoid the copy when you can prove the underlying buffer outlives the conversion.
- **Reuse buffers** via `bytes.Buffer.Reset()` instead of new allocations.

### 19.6 HTTP client tuning

Reuse the `http.Client` (and its underlying `Transport`) across requests — don't allocate per request. The default `http.DefaultTransport` has `MaxIdleConnsPerHost = 2`, which is far too low for backend-to-backend traffic:

```go
transport := &http.Transport{
    MaxIdleConns:        100,
    MaxIdleConnsPerHost: 100,
    IdleConnTimeout:     90 * time.Second,
    TLSHandshakeTimeout: 10 * time.Second,
}
client := &http.Client{Transport: transport, Timeout: 30 * time.Second}
```

Always set `Client.Timeout` — without it, a hung downstream blocks your goroutine forever.

---

## 20. Large Project Maintenance

### 20.1 Dependencies

- **Audit**: `govulncheck ./...` per PR. `go list -m -u all` monthly to see what's stale.
- **Update**: one major bump per PR. Pin versions in `go.mod`. Commit `go.sum`.
- **Trace**: `go mod why -m <module>` to find what pulls a transitive dep in.
- **Prune**: `go mod tidy` after every change.

### 20.2 Dead code

- **`staticcheck`** rule `U1000` flags unused private identifiers.
- **`deadcode`** (`golang.org/x/tools/cmd/deadcode`) finds cross-package unreachable code.
- Don't keep "in case we need it later" — Git remembers everything you delete.

### 20.3 Refactoring

Refactor only when:

- The current task requires it (you're touching the code anyway and the existing shape is in your way).
- You've been explicitly asked.

One thing per PR. Tests first — refactoring with thin tests is changing behavior with no safety net. Flag tech debt to the team rather than silently fixing — silent fixes destroy `git blame` for the original author and surprise reviewers.

### 20.4 Migration patterns

For non-trivial changes (auth library swap, DB migration, framework upgrade), do it incrementally:

1. New behavior behind a flag (env var, feature flag service).
2. Old and new code coexist.
3. Roll out the flag gradually (1% → 10% → 50% → 100%).
4. After full rollout and a bake period, remove the old code in a follow-up PR.

Document the canonical pattern in `CLAUDE.md` so it doesn't drift.

### 20.5 Performance budgets

Set SLOs per endpoint (p50, p95, p99 latency; error rate). Track them in production via your APM. Alert when they regress.

CI benchmark check on hot paths: snapshot benchmarks in the repo, fail PRs that regress beyond a threshold:

```bash
go test -bench=. -count=10 ./hotpath > new.txt
benchstat -delta-test=utest baseline.txt new.txt
```

### 20.6 Module boundaries

Most services should be one Go module. When `internal/` grows past ~10 domain packages, consider splitting:

- Multiple modules in a monorepo with `go.work` (Go 1.18+ workspaces).
- Microservices (separate repos, separate deploys).

Don't split prematurely. The cost of refactoring within a module is much lower than across modules.

---

## 21. New Project Setup

A 30-minute checklist to bootstrap a production-ready Go service:

1. **Create the module.**
   ```bash
   mkdir my-service && cd my-service
   go mod init github.com/myorg/my-service
   ```
   Set `go 1.24` (or current minimum) in `go.mod`.

2. **Skeleton.**
   ```bash
   mkdir -p cmd/api internal/{config,http,user,platform/{database,logger,otelx,awsx}} migrations docs
   ```
   Add `Dockerfile`, `Makefile`, `.golangci.yml`, `.env.example`, `.gitignore` (cover `bin/`, `*.out`, `.env*`, `tmp/`).

3. **Wire `cmd/api/main.go`** with: `slog` JSON logger, `automaxprocs` import, env-driven config, graceful shutdown, `/healthz`, `/readyz`, stdlib `net/http` server with the timeouts set.

4. **Pick infrastructure.**
   - Postgres: `pgx/v5` + `sqlc`. Add `sqlc.yaml`, `db/schema.sql`, `db/queries/`. Run `sqlc generate`.
   - DynamoDB: `aws-sdk-go-v2/service/dynamodb`. Wrap in a `Store` interface in your domain package.
   - Migrations: `golang-migrate`. Versioned SQL in `migrations/`. `cmd/migrate/main.go` to apply.

5. **Add observability.** `go.opentelemetry.io/otel` SDK, OTLP exporter to ADOT collector. `otelhttp` on the server, `otelpgx` on the pool, `otelaws` on AWS clients.

6. **CI** (GitHub Actions / GitLab / CodeBuild):
   - `go vet ./...`
   - `golangci-lint run`
   - `govulncheck ./...`
   - `go test -race -coverprofile=cov.out ./...`
   - `go build -trimpath ./cmd/...`
   - Container build + push to ECR
   - Integration tests: `go test -tags=integration ./...`

7. **Deploy.**
   - Multi-stage Dockerfile → distroless arm64 image.
   - ECS Fargate task: task role + execution role (least privilege, distinct).
   - CloudWatch logs, ALB or API Gateway, auto-scale on CPU + ALB request count.
   - For Lambda: `bootstrap` binary, `provided.al2023` runtime, arm64.

8. **Secrets.**
   - Commit `.env.example` with placeholders.
   - Production secrets in Secrets Manager / SSM Parameter Store.
   - Never commit `.env` or AWS credentials.

---

## 22. Working on an Existing Project

Before making changes:

1. **Read first.** Open `cmd/api/main.go`, the relevant `internal/<domain>/` files, the related handler, the store impl. Look at the `Makefile`, `Dockerfile`, `.golangci.yml` to understand the team's conventions.
2. **Match conventions.** If they use `gin` and you'd use stdlib, use `gin`. Don't migrate without approval — consistency beats theoretical best practice.
3. **Don't refactor unrelated code.** Don't add types/comments to code you didn't change. Don't introduce new libraries without explicit approval.
4. **Trace data flow** for the feature you're touching: handler → service → store. Don't skip layers (handler reaching directly into the store) unless that's the existing pattern.
5. **Audit context propagation.** Every function on the path should take `ctx` and pass it down. Long-running goroutines should listen on `ctx.Done()`.
6. **Audit observability.** New endpoints get spans + metrics + structured logs with the request ID. New downstream calls get traced via `otelhttp` / `otelaws` / `otelpgx`.
7. **Run the local checks** before committing: `go vet`, `golangci-lint run`, `go test -race ./...`, `go mod tidy`.

---

## 23. Pre-Completion Checklist

Match `CLAUDE.md`'s checklist; expanded reasoning here:

- **Format / lint / test pass.** No exceptions. If a step doesn't apply, document why.
- **Doc comments on exported identifiers.** Required by `revive`. Helps `gopls` show docs on hover, and `go doc <pkg>` give useful output.
- **Context propagation.** Every blocking function takes `ctx`. No `context.Background()` in handlers / library code.
- **Errors wrapped.** `%w`, not `%v`. `errors.Is` / `errors.As`, not string compare.
- **No `panic` outside startup / recovery.** `log.Fatal` only in `main`.
- **No string-built SQL.** Parameter placeholders only.
- **All input validated.** Bodies size-capped via `MaxBytesReader`. All other input through `validator` or hand-rolled validation.
- **No secrets in code, env defaults, or logs.** `crypto/rand`, not `math/rand`.
- **HTTP server timeouts set.** Graceful shutdown trapped on SIGTERM.
- **Goroutines have termination paths.** Concurrent map writes guarded.
- **Logs are structured JSON.** Trace IDs propagated. No PII.
- **Spans cover business operations.** HTTP / DB / AWS auto-instrumented.
- **DTOs returned to clients.** No raw domain or DB types.
- **AWS SDK v2 only.** Singletons. Paginators used.
- **Dockerfile multi-stage, distroless / scratch.** Image scanned. arm64 unless blocked.
- **Tests added.** Table-driven, `t.Parallel`, race detector clean.
- **No CI / CD changes without approval.** New deps justified.

---

## 24. CPU Architecture: arm64 vs amd64

The earlier sections default to `arm64` (Graviton2/3) for AWS Lambda and ECS Fargate because, for general-purpose Go workloads, it's about **20% cheaper for duration / vCPU-hour** at **~19% better performance** than x86_64. Most pure-Go services see no downside. Before choosing, walk through this decision:

### 24.1 When arm64 is the right default

- Pure-Go service (no CGO).
- Standard HTTP / gRPC / worker workload.
- Postgres / DynamoDB / Redis / SQS / Kafka — none of which care about your CPU arch.
- Deploying on AWS Lambda (`provided.al2023`), ECS Fargate (Graviton-backed), EKS (Graviton node groups), or any other arm64-capable host.

This covers the vast majority of new backend services. Default to arm64 unless one of the next subsections applies.

### 24.2 When you need amd64 (x86_64)

Pick amd64 when **any** of these is true:

- **CGO dependency without arm64 support.** Some C libraries — older proprietary SDKs, certain database drivers, native ML/CUDA libraries — only ship x86_64 binaries. Attempting to cross-compile against an amd64-only library fails at link time.
- **CUDA / GPU workloads.** Most NVIDIA data-center GPUs in cloud environments (A10, L4, L40S, A100, H100, B200 in AWS `g5`/`g6`/`p4`/`p5`/`p6` instances; Azure `NC`/`ND` series; GCP `A2`/`A3`) are paired with amd64 hosts. NVIDIA's CUDA driver and runtime are first-class on `linux/amd64`; arm64 is supported on Jetson / Grace Hopper but support across CUDA-adjacent libraries (TensorRT, cuDNN versions, third-party CUDA wrappers) lags. If your service is GPU-adjacent, default to amd64.
- **Specific x86 instructions.** AVX-512, AMX, or other Intel/AMD-only ISA features used by a hot path or a hand-tuned dependency. Rare in pure Go (the compiler doesn't emit AVX-512 broadly), but real for some crypto / compression / vector libraries that ship hand-tuned amd64 paths.
- **Customer / compliance constraints.** A customer's on-prem environment is x86-only; a regulated environment has only-x86-validated AMIs; a vendor product you wrap requires x86.
- **Cost equivalence at scale.** For some niche workloads (heavy SIMD, certain in-memory databases) amd64's per-vCPU performance can offset its higher hourly price. Benchmark before assuming.

### 24.3 Cross-compiling to amd64

Same shape as arm64, just swap `GOARCH`:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
go build -trimpath -ldflags "-s -w -X main.version=${VERSION}" \
  -o bin/api ./cmd/api
```

No emulation needed if the host is itself x86 (or if the host is ARM and `CGO_ENABLED=0` — pure Go cross-compiles trivially in either direction).

With CGO enabled (`CGO_ENABLED=1`), you need the cross-compilation toolchain for the target. The cleanest path is **Docker buildx with multi-platform builds**:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag ${REGISTRY}/api:${SHA} \
  --push .
```

`buildx` uses QEMU emulation under the hood when building for a non-native arch; it's slow (often 5-10x slower than native) but it works without managing per-arch toolchains. For CI/CD, prefer running the build on a native runner per arch (GitHub Actions `runs-on: ubuntu-24.04-arm` or self-hosted Graviton runners) and then `docker manifest create` to combine.

### 24.4 Multi-arch container images

If you publish images that may run on either arch (open-source projects, internal libraries), build a multi-arch manifest. Consumers pull whichever arch matches their host:

```bash
docker buildx create --name builder --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag ghcr.io/myorg/api:1.4.2 \
  --push .
```

For internal services running on a known arch, single-arch images are simpler and faster to build. Publish multi-arch only when consumers genuinely need it.

### 24.5 Decision matrix at a glance

| Constraint | Recommended arch |
|---|---|
| Pure Go, AWS Lambda or Fargate | **arm64** (cheaper + faster) |
| Pure Go, EKS with mixed nodes | **arm64** preferred; multi-arch image if migrating gradually |
| CGO dep with arm64 support | **arm64** still |
| CGO dep without arm64 support | **amd64** |
| CUDA / GPU workload | **amd64** (drivers + ecosystem maturity) |
| AVX-512 / AMX hot path | **amd64** |
| Customer / on-prem x86-only | **amd64** |
| Open-source library / image consumed by both | **multi-arch** |

---

## 25. GPU and NVIDIA Workloads

Go is not the language you write CUDA kernels in. It's not the language you train PyTorch models in. But it's an excellent language for the **service layer that fronts GPU workloads**: API gateway in front of an inference server, orchestration of GPU jobs, metrics/observability sidecar, Kubernetes operators (NVIDIA's own GPU Operator is written in Go), CLI tooling, autoscaling controllers.

The pattern: **Python (or C++) does the GPU math; Go does the request handling, validation, auth, observability, and routing.** Splitting along that line lets each language do what it's good at.

### 25.1 What Go does well in a GPU stack

- **HTTP / gRPC API in front of an inference server.** Triton, TGI, vLLM, Ollama, NIM all expose HTTP / gRPC. Go fronts them with the same patterns this guide covers — validation, auth, rate limit, structured logging, OTel traces — and forwards inference calls to the model server.
- **Request routing and batching.** Aggregating low-latency single-request traffic into larger batches that GPUs prefer. Mature pattern: a Go service accumulates requests for ~5-50 ms, fires a batched gRPC call, fans the response back out.
- **Tenant isolation, queueing, prioritization.** The Go service knows about your customers, quotas, billing tiers; the model server doesn't.
- **GPU job orchestration.** Submit training / fine-tuning / batch jobs to Kubernetes (with `nvidia.com/gpu` resource requests), poll status, surface metrics to the user. The Kubernetes Go client (`client-go`) is the standard.
- **Observability collection.** Pull GPU metrics (utilization, memory, SM activity, power, temperature) from NVIDIA DCGM or `dcgm-exporter`; publish to Prometheus / OTel / CloudWatch.
- **CLI / dev tooling around model lifecycle.** Pull/push to model registries, validate model configs, drive deployments. Go's single-binary distribution shines here.

### 25.2 What Go does **not** do well

- **Training.** Use PyTorch / JAX / TensorFlow in Python. Don't try to reproduce them in Go.
- **Custom CUDA kernels.** Use C++/CUDA. Wrap with a C++ binary your Go code shells out to or talks to over gRPC.
- **Heavy numerical preprocessing.** NumPy / cuPy / Pandas / Polars / Arrow in Python or Rust. Go's stdlib has no good numerical computing story; `gonum` is fine for moderate work but not the right call here.
- **Direct CUDA from CGO** (with a few exceptions). Possible but painful — bindings exist (`gocuda`, custom wrappers) but the Python ecosystem moves faster than any Go CUDA binding will. Talk to a CUDA process over gRPC instead.

### 25.3 NVIDIA Triton Inference Server

Triton is NVIDIA's open-source inference server. It hosts models from PyTorch, TensorFlow, ONNX, TensorRT, OpenVINO, Python backends, and others behind a unified HTTP / gRPC API. It handles batching, multiple model versions, ensembles, GPU/CPU scheduling, monitoring.

For Go services that need to call inference, **Triton's gRPC API is the right interface**. NVIDIA publishes the `.proto` files and example Go client code — generate your own client with `protoc-gen-go` + `protoc-gen-go-grpc`:

```bash
protoc --go_out=. --go-grpc_out=. \
  --go_opt=paths=source_relative \
  --go-grpc_opt=paths=source_relative \
  grpc_service.proto model_config.proto
```

Then use the generated client like any other gRPC service:

```go
conn, err := grpc.NewClient("triton:8001", grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil { return fmt.Errorf("grpc dial: %w", err) }
defer conn.Close()
client := triton.NewGRPCInferenceServiceClient(conn)

resp, err := client.ModelInfer(ctx, &triton.ModelInferRequest{
    ModelName:    "bert-base",
    ModelVersion: "1",
    Inputs:       []*triton.ModelInferRequest_InferInputTensor{ /* ... */ },
})
```

Triton on port 8001 is gRPC; port 8000 is HTTP/REST; port 8002 is Prometheus metrics. Use gRPC for high-throughput inference (lower per-call overhead, streaming for sequence models); HTTP for ad-hoc / debugging.

### 25.4 NVIDIA NIM (NVIDIA Inference Microservices)

NIM packages popular models (LLMs, vision, embeddings) as ready-to-deploy containers with optimized inference (TensorRT-LLM under the hood for LLMs). Each NIM exposes a standard OpenAI-compatible REST API plus, for some models, gRPC.

For a Go service consuming a NIM, treat it like any other HTTP API:

- One reusable `*http.Client` with sensible `Transport` settings.
- Wrap calls in a typed function in `internal/<domain>/nim_client.go`.
- Add OTel tracing (`otelhttp.NewTransport`) for end-to-end traces from your handler through to the GPU.
- Honor `ctx` cancellation — long-running generations can be aborted server-side via the OpenAI streaming protocol.

NIM containers can run on any NVIDIA-GPU host: ECS / EKS with `g5`/`g6`/`p4`/`p5` instances, on-prem DGX, NVIDIA DGX Cloud, partner clouds (Lambda Labs, CoreWeave, RunPod), or self-hosted with the NVIDIA Container Toolkit.

### 25.5 NVIDIA DGX Cloud and Cloud Functions (NVCF)

For workloads that need GPUs but you don't want to manage capacity:

- **DGX Cloud Lepton / DGX Cloud Create** — managed multi-tenant GPU clusters, accessible through partnerships with hyperscalers (AWS, Azure, GCP, Oracle). Pay for capacity.
- **NVIDIA Cloud Functions (NVCF)** — serverless inference. You upload a container (often a NIM); NVCF auto-scales it across GPU clusters. Invocation API is HTTP / gRPC. Cold starts exist; for low-latency paths, use provisioned capacity. Good fit when traffic is bursty or you want to avoid running idle GPUs.

From Go's perspective, both are "an HTTP/gRPC endpoint that performs inference". Wrap with a typed client; include auth (NVCF uses API keys); set timeouts generously (LLM generations can take seconds); stream responses where the API supports it.

### 25.6 Self-hosting GPU workloads on AWS

If you want to run your own inference server (Triton + your model) on AWS:

- **EC2 GPU instances** (`g5`, `g6`, `g6e`, `p4d`, `p5`, `p5e`, `p6`): direct control, lowest cost per hour, you manage the AMI / drivers / patches. Use the Deep Learning AMI (CUDA + cuDNN + container toolkit pre-installed) or the official NVIDIA AMI.
- **ECS on EC2 with GPU instances**: ECS schedules tasks onto GPU-enabled hosts. Task definition declares `resourceRequirements: [{ type: GPU, value: "1" }]`. Container automatically gets GPU access via NVIDIA Container Runtime.
- **EKS with NVIDIA GPU Operator**: Kubernetes-native. The GPU Operator (Go-based, by NVIDIA) installs drivers, container toolkit, monitoring, time-slicing, MIG configuration. Pods request `nvidia.com/gpu: 1` and get scheduled accordingly.
- **AWS Batch** for offline / training jobs.
- **SageMaker Inference** if you'd rather AWS manage the serving (SageMaker has its own model formats and SDK; less Go-friendly than self-hosting Triton).

**Fargate does not support GPUs.** For GPU workloads, you're on EC2 (directly, via ECS, or via EKS). Plan capacity accordingly — GPU instances don't scale-from-zero like Fargate does.

### 25.7 NVIDIA Container Toolkit (running GPU containers)

To expose host GPUs to a container, install the **NVIDIA Container Toolkit** on the host and run with the runtime:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

For a Go binary that doesn't itself link CUDA but needs to call into a GPU-using sibling process / library (rare in Go):

```dockerfile
# Multi-stage: build stage uses standard golang image
FROM golang:1.24 AS build
# ... build as usual ...

# Runtime stage: NVIDIA CUDA base image
FROM nvcr.io/nvidia/cuda:12.4.1-base-ubuntu22.04
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=build /out/api /api
USER 65532:65532
ENTRYPOINT ["/api"]
```

For most Go services that **don't** call CUDA directly (the common case — they call a separate Triton/NIM container), keep using **distroless**. Your Go container has no GPU dependency; only the inference container needs the CUDA base image and `--gpus`.

### 25.8 CUDA from Go via CGO (rarely the right call)

You can call CUDA from Go via CGO bindings (`gocuda`, `gomlx`, custom). It's **almost always the wrong choice**:

- Build complexity: CGO + CUDA toolchain on every dev machine and CI runner.
- Debugging: race detector, pprof, and `delve` all degrade or break with heavy CGO.
- Velocity: the Python ecosystem ships new CUDA features (FlashAttention, FP8, sparse kernels, MoE routing) within weeks of release. Go bindings lag by quarters or years.
- Operations: a CGO binary loses the "single static binary" appeal — you're back to managing libc, CUDA driver compatibility, runtime libraries.

The right pattern is: **C++ or Python does the CUDA work in a separate process; Go talks to it over gRPC or shared memory.** This keeps your Go service cleanly Go.

The exceptions: a tightly-controlled internal tool where the CUDA call is small and stable, or a niche library where the latency of an extra IPC hop genuinely matters. Both are rare.

### 25.9 GPU observability

NVIDIA's **DCGM (Data Center GPU Manager)** is the standard for GPU telemetry. The companion **`dcgm-exporter`** scrapes DCGM and exposes Prometheus-format metrics: utilization (`DCGM_FI_DEV_GPU_UTIL`), memory (`DCGM_FI_DEV_FB_USED`), SM activity, power draw, temperature, ECC errors.

Run `dcgm-exporter` as a sidecar (ECS) or DaemonSet (EKS). Scrape with your Prometheus, ADOT collector, or CloudWatch agent. Add to OTel traces: when your Go service calls Triton / NIM, attach the upstream model name, version, and (if available) the response's `triton_inference_count` header — this lets you correlate request-level Go traces with GPU-level utilization metrics.

Useful dashboards:

- GPU utilization vs request rate (find utilization gaps to drive batching).
- GPU memory vs concurrent requests (find OOM headroom).
- Inference latency p50/p95/p99 by model (regression detection).
- Cost per inference (request count × hourly GPU cost / GPU count).

### 25.10 Decision matrix: hosting GPU inference

| Need | Best fit |
|---|---|
| Bursty / unknown traffic, want zero idle cost | **NVCF** (NVIDIA Cloud Functions) |
| Steady inference traffic, AWS-native | **EC2 G5/G6 + ECS + Triton** or **EKS + GPU Operator + Triton** |
| Off-the-shelf LLM behind an API | **NIM** (self-hosted on EC2 / EKS, or via DGX Cloud) |
| Multi-tenant managed GPUs across clouds | **DGX Cloud** |
| Training / batch jobs | **AWS Batch + EC2 G6/P5** or **SageMaker Training** |
| Lowest cost per hour, willing to manage | **CoreWeave / Lambda Labs / RunPod** (often cheaper than hyperscalers for raw GPU) |
| AWS-managed end-to-end | **SageMaker Inference** (less Go-friendly; SDK is Python-first) |

### 25.11 Architecture choice for GPU hosts

GPU hosts are predominantly **amd64**. CUDA on arm64 is well-supported on Jetson and the Grace family (Grace Hopper, Grace Blackwell), but most data-center GPU instances in AWS / Azure / GCP today are amd64. Default to **`GOARCH=amd64`** for any Go binary that runs on a GPU host or directly interacts with CUDA libraries — even if your sibling Go services run arm64 elsewhere. Multi-arch images are an option if you want a single image for both fleets.

---

## Appendix: Recommended Reading

- [Effective Go](https://go.dev/doc/effective_go) — the canonical idioms guide. Read once a year.
- [Go Wiki: Code Review Comments](https://go.dev/wiki/CodeReviewComments) — what reviewers will flag.
- [Practical Go](https://dave.cheney.net/practical-go/presentations/qcon-china.html) (Dave Cheney) — pragmatic guidance from a longtime contributor.
- [The Go Programming Language](https://www.gopl.io) (Donovan & Kernighan) — the textbook. Still relevant.
- [Concurrency in Go](https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/) (Cox-Buday) — the deep dive on goroutines, channels, context.
- [Go release notes](https://go.dev/doc/) — read every minor release.
- [AWS SDK for Go v2 docs](https://aws.github.io/aws-sdk-go-v2/docs/) — authoritative API + idioms.
- [AWS Lambda Go runtime guide](https://docs.aws.amazon.com/lambda/latest/dg/lambda-golang.html) — `provided.al2023`, `bootstrap`, deploy.
- [AWS Distro for OpenTelemetry](https://aws-otel.github.io/) — the AWS-blessed OTel collector.
- [pgx documentation](https://github.com/jackc/pgx) — driver reference.
- [sqlc documentation](https://docs.sqlc.dev/) — query generator reference.
- [OpenTelemetry Go docs](https://opentelemetry.io/docs/languages/go/) — SDK, instrumentation, semantic conventions.
- [`go.dev/security/best-practices`](https://go.dev/doc/security/best-practices) — official security guide.
- [NVIDIA Triton Inference Server](https://github.com/triton-inference-server/server) — open-source inference server; Go gRPC client examples in [triton-inference-server/client](https://github.com/triton-inference-server/client).
- [NVIDIA NIM Microservices](https://www.nvidia.com/en-us/ai-data-science/products/nim-microservices/) — packaged inference containers.
- [NVIDIA Cloud Functions (NVCF)](https://developer.nvidia.com/dgx-cloud/nvcf) — serverless GPU inference.
- [NVIDIA DGX Cloud](https://www.nvidia.com/en-us/data-center/dgx-cloud/) — managed GPU clusters via hyperscaler partners.
- [NVIDIA GPU Operator](https://github.com/NVIDIA/gpu-operator) — Kubernetes operator (written in Go) for GPU lifecycle management.
- [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-container-toolkit) — runtime for exposing GPUs to containers.
- [DCGM and `dcgm-exporter`](https://github.com/NVIDIA/dcgm-exporter) — GPU telemetry exposed as Prometheus metrics.

---

This guide is a snapshot. Go evolves, AWS evolves, the ecosystem evolves. When something here disagrees with the current Go release notes or AWS docs, trust those — and PR an update.
