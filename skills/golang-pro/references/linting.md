---
name: golangci-lint
description: Run golangci-lint on Go projects with a comprehensive linter configuration. Enforces code quality, security, style consistency, and best practices using 60+ enabled linters.
---

# golangci-lint Skill

## When to use

Use this skill when:
- Linting Go code in this project
- Checking code quality before committing
- Running CI/CD lint checks
- Verifying code changes meet project standards
- Fixing linting issues reported in CI

## When NOT to use

- For non-Go projects (use language-specific linters)
- When running `go vet` alone is sufficient (golangci-lint is more comprehensive)
- If you need to skip specific linters (use `--disable` flags directly)

## Prerequisites

- Go 1.21+ installed
- golangci-lint v2.10.1+ (matches the golden config version)
- Project has a `.golangci.yml` at the repository root

## Workflow

### 1. Run linting

Execute golangci-lint from the project root:

```bash
# Run all linters on the entire project
golangci-lint run

# Run with verbose output
golangci-lint run -v

# Run on specific packages only
golangci-lint run ./internal/...

# Run with specific output format (useful for CI)
golangci-lint run --out-format=line-number
```

### 2. Fix auto-fixable issues

Some linters support automatic fixing:

```bash
# Fix issues where possible (formatters, imports, etc.)
golangci-lint run --fix
```

### 3. Run specific linters only

```bash
# Run a single linter
golangci-lint run --disable-all -E errcheck

# Run a subset of linters
golangci-lint run --disable-all -E govet,errcheck,staticcheck
```

### 4. Configuration reference

This project uses [`.golangci.yml`](.golangci.yml) at the repository root.

For a comprehensive example configuration, see [`assets/.golangci.yml`](assets/.golangci.yml) which includes:
- 60+ enabled linters across security, errors, complexity, and style
- `gci` formatter with local prefix grouping
- `gomod` relative path mode
- Parallel runner support with serialization
- Detailed linter-specific settings (cyclop, funlen, errcheck, govet, etc.)
- Exclusion rules for test files and common false positives

### 5. Common CI usage

```bash
# Full lint check (used in CI)
golangci-lint run ./...

# Quick check on changed files only
golangci-lint run --new-from-rev=HEAD~1
```

## Key Enabled Linters

| Category | Linters |
|----------|---------|
| **Security** | gosec |
| **Errors** | errcheck, errorlint, errname |
| **Complexity** | cyclop, gocognit, gocyclo, funlen |
| **Style** | gci, goimports, godot, gocritic |
| **Performance** | prealloc, ineffassign |
| **Correctness** | govet, staticcheck, gosimple |
| **Best Practices** | gochecknoglobals, gochecknoinits, exhaustive |

## Troubleshooting

### Out of memory errors
Reduce concurrency:
```bash
golangci-lint run --concurrency=2
```

### Slow runs
Run only changed files:
```bash
golangci-lint run --new-from-rev=origin/main
```

### False positives
**Important: Do not use `//nolint:...` unless absolutely necessary.

If you encounter false positives:
1. First, check if the code can be refactored to satisfy the linter
2. If the issue is in test files, check existing exclusion rules in `.golangci.yml`
3. Only as a last resort, use `//nolint` with specific linter and justification:

```go
//nolint:gosec // justification here - explain why this is necessary
```

The project requires explanations for all `nolint` directives (`nolintlint.require-explanation: true`).
