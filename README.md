# Container-On-Arch GitHub Action

[![](https://github.com/uraimo/run-on-arch-action/workflows/test/badge.svg)](https://github.com/uraimo/run-on-arch-action)

A GitHub Action that executes commands on non-x86 CPU architecture (armv6, armv7, aarch64, s390x, ppc64le) via QEMU.

Originally forked and based on [Run-On-Arch](https://github.com/uraimo/run-on-arch-action) action.

## Usage

This action requires three input parameters:

- `arch`: CPU architecture: `armv6`, `armv7`, `aarch64`, `s390x`, or `ppc64le`. See [Supported Platforms](#supported-platforms) for the full matrix.
- `image`: Image name. Examples: `ubuntu:20.04`, `debian:bullseye`, `nginx`,
- `run`: Shell commands to execute in the container.

The action also accepts some optional input parameters:

- `githubToken`: Your GitHub token, used for logging in to pull private images in ghcr.io. Usually this would just be `${{ github.token }}`.
- `env`: Environment variables to propagate to the container. YAML, but must begin with a `|` character. These variables will be available in both run and setup.
- `shell`: The shell to run commands with in the container. Default: `/bin/sh`.
- `dockerRunArgs`: Additional arguments to pass to `docker run`, such as volume mappings. See [`docker run` documentation](https://docs.docker.com/engine/reference/commandline/run).
- `setup`: Shell commands to execute on the host before running the container, such as creating directories for volume mappings.

### Basic example

A basic example that sets an output variable for use in subsequent steps:

```yaml
on: [push, pull_request]

jobs:
  armv7_job:
    # The host should always be Linux
    runs-on: ubuntu-18.04
    name: Build on ubuntu-18.04 armv7
    steps:
      - uses: actions/checkout@v2.1.0
      - uses: uraimo/run-on-arch-action@v2
        name: Run commands
        id: runcmd
        with:
          arch: armv7
          distro: ubuntu18.04

          # Not required, but speeds up builds by storing container images in
          # a GitHub package registry.
          githubToken: ${{ github.token }}

          # Set an output parameter `uname` for use in subsequent steps
          run: |
            uname -a
            echo ::set-output name=uname::$(uname -a)

      - name: Get the output
        # Echo the `uname` output parameter from the `runcmd` step
        run: |
          echo "The uname output was ${{ steps.runcmd.outputs.uname }}"
```

### Advanced example

This shows how to use a matrix to produce platform-specific artifacts, and includes example values for the optional input parameters `setup`, `shell`, `env`, and `dockerRunArgs`.

```yaml
on: [push, pull_request]

jobs:
  build_job:
    # The host should always be linux
    runs-on: ubuntu-18.04
    name: Build on ${{ matrix.distro }} ${{ matrix.arch }}

    # Run steps on a matrix of 3 arch/distro combinations
    strategy:
      matrix:
        include:
          - arch: aarch64
            distro: ubuntu18.04
          - arch: ppc64le
            distro: alpine_latest
          - arch: s390x
            distro: fedora_latest

    steps:
      - uses: actions/checkout@v2.1.0
      - uses: uraimo/run-on-arch-action@v2
        name: Build artifact
        id: build
        with:
          arch: ${{ matrix.arch }}
          distro: ${{ matrix.distro }}

          # Not required, but speeds up builds
          githubToken: ${{ github.token }}

          # Create an artifacts directory
          setup: |
            mkdir -p "${PWD}/artifacts"

          # Mount the artifacts directory as /artifacts in the container
          dockerRunArgs: |
            --volume "${PWD}/artifacts:/artifacts"

          # Pass some environment variables to the container
          env: | # YAML, but pipe character is necessary
            artifact_name: git-${{ matrix.distro }}_${{ matrix.arch }}

          # The shell to run commands with in the container
          shell: /bin/sh

          # Install some dependencies in the container. This speeds up builds if
          # you are also using githubToken. Any dependencies installed here will
          # be part of the container image that gets cached, so subsequent
          # builds don't have to re-install them. The image layer is cached
          # publicly in your project's package repository, so it is vital that
          # no secrets are present in the container state or logs.
          install: |
            case "${{ matrix.distro }}" in
              ubuntu*|jessie|stretch|buster|bullseye)
                apt-get update -q -y
                apt-get install -q -y git
                ;;
              fedora*)
                dnf -y update
                dnf -y install git which
                ;;
              alpine*)
                apk update
                apk add git
                ;;
            esac

          # Produce a binary artifact and place it in the mounted volume
          run: |
            cp $(which git) "/artifacts/${artifact_name}"
            echo "Produced artifact at /artifacts/${artifact_name}"

      - name: Show the artifact
        # Items placed in /artifacts in the container will be in
        # ${PWD}/artifacts on the host.
        run: |
          ls -al "${PWD}/artifacts"
```

## Architecture emulation

This project makes use of an additional QEMU container to be able to emulate via software architectures like ARM, s390x, ppc64le, etc... that are not natively supported by GitHub. You should keep this into consideration when reasoning about the expected running time of your jobs, there will be a visible impact on performance when compared to a job executed on a vanilla runner.

## Authors

[Umberto Raimondi](https://github.com/uraimo)

[Elijah Shaw-Rutschman](https://github.com/elijahr)

[Abel Boldú](https://github.com/vdo)

## License

This project is licensed under the [BSD 3-Clause License](https://github.com/uraimo/run-on-arch-action/blob/master/LICENSE).
