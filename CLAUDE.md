# mcpb-pack

GitHub Action to package MCP servers into .mcpb bundles.

## Critical

- Architecture names use Node.js conventions: `x64` (not `amd64`), `arm64`
- Retry only on specific transient errors (match error message, not just HTTP code)
- GitHub asset uploads take 10-30s to propagate to API - retry delays must be substantial

## Supported Platforms

- `linux-x64`
- `linux-arm64`
- `darwin-arm64`

## Platform Normalization

```
OS:   Darwin → darwin, Linux → linux
Arch: x86_64 → x64, aarch64/arm64 → arm64
```

## Release Flow

```bash
git add . && git commit -m "..."
git push origin main
git tag vX.Y.Z && git push origin vX.Y.Z
# Workflow auto-creates release and updates major tag (v2)
```

## Retry Pattern

For transient errors (API propagation):
- 4 attempts max
- Linear backoff: 5s, 10s, 15s (30s total)
- Match specific error: `grep -q "not found in release"`
- Fail immediately on non-transient errors

## Testing

Matrix tests all supported platforms via `.github/workflows/test.yml`
