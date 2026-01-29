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

## Requirements

- Bash 4.0+
- [Nix 2.18+](https://nixos.org/download/) with flakes enabled
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated, or a `github_token` env var

## Setup

### 1. Clone with submodules

```bash
git clone --recurse-submodules https://github.com/NerdGGuy/buildbuild.git
cd buildbuild
```

### 2. Configure your source repo

Edit `PULL/cache/config.json` to point at your project:

```json
{
  "source_repo": {
    "owner": "YOUR_USER",
    "repo": "YOUR_PROJECT",
    "default_ref": "main"
  },
  "cache_repo": {
    "owner": "YOUR_USER",
    "repo": "YOUR_CACHE_REPO"
  },
  "status_repo": {
    "owner": "YOUR_USER",
    "repo": "YOUR_STATUS_REPO"
  }
}
```

Also update the `src` input URL in `PULL/flake.nix` to match your source repo.

### 3. Start the daemon

```bash
./polld start
```

That's it. The daemon auto-detects your GitHub token from `gh` CLI, polls for pushes and PRs, builds all 8 variants, and updates commit status on GitHub.

To customize:

```bash
repo="owner/repo" poll_interval=60 build_timeout=7200 ./polld start
```

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
| `fuzz` | Fuzzing | Instrumented build for fuzz testing |

Each base variant has a corresponding `*-test-run` variant that builds and runs the test suite (e.g., `nix build .#asan-test-run`).

## Usage

### Building locally

```bash
cd PULL

# Build a specific variant
nix build .#release
nix build .#asan

# Build and run tests under a variant
nix build .#release-test-run
nix build .#asan-test-run

# Enter development shell (includes clang, gcc, cmake, etc.)
nix develop
```

### Managing the daemon

```bash
./polld start          # Start (auto-detects gh token)
./polld status         # Check if running
./polld log            # Last 50 lines of output
./polld log 100        # Last 100 lines
./polld follow         # Tail logs
./polld stop           # Stop
./polld restart        # Restart
```

### Running the E2E test

The `test-ci` script pushes a test commit, runs a full poll cycle, builds all 8 variants, and verifies the GitHub commit status:

```bash
./test-ci
```

Override defaults:

```bash
repo="owner/repo" timeout=600 ./test-ci
```

### Using the binary cache

Add to your project's `flake.nix`:

```nix
{
  nixConfig = {
    extra-substituters = [ "https://raw.githubusercontent.com/YOUR_USER/YOUR_CACHE_REPO/main" ];
    extra-trusted-public-keys = [ "buildbuild-cache:BASE64-PUBLIC-KEY" ];
  };
}
```

### Build status dashboard

View at `https://YOUR_USER.github.io/YOUR_STATUS_REPO/` or embed badges:

```markdown
![release](https://YOUR_USER.github.io/YOUR_STATUS_REPO/badges/release.svg)
![asan](https://YOUR_USER.github.io/YOUR_STATUS_REPO/badges/asan.svg)
```

## Components

| Component | Purpose | Details |
|-----------|---------|---------|
| [POLL](POLL/README.md) | Change detection | Monitors source repos via GitHub Events API |
| [PULL](PULL/README.md) | Build orchestration | Nix flake with variant definitions |
| [PUSH](PUSH/README.md) | Artifact storage | Git-based binary cache with uncompressed NARs |
| [POST](POST/README.md) | Status reporting | GitHub Pages dashboard with badges |

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
| `CACHE_REPO_TOKEN` | PULL/PUSH | Token with write access to cache repo |
| `STATUS_REPO_TOKEN` | PULL/POST | Token with write access to status repo |
| `NIX_SIGNING_KEY` | PULL | Secret key for NAR signing |
| `build_timeout` | POLL | Build timeout in seconds (default: 3600) |

## License

MIT
