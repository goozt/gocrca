# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`goca` is a Go HTTP API server that acts as a small private Certificate Authority. It generates and serves a Root CA + Intermediate CA, signs client/node certificates over HTTP, persists issuance/revocation state to a local gob-encoded DB (`ca.db`), and serves a CRL. Module path: `github.com/goozt/gopgbase/infra/ca`. Go version: 1.25.8. Standard library only (no third-party Go dependencies in `go.mod`).

## Common commands

Development tasks are driven by [Taskfile](https://taskfile.dev). Install with `task init` (installs `air`, `ineffassign`, `misspell`, `gocyclo`, `govulncheck`).

- `task` — run the dev server via `air` with `-g -c` flags (auto-generates CA + client root cert on boot, hot-reloads).
- `task build` — `go build -o goca .`
- `task test` — `go test ./...`
- Run a single test: `go test ./internal/api -run TestName` (or `-run TestName/subtest` for subtests).
- `task ready` — pre-commit gate: `go vet`, `gofmt -l -s -w`, `ineffassign`, `misspell -w`, `gocyclo -over 15`, `govulncheck`. Run this before declaring work done.
- `task docker:build` / `task docker:run` / `task docker:clean` / `task docker:rmv` — Docker lifecycle. The container is `scratch`-based, runs as UID 1000, and uses `VOLUME /.rootCA` and `/.ca`.

Run the binary directly:

```
./goca -p 8000 -g -c -root .rootCA [certs_dir]
```

Key flags: `-g` generate CA if missing, `-f` force regen, `-c` also issue a root client cert, `-r/-root` root CA dir (default `.rootCA`), `--tls-cert` + `--tls-key` enable HTTPS. The positional `certs_directory` (or `CERTS_DIR` env var) holds the **intermediate** CA (`ca.crt`, `ca.key`); default `./.ca`.

## Architecture

### Two-tier PKI

There is a strict separation between two directories, both required at runtime:

- **Root CA dir** (`-root`, default `.rootCA`): holds `rootCA.crt`, `rootCA.key`, and the persistent state DB `ca.db`. 15-year RSA root, self-signed, `MaxPathLen: 1`.
- **Intermediate CA dir** (positional arg or `CERTS_DIR`, default `./.ca`): holds `ca.crt`, `ca.key`, optionally `client.root.crt`/`client.root.key`. ECDSA intermediate signed by the root. All issued client/node certs are signed by the **intermediate**, never the root.

`utils.GetCertDir` / `utils.GetRootCertDir` are the single source of truth for these paths at runtime — they cache values resolved by `utils.VerifyCertDir` during startup. Do not re-derive paths elsewhere.

### Request pipeline

`main.go` wires the middleware chain in this exact outer-to-inner order: `recovery → logging → securityHeaders → rateLimit → errorHandling → mux`. The mux registers `/health` (public) and `/ca/*` (wrapped in `authMiddleware`, which is a no-op when `API_KEY` env is unset and otherwise requires `X-API-Key`). All `/ca/*` routes are defined in `internal/api/router.go` via `CaHandler.RegisterRoutes()` and mounted with `http.StripPrefix("/ca", ...)`.

`errorHandlingMiddleware` uses a `blockingResponseWriter` (see `response.go`) to buffer downstream handler output, so non-JSON 4xx/5xx responses can be rewritten as the canonical `utils.APIError` JSON shape. Be careful when adding handlers that stream — they will be buffered.

`middlewares.go` contains an in-process per-IP sliding-window rate limiter (`60/min`, stdlib-only). It is package-global (`defaultLimiter`) and starts a cleanup goroutine on construction.

### Certificate operations

- `internal/ca/` — all cryptographic primitives. `common.go` builds `pkix.Name` subjects and computes SKIDs; `ca.go`/`interca.go`/`client.go`/`server.go` are the four cert-template + sign flows; `sign.go` covers `SignCertificate`, `SignCSR`, and CRL building (`CreateCRL`, `CreateCRLFromRevocations`, 7-day validity). `file.go` handles PEM I/O and validates that a loaded key matches its cert's public half.
- `internal/api/handlers.go` — REST handlers. A package-level `certOpMu sync.Mutex` serialises create/delete operations to avoid TOCTOU on cert files. Hostname validation is RFC 1123/1035 style with explicit IP-literal handling. `inCertDir` resolves symlinks to prevent directory escape.
- Top-level `ca.go` (in `package main`) contains the three bootstrap orchestrators called only from `generateCerts`: `GenerateCA`, `GenerateInterCA`, `GenerateClientRootCert`. Library code in `internal/ca` should not be moved here.

### Persistence

`internal/db/` is a single-file gob-encoded store living at `<rootCertDir>/ca.db`. `DB.persistLocked` writes atomically via `tmp` + `rename`. The global is initialised by `db.InitDB()` in `main.init()` — it `log.Fatal`s on failure, so any test that imports `main` will hit this. Use `db.GetDB()` from handlers. State tracked: `Issued` (keyed by hostname), `Revoked` (keyed by serial hex), and a monotonic `NextCRLNum`.

## Conventions worth knowing

- Logging is `log/slog` with JSON handler; level set via `LOG_LEVEL` env (`debug|warn|error`, default `info`). Use structured key/value logging consistently.
- Error responses always go through `utils.WriteError` / `utils.WriteJSON` to produce the `APIError` envelope.
- Security headers (`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Cache-Control: no-store`) are applied globally; do not strip them per-handler.
- File permissions: certs `0644`, keys `0600`, dirs `0755`. Maintain these when adding new write paths.
- The Docker image is `FROM scratch`, runs as UID 1000, with `/etc/passwd` + `/etc/group` copied from the builder. No shell available in the runtime image.
