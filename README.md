# .github

Reusable GitHub Actions workflows and composite actions shared across my Go
repositories.

Everything here is referenced with `pushkar-anand/.github/...@main`. Third-party
actions are pinned to full commit SHAs with the version in a trailing comment.

## Contents

| Path | Type | Purpose |
|------|------|---------|
| `.github/workflows/go-ci.yml` | Reusable workflow | Build, vet, gofmt check, `go mod tidy` check, tests, optional staticcheck lint |
| `.github/workflows/go-coverage.yml` | Reusable workflow | Test coverage with a per-package diff comment against `main` |
| `.github/workflows/go-autoformat.yml` | Reusable workflow | Runs `make fmt` and pushes the result back to the PR branch |
| `.github/workflows/go-release.yml` | Reusable workflow | GoReleaser binaries + multi-arch Docker images on GHCR |
| `.github/actions/go-coverage-comment` | Composite action | Posts/updates the coverage table comment on a PR |

---

## `go-ci.yml`

Build, vet, format check, tidy check and race-enabled tests. Optionally runs
staticcheck via reviewdog, which reports findings as inline PR review comments.

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: pushkar-anand/.github/.github/workflows/go-ci.yml@main
    with:
      run-lint: true
```

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `build-tags` | string | `""` | Space-separated Go build tags (e.g. `goolm`) applied to build, vet, test and staticcheck |
| `go-version-file` | string | `go.mod` | Path to the `go.mod` that pins the Go version |
| `run-lint` | boolean | `false` | Run the staticcheck job |
| `generate-command` | string | `""` | Command that regenerates code (e.g. `make generate`). When set, CI runs it and fails if the working tree is dirty afterwards |

### Jobs

- **Build, Vet & Test** — `go mod verify`, `go mod tidy` diff check, optional
  generate + drift check, build, vet, `gofmt -l`, `go test -race -count=1`.
- **Lint (staticcheck)** — only when `run-lint: true`. Needs
  `pull-requests: write`, which the reusable workflow requests itself.

---

## `go-coverage.yml`

Runs the test suite with coverage, caches the `main` profile as a baseline, and
comments a per-package coverage table with the delta versus `main` on every PR.
Packages with no change are collapsed into a `<details>` block. The comment is
edited in place on subsequent pushes rather than duplicated.

```yaml
jobs:
  coverage:
    uses: pushkar-anand/.github/.github/workflows/go-coverage.yml@main
```

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `build-tags` | string | `""` | Space-separated Go build tags |
| `go-version-file` | string | `go.mod` | Path to the `go.mod` that pins the Go version |

The baseline comes from an Actions cache written on pushes to `main`, so the
first PR after adoption shows totals without a delta.

---

## `go-autoformat.yml`

Runs `make fmt` on the PR branch and pushes a `style: apply gofmt + goreturns`
commit if anything changed. The calling repository must provide a `fmt` target
in its `Makefile`.

```yaml
jobs:
  autoformat:
    uses: pushkar-anand/.github/.github/workflows/go-autoformat.yml@main
```

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `go-version-file` | string | `go.mod` | Path to the `go.mod` that pins the Go version |

> [!NOTE]
> Commits pushed with `GITHUB_TOKEN` do not trigger further workflow runs. Call
> this from a workflow where that is acceptable, or use a PAT in the caller.

---

## `go-release.yml`

On a published release: builds binaries with GoReleaser, then builds and pushes
`linux/amd64` and `linux/arm64` images to GHCR and joins them under a single
multi-arch manifest tagged with the release tag and `latest`. The arm64 image is
built natively on `ubuntu-24.04-arm`.

```yaml
name: Release
on:
  release:
    types: [published]

jobs:
  release:
    uses: pushkar-anand/.github/.github/workflows/go-release.yml@main
    with:
      image-name: my-service
```

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `image-name` | string | **required** | Image name; the owner is taken from `github.repository_owner`, producing `ghcr.io/<owner>/<image-name>` |
| `go-version-file` | string | `go.mod` | Path to the `go.mod` that pins the Go version |

Requires a `.goreleaser.yml` and a `Dockerfile` at the repository root.

---

## `go-coverage-comment` action

Used internally by `go-coverage.yml`. It reads `coverage.out` (and
`main-coverage.out` if present) from the workspace and posts the rendered table.

```yaml
- uses: pushkar-anand/.github/.github/actions/go-coverage-comment@main
  with:
    pr-number: ${{ github.event.pull_request.number }}
    gh-token: ${{ secrets.GITHUB_TOKEN }}
```

### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `pr-number` | yes | Pull request number to comment on |
| `gh-token` | yes | Token with `pull-requests: write` |

---

## Maintenance

Third-party actions are SHA-pinned. To review what is out of date:

```bash
grep -rn 'uses: [a-z].*@' .github/workflows .github/actions
```

Resolve a new pin with:

```bash
gh api repos/<owner>/<repo>/git/ref/tags/<tag> --jq '.object.sha'
```

If the ref is an annotated tag the API returns a tag object; dereference it to
the commit with `gh api repos/<owner>/<repo>/git/tags/<sha> --jq '.object.sha'`,
since `uses:` must reference a commit SHA.
