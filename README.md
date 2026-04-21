# safe

`safe` is a macOS `sandbox-exec` wrapper that runs coding agents with project-scoped permissions defined in `.safe.yml`.

## Install

Copy the script to your PATH

## Usage

```sh
safe claude    # runs claude --dangerously-skip-permissions in the sandbox
safe codex     # etc.
safe opencode  # etc.
```

## Config

Add a `.safe.yml` to your project

```yml
# Sandbox config for bin/safe. Shared with the team.
# Per-machine additions go in config/safe.local.yml (gitignored).
# Your local config can tighten (add denies) but cannot loosen the denies below.

# Network policy: open | outbound | localhost | none
network: open

# Named bundles of path grants defined in bin/safe.
# Available: devbox, git, gh, claude, codex, opencode
use:
  - devbox
  - git
  - gh
  - claude
  - codex
  - opencode

allow:
  read:
    # ruby Resolv in stdlib reads this on startup
    - /etc/resolv.conf
    # devbox
    - /nix
    # rubocop
    - ~/.cache/rubocop_cache
  write:
    - ~/.cache/rubocop_cache

# Raw SBPL escape hatch for anything the schema doesn't express.
# append: |
#   (deny file-read* (subpath "/Users/shared/secrets"))
```

Commit `.safe.yml`, and/or add a `.safe.local.yml` with additional `allow:` directives as needed.
