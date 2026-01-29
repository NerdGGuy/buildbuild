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
| [PULL](PULL/README.md) | Build orchestration | Nix flake with variant definitions |
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
cd PULL

# Enter development shell
nix develop

# Build specific variant
nix build .#release
nix build .#debug
nix build .#asan

# Run tests
./result/bin/test_calculator
```

### Full CI Workflow

Start the poll daemon to automatically detect pushes and build all variants:

```bash
# Set tokens for cache and status uploads (optional)
export CACHE_REPO_TOKEN="ghp_..."
export STATUS_REPO_TOKEN="ghp_..."

# Start the daemon
./polld start
```

The daemon will:
1. Poll GitHub Events API for new pushes and PRs
2. Set commit status to "pending"
3. Update all variants to the detected commit
4. Build and export all 8 variants to NAR format
5. Upload artifacts to the cache repository (if CACHE_REPO_TOKEN is set)
6. Update the status dashboard (if STATUS_REPO_TOKEN is set)
7. Set commit status to "success" or "failure"

Override defaults via environment:

```bash
repo="owner/repo" poll_interval=60 build_timeout=7200 ./polld start
```

### Managing the Poll Daemon

```bash
# Start the daemon (auto-detects token from gh CLI)
./polld start

# Check status
./polld status

# View recent log output
./polld log
./polld log 100    # last 100 lines

# Follow log in real time
./polld follow

# Stop the daemon
./polld stop

# Restart
./polld restart
```

Override defaults via environment:

```bash
repo="owner/repo" poll_interval=60 ./polld start
```

### Running Tests

```bash
# End-to-end CI pipeline test (requires GitHub token + PROJ write access)
./test-ci

# Build and run tests for a specific variant
cd PULL
nix build .#release-test-run
nix build .#asan-test-run
```

The `test-ci` script:
1. Pushes a test commit to PROJ
2. Runs a poll cycle to detect the commit
3. Builds all 8 variants (via `--override-input src`)
4. Checks GitHub commit status is set to "success"

Override defaults via environment:

```bash
repo="owner/repo" timeout=600 ./test-ci
```

### Using the Cache

Add to your flake.nix:

```nix
{
  nixConfig = {
    extra-substituters = [ "https://raw.githubusercontent.com/NerdGGuy/PUSH/main" ];
    extra-trusted-public-keys = [ "buildbuild-cache:BASE64-PUBLIC-KEY" ];
  };
}
```

Then build with automatic cache lookup:

```bash
nix build github:NerdGGuy/PROJ#release
```

### Checking Build Status

View the dashboard at `https://NerdGGuy.github.io/POST/` or embed badges:

```markdown
![release](https://NerdGGuy.github.io/POST/badges/release.svg)
![asan](https://NerdGGuy.github.io/POST/badges/asan.svg)
```

## Data Flow

1. **POLL** detects changes via GitHub Events API (push, pull request)
2. **POLL** sets commit status to "pending" and triggers build
3. **PULL** updates source definitions and builds all variants
4. **PULL** exports builds to NAR format
5. **PUSH** stores build outputs as uncompressed NARs with content-addressed logs
6. **POST** publishes status JSON/HTML to GitHub Pages dashboard
7. **POLL** updates commit status to "success" or "failure"

## Repository Structure

```
buildbuild/
├── polld                    # Daemon management (start/stop/status/log)
├── test-ci                  # End-to-end CI pipeline test
├── POLL/                    # Change detection
│   ├── poll                 # Main polling script
│   └── lib/api.sh           # GitHub API wrapper
├── PULL/                    # Build orchestration
│   ├── flake.nix            # Nix flake definition
│   ├── variants/            # Per-variant configurations
│   ├── nix/                 # Nix helper modules
│   ├── toolchains/          # Compiler configurations
│   └── *.sh                 # CI scripts
├── PUSH/                    # Binary cache
│   └── upload.sh            # Upload script
├── POST/                    # Status dashboard
│   ├── index.html           # Dashboard page
│   ├── style.css            # Styling
│   ├── app.js               # Client-side logic
│   ├── update.sh            # Status update script
│   ├── data/                # Status data
│   └── badges/              # SVG badges
└── PROJ/                    # Example C++ project
    ├── src/                 # Source code
    ├── tests/               # Tests
    └── Makefile             # Build system
```

## Configuration

### Environment Variables

| Variable | Component | Description |
|----------|-----------|-------------|
| `repo` | POLL | Repository in `owner/repo` format |
| `github_token` | POLL | GitHub token for API access |
| `poll_interval` | POLL | Seconds between polls (default: 60) |
| `GITHUB_TOKEN` | PULL/PUSH/POST | GitHub token for API access |
| `CACHE_REPO_TOKEN` | PULL/PUSH | Token with write access to cache repo |
| `STATUS_REPO_TOKEN` | PULL/POST | Token with write access to status repo |
| `NIX_SIGNING_KEY` | PULL | Secret key for NAR signing |

### PULL Configuration

Edit `PULL/cache/config.json`:

```json
{
  "source_repo": {
    "owner": "NerdGGuy",
    "repo": "PROJ",
    "default_ref": "main"
  },
  "cache_repo": {
    "owner": "NerdGGuy",
    "repo": "PUSH"
  },
  "status_repo": {
    "owner": "NerdGGuy",
    "repo": "POST"
  },
  "signing": {
    "key_name": "buildbuild-cache",
    "secret_env": "NIX_SIGNING_KEY"
  }
}
```

## Requirements

- Bash 4.0+
- Nix 2.18+ with flakes enabled
- curl
- jq
- nix-prefetch-github

## License

MIT
