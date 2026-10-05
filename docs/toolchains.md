# Adding a toolchain, adding an image

The build image `ci-build` is a JDK 21 plus a set of **toolchains**: the tools a consumer's build job, test
runner or laptop needs at one pinned version. Each toolchain is one block in
`docker/base/ci-build/Dockerfile`, one line in the image's self-check and one line in
`docker/base/ci-build/verify.args`. Nothing else knows the list, so adding one is a small, reviewable pull
request that `pr.yml` proves by building and verifying the image before anything is published.

## A toolchain, step by step

1. **Pin it.** Add a version `ARG` at the top of the `FROM ${TEMURIN_IMAGE}` stage (or before the tool
   stages for a static binary copied from an image), under a `# renovate:` hint so Renovate bumps it:

   ```dockerfile
   # renovate: datasource=github-releases depName=yannh/kubeconform
   ARG KUBECONFORM_VERSION=v0.8.0
   ```

   Datasources that work with the hint: `github-releases`, `github-tags` (with `extractVersion` when the tag
   has a prefix the value does not), `docker` (a tool image tag). The hint and the `ARG` line must be
   adjacent: `renovate.json` matches them as a pair.

2. **Install it, and check the download.** One `RUN` block (or a `FROM <tool-image> AS <tool>` stage plus a
   `COPY --from`). Every download is checked before it is installed: a release binary against the SHA-256 sum
   its project publishes, an apt repository through its signing key's fingerprint, a static binary by taking
   it from the tool's own image at a pinned tag. Download into `mktemp -d`, `install -m 0755` into
   `/usr/local/bin/`, remove the temporary directory, support `amd64` and `arm64` through `TARGETARCH`:

   ```dockerfile
   # Toolchain: kubeconform — release tarball checked against the project's CHECKSUMS file.
   RUN arch="${TARGETARCH:-$(dpkg --print-architecture)}" \
    && tmp="$(mktemp -d)" \
    && tgz="kubeconform-linux-${arch}.tar.gz" \
    && curl -fsSLo "${tmp}/${tgz}" "https://github.com/yannh/kubeconform/releases/download/${KUBECONFORM_VERSION}/${tgz}" \
    && echo "$(curl -fsSL "https://github.com/yannh/kubeconform/releases/download/${KUBECONFORM_VERSION}/CHECKSUMS" | grep " ${tgz}$" | cut -d ' ' -f 1)  ${tmp}/${tgz}" | sha256sum -c - \
    && tar -xzf "${tmp}/${tgz}" -C "${tmp}" kubeconform \
    && install -m 0755 "${tmp}/kubeconform" /usr/local/bin/kubeconform \
    && rm -rf "${tmp}"
   ```

   Apt packages from Ubuntu's own archive stay unpinned (the weekly rebuild picks up patches, and pinned
   versions vanish from the archive; `.hadolint.yaml` relaxes DL3008 for that reason). A third-party apt
   repository is added like Docker's: key fingerprint checked, package version pinned to the `ARG`.

3. **Make the build fail without it.** Add `&& <tool> --version` to the last `RUN` of the Dockerfile, the
   self-check that already runs every tool. A broken install then fails the image build, not a consumer's job.

4. **Check it from the outside.** Add `--check '<tool> --version'` to `docker/base/ci-build/verify.args`.
   `verify-image.sh` runs each `--check` in a fresh container of the built image, as the image's user, and
   `pr.yml` and `base-image.yml` refuse to push an image that fails one. A tool with a sub-command check
   (`docker buildx version`, `kubectl version --client`) is written the same way.

5. **Tell the consumers.** Mention the tool in the `LABEL org.opencontainers.image.description` of the
   Dockerfile and in the README's image table. Consumers whose host jobs install the same tool (a kind or
   Helm job that runs on the runner host, not in the container) pin the same version in their own
   `versions.env`; the dated tag they bump to carries the change.

The pull request's `build` job builds `ci-build`, runs the self-check and `verify-image.sh`, and the
`lint` job runs hadolint on the Dockerfile. When the pull request merges, `base-image.yml` publishes a new
`<yyyymmdd>-<run>` tag; consumers get it through their Renovate bump of the tag.

### Removing a toolchain

Remove the four pieces (ARG and hint, install block or stage, self-check line, `verify.args` line) in one
pull request. Consumers that still use the tool fail at their next bump, which is the right place to find out.

## A toolchain that needs its own image

A toolchain that only one consumer needs, or one that would double the image (a second language
runtime, a browser for end-to-end tests), gets its **own image** instead of growing `ci-build`:

1. Create `docker/base/<name>/` with a `Dockerfile` and a `verify.args`. The directory name is the image
   name: `ghcr.io/crazymatthsu/github-cicd-simple-base/<name>`, a lowercase OCI path segment.
2. Start the Dockerfile `FROM` the Temurin JDK (or `FROM` the published `ci-build` to layer on top of it)
   and copy the CA block from `ci-build/Dockerfile` unchanged, so the new image trusts the same bundle;
   the build context stays `docker/ca`.
3. End with a numeric non-root `USER`; a build-job image uses `1001:1001` (the GitHub-hosted runner UID) so
   the bind-mounted workspace keeps its owner.
4. Write `verify.args`: `--kind ci-build --expect-uid 1001` for a build-job image (its default tool checks
   assume the `ci-build` tool set; add `--no-default-checks` and your own `--check` lines for a different
   set), `--kind runtime` for a runtime base, plus `--tls-url https://github.com` to prove TLS through the
   OS store.

Nothing else changes: `pr.yml` and `base-image.yml` list the images from the tree, build and verify each
one, and publish them with the same dated tag.

## A different Java version

A runtime or build image for another JDK line is an image of its own (`jre25`, `ci-build-25`): copy the
Dockerfile, change `TEMURIN_IMAGE`, keep everything else. The images are siblings, never children of each
other, so a change to one line never rebuilds the other.
