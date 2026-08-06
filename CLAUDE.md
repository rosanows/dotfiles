# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a personal dotfiles repository managed with [chezmoi](https://www.chezmoi.io/). It is not an application with a build/test suite — it's a collection of shell, editor, terminal, SSH, and Git configuration files that chezmoi turns into a live `$HOME` on the owner's machines (currently a Linux desktop `icecrown` and a macOS laptop `mba`).

There is no build step, package manifest, linter invocation, or test runner beyond the pre-commit hooks described below.

## chezmoi naming conventions

chezmoi maps repo filenames to target paths in `$HOME` using prefixes/suffixes. When adding or editing files, follow these existing conventions:

- `dot_foo` → installed as `~/.foo` (e.g. `dot_bashrc` → `~/.bashrc`, `dot_gitconfig` → `~/.gitconfig`).
- `dot_bashrc.d/*` → installed under `~/.bashrc.d/`; files are sourced in lexical order by `dot_bashrc`, hence the numeric prefixes (`02_direnv`, `10_aliases`, `30_fzf.tmpl`, ...). A `remove_` prefix (see `dot_bashrc.d/remove_01_simon_old`) tells chezmoi to delete that target file from `$HOME` if present.
- `private_foo` → installed with restrictive permissions (0600), used for anything under `dot_ssh/`.
- `executable_foo` → installed with the executable bit set (used in `bin/`, which maps to `~/bin/`).
- `*.tmpl` → processed as a Go template before being written, with chezmoi data such as `.chezmoi.hostname`, `.chezmoi.os`, and (on Linux) `.chezmoi.osRelease.id`/`.chezmoi.osRelease.versionID` available (see `dot_config/kitty/kitty.conf.tmpl`, `dot_ssh/private_authorized_keys.tmpl`, `dot_bashrc.d/30_fzf.tmpl` — the latter branches on `.chezmoi.osRelease` to work around an Ubuntu-24.04-specific fzf quirk). `and`/`or` short-circuit in chezmoi's Go template engine, so it's safe to check `.chezmoi.os` before dereferencing OS-specific fields in the same expression.
- `.chezmoitemplates/` → reusable template partials, included from other templates via `{{ template "name" . }}` (see the `authorized_keys/*` partials, each one a per-device public key list).
- `.chezmoiignore` → a templated list of paths to exclude from the target `$HOME` for the current machine. It's used here to skip installing the *other* host's kitty config (e.g. on `mba`, `icecrown.conf` is ignored, and vice versa) so only the config relevant to the current machine template renders.

## Host-specific configuration pattern

Several tools are configured per-machine rather than with runtime `if` branches:

- **kitty**: `dot_config/kitty/kitty.conf.tmpl` renders to `include common.conf` + `include <hostname>.conf`. Shared settings live in `common.conf`; machine-specific settings (Wayland/KDE bindings for `icecrown`, macOS bindings for `mba`) live in the per-host `.conf` files. `.chezmoiignore` prevents the non-matching host file from being installed.
- **SSH authorized_keys**: `dot_ssh/private_authorized_keys.tmpl` composes per-device key lists from `.chezmoitemplates/authorized_keys/{mba,icecrown,iphone,BREAK-GLASS}`.
- When adding a new machine, follow this pattern: add a new per-host template/partial rather than adding conditionals inside the shared file.

## Shell configuration architecture

`dot_bashrc` is the entrypoint; it sources `/etc/bashrc`, sets up `PATH` (`~/.local/bin`, `~/bin`), then bails out early for non-interactive shells, then sources every file in `~/.bashrc.d/` in order. Each `dot_bashrc.d/NN_name` file:

- Is guarded with `command -v <tool> >/dev/null` before configuring that tool, so the same files work whether or not a given tool (fzf, eza, ugrep, zoxide, starship, direnv) is installed on the current machine.
- Is single-purpose (one tool/concern per file) — prefer adding a new numbered file over growing an existing one.
- Numeric prefixes control sourcing order (`02_direnv` before `50_prompt`, etc.); `99_simon_after` runs last and handles PATH additions (Rust, asdf, Go), `$EDITOR`/`$VISUAL`, and SSH keychain setup.

macOS/Linux portability for Bash config is a stated goal (see commit "Make Bash configuration portable across Linux and macOS") — avoid tool/OS-specific assumptions in these files without a `command -v` or hostname guard.

## Other components

- `bin/` — standalone executable scripts installed to `~/bin` (`git-wtf` for branch/remote status, `movieme` for animated GIFs from video via ffmpeg, `rustup.sh` — a vendored/legacy Rust installer script). These are third-party or long-lived personal scripts; treat their internal style as already established rather than reformatting.
- `dot_gitconfig` — includes `~/.gitconfig_local` for machine-specific overrides; don't hardcode machine-specific values here.
- `dot_ssh/private_config` — includes `~/.ssh/config_local` first, then applies hardened global `Ciphers`/`KexAlgorithms`/`MACs`/`HostKeyAlgorithms` for all hosts.
- `dot_config/starship.toml` — Catppuccin-themed prompt; palette tables are intentionally kept at the end of the file (see in-file comment).
- `dot_gemrc` — forces `--user-install` for all `gem` operations (no system-wide installs).

## Pre-commit hooks

`.pre-commit-config.yaml` defines the checks that must pass on commits:

- **Hygiene** (pre-commit-hooks): `check-added-large-files`, `check-case-conflict`, `check-executables-have-shebangs`, `check-illegal-windows-names`, `check-toml`, `check-yaml`, `check-merge-conflict`, `detect-private-key`, `end-of-file-fixer`, `mixed-line-ending`, `trailing-whitespace`. (`check-json`/`check-xml` were dropped — no such files tracked.)
- **`yamllint`** — relaxed mode (`-d relaxed`).
- **`shellcheck`** — scoped to the maintained bash only (`dot_bashrc`, `dot_bashrc.d/*`), forced to the bash dialect (`-s bash`) since those files have no shebang/extension. `.tmpl` files and vendored `bin/` scripts are excluded. `SC1090`/`SC1091` (can't-follow-source) are disabled.
- **`shfmt`** — formats the same maintained bash to 4-space, case-indent (`-i 4 -ci`). Requires `shfmt` on PATH (`brew install shfmt`). Not applied to `bin/` (vendored) or `.tmpl`.
- **`gitleaks`** — secret scanning (stronger than `detect-private-key`).
- **`chezmoi-render`** (local) — runs `chezmoi apply --dry-run --source . --destination /tmp/...` to validate that all Go templates render without error, without touching `$HOME`. Requires `chezmoi` on PATH.

Run `pre-commit run --all-files` before committing if pre-commit is installed locally. Bump pinned hook versions with `pre-commit autoupdate`.

## Working in this repo

- Test changes by applying them with chezmoi (`chezmoi diff`, `chezmoi apply`) rather than assuming syntax is correct — templates in particular are easy to break silently.
- Never commit private key material; `dot_ssh/` only holds `private_config` and templated `authorized_keys` (public keys only) — no actual key files belong in this repo.
- Preserve the numbered-file/guarded-sourcing pattern in `dot_bashrc.d/` and the common/per-host split for `kitty` when extending either area.
