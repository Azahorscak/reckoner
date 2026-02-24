# Plan: Build Reckoner as a Flox App

Implementation plan for packaging reckoner as a shippable flox app, following flox best practices as documented in the [flox-md](https://github.com/Azahorscak/flox-md) reference pattern.

---

## Prerequisites

- `flox` CLI installed
- Go 1.20+ available (will also be declared in manifest)
- Access to the reckoner repository

---

## Step 1: Vendor Go Dependencies

The flox build sandbox has **no network access**. All Go module dependencies must be vendored into the repository so `go build -mod=vendor` can succeed without downloading anything.

```bash
go mod vendor
```

This creates a `vendor/` directory containing all 60+ transitive dependencies. The `vendor/` directory must be committed to the repository since the flox build sandbox cannot fetch modules at build time.

**Files created:**
- `vendor/` directory (committed to git)

**Verification:**
```bash
CGO_ENABLED=0 go build -mod=vendor -o reckoner -ldflags "-X main.version=test -X main.commit=test -s -w" .
./reckoner version
```

---

## Step 2: Initialize the Flox Environment

Create the `.flox/` directory structure that flox expects.

```bash
flox init
```

This generates:
- `.flox/env.json` — environment metadata
- `.flox/env/manifest.toml` — the core build/environment definition (empty template)
- `.flox/.gitignore` — ignores runtime artifacts (run/, cache/, lib/, log/)
- `.flox/.gitattributes` — marks `manifest.lock` as generated

**All generated `.flox/` files should be committed to git.**

---

## Step 3: Configure `manifest.toml`

Replace the generated `.flox/env/manifest.toml` with the full build configuration. This single file defines the development environment, build process, and runtime dependencies.

```toml
version = 1

# ── Development Environment ────────────────────────────────────────
# Packages available when running `flox activate` (dev shell).

[install]
go.pkg-path = "go"
gnumake.pkg-path = "gnumake"
golangci-lint.pkg-path = "golangci-lint"

# ── Environment Variables ──────────────────────────────────────────

[vars]
RECKONER_VERSION = "0.0.0"

# ── Activation Hook ───────────────────────────────────────────────
# Runs each time `flox activate` starts a dev shell.

[hook]
on-activate = '''
  echo "Reckoner Development Environment"
  echo "================================="
  echo ""
  echo "Build:  flox build reckoner"
  echo "Test:   go test ./..."
  echo "Lint:   golangci-lint run"
'''

# ── Shell Profile ─────────────────────────────────────────────────
# Helper functions available in the dev shell.

[profile]
common = '''
  test_reckoner() {
    echo "Building reckoner..."
    if flox build reckoner; then
      echo "Build successful!"
      ./result-reckoner/bin/reckoner version
    else
      echo "Build failed"
      return 1
    fi
  }
'''

# ── Build Definition ──────────────────────────────────────────────
# `flox build reckoner` executes this to produce the package.

[build.reckoner]
description = "Declarative Helm release management tool"
version = "0.0.0"
runtime-packages = ["kubernetes-helm"]
command = '''
  set -euo pipefail

  mkdir -p "$out/bin"

  VERSION="${RECKONER_VERSION:-0.0.0}"
  COMMIT="flox-build"

  CGO_ENABLED=0 go build \
    -mod=vendor \
    -o "$out/bin/reckoner" \
    -ldflags "-X main.version=${VERSION} -X main.commit=${COMMIT} -s -w" \
    .

  echo "Built reckoner ${VERSION} -> $out/bin/reckoner"
'''

# ── Platform Support ──────────────────────────────────────────────

[options]
systems = ["x86_64-linux", "aarch64-linux", "x86_64-darwin", "aarch64-darwin"]
```

### Key design decisions in this manifest

| Decision | Rationale |
|---|---|
| `go.pkg-path = "go"` | Uses the default Go from the flox catalog. Reckoner targets Go 1.20 but has no known incompatibilities with newer versions. If issues arise, pin with a version constraint. |
| `-mod=vendor` | Required because the build sandbox has no network access. |
| `CGO_ENABLED=0` | Matches upstream build; produces fully static binaries with no libc dependency. |
| `runtime-packages = ["kubernetes-helm"]` | Helm is found via `exec.LookPath("helm")` at runtime. Declaring it here ensures `helm` is on `$PATH` when installed via flox — users don't need to install helm separately. |
| `COMMIT="flox-build"` | Git metadata is unavailable in the sandbox. Using a static identifier is more reliable than trying to make `.git/` and `git` available in the sandbox. |
| `version` in `[vars]` and `[build.reckoner]` | Kept in sync manually. Update both when cutting a release. |

---

## Step 4: Verify the Helm Package Name

The exact `pkg-path` for Helm 3 in the flox catalog needs to be confirmed before the first build.

```bash
flox search helm
```

The manifest above uses `kubernetes-helm` as the runtime package. If the search shows a different name (e.g., `helm`, `kubernetes-helm-wrapped`), update the `runtime-packages` array in `manifest.toml` accordingly.

---

## Step 5: Build and Test

### Build the package

```bash
flox activate
flox build reckoner
```

This produces a `result-reckoner` symlink pointing to the Nix store output.

### Verify the binary

```bash
./result-reckoner/bin/reckoner version
```

Expected output should show the version and `flox-build` as the commit.

### Verify helm is available as a runtime dependency

```bash
./result-reckoner/bin/reckoner plot --help
```

The binary should run without errors about missing helm (helm itself is bundled as a runtime dependency).

---

## Step 6: Update `.gitignore`

Add the flox build result symlinks to the project's `.gitignore` so they are not committed:

```
# Flox build artifacts
result-*
```

The `.flox/.gitignore` handles the flox-internal runtime directories, but the top-level `result-*` symlinks need to be excluded at the project level.

---

## Step 7: Commit All Changes

The following should be committed:

| Path | Description |
|---|---|
| `vendor/` | Vendored Go dependencies (required for sandboxed builds) |
| `.flox/env.json` | Flox environment metadata |
| `.flox/env/manifest.toml` | Build and environment configuration |
| `.flox/env/manifest.lock` | Resolved package versions (auto-generated) |
| `.flox/.gitattributes` | Marks lock file as generated |
| `.flox/.gitignore` | Ignores flox runtime directories |
| `.gitignore` (update) | Excludes `result-*` symlinks |

---

## Open Questions to Resolve During Implementation

1. **Helm package name**: The exact `pkg-path` for Helm 3 must be confirmed via `flox search helm`. The manifest uses `kubernetes-helm` as a best guess.

2. **Go version compatibility**: The project specifies Go 1.20 in `go.mod`. The flox catalog default Go may be newer. If the build fails due to version incompatibilities, pin Go with `go.pkg-path = "go_1_20"` or update the code to be compatible.

3. **Vendor directory size**: Vendoring ~60 transitive dependencies will add significant size to the repository. This is a known trade-off required by the sandboxed build model.

4. **Version management workflow**: The version is hardcoded in two places (`[vars].RECKONER_VERSION` and `[build.reckoner].version`). A release process should update both. This could be automated with a script or CI step in the future.

---

## Summary

The implementation is 7 steps:

1. **Vendor dependencies** — `go mod vendor` and commit `vendor/`
2. **Initialize flox** — `flox init` to create `.flox/` scaffolding
3. **Write manifest** — Configure `.flox/env/manifest.toml` with build definition, dev tools, and runtime dependencies
4. **Verify helm name** — `flox search helm` to confirm the correct package path
5. **Build and test** — `flox build reckoner` and verify the output binary
6. **Update gitignore** — Exclude `result-*` symlinks
7. **Commit** — Commit all new and modified files
