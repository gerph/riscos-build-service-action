# RISC OS Build Service GitHub Action

A GitHub composite action that calls the [RISC OS Build Service](https://build.riscos.online/)
to build RISC OS projects without requiring a local RISC OS environment.

## What it does

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
| `tool-version` | Version of `riscos-build-online` to use | `0.07` |

### Outputs

| Output | Description |
|---|---|
| `output_filename` | Full path of the output file created by the build, or empty if none |
| `output_filetype` | RISC OS filetype extension of the output (e.g. `a91`, `feb`, `ff8`) |
| `capture_filename` | Filename created for captured console output, or empty |

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

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: riscos-output
          path: ${{ steps.robuild.outputs.output_filename }}
```

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
