# Contributing to MCPB Pack

Thanks for your interest in contributing! This document outlines how to get involved.

## Reporting Issues

- **Bugs**: Open an issue with steps to reproduce, expected behavior, and actual behavior
- **Features**: Open an issue describing the use case and proposed solution

## Development Setup

1. Fork and clone the repository
2. Make changes to `action.yml` or supporting files
3. Test your changes (see below)

## Testing Changes

### Local Testing

Reference your branch in a test workflow:

```yaml
- uses: your-username/mcpb-pack@your-branch
  with:
    announce: false  # Don't announce during testing
```

### CI Testing

The repo includes a test workflow that runs on every PR. It:
- Creates a minimal Python MCP server
- Builds a `.mcpb` bundle
- Verifies the bundle contains vendored dependencies

## Submitting Pull Requests

1. Create a feature branch from `main`
2. Make your changes
3. Ensure tests pass
4. Submit a PR with a clear description of the change

### PR Guidelines

- Keep changes focused and atomic
- Update README.md if adding/changing inputs or behavior
- Add tests for new functionality when possible

## Code Style

- Use clear, descriptive variable names in shell scripts
- Add comments for non-obvious logic
- Quote variables to prevent word splitting: `"$VAR"` not `$VAR`

## Questions?

Open an issue or discussion if you're unsure about anything.
