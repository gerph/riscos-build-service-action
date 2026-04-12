# Creating a GitHub Composite Action for RISC OS Build Service

## Overview

This document describes the steps to create a custom GitHub composite action that wraps the
RISC OS Build Service client (`riscos-build-online`). A composite action lets us encapsulate
reusable CI logic so that multiple repositories can invoke the build service with consistent
parameters without duplicating bash steps.

## What is a Composite Action?

A composite action is a GitHub Action defined in an `action.yaml` (or `action.yml`) file that
uses `runs.using: "composite"` to execute a series of steps written in shell (typically bash).
Unlike Docker-based actions, composite actions run directly on the runner's host environment,
making them simpler to maintain and debug.

## Steps to Create a Custom Action

### 1. Create the action.yaml file

Place the file in `.github/actions/<action-name>/` for use within a single repository, or in
the root of a dedicated action repository.

### 2. Define metadata

At the top of the file, provide:

- **name**: A human-readable name for the action.
- **description**: A short summary of what the action does.
- **author** (optional): The maintainer's name.
- **branding** (optional): An `icon` and `color` for display on the GitHub Marketplace.

### 3. Declare inputs

Under `inputs:`, list each parameter the action accepts. For each input specify:

- **description**: What the input controls.
- **required**: `true` or `false`.
- **default**: A fallback value when the caller does not supply one.

Inputs are referenced inside the action using the expression
`${{ inputs.<input-id> }}`.

### 4. Declare outputs

Under `outputs:`, list any values the action will expose to the workflow. Each output needs a
`description`. The actual value must be wired to a step output using
`${{ steps.<step-id>.outputs.<key> }}`.

### 5. Define the runs block

This is the core of the action:

```yaml
runs:
  using: "composite"
  steps:
    - run: |
        echo "Your bash commands here"
      shell: bash
```

Every executable step **must** declare `shell: bash` explicitly.

### 6. Map inputs to environment variables

To safely use inputs in bash, map them through an `env:` block. GitHub converts input IDs to
uppercase with hyphens replaced by underscores:

```yaml
    - run: echo "$INPUT_MY_PARAM"
      shell: bash
      env:
        INPUT_MY_PARAM: ${{ inputs.my-param }}
```

### 7. Produce step outputs

Write outputs to the `$GITHUB_OUTPUT` file:

```yaml
    - id: my-step
      run: |
        echo "result=some_value" >> $GITHUB_OUTPUT
      shell: bash
```

Then wire them in the outputs section:

```yaml
outputs:
  result:
    description: "The result"
    value: ${{ steps.my-step.outputs.result }}
```

### 8. Add the action path to PATH (if using local scripts)

If your action ships with helper scripts, add the action directory to PATH:

```yaml
    - run: echo "$GITHUB_ACTION_PATH" >> $GITHUB_PATH
      shell: bash
```

### 9. Security considerations

- Treat all inputs as untrusted.
- Avoid interpolating user input directly into shell commands without validation.
- Use environment variables rather than inline string interpolation where possible.

## The RISC OS Build Service Action

### Inputs

| Input | Description | Default | Maps to tool flag |
|---|---|---|---|
| `architecture` | `'aarch32'` or `'aarch64'` | `aarch32` | `-A` |
| `timeout` | Timeout in seconds (max 600) | `10` | `-t` |
| `colour` | `'yes'`/`'no'` for ANSI colour | `yes` | `-a on`/`-a off` |
| `files` | Space-separated file patterns to send | `*` | file list in zip |
| `quiet` | `'no'`, `'yes'`, or `'silent'` | `no` | `-Q` / `-q` / none |
| `directory` | Working directory for collecting files | `.` | `cd` before zip |
| `output` | Output file location or prefix | `output` | `-o` |

### Outputs

| Output | Description |
|---|---|
| `capture_filename` | Filename created for output, or `""` if stdout |
| `output_filename` | Full filename of the created output, or `""` |
| `output_filetype` | Filetype extension of the created file (e.g. `feb`, `a91`) |

### Behaviour

The action performs these steps:

1. **Download the tool**: Fetch `riscos-build-online` from the GitHub releases at the version
   specified by `tool-version`.
2. **Prepare files**: Change to `directory`, then zip the specified `files` patterns plus
   `.robuild.yaml`.
3. **Run the build**: Invoke `riscos-build-online` with the mapped arguments.
4. **Process outputs**: Determine what output file was created (if any) and set the action
   outputs accordingly.

### Example usage in a workflow

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: <owner>/<action-repo>@v1
        with:
          architecture: aarch32
          timeout: 120
          output: /tmp/built
```

## Key conventions

- Always send `.robuild.yaml` along with the files, as it contains the RISC OS build
  instructions.
- The tool version is configurable so workflows can pin to a specific release.
- Output files use RISC OS filetype extensions (e.g. `,a91` for Zip, `,feb` for templates).
  The action detects which file was produced and reports it via outputs.
