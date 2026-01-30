# buildbuild

A webhook-free CI system for Nix-based C++ projects. Polls GitHub's Events API for changes, builds 8 sanitizer/analysis variants, and stores artifacts as uncompressed NAR files in a Git-based binary cache -- no S3, no Git LFS, no dedicated cache infrastructure required.

## Why buildbuild?

Traditional CI systems rely on webhooks, cloud storage, and complex infrastructure. buildbuild takes a different approach:

- **No webhooks** -- Polls GitHub's Events API, so no public endpoint or tunnel is needed. Works behind NATs and firewalls.
- **Git-based binary cache** -- Build artifacts are stored as uncompressed NARs in a plain Git repository. Git's native delta compression efficiently deduplicates similar builds across variants and versions.
- **No cloud storage** -- No S3 buckets, no Git LFS, no Cachix subscription. Just Git repositories you already know how to manage.
- **8 build variants out of the box** -- Every push is built with release, debug, ASan, UBSan, TSan, MSan, coverage, and fuzz configurations. Each variant also has a `*-test-run` counterpart that executes the test suite.
- **Pure Nix** -- Reproducible builds via Nix Flakes with flake-parts. Source is a flake input, overridden per-build via `--override-input`.

## Architecture

```
┌──────────────────┐         ┌──────────────────┐
│   SOURCE REPO    │────────>│      POLL        │
│   (C++ code)     │  push   │  change detect   │
└──────────────────┘         └────────┬─────────┘
                                      │
                                      v
┌──────────────────┐         ┌──────────────────┐
│      PUSH        │<────────│      PULL        │
│  binary cache    │  export │  build variants  │
└────────┬─────────┘         └──────────────────┘
         │
         v
┌──────────────────┐
│      POST        │
│    dashboard     │
└──────────────────┘
```

**Data flow:** POLL detects a push or pull request via the GitHub Events API and sets the commit status to "pending". PULL builds all 8 variants, exports each to NAR format, and uploads them to PUSH (the binary cache). POST updates the GitHub Pages dashboard with results and badges. POLL then sets the final commit status to "success" or "failure".

## Components

| Component | Role | Description |
|-----------|------|-------------|
| [POLL](POLL/README.md) | Change detection | Polls GitHub Events API for PushEvent/PullRequestEvent, sets commit status, triggers builds. State tracked in `~/.local/state/poll/`. Skips fork PRs for security. |
| [PULL](PULL/README.md) | Build orchestration | Nix flake (flake-parts) defining 8 base build variants plus `*-test-run` overrides. Source is a flake input (`src`), overridden at build time via `--override-input src github:owner/repo/SHA`. |
| [PUSH](PUSH/README.md) | Artifact storage | Git repository serving as a Nix binary cache. Stores uncompressed NARs (for Git delta compression), content-addressed build logs, and per-variant manifests. Implements the Nix substituter protocol. |
| [POST](POST/README.md) | Status dashboard | GitHub Pages site with auto-refresh, historical trends, and SVG badges. Data stored in `POST/data/`. |
| [PROJ](PROJ/README.md) | Example project | C++ calculator project used for testing and as a reference integration. Makefile-based build. |

## Prerequisites

- **Nix 2.18+** with flakes enabled ([install](https://nixos.org/download/))
- **Bash 4.0+**
- **GitHub CLI** (`gh`) authenticated, or a `github_token` environment variable
- **curl** and **jq**
- **nix-prefetch-github**

To enable flakes, add to `~/.config/nix/nix.conf`:

```
experimental-features = nix-command flakes
```

## Setup

### 1. Clone with submodules

```bash
git clone --recurse-submodules https://github.com/NerdGGuy/buildbuild.git
cd buildbuild
```

### 2. Create the required GitHub repositories

buildbuild uses three additional repositories beyond your source code:

| Repository | Purpose | GitHub Pages |
|------------|---------|--------------|
| **Cache repo** (e.g. `PUSH`) | Stores NAR artifacts and build logs | Not required |
| **Status repo** (e.g. `POST`) | Hosts the build dashboard and badges | Enable GitHub Pages on `main` branch |
| **Source repo** (e.g. `PROJ`) | Your C++ project | Not required |

Create these repos on GitHub (they can be public or private).

### 3. Configure `PULL/cache/config.json`

Edit `PULL/cache/config.json` to point at your repositories:

```json
{
  "source_repo": {
    "owner": "YOUR_USER",
    "repo": "YOUR_PROJECT",
    "default_ref": "main"
  },
  "cache_repo": {
    "owner": "YOUR_USER",
    "repo": "YOUR_CACHE_REPO",
    "public_key": "YOUR_PUBLIC_KEY_HERE"
  },
  "status_repo": {
    "owner": "YOUR_USER",
    "repo": "YOUR_STATUS_REPO"
  },
  "signing": {
    "key_name": "buildbuild-cache",
    "secret_env": "NIX_SIGNING_KEY"
  },
  "variants": {
    "release": { "mode": "branch", "ref": "main" },
    "debug":   { "mode": "branch", "ref": "main" },
    "asan":    { "mode": "branch", "ref": "main" },
    "ubsan":   { "mode": "branch", "ref": "main" },
    "tsan":    { "mode": "branch", "ref": "main" },
    "msan":    { "mode": "branch", "ref": "main" },
    "coverage":{ "mode": "branch", "ref": "main" },
    "fuzz":    { "mode": "branch", "ref": "main" }
  }
}
```

### 4. Configure the `src` flake input

Edit the `src` input in `PULL/flake.nix` to point at your source repository:

```nix
src = {
  url = "git+ssh://git@github.com/YOUR_USER/YOUR_PROJECT";
  flake = false;
};
```

Use `git+ssh://` for private repos or `github:YOUR_USER/YOUR_PROJECT` for public ones.

### 5. Generate Nix signing keys

The signing key ensures NAR integrity when consumers pull from your binary cache.

```bash
# Generate a secret key
nix key generate-secret --key-name buildbuild-cache > secret-key.pem

# Derive the public key
nix key convert-secret-to-public < secret-key.pem
```

Save the secret key securely. Add the public key to the `cache_repo.public_key` field in `config.json` and distribute it to cache consumers.

### 6. Set environment variables

```bash
export NIX_SIGNING_KEY="$(cat secret-key.pem)"
export CACHE_REPO_TOKEN="ghp_..."   # Token with write access to the cache repo
export STATUS_REPO_TOKEN="ghp_..."  # Token with write access to the status repo
```

If you do not set `github_token`, the daemon will auto-detect credentials from `gh auth status`.

### 7. Start the daemon

```bash
./polld start
```

The daemon polls for pushes and pull requests, builds all 8 variants (plus test-run), exports NARs, uploads to the cache, updates the dashboard, and sets commit status on GitHub.

### 8. Verify with the end-to-end test

```bash
./test-ci
```

This pushes a test commit to the source repo, triggers a full poll cycle, builds all variants, and verifies the final GitHub commit status.

## Build Variants

| Variant | Toolchain | Purpose | Detects |
|---------|-----------|---------|---------|
| `release` | Clang | Optimized production build | -- |
| `debug` | Clang | Debug symbols, no optimization | -- |
| `asan` | Clang | AddressSanitizer | Buffer overflows, use-after-free, double-free |
| `ubsan` | Clang | UndefinedBehaviorSanitizer | Signed overflow, null pointer dereference, type mismatches |
| `tsan` | Clang | ThreadSanitizer | Data races, deadlocks |
| `msan` | Clang | MemorySanitizer | Reads of uninitialized memory |
| `coverage` | Clang | Code coverage instrumentation | Untested code paths |
| `fuzz` | Clang | Fuzzing instrumentation | Crash-inducing inputs |

Each base variant has a corresponding `*-test-run` variant that builds the project **and** executes its test suite. For example, `nix build .#asan-test-run` builds under AddressSanitizer and runs all tests -- if any test triggers a sanitizer error, the build fails.

The `*-test-run` variants are generated via `makeOverridable` by setting `doCheck = true` on the base derivation.

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

# Run tests from a build output
./result/bin/test_calculator

# Enter the development shell (includes clang, gcc, cmake, ninja, etc.)
nix develop
```

### Managing the daemon

```bash
./polld start          # Start (auto-detects gh token)
./polld status         # Check if running
./polld log            # Last 50 lines of output
./polld log 100        # Last 100 lines
./polld follow         # Tail logs in real time
./polld stop           # Stop the daemon
./polld restart        # Stop and restart
```

Override defaults at startup:

```bash
repo="owner/repo" poll_interval=120 build_timeout=7200 ./polld start
```

### Running the end-to-end test

```bash
./test-ci
```

Override the target repo or timeout:

```bash
repo="owner/repo" timeout=600 ./test-ci
```

### Using the binary cache

Add your cache repo as a Nix substituter in your project's `flake.nix`:

```nix
{
  nixConfig = {
    extra-substituters = [
      "https://raw.githubusercontent.com/YOUR_USER/YOUR_CACHE_REPO/main"
    ];
    extra-trusted-public-keys = [
      "buildbuild-cache:YOUR_BASE64_PUBLIC_KEY"
    ];
  };
}
```

Nix will then transparently fetch cached build outputs instead of rebuilding them.

### Dashboard and badges

The POST component serves a GitHub Pages dashboard at:

```
https://YOUR_USER.github.io/YOUR_STATUS_REPO/
```

Embed per-variant status badges in your project README:

```markdown
![release](https://YOUR_USER.github.io/YOUR_STATUS_REPO/badges/release.svg)
![asan](https://YOUR_USER.github.io/YOUR_STATUS_REPO/badges/asan.svg)
```

## Running as a systemd Service

To run buildbuild as a persistent service, create a systemd unit file:

```ini
# /etc/systemd/system/buildbuild.service
[Unit]
Description=buildbuild CI daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ci
WorkingDirectory=/home/ci/buildbuild
ExecStart=/home/ci/buildbuild/polld start --foreground
ExecStop=/home/ci/buildbuild/polld stop
Restart=on-failure
RestartSec=30

Environment=NIX_SIGNING_KEY=<your-secret-key>
Environment=CACHE_REPO_TOKEN=<your-cache-token>
Environment=STATUS_REPO_TOKEN=<your-status-token>
Environment=repo=owner/repo
Environment=poll_interval=60
Environment=build_timeout=3600

[Install]
WantedBy=multi-user.target
```

Then enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now buildbuild.service
sudo journalctl -u buildbuild.service -f   # Follow logs
```

For sensitive tokens, consider using `EnvironmentFile=` pointing to a root-owned file with restricted permissions instead of inline `Environment=` directives.

## Environment Variables

| Variable | Component | Default | Description |
|----------|-----------|---------|-------------|
| `repo` | POLL | -- | Target repository in `owner/repo` format |
| `github_token` | POLL | auto-detected from `gh` | GitHub API token for polling and status updates |
| `poll_interval` | POLL | `60` | Seconds between poll cycles |
| `build_timeout` | POLL | `3600` | Maximum build time in seconds before timeout |
| `GITHUB_TOKEN` | PULL/PUSH/POST | -- | GitHub API access for build and upload scripts |
| `CACHE_REPO_TOKEN` | PULL/PUSH | -- | Token with write access to the cache repository |
| `STATUS_REPO_TOKEN` | PULL/POST | -- | Token with write access to the status repository |
| `NIX_SIGNING_KEY` | PULL | -- | Nix secret signing key for NAR artifacts |

## Troubleshooting

### Daemon does not start

- Verify `gh auth status` succeeds, or set `github_token` explicitly.
- Check that Nix flakes are enabled: `nix flake show` should work without errors.
- Review logs: `./polld log` or `./polld follow`.

### Builds fail for msan or fuzz variants

- MSan requires a fully instrumented C library. The PULL flake uses musl libc for the msan variant. Ensure the musl toolchain is available in `PULL/toolchains/musl.nix`.
- Fuzz builds require Clang's libFuzzer support. Verify your Clang version includes fuzzer instrumentation.

### Cache upload fails

- Confirm `CACHE_REPO_TOKEN` has write (push) access to the cache repository.
- Check that the cache repo exists and is initialized with at least one commit.
- Run `./polld log` to see the specific upload error.

### NAR signature verification fails on consumers

- Ensure the `public_key` in `config.json` matches the key derived from your secret key.
- Re-derive with: `nix key convert-secret-to-public < secret-key.pem`
- Verify the consumer's `extra-trusted-public-keys` matches exactly.

### GitHub commit status not updating

- Confirm `STATUS_REPO_TOKEN` (or `github_token`) has the `repo:status` scope.
- Check that `repo` is set to the correct `owner/repo` value.

### Poll detects no events

- The Events API has a delay of a few seconds. Wait at least one poll interval.
- Fork pull requests are intentionally skipped for security.
- Verify the source repo is correct in `config.json` and the `repo` environment variable.

### `nix build` works but `test-ci` fails

- `test-ci` pushes a test commit to the source repo. Ensure you have write access.
- Check the `timeout` value -- large projects may need more time.

## Repository Structure

```
buildbuild/
├── polld                    # Daemon management (start/stop/status/log)
├── test-ci                  # End-to-end CI pipeline test
├── CLAUDE.md                # AI assistant instructions
├── README.md                # This file
├── POLL/                    # Change detection submodule
│   ├── poll                 # Main polling script
│   └── lib/api.sh           # GitHub API wrapper
├── PULL/                    # Build orchestration submodule
│   ├── flake.nix            # Nix flake definition (flake-parts)
│   ├── cache/config.json    # Repository and variant configuration
│   ├── variants/            # Per-variant build overrides
│   │   ├── release/default.nix
│   │   ├── debug/default.nix
│   │   ├── asan/default.nix
│   │   ├── ubsan/default.nix
│   │   ├── tsan/default.nix
│   │   ├── msan/default.nix
│   │   ├── coverage/default.nix
│   │   └── fuzz/default.nix
│   ├── nix/                 # Nix helper modules
│   │   ├── base.nix         # Base package definition
│   │   ├── lib.nix          # Shared library functions
│   │   └── checks.nix       # CI checks
│   ├── toolchains/          # Compiler configurations
│   │   ├── clang.nix
│   │   ├── gcc.nix
│   │   └── musl.nix
│   └── *.sh                 # Build/export/upload scripts
├── PUSH/                    # Binary cache submodule
│   └── upload.sh            # Cache upload script
├── POST/                    # Status dashboard submodule
│   ├── index.html           # Dashboard page
│   ├── style.css            # Styling
│   ├── app.js               # Client-side logic
│   ├── update.sh            # Status update script
│   ├── data/                # Status JSON data
│   └── badges/              # SVG status badges
└── PROJ/                    # Example C++ project submodule
    ├── src/                 # Source code
    ├── tests/               # Test suite
    └── Makefile             # Build system
```

## License

MIT
