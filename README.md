# Go-with-AI

A `CLAUDE.md` template that teaches [Claude Code](https://docs.anthropic.com/en/docs/claude-code) modern Go backend best practices for services running on AWS or NVIDIA GPU hosts. Drop it into any Go HTTP / gRPC API, worker, or model-serving gateway and Claude will follow production-grade conventions for project layout, the stdlib `net/http` server, `context` propagation, `pgx` + `sqlc` database access, AWS SDK v2, AWS Lambda (`provided.al2023` + arm64), ECS Fargate, **NVIDIA GPU workloads (Triton / NIM / NVCF / DGX Cloud)**, **arm64 vs amd64 architecture choice**, OpenTelemetry, `slog`, security, and testing — automatically, on every interaction.

## Why

Claude Code reads `CLAUDE.md` from your working directory and applies its rules to every conversation (including sub-agents). Without guardrails, AI-generated Go code can drift in subtle ways that don't show up until production: a missing `ctx` argument that silently disables cancellation, a `fmt.Sprintf`-built SQL query (instant injection), `panic` in a request handler (process restart on every bad input), the wrong AWS SDK (v1 reached end-of-support July 2025), `ReadHeaderTimeout` left at zero (Slowloris bait), `math/rand` for tokens (predictable session IDs), bare goroutines with no termination path (memory leaks), `Error`-level logs for everything (alerting becomes useless). This template codifies the conventions experienced Go teams enforce in code review — small interfaces in the consumer package, the domain-package store pattern, errgroup for fan-out, slog with JSON in prod, AWS SDK v2 singletons, multi-stage distroless arm64 Docker images — so Claude follows them from the first line.

## Two Versions

Pick the one that fits your project.

| File | Size | Contains | Best for |
|---|---|---|---|
| [`CLAUDE.md.FULL`](./CLAUDE.md.FULL) | 39,951 chars | All rules **plus canonical code examples** (HTTP handler, sqlc store, table-driven test, project tree) | Teams that want rules plus concrete patterns — Claude follows rules-with-examples more reliably |
| [`CLAUDE.md.SHORT`](./CLAUDE.md.SHORT) | 37,423 chars | All rules, **no fenced code blocks** | Token-constrained setups, or projects that just need guardrails without inline samples |

Both files cover the **same rules** — the only difference is whether each rule is illustrated with a code block. SHORT is ~6% smaller because it strips all fenced code blocks while preserving every rule, table, and checklist.

<details>
<summary><strong>What's inside (click to expand)</strong></summary>

| Area | Key rules |
|---|---|
| **Code quality** | Errors are values, accept interfaces / return concrete types, tiny consumer-defined interfaces, doc comments on exported identifiers, `gofmt` non-negotiable |
| **Naming** | Lowercase package names, no `util`/`common`/`helpers`/`models`, acronyms uniform case (`URL`, `userID`), `ErrNotFound` / `NotFoundError`, no `UPPER_SNAKE` constants |
| **Modules** | One `go.mod` per module, `go 1.24`+, `go mod tidy` per commit, one major bump per PR |
| **Generics / Iterators** | `any` not `interface{}`; generics only when duplication is forced; range-over-function for streaming only; defined types over aliases for new domain types |
| **Errors** | `%w` wrapping, sentinel errors (`errors.Is`), custom error types (`errors.As`), `errors.Join`, no string compare, no swallowing |
| **Project layout** | `cmd/<binary>/main.go` wiring only; `internal/` private; organize by **domain** not technical layer; avoid `pkg/` unless externally consumed |
| **HTTP server** | Stdlib `net/http` (1.22+ `ServeMux`); explicit timeouts (`ReadHeaderTimeout` etc.); graceful shutdown on SIGTERM; DTOs (never raw domain/DB structs); stable error shape |
| **Context** | First parameter of every blocking function; `r.Context()` in handlers; `context.WithTimeout` at boundaries; `errgroup.WithContext` for fan-out fail-fast; `context.Cause` for debugging |
| **Concurrency** | Goroutines must have termination paths; channels for handoff, mutexes for state; sender closes channels; `go test -race` non-negotiable in CI |
| **Database** | `pgx/v5` + `sqlc`; one `*pgxpool.Pool` per process; tuned `MaxConns` / `MaxConnLifetime`; transactions short; placeholders only (no `fmt.Sprintf`); store interface in domain package |
| **AWS SDK** | v2 only (v1 EOL July 2025); singletons; `ctx` always; IAM roles (never keys); paginators; per-service clients via functional options |
| **Lambda** | `provided.al2023` runtime (`go1.x` deprecated); arm64 Graviton2 (20% cheaper); `bootstrap` binary; package-level init; cold start <60ms achievable |
| **ECS / Fargate** | Multi-stage Dockerfile → distroless/scratch arm64; `/healthz` (liveness) vs `/readyz` (readiness); distinct task vs execution roles; secrets via task definition; Fargate Spot for tolerant workloads (no GPU on Fargate) |
| **CPU architecture** | arm64 default (Graviton, ~20% cheaper); amd64 when CGO deps lack arm64 wheels, GPU/CUDA, AVX-512/AMX, or x86-only customer; `docker buildx` for multi-arch when consumers need both |
| **GPU / NVIDIA** | Go is the service layer (routing, auth, batching, observability) — Python/C++ does GPU math; talk to Triton / TGI / vLLM / NIM over gRPC (generate Go client from `.proto`); host on **NVCF** (serverless GPU), **DGX Cloud** (managed), or **EC2 G5/G6/P4/P5 + ECS-on-EC2 / EKS + GPU Operator**; CUDA via CGO is almost always wrong; DCGM via `dcgm-exporter` for telemetry |
| **Configuration** | Env vars at startup, validate fail-fast; one source of truth in `internal/config`; secrets in Secrets Manager / SSM; never log config |
| **Logging** | `log/slog` JSON in prod; pass logger (don't use global); `*Context` variants; key-value pairs not formatted strings; `LogValuer` for redaction |
| **Observability** | OpenTelemetry + ADOT (X-Ray transitioning to OTel); auto-instrument HTTP / DB / AWS via `otelhttp` / `otelpgx` / `otelaws`; manual spans for business ops; baseline RED metrics |
| **Validation** | Validate at the edge; `go-playground/validator`; `http.MaxBytesReader`; treat all external input as untrusted |
| **Security** | Parameterized SQL (sqlc); no `exec.Command("sh", "-c", input)`; SSRF guards; `crypto/rand` not `math/rand`; argon2id passwords; httpOnly Secure cookies; `govulncheck` + `gosec` in CI |
| **Testing** | Stdlib `testing`; table-driven with `t.Run` + `t.Parallel()`; hand-written fakes over generated mocks; `testcontainers-go` for integration tests gated by `//go:build integration` |
| **Build** | `CGO_ENABLED=0 GOARCH=arm64`; `-trimpath -ldflags "-s -w -X main.version=..."`; multi-stage Dockerfile; distroless `static-debian12:nonroot`; image scanned (Trivy/Grype) |
| **Tooling** | `gofmt`/`goimports`, `go vet`, `golangci-lint` (curated set), `govulncheck`, `gosec`, `pprof`, `benchstat`, `gopls`, `delve` |
| **Performance** | Profile first (`pprof`); `automaxprocs` for cgroup CPU; `GOMEMLIMIT` for memory; allocations dominate latency; reuse `http.Client.Transport` |
| **Large projects** | Dep audits, dead code removal (`staticcheck` U1000), incremental migration behind flags, performance budgets via `benchstat`, monorepo with `go.work` past ~10 domain packages |

Both versions also include token-efficient output rules, a **new-project setup checklist**, an **existing-project onboarding checklist**, a **pre-completion verification checklist**, and a customizable **Project-Specific Overrides** section at the bottom.

</details>

## Installation

1. **Download** the version you want into your Go project root:

   ```bash
   # FULL (with code examples)
   curl -o CLAUDE.md https://raw.githubusercontent.com/maxart/Go-with-AI/main/CLAUDE.md.FULL

   # Or SHORT (rules only)
   curl -o CLAUDE.md https://raw.githubusercontent.com/maxart/Go-with-AI/main/CLAUDE.md.SHORT
   ```

   The `curl` commands above already rename the file to `CLAUDE.md` — if you download manually, rename it yourself.

2. **Customize** the `Project-Specific Overrides` section at the bottom of `CLAUDE.md` with your stack details — HTTP router choice, database client, AWS deployment target (Lambda vs Fargate vs EKS), observability backend, test framework, lint config.

Claude Code picks it up automatically on every conversation. No slash commands, no configuration.

## How It Works

```
your-go-service/
  CLAUDE.md            <-- Claude reads this automatically
  cmd/api/main.go      <-- HTTP server entry point
  internal/            <-- private packages (Go enforces)
    user/, order/      <-- domain packages
    platform/          <-- database, awsx, logger, otelx
  go.mod, go.sum
  Dockerfile, Makefile, .golangci.yml
```

Claude Code loads `CLAUDE.md` from the working directory (and parent directories) at the start of every conversation. The instructions become part of Claude's system context and apply to the main agent and all sub-agents.

<details>
<summary><strong>How scope works</strong></summary>

| Placement | Effect |
|---|---|
| Project root | Applies to all work in the project |
| `cmd/` subdirectory | Applies only when working in `cmd/` (entry-point rules) |
| `internal/` subdirectory | Applies only when working in `internal/` (domain-package rules) |
| Multiple levels | Rules stack — deeper files add to (or override) parent rules |

</details>

## Customization

The template is a starting point, not a straitjacket.

- **Remove** rules that conflict with your project (e.g., delete the Lambda section if you only run on ECS, or the `pgx` section if you use DynamoDB).
- **Add** project-specific rules anywhere — auth conventions, custom error codes, deployment runbooks, internal package naming.
- **Override** defaults in the `Project-Specific Overrides` block at the bottom.

Rules with code examples (FULL version) are followed more reliably than abstract guidelines. If Claude keeps missing a rule, make it more specific or add an example.

<details>
<summary><strong>Example: Project-Specific Overrides</strong></summary>

```markdown
## Project-Specific Overrides

### HTTP Layer
- Stdlib net/http (Go 1.22+ ServeMux); chi only for v1 endpoints under /v1/*
- Middleware order: recover -> request_id -> otelhttp -> slog -> auth -> ratelimit
- DTOs in internal/<domain>/dto.go; never return domain or store rows directly

### Database
- Postgres via pgx/v5 + sqlc; queries in db/queries/*.sql, generated to internal/db/
- Migrations in migrations/ (golang-migrate); applied by cmd/migrate before deploy
- Pool: MaxConns=20, MaxConnLifetime=30m, idle=5m

### AWS
- ECS Fargate ARM64; task role per service (least privilege); secrets in Secrets Manager
- ADOT sidecar exports to X-Ray + CloudWatch + Prometheus workspace
- Lambda: provided.al2023, arm64, bootstrap binary, ADOT lambda layer

### Logging / Observability
- slog JSONHandler; level from LOG_LEVEL env (default info); request_id and trace_id always
- OpenTelemetry SDK; OTLP gRPC to localhost:4317; sample 5% in prod, 100% in staging
- RED metrics on every HTTP route; queue depth + processing latency on every worker

### Testing
- testify/require; table-driven; t.Parallel everywhere safe
- Integration tests build tag //go:build integration — testcontainers Postgres + LocalStack
- Coverage gate: 70% on internal/<domain>/; no gate on cmd/

### Tooling / CI
- golangci-lint v1.60+; .golangci.yml in repo root with curated set
- Makefile entry points: lint, test, integration, build, docker, deploy
- One Dependabot PR per major bump; security PRs auto-merged after CI green
```

</details>

## Companion File

| File | Description |
|---|---|
| [`docs/GO_BEST_PRACTICES_GUIDE.md`](./docs/GO_BEST_PRACTICES_GUIDE.md) | Long-form tutorial covering each rule in depth with explanations, alternatives, and gotchas. For learning or onboarding — not meant to be loaded as `CLAUDE.md`. |

## Compatibility

- **Claude Code**: CLI, desktop app, web app, IDE extensions (VS Code, JetBrains)
- **Go**: tested against **1.24+** (current stable; 1.25 also supported, 1.26 expected Feb 2026). Most rules apply to all 1.21+; the 1.22+ enhanced `ServeMux` patterns require 1.22+; `range`-over-function iterators require 1.23+; generic type aliases require 1.24+; `GOMEMLIMIT` requires 1.19+. Adapt or remove if you're on an older minor.
- **AWS SDK for Go**: **v2 only** (`github.com/aws/aws-sdk-go-v2/...`). v1 reached end-of-support **July 31, 2025** — migrate any v1 code; do not start anything new on it.
- **AWS Lambda runtime**: `provided.al2023` (the `go1.x` runtime was deprecated December 31, 2023 and rejects new deployments).
- **Database**: rules tuned for Postgres + `pgx/v5` + `sqlc`. Most apply to MySQL via `database/sql` + a similar generator; DynamoDB section adapts; `lib/pq` is in maintenance mode and not recommended.
- **Deployment**: AWS-first (Lambda, ECS Fargate, EKS, EC2). Cloud Run / Fly.io / Render apply equally for the language sections; the Lambda / ECS / IAM / ADOT specifics are AWS-only. **GPU hosting** covers AWS EC2 G/P-family + ECS-on-EC2 / EKS, **NVIDIA Cloud Functions (NVCF)**, **NVIDIA DGX Cloud**, plus a note on CoreWeave / Lambda Labs / RunPod for raw GPU.
- **CPU architecture**: rules default to **arm64** (Graviton on AWS) and call out when **amd64** is the right call (CGO deps, CUDA / GPU workloads, AVX-512, customer constraints). Multi-arch via `docker buildx` is documented.
- **HTTP framework**: rules default to stdlib `net/http` (1.22+ `ServeMux`). `chi`, `echo`, `gin` are noted as acceptable; `fiber` is flagged as non-stdlib (built on `fasthttp`). Adapt naming and middleware patterns to your choice.

Not for `pages/`-style web frameworks, gRPC-only services without REST, or non-AWS clouds. Most language-level rules (errors, context, concurrency, testing, security, tooling) apply universally; the deployment, Lambda, ECS, and ADOT sections are AWS-specific. If you're on GCP / Azure / on-prem, keep the language sections and replace the deployment/observability sections with your platform's equivalents.

## FAQ

**Which version should I pick?**
Start with **FULL** if you're unsure. The canonical code examples (HTTP handler with error mapping, the sqlc-backed Postgres store, the table-driven test) make Claude more consistent at applying each rule, especially on Go-specific patterns (context propagation, error wrapping, the consumer-defined store interface) where a subtle mistake costs hours. Switch to **SHORT** if context size matters (large monorepo, heavy sub-agent use) or once your team has settled on conventions and Claude reliably follows them.

**Does this work outside AWS?**
The language sections (TypeScript… er, Go: errors, context, concurrency, testing, security, tooling, performance) are framework-and-cloud agnostic. The HTTP server section assumes stdlib `net/http` which works anywhere. The Database section assumes Postgres which works anywhere. The Lambda, ECS Fargate, AWS SDK, and ADOT sections are AWS-specific — remove or adapt if you're on GCP (Cloud Run, Cloud Functions, Cloud SQL), Azure (Container Apps, Functions, SQL DB), or Kubernetes generically.

**Does this work for gRPC services?**
Most rules transfer directly: errors, context, concurrency, observability, testing, project structure, security, build/deploy. Replace the HTTP server section with `google.golang.org/grpc` patterns: `grpc.NewServer` with interceptors, `protoc`-generated handlers, `otelgrpc` for tracing, server reflection in dev only, status codes via `google.golang.org/grpc/codes`. The DTO discussion becomes proto messages.

**Does this work for CLI tools or worker services without HTTP?**
Yes — drop the HTTP server section. Everything else (project layout, errors, context, concurrency, AWS SDK, observability, testing, build, security) applies. For workers, the "context propagation" rules become "every job handler takes `ctx` from the queue consumer".

**Will this slow Claude down?**
No. `CLAUDE.md` is loaded once per conversation and is negligible in the context window.

**Can I use this with other AI coding tools?**
The file targets Claude Code, but the content is standard Go practice and portable.

- **Agents that read `AGENTS.md`** (Codex, OpenCode, Cursor's `AGENTS.md` mode, and others adopting that convention): download it directly as `AGENTS.md`.

  ```bash
  curl -o AGENTS.md https://raw.githubusercontent.com/maxart/Go-with-AI/main/CLAUDE.md.FULL
  ```

- **Both Claude Code and an `AGENTS.md`-based agent in the same repo**: save the file as `AGENTS.md`, then create a one-line `CLAUDE.md` that imports it. Claude Code expands `@path` imports into context at session start, so both tools read identical instructions from a single source — no duplication, no drift.

  ```bash
  curl -o AGENTS.md https://raw.githubusercontent.com/maxart/Go-with-AI/main/CLAUDE.md.FULL
  echo '@AGENTS.md' > CLAUDE.md
  ```

- **Cursor, Windsurf, Copilot**: adapt the rules into `.cursor/rules/`, `.windsurfrules`, or `.github/copilot-instructions.md`.

**Claude is ignoring a rule.**
`CLAUDE.md` is high-priority but not absolute. If a rule is consistently ignored, make it more specific or add an example. Rules with code examples are followed more reliably — the main reason to prefer FULL over SHORT.

**What about authentication?**
The template stays library-agnostic. The rules that always apply: re-authenticate inside every handler / mutation (don't trust the proxy / API gateway / WAF), check resource ownership (not just login state), opaque session IDs in httpOnly Secure SameSite=Strict cookies backed by a server-side store as the default, JWT only with asymmetric algorithms (RS256/Ed25519) and forced expected `alg` if you must.

**What about the database?**
The template defaults to Postgres + `pgx/v5` + `sqlc`. The rules generalize: queries live in domain-package store implementations, the store interface is consumer-defined in the domain package, transactions are short, parameter placeholders only. For DynamoDB, replace the sqlc section with `aws-sdk-go-v2/service/dynamodb` patterns (typed input/output structs, paginators, conditional expressions). For MySQL, swap `pgx` for `database/sql` + the MySQL driver.

**Why stdlib `net/http` instead of `chi` / `gin` / `echo`?**
For most of Go's history, the stdlib router was too primitive. Go 1.22 (Feb 2024) added method matching, path wildcards, and exact-path patterns to `ServeMux` — for most services this is enough. Reach for `chi` if you want a richer middleware ecosystem; `echo` / `gin` only if your team is already invested or you use the integrated bindings / validators. The template doesn't ban them — but it defaults to stdlib because fewer dependencies = less audit surface, fewer breaking changes, more new-team-member familiarity.

**Why arm64 / Graviton2?**
About 20% cheaper duration on Lambda and Fargate, ~19% faster for typical Go workloads. Cross-compiling Go for arm64 is trivial (`GOARCH=arm64`); no Docker emulation needed. Fall back to **amd64** when a CGO dependency lacks arm64 wheels, the workload uses CUDA / NVIDIA GPUs (drivers + ecosystem are amd64-first), a hot path needs AVX-512 / AMX, or a customer / on-prem environment is x86-only. The guide includes a decision matrix.

**What about NVIDIA GPUs / Cloud?**
Go isn't the language you write CUDA in — it's the language for the **service layer in front of GPU workloads**: API gateway, request batching, auth, observability, Kubernetes operators (NVIDIA's own GPU Operator is Go). The pattern: Python or C++ does the GPU math in Triton / TGI / vLLM / NIM; Go fronts it over gRPC. Hosting options: **NVCF** (NVIDIA Cloud Functions — serverless GPU, pay per invocation), **DGX Cloud** (managed multi-tenant GPU clusters via hyperscaler partners), or **AWS EC2 G5/G6/P4/P5 + ECS-on-EC2 or EKS + the GPU Operator** for self-managed. Fargate has no GPUs. Calling CUDA from Go via CGO is almost always wrong (build complexity, debugging loss, ecosystem velocity); talk to a separate Python/C++ inference process over gRPC instead. The guide has a full section on GPU patterns, hosting options, NVIDIA Container Toolkit, DCGM observability, and a hosting decision matrix.

**Why OpenTelemetry instead of the AWS X-Ray SDK?**
AWS X-Ray transitioned to ingest OTLP, and AWS Distro for OpenTelemetry (ADOT) is now the AWS-blessed collector. The X-Ray SDK is on a deprecation trajectory. New code should use OTel directly with `otelhttp` / `otelpgx` / `otelaws` for auto-instrumentation; ADOT forwards to X-Ray, CloudWatch Metrics, Prometheus Workspace, and any other OTLP-compatible backend (Datadog, Honeycomb, New Relic, Grafana, Splunk).

## Further Reading

- [Effective Go](https://go.dev/doc/effective_go) — the canonical idioms guide
- [Go Wiki: Code Review Comments](https://go.dev/wiki/CodeReviewComments) — what reviewers will flag
- [Go release notes](https://go.dev/doc/) — read every minor release
- [AWS SDK for Go v2](https://aws.github.io/aws-sdk-go-v2/docs/) — authoritative API and idioms
- [AWS Lambda Go runtime guide](https://docs.aws.amazon.com/lambda/latest/dg/lambda-golang.html) — `provided.al2023`, `bootstrap`, deploy
- [AWS Distro for OpenTelemetry](https://aws-otel.github.io/) — AWS-blessed OTel collector
- [pgx documentation](https://github.com/jackc/pgx) — driver reference
- [sqlc documentation](https://docs.sqlc.dev/) — query generator reference
- [OpenTelemetry Go docs](https://opentelemetry.io/docs/languages/go/) — SDK, instrumentation, semantic conventions
- [Go Security Best Practices](https://go.dev/doc/security/best-practices) — official security guide
- [`govulncheck`](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck) — call-graph-aware vulnerability scanner
- [`gosec`](https://github.com/securego/gosec) — Go SAST
- [`golangci-lint`](https://golangci-lint.run/) — meta-linter
- [`testcontainers-go`](https://golang.testcontainers.org/) — integration tests with real Docker dependencies
- [Distroless images](https://github.com/GoogleContainerTools/distroless) — minimal base images for Go
- [NVIDIA Triton Inference Server](https://github.com/triton-inference-server/server) — open-source inference server, gRPC + HTTP
- [NVIDIA NIM Microservices](https://www.nvidia.com/en-us/ai-data-science/products/nim-microservices/) — packaged inference containers
- [NVIDIA Cloud Functions (NVCF)](https://developer.nvidia.com/dgx-cloud/nvcf) — serverless GPU inference
- [NVIDIA DGX Cloud](https://www.nvidia.com/en-us/data-center/dgx-cloud/) — managed multi-tenant GPU clusters
- [NVIDIA GPU Operator](https://github.com/NVIDIA/gpu-operator) — Kubernetes operator (in Go) for GPU lifecycle
- [`dcgm-exporter`](https://github.com/NVIDIA/dcgm-exporter) — GPU telemetry as Prometheus metrics
- [React-with-AI](https://github.com/maxart/React-with-AI) and [Nextjs-with-AI](https://github.com/maxart/Nextjs-with-AI) — companion templates for the frontend side of the stack

## License

MIT
