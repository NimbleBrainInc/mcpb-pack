# MCPB Pack

[![Test](https://github.com/NimbleBrainInc/mcpb-pack/actions/workflows/test.yml/badge.svg)](https://github.com/NimbleBrainInc/mcpb-pack/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/v/release/NimbleBrainInc/mcpb-pack)](https://github.com/NimbleBrainInc/mcpb-pack/releases)

A GitHub Action to package MCP servers into `.mcpb` bundles, upload them to releases, and announce them to the [mpak registry](https://www.mpak.dev).

## Features

- **Build**: Package MCP servers with vendored dependencies into portable `.mcpb` bundles
- **Upload**: Automatically upload bundles to GitHub releases
- **Announce**: Register bundles with the [mpak registry](https://www.mpak.dev) for public discovery

## Quick Start

Trigger on release for the simplest workflow:

```yaml
name: Release
on:
  release:
    types: [published]

permissions:
  contents: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: NimbleBrainInc/mcpb-pack@v2
```

That's it. When you publish a release, this will:
1. Build the `.mcpb` bundle
2. Upload it to the release
3. Announce it to the mpak registry

## Usage

### Single Architecture (Recommended)

```yaml
name: Release
on:
  release:
    types: [published]

permissions:
  contents: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: NimbleBrainInc/mcpb-pack@v2
```

### Multi-Architecture

For servers with native dependencies that need platform-specific builds:

```yaml
name: Release
on:
  release:
    types: [published]

permissions:
  contents: write
  id-token: write

jobs:
  build:
    strategy:
      matrix:
        include:
          - arch: amd64
            runner: ubuntu-latest
          - arch: arm64
            runner: ubuntu-24.04-arm
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4
      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          output: "{name}-{version}-linux-${{ matrix.arch }}.mcpb"
          announce: false  # Announce once after all builds

  announce:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          build: false
          announce: true
```

### Build Only (No Upload/Announce)

For testing or CI validation:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    upload: false
    announce: false
```

### Announce Only

For repos that build bundles separately:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    build: false
    announce: true
```

## Inputs

| Input            | Default                                    | Description                                                  |
| ---------------- | ------------------------------------------ | ------------------------------------------------------------ |
| `directory`      | `.`                                        | Directory containing the MCP server and manifest.json        |
| `output`         | `{name}-{version}.mcpb`                    | Output filename (supports `{name}`, `{version}` placeholders)|
| `python-version` | `3.13`                                     | Python version for vendoring deps (if Python server)         |
| `build`          | `true`                                     | Whether to build the .mcpb bundle                            |
| `upload`         | `true`                                     | Whether to upload bundle to release (requires release event) |
| `announce`       | `true`                                     | Whether to announce to the mpak registry                     |
| `announce-url`   | `https://api.mpak.dev/v1/bundles/announce` | URL of the announce endpoint                                 |

## Outputs

| Output          | Description                                   |
| --------------- | --------------------------------------------- |
| `bundle-path`   | Path to the generated .mcpb file              |
| `bundle-size`   | Size of the bundle in bytes                   |
| `bundle-sha256` | SHA256 hash of the bundle                     |
| `announced`     | Whether the bundle was announced successfully |

## Permissions

The action requires these permissions:

```yaml
permissions:
  contents: write   # Upload to release
  id-token: write   # OIDC for announce
```

## About Announcing

### What it does

By default, this action announces your bundle to the [mpak registry](https://www.mpak.dev), a public directory of MCP bundles. This makes your server discoverable via `mpak search` and installable via `mpak install`.

When you announce, the following information is sent to the registry:

- Your `manifest.json` contents (name, version, description, tools, etc.)
- The release tag (e.g., `v1.0.0`)
- Provenance data proving the bundle came from your GitHub repository

The registry then fetches the `.mcpb` bundle files directly from your GitHub release.

### Opting out

Announcing is optional. To build and upload without announcing:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    announce: false
```

You might want to disable announcing if:

- Your MCP server is private or internal
- You're using a self-hosted registry
- You want to test builds before publishing
- You prefer to distribute bundles through other channels

### Authentication

The action uses [GitHub OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) to authenticate with the mpak registry. This provides cryptographic proof that the bundle was published from your GitHub repository, without requiring any secrets or API keys.

## License

MIT
