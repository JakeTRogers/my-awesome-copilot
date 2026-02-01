---
description: 'Shell scripting best practices and conventions for bash, sh, zsh, and other shells'
applyTo: '**/*.sh, **/*.bash, **/*.zsh'
---

# Shell Scripting Guidelines

Instructions for writing clean, safe, and maintainable shell scripts for bash, sh, zsh, and other shells.

## General Principles

- Generate code that is clean, simple, and concise
- Ensure scripts are easily readable and understandable
- Add comments where helpful for understanding how the script works
- Generate concise and simple echo outputs to provide execution status
- Avoid unnecessary echo output and excessive logging
- Use shellcheck for static analysis when available
- Assume scripts are for automation and testing rather than production systems unless specified otherwise
- Prefer safe expansions: double-quote variable references (`"$var"`), use `${var}` for clarity, and avoid `eval`
- Use modern Bash features (`[[ ]]`, `local`, arrays) when portability requirements allow; fall back to POSIX constructs only when needed
- Choose reliable parsers for structured data instead of ad-hoc text processing

## Error Handling & Safety

- Always enable `set -euo pipefail` to fail fast on errors, catch unset variables, and surface pipeline failures
- Validate all required parameters before execution
- Provide clear error messages with context
- Use `trap` to clean up temporary resources or handle unexpected exits when the script terminates
- Declare immutable values with `readonly` (or `declare -r`) to prevent accidental reassignment
- Use `mktemp` to create temporary files or directories safely and ensure they are removed in your cleanup handler

## Script Structure

- Start with a clear shebang: `#!/usr/bin/env bash` unless specified otherwise
- Include a header comment explaining the script's purpose
- Define default values for all variables at the top
- Use functions for reusable code blocks
- Create reusable functions instead of repeating similar blocks of code
- Keep the main execution flow clean and readable

## Working with JSON and YAML

- Prefer dedicated parsers (`jq` for JSON, `yq` for YAML—or `jq` on JSON converted via `yq`) over ad-hoc text processing with `grep`, `awk`, or shell string splitting
- When `jq`/`yq` are unavailable or not appropriate, choose the next most reliable parser available in your environment, and be explicit about how it should be used safely
- Validate that required fields exist and handle missing/invalid data paths explicitly (e.g., by checking `jq` exit status or using `// empty`)
- Quote jq/yq filters to prevent shell expansion and prefer `--raw-output` when you need plain strings
- Treat parser errors as fatal: combine with `set -euo pipefail` or test command success before using results
- Document parser dependencies at the top of the script and fail fast with a helpful message if `jq`/`yq` (or alternative tools) are required but not installed

```bash
#!/usr/bin/env bash

# ============================================================================
# Script Description Here
# ============================================================================

set -euo pipefail

cleanup() {
    # Remove temporary resources or perform other teardown steps as needed
    if [[ -n "${TEMP_DIR:-}" && -d "$TEMP_DIR" ]]; then
        rm -rf "$TEMP_DIR"
    fi
}

trap cleanup EXIT

# Default values
RESOURCE_GROUP=""
REQUIRED_PARAM=""
OPTIONAL_PARAM="default-value"
file=
verbose=0

readonly SCRIPT_NAME="$(basename "$0")"

TEMP_DIR=""

# Functions
usage() {
    echo "Usage: $SCRIPT_NAME [OPTIONS]"
    echo "Options:"
    echo "  -g, --resource-group   Resource group (required)"
    echo "  -h, --help            Show this help"
    exit 0
}

validate_requirements() {
    if [[ -z "$RESOURCE_GROUP" ]]; then
        die "Error: Resource group is required"
    fi
}

die() {
    printf '%s\n' "$1" >&2
    exit 1
}


main() {
    validate_requirements

    TEMP_DIR="$(mktemp -d)"
    if [[ ! -d "$TEMP_DIR" ]]; then
        die "Error: failed to create temporary directory"
    fi

    echo "============================================================================"
    echo "Script Execution Started"
    echo "============================================================================"

    # Main logic here

    echo "============================================================================"
    echo "Script Execution Completed"
    echo "============================================================================"
}

# Parse arguments
# POSIX

while :; do
    case $1 in
        -h|-\?|--help)
            usage                     # Display a usage synopsis.
            exit
            ;;
        -g|--resource-group)          # Takes an option argument; ensure it has been specified.
            if [ "$2" ]; then
                RESOURCE_GROUP=$2
                shift
            else
                die 'ERROR: "--resource-group" requires a non-empty option argument.'
            fi
            ;;
        --resource-group=?*)
            RESOURCE_GROUP=${1#*=}    # Delete everything up to "=" and assign the remainder.
            ;;
        --resource-group=)            # Handle the case of an empty --resource-group=
            die 'ERROR: "--resource-group" requires a non-empty option argument.'
            ;;
        -f|--file)                    # Takes an option argument; ensure it has been specified.
            if [ "$2" ]; then
                file=$2
                shift
            else
                die 'ERROR: "--file" requires a non-empty option argument.'
            fi
            ;;
        --file=?*)
            file=${1#*=}              # Delete everything up to "=" and assign the remainder.
            ;;
        --file=)                      # Handle the case of an empty --file=
            die 'ERROR: "--file" requires a non-empty option argument.'
            ;;
        -v|--verbose)
            verbose=$((verbose + 1))  # Each -v adds 1 to verbosity.
            ;;
        --)                           # End of all options.
            shift
            break
            ;;
        -?*)
            printf 'WARN: Unknown option (ignored): %s\n' "$1" >&2
            ;;
        *)                            # Default case: No more options, so break out of the loop.
            break
    esac

    shift
done

# if --file was provided, open it for writing, else duplicate stdout
if [ "$file" ]; then
    exec 3> "$file"
else
    exec 3>&1
fi

# If there are input files that follow the options, they will remain in the "$@" positional parameters.

# Execute main function
main "$@"
```
