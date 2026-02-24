# Reckoner Binary Build Research

Research into building [reckoner](https://github.com/FairwindsOps/reckoner) as a shippable binary using [flox](https://flox.dev), modeled after the [flox-md](https://github.com/Azahorscak/flox-md) build pattern.

---

## 1. What is Reckoner?

Reckoner is a **declarative Helm release management tool** written in Go. It allows users to define multiple Helm chart releases in a single YAML configuration file (a "course file") and manage them as a group. Key capabilities include:

- Install/upgrade multiple Helm releases from a single declarative config
- Install charts from git repositories at specific commits/branches/tags
- Pre/post install hooks at both course-level and release-level
- Namespace creation and management with labels/annotations
- Schema validation of course files
- Diff current cluster state against desired state
- Import existing Helm releases into reckoner's declarative format
- Template rendering with optional ArgoCD Application generation
- Self-update capability (`reckoner update-cli`)

Originally written in Python, reckoner was rewritten in Go (as noted in `DESIGN.md`) to provide a better UX via a pre-compiled binary.

---

## 2. Language and Build Requirements

### Go Version

- **Go 1.20** — specified in `go.mod` line 3 and confirmed in CI (`cimg/go:1.20`, `go1.20.7`)
- Uses `go:embed` directive (requires Go 1.16+, satisfied by 1.20)
- **CGO is disabled** (`CGO_ENABLED=0`) — this is critical for producing static binaries

### Build Command

From the `Makefile`:

```makefile
COMMIT := $(shell git rev-parse HEAD)
VERSION := "0.0.0"

build:
    $(GOBUILD) -o $(BINARY_NAME) -ldflags "-X main.version=$(VERSION) -X main.commit=$(COMMIT) -s -w" -v
```

Key ldflags:
- `-X main.version=$(VERSION)` — injects the version string
- `-X main.commit=$(COMMIT)` — injects the git commit hash
- `-s -w` — strips debug info and DWARF symbols for smaller binary

### Cross-compilation Targets (from `.goreleaser.yml`)

| OS      | Architectures           |
|---------|------------------------|
| Linux   | amd64, arm (v6, v7), arm64 |
| macOS   | amd64, arm64           |
| Windows | amd64, arm (v6, v7), arm64 |

---

## 3. Dependency Analysis

### Go Module Dependencies (Direct — 19 packages)

| Dependency | Purpose |
|---|---|
| `github.com/Masterminds/semver/v3` | Semantic version parsing (helm version checks) |
| `github.com/blang/semver` | Semver for the self-update feature |
| `github.com/fatih/color` | Colored terminal output |
| `github.com/go-git/go-git/v5` | Pure-Go git implementation (cloning charts from git repos) |
| `github.com/gookit/color` | Additional colored output |
| `github.com/imdario/mergo` | Deep merging of structs |
| `github.com/rhysd/go-github-selfupdate` | Self-update from GitHub releases |
| `github.com/sergi/go-diff` | Text diffing (for `reckoner diff`) |
| `github.com/spf13/cobra` | CLI framework |
| `github.com/spf13/pflag` | CLI flag parsing |
| `github.com/stretchr/testify` | Testing assertions (test-only) |
| `github.com/thoas/go-funk` | Functional utilities |
| `github.com/xeipuuv/gojsonschema` | JSON schema validation of course files |
| `gopkg.in/yaml.v3` | YAML parsing |
| `k8s.io/api` | Kubernetes API types |
| `k8s.io/apimachinery` | Kubernetes object metadata/types |
| `k8s.io/client-go` | Kubernetes client (namespace CRUD, kubeconfig) |
| `k8s.io/klog/v2` | Kubernetes-style structured logging |
| `sigs.k8s.io/controller-runtime` | Kubeconfig loading helper |

All Go dependencies are handled by Go modules (`go.mod` / `go.sum`) and resolved at build time. No external dependency manager needed — `go build` handles everything.

### Runtime Dependencies (External Binaries)

| Tool | How Used | Required? |
|---|---|---|
| **helm** (>= 3.0.0) | Found via `exec.LookPath("helm")` in `pkg/helm/helm.go:34`. All helm operations (install, upgrade, template, diff, repo add, dependency build, get manifest, get values, list) are executed by shelling out to the helm binary. | **Required** |
| **kubectl / kubeconfig** | Used implicitly through `k8s.io/client-go` for Kubernetes API access (namespace management, context selection). A valid kubeconfig must be available. | **Required for non-dry-run** |
| **git** | NOT required as an external binary — reckoner uses the pure-Go `go-git` library (`pkg/reckoner/git.go`) | **Not required** |

### Build-time Only Dependencies (Not needed at runtime)

| Tool | Current Use |
|---|---|
| goreleaser v1.20.0 | Release builds (replaced by flox build) |
| golangci-lint | Linting (dev-only) |
| cosign | Artifact signing (release-only) |

---

## 4. Current Build and Distribution System

### Makefile (Local Development)

- `make build` — compiles with ldflags for version/commit injection
- `make test` — runs Go tests with benchmarks, coverage, and vet
- `make lint` — runs `golangci-lint`
- `make build-linux` — cross-compiles for Linux/amd64 with `CGO_ENABLED=0`
- `make build-apple-silicon` — cross-compiles for Darwin/arm64

### GoReleaser (Release Distribution)

The `.goreleaser.yml` handles multi-platform builds, Docker image creation, checksum generation, and cosign signing. This is the current release pipeline but would be replaced by flox build for our purposes.

### Docker Image

The `Dockerfile` is minimal (scratch-based):

```dockerfile
FROM scratch
USER nobody
COPY reckoner /
WORKDIR /
ENTRYPOINT ["/reckoner"]
```

### CI/CD (CircleCI)

- Test job uses `cimg/go:1.20`
- Snapshot builds use goreleaser
- E2E tests run against Kubernetes 1.25, 1.26, 1.27 via KinD
- Release triggered on `v*.*.*` tags

---

## 5. Architecture Overview

```
main.go                          Entry point; embeds JSON schema; calls cmd.Execute()
  │
cmd/
  root.go                        Cobra CLI setup with subcommands:
    plot                         Install/upgrade releases
    template                     Render templates (dry-run)
    diff                         Compare desired vs. actual state
    lint                         Validate course file schema
    convert                      Convert v1 -> v2 course schema
    get-manifests                Get current manifests from cluster
    update                       Conditional install (only if changed)
    import                       Generate course YAML from existing release
    update-cli                   Self-update binary from GitHub
    version                      Print version
  │
pkg/
  course/
    course.go                    Course file parsing (v1/v2), YAML unmarshaling,
                                 env variable expansion, JSON schema validation
    coursev2.schema.json         Embedded JSON schema for validation
    namespace.go                 Namespace config struct/defaults
    shellSecrets.go              ShellExecutor secret backend
    argocd.go                    ArgoCD Application definitions
  │
  helm/
    helm.go                      Helm client wrapper; finds helm binary, executes
                                 helm commands via os/exec
  │
  reckoner/
    client.go                    Main client struct; init, version checks, filter releases
    plot.go                      Plot logic (namespace mgmt -> repos -> hooks -> helm upgrade)
    diff.go                      Diff desired templates vs. current manifests
    update.go                    Conditional update (diff first, plot only if changed)
    git.go                       Git clone/checkout using go-git library
    namespace.go                 Kubernetes namespace CRUD via client-go
    hook.go                      Execute pre/post hooks as shell commands
    import.go                    Import existing helm release to course format
    manifests.go                 Get currently deployed manifests
```

Key design patterns:
- **Helm** interaction is via `exec` (shelling out to the `helm` binary), not via Helm's Go libraries
- **Git** interaction is via pure Go (`go-git`), not shelling out to git
- **Kubernetes** interaction is via `client-go` for namespace operations only
- Version/commit are injected at build time via ldflags
- The JSON schema for course validation is embedded via `go:embed`

---

## 6. Flox Build Pattern (from flox-md Reference)

### Repository Structure

```
flox-md/
├── .flox/
│   ├── .gitattributes        # Marks manifest.lock as generated
│   ├── .gitignore            # Ignores run/, cache/, lib/, log/ but keeps env/
│   ├── env.json              # Flox environment metadata {"name": "flox-md", "version": 1}
│   └── env/
│       ├── manifest.toml     # THE core file — all build logic lives here
│       └── manifest.lock     # Auto-generated lockfile pinning resolved packages
├── FLOX.md                   # Source content
├── README.md                 # Usage/installation instructions
├── result-flox-md            # Symlink -> /nix/store/...-flox-md-1.0.0
└── result-flox-md-log        # Symlink -> build log
```

### Manifest Structure (`manifest.toml`)

The entire build system is defined in a single `.flox/env/manifest.toml` file with these key sections:

```toml
version = 1

# Packages available during development (flox activate)
[install]
bat.pkg-path = "bat"
glow.pkg-path = "glow"

# Environment variables
[vars]
FLOX_MD_VERSION = "1.0.0"

# Runs on `flox activate` (NOT during build)
[hook]
on-activate = '''
  echo "Welcome message"
'''

# Shell functions available during development
[profile]
common = '''
  test_package() { flox build flox-md; }
'''

# The actual build definition
[build.flox-md]
description = "Package description"
version = "1.0.0"
runtime-packages = ["bat", "glow", "ripgrep", "fzf", "less"]
command = '''
  set -euo pipefail
  mkdir -p "$out"/{bin,share/doc/flox-md}
  cp FLOX.md "$out/share/doc/flox-md/"
  cat > "$out/bin/flox-md" << 'EOF'
  #!/usr/bin/env bash
  # ... script content ...
  EOF
  chmod +x "$out/bin/flox-md"
'''

# Supported platforms
[options]
systems = ["x86_64-linux", "aarch64-linux", "x86_64-darwin", "aarch64-darwin"]
```

### Key Build Concepts

1. **`$out` variable**: Provided by flox/Nix — points to the output path in the Nix store. All build artifacts must be placed under `$out`.
2. **FHS conventions**: Executables go to `$out/bin/`, data to `$out/share/`, man pages to `$out/share/man/man1/`.
3. **`runtime-packages`**: Dependencies that are automatically available when the built package is installed.
4. **Sandboxed build**: The build runs in isolation. Only declared dependencies and project source files are available.
5. **Result symlinks**: After `flox build <target>`, a `result-<target>` symlink points to the Nix store output.

### Build Workflow

```bash
flox activate          # Enter dev environment
flox build <target>    # Build the package (runs [build.<target>].command)
./result-<target>/bin/<cmd>  # Test locally
flox publish <target>  # Ship to Flox catalog
```

---

## 7. Applying the Flox Pattern to Reckoner

### What Needs to Happen

To build reckoner with flox, we need a `.flox/env/manifest.toml` that:

1. **Declares Go as a build dependency** in `[install]` so the Go toolchain is available
2. **Declares helm as a runtime dependency** in `[build.reckoner].runtime-packages` so it's available when the built binary is used
3. **Compiles the Go binary** in the `[build.reckoner].command` script with the correct ldflags
4. **Outputs the binary** to `$out/bin/reckoner`
5. **Supports multiple platforms** via `[options].systems`

### Key Considerations

#### Go Compilation in Flox Build Sandbox

- The flox build sandbox **does not have network access**. Go modules must be vendored or pre-fetched.
- **Solution**: Use `go mod vendor` to vendor dependencies into the repo, then build with `-mod=vendor`. Alternatively, the Go module cache can be pre-populated during a hook or the `[install]` section can provide Go with module proxy access.
- The safest approach: **vendor dependencies** (`go mod vendor`) and commit the `vendor/` directory, then build with `go build -mod=vendor`.

#### Version and Commit Injection

- The current build uses `-X main.version=$(VERSION) -X main.commit=$(COMMIT)` ldflags
- In flox, the version can be set via `[vars]` and the commit can be derived from `git rev-parse HEAD` in the build script (if git is available in the sandbox)
- Alternative: set a `[vars].RECKONER_VERSION` and pass it via ldflags, and accept a static commit hash or use `"flox-build"` as the commit identifier

#### CGO

- Reckoner builds with `CGO_ENABLED=0` for static binaries
- This simplifies the flox build — no C compiler or system libraries needed
- The Go toolchain from Nixpkgs is sufficient

#### Helm as Runtime Dependency

- Reckoner finds helm via `exec.LookPath("helm")` at runtime
- Declaring `helm` in `runtime-packages` ensures it's on `$PATH` when installed via flox
- This is one of the major advantages of flox — the user doesn't need to separately install helm

#### Platform Support

- Go cross-compilation (`GOOS`/`GOARCH`) is straightforward
- Flox's `[options].systems` should cover the primary targets: `x86_64-linux`, `aarch64-linux`, `x86_64-darwin`, `aarch64-darwin`
- Note: flox builds natively on each platform (no cross-compilation) — each platform builds with its native Go toolchain

### Proposed Manifest Skeleton

```toml
version = 1

[install]
go.pkg-path = "go"
gnumake.pkg-path = "gnumake"           # Optional: if we want to use existing Makefile
golangci-lint.pkg-path = "golangci-lint"  # Optional: for development linting

[vars]
RECKONER_VERSION = "0.0.0"

[hook]
on-activate = '''
  echo "Reckoner Development Environment"
  echo "================================="
  echo ""
  echo "Build:  flox build reckoner"
  echo "Test:   go test ./..."
  echo "Lint:   golangci-lint run"
'''

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

[options]
systems = ["x86_64-linux", "aarch64-linux", "x86_64-darwin", "aarch64-darwin"]
```

### Open Questions / Risks

1. **Go module vendoring**: The `vendor/` directory is not currently in the repo. We need to run `go mod vendor` and decide whether to commit it or handle it in a pre-build step. Since flox build sandboxes typically lack network access, vendoring is likely required.

2. **Helm package name in Nixpkgs**: The exact `pkg-path` for Helm 3 in the Flox catalog needs to be verified. It may be `kubernetes-helm`, `helm`, or something else. This should be checked with `flox search helm`.

3. **Go version pinning**: The `go` package in Nixpkgs may not be exactly Go 1.20. We may need to specify a version constraint or use a specific attribute path like `go_1_20`. The code should be checked for compatibility with newer Go versions (1.21, 1.22+).

4. **Build sandbox network access**: If `go mod vendor` is not used, the build will fail trying to download modules. Need to confirm flox build sandbox behavior.

5. **Git availability in build sandbox**: If we want to inject the actual commit hash, `git` needs to be available during build and the `.git` directory needs to be accessible in the sandbox. This may not be the case — using a static commit identifier may be more reliable.

6. **Binary size**: With `CGO_ENABLED=0` and `-s -w`, the binary should be reasonably sized. No additional stripping needed.

7. **Test execution**: Tests require a running environment with certain expectations. Tests should be run via `flox activate` (development mode), not during `flox build`.

---

## 8. Summary

| Aspect | Reckoner Current | Flox Build Target |
|---|---|---|
| Build tool | Makefile + GoReleaser | `.flox/env/manifest.toml` |
| Build deps | Go 1.20, goreleaser, cosign | Go (via flox `[install]`) |
| Runtime deps | helm (user-installed) | helm (via `runtime-packages`) |
| Distribution | GitHub Releases, Docker | `flox build` + `flox publish` |
| Dependency mgmt | `go.mod` + network fetch | `go.mod` + vendored (`-mod=vendor`) |
| Cross-platform | GoReleaser matrix | `[options].systems` (native per-platform) |
| Version injection | ldflags via Makefile/goreleaser | ldflags via `[build.reckoner].command` |
| CGO | Disabled | Disabled |

The path to a flox-built reckoner is straightforward:
1. Add `.flox/` configuration with `manifest.toml`
2. Vendor Go dependencies (`go mod vendor`)
3. Write the build command to compile with proper ldflags
4. Declare helm as a runtime dependency
5. Test with `flox build reckoner` and validate the output binary
