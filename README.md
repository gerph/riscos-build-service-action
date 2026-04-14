# RISC OS Build Service GitHub Action

A GitHub composite action that calls the [RISC OS Build Service](https://build.riscos.online/)
to build RISC OS projects without requiring a local RISC OS environment.

This means that you can very easily build RISC OS projects
without having to deal with the communication with the
build service, or toolchain issues.

## What it does

Your repository can contain code to build RISC OS components, or to run RISC OS binaries. This action
runs the binaries, and returns the results (output and
archived files) back to you.

### How it does it

This action wraps the `riscos-build-online` client tool, allowing you to:

- Send your repository files to the RISC OS Build Service
- Execute build commands defined in a `.robuild.yaml` file
- Retrieve the built output as an artifact
- Configure the architecture (32-bit or 64-bit), timeout, and other build parameters

## Prerequisites

Your repository must contain a `.robuild.yaml` file that describes what should be built on
RISC OS. See the [RISC OS Build documentation](https://build.riscos.online/robuildyaml.html) for the
file format.

## Usage

Add the action to a workflow step, referencing this repository and the desired version:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

        uses: gerph/riscos-build-service-action@v1
        with:
          architecture: aarch32
          timeout: 120
          output: /tmp/built
```

### Inputs

| Input | Description | Default |
|---|---|---|
| `architecture` | Architecture for the build: `aarch32` or `aarch64` | `aarch32` |
| `timeout` | Build timeout in seconds (max 600) | `10` |
| `colour` | Enable coloured output: `yes` or `no` | `yes` |
| `files` | Space-separated file patterns to send to the service (glob wildcards supported) | `*` |
| `quiet` | Output verbosity: `no` (full), `yes` (quiet non-failure), `silent` (minimal) | `no` |
| `directory` | Working directory for collecting files | `.` |
| `output` | Output file location (or prefix for extracted files) | `output` |
| `capture-filename` | Filename to write the captured build output (stdout/stderr) to, or empty to write to the console | *(empty)* |
| `release-name` | Name of the release archive (e.g. `MyModule-1.00`). Overrides `output`. If empty, the name is derived from `VersionNum` when `release` is `yes` | *(empty)* |
| `release` | When `yes`, automatically determine the release name from `VersionNum` (as `ComponentName-Version`) and upload the resulting artifact | `no` |
| `tool-version` | Version of `riscos-build-online` to use | `0.07` |

### Outputs

| Output | Description |
|---|---|
| `output_filename` | Full path of the output file created by the build, or empty if none. When `release-name` or `release` is used, this is the versioned name |
| `output_filetype` | RISC OS filetype extension of the output (e.g. `a91`, `feb`, `ff8`) |
| `capture_filename` | Filename created for captured console output, or empty |
| `project_version` | Version number extracted from `VersionNum`, or the short git SHA if not found |

## Example

A typical workflow that builds a project and uploads the result as a GitHub Actions artifact:

```yaml
name: RISC OS CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Build on RISC OS
        id: robuild
        uses: gerph/riscos-build-service-action@v1
        with:
          architecture: aarch32
          timeout: 120
          output: /tmp/built
          capture-filename: build.log

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: riscos-output
          path: ${{ steps.robuild.outputs.output_filename }}

      - name: Upload build log
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: build-log
          path: ${{ steps.robuild.outputs.capture_filename }}
```

### Release workflow

When you have a `VersionNum` file in your repository, the action can automatically version
the output and upload it as a release artifact:

```yaml
      - name: Build and release on RISC OS
        id: robuild
        uses: gerph/riscos-build-service-action@v1
        with:
          architecture: aarch32
          timeout: 120
          release: yes
          # release-name: MyModule-1.00   # optional: override the auto-detected name

      # The action automatically uploads the versioned artifact when release: yes
      # The uploaded artifact will be named after the release (e.g. MyModule-1.00)
```

The release name is derived from the `Module_ComponentName` and `Module_MajorVersion` values
in the `VersionNum` file (e.g. `WimpTemplates-0.02`). If `VersionNum` is not found, the short
git SHA is used as the version instead.

## .robuild.yaml

The build service uses a `.robuild.yaml` file in your repository to determine what commands
to run. A minimal example:

```yaml
jobs:
  build:
    script:
      - riscos-amu
    artifacts:
      - path: aif32
```

Refer to the [RISC OS Build documentation](https://build.riscos.online/robuildyaml.html) for the
full specification.
