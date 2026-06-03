# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

The AppsCode fork of [Gotenberg](https://github.com/gotenberg/gotenberg) — a containerized HTTP API for converting documents (HTML, Markdown, URLs, Office docs) to PDF. The binary itself is small Go glue; the value lives in the Docker image, which bundles Chromium, LibreOffice, PDFtk, qpdf, pdfcpu and ExifTool behind a single HTTP server.

Module path is unchanged from upstream: `github.com/gotenberg/gotenberg/v8`. Git remote is `git@github.com:appscode-cloud/gotenberg.git`. This fork tracks upstream and carries AppsCode-specific patches (notably `Dockerfile.ubi` for Red Hat UBI 10).

## Tech stack

| Layer | Library / Tool |
|---|---|
| Language | Go (see `go.mod` — currently `go 1.25.5`) |
| HTTP framework | `github.com/labstack/echo/v4` |
| CLI flags | `github.com/spf13/pflag` (also overridable via env vars) |
| Logging | `go.uber.org/zap` + `go.uber.org/multierr` |
| Health checks | `github.com/alexliesenfeld/health` |
| Headless browser | `github.com/chromedp/chromedp` + `cdproto` (CDP protocol) |
| Office conversion | LibreOffice + `unoconverter` (Python helper) invoked out-of-process |
| PDF engines | `pdftk` (Java), `qpdf`, `pdfcpu` (Go), `exiftool` (Perl) shelled out per request |
| Markdown rendering | `github.com/gomarkdown/markdown` + `microcosm-cc/bluemonday` sanitizer |
| Regex (PCRE-flavored) | `github.com/dlclark/regexp2` (for allow/deny lists) |
| HTTP webhook retry | `github.com/hashicorp/go-retryablehttp` |
| Metrics | `github.com/prometheus/client_golang` |
| Process introspection | `github.com/shirou/gopsutil/v4` |
| Archives (testing) | `github.com/mholt/archives` |
| Integration test runner | `github.com/cucumber/godog` (Gherkin) + `github.com/testcontainers/testcontainers-go` |
| Linter | `golangci-lint` v2 (`.golangci.yml`) |
| Formatter | `golangci-lint fmt` (gci, gofmt, gofumpt, goimports) for Go; `prettier` for everything else |
| Node (prettier only) | `.node-version` → 24.11.0 |

## Folder structure

```
gotenberg/
├── cmd/
│   ├── gotenberg.go            # Package gotenbergcmd: parses flags + env, starts/stops App modules
│   └── gotenberg/
│       └── main.go             # `package main` — imports pkg/standard (blank) then calls gotenbergcmd.Run()
├── pkg/
│   ├── gotenberg/              # Core module system (inspired by Caddy)
│   │   ├── modules.go          # Module/ModuleDescriptor/App/Provisioner/Validator interfaces + registry
│   │   ├── context.go          # Module context: flags, dependency lookup
│   │   ├── flags.go            # ParsedFlags wrapper around pflag.FlagSet
│   │   ├── env.go              # Env-var override plumbing
│   │   ├── pdfengine.go        # PdfEngine interface (Merge/Split/Convert/ReadMetadata/...)
│   │   ├── supervisor.go       # ProcessSupervisor: restart-after-N, max-queue-size for chromium/libreoffice
│   │   ├── fs.go               # Per-request scratch FileSystem under a unique dir
│   │   ├── filter.go           # PCRE allow/deny list helper (regexp2)
│   │   ├── gc.go               # Background garbage collection of stale temp dirs
│   │   ├── logging.go          # zap logger construction
│   │   ├── metrics.go          # Prometheus Collector interface for modules
│   │   ├── debug.go            # Build-time debug-data assembly
│   │   ├── shutdown.go         # Graceful shutdown helpers
│   │   ├── sort.go             # ModuleDescriptor sorting
│   │   ├── cmd.go              # Shell-out helper for external binaries
│   │   ├── version.go          # `Version` symbol set via -ldflags
│   │   ├── mocks.go            # Test doubles
│   │   └── doc.go              # Package doc — credits Caddy module system
│   ├── standard/
│   │   ├── imports.go          # Blank-imports every standard module to register them
│   │   └── doc.go
│   └── modules/                # One sub-package per pluggable module
│       ├── api/                # HTTP server (Echo): routes, middlewares, health, formdata, download-from
│       ├── chromium/           # CDP browser pool: HTML/Markdown/URL → PDF + screenshots
│       ├── libreoffice/        # /forms/libreoffice/convert
│       │   ├── api/            # `Uno` interface: spawns soffice + unoconverter per job
│       │   └── pdfengine/      # PdfEngine adapter using LibreOffice headlessly
│       ├── pdfengines/         # Merge/split/flatten/convert/metadata/encrypt/embed routes; multiplexes engines
│       ├── pdftk/              # PdfEngine wrapping pdftk-java
│       ├── qpdf/               # PdfEngine wrapping qpdf
│       ├── pdfcpu/             # PdfEngine wrapping pdfcpu
│       ├── exiftool/           # PdfEngine for metadata read/write
│       ├── webhook/            # Async webhook delivery (retryablehttp) with allow/deny lists
│       ├── prometheus/         # /prometheus/metrics endpoint + collector wiring
│       └── logging/            # zap config module (level, format, GCP fields)
├── build/
│   ├── Dockerfile              # Canonical multi-stage image (Debian 13 slim + Chromium + LibreOffice + …)
│   ├── Dockerfile.cloudrun     # FROM main image; sets API_PORT_FROM_ENV=PORT, auto-start, sync webhook, GCP log fields
│   ├── Dockerfile.aws-lambda   # FROM main image; adds aws-lambda-adapter extension; API_PORT_FROM_ENV=AWS_LWA_PORT
│   ├── Dockerfile.ubi          # AppsCode patch — UBI 10 base for Red Hat certification (keep in sync with Dockerfile)
│   ├── fonts.conf              # Fontconfig tuning (subpixel hinting/smoothing) for nicer renders
│   └── chromium-hyphen-data/   # Hyphenation dictionaries shipped to Chromium (issue #1293)
├── test/
│   └── integration/
│       ├── main_test.go        # `//go:build integration` — godog harness, flags for Docker repo/version/platform/tags
│       ├── features/           # Gherkin .feature files, one per module-route group
│       ├── scenario/           # Step bindings + helpers (containers, http, compare, server)
│       ├── testdata/           # HTML/Markdown samples (feature-rich, header-footer, page-N-*)
│       └── teststore/          # Persistent state between scenarios
├── .github/
│   ├── actions/                # build-test-push, merge, clean composite actions
│   └── workflows/
│       ├── continuous-integration.yml   # lint + prettier + unit + 5-platform snapshot/edge images
│       ├── continuous-delivery.yml      # release builds on GitHub Release "published"
│       └── pull-request-cleanup.yml
├── .env                        # Make variables: GOTENBERG_VERSION=snapshot, registry, repo, Dockerfile path
├── Makefile                    # build / run / test-unit / test-integration / lint / fmt / godoc
├── package.json                # Only `prettier` + plugins for non-Go formatting; NOT shipped in the image
├── package-lock.json
├── .golangci.yml               # golangci-lint v2 config (linters + gci section order)
├── .prettierrc / .prettierignore
├── .node-version               # Node version for the prettier check job
├── go.mod / go.sum
├── README.md / SECURITY.md / LICENSE
└── AGENTS.md                   # This file (CLAUDE.md is a one-line `@AGENTS.md` pointer)
```

## Module system (Caddy-inspired)

Everything except `cmd/` and `pkg/gotenberg/` is a module. Modules implement at least `Module.Descriptor()` returning a `ModuleDescriptor{ID, FlagSet, New}` and self-register in an `init()` via `gotenberg.MustRegisterModule(new(Xxx))`. Optional interfaces opt-in to extra behavior:

| Interface | Purpose |
|---|---|
| `Provisioner` | `Provision(*Context) error` — wire up after flags parsed; look up sibling modules via `ctx.Modules(...)` |
| `Validator` | `Validate() error` — post-provision sanity checks |
| `App` | `Start() / StartupMessage() / Stop(ctx) error` — long-running module (e.g., the HTTP server) |
| `SystemLogger` | `SystemMessages() []string` — banner messages on startup |
| `Debuggable` | Contributes to the `/debug` route payload |
| `api.Router` | `Routes() ([]Route, error)` — adds Echo routes to the API module |
| `api.MiddlewareProvider` | `Middlewares() ([]Middleware, error)` |
| `api.HealthChecker` | Hooks into the `/health` endpoint |
| `api.AsynchronousCounter` | Reports in-flight async work for graceful shutdown |
| `gotenberg.PdfEngine` | Merge/Split/Flatten/Convert/Read|WriteMetadata/Encrypt/Embed |

`pkg/standard/imports.go` blank-imports every standard module — adding a new module typically means: write the package, `init()`-register, and add a blank import here (or to a custom main).

## HTTP API surface

All file-producing routes are `POST` and accept `multipart/form-data`. Multipart routes are required to start with `/forms/` (enforced in `pkg/modules/api/api.go`). Mounted under the configurable `--api-root-path` (default `/`).

| Route | Module |
|---|---|
| `POST /forms/chromium/convert/url` | chromium |
| `POST /forms/chromium/convert/html` | chromium |
| `POST /forms/chromium/convert/markdown` | chromium |
| `POST /forms/chromium/screenshot/url` | chromium |
| `POST /forms/chromium/screenshot/html` | chromium |
| `POST /forms/chromium/screenshot/markdown` | chromium |
| `POST /forms/libreoffice/convert` | libreoffice |
| `POST /forms/pdfengines/merge` | pdfengines |
| `POST /forms/pdfengines/split` | pdfengines |
| `POST /forms/pdfengines/flatten` | pdfengines |
| `POST /forms/pdfengines/convert` | pdfengines |
| `POST /forms/pdfengines/encrypt` | pdfengines |
| `POST /forms/pdfengines/embed` | pdfengines |
| `POST /forms/pdfengines/metadata/read` | pdfengines |
| `POST /forms/pdfengines/metadata/write` | pdfengines |
| `GET /health` | api (health module) |
| `GET /version` | api |
| `GET /debug` | api (if `--api-enable-debug-route=true`) |
| `GET /prometheus/metrics` (configurable) | prometheus |

## Configuration

Every flag has a corresponding environment variable: replace `-` with `_` and uppercase. So `--api-port=3000` ≡ `API_PORT=3000`. Env vars take precedence after flag parsing (see `cmd/gotenberg.go`). Slice values are comma-separated and *replace* rather than append.

The `Makefile`'s `run` target documents the full flag matrix in a single command — read it to learn every supported tunable. Highlights:

- `--api-port` / `--api-bind-ip` / `--api-port-from-env` (Cloud Run uses `PORT`, AWS LWA uses `AWS_LWA_PORT`)
- `--api-enable-basic-auth` + `GOTENBERG_API_BASIC_AUTH_USERNAME/PASSWORD`
- `--api-download-from-allow-list` / `--api-download-from-deny-list` (PCRE via regexp2)
- `--chromium-restart-after` (request count before browser respawn), `--chromium-auto-start`, `--chromium-max-queue-size`
- `--libreoffice-restart-after`, `--libreoffice-auto-start`
- `--pdfengines-merge-engines`, `--pdfengines-split-engines`, `--pdfengines-convert-engines`, `--pdfengines-{read,write}-metadata-engines`, `--pdfengines-encrypt-engines`, `--pdfengines-flatten-engines`, `--pdfengines-embed-engines` — ordered fallback lists per operation
- `--webhook-*` — sync mode, allow/deny lists, retry/backoff, timeout
- `--log-level`, `--log-format=auto|json|text`, `--log-enable-gcp-fields`
- `--prometheus-metrics-path` (this fork added `--prometheus-metrics-path`, see commit history)

## Process supervision

LibreOffice and Chromium are heavy and long-lived. `pkg/gotenberg/supervisor.go` wraps each in a `ProcessSupervisor` that:

- queues requests up to `--*-max-queue-size` (0 = unbounded)
- recycles the underlying process every N successful jobs (`--*-restart-after`)
- coordinates with the graceful-shutdown path so in-flight work finishes before `Stop()` returns

`pkg/gotenberg/fs.go` allocates a unique temp dir per request; `pkg/gotenberg/gc.go` sweeps stragglers.

## PDF-engine multiplexing

`pkg/modules/pdfengines/multi.go` implements a `multiPdfEngines` that walks the configured engine list and returns the first successful result, accumulating errors via `multierr`. The configured order matters — engines have different feature support (e.g., `pdfcpu` is the only `embed` engine by default).

## Docker images & deployment

- **`build/Dockerfile`** (default) — four-stage Debian 13 slim build:
  1. `pdfcpu-binary-stage` (golang) — builds pdfcpu for current arch.
  2. `gotenberg-binary-stage` (golang) — builds the gotenberg binary with `-X cmd.Version=...`.
  3. `custom-jre-stage` (debian + JDK + `jlink`) — produces a minimal JRE (modules: `java.base,java.desktop,java.naming,java.sql`) for PDFtk.
  4. Final stage installs Chromium, LibreOffice (from `trixie-backports`), `unoconverter`, PDFtk JAR, qpdf, exiftool, MS core fonts + many CJK/Asian/RTL font packages, hyphen dictionaries, copies fonts.conf and chromium-hyphen-data. Runs as `gotenberg:gotenberg` (UID/GID 1001), `EXPOSE 3000`, entrypoint `tini -- gotenberg`.
- **`build/Dockerfile.cloudrun`** — thin overlay: enables `API_PORT_FROM_ENV=PORT`, auto-start for chromium/libreoffice, synchronous webhook, GCP log fields, gives `gotenberg` ownership of `/usr/bin/tini` (Cloud Run requirement).
- **`build/Dockerfile.aws-lambda`** — thin overlay: adds `aws-lambda-adapter` extension, `API_PORT_FROM_ENV=AWS_LWA_PORT`, sync webhook, debug data off.
- **`build/Dockerfile.ubi`** — AppsCode-specific. Same multi-stage layout but final stage is `registry.access.redhat.com/ubi10/ubi:latest`. Keep in sync with `Dockerfile` when changing build args, copy paths, or installed packages.
- OpenShift support: the final stage chown/chmods `/home/gotenberg` to GID 0 so the container survives arbitrary UID assignment (see issue gotenberg/gotenberg#1049).

CI publishes images for `linux/amd64`, `linux/386`, `linux/arm64`, `linux/arm/v7`, `linux/ppc64le` and merges into a manifest list; release tags are pushed to both `gotenberg/gotenberg` and `thecodingmachine/gotenberg`.

## Common commands

| Command | Purpose |
|---|---|
| `make help` | List Make targets with descriptions |
| `make build` | Build the Docker image at `$(DOCKER_REGISTRY)/$(DOCKER_REPOSITORY):$(GOTENBERG_VERSION)` (defaults set in `.env`) |
| `make run` | Run the built container with the full flag/env matrix from the Makefile |
| `make test-unit` | `go test -race ./...` |
| `make test-integration TAGS=...` | Run godog suite against the built Docker image. Tags filter scenarios; available tags listed in the Makefile (e.g., `chromium`, `libreoffice`, `pdfengines-merge`, `webhook`, `health`, …) |
| `make lint` | `golangci-lint run` |
| `make lint-prettier` | `npx prettier --check .` |
| `make lint-todo` | `golangci-lint run --no-config --disable-all --enable godox` (find TODO/FIXME) |
| `make fmt` | `golangci-lint fmt` + `go mod tidy` |
| `make prettify` | `npx prettier --write .` |
| `make godoc` | Local godoc on `:6060` (requires `golang.org/x/tools/cmd/godoc`) |

Integration tests need Docker running locally and pull the image you just built. Set `PLATFORM=linux/amd64` (or similar) and `NO_CONCURRENCY=true` if you hit flakiness on a constrained machine.

## Integration tests (godog / Cucumber)

- Build tag: `//go:build integration`. Pure unit tests do *not* compile this package.
- One `.feature` file per logical surface in `test/integration/features/`. Add scenarios there, not in Go.
- Step definitions live in `test/integration/scenario/`. Use the existing `compare.go`, `containers.go`, `http.go`, `server.go` helpers — don't roll your own HTTP client.
- Testcontainers spins up the Gotenberg image specified by `--gotenberg-docker-repository` + `--gotenberg-version`.
- `testdata/` carries deterministic HTML/Markdown fixtures (page-1 / pages-3 / pages-12 variants, plus `feature-rich-*` and `header-footer-html`). Reuse them rather than inventing one-off inputs.
- Concurrency defaults to `runtime.NumCPU()`; pass `--no-concurrency=true` for serial runs when debugging.

## Conventions & best practices

- **Module path stays upstream.** Always import as `github.com/gotenberg/gotenberg/v8/...`; never as `appscode-cloud/...`.
- **One module = one package** under `pkg/modules/<id>/`. Always self-register in `init()` and add a blank import to `pkg/standard/imports.go` (or your own composition root).
- **Lookup, don't import.** Sibling modules are resolved through `ctx.Module(new(InterfaceType))` / `ctx.Modules(...)` in `Provision`, not by direct package imports — this preserves the pluggable architecture.
- **Don't shell out yourself.** Use `pkg/gotenberg/cmd.go` helpers so logging, context cancellation, and timeouts behave consistently.
- **Per-request scratch dirs.** Allocate via `api.Context.CreateSubDirectory` / the request `*gotenberg.FileSystem`; the GC will clean them up. Never write to absolute paths.
- **PCRE allow/deny lists.** Always use `regexp2`, not `regexp` — the documented syntax for `*-allow-list` / `*-deny-list` flags is .NET-flavored.
- **Flag ⇄ env naming.** Keep flags kebab-case so the auto env-var mapping (`UPPER_SNAKE`) works without special-casing.
- **Engine ordering matters.** When adding a new PDF engine, decide which operations it supports and update both its `PdfEngine` impl *and* the default value of the relevant `--pdfengines-*-engines` flag.
- **Import grouping.** `.golangci.yml` enforces three gci sections in order: standard, default, `prefix(github.com/gotenberg/gotenberg/v8)`. `make fmt` will fix this for you.
- **Linters.** The enabled set includes `gosec`, `errcheck`, `staticcheck`, `bodyclose`, `prealloc`, `dupl`, `exhaustive`, `usetesting`, etc. Tests are excluded from linting (`run.tests: false`).
- **No `tsc`-style equivalents.** Go-only; the `package.json` exists solely for `prettier` to format non-Go files. Don't add JS code or shell out to Node from the binary.
- **Keep all four Dockerfiles in sync** when changing build args, base versions, copy paths or runtime env defaults. The Cloud Run and AWS Lambda overlays expect the base image's user, paths and entrypoint to be stable.
- **Upstream-tracking fork.** Prefer rebasing onto upstream over diverging. Keep AppsCode-specific patches small and isolated (currently: `Dockerfile.ubi`, `--prometheus-metrics-path`, this `AGENTS.md`). The fork uses `main`, not `master`.
- **Sign off commits** with `git commit -s` — CI/upstream expects DCO.
- **License**: see `LICENSE` (MIT; upstream Gotenberg is MIT licensed).
