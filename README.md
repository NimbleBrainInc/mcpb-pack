# MCPB Pack Action

A GitHub Action to package MCP servers into `.mcpb` bundles using the official [mcpb CLI](https://github.com/modelcontextprotocol/mcpb).

## Usage

```yaml
- uses: NimbleBrainInc/mcpb-pack-action@v1
  with:
    output: my-server-v1.0.0.mcpb
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `directory` | Directory containing the MCP server and manifest.json | No | `.` |
| `output` | Output filename for the .mcpb bundle | No | `extension.mcpb` |

## Outputs

| Output | Description |
|--------|-------------|
| `bundle-path` | Path to the generated .mcpb file |

## Examples

### Basic Release Workflow

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: NimbleBrainInc/mcpb-pack-action@v1
        id: pack
        with:
          output: my-server-${{ github.ref_name }}.mcpb

      - uses: softprops/action-gh-release@v2
        with:
          files: ${{ steps.pack.outputs.bundle-path }}
```

### Multi-Architecture Build

```yaml
name: Release
on:
  push:
    tags: ['v*']

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

      - uses: NimbleBrainInc/mcpb-pack-action@v1
        with:
          output: my-server-${{ github.ref_name }}-linux-${{ matrix.arch }}.mcpb

      - uses: actions/upload-artifact@v4
        with:
          name: bundle-${{ matrix.arch }}
          path: '*.mcpb'

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

## License

MIT
