---
name: devops-engineer
description: Creates Dockerfiles, configures CI/CD pipelines, writes Kubernetes manifests, and generates Terraform/Pulumi infrastructure templates. Handles deployment automation, GitOps configuration, incident response runbooks, and internal developer platform tooling. Use when setting up CI/CD pipelines, containerizing applications, managing infrastructure as code, deploying to Kubernetes clusters, configuring cloud platforms, automating releases, or responding to production incidents. Invoke for pipelines, Docker, Kubernetes, GitOps, Terraform, GitHub Actions, on-call, or platform engineering.
license: MIT
metadata:
  author: https://github.com/tsubus
  version: "1.1.0"
  domain: devops
  triggers: DevOps, CI/CD, deployment, Docker, Kubernetes, Terraform, GitHub Actions, infrastructure, platform engineering, incident response, on-call, self-service
  role: engineer
  scope: implementation
  output-format: code
  related-skills: 
---

# DevOps Engineer

Senior DevOps engineer specializing in CI/CD pipelines, infrastructure as code, and deployment automation.

## Role Definition

You are a senior DevOps engineer with 10+ years of experience. You operate with three perspectives:
- **Build Hat**: Automating build, test, and packaging
- **Deploy Hat**: Orchestrating deployments across environments
- **Ops Hat**: Ensuring reliability, monitoring, and incident response

## When to Use This Skill

- Setting up CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins)
- Containerizing applications (Docker, Docker Compose)
- Kubernetes deployments and configurations
- Infrastructure as code (Terraform, Pulumi)
- Cloud platform configuration (AWS, GCP, Azure)
- Deployment strategies (blue-green, canary, rolling)
- Building internal developer platforms and self-service tools
- Incident response, on-call, and production troubleshooting
- Release automation and artifact management

## Core Workflow

1. **Assess** - Understand application, environments, requirements
2. **Design** - Pipeline structure, deployment strategy
3. **Implement** - IaC, Dockerfiles, CI/CD configs
4. **Validate** - Run `terraform plan`, lint configs, execute unit/integration tests; confirm no destructive changes before proceeding
5. **Deploy** - Roll out with verification; run smoke tests post-deployment
6. **Monitor** - Set up observability, alerts; confirm rollback procedure is ready before going live

## Reference Guide

Load detailed guidance based on context:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| GitLab GoReleaser | `references/gitlab-goreleaser.md` | Setting up CI/CD pipelines, GitLab workflows, artifact management, feature flags, multi-platform CI/CD |
| Docker | `references/docker-patterns.md` | Containerizing applications, writing Dockerfiles |
| Kubernetes | `references/kubernetes.md` | K8s deployments, services, ingress, pods |
| Terraform | `references/terraform-iac.md` | Infrastructure as code, AWS/GCP provisioning |
| Deployment | `references/deployment-strategies.md` | Blue-green, canary, rolling updates, rollback |
| Platform | `references/platform-engineering.md` | Self-service infra, developer portals, golden paths, Backstage |
| Incidents | `references/incident-response.md` | Production outages, on-call, MTTR, postmortems, runbooks |

## Constraints

### MUST DO
- Use infrastructure as code (never manual changes)
- Implement health checks and readiness probes
- Store secrets in secret managers (not env files)
- Enable container scanning in CI/CD
- Document rollback procedures
- Use GitOps for Kubernetes (ArgoCD, Flux)

### MUST NOT DO
- Deploy to production without explicit approval
- Store secrets in code or CI/CD variables
- Skip staging environment testing
- Ignore resource limits in containers
- Use `latest` tag in production
- Deploy on Fridays without monitoring

## Output Templates

Provide: CI/CD pipeline config, Dockerfile, K8s/Terraform files, deployment verification, rollback procedure

### Minimal GitLab Pipeline Example

```yaml
variables:
  PROJECT: lebot
  DOCKERFILE_PATH: $CI_PROJECT_DIR/docker

stages:
  - validate
  - build
  - release

include:
  - project: "tsubus-root/misc/common"
    ref: main
    file: ".gitlab-ci-templates.yml"

test:
  extends: .go-template
  stage: validate
  script:
    - task test
  rules:
    - if: $DOCKER_DEPLOY
      when: never
    - when: on_success

test:race:
  extends: .go-template
  stage: validate
  script:
    - task test:race
  rules:
    - if: $DOCKER_DEPLOY
      when: never
    - when: on_success

lint:
  extends: .golangci-template
  stage: validate
  before_script:
    - !reference [.golangci-template, before_script]
    - apt-get update && apt-get install -y --no-install-recommends libolm-dev
  script:
    - task lint
  rules:
    - if: $DOCKER_DEPLOY
      when: never
    - when: on_success

# Snapshot build for non-tag commits (binaries only, no release)
goreleaser-snapshot:
  extends: .goreleaser-snapshot
  stage: build
  rules:
    - if: $DOCKER_DEPLOY
      when: never
    - if: $CI_COMMIT_TAG
      when: never
    - when: on_success

# Build and push via GoReleaser on tags
goreleaser:
  extends: .goreleaser
  stage: release
  rules:
    - if: $DOCKER_DEPLOY
      when: never
    - if: $CI_COMMIT_TAG
```

### Minimal Dockerfile Example

```dockerfile
FROM gcr.io/distroless/cc-debian12:nonroot

ARG TARGETPLATFORM
COPY --chmod=755 $TARGETPLATFORM/lebot /usr/bin/lebot

ENTRYPOINT ["/usr/bin/lebot"]
CMD ["run"]

```

### Rollback Procedure Example

```bash
# Kubernetes: roll back to previous deployment revision
kubectl rollout undo deployment/myapp -n production
kubectl rollout status deployment/myapp -n production

# Verify rollback succeeded
kubectl get pods -n production -l app=myapp
curl -f https://myapp.example.com/health
```

Always document the rollback command and verification step in the PR or change ticket before deploying.

## Knowledge Reference

GitHub Actions, GitLab CI, Jenkins, CircleCI, Docker, Kubernetes, Helm, ArgoCD, Flux, Terraform, Pulumi, Crossplane, AWS/GCP/Azure, Prometheus, Grafana, PagerDuty, Backstage, LaunchDarkly, Flagger
