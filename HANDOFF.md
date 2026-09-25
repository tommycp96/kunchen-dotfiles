# Handoff: adapting Kun's dotfiles into my own Nix setup

Read this first in a new session. It is the context for the work in progress on this branch.
The detailed implementation plan is in `PLAN.md`. This file says where things stand and what to do next.

## What this repo is

A fork of [kunchenguid/dotfiles](https://github.com/kunchenguid/dotfiles): a macOS setup managed with nix-darwin plus home-manager.
One `user` variable in `flake.nix`, one host label `mac`, `bootstrap.sh` for the first switch, `rebuild.sh` for every later change.
Real config files live under `home/` and `home.nix` symlinks them into place with `mkOutOfStoreSymlink`, so edits are live without a rebuild.
`homebrew.onActivation.cleanup = "zap"` is deliberate and stays (see `CLAUDE.md`): every Homebrew package must be declared or it gets removed on switch.

Owner of this fork: tommycp96. Remote: `https://github.com/tommycp96/kunchen-dotfiles`.

## Goal

Keep Kun's structure, swap the parts that are specific to his workflow for mine:

| Area | Kun | Me |
| --- | --- | --- |
| Terminal | WezTerm | Ghostty |
| Shell | zsh + starship | zsh + oh-my-zsh plugins, starship prompt |
| Agents | Pi + Claude Code + Codex + opencode | Claude Code + Codex CLI only |
| Editor | Neovim | VS Code daily, Neovim kept for quick terminal edits |
| Agent multiplexer | herdr | herdr, kept |
| Agent policy | Kun's `home/AGENTS.md` | rewritten for me |

## Decisions already made

- Ghostty replaces WezTerm. Cask plus a symlinked `home/.config/ghostty/config`. Do not enable the home-manager `programs.ghostty` module on macOS.
- oh-my-zsh is enabled for plugins only. `theme = ""`. Starship owns the prompt. Kun's starship block stays unchanged.
- herdr stays. Verify it under Ghostty on first use (Escape handling, mouse wheel, `ctrl+b` prefix).
- VS Code is the daily editor: add the cask, set `EDITOR` to `code --wait`, rely on Settings Sync rather than symlinking its settings file.
- Neovim stays installed with Kun's config for now. Shrinking it to a plain `init.lua` is optional later.
- Pi is removed entirely: `home/.pi/`, the four `.pi/agent/*` symlinks in `home.nix`, `tests/pi-calm.test.sh`, the Pi gitignore lines, and both README sections.
- Codex CLI is added as the `codex` cask with a single symlinked `home/.codex/config.toml`. Never symlink the whole `~/.claude` or `~/.codex` directories, they hold auth and sessions.

## Still open (defaults if not answered)

- Keep the `cc` and `co` aliases that run Claude and Codex with permissions skipped. Default: keep.
- Rename the GitHub repo to `dotfiles`. Default: leave it, the flake does not depend on the name.
- Neovim: keep Kun's plugin config or shrink it. Default: keep.

## What has been done on this branch

Branch: `claude/personal-nix-setup-plan-nftu29`, based on `main`. Three commits so far, all documentation:

1. `PLAN.md` added: seven phases from Mac inventory through first switch and verification.
2. Phase 5 rewritten: herdr kept, VS Code added, Neovim demoted.
3. Starship recorded as the prompt decision.

No config files have been changed yet. `flake.nix`, `configuration.nix`, `home.nix`, and everything under `home/` are still Kun's originals.

## Prompt audit of `home/AGENTS.md` (proposed, not applied)

`home/AGENTS.md` is installed as `~/.claude/CLAUDE.md` and `~/.codex/AGENTS.md`. An audit against the current Claude model found these edits worth making when the file is rewritten in Phase 4:

- Replace the "never add agent as co-author" rule with config. Claude Code: `"attribution": { "commit": "", "pr": "" }` in `home/.claude/settings.json`. Codex: `commit_attribution = ""` in `home/.codex/config.toml`. The harness system prompt outranks a CLAUDE.md line, so prose alone does not work.
- Remove the "dynamic workflows / ultra code" approval rule. The harness gates that feature itself, and suppressing delegation costs the current model one of its strengths.
- Rewrite the two "fix things you notice along the way" rules without the `try to` hedge and add "say what you changed" so unrequested fixes are visible in the report.
- Remove the `.no-mistakes/` rule from `AGENTS.md`, `CLAUDE.md`, and `.gitignore` unless I run that validation pipeline. I do not.
- Trim the "direct path" rule to its principle, dropping the list of things not to build.
- Add a short communication line: say what you are about to do, brief updates on long runs, lead with the outcome, no mannered prose.
- Keep: em dash rule, no editing generated files, quality over development cost, reproduce bugs end to end first.

## Next steps, in order

1. Phase 0 on the Mac: `brew list --formula`, `brew list --cask`, `brew tap`. Decide what to declare before the first switch, because zap removes the rest.
2. Phase 1: rewrite `README.md`; delete or rewrite `CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md`, `.github/workflows/close-prs.yml`, and the issue templates, which all say "Kun's personal setup".
3. Phase 2: Ghostty. Phase 3: oh-my-zsh. Phase 4: agents, including the AGENTS.md rewrite above. Phase 5: VS Code and EDITOR. Phase 6: packages, macOS defaults, git identity.
4. Validate on a Mac with Nix before applying: `nix flake check --no-build` and `nix build .#darwinConfigurations.mac.system --dry-run`. These cannot run in a Linux cloud session.
5. Phase 7: `./bootstrap.sh`, then the verification checklist in `PLAN.md`.

One phase per commit. Each phase must leave the config buildable.

## How to resume

```sh
git fetch origin claude/personal-nix-setup-plan-nftu29
git checkout claude/personal-nix-setup-plan-nftu29
```

Then read `PLAN.md`, confirm the three open decisions or accept the defaults, and start at the first unfinished phase.
