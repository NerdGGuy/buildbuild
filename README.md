# buildbuild

A lightweight, webhook-free CI system for Nix-based C++ projects. Produces multiple build variants (release, debug, sanitizers, coverage, fuzzing) and stores artifacts in a Git-based binary cache.

## Overview

buildbuild uses standard Git repositories as binary caches with uncompressed NAR files, enabling Git's native delta compression to efficiently deduplicate similar builds across variants and versions. No Git LFS, S3, or dedicated cache infrastructure required.

```
┌──────────────────┐         ┌──────────────────┐
│   SOURCE REPO    │────────▶│      POLL        │
│   (C++ code)     │  push   │  change detect   │
└──────────────────┘         └────────┬─────────┘
                                      │
                                      ▼
┌──────────────────┐         ┌──────────────────┐
│      PUSH        │◀────────│      PULL        │
│  binary cache    │  export │  build variants  │
└────────┬─────────┘         └──────────────────┘
         │
         ▼
┌──────────────────┐
│      POST        │
│    dashboard     │
└──────────────────┘
```

## Components

| Component | Purpose | Details |
|-----------|---------|---------|
| [POLL](POLL/README.md) | Change detection | Monitors source repos via GitHub Events API |
| [PULL](PULL/README.md) | Build orchestration | Nix expressions, variant definitions, CI workflows |
| [PUSH](PUSH/README.md) | Artifact storage | Git-based binary cache with uncompressed NARs |
| [POST](POST/README.md) | Status reporting | GitHub Pages dashboard with badges |

## Build Variants

| Variant | Purpose | Use Case |
|---------|---------|----------|
| `release` | Production deployment | Shipping binaries |
| `debug` | Development | Daily development |
| `asan` | Address sanitizer | Buffer overflows, use-after-free |
| `ubsan` | Undefined behavior sanitizer | Signed overflow, null deref |
| `tsan` | Thread sanitizer | Data races, deadlocks |
| `msan` | Memory sanitizer | Uninitialized values |
| `coverage` | Code coverage | Test coverage analysis |
| `fuzz` | Fuzzing | Automated vulnerability discovery |

## Quick Start

### Building Locally

```bash
# Enter development shell
nix develop

# Build specific variant
nix build .#release
nix build .#asan

# Run tests with sanitizer
nix build .#asan && ./result/bin/test-suite
```

### Using the Cache

```nix
{
  nixConfig = {
    extra-substituters = [ "https://raw.githubusercontent.com/ORG/CACHE-REPO/main" ];
    extra-trusted-public-keys = [ "cache-name:BASE64-PUBLIC-KEY" ];
  };
}
```

```bash
# Automatic cache lookup
nix build github:org/project#release
```

### Checking Build Status

View the dashboard at `https://ORG.github.io/STATUS-REPO/` or embed badges:

```markdown
![release](https://ORG.github.io/STATUS-REPO/badges/release.svg)
```

## Data Flow

1. **POLL** detects changes via GitHub Events API (push, tag, release)
2. **PULL** updates source definitions and builds all variants in parallel
3. **PUSH** stores build outputs as uncompressed NARs with content-addressed logs
4. **POST** publishes status JSON/HTML to GitHub Pages dashboard

## Requirements

- Bash 4.0+
- Nix 2.18+ with flakes enabled
- gh CLI 2.0+
- curl, jq, timeout (coreutils)

## License

MIT
