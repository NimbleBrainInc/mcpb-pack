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
  "name": "@your-github-org/your-server",
  "version": "1.0.0",
  "description": "What your server does",
  "server": {
    "type": "python",
    "entry_point": "your_package.server",
    "mcp_config": {
      "command": "python",
      "args": ["-m", "your_package.server"]
    }
  }
}
```

> **Note:** The `@scope` must match your GitHub organization or username. The registry verifies this via OIDC.

2. **Your MCP server code** with stdio entrypoint:

```python
# At end of server.py
if __name__ == "__main__":
    mcp.run()  # Required for mpak run / Claude Desktop
```

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
            arch: x64
            runner: ubuntu-latest
          - os: linux
            arch: arm64
            runner: ubuntu-24.04-arm
          - os: darwin
            arch: arm64
            runner: macos-latest
          - os: darwin
            arch: x64
            runner: macos-15-intel
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4
      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          output: "{name}-{version}-${{ matrix.os }}-${{ matrix.arch }}.mcpb"
```

Each job builds and registers its own platform-specific bundle. The registry merges them automatically.

### Runner Reference

| Platform | Runner Label | Architecture | Notes |
|----------|--------------|--------------|-------|
| **Linux x64** | `ubuntu-latest` | x64 | Free |
| **Linux ARM** | `ubuntu-24.04-arm` | arm64 | Free |
| **macOS ARM** | `macos-latest` / `macos-15` | arm64 (M1) | Free, 3 vCPU, 7 GB |
| **macOS Intel** | `macos-15-intel` | x64 | Free, 4 vCPU, 14 GB |

**Paid larger runners** (Team/Enterprise plans):

| Platform | Runner Label | Architecture | Notes |
|----------|--------------|--------------|-------|
| **macOS Intel** | `macos-15-large` | x64 | 12 vCPU, 30 GB |
| **macOS ARM** | `macos-15-xlarge` | arm64 (M2) | 5 vCPU + 8 GPU, 14 GB |

> **Note:** `macos-13` is [retiring December 2025](https://github.blog/changelog/2025-09-19-github-actions-macos-13-runner-image-is-closing-down/). Use `macos-15-intel` for Intel macOS builds.

### Build Only (No Publish)

For CI validation or private servers:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v2
  with:
    upload: false
    announce: false
```

### Manual Re-announce

To re-announce an existing release (e.g., if the registry was down or you're announcing to a different registry), add `workflow_dispatch` to your workflow:

```yaml
on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      build:
        description: 'Build bundle'
        type: boolean
        default: true
      announce:
        description: 'Announce to registry'
        type: boolean
        default: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          build: ${{ inputs.build }}
          announce: ${{ inputs.announce }}
```

Then trigger manually from the Actions tab, checking out the release tag. The action uses `github.ref_name` as the release tag when not triggered by a release event.

> **Note:** The `upload` input only works with release events. For manual triggers, upload the bundle to the release manually or use `gh release upload`.

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
| Binary  | `server.type: "binary"`   | None (you build the binary)   |

### Binary Servers (Go, Rust, etc.)

For compiled languages, build your binary before running mcpb-pack:

```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - os: linux
            arch: x64
            runner: ubuntu-latest
          - os: linux
            arch: arm64
            runner: ubuntu-24.04-arm
          - os: darwin
            arch: arm64
            runner: macos-latest
          - os: darwin
            arch: x64
            runner: macos-15-intel
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"

      - name: Build binary
        run: |
          mkdir -p bin
          go build -o bin/server ./cmd/server

      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          output: "{name}-{version}-${{ matrix.os }}-${{ matrix.arch }}.mcpb"
```

The manifest.json for binary servers:

```json
{
  "name": "@your-org/your-server",
  "version": "1.0.0",
  "server": {
    "type": "binary",
    "entry_point": "bin/server",
    "mcp_config": {
      "command": "${__dirname}/bin/server",
      "args": []
    }
  }
}
```

### Node.js Servers

```json
{
  "name": "@your-org/your-server",
  "version": "1.0.0",
  "server": {
    "type": "node",
    "entry_point": "dist/index.js",
    "mcp_config": {
      "command": "node",
      "args": ["${__dirname}/dist/index.js"]
    }
  }
}
```

## Example Repositories

- [mcp-echo](https://github.com/NimbleBrainInc/mcp-echo) - Simple Python MCP server with multi-platform builds

## Learn More

- [MCP Documentation](https://modelcontextprotocol.io) - Model Context Protocol specification
- [mpak Registry](https://www.mpak.dev) - Browse and search published MCP bundles
- [mcpb CLI](https://github.com/anthropics/mcpb) - Command-line tool for building bundles locally

## License

MIT
