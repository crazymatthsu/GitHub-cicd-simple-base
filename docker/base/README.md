# Company base images

One directory per image: `Dockerfile` and `verify.args`. The directory name is the image name under
`ghcr.io/crazymatthsu/github-cicd-simple-base/` (the registry and project come from `platform.yml`). The
workflows read this directory, so a new image is a new directory, nothing else.

| Image | From | Adds | Promises checked by `verify.args` |
|---|---|---|---|
| `jre21` | `eclipse-temurin:21-jre` | the CA in the OS store and the JVM `cacerts`, `tzdata`, `curl`, user `app` (10001:10001), `/app`, `/app/logs` (writable), `/config`, `TZ=UTC` | non-root user, every bundle certificate in both stores, `java`, `curl`, the directories, `TZ`, TLS to github.com |
| `ci-build` | `eclipse-temurin:21-jdk` | the same CA steps; the toolchains: Docker CLI with compose and buildx, `helm`, `kubectl`, `kind`, `hadolint`, `shellcheck`, `yq`, `jq`, `git`, `curl`, `crane`; user `runner` (1001:1001), `WORKDIR /workspace` | UID 1001, the CA in both stores, `java` and `javac`, every tool runs, `/workspace` writable, TLS to github.com |

Both images label the bundle they carry: `com.example.ca-bundle=<CA_BUNDLE_VERSION>` (D3 §6.5); the same
label on an app image comes from its base. The build context of both is `docker/ca`, the directory holding the
CA bundle: neither Dockerfile copies anything else, so the context stays tiny and the enterprise passes the
directory its versioned bundle was downloaded into instead.

Both Dockerfiles pass `hadolint --failure-threshold style` with `.hadolint.yaml`: the only suppressions are
inline, for unpinned apt packages (DL3008: the weekly rebuild exists to pick up Ubuntu security fixes, and
pinned versions vanish from the archive) and for sourcing `/etc/os-release` (SC1091). Docker's client packages
are pinned (`ARG`s) and its repository key is checked against the published fingerprint; `helm`, `kubectl` and
`kind` are checked against the SHA-256 files their projects publish; `hadolint`, `shellcheck`, `yq` and
`crane` are static binaries from their official images at pinned tags. The enterprise pins the sums as `ARG`s
and downloads through JFrog remotes.

## Verification

Each build verifies itself: it fails unless `keytool -list -cacerts -alias <CA_ALIAS>` finds the CA, `openssl
verify` accepts every bundle certificate against the OS store and, for `ci-build`, every tool answers
`--version`. Then `scripts/ci/verify-image.sh` checks the loaded image from the outside with the arguments in
`verify.args` (user, every certificate in both stores by fingerprint, tools, TLS through the OS store), and
only a passing image is pushed.

## Tags

`<yyyymmdd>-<run_number>` (D3 §6.1): the UTC build date and the workflow run number. Tags are immutable and
sort chronologically; `_build-image.yml` refuses to publish a tag that already exists. `latest` follows the
newest build and is proven to point at the dated digest after every push. Consumers pin a dated tag (Renovate
bumps it with the regex versioning in the README) or resolve `latest` to a digest once per run.

## Toolchains

`ci-build`'s toolchains are the `ARG`s at the top of its Dockerfile, each under a `# renovate:` hint, each
installed by one checked block and listed in the self-check and in `verify.args`. Adding, removing or
isolating one into its own image: [`../../docs/toolchains.md`](../../docs/toolchains.md).

## CA rotation

A rebuild cascade, not a runtime mount (D3 §6.3; [`../ca/README.md`](../ca/README.md)): add the new root
next to the old one, publish with a new `CA_BUNDLE_VERSION` (`base-image.yml` input `ca-bundle-version`), let
the consumers bump their tags, remove the old root once every environment carries the new label.
