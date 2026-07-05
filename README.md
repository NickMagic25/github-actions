# Reusable GitHub Actions and Runner Images

This repository contains reusable composite GitHub Actions and APKO image definitions that can be shared across public and private projects.

The current focus is on:

- Building and publishing Docker images.
- Validating and publishing Helm charts as OCI artifacts.
- Building Wolfi packages with Melange.
- Building or publishing APKO images, optionally including locally built Melange packages.
- Keeping common CI runner images in one place.

## Repository Layout

```text
.
├── apko-publish/       # Composite action for APKO image builds/publishes
├── docker-build/       # Composite action for Docker buildx builds/publishes
├── helm-oci-publish/   # Composite action for Helm chart OCI publishes
├── images/             # APKO image definitions
└── melange-build/      # Composite action for Melange package builds
```

## Actions

Use these actions from another workflow with:

```yaml
uses: NickMagic25/github-actions/<action-directory>@main
```

For long-lived workflows, prefer pinning to a tag or commit SHA once you start versioning this repository.

### `docker-build`

Logs in to a container registry on non-PR events, then builds a Docker image with `docker buildx`.

- Pull requests: builds the image without pushing it.
- Non-PR events: builds and pushes the image.

Example:

```yaml
jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: NickMagic25/github-actions/docker-build@main
        with:
          dockerfile: ./Dockerfile
          context: .
          image-name: ghcr.io/my-org/my-app:${{ github.sha }}
          registry: ghcr.io
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

Inputs:

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `dockerfile` | Yes |  | Path to the Dockerfile. |
| `context` | Yes |  | Docker build context path. |
| `image-name` | Yes |  | Full image name including tag. |
| `platform` | No | `linux/amd64` | Target build platform. |
| `registry` | No |  | Registry hostname. Required for push events. |
| `registry-username` | No |  | Registry username. Required for push events. |
| `registry-password` | No |  | Registry password or token. Required for push events. |

### `helm-oci-publish`

Validates a Helm chart, packages it, and publishes it to an OCI registry.

- Pull requests: runs dependency build, lint, render validation, and local packaging without publishing.
- Non-PR events: runs the same validation and packaging, logs in to the registry, and pushes the packaged chart with `helm push`.

Example:

```yaml
jobs:
  chart:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: NickMagic25/github-actions/helm-oci-publish@main
        with:
          chart-path: charts/my-app
          registry: ghcr.io
          registry-project: my-org/charts
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

Inputs:

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `chart-path` | Yes |  | Path to the Helm chart directory. |
| `registry` | No |  | Registry hostname. Required for non-PR publishing. |
| `registry-project` | No |  | Registry namespace or project path. Optional for registries that publish at the root. |
| `registry-username` | No |  | Registry username. Required for non-PR publishing. |
| `registry-password` | No |  | Registry password or token. Required for non-PR publishing. |
| `package-destination` | No | `.helm-packages` | Directory where the packaged chart archive is written. |
| `chart-version` | No |  | Override the chart version during packaging. |
| `app-version` | No |  | Override the app version during packaging. |
| `dependency-build` | No | `true` | Run `helm dependency build` before validation. |
| `lint-strict` | No | `false` | Run `helm lint` with `--strict`. |

Output:

| Output | Description |
| --- | --- |
| `package-path` | Path to the packaged chart archive. |

### `melange-build`

Generates a Melange signing key, builds one or more Melange package configs, and uploads the generated packages plus public key as a workflow artifact.

The artifact name is:

```text
<component>-melange-packages
```

Example:

```yaml
jobs:
  packages:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          path: src

      - uses: NickMagic25/github-actions/melange-build@main
        with:
          component: my-component
          configs: melange.yaml
          source-path: src
          arch: x86_64
          runner: docker
```

Inputs:

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `component` | Yes |  | Component directory name. Used for paths and artifact naming. |
| `configs` | Yes |  | Comma-separated Melange config paths relative to the component directory. |
| `source-path` | Yes |  | Checkout path of the source repository. |
| `arch` | No | `x86_64` | Build architecture. |
| `runner` | No | `docker` | Melange runner backend: `bubblewrap`, `docker`, or `qemu`. |

Notes:

- The action writes packages to `<source-path>/<component>/packages`.
- The action writes the signing key to `<source-path>/<component>/melange.rsa`.
- For `runner: qemu`, the action provisions an Alpine `vmlinuz-virt` kernel if `QEMU_KERNEL_IMAGE` is not already set.

### `apko-publish`

Builds or publishes an APKO image.

- Pull requests: builds a local image tarball without publishing.
- Non-PR events: publishes the image to the configured registry.
- If `component` is provided, the action downloads the matching Melange artifact and appends the generated package repository and signing key to the APKO build.
- If `component` is omitted, the image is built directly from public APKO repositories.

Example with Melange packages:

```yaml
jobs:
  image:
    runs-on: ubuntu-latest
    needs: packages
    steps:
      - uses: actions/checkout@v4
        with:
          path: src

      - uses: NickMagic25/github-actions/apko-publish@main
        with:
          component: my-component
          source-path: src
          apko-config: apko.yaml
          image-name: my-component
          image-tag: ${{ github.sha }}
          registry: ghcr.io
          registry-project: my-org
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

Example for an image that only uses public package repositories:

```yaml
jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: NickMagic25/github-actions/apko-publish@main
        with:
          apko-config: images/base/apko.yaml
          image-name: base
          image-tag: latest
          registry: ghcr.io
          registry-project: my-org
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

Inputs:

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `component` | No |  | Component name. Enables Melange artifact download and local package repository use. |
| `image-name` | Yes |  | Image name suffix for the registry reference. |
| `image-tag` | No | `latest` | Image tag to build or publish. |
| `apko-config` | Yes |  | Path to `apko.yaml`. Relative to the component directory when `component` is set, otherwise relative to the workspace root. |
| `source-path` | No |  | Checkout path of the source repo. Required when `component` is set. |
| `registry` | Yes |  | Registry hostname. |
| `registry-project` | Yes |  | Registry namespace or project. |
| `registry-username` | No |  | Registry username. Required for non-PR publishing. |
| `registry-password` | No |  | Registry password or token. Required for non-PR publishing. |
| `arch` | No | `x86_64` | Build architecture. |

## Image Definitions

The `images/` directory contains APKO definitions for reusable CI images. They are based on Wolfi packages, run as a non-root `runner` user with UID/GID `1001`, set `HOME=/home/runner`, and use `/bin/sh` as the entrypoint.

| Image | Config | Included tools |
| --- | --- | --- |
| `base` | `images/base/apko.yaml` | Shell/core utilities, Git, curl, APKO, Melange, Bubblewrap, Docker CLI, buildx, BuildKit. |
| `flux` | `images/flux/apko.yaml` | Shell/core utilities, Git, curl, Flux. |
| `go` | `images/go/apko.yaml` | Shell/core utilities, Git, curl, Go. |
| `python` | `images/python/apko.yaml` | Shell/core utilities, Git, curl, Python 3.12 dev base, uv. |
| `terraform` | `images/terraform/apko.yaml` | Shell/core utilities, Git, curl, OpenTofu 1.11, Terragrunt. |

## Publishing the Shared Images

Each shared image can be published with `apko-publish` because these definitions only use public Wolfi repositories.

```yaml
jobs:
  publish-images:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        image:
          - base
          - flux
          - go
          - python
          - terraform
    steps:
      - uses: actions/checkout@v4

      - uses: NickMagic25/github-actions/apko-publish@main
        with:
          apko-config: images/${{ matrix.image }}/apko.yaml
          image-name: ${{ matrix.image }}
          image-tag: latest
          registry: ghcr.io
          registry-project: my-org/github-actions
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

## Runner Requirements

These composite actions expect the runner environment to provide the tools they call:

- `docker` and `docker buildx` for `docker-build`.
- `helm` 3.8 or newer for `helm-oci-publish`.
- `melange` for `melange-build`.
- `apko` for `apko-publish`.
- `curl` when using `melange-build` with `runner: qemu`.

The `images/base/apko.yaml` image is intended to provide the common toolchain for APKO and Melange workflows.

## Versioning

Until tags are added, consumers can reference `@main` for the latest version:

```yaml
uses: NickMagic25/github-actions/docker-build@main
```

Once workflows depend on these actions in production, create immutable release tags and update consumers to use those tags or commit SHAs.
