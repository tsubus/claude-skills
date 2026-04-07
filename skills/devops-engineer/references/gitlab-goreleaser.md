---
name: gitlab-goreleaser
description: Configure GitLab CI/CD pipelines with GoReleaser for Go projects. Handles multi-arch builds, container images, SBOM generation, and automated releases.
---

# GitLab + GoReleaser Skill

## When to use

Use this skill when setting up or modifying CI/CD for a Go project that needs:
- Automated builds for multiple platforms (Linux, macOS, Windows)
- Multi-architecture support (amd64, arm64)
- Docker container images pushed to registry
- Automated GitHub/GitLab releases with changelogs
- SBOM (Software Bill of Materials) generation
- Snapshot builds for non-tag commits

## When NOT to use

- For non-Go projects (use language-specific tooling instead)
- When you only need simple Docker builds without multi-arch support
- If using GitHub Actions exclusively (use `goreleaser/goreleaser-action` instead)
- When you need custom release logic beyond GoReleaser's capabilities

## Prerequisites

- Go 1.21+ project with `go.mod`
- GitLab repository with CI/CD runners
- Container registry access (GitLab Registry or external)
- (Optional) GitLab project token for releases

## Workflow

### 1. Create GoReleaser configuration

Create `.goreleaser.yaml` at project root:

```yaml
version: 2

project_name: <your-project>

before:
  hooks:
    - go mod tidy

env:
  - CGO_ENABLED=0

builds:
  - id: <your-project>
    main: .
    binary: <your-project>
    goos:
      - linux
      - darwin
    goarch:
      - amd64
      - arm64
    flags:
      - -trimpath
    ldflags:
      - -s -w
      - -X main.BuildDate={{.Date}}
      - -X main.BuildRevision={{.Commit}}
      - -X main.BuildTag={{.Version}}

archives:
  - id: <your-project>
    name_template: >-
      {{ .ProjectName }}_
      {{- .Version }}_
      {{- title .Os }}_
      {{- if eq .Arch "amd64" }}x86_64
      {{- else }}{{ .Arch }}{{ end }}
    files:
      - README.md

checksum:
  name_template: "checksums.txt"
  algorithm: sha256

sboms:
  - id: <your-project>
    artifacts: binary
    documents:
      - "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}_sbom.spdx.json"

changelog:
  use: git
  format: "{{ .SHA }}: {{ .Message }} ({{ .AuthorName }})"
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^ci:"
      - "Merge pull request"
      - "Merge branch"
  sort: asc

release:
  gitlab:
    owner: <gitlab-group>
    name: <path/to/project>
  draft: false
  prerelease: auto

snapshot:
  version_template: "{{ incpatch .Version }}-next"
```

Key decisions:
- Set `CGO_ENABLED=0` for static binaries (disable if you need CGO)
- Adjust `goos`/`goarch` for your target platforms
- Add `windows` to `goos` if Windows support needed
- Update `ldflags` to match your version variables

### 2. Create Dockerfile for GoReleaser

Create `docker/Dockerfile.goreleaser` (used by GoReleaser, not for direct builds):

```dockerfile
FROM scratch

# Copy binary from goreleaser build
COPY <your-project> /<your-project>

# If you need timezone data or CA certs, use:
# COPY --from=alpine:latest /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
# COPY --from=alpine:latest /usr/share/zoneinfo /usr/share/zoneinfo

EXPOSE 8080
ENTRYPOINT ["/<your-project>"]
```

### 3. Create GitLab CI configuration

Create `.gitlab-ci.yml` at project root:

```yaml
variables:
  PROJECT: <your-project>
  DOCKERFILE_PATH: $CI_PROJECT_DIR/docker

stages:
  - validate
  - build
  - release

test:
  stage: validate
  image: golang:1.23
  script:
    - go test ./...
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - when: on_success

lint:
  stage: validate
  image: golangci/golangci-lint:latest
  script:
    - golangci-lint run
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - when: on_success

# Snapshot build for non-tag commits (binaries only, no release)
goreleaser-snapshot:
  stage: build
  image: goreleaser/goreleaser:latest
  script:
    - goreleaser release --snapshot --clean
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - when: on_success

# Full release on tags
goreleaser:
  stage: release
  image: goreleaser/goreleaser:latest
  script:
    - goreleaser release --clean
  rules:
    - if: $CI_COMMIT_TAG
```

### 4. Configure ldflags version injection

Ensure your `main.go` or version package defines variables matching the ldflags:

```go
package main

var (
    BuildDate     string
    BuildRevision string
    BuildTag      string
)

func main() {
    // These will be populated at build time by GoReleaser
    fmt.Printf("Version: %s (built %s, commit %s)\n", BuildTag, BuildDate, BuildRevision)
}
```

### 5. Add Docker multi-arch support (optional)

Add to `.goreleaser.yaml`:

```yaml
dockers_v2:
  - ids:
      - <your-project>
    images:
      - "{{ .Env.CI_REGISTRY_IMAGE }}"
    dockerfile: docker/Dockerfile.goreleaser
    tags:
      - "{{ .Version }}"
      - "latest"
    platforms:
      - linux/amd64
      - linux/arm64
    labels:
      org.opencontainers.image.created: "{{.Date}}"
      org.opencontainers.image.title: "{{.ProjectName}}"
      org.opencontainers.image.revision: "{{.FullCommit}}"
      org.opencontainers.image.version: "{{.Version}}"
      org.opencontainers.image.source: "{{.GitURL}}"
    flags:
      - --pull
```

### 6. Test locally

Before pushing:

```bash
# Validate GoReleaser config
goreleaser check

# Test snapshot build (no release, no tag needed)
goreleaser release --snapshot --clean --skip=publish

# Check generated artifacts
ls -la dist/
```

## Common Patterns

### Using shared GitLab CI templates

If your organization has shared templates:

```yaml
include:
  - project: "group/ci-templates"
    ref: main
    file: ".gitlab-ci-go.yml"

goreleaser:
  extends: .goreleaser
  stage: release
```

### Private Go modules

If using private Go modules, add to CI:

```yaml
before_script:
  - git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com/".insteadOf "https://gitlab.com/"
```

### Custom build tags

Add build tags in `.goreleaser.yaml`:

```yaml
builds:
  - tags:
      - netgo
      - osusergo
      - timetzdata
```

### Exclude platforms

To skip specific OS/arch combinations:

```yaml
builds:
  - ignore:
      - goos: darwin
        goarch: arm
      - goos: windows
        goarch: arm64
```

## Release Process

1. **Development**: Push commits to any branch → snapshot build runs, artifacts available for 1 week
2. **Release**: Create and push a Git tag → full release runs:
   - Binaries built for all platforms
   - Docker images pushed to registry
   - GitLab release created with changelog
   - SBOMs generated and attached

```bash
# Create release
git tag v1.2.3
git push origin v1.2.3
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `unknown flag: --snapshot` | Ensure `version: 2` in `.goreleaser.yaml` |
| Docker push fails | Verify `CI_REGISTRY_IMAGE` is set and registry credentials configured |
| SBOM generation fails | Ensure `syft` or `bom` is available; GoReleaser uses built-in now |
| Version shows "dev" | Check ldflags variable names match your code |
| Missing Windows binaries | Add `windows` to `goos` list |
| CGO errors | Set `CGO_ENABLED=0` or use alpine-based builder |

## Verification Checklist

Before considering setup complete:

- [ ] `goreleaser check` passes locally
- [ ] Snapshot build produces expected artifacts
- [ ] Binaries run on target platforms
- [ ] Version flags show correct information
- [ ] Docker image builds and runs
- [ ] CI pipeline passes on branch push
- [ ] Release works on tag push
- [ ] Changelog format is acceptable
