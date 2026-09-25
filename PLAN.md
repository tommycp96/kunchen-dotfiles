# Plan: turn Kun's dotfiles into my own setup

Goal: keep the parts of Kun's nix-darwin + home-manager design that work (one `user` variable, one host label, edit-in-place symlinks from `home/`, Homebrew `zap` cleanup, `bootstrap.sh` then `rebuild.sh`) and swap the pieces that are specific to Kun's workflow for mine.

What changes, in one table:

| Area | Kun | Me |
| --- | --- | --- |
| Terminal | WezTerm (cask + `home/.config/wezterm`) | Ghostty (cask + `home/.config/ghostty/config`) |
| Shell | zsh + starship, no framework | zsh + oh-my-zsh plugins, starship prompt |
| Agent harness | Pi (themes, extensions, pinned packages) + Claude + Codex + opencode | Claude Code + Codex CLI only |
| Agent policy | `home/AGENTS.md` written by Kun | rewritten for me |
| Agent multiplexer | herdr | keep |
| Editor | Neovim as the daily editor | VS Code daily, Neovim only for quick terminal edits |
| Repo identity | kunchenguid/dotfiles, no-PR policy, Kun's name in CI and templates | my fork, my README, my policy |

Everything else (flake wiring, macOS defaults, CLI packages, Hack Nerd Font) stays until I have a reason to change it.

## Ground rules

- One phase per commit or PR, in the order below. Each phase leaves the config buildable.
- Validate before applying: `nix flake check --no-build` and `nix build .#darwinConfigurations.mac.system --dry-run`. Both need a Mac with Nix; they cannot run in a Linux CI box.
- Do not touch `homebrew.onActivation.cleanup = "zap"`. See `CLAUDE.md`.
- Keep the edit-in-place model: real files live under `home/`, `home.nix` only symlinks them with `mkOutOfStoreSymlink`. Do not mix in `programs.<x>.settings` for the same tool.
- Never symlink a whole agent state directory (`~/.claude`, `~/.codex`). Link individual authored files only. Both directories hold auth, sessions, and caches.

## Phase 0: inventory the current Mac (before any switch)

`zap` uninstalls every Homebrew formula and cask not declared in `configuration.nix` on the first switch. Capture what is installed now so nothing is lost:

```sh
brew list --formula
brew list --cask
brew tap
ls ~/Applications /Applications
```

Decide per item: declare it in `brews`/`casks`, or accept that it goes. Also note the macOS short username (`whoami`); `bootstrap.sh` rewrites the `user` line in `flake.nix` from it.

## Phase 1: make the repo mine

- `README.md`: rewrite the intro, clone URL (`tommycp96/kunchen-dotfiles` or whatever the repo is renamed to), and the "What you get" list to match the table above. Drop the WezTerm, Pi, herdr, and opencode paragraphs.
- `CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md`, `.github/workflows/close-prs.yml`, `.github/ISSUE_TEMPLATE/*`: these say "Kun's personal setup". Either delete them or rewrite in my name. Recommendation: delete the auto-close workflow and PR template, keep a two-line CONTRIBUTING that says it is a personal fork.
- `CLAUDE.md` / `AGENTS.md` at the repo root: keep, they are project notes for agents working on this repo. Remove the `.no-mistakes/` bullet if I do not use that pipeline.
- `LICENSE` (MIT-0): keep; it permits this fork.
- Consider renaming the GitHub repo to `dotfiles`. Not required; the flake does not depend on the repo name.

## Phase 2: Ghostty replaces WezTerm

`configuration.nix`:

- `casks`: replace `"wezterm"` with `"ghostty"`.

`home.nix`:

- Replace the wezterm symlink with `home.file.".config/ghostty".source = mkOutOfStoreSymlink "${dotfiles}/home/.config/ghostty";`.
- Do not enable `programs.ghostty`. Its `package` defaults to `null` on Darwin (nixpkgs has no Darwin Ghostty), and enabling it would generate `~/.config/ghostty/config` and conflict with the symlink. The cask installs the app, the symlink supplies the config.

`home/.config/ghostty/config` (new file, first cut; confirm key names with `ghostty +show-config --default --docs` and theme names with `ghostty +list-themes`):

```
theme = rose-pine-moon
font-family = Hack Nerd Font
font-size = 15
background-opacity = 0.8
background-blur = 20
macos-titlebar-style = hidden
macos-option-as-alt = true
shell-integration = zsh
unfocused-split-opacity = 0.7
window-padding-x = 8
window-padding-y = 6
confirm-close-surface = false
```

Notes:

- Ghostty reloads config on `cmd+shift+,`; no rebuild needed for config edits, same as WezTerm.
- Kun's WezTerm dims unfocused windows. Ghostty only dims unfocused splits (`unfocused-split-opacity`). Accept that.
- Delete `home/.config/wezterm/`.

## Phase 3: zsh with oh-my-zsh

`home.nix`, inside `programs.zsh`:

```nix
oh-my-zsh = {
  enable = true;
  plugins = [ "git" "z" "fzf" "docker" ];   # trim to what I use
  theme = "";                               # empty on purpose: starship owns the prompt
};
```

Keep `autosuggestion.enable`, `syntaxHighlighting.enable`, and the `bindkey '^f' autosuggest-accept` line. Home Manager sources oh-my-zsh at order 800 in `.zshrc`, so `initContent` (default order 1000) runs after it and my bindings win.

Prompt: starship, decided. Keep the `programs.starship` block as is. `theme` stays empty so oh-my-zsh never sets its own prompt; oh-my-zsh is used only for its plugins. Starship's init is emitted after oh-my-zsh in `.zshrc`, so it wins regardless.

Aliases: keep `..`, `add`, `push`, `pull`, `m`. Keep or drop `cc` and `co` deliberately; they skip permission prompts. Remove any aliases that oh-my-zsh's `git` plugin already provides if they collide (`gp`, `gl`, etc. do not collide with Kun's names, so no conflict today).

Migration: if `~/.zshrc` exists on the Mac, Home Manager refuses to overwrite it. Move it aside first (`mv ~/.zshrc ~/.zshrc.pre-nix`) and port anything worth keeping into `home.nix`. Same for an existing `~/.oh-my-zsh` clone: Home Manager uses the nixpkgs `oh-my-zsh` package and sets `$ZSH` itself, so the manual clone can be deleted after the first successful switch.

## Phase 4: agents, Claude Code + Codex CLI only

Remove Pi entirely:

- `home.nix`: delete the four `.pi/agent/*` symlinks and the opencode `AGENTS.md` link.
- Delete `home/.pi/`, `tests/pi-calm.test.sh`, and the Pi lines in `.gitignore`. Keep `tests/lib.sh` only if a new test uses it; otherwise delete `tests/`.
- README: drop both Pi sections.

Claude Code (keep):

- `casks`: keep `"claude-code"`.
- Keep `home/.claude/settings.json` (theme + status line) and the `~/.claude/settings.json` symlink.
- Keep `~/.claude/CLAUDE.md` -> `home/AGENTS.md`.
- Optional later: `home/.claude/keybindings.json`, project-agnostic skills under `home/.claude/skills/`, each linked individually.

Codex CLI (add):

- `casks`: add `"codex"` (the Homebrew cask for the OpenAI Codex CLI; `brew info --cask codex` to confirm before the first switch).
- New `home/.codex/config.toml` with my defaults (model, `approval_policy`, sandbox, MCP servers). Link it as a single file: `home.file.".codex/config.toml".source = mkOutOfStoreSymlink "${dotfiles}/home/.codex/config.toml";`.
- Keep `~/.codex/AGENTS.md` -> `home/AGENTS.md`.

Rewrite `home/AGENTS.md` as my own global policy. Kun's rules that are worth keeping regardless: no em dashes, never auto-add the agent as co-author, never hand-edit generated files, reproduce bugs end to end before fixing.

Risk to note in README: both `claude-code` and `codex` are casks under `zap` cleanup. Removing either from `casks` and switching will run the cask's zap stanza, which can delete `~/.claude` or `~/.codex` state. Do not remove them casually.

## Phase 5: herdr, VS Code, Neovim

herdr (keep):

- Keep `"herdr"` in `brews`, the `~/.config/herdr` symlink, `home/.config/herdr/config.toml`, and the herdr lines in `.gitignore`.
- Kun tuned it against WezTerm (see the `herdr,wezterm` commits in the log about Escape and mouse capture). Verify under Ghostty on first use: Escape inside a pane, mouse wheel in alt-screen apps, and the `ctrl+b` prefix not clashing with a Ghostty keybind. Fix in `home/.config/ghostty/config` or `config.toml` if anything is off.

VS Code (add, daily editor):

- `casks`: add `"visual-studio-code"`.
- `home.nix`: change `home.sessionVariables.EDITOR` from `"nvim"` to `"code --wait"` so git and other tools open VS Code.
- Settings: use VS Code Settings Sync, do not symlink `~/Library/Application Support/Code/User/settings.json`. That file is rewritten by the app, extension state lives next to it, and Settings Sync already gives the cross-machine behaviour this repo is for.
- Optional later: `home/.vscode/extensions.txt` produced by `code --list-extensions`, restored by a one-line `code --install-extension` loop in `bootstrap.sh`. Only if Settings Sync turns out not to cover extensions well enough.

Neovim (demote, do not remove):

- Keep the `neovim` package in `home.packages` for quick edits over SSH and in herdr panes.
- Keep Kun's `home/.config/nvim` config as is for now. It is self-contained and bootstraps lazy.nvim on first launch. If the plugin set feels like maintenance for an editor I rarely open, replace it with a plain `init.lua` (colorscheme, line numbers, clipboard) and delete `lazy-lock.json` and `lua/plugins/`.
- Drop the Neovim bullet from the README's "What you get" list or reword it as a secondary editor.

## Phase 6: packages and macOS defaults

- `home.packages`: keep ripgrep, fd, fzf, jq, lazygit, neovim, nerd-fonts.hack. Add my own CLI tools here rather than in `brews` when nixpkgs has them.
- `brews` / `casks`: add everything from the Phase 0 inventory that I decided to keep.
- `system.defaults`: review each of Kun's choices (dark mode, fast key repeat, hidden menu bar, autohide dock, Finder list view, no desktop icons, tap to click). Flip any I do not want.
- `programs.git`: Kun leaves identity unset on purpose. Add `programs.git.enable = true` with my name and email so a fresh machine can commit immediately.

## Phase 7: first switch and verification

1. On the Mac: `git clone <my repo> && cd <repo> && ./bootstrap.sh`. Answer `y` when it offers to rewrite the `user` line.
2. Open Ghostty: theme, font, and blur applied; `echo $ZSH` points into the Nix store; `starship` (or the chosen theme) renders; `^f` accepts a suggestion.
3. `claude --version`, `codex --version`; `ls -l ~/.claude/CLAUDE.md ~/.codex/AGENTS.md ~/.codex/config.toml ~/.claude/settings.json` all resolve into `~/.dotfiles/home/`.
4. `brew list` shows only declared packages.
5. Commit anything `bootstrap.sh` changed (`flake.nix` user line).

After that, every change is edit, `./rebuild.sh`, commit.

## Open decisions

These change the work, so settle them before Phase 3:

- Keep the `cc` / `co` full-auto aliases.
- Rename the repo to `dotfiles` or leave it.
- Neovim: keep Kun's plugin config or shrink it to a plain `init.lua`.
