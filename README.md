# infrapi/ci

The CI of a Go API made with
[`infrapi-cli app init`](https://infrapi.github.io/products/cli/docs/develop/develop-an-api/), to
import and configure, on GitHub Actions or GitLab CI:

- **on a push, a pull or merge request:** `go vet`, golangci-lint, `go test -race`, and
  `go build ./cmd/<binary>`;
- **on a tag `v<x.y.z>`:** the release. GoReleaser reads the project's
  `.goreleaser.yaml`, so the project decides what a release builds: archives, deb and
  rpm packages, a multi-platform image.

`infrapi-cli app init` writes the caller below for the CI it is given (`--ci github`,
`--ci gitlab`). Every input but `binary` has a default.

## GitHub Actions

A reusable workflow, [.github/workflows/go.yml](.github/workflows/go.yml).
`.github/workflows/ci.yml` in the project:

```yaml
name: CI

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:

jobs:
  go:
    uses: infrapi/ci/.github/workflows/go.yml@v1
    permissions:
      contents: write   # the release
      packages: write   # the image, on ghcr.io
    with:
      binary: compute
      docker: true
    secrets: inherit
```

| Input | Default | |
|---|---|---|
| `binary` | (required) | the binary, built from `./cmd/<binary>` |
| `go-version-file` | `go.mod` | where Go's version is read from |
| `golangci-lint-version` | `v2.15.0` | |
| `goprivate` | | `GOPRIVATE`: the private modules' patterns, `github.com/acme/*` |
| `docker` | `false` | the release builds and pushes the image too |
| `registry` | `ghcr.io` | the image's registry |
| `goreleaser-version` | `~> v2` | |

| Secret | |
|---|---|
| `GOPRIVATE_TOKEN` | a token reading the private modules, with `goprivate` |
| `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` | the registry's credentials; for ghcr.io, the job's own token |

## GitLab CI

An include with inputs, [gitlab/go.yml](gitlab/go.yml). `.gitlab-ci.yml` in the
project:

```yaml
include:
  - remote: https://raw.githubusercontent.com/infrapi/ci/v1/gitlab/go.yml
    inputs:
      binary: compute
      docker: true
```

| Input | Default | |
|---|---|---|
| `binary` | (required) | the binary, built from `./cmd/<binary>` |
| `go-image` | `docker.io/library/golang:1.27` | the image the checks run in |
| `golangci-lint-version` | `v2.15.0` | |
| `goprivate` | | `GOPRIVATE`: the private modules' patterns, `gitlab.example.com/*` |
| `docker` | `false` | the release builds and pushes the image too, in `docker:dind` |
| `registry` | the project's | the image's registry |
| `goreleaser-image` | `docker.io/goreleaser/goreleaser:latest` | |

| CI/CD variable | |
|---|---|
| `GOPRIVATE_TOKEN` | a token reading the private modules; `CI_JOB_TOKEN` when unset |
| `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` | with `registry`: its credentials |

The release is published to the project's GitLab release with the job's token: the
project's `.goreleaser.yaml` has `gitlab_urls` with `use_job_token: true`, and its image
is `{{ .Env.CI_REGISTRY_IMAGE }}` unless set otherwise.

## Versions

`v1` is a moving tag on the latest `v1.x.y`: a caller on `@v1` gets its fixes, not its
breaking changes. Pin `@v1.2.3` to stay put.
