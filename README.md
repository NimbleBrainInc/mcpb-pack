# MCPB Pack

[![Test](https://github.com/NimbleBrainInc/mcpb-pack/actions/workflows/test.yml/badge.svg)](https://github.com/NimbleBrainInc/mcpb-pack/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/v/release/NimbleBrainInc/mcpb-pack)](https://github.com/NimbleBrainInc/mcpb-pack/releases)

A GitHub Action that packages an MCP server into a `.mcpb` bundle and attaches it to your GitHub release.

> [!WARNING]
> **mcpb-pack is retired and will be archived.** Existing workflows keep running. For new ones, pack with the [mcpb CLI](https://github.com/modelcontextprotocol/mcpb) directly: `npx -y @anthropic-ai/mcpb@<version> pack <dir> <name>.mcpb`, which validates the manifest first. Then attach the file to your release with `gh release create` or `gh release upload`.

## What is this?

**MCP** (Model Context Protocol) is a standard for AI assistants to interact with external tools and services. An MCP server exposes tools (like "search files" or "query database") that AI assistants can call.

**MCPB** is a bundle format (`.mcpb` files) that packages an MCP server with all its dependencies into a single portable file. This makes MCP servers easy to distribute and install.

**This action** builds your MCP server into a bundle with its dependencies vendored and attaches it to your GitHub release, where Claude Desktop and other MCPB hosts can install it.

## Quick Start

### Prerequisites

Your repository needs:

1. **A `manifest.json`** describing your MCP server:

```json
{
  "manifest_version": "0.4",
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

2. **Your MCP server code** with stdio entrypoint:

```python
# At end of server.py
if __name__ == "__main__":
    mcp.run()  # Required for Claude Desktop and other MCPB hosts
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

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: NimbleBrainInc/mcpb-pack@v3
```

When you publish a GitHub release, this will:
1. Vendor all dependencies into the bundle
2. Build a `.mcpb` file
3. Upload it to your release

## Usage

### Single Platform

For pure Python/Node servers without native dependencies:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: NimbleBrainInc/mcpb-pack@v3
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
      - uses: actions/checkout@v6
      - uses: NimbleBrainInc/mcpb-pack@v3
        with:
          output: "{name}-{version}-${{ matrix.os }}-${{ matrix.arch }}.mcpb"
```

Each job builds its own platform bundle and attaches it to the same release.

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

### Companion Action Versions

Release workflows that call this action pin the same set of companion actions:

| Action / setting | Pin |
|---|---|
| `actions/checkout` | `@v6` |
| `actions/setup-node` (Node servers, UI builds) | `@v6` |
| `node-version` | `'22'` (the build runtime, independent of the action runtime) |
| `NimbleBrainInc/mcpb-pack` | `@v3` |

`checkout` and `setup-node` at `@v6` run on the Node 24 action runtime. When you edit a release workflow for any reason, move its pins to this set in the same change.

### Build Only (No Publish)

For CI validation or private servers:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v3
  with:
    upload: false
```

### Use Existing Bundle

If you already build your `.mcpb` bundle separately (e.g., committed to the repo or built in a prior step), you can skip the build and just upload it:

```yaml
- uses: NimbleBrainInc/mcpb-pack@v3
  with:
    directory: packages/mcp/mcpb
    bundle-path: context7.mcpb
    build: false
```

This is useful when:
- Your bundle is pre-built and committed to the repository
- You have a custom build process

The action computes the SHA256 hash and size of the provided bundle and uploads it to the release.

## Inputs

| Input            | Default                                    | Description                                                   |
| ---------------- | ------------------------------------------ | ------------------------------------------------------------- |
| `directory`      | `.`                                        | Directory containing manifest.json and server code            |
| `bundle-path`    |                                            | Path to existing .mcpb bundle (use with `build: false`)       |
| `output`         | `{name}-{version}.mcpb`                    | Output filename (`{name}` and `{version}` are replaced)       |
| `python-version` | `3.13`                                     | Python version for vendoring (if Python server)               |
| `build`          | `true`                                     | Whether to build the bundle                                   |
| `upload`         | `true`                                     | Whether to upload to the GitHub release                       |

`announce`, `announce-url`, `announce-required`, `platform-os` and `platform-arch` are accepted and ignored. They served the mpak registry, which is retired, and GitHub prints a deprecation warning when a workflow still sets them.

## Outputs

| Output          | Description                        |
| --------------- | ---------------------------------- |
| `bundle-path`   | Path to the generated .mcpb file   |
| `bundle-size`   | Size of the bundle in bytes        |
| `bundle-sha256` | SHA256 hash for integrity checks   |

## Permissions

```yaml
permissions:
  contents: write   # Required to upload to releases
```

## How It Works

### Building

The action:
1. Validates `manifest.json` (mcpb v0.4 required fields)
2. Detects your server type from `manifest.json` (Python or Node.js)
3. Vendors all dependencies into the bundle (Python: `deps/`, Node: `node_modules/`)
4. Packages everything into a `.mcpb` file using the [mcpb CLI](https://github.com/anthropics/mcpb)

## Listing Your Server

The action publishes to your GitHub release and nowhere else. To make the server discoverable, publish a `server.json` to the [official MCP Registry](https://github.com/modelcontextprotocol/registry) with a package of `registryType: "mcpb"` whose `identifier` is the release asset's download URL and whose `fileSha256` is this action's `bundle-sha256` output.

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
      - uses: actions/checkout@v6

      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"

      - name: Build binary
        run: |
          mkdir -p bin
          go build -o bin/server ./cmd/server

      - uses: NimbleBrainInc/mcpb-pack@v3
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
- [MCP Registry](https://github.com/modelcontextprotocol/registry) - The official registry of MCP servers
- [mcpb CLI](https://github.com/anthropics/mcpb) - Command-line tool for building bundles locally

## License

MIT
