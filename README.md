# infrapi/ci

The CI of a Go project, to import and configure, on GitHub Actions or GitLab
CI: an API made with
[`infrapi-cli app init`](https://infrapi.github.io/products/cli/docs/develop/develop-an-api/),
and InfraPI's own repositories (lib, Kayra, infrapi-cli, tofu-api, ulgen).

- **on a push, a pull or merge request:** gofmt, `go vet`, golangci-lint, gosec,
  `go test -race`, and the build of the binary, if there is one;
- **the release:** GoReleaser reads the project's `.goreleaser.yaml`, so the project
  decides what a release builds: archives, deb and rpm packages, a multi-platform
  image. It comes from a tag `v<x.y.z>` you push (`release: tag`, the default), or, on
  GitHub, from go-semantic-release on a push to `main` (`release: semantic`).

`infrapi-cli app init` writes the caller below for the CI it is given (`--ci github`,
`--ci gitlab`). Every input has a default.

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
| `binary` | | the binary, built from `main`; none for a library |
| `main` | `./cmd/<binary>` | the binary's main package: `.` for one at the module's root |
| `go-version-file` | `go.mod` | where Go's version is read from, gofmt's too |
| `golangci-lint-version` | `v2.15.0` | |
| `gosec` | `true` | run gosec (`-exclude-dir _local`) |
| `gosec-version` | `v2.30.0` | |
| `goprivate` | | `GOPRIVATE`: the private modules' patterns, `github.com/acme/*` |
| `release` | `tag` | `tag`: on a tag `v<x.y.z>`; `semantic`: on a push to `main`, go-semantic-release makes the version, its tag and the GitHub release from the conventional commits, then GoReleaser adds the files to it (its `.goreleaser.yaml` has `release.mode: append`); `none` |
| `goreleaser` | `true` | the release runs GoReleaser; `false` for a library, whose release is its tag |
| `docker` | `false` | the release builds and pushes the image too |
| `registry` | `ghcr.io` | the image's registry |
| `goreleaser-version` | `~> v2` | |
| `notify-docs` | `false` | InfraPI's repositories: tell [infrapi/docs](https://github.com/infrapi/docs) and infrapi/.github after a release, a push to `main` changing `docs/`, or a manual run (`workflow_dispatch`) |

| Secret | |
|---|---|
| `GOPRIVATE_TOKEN` | a token reading the private modules, with `goprivate` |
| `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` | the registry's credentials; for ghcr.io, the job's own token |
| `DOCS_APP_PRIVATE_KEY` | with `notify-docs`: the private key of the "InfraPI docs" GitHub App (Contents read and write on infrapi/docs and infrapi/.github), whose id is the org variable `DOCS_APP_ID` |

| Output | |
|---|---|
| `version` | the version `release: semantic` released, empty when none |

An InfraPI repository's caller, Kayra's:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  go:
    uses: infrapi/ci/.github/workflows/go.yml@v1
    permissions:
      contents: write   # the tag and the release
      packages: write   # the image, on ghcr.io
    with:
      binary: kayra
      release: semantic
      docker: true
      notify-docs: true
    secrets: inherit
```

lib's has no `binary`, `goreleaser: false` and no `docker`.

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
| `binary` | | the binary, built from `main`; none for a library |
| `main` | `./cmd/<binary>` | the binary's main package |
| `go-image` | `docker.io/library/golang:1.27` | the image the checks run in |
| `golangci-lint-version` | `v2.15.0` | |
| `gosec` | `true` | run gosec |
| `gosec-version` | `v2.30.0` | |
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
is `{{ .Env.CI_REGISTRY_IMAGE }}` unless set otherwise. It runs on a tag `v<x.y.z>`:
a project that tags with semantic-release includes its own job for that.

## Versions

`v1` is a moving tag on the latest `v1.x.y`: a caller on `@v1` gets its fixes, not its
breaking changes. Pin `@v1.2.3` to stay put.
