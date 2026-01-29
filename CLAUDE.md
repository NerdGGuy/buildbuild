# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

buildbuild is a webhook-free CI system for Nix-based C++ projects. It polls GitHub's Events API instead of using webhooks, builds 10 variants (release, debug, asan, ubsan, tsan, msan, coverage, fuzz, test-build, test-run), and stores artifacts as uncompressed NAR files in a Git-based binary cache for delta compression.

## Architecture

Five components, each in its own Git submodule:

- **POLL** — Polls GitHub Events API for PushEvent/PullRequestEvent, sets commit status, triggers builds. State tracked in `~/.local/state/poll/`. Skips fork PRs for security.
- **PULL** — Nix flake that defines all 10 build variants. Shell scripts orchestrate: update source → build → export NAR → upload cache → update status. Config in `PULL/cache/config.json`.
- **PUSH** — Git repo serving as a binary cache. Stores uncompressed NARs (for Git delta compression), content-addressed build logs, and per-variant manifests. Implements the Nix substituter protocol.
- **POST** — GitHub Pages dashboard with auto-refresh, historical trends, and SVG badges. Data in `POST/data/`.
- **PROJ** — Example C++ calculator project used for testing. Makefile-based build.

Data flow: POLL detects push → sets "pending" status → PULL builds all variants → exports to NAR → PUSH stores artifacts → POST updates dashboard → POLL sets "success"/"failure" status.

## Common Commands

### Building locally
```bash
cd PULL
nix develop            # Enter dev shell
nix build .#release    # Build a specific variant (release|debug|asan|ubsan|tsan|msan|coverage|fuzz|test-build|test-run)
./result/bin/test_calculator  # Run tests from build output
```

### Full CI workflow
```bash
cd PULL
./update-variant.sh release       # Update source to latest commit
./export-cache.sh all ./export    # Build and export all variants
./upload-cache.sh ./export        # Upload to cache repo (needs CACHE_REPO_TOKEN)
./upload-status.sh ./export       # Update dashboard (needs STATUS_REPO_TOKEN)
```

### Poll daemon management
```bash
./polld start          # Start daemon (auto-detects token from gh CLI)
./polld status         # Check if running
./polld log            # View recent output
./polld follow         # Tail logs
./polld stop           # Stop daemon
./polld restart        # Restart
```

Override defaults: `repo="owner/repo" poll_interval=60 ./polld start`

### End-to-end test
```bash
./test-ci              # Pushes test commit to PROJ, verifies poll detects and builds it
```

## Key Environment Variables

| Variable | Used by | Purpose |
|----------|---------|---------|
| `repo` | POLL | Target repo as `owner/repo` |
| `github_token` | POLL | GitHub API token |
| `poll_interval` | POLL | Seconds between polls (default: 60) |
| `GITHUB_TOKEN` | PULL/PUSH/POST | GitHub API access |
| `CACHE_REPO_TOKEN` | PULL/PUSH | Write access to cache repo |
| `STATUS_REPO_TOKEN` | PULL/POST | Write access to status repo |
| `NIX_SIGNING_KEY` | PULL | Secret key for NAR signing |

## Technology Stack

- **Languages**: Bash (primary), Nix (build config), C++ (test project), JavaScript (dashboard)
- **Build**: Nix Flakes with variant-specific Nix derivations
- **Toolchains**: Clang/LLVM (default for most variants), GCC, musl libc — defined in `PULL/toolchains/`
- **Requirements**: Bash 4.0+, Nix 2.18+ with flakes, curl, jq, nix-prefetch-github

## Key Files

- `PULL/flake.nix` — Central Nix flake defining all build outputs
- `PULL/cache/config.json` — Repository configuration (source, cache, status repos)
- `PULL/variants/*/default.nix` — Per-variant build overrides (toolchain, flags)
- `PULL/variants/*/source.json` — Tracks current source commit per variant
- `PULL/nix/base.nix` — Base package definition; `builder.nix` — CMake/Meson helpers
- `POLL/poll` — Main polling loop; `POLL/lib/api.sh` — GitHub API wrapper
