# B-0001 — The company base images live in this standalone repository

| | |
|---|---|
| Status | Accepted |
| Date | 2026-10-05 |
| Platform decisions applied | DL-13, DL-28, DL-42 (D12), DL-47 of github-demo |

## Context

The `github-demo` monorepo built the two company base images — the runtime base `jre21` and the build image
`ci-build` — with its `base-image.yml` next to the connector apps, the pipeline and the design documents. The
apps moved out into `github-cicd-simple-apps` (R-0001 there) and kept consuming `ghcr.io/crazymatthsu/base/*`
from github-demo. A base image is a platform concern with its own owners (platform team, security for the CA),
its own cadence (weekly rebuilds, tool bumps, CA rotation) and its own blast radius (every app image, every
build job): it belongs in a repository that does nothing else, where a pull request to it can be reviewed and
approved on its own terms. The platform owner asked for exactly that: a standalone repository that only builds
base images for Java 21 applications and can grow toolchains, with `main` behind pull requests and one human
approval.

## Decision

1. **This repository is the base-image repository** of the platform (`platform.yml`: `kind: base`, DL-47). It
   holds the Dockerfiles, the CA bundle, the verification and the publishing workflow of the base images, and
   nothing that is built on them.
2. **The project is the repository**, as for the apps (R-0001): images are
   `ghcr.io/crazymatthsu/github-cicd-simple-base/<name>` — `<registry>/<project>/<name>` with the project of
   `platform.yml`. New names, because the GHCR packages `base/jre21` and `base/ci-build` are linked to
   github-demo, which a token of this repository cannot push to, while new names link to this repository on
   their first push. Consumers move to the new names with their next base bump.
3. **One image per directory, declared once.** `docker/base/<name>/Dockerfile` is the image; the workflows
   derive the matrix from the tree. Each image declares what it promises in `docker/base/<name>/verify.args`,
   the arguments of `scripts/ci/verify-image.sh`, which runs against the built image before any push. The CA
   bundle directory `docker/ca/` is the whole build context.
4. **Toolchains are blocks of `ci-build`**, each one `ARG` under a `# renovate:` hint, one checked install, one
   self-check line and one `--check` in `verify.args` (`docs/toolchains.md`). A toolchain that would double the
   image, or that one consumer alone needs, becomes an image of its own instead.
5. **Proven in the pull request, published from `main`.** `pr.yml` builds and verifies every image without
   pushing and reports `pr-gate`, the one required check; `base-image.yml` builds, verifies and publishes from
   `main`, weekly without the layer cache, and on demand. Both call the same `_build-image.yml`, so what the
   pull request proved is what `main` publishes. Dated tags `<yyyymmdd>-<run>` are immutable; `latest` moves
   only after the dated tag was pushed and resolves to the same digest.
6. **No release line, no writes to `main`.** The images are versioned by their dated tags alone; there is no
   release-please, no version file and no job with `contents: write`. The ruleset on `main` (pull request, one
   approval, `pr-gate`, linear history, no bypass) therefore blocks nothing the pipeline does.

## Alternatives considered

- **Keep the base images in github-demo** and grant the apps repository read access: one less repository, but
  the images stay tied to the monorepo's pipeline, reviewers and cadence, and every base change rides a
  monorepo pull request.
- **Build the base images inside each consumer** (the bootstrap path of `setup-build-env`): no shared image to
  publish, but N copies of the CA and toolchain steps drift, and a CA rotation touches every repository.
- **Keep the package names `base/*`** by granting this repository write access to github-demo's packages:
  avoids a consumer change, but leaves the packages linked to a repository that no longer owns them, and
  contradicts D12's `<registry>/<project>/<name>` naming (R-0001 made the same call for the apps).
- **One toolchain per install script** (`toolchains/<tool>.sh` composed by a `TOOLCHAINS` build argument):
  composable, but every pinned, checksum-verified install already is one block, and a script layer would hide
  the pins from hadolint and Renovate.

## Consequences

- The consumers (`github-cicd-simple-apps` first) change their `BASE_IMAGE` default, their `ci-build` probe
  and `CI_BUILD_IMAGE` pin to `ghcr.io/crazymatthsu/github-cicd-simple-base/<name>`; their `setup-build-env`
  bootstrap (building a base locally when none is published) becomes unnecessary once this repository has
  published.
- github-demo records DL-47 and keeps its own `docker/base/` and `base-image.yml` until the monorepo consumes
  these images; then they are removed there.
- The GHCR packages are created private by the first publishing run; their visibility is a repository
  setting (docs/README.md).
- `registry-login`, `resolve-image.sh` and `verify-image.sh` are vendored copies until `platform-ci` exists
  (D12 §6.9).

## References

- github-demo: D3 (`docs/03-docker-images.md`), D10 §5, D12 §6.1 / §6.6 / §6.10; ADRs DL-13, DL-28, DL-42, DL-47
- github-cicd-simple-apps: ADR R-0001 (the same naming decision for the apps)
