# Publish Packages Action

GitHub Actions for publishing Linux packages (DEB and RPM) to a package server with hosted
APT and Yum repositories.

The upload paths and the privilege names in [Server permissions](#server-permissions)
follow the `/repository/<name>/` layout used by Sonatype Nexus Repository.

This repository provides three actions:

| Action | Description |
|---|---|
| `JohannesPosch/publish-packages-action@v1` | Publish both DEB and RPM packages |
| `JohannesPosch/publish-packages-action/upload-deb@v1` | Publish DEB packages to an APT repository |
| `JohannesPosch/publish-packages-action/upload-rpm@v1` | Publish RPM packages to a Yum repository |

## Requirements

The runner requires:

- `bash`
- `curl`
- `find`

The package server must contain hosted repositories for the package formats you want to publish:

- APT (hosted) for `.deb`
- Yum (hosted) for `.rpm`

It is recommended to use a dedicated package server user with only the permissions required for publishing packages.

## Publish DEB and RPM packages

The root action publishes all `.deb` and `.rpm` files found recursively below `packages-path`.

```yaml
- name: Publish packages
  uses: PoschLabs/publish-packages@v1
  with:
    packages-path: packages
    server-url: https://packages.example.com
    server-username: ${{ secrets.PACKAGE_SERVER_USERNAME }}
    server-password: ${{ secrets.PACKAGE_SERVER_PASSWORD }}
    apt-repository: apt
    yum-repository: yum
```

### Inputs

| Input | Required | Default | Description |
|---|---:|---|---|
| `packages-path` | No | `packages` | Directory containing the packages |
| `server-url` | Yes | | Base URL of the package server |
| `server-username` | Yes | | Package server username |
| `server-password` | Yes | | Package server password |
| `apt-repository` | No | `apt` | APT hosted repository |
| `yum-repository` | No | `yum` | Yum hosted repository |

The combined action expects both DEB and RPM packages to exist. If either package type is missing, the action fails.

## Publish DEB packages only

The `upload-deb` action can be used independently when a project only produces Debian packages.

```yaml
- name: Publish DEB packages
  uses: PoschLabs/publish-packages/upload-deb@v1
  with:
    packages-path: packages
    server-url: https://packages.example.com
    server-username: ${{ secrets.PACKAGE_SERVER_USERNAME }}
    server-password: ${{ secrets.PACKAGE_SERVER_PASSWORD }}
    repository: apt
```

### Inputs

| Input | Required | Default | Description |
|---|---:|---|---|
| `packages-path` | No | `packages` | Directory containing `.deb` packages |
| `server-url` | Yes | | Base URL of the package server |
| `server-username` | Yes | | Package server username |
| `server-password` | Yes | | Package server password |
| `repository` | No | `apt` | APT hosted repository |

All `.deb` files below `packages-path` are uploaded recursively.

The action fails if no `.deb` files are found.

## Publish RPM packages only

The `upload-rpm` action can be used independently when a project only produces RPM packages.

```yaml
- name: Publish RPM packages
  uses: PoschLabs/publish-packages/upload-rpm@v1
  with:
    packages-path: packages
    server-url: https://packages.example.com
    server-username: ${{ secrets.PACKAGE_SERVER_USERNAME }}
    server-password: ${{ secrets.PACKAGE_SERVER_PASSWORD }}
    repository: yum
```

### Inputs

| Input | Required | Default | Description |
|---|---:|---|---|
| `packages-path` | No | `packages` | Directory containing `.rpm` packages |
| `server-url` | Yes | | Base URL of the package server |
| `server-username` | Yes | | Package server username |
| `server-password` | Yes | | Package server password |
| `repository` | No | `yum` | Yum hosted repository |

All `.rpm` files below `packages-path` are uploaded recursively.

The action fails if no `.rpm` files are found.

## Server permissions

A dedicated package server user should be used for package publishing.

For repositories named `apt` and `yum`, a minimal publisher role can use the following privileges:

```text
nx-repository-view-apt-apt-add
nx-repository-view-apt-apt-read

nx-repository-view-yum-yum-add
nx-repository-view-yum-yum-edit
nx-repository-view-yum-yum-read
```

The exact privilege names depend on the repository names:

```text
nx-repository-view-<format>-<repository>-<action>
```

For example, if the Yum repository is called `linux-rpm`:

```text
nx-repository-view-yum-linux-rpm-add
nx-repository-view-yum-linux-rpm-edit
nx-repository-view-yum-linux-rpm-read
```

`delete` permissions are not required.

It is recommended to configure release repositories with **Disable redeploy** so that an already published package version cannot accidentally be overwritten.

## Example release workflow

A typical workflow can build packages in one job, sign them in another job, and publish the final artifacts afterwards.

```yaml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # Build packages...

      - name: Upload unsigned packages
        uses: actions/upload-artifact@v4
        with:
          name: unsigned-linux-packages
          path: |
            dist/*.deb
            dist/*.rpm
          if-no-files-found: error

  sign-release:
    needs: build

    runs-on:
      - self-hosted
      - signing-server

    permissions:
      contents: write
      id-token: write

    steps:
      - name: Download unsigned packages
        uses: actions/download-artifact@v4
        with:
          name: unsigned-linux-packages
          path: packages

      # Sign RPM packages...

      - name: Upload signed packages
        uses: actions/upload-artifact@v4
        with:
          name: signed-linux-packages
          path: |
            packages/*.deb
            packages/*.rpm
          if-no-files-found: error

      - name: Cleanup
        if: always()
        run: rm -rf packages

  publish-packages:
    needs: sign-release

    runs-on: ubuntu-latest

    steps:
      - name: Download signed packages
        uses: actions/download-artifact@v5
        with:
          name: signed-linux-packages
          path: packages

      - name: Publish packages
        uses: PoschLabs/publish-packages@v1
        with:
          packages-path: packages
          server-url: https://packages.example.com
          server-username: ${{ secrets.PACKAGE_SERVER_USERNAME }}
          server-password: ${{ secrets.PACKAGE_SERVER_PASSWORD }}
```

## Reusable workflows

To avoid writing a job yourself, call one of the reusable workflows. Each one downloads
your build artifacts, then runs the matching action.

| Workflow | Description |
|---|---|
| `.github/workflows/publish-packages.yml` | Publish DEB and RPM packages |
| `.github/workflows/publish-deb.yml` | Publish DEB packages only |
| `.github/workflows/publish-rpm.yml` | Publish RPM packages only |

```yaml
jobs:
  build:
    # ... builds packages and uploads them with actions/upload-artifact

  publish:
    needs: build
    uses: PoschLabs/publish-packages/.github/workflows/publish-packages.yml@v1
    with:
      server-url: https://packages.example.com
    secrets:
      server-username: ${{ secrets.PACKAGE_SERVER_USERNAME }}
      server-password: ${{ secrets.PACKAGE_SERVER_PASSWORD }}
```

### Inputs

| Input | Required | Default | Description |
|---|---:|---|---|
| `server-url` | Yes | | Base URL of the package server |
| `artifact-name` | No | all artifacts | Name of the artifact containing the packages |
| `packages-path` | No | `packages` | Directory the artifacts are downloaded to |
| `apt-repository` | No | `apt` | APT repository (`publish-packages` only) |
| `yum-repository` | No | `yum` | Yum repository (`publish-packages` only) |
| `repository` | No | `apt` / `yum` | Repository (`publish-deb` / `publish-rpm` only) |
| `runs-on` | No | `ubuntu-latest` | Runner label |

Secrets `server-username` and `server-password` are required.

## Versioning

Consumers should reference a stable major version:

```yaml
uses: PoschLabs/publish-packages@v1
```

Specific releases can also be pinned:

```yaml
uses: PoschLabs/publish-packages@v1.0.0
```

For security-sensitive environments, pinning the action to a full commit SHA provides the strongest protection against changes to tags.

## License

See [LICENSE](LICENSE).
