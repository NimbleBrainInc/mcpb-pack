# Releasing

## Version Strategy

- `v1.0.0`, `v1.1.0`, etc. - Semantic versions for changelogs
- `v1` - Floating tag that always points to latest `v1.x.x`
- `v2` - Breaking changes warrant a new major version

## Release Process

1. **Make changes** and push to `main`

2. **Tag the release**:
   ```bash
   git tag v1.x.x
   git push origin v1.x.x
   ```

3. **Automated steps** (via `.github/workflows/release.yml`):
   - Creates GitHub Release with auto-generated notes
   - Updates floating `v1` tag to point to new version

4. **Publish to Marketplace** (manual, one-time per major version):
   - Edit the release on GitHub
   - Check "Publish this Action to the GitHub Marketplace"
   - Select categories: "Continuous Integration", "Utilities"
   - Save

## Testing Before Release

Test changes by referencing the branch:
```yaml
uses: NimbleBrainInc/mcpb-pack@my-feature-branch
```

Or reference a specific commit:
```yaml
uses: NimbleBrainInc/mcpb-pack@abc123
```
