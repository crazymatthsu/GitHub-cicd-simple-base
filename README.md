# github-cicd-simple-base — company base images for Java 21 applications

The two company base images of the Deephaven data platform, extracted from the
[`github-demo`](https://github.com/crazymatthsu/github-demo) monorepo into a **base-image repository**
(platform decision DL-47; design document D3 and ADRs DL-13, DL-28 there). It builds, verifies and
publishes base images and nothing else: no application code, no deployment.

| Image | From | Adds | Used by |
|---|---|---|---|
| `ghcr.io/crazymatthsu/github-cicd-simple-base/jre21` | `eclipse-temurin:21-jre` | the enterprise CA in the OS store and the JVM `cacerts`, `tzdata`, `curl`, user `app` (10001:10001), `/app`, `/app/logs`, `/config`, `TZ=UTC` | the `FROM` of every Java 21 app image |
| `ghcr.io/crazymatthsu/github-cicd-simple-base/ci-build` | `eclipse-temurin:21-jdk` | the same CA steps and the **toolchains**: Docker CLI with compose and buildx, `helm`, `kubectl`, `kind`, `hadolint`, `shellcheck`, `yq`, `jq`, `git`, `curl`, `crane`; user `runner` (1001:1001, the GitHub-hosted runner UID), `WORKDIR /workspace` | the `container:` of a consumer's build job, its test-runner compose service, laptops |

Tags are `<yyyymmdd>-<run_number>`, immutable; `latest` follows the newest build. Every tag is published
only after the image passed its checks from the outside; `latest` is then proven to point at the same digest.

| Path | What |
|---|---|
| `platform.yml` | the manifest: platform major, kind `base`, registry and project — the image path is `<registry>/<project>/<name>` |
| `docker/base/<name>/Dockerfile` | one image per directory; the directory name is the image name |
| `docker/base/<name>/verify.args` | what the image promises: the arguments of `scripts/ci/verify-image.sh`, run against the built image before it is pushed |
| `docker/ca/` | the CA bundle and the whole build context of both images ([`docker/ca/README.md`](docker/ca/README.md)) |
| `scripts/ci/` | `verify-image.sh` (outside checks), `resolve-image.sh` (tag → digest, no pull) |
| `scripts/test/` | plain-bash tests of the scripts (stubbed `docker`, throwaway CA) |
| `.github/workflows/` | `pr.yml` (lint, build and verify without pushing, `pr-gate`), `base-image.yml` (publish), `_build-image.yml` (the shared build → verify → push job) |
| `docs/` | [`docs/README.md`](docs/README.md), [`docs/toolchains.md`](docs/toolchains.md) (adding a toolchain or an image), [`docs/adr/`](docs/adr/README.md) |

## Build and check locally

```bash
tag=dev
docker buildx build -f docker/base/jre21/Dockerfile    -t ghcr.io/crazymatthsu/github-cicd-simple-base/jre21:$tag    --load docker/ca
docker buildx build -f docker/base/ci-build/Dockerfile -t ghcr.io/crazymatthsu/github-cicd-simple-base/ci-build:$tag --load docker/ca

# the same checks CI runs before pushing (docker/base/<name>/verify.args holds each image's arguments)
scripts/ci/verify-image.sh --kind runtime  --check 'java -version' --tls-url https://github.com ghcr.io/crazymatthsu/github-cicd-simple-base/jre21:$tag
scripts/ci/verify-image.sh --kind ci-build --expect-uid 1001 ghcr.io/crazymatthsu/github-cicd-simple-base/ci-build:$tag

# Podman: --format docker keeps Docker-format metadata such as HEALTHCHECK in derived images (D3 §6.7)
podman build --format docker -f docker/base/jre21/Dockerfile -t ghcr.io/crazymatthsu/github-cicd-simple-base/jre21:$tag docker/ca
```

Build arguments (all optional): `TEMURIN_IMAGE` (enterprise: the JFrog remote, pinned by digest),
`CA_BUNDLE_FILE`, `CA_BUNDLE_VERSION` (the `com.example.ca-bundle` label), `CA_ALIAS`, the OCI label values
`IMAGE_VERSION` / `GIT_SHA` / `CREATED` (set by the workflow), and one `<TOOL>_VERSION` per toolchain of
`ci-build` (see the `ARG`s at the top of its Dockerfile).

## Pipeline

| Workflow | Trigger | Does |
|---|---|---|
| `pr.yml` | branch push; pull request | lint (hadolint, ShellCheck, script tests, actionlint, every image has a `verify.args`) → on a pull request also build **and verify every image without pushing** → **`pr-gate`**, the one required check |
| `base-image.yml` | merge to `main` touching `docker/**`, `scripts/ci/**`, `platform.yml` or the workflows; Mondays 04:23 UTC (no layer cache); manual (one image, no cache, a CA bundle version) | for every `docker/base/<name>/`: build → verify → push `<yyyymmdd>-<run>` and `latest`, `latest` checked against the dated digest |
| `_build-image.yml` | reusable | the one job both call, with `push: false` or `push: true` |

A change to an image is therefore proven twice before anyone consumes it: in the pull request (build and
verify, nothing published) and on `main` (the same, then published). Details, the first-run notes and the
repository settings the pipeline needs: [`docs/README.md`](docs/README.md).

## Consuming the images

```dockerfile
ARG BASE_IMAGE=ghcr.io/crazymatthsu/github-cicd-simple-base/jre21:20261005-7
FROM ${BASE_IMAGE}
```

```yaml
jobs:
  build:
    container:
      image: ghcr.io/crazymatthsu/github-cicd-simple-base/ci-build:20261005-7@sha256:…
```

Pin a dated tag (with its digest where the consumer records digests) and let Renovate bump it: the tags
are `<yyyymmdd>-<n>`, so the Renovate rule is
`"versioning": "regex:^(?<major>\\d{4})(?<minor>\\d{2})(?<patch>\\d{2})-(?<build>\\d+)$"`. `latest` is for
bootstrapping and local use; a pipeline resolves it to a digest once per run (`scripts/ci/resolve-image.sh`).

## Adding a toolchain or an image

A toolchain is one block in `docker/base/ci-build/Dockerfile` (a version `ARG` under a `# renovate:` hint, a
checked install, a line in the self-check) plus one `--check` line in `docker/base/ci-build/verify.args`. An
image is a new directory `docker/base/<name>/` with a `Dockerfile` and a `verify.args`; the workflows pick it
up from the tree. Both are described step by step in [`docs/toolchains.md`](docs/toolchains.md).

## What is vendored, and why

`.github/actions/registry-login`, `scripts/ci/resolve-image.sh` and `scripts/ci/verify-image.sh` are the
platform's shared pieces (D12 §6.9 has every repository consume them from a versioned `platform-ci`
repository, which does not exist yet). Until then they are copies — `registry-login` and `resolve-image.sh`
from github-demo, `verify-image.sh` from its `gha-build-images` skill — and a fix is made there and ported here.
