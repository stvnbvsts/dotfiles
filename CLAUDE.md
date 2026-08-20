# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A [chezmoi](https://www.chezmoi.io/) source directory — the dotfiles/machine-setup repo for `stvnbvsts@gmail.com`'s Mac(s). This directory (`~/.local/share/chezmoi`) is **not** applied directly; chezmoi renders it into the destination directory (`$HOME`) via naming conventions and templating. See `README.md` for the full handbook (bootstrap steps, chezmoi command cheatsheet, naming conventions, templating, scripts).

## Commands

```sh
chezmoi diff                    # preview pending changes before writing anything
chezmoi apply -v                # write pending changes to $HOME (verbose)
chezmoi status                  # summary of what's pending
chezmoi execute-template < f    # render a single .tmpl file to check its output without applying
```

There is no build/lint/test suite — this is a dotfiles repo, not an application. "Testing" a change means rendering it with `chezmoi execute-template` or `chezmoi diff`/`chezmoi apply -v` and inspecting the result.

**Important:** `chezmoi apply` runs shell scripts (Homebrew installs, etc.) and can prompt interactively (e.g. "file has changed since chezmoi last wrote it — overwrite?", or a `sudo` password prompt). These prompts require a real TTY — running `apply` in a non-interactive/background context will hang or error (`could not open a new TTY`). Run `apply` in the foreground when a script might need input, and prefer surfacing the interactive prompt to the user rather than reaching for `--force`.

**The user runs `chezmoi apply`/`chezmoi add` (and any other command that installs packages or writes dotfiles to `$HOME`) themselves, in their own terminal.** Edit source files, then hand back the command to run rather than running it yourself. `chezmoi diff`, `chezmoi status`, and `chezmoi execute-template` (read-only preview commands) are fine to run directly.

## Architecture: source → destination mapping

chezmoi encodes destination path and file behavior in the *source* filename:

| Source | Destination | Notes |
|---|---|---|
| `dot_config/ghostty/config.ghostty` | `~/.config/ghostty/config.ghostty` | `dot_` → `.`; `config.ghostty` is Ghostty's config filename since 1.2.3 (was `config`) |
| `dot_zshrc` | `~/.zshrc` | Oh My Zsh config; expects `~/.oh-my-zsh` to exist |
| `Brewfile` | *(not applied — see `.chezmoiignore`)* | consumed by the install script, not written to `$HOME` |
| `run_onchange_before_install-packages.sh.tmpl` | *(script, not a file)* | see below |

`.chezmoiignore` lists source paths that should **not** be applied to `$HOME` as literal files — currently just `Brewfile`, which is machine-setup input, not a dotfile.

### Package installation flow (`run_onchange_before_install-packages.sh.tmpl`)

- **`run_`** — this is a script, executed during `chezmoi apply`, not written to disk as a dotfile.
- **`onchange_`** — chezmoi hashes the *rendered* script content and only reruns it when that hash changes. The Brewfile's contents are pulled into the hash via `{{ include "Brewfile" | sha256sum }}` in a comment, so editing `Brewfile` is what actually triggers a rerun (the script body itself rarely changes).
- **`before_`** — this script runs before chezmoi writes any managed files, sorted alongside them by target path. This ordering is intentional: if a package's install process can clobber a dotfile chezmoi is about to write (e.g. Oh My Zsh's installer overwrites `~/.zshrc` with its own default on first run), the package must be installed *first* so chezmoi's own file write happens last and wins.
- The script self-bootstraps Homebrew if missing (arch-aware: `{{ .chezmoi.arch }}` picks `/opt/homebrew` vs `/usr/local`), then runs `brew bundle --file="{{ .chezmoi.sourceDir }}/Brewfile"`. `brew bundle` is idempotent — already-installed casks/formulae are skipped, not reinstalled or errored on.

### Adding a new package

Edit `Brewfile` (add a `cask "..."` or `brew "..."` line) — the onchange hash picks up the change automatically on the next `chezmoi apply`. No need to touch the script itself unless the install *logic* changes.

### Adding a new dotfile

Use `chezmoi add <path>` rather than hand-authoring source files, so naming/attributes (`dot_`, `private_`, `executable_`, etc.) are applied correctly. If the new tool's installer can overwrite the dotfile on first run (like Oh My Zsh does), make sure its install step is a `run_before_` script, not `run_after_` or unprefixed.
