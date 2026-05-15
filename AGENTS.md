# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

The AppsCode fork of [Gotenberg](https://github.com/gotenberg/gotenberg) — a containerized API for PDF conversion (LibreOffice / Chromium / PDFtk / PDFCPU under the hood). Used by AppsCode services that need to render HTML/markdown/Office docs to PDF.

Module path is unchanged from upstream: `github.com/gotenberg/gotenberg/v8`. Git remote is `appscode-cloud/gotenberg`. This fork tracks upstream and carries AppsCode-specific patches.

## Architecture (upstream layout)

- `cmd/` — entry points (the main gotenberg binary).
- `pkg/` — modular packages, one per conversion module / API surface.
- `build/` — Dockerfile and build scripts (Gotenberg's release image bundles LibreOffice, Chromium, etc.).
- `test/` — integration tests.
- `Makefile` — upstream build harness (local Go toolchain).
- `package.json` — used by the docs/site, not by the binary.

## Common commands

Consult the upstream Makefile for the full target set. Common:

- `make build` — Go build.
- `make tests` — test suite.
- `make lint` — golangci-lint.
- `make image` — build the gotenberg Docker image (heavy — pulls LibreOffice + Chromium).

## Conventions

- Module path is `github.com/gotenberg/gotenberg/v8` (**upstream**); imports must use that.
- **Upstream-tracking** fork. Prefer rebasing onto upstream over diverging. Isolate AppsCode-specific patches.
- License: see `LICENSE` (Apache or MIT — check upstream).
- Sign off commits (`git commit -s`).
- The base branch on this fork is `main` (not `master`).
- The release Docker image is large (~2GB) by design — it bundles every conversion backend. Don't try to slim it without coordinating with downstream consumers.
