<!-- GitHub Copilot Instructions for the Gotenberg repo -->
# Gotenberg — Copilot Instructions

## Architecture Overview

Gotenberg is a containerized API for PDF conversion using Chromium, LibreOffice, and PDF tools.

- **Module system** (`pkg/gotenberg/`): Caddy-inspired plugin architecture. Core interfaces: `Module`, `Provisioner`, `Validator`, `App`, `Router`.
- **Standard modules** (`pkg/modules/`): `api`, `chromium`, `libreoffice`, `pdfengines`, `qpdf`, `pdfcpu`, `pdftk`, `exiftool`, `prometheus`, `webhook`, `logging`.
- **Module registration**: Each module has `init()` calling `gotenberg.MustRegisterModule()`. Modules are imported via `pkg/standard/imports.go`.
- **Binary entry**: `cmd/gotenberg/main.go` imports `pkg/standard` to load all modules, then calls `gotenbergcmd.Run()`.

## Developer Workflows

```bash
make build           # Build Docker image (requires .env with DOCKERFILE, DOCKER_REGISTRY)
make run             # Run container locally with all CLI flags
make test-unit       # go test -race ./...
make test-integration TAGS="chromium"  # Long-running; uses testcontainers-go
make lint && make fmt                  # golangci-lint
make lint-prettier && make prettify    # npx prettier for non-Go files
```

Integration tests use Cucumber/Godog with `.feature` files in `test/integration/features/`.

## Module Patterns

### Creating a New Module
1. Create package under `pkg/modules/yourmodule/`
2. Implement `gotenberg.Module` interface with `Descriptor()` returning `ModuleDescriptor{ID, FlagSet, New}`
3. Register in `init()`: `gotenberg.MustRegisterModule(new(YourModule))`
4. Add import to `pkg/standard/imports.go`: `_ "github.com/gotenberg/gotenberg/v8/pkg/modules/yourmodule"`

### Implementing HTTP Routes
Implement `api.Router` interface:
```go
func (mod *YourModule) Routes() ([]api.Route, error) {
    return []api.Route{{Method: http.MethodPost, Path: "/forms/yourmodule/action", Handler: handler}}, nil
}
```

### Handling Form Data
Use `api.Context.FormData()` for multipart requests:
```go
form := ctx.FormData().
    String("optionalField", &val, "default").
    MandatoryString("requiredField", &required).
    Bool("flag", &flag, false).
    Duration("timeout", &timeout, 30*time.Second)
if err := form.Validate(); err != nil { return err }
```

### CLI Flags
Define in `Descriptor().FlagSet`, access via `ctx.ParsedFlags().MustString("flag-name")`.
Convention: `--module-name-flag-name` CLI → `MODULE_NAME_FLAG_NAME` env var.

## Key Interfaces

| Interface | Purpose | Example |
|-----------|---------|---------|
| `gotenberg.PdfEngine` | PDF operations (merge, split, convert) | `pkg/modules/qpdf/`, `pkg/modules/pdfcpu/` |
| `api.Router` | HTTP route registration | `pkg/modules/chromium/routes.go` |
| `api.MiddlewareProvider` | Custom middleware | `pkg/modules/prometheus/` |
| `gotenberg.ProcessSupervisor` | External process lifecycle | Chromium browser management |

## Integration Tests

- Feature files: `test/integration/features/*.feature`
- Scenario logic: `test/integration/scenario/`
- Run specific tags: `make test-integration TAGS="chromium-convert-html"`
- Available tags: `chromium`, `libreoffice`, `pdfengines`, `webhook`, `prometheus-metrics`, etc.

## File Reference

| Path | Purpose |
|------|---------|
| `pkg/gotenberg/modules.go` | Module registration and lifecycle |
| `pkg/gotenberg/context.go` | Module provisioning context |
| `pkg/modules/api/api.go` | HTTP server and route management |
| `pkg/modules/api/formdata.go` | Form data parsing helpers |
| `pkg/modules/chromium/chromium.go` | Chromium module (good reference) |
| `build/Dockerfile` | Multi-stage Docker build |
| `Makefile` | All CLI flags and env vars |

## Conventions

- Flags: `--kebab-case` CLI, `UPPER_SNAKE_CASE` env in Makefile
- Module IDs: `snake_case` (e.g., `pdf_engines`)
- Unit tests: `*_test.go` alongside source
- Mocks: `mocks.go` in each package
- Errors: Sentinel errors as package-level `var` (e.g., `ErrInvalidPrinterSettings`)
