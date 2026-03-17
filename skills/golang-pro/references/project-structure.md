# Project Structure and Module Management

## Standard Project Layout

```
myproject/
├── cmd/                  # Commands, config structs, root wiring (Cobra)
├── internal/              # Private application code
│   ├── app/              # Business logic
│   ├── repository/       # Data access layer
│   └── web/              # API handlers
├── pkg/                   # Public library code
│   └── models/           # Shared models
├── api/                   # API definitions
│   ├── openapi.yaml      # OpenAPI spec
│   └── proto/            # Protocol buffers
├── web/                   # Web assets
│   ├── static/
│   └── templates/
├── scripts/               # Build and install scripts
├── configs/              # Configuration files
├── deployments/          # Docker, K8s configs
├── docs/                 # Documentation
├── go.mod               # Module definition
├── go.sum               # Dependency checksums
├── main.go               # Entry point (build version, delegates to cmd)
├── Taskfile.yaml         # Build automation
└── README.md
```

## go.mod Basics

```go
// Initialize module
// go mod init gitlab.com/user/project

module gitlab.com/user/myproject

go 1.26

require (
	github.com/containrrr/shoutrrr v0.4.5-0.20251218173108-262ac52fc3b2
	github.com/danielgtaylor/huma/v2 v2.37.2
	github.com/go-chi/chi/v5 v5.2.5
	github.com/go-chi/jwtauth/v5 v5.4.0
	github.com/lestrrat-go/jwx/v3 v3.0.13
	github.com/rs/zerolog v1.34.0
	github.com/sethvargo/go-envconfig v1.3.0
	github.com/spf13/cobra v1.10.2
	github.com/stretchr/testify v1.11.1
	github.com/vmihailenco/msgpack/v5 v5.4.1
	gitlab.com/tsubus-go/tsubuslog v0.2.1
	go.etcd.io/bbolt v1.4.3
	golang.org/x/crypto v0.49.0
)

require (
	github.com/davecgh/go-spew v1.1.1 // indirect
	github.com/decred/dcrd/dcrec/secp256k1/v4 v4.4.0 // indirect
	github.com/fatih/color v1.15.0 // indirect
	github.com/goccy/go-json v0.10.5 // indirect
	github.com/inconshreveable/mousetrap v1.1.0 // indirect
	github.com/lestrrat-go/blackmagic v1.0.4 // indirect
	github.com/lestrrat-go/dsig v1.0.0 // indirect
	github.com/lestrrat-go/dsig-secp256k1 v1.0.0 // indirect
	github.com/lestrrat-go/httpcc v1.0.1 // indirect
	github.com/lestrrat-go/httprc/v3 v3.0.2 // indirect
	github.com/lestrrat-go/option/v2 v2.0.0 // indirect
	github.com/mattn/go-colorable v0.1.14 // indirect
	github.com/mattn/go-isatty v0.0.20 // indirect
	github.com/pmezard/go-difflib v1.0.0 // indirect
	github.com/robfig/cron/v3 v3.0.1 // indirect
	github.com/segmentio/asm v1.2.1 // indirect
	github.com/spf13/pflag v1.0.10 // indirect
	github.com/valyala/fastjson v1.6.7 // indirect
	github.com/vmihailenco/tagparser/v2 v2.0.0 // indirect
	golang.org/x/sys v0.42.0 // indirect
	gopkg.in/yaml.v3 v3.0.1 // indirect
)

// Replace directive for local development
replace gitlab.com/user/mylib => ../mylib

// Retract directive to mark bad versions
retract v1.0.1 // Contains critical bug
```

## Module Commands

```bash
# Initialize module
go mod init gitlab.com/user/project

# Add missing dependencies
go mod tidy

# Download dependencies
go mod download

# Verify dependencies
go mod verify

# Show module graph
go mod graph

# Show why package is needed
go mod why gitlab.com/user/package

# Vendor dependencies (copy to vendor/)
go mod vendor

# Update dependency
go get -u gitlab.com/user/package

# Update to specific version
go get gitlab.com/user/package@v1.2.3

# Update all dependencies
go get -u ./...

# Remove unused dependencies
go mod tidy
```

## Internal Packages

```go
// internal/ packages can only be imported by code in the parent tree

myproject/
├── internal/
│   ├── auth/           # Can only be imported by myproject
│   │   └── jwt.go
│   └── database/
│       └── database.go
└── pkg/
    └── models/         # Can be imported by anyone
        └── user.go

// This works (same project):
import "gitlab.com/user/myproject/internal/auth"

// This fails (different project):
import "github.com/other/project/internal/auth" // Error!

// Internal subdirectories
myproject/
└── api/
    └── internal/       # Can only be imported by code in api/
        └── helpers.go
```

## Package Organization

```go
// user/user.go - Domain package
package user

import (
    "context"
    "time"
)

// User represents a user entity
type User struct {
    ID        string
    Email     string
    CreatedAt time.Time
}

// Repository defines data access interface
type Repository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

// Service handles business logic
type Service struct {
    repo Repository
}

// NewService creates a new user service
func NewService(repo Repository) *Service {
    return &Service{repo: repo}
}

func (s *Service) RegisterUser(ctx context.Context, email string) (*User, error) {
    user := &User{
        ID:        generateID(),
        Email:     email,
        CreatedAt: time.Now(),
    }
    return user, s.repo.Create(ctx, user)
}
```

## Multi-Module Repository (Monorepo)

```
monorepo/
├── go.work              # Workspace file
├── services/
│   ├── api/
│   │   ├── go.mod
│   │   └── main.go
│   └── worker/
│       ├── go.mod
│       └── main.go
└── shared/
    └── models/
        ├── go.mod
        └── user.go

// go.work
go 1.26

use (
    ./services/api
    ./services/worker
    ./shared/models
)

// Commands:
// go work init ./services/api ./services/worker
// go work use ./shared/models
// go work sync
```

## Build Tags and Constraints

```go
// +build integration
// integration_test.go

package myapp

import "testing"

func TestIntegration(t *testing.T) {
    // Integration test code
}

// Build: go test -tags=integration

// File-level build constraints (Go 1.17+)
//go:build linux && amd64

package myapp

// Multiple constraints
//go:build linux || darwin
//go:build amd64

// Negation
//go:build !windows

// Common tags:
// linux, darwin, windows, freebsd
// amd64, arm64, 386, arm
// cgo, !cgo
```

## Taskfile Example

```yaml
version: "3"

vars:
  BIN: "{{.ROOT_DIR | toSlash}}/bin"
  EXT: "{{default exeExt .EXT}}"
  DEFAULT_APP:
    sh: basename {{.ROOT_DIR | toSlash}} -dev | tr '[:upper:]' '[:lower:]'
  GOFLAGS: -gcflags=-trimpath={{.ROOT_DIR | toSlash}} -asmflags=-trimpath={{.ROOT_DIR | toSlash}} -tags timetzdata {{.GO_VENDOR | default "-mod=vendor"}}
  LDFLAGS: -ldflags "-w -s -X main.BuildRevision=$REVISION -X main.BuildDate={{now | date "2006-01-02T15:04:05Z07:00"}} -X main.BuildTag=$TAG"

env:
  CGO_ENABLED: "0"
  GOOS: "{{default OS .GOOS}}"
  GOARCH: "{{default ARCH .GOARCH}}"

  # customize the next 3 variables
  GO_VENDOR: " " # remove this variable to use vendor folder
  DOCS_EXTRA_EXCLUDE_DIRS: ",."
  APP: APPNAME

tasks:
  default:
    cmds:
    - task: build

  build:
    cmds:
    - mkdir -p bin
    - go build {{.GOFLAGS}} {{.LDFLAGS}} -o bin/{{.APP | default .DEFAULT_APP}}{{.EXT}} {{.APP_FOLDER | default "."}}

  build:linux:
    cmds:
    - task: build
      vars:
        EXT: " "
        GOOS: linux
        GOARCH: amd64

  build:win:
    cmds:
    - task: build
      vars:
        EXT: ".exe"
        GOOS: windows
        GOARCH: amd64

  deploy:
    vars:
      DEPLOY_HOST: tsubus@nankatsu
      DEPLOY_PATH: /home/tsubus/bin
    env:
      GOOS: linux
      GOARCH: amd64
    cmds:
    - mkdir -p bin
    - go build {{.GOFLAGS}} {{.LDFLAGS}} -o bin/{{.APP | default .DEFAULT_APP}} {{.APP_FOLDER | default "."}}
    - scp bin/{{.APP | default .DEFAULT_APP}} {{.DEPLOY_HOST}}:{{.DEPLOY_PATH}}/{{.APP | default .DEFAULT_APP}}

  install:
    env:
      REVISION:
        sh: git rev-parse HEAD
      TAG:
        sh: git describe --tags --abbrev=0
    cmds:
    - go install {{.GOFLAGS}} {{.LDFLAGS}} {{.APP_FOLDER | default "."}}

  test:
    cmds:
    - go test -v ./...

  test:race:
    cmds:
    - CGO_ENABLED=1 go test -race -v ./...

  test:cover:
    cmds:
    - go test -cover html=coverage.out ./...

  test:integration:
    dotenv: ['.env']
    cmds:
    - INTEGRATION=1 go test -v ./...

  lint:
    cmds:
    - cmd: golangci-lint run -v

  lint:fix:
    cmds:
    - cmd: golangci-lint run --fix -v

  docs:
    cmds:
    # go install github.com/princjef/gomarkdoc/cmd/gomarkdoc@latest
    - gomarkdoc -f plain --output '{{"{{"}}.Dir{{"}}"}}/README.md' --exclude-dirs .../testdata/...,./vendor/...,./bin/...{{.DOCS_EXTRA_EXCLUDE_DIRS}} -e ./...

  openapi:
    cmds:
    - go run . openapi > openapi.yaml
```

## Dockerfile Multi-Stage Build

```dockerfile
FROM golang:latest as builder

WORKDIR /app

COPY . .

ARG CI_SERVER_PROTOCOL
ARG CI_JOB_TOKEN
ARG CI_SERVER_HOST
ARG CI_COMMIT_SHA
ARG CI_COMMIT_TAG

RUN \
  sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b /usr/local/bin && \
  git config --global --add url."$CI_SERVER_PROTOCOL://gitlab-ci-token:$CI_JOB_TOKEN@$CI_SERVER_HOST/tsubus-root/go".insteadOf "$CI_SERVER_PROTOCOL://$CI_SERVER_HOST/tsubus-go" && \
  REVISION=${CI_COMMIT_SHA} \
  TAG=${CI_COMMIT_TAG} \
  task build

FROM scratch

# copy app binary
COPY --from=builder /app/bin/ /usr/bin/

# copy certificates
COPY --from=builder /etc/ssl/certs/ /etc/ssl/certs/
COPY --from=builder /usr/local/share/ca-certificates/ /usr/local/share/ca-certificates/
COPY --from=builder /usr/share/ca-certificates/ /usr/share/ca-certificates

ENTRYPOINT [ "/usr/bin/lobstarr" ]
CMD [ "run" ]
```

## Version Information

```go
// main.go
package main

import (
	"os"

	"gitlab.com/user/project/cmd"
)

//nolint:gochecknoglobals // Set by ldflags during build
var (
	BuildRevision string
	BuildDate     string
	BuildTag      string
)

func main() {
	if err := cmd.Execute(BuildRevision, BuildDate, BuildTag); err != nil {
		os.Exit(1)
	}
}

// Build with version info:
// go build -ldflags "-X gitlab.com/user/project/version.BuildTag=1.0.0 \
//   -X gitlab.com/user/project/version.BuildRevision=$(git rev-parse HEAD) \
//   -X gitlab.com/user/project/version.BuildDate=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

## Go Generate

```go
// models/user.go
//go:generate mockgen -source=user.go -destination=../mocks/user_mock.go -package=mocks

package models

type UserRepository interface {
    GetUser(id string) (*User, error)
    SaveUser(user *User) error
}

// tools.go - Track tool dependencies
//go:build tools

package tools

import (
    _ "github.com/golang/mock/mockgen"
    _ "golang.org/x/tools/cmd/stringer"
)

// Install tools:
// go install github.com/golang/mock/mockgen@latest

// Run generate:
// go generate ./...
```

## Configuration Management

```go
// config/config.go
package config

import (
    "context"
    "time"

    "github.com/sethvargo/go-envconfig"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Redis    RedisConfig
}

type ServerConfig struct {
    Host         string        `env:"SERVER_HOST,default=0.0.0.0"`
    Port         int           `env:"SERVER_PORT,default=8080"`
    ReadTimeout  time.Duration `env:"SERVER_READ_TIMEOUT,default=10s"`
    WriteTimeout time.Duration `env:"SERVER_WRITE_TIMEOUT,default=10s"`
}

type DatabaseConfig struct {
    URL          string `env:"DATABASE_URL,required"`
    MaxOpenConns int    `env:"DB_MAX_OPEN_CONNS,default=25"`
    MaxIdleConns int    `env:"DB_MAX_IDLE_CONNS,default=5"`
}

type RedisConfig struct {
    Addr     string `env:"REDIS_ADDR,default=localhost:6379"`
    Password string `env:"REDIS_PASSWORD"`
    DB       int    `env:"REDIS_DB,default=0"`
}

// Load loads configuration from environment
func Load() (*Config, error) {
    ctx := context.Background()
    var cfg Config
    if err := envconfig.Process(ctx, &cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `go mod init` | Initialize module |
| `go mod tidy` | Add/remove dependencies |
| `go mod download` | Download dependencies |
| `go get package@version` | Add/update dependency |
| `go build -ldflags "-X ..."` | Set version info |
| `go generate ./...` | Run code generation |
| `GOOS=linux go build` | Cross-compile |
| `go work init` | Initialize workspace |
