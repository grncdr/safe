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

# Optional: restrict the sandboxed agent to a single AWS profile.
# safe runs `aws configure export-credentials --profile <name>` on the host,
# caches the result under ~/.cache/safe/aws/<name>/, grants the sandbox read
# access to exactly that directory, and hard-denies ~/.aws. A background
# thread refreshes the cached credentials before they expire.
# Put this in .safe.local.yml if the profile name varies per machine.
aws:
  profile: my-profile

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

## Architecture

Single Ruby file (`safe`). Flow on invocation:

1. Walk up from CWD to find `.safe.yml` → that dir is `ROOT`.
2. If `devbox.json` exists at `ROOT`, re-exec under `devbox run`.
3. Merge `.safe.yml` with optional `.safe.local.yml` (local can only tighten).
4. Render an SBPL policy to `~/.cache/safe/policy-<sha>.sb` (cached; hash keyed on configs, script source, `ROOT`, `HOME`, worktree).
5. `Process.spawn('sandbox-exec', '-f', policy, '--', *agent_argv(ARGV))` and wait.

Policy structure (order matters — SBPL is last-match-wins):

- `BASELINE` — deny-default plus the minimum macOS paths/mach services a process needs to run. Includes RW on `/tmp` and `/var/folders`.
- Network mode from `NETWORK` (`open`/`outbound`/`localhost`/`none`).
- `use:` recipes from `RECIPES` (named bundles: `devbox`, `git`, `gh`, `claude`, `codex`, `opencode`).
- User `allow.read` / `allow.write` via `allow_rule` → `matcher` picks `:subpath` (trailing `/` or real dir), `:prefix` (trailing `*`), or `:literal`. Each allow emits ancestor-literal reads so `stat` up the tree works.
- `ROOT` RW. If `ROOT` is a git worktree, the main repo's `.git` dir is also RW.
- `HARD_DENIES` emitted last so they override everything above. Also user `deny.*` rules and the write-deny on `.safe.yml`/`.safe.local.yml`.

Known agents get permission-prompt bypass flags via `agent_argv` (`--dangerously-skip-permissions` etc.) — the sandbox is the real guardrail.

AWS profile gating (`aws.profile` in config):

- Creds cached at `~/.cache/safe/aws/<profile>/{credentials,config,meta.json}`; the sandbox gets read access to exactly that subpath (and `~/.aws` is hard-denied).
- `aws_ensure_fresh!` serializes refresh via `flock` on `<aws_dir>/lock` with double-check-under-lock so concurrent sessions sharing a profile don't stampede `aws configure export-credentials`.
- Background thread re-reads `meta.json` each iteration and sleeps until `AWS_REFRESH_LEEWAY` before expiry.
- `AWS_SHARED_CREDENTIALS_FILE` / `AWS_CONFIG_FILE` point the sandboxed child at the cache; all other `AWS_*` env vars are stripped.
