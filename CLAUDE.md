# CLAUDE.md - AI Assistant Guide for AIStore

## Project Overview

AIStore (AIS) is a lightweight, open-source distributed object storage system purpose-built for AI and machine learning workloads. It is developed by NVIDIA and licensed under the MIT license.

Key characteristics:
- Scales linearly with each added storage node and disk
- Deploys on commodity Linux hardware (VMs, bare-metal, Kubernetes, Docker)
- Provides S3-compatible REST API and native Go/Python SDKs
- Supports multiple cloud backends (AWS S3, Google Cloud, Microsoft Azure)
- Implements erasure coding, mirroring, rebalancing, and self-healing
- Offloads ETL operations close to data

## Repository Structure

```
aistore/
├── ais/                 # Core daemon logic (proxies, targets, cluster coordination)
│   └── test/           # Integration tests (~40 test files)
├── api/                # Public API definitions
│   └── apc/           # API constants and action definitions
├── cmd/                # Command-line executables
│   ├── aisnode/       # Main daemon binary
│   ├── cli/           # CLI tool (separate Go module)
│   ├── authn/         # Authentication server
│   ├── aisloader/     # Load generator/benchmarking
│   └── xmeta/         # Metadata inspection tool
├── cmn/                # Common utilities and shared code
│   ├── cos/           # Common operations (strings, IO, math, checksums)
│   ├── archive/       # TAR/ZIP/TGZ archive handling
│   ├── nlog/          # Logging
│   ├── k8s/           # Kubernetes utilities
│   ├── feat/          # Feature flags
│   ├── kvdb/          # Key-value database
│   ├── atomic/        # Atomic operations
│   └── debug/         # Debug utilities
├── core/               # Core abstractions (LOM, interfaces)
├── xact/               # Extended actions (async batch jobs)
│   ├── xs/            # Concrete xaction implementations
│   └── xreg/          # Xaction registry
├── fs/                 # Filesystem abstraction (mountpaths, FQN parsing)
├── transport/          # Inter-node communication (HTTP streaming)
├── ec/                 # Erasure coding
├── mirror/             # Data mirroring
├── reb/                # Rebalancing logic
├── memsys/             # Memory system (SGL/Slab allocators)
├── stats/              # Statistics collection
├── space/              # Storage capacity management
├── hk/                 # Housekeeping routines
├── ios/                # I/O statistics
├── res/                # Resource management
├── nl/                 # Notifications
├── ext/                # External integrations
├── python/             # Python SDK, PyTorch integration
├── docs/               # Documentation
├── deploy/             # Deployment scripts
│   ├── dev/local/     # Local development deployment
│   ├── dev/docker/    # Docker-based deployment
│   └── prod/          # Production deployments
├── scripts/            # Utility and CI scripts
├── bench/              # Benchmarking tools
├── tools/              # Development and testing utilities
├── 3rdparty/           # Third-party dependencies
├── Makefile            # Build automation
├── go.mod              # Main Go module (github.com/NVIDIA/aistore)
└── .golangci.yml       # Linter configuration
```

## Languages and Versions

| Language | Version | Usage |
|----------|---------|-------|
| Go | 1.22.3+ | Primary language (>90% of codebase) |
| Python | 3.x | SDK, PyTorch integration, tests |
| Bash | - | Deployment and CI scripts |

## Build Commands

All builds are driven through the Makefile:

```bash
# Build all binaries
make all

# Build individual components
make node              # aisnode daemon
make cli               # CLI (static binary, CGO disabled)
make authn             # Authentication server
make aisloader         # Load generator

# Deployment
make deploy            # Build and deploy cluster locally (interactive)
make run               # Deploy using existing config (no regeneration)
make restart           # Kill and restart cluster
make kill              # Kill all locally deployed daemons
make clean             # Remove all AIS files and binaries

# Build flags
MODE=debug make node                    # Debug build with asserts
TAGS=aws,azure,gcp make node           # Enable cloud backends
TAGS=nethttp make node                  # Use net/http instead of fasthttp
TAGS=statsd make node                   # Use StatsD instead of Prometheus
GORACE='log_path=/tmp/race' make deploy # Race detector
```

## Testing

### Test Framework
- **Go tests**: Ginkgo v2 + Gomega (BDD-style)
- **Python tests**: pytest
- Tests require a running AIStore cluster

### Running Tests

```bash
# Short tests (unit + basic integration, 30m timeout)
BUCKET=ais://test make test-short

# Long tests (full integration suite, 4h timeout)
BUCKET=<bucket-name> make test-long

# Run specific test pattern
BUCKET=ais://test make test-run RE=<TestPattern>

# Benchmarks
make test-bench

# Python tests
cd python && make python_tests
```

### Test Configuration
- Tests run sequentially (`-p 1`) due to shared cluster state
- Short tests need: 3+ targets, 1+ proxy
- Long tests may need cloud bucket access (AWS/GCP)
- Test utilities are in `/tools/` (tassert, tlog, trand, client)
- Integration tests are centralized in `ais/test/`

### Test File Conventions
- Unit tests: `*_test.go` alongside source code
- Integration tests: `ais/test/*_test.go`
- Use `testpackage` linter rule (tests in separate `_test` package)

## Code Quality

### Linting

```bash
make lint              # Run golangci-lint (requires v1.59.1)
make fmt-check         # Check Go/Python formatting
make fmt-fix           # Fix formatting
make spell-check       # Check spelling
make spell-fix         # Fix spelling
make ci                # Run all CI checks
```

### Linter Configuration (.golangci.yml)
- 42 linters enabled including: revive, govet, errcheck, gosec, staticcheck, gofmt, goimports
- Build tags for linting: `hrw`, `aws`, `azure`, `gcp`
- Key rules:
  - `math/rand` is forbidden (use `math/rand/v2`)
  - US English spelling enforced
  - Copyright headers required
  - Import ordering enforced (stdlib, external, internal)
  - Test files must use separate test package

### MessagePack Code Generation

```bash
make msgp-update       # Regenerate MessagePack serialization code
```

## Code Conventions

### Package Naming
- Lowercase, no underscores: `ais`, `cmn`, `xact`, `fs`
- Descriptive sub-packages: `cmn/cos`, `cmn/nlog`, `api/apc`

### File Naming
- Descriptive: `daemon.go`, `proxy.go`, `target.go`
- Prefix patterns: `prx*.go` (proxy-specific), `tgt*.go` (target-specific)
- Transport variants: `*_fasthttp.go`, `*_nethttp.go`
- OS-specific: `*_linux.go`, `*_darwin.go`

### Type/Variable Naming
- Common abbreviations: `bck` (bucket), `obj` (object), `md` (metadata), `ctx` (context), `xact` (extended action)
- Key acronyms: `LOM` (Local Object Metadata), `FQN` (Fully Qualified Name)
- Struct receivers: single letter or short abbreviation (`d *Daemon`, `t *Target`, `p *Proxy`)
- Constants in `api/apc/`: `ActRebalance`, `ActPrefetch`, etc.

### File Organization
1. Copyright header (NVIDIA 2018-2024)
2. Package declaration with doc comment
3. Imports (grouped: stdlib, external, internal)
4. Constants
5. Type definitions
6. Interface definitions
7. Exported functions/methods
8. Unexported functions/methods

### Error Handling
- Explicit error returns (no panic except for critical failures)
- Custom error types in `cmn/err.go`
- HTTP status code mapping for API errors
- Guard clauses with early returns preferred

### Copyright Header
All Go files must have a copyright header:
```go
// Package <name> ...
// Copyright (c) 2018-2024, NVIDIA Corporation. All rights reserved.
```

## Architecture

### Node Types
1. **Proxy (Gateway)** - Control plane: request routing, cluster map, metadata sync
2. **Target (Storage Node)** - Data plane: object storage, mountpath management, xactions

### Key Abstractions

- **LOM (Local Object Metadata)** (`core/lom.go`): Lightweight object metadata managing attributes, xattrs, integrity
- **Xactions** (`xact/`): Async batch operations (rebalance, prefetch, download, erasure coding)
- **Cluster Map** (`ais/clustermap.go`): Distributed membership consensus
- **Transport** (`transport/`): Inter-node HTTP streaming (fasthttp default, net/http optional)
- **Memsys** (`memsys/`): SGL/Slab memory allocators to reduce GC pressure

### Configuration
- Hierarchical: global (cluster) + local (node-specific)
- Config directory: `~/.ais` (node), `~/.config/ais` (apps)
- Logs: `/tmp/ais`
- Default ports: 8080 (API), 9080 (intra-control), 10080 (intra-data)

### Cloud Backends
Enabled via build tags: `aws`, `azure`, `gcp`. Provider-agnostic bucket abstraction in `cmn/bck.go`.

## Local Development Setup

```bash
# Quick local cluster (3 targets, 1 proxy)
make deploy

# Or use clean_deploy script with custom configuration
scripts/clean_deploy.sh --target-cnt 3 --proxy-cnt 1 --mountpath-cnt 2

# Kill running cluster
make kill

# View logs
ls /tmp/ais/log/
```

Environment variables for deployment:
- `AIS_USE_HTTPS=false` - HTTPS support
- `AIS_AUTHN_ENABLED=false` - Authentication
- `DEPLOY_AS_NEXT_TIER=1` - Multi-tier setup

## CI/CD

### GitHub Actions (`.github/workflows/`)
- **lint.yml**: golangci-lint v1.59.1 with multiple build tag combinations
- **build.yml**: Multi-platform builds (Linux + macOS) with 8 configurations
- **test-short.yml**: Integration tests (3 targets, 1 proxy, 2 mountpaths)
- **test-python.yml**: Python SDK tests
- **docker.yml**: Docker image builds

### GitLab CI (`.gitlab-ci.yml`)
- Stages: build, test-short, test-long, security
- Checkmarx security scanning
- Cloud backend integration tests

## Contribution Requirements

1. **Sign-off**: All commits must be signed (`git commit -s`)
2. **Squash**: PRs with multiple commits must be squashed
3. **Format**: Must pass `make fmt-check` and `make spell-check`
4. **Lint**: Must pass `make lint`
5. **Tests**: Must pass `make test-short` at minimum

## Key Dependencies

| Dependency | Purpose |
|------------|---------|
| fasthttp | Default HTTP transport (alternative: net/http) |
| msgp | MessagePack serialization |
| prometheus | Metrics collection (alternative: StatsD) |
| ginkgo/gomega | Testing framework |
| AWS/GCP/Azure SDKs | Cloud backend support |
| k8s client-go | Kubernetes integration |
| lz4, brotli | Compression |
| reedsolomon | Erasure coding |

## Important Notes for AI Assistants

- The CLI (`cmd/cli/`) is a separate Go module with its own `go.mod`
- Default transport is `fasthttp`; `net/http` is selected via `TAGS=nethttp`
- Default metrics is Prometheus; StatsD is selected via `TAGS=statsd`
- Never use `math/rand`; always use `math/rand/v2`
- Integration tests share cluster state and must run sequentially
- The `tools/` directory contains test utilities, not production code
- When modifying serialization: run `make msgp-update` to regenerate MessagePack code
- Build tags control optional features: cloud backends, transport, metrics, debug
