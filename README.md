# MCPB Pack

[![Test](https://github.com/NimbleBrainInc/mcpb-pack/actions/workflows/test.yml/badge.svg)](https://github.com/NimbleBrainInc/mcpb-pack/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/v/release/NimbleBrainInc/mcpb-pack)](https://github.com/NimbleBrainInc/mcpb-pack/releases)

A GitHub Action to package MCP servers into distributable bundles and publish them to the [mpak registry](https://www.mpak.dev).

## What is this?

**MCP** (Model Context Protocol) is a standard for AI assistants to interact with external tools and services. An MCP server exposes tools (like "search files" or "query database") that AI assistants can call.

**MCPB** is a bundle format (`.mcpb` files) that packages an MCP server with all its dependencies into a single portable file. This makes MCP servers easy to distribute and install.

**mpak** is a public registry where you can publish and discover MCP bundles. Think of it like npm, but for MCP servers.

**This action** automates the entire workflow: build your MCP server into a bundle, attach it to your GitHub release, and register it with mpak so others can find and install it.

## Quick Start

### Prerequisites

Your repository needs:

1. **A `manifest.json`** describing your MCP server:

```json
{
  "name": "@your-org/your-server",
  "version": "1.0.0",
  "description": "What your server does",
  "server": {
    "type": "python",
    "entry_point": "your_package/server.py"
  }
}
```

2. **Your MCP server code** (Python or Node.js)

### Minimal Workflow

Add this to `.github/workflows/release.yml`:

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

When you publish a GitHub release, this will:
1. Vendor all dependencies into the bundle
2. Build a `.mcpb` file
3. Upload it to your release
4. Register it with mpak.dev

Your server is now discoverable via `mpak search` and installable via `mpak pull`.

## Usage

### Single Platform

For pure Python/Node servers without native dependencies:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: NimbleBrainInc/mcpb-pack@v2
```

### Multi-Platform

For servers with native dependencies (C extensions, Rust bindings, etc.), you need to build on each target platform:

```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - os: linux
            arch: amd64
            runner: ubuntu-latest
          - os: linux
            arch: arm64
            runner: ubuntu-24.04-arm
          - os: darwin
            arch: arm64
            runner: macos-latest
          - os: darwin
            arch: amd64
            runner: macos-13
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4
      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          output: "{name}-{version}-${{ matrix.os }}-${{ matrix.arch }}.mcpb"
```

Each job builds and registers its own platform-specific bundle. The registry merges them automatically.

### Build Only (No Publish)

For CI validation or private servers:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    upload: false
    announce: false
```

## Inputs

| Input            | Default                                    | Description                                                   |
| ---------------- | ------------------------------------------ | ------------------------------------------------------------- |
| `directory`      | `.`                                        | Directory containing manifest.json and server code            |
| `output`         | `{name}-{version}.mcpb`                    | Output filename (`{name}` and `{version}` are replaced)       |
| `python-version` | `3.13`                                     | Python version for vendoring (if Python server)               |
| `build`          | `true`                                     | Whether to build the bundle                                   |
| `upload`         | `true`                                     | Whether to upload to the GitHub release                       |
| `announce`       | `true`                                     | Whether to register with mpak.dev                             |
| `announce-url`   | `https://api.mpak.dev/v1/bundles/announce` | Registry endpoint (change for self-hosted registries)         |

## Outputs

| Output          | Description                        |
| --------------- | ---------------------------------- |
| `bundle-path`   | Path to the generated .mcpb file   |
| `bundle-size`   | Size of the bundle in bytes        |
| `bundle-sha256` | SHA256 hash for integrity checks   |
| `announced`     | Whether registration succeeded     |

## Permissions

```yaml
permissions:
  contents: write   # Required to upload to releases
  id-token: write   # Required for OIDC authentication with registry
```

## How It Works

### Building

The action:
1. Detects your server type from `manifest.json` (Python or Node.js)
2. Vendors all dependencies into the bundle (Python: `deps/`, Node: `node_modules/`)
3. Packages everything into a `.mcpb` file using the [mcpb CLI](https://github.com/anthropics/mcpb)

### Announcing

When you announce to mpak.dev:
1. The action requests an [OIDC token](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) from GitHub
2. This token cryptographically proves the bundle came from your repository
3. The registry verifies the token and registers your bundle
4. No API keys or secrets needed

Each platform build announces its own artifact. The registry tracks all artifacts for a version, so users can install the right bundle for their system.

### Opting Out of Announcing

To build without publishing to the public registry:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    announce: false
```

You might want this if:
- Your server is private or internal
- You're using a self-hosted registry
- You want to test before publishing
- You distribute through other channels

## Supported Runtimes

| Runtime | Detected via              | Dependency vendoring          |
| ------- | ------------------------- | ----------------------------- |
| Python  | `server.type: "python"`   | `uv pip install --target`     |
| Node.js | `server.type: "node"`     | `npm install --omit=dev`      |

## Example Repositories

- [mcp-echo](https://github.com/NimbleBrainInc/mcp-echo) - Simple Python MCP server with multi-platform builds

## Learn More

- [MCP Documentation](https://modelcontextprotocol.io) - Model Context Protocol specification
- [mpak Registry](https://www.mpak.dev) - Browse and search published MCP bundles
- [mcpb CLI](https://github.com/anthropics/mcpb) - Command-line tool for building bundles locally

## License

MIT
