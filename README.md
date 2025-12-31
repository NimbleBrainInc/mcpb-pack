# MCPB Pack

[![Test](https://github.com/NimbleBrainInc/mcpb-pack/actions/workflows/test.yml/badge.svg)](https://github.com/NimbleBrainInc/mcpb-pack/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/v/release/NimbleBrainInc/mcpb-pack)](https://github.com/NimbleBrainInc/mcpb-pack/releases)

A GitHub Action to package MCP servers into `.mcpb` bundles and announce them to the [mpak registry](https://www.mpak.dev).

## Features

- **Build**: Package MCP servers with vendored dependencies into portable `.mcpb` bundles
- **Announce** (optional): Register bundles with the [mpak registry](https://www.mpak.dev) for public discovery
- **Flexible**: Build + announce, build only, or announce-only modes

## Usage

### Build and Announce (Default)

Build a new bundle and announce it to the registry:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    output: my-server-${{ github.ref_name }}.mcpb
```

### Announce Only (Existing Bundle)

For repos that already build `.mcpb` bundles, just announce them:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    build: false
    announce: true
```

### Build Only (No Announce)

Build without announcing (e.g., for testing):

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    output: my-server.mcpb
    announce: false
```

## Inputs

| Input            | Description                                           | Required | Default                                    |
| ---------------- | ----------------------------------------------------- | -------- | ------------------------------------------ |
| `directory`      | Directory containing the MCP server and manifest.json | No       | `.`                                        |
| `output`         | Output filename for the .mcpb bundle                  | No       | `extension.mcpb`                           |
| `python-version` | Python version for vendoring deps (if Python server)  | No       | `3.13`                                     |
| `build`          | Whether to build the .mcpb bundle                     | No       | `true`                                     |
| `announce`       | Whether to announce the bundle to the registry        | No       | `true`                                     |
| `announce-url`   | URL of the announce endpoint                          | No       | `https://api.mpak.dev/v1/bundles/announce` |
| `bundle-url`     | URL of existing bundle (for announce-only mode)       | No       | Auto-detects from release                  |

> **Note:** Announcing uses GitHub OIDC for authentication. Your workflow must have `permissions: id-token: write` for the announce step to work.

## Outputs

| Output          | Description                                   |
| --------------- | --------------------------------------------- |
| `bundle-path`   | Path to the generated .mcpb file              |
| `bundle-size`   | Size of the bundle in bytes                   |
| `bundle-sha256` | SHA256 hash of the bundle                     |
| `announced`     | Whether the bundle was announced successfully |

## Examples

### Basic Release Workflow

```yaml
name: Release
on:
  push:
    tags: ["v*"]

permissions:
  contents: write
  id-token: write # Required for OIDC announce

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: NimbleBrainInc/mcpb-pack@v2
        id: pack
        with:
          output: my-server-${{ github.ref_name }}.mcpb

      - uses: softprops/action-gh-release@v2
        with:
          files: ${{ steps.pack.outputs.bundle-path }}
```

### Announce Existing Bundle

For repos that already build their own `.mcpb`:

```yaml
name: Announce MCPB
on:
  release:
    types: [published]

permissions:
  contents: read
  id-token: write # Required for OIDC announce

jobs:
  announce:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          build: false
          announce: true
```

The action will auto-detect the `.mcpb` asset from the release and announce it.

### Multi-Architecture Build

```yaml
name: Release
on:
  push:
    tags: ["v*"]

jobs:
  build:
    runs-on: ${{ matrix.runner }}
    strategy:
      matrix:
        include:
          - runner: ubuntu-latest
            arch: amd64
          - runner: ubuntu-24.04-arm
            arch: arm64
    steps:
      - uses: actions/checkout@v4

      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          output: my-server-${{ github.ref_name }}-linux-${{ matrix.arch }}.mcpb
          announce: false # Announce once after all builds complete

      - uses: actions/upload-artifact@v4
        with:
          name: bundle-${{ matrix.arch }}
          path: "*.mcpb"

  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: dist
          merge-multiple: true

      - uses: softprops/action-gh-release@v2
        with:
          files: dist/*.mcpb
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

**Announcing is completely optional.** To build bundles without announcing:

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
