# Plan: Add Flox Build Support to Reckoner

## Goal

Add the necessary files to build and ship reckoner as a flox package, following the [flox-md](https://github.com/Azahorscak/flox-md) reference pattern. After this work, a user with flox installed can run `flox build reckoner` to produce a working binary with helm bundled as a runtime dependency.

---

## Steps

### 1. Vendor Go dependencies

The flox build sandbox has **no network access**. Go modules must be available locally at build time.

- Run `go mod vendor` to create a `vendor/` directory containing all dependencies.
- Commit the `vendor/` directory to the repository.
- The build command will use `-mod=vendor` to reference these local copies.

**Why**: Without vendoring, `go build` will attempt to download modules from the internet, which will fail inside the sandboxed flox build environment.

### 2. Create the `.flox/` directory structure

Create the following files, matching the flox-md pattern:

```
.flox/
├── .gitattributes        # Mark manifest.lock as generated
├── .gitignore            # Ignore transient flox state (run/, cache/, lib/, log/)
├── env.json              # Flox environment metadata
└── env/
    └── manifest.toml     # Build definition (core file)
```

#### 2a. `.flox/.gitattributes`

```
/env/manifest.lock linguist-generated=true
```

Marks the lockfile as auto-generated so it is collapsed in diffs and excluded from language stats.

#### 2b. `.flox/.gitignore`

```
run/
cache/
lib/
log/
```

Keeps transient flox runtime state out of version control. The `env/` directory (containing `manifest.toml` and `manifest.lock`) is **not** ignored — it must be committed.

#### 2c. `.flox/env.json`

```json
{
  "name": "reckoner",
  "version": 1
}
```

Identifies this as a flox environment named "reckoner".

#### 2d. `.flox/env/manifest.toml` (core file)

This is the single file that defines the entire build. Sections:

**`[install]`** — Development dependencies available during `flox activate`:
- `go` — Go toolchain (for compiling)
- `gnumake` — GNU Make (to use existing Makefile targets during development)
- `golangci-lint` — Linter (development convenience)

**`[vars]`** — Environment variables:
- `RECKONER_VERSION` — Version string injected into the binary via ldflags

**`[hook]`** — `on-activate` script that prints a welcome message when entering the dev shell with available commands.

**`[profile]`** — Shell functions available during development:
- `test_reckoner()` — convenience function to build and verify

**`[build.reckoner]`** — The build definition:
- `description` — Package description
- `version` — Package version
- `runtime-packages` — `["kubernetes-helm"]` so helm is available on `$PATH` at runtime
- `command` — Shell script that:
  1. Creates `$out/bin/`
  2. Sets `CGO_ENABLED=0`
  3. Runs `go build -mod=vendor` with ldflags for version/commit injection
  4. Outputs binary to `$out/bin/reckoner`

**`[options]`** — Target platforms:
- `x86_64-linux`
- `aarch64-linux`
- `x86_64-darwin`
- `aarch64-darwin`

### 3. Verify helm package name

The research notes that the exact `pkg-path` for Helm 3 in the flox catalog needs verification. The manifest will use `kubernetes-helm` as the runtime package (the standard Nixpkgs attribute name for Helm 3). If this is incorrect, it can be checked with `flox search helm` and adjusted.

### 4. Handle version and commit injection

- **Version**: Read from the `RECKONER_VERSION` variable defined in `[vars]`. This keeps version management in a single place.
- **Commit**: Use the string `"flox-build"` as a static identifier. The build sandbox may not have access to `.git/`, making `git rev-parse HEAD` unreliable. A static string is the safer approach and clearly identifies flox-produced binaries.

### 5. Commit and push

- Commit the `vendor/` directory, `.flox/` directory, and `plan.md`.
- Push to the designated branch.

---

## File Inventory

| File | Action | Purpose |
|------|--------|---------|
| `vendor/` | Create via `go mod vendor` | Vendored Go dependencies for offline build |
| `.flox/.gitattributes` | Create | Mark lockfile as generated |
| `.flox/.gitignore` | Create | Ignore transient flox state |
| `.flox/env.json` | Create | Flox environment metadata |
| `.flox/env/manifest.toml` | Create | Build definition — the core file |
| `plan.md` | Create | This plan |

---

## Decisions and Rationale

| Decision | Rationale |
|----------|-----------|
| Vendor dependencies | Flox build sandbox has no network access; vendoring is the only reliable option |
| Static commit identifier (`"flox-build"`) | `.git/` may not be available in sandbox; avoids fragile workaround |
| `kubernetes-helm` as runtime package | Standard Nixpkgs attribute for Helm 3; bundles helm with the binary automatically |
| `CGO_ENABLED=0` | Matches existing build config; produces static binary with no C dependencies |
| `-s -w` ldflags | Matches existing build config; strips debug symbols for smaller binary |
| Four target platforms | Covers the same Linux + macOS targets as goreleaser, excluding Windows (not a flox target) |

---

## Usage After Implementation

```bash
# Enter development environment
flox activate

# Build the package
flox build reckoner

# Test the built binary
./result-reckoner/bin/reckoner version

# Publish to flox catalog (when ready)
flox publish reckoner
```

---

## Open Items

1. **Helm package name verification** — Confirm `kubernetes-helm` resolves correctly in the flox catalog. Adjust to `helm` or another attribute if needed.
2. **Go version compatibility** — The repo specifies Go 1.20 in `go.mod`. The flox catalog `go` package may provide a newer version. The codebase should be compatible with Go 1.20+, but this should be validated by a successful build.
3. **`manifest.lock` generation** — The lockfile is auto-generated by flox on first `flox activate` or `flox build`. It will be committed after initial generation.
