# mac-config

Reproducible configuration recipes for Apple Silicon macOS workstations.

## General preferences

- Prefer Linux-like terminal ergonomics on macOS.
- Use iTerm2 as the default terminal, with a persistent dark palette.
- Copy terminal text to the macOS clipboard immediately upon mouse selection.
- Treat the middle mouse button as `Command-V` system-wide.
- Visually distinguish focused terminal windows or panes by dimming inactive ones.
- Minimize routine prompts and manual UI work when a safe, reversible local action is clear.

## Software management

- Use [Homebrew](https://brew.sh/) as the primary package manager.
- On Apple Silicon, Homebrew is expected under `/opt/homebrew`.
- Use `brew install` for command-line formulae and `brew install --cask` for macOS applications.
- Check for an existing installation or configuration before adding a competing tool.
- Avoid broad package upgrades when installing or configuring one unrelated tool.
- Keep setup steps reproducible and document any required macOS privacy permissions.

## Safety and portability

- Prefer reversible local automation and preserve existing user configuration with backups.
- Confirm before destructive operations or external communication.
- Do not commit credentials, account identifiers, host-specific secrets, or private clipboard contents.
- Keep machine-specific values out of shared recipes unless they are essential and clearly labeled.

## Git workflow

- For lightweight personal repositories, commit and push directly to the default branch.
- Do not create pull requests unless explicitly requested.

## Recipes

- [Linux-like terminal selection and global middle-click paste](clipboard/macos-terminal-linux-clipboard-recipe.md)
- [Slack configuration recipes](slack/README.md)

The terminal recipe covers iTerm2 installation and defaults, Apple Terminal color conversion, selection-to-clipboard behavior, Hammerspoon middle-click paste, verification, and rollback.
