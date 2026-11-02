# dotfiles

My personal [chezmoi](https://www.chezmoi.io/) repo: dotfiles and machine setup, kept in sync across machines.

## Contents

- `dot_config/ghostty/config.ghostty` → `~/.config/ghostty/config.ghostty`

## 1. Bootstrap a new machine

Install chezmoi and apply this repo's dotfiles in one step:

```sh
sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply --no-pager stvnbvsts
```

This:
1. Downloads and installs the `chezmoi` binary (to `~/.local/bin` if you don't already have it).
2. Clones this repo into `~/.local/share/chezmoi` (the *source directory*).
3. Applies it, writing the managed files into your actual home directory (the *destination directory*, usually `~`).

If chezmoi is already installed, you can do the same in two steps:

```sh
chezmoi init stvnbvsts   # clone the repo, don't apply yet
chezmoi diff             # review what would change
chezmoi apply --no-pager # install packages and write the files
```

Using `git@github.com:...` instead of the GitHub username also works if you prefer SSH.

### Package installation

`run_onchange_before_install-packages.sh.tmpl` installs Homebrew if missing, runs `brew bundle --no-upgrade` using `Brewfile`, installs Oh My Zsh if missing, and selects Homebrew's zsh as the login shell. It runs before writing dotfiles whenever the rendered script or Brewfile changes. Cask installation and login-shell setup can request your macOS administrator password.

### Password prompt stuck at “Waiting for data…”

“Waiting for data... (^X or interrupt to abort)” is a message from the `less` pager. If it appears alongside `Password:`, the pager and an installer or `sudo` prompt may be competing for terminal input. Oh My Zsh defaults `PAGER` to `less`, so changing terminal apps does not necessarily avoid it.

Stop the stuck invocation with Ctrl-C (if still in `less`, press `q` to exit it), then run these commands separately in the same terminal:

```sh
sudo -v
chezmoi apply --no-pager -v
```

Password entry normally shows no characters. `--no-pager` leaves interactive authentication enabled while disabling the pager. Do not run chezmoi itself with `sudo`. If this still stalls, note the last installation step printed by the verbose command. Chezmoi's upstream tracker also documents a [pager/prompt deadlock](https://github.com/twpayne/chezmoi/issues/4874).

## 2. Chezmoi handbook

The core idea: this repo (the **source directory**, `~/.local/share/chezmoi`) holds *representations* of files that get rendered into your **home directory** (the **destination**). You never edit `~/.config/ghostty/config` directly and hope it matches the repo — you go through chezmoi so both stay in sync.

### The two directories

| | Path | What it is |
|---|---|---|
| Source | `~/.local/share/chezmoi` | This git repo. File names are prefixed/encoded (`dot_config` → `.config`). |
| Destination | `~` (your home dir) | The real files chezmoi manages. |

### Daily workflow

The golden rule: **edit through chezmoi, never edit the destination file directly and forget to sync it back.**

```sh
chezmoi edit ~/.config/ghostty/config   # opens the *source* file in $EDITOR
chezmoi diff                            # preview what would change on disk
chezmoi apply --no-pager                # install packages and write changes
```

If you *did* edit the real file directly (e.g. `~/.config/ghostty/config`) and want to pull that change back into the repo:

```sh
chezmoi re-add                # or: chezmoi add ~/.config/ghostty/config
```

Other everyday commands:

```sh
chezmoi cd                    # drop into a subshell inside the source dir (plain `cd`/git work here)
chezmoi status                # what's changed, source vs. destination
chezmoi diff                  # full diff of pending changes
chezmoi apply --no-pager -v    # apply, verbosely, with interactive prompts
chezmoi apply --dry-run -v     # see what *would* happen without writing anything
chezmoi update --no-pager      # git pull + apply — pick up changes made elsewhere
```

### Adding a new file to be managed

```sh
chezmoi add ~/.zshrc
```

This copies `~/.zshrc` into the source dir as `dot_zshrc`, tracked in this repo. Commit it like any other file:

```sh
chezmoi cd
git add dot_zshrc
git commit -m "Add zshrc"
git push
```

### Naming conventions (source ↔ target)

chezmoi encodes attributes in the filename so it doesn't need extra config for common cases:

| Source name | Destination |
|---|---|
| `dot_gitconfig` | `~/.gitconfig` |
| `dot_config/nvim/init.vim` | `~/.config/nvim/init.vim` |
| `private_dot_ssh/config` | `~/.ssh/config` (perms `600`) |
| `executable_bin_script` | `~/bin/script`, marked `+x` |
| `run_once_install.sh` | script run once, never again (unless removed from state) |
| `run_onchange_install.sh.tmpl` | script re-run whenever its rendered content changes |
| `symlink_dot_something` | creates a symlink instead of a regular file |

Full reference: `chezmoi source-path` / [chezmoi attributes docs](https://www.chezmoi.io/reference/target-types/).

### Templating (per-machine differences)

Any file suffixed `.tmpl` is run through Go templates before being written. Useful for work vs. personal machine differences, e.g. in `dot_gitconfig.tmpl`:

```
[user]
    email = {{ .email }}
```

Values like `.email` come from `~/.local/share/chezmoi/.chezmoi.toml.tmpl` (prompted on `chezmoi init`, or set as static data). Built-in variables like `{{ .chezmoi.os }}` and `{{ .chezmoi.hostname }}` let a single file branch per-machine:

```
{{ if eq .chezmoi.os "darwin" }}
# macOS-only config
{{ end }}
```

### Secrets

Never commit secrets in plaintext. chezmoi integrates with password managers/secret stores (1Password, Bitwarden, pass, macOS Keychain, etc.) via template functions, e.g.:

```
token = {{ (onepasswordRead "op://Personal/GitHub/token") }}
```

The secret is fetched at `apply` time and never stored in the repo.

### Scripts (`run_` files)

Scripts live alongside dotfiles in the source dir and run as part of `chezmoi apply`:

- `run_once_*` — runs exactly once per machine (tracked in chezmoi's internal state).
- `run_onchange_*` — reruns whenever the script's own content (or templated content) changes; this is the mechanism for "install packages when my package list changes."
- Prefix with `before_`/`after_` to control ordering relative to file writes, e.g. `run_before_...` / `run_after_...`.

This is where package installation (Homebrew bundle, Ansible, etc.) or macOS defaults (`defaults write ...`) belong once added.

### Useful cheatsheet

```sh
chezmoi init <repo>            # clone + set up, no apply
chezmoi init --apply <repo>    # clone + apply in one go
chezmoi edit <file>            # edit source file for a managed target
chezmoi add <file>             # start managing a new file
chezmoi re-add                 # pull manual edits back into source
chezmoi diff                   # preview pending changes
chezmoi apply                  # write pending changes to disk
chezmoi update                 # git pull + apply (sync from remote)
chezmoi cd                     # shell into the source dir
chezmoi doctor                 # sanity-check your setup
chezmoi managed                # list all managed files
chezmoi unmanaged               # list files chezmoi is NOT tracking (useful for finding what to `add`)
```
