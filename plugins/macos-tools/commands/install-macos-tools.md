---
description: Install macOS command line tools for Claude Code productivity
allowed-tools: Bash, Read, Edit, Write
---

# Install macOS Tools Command

Install essential command line tools on macOS for optimal Claude Code usage.

## Step 1: Install Homebrew

Check if Homebrew is installed. If not, install it:
- Check with: `command -v brew`
- If missing, install: `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
- For Apple Silicon Macs (arm64), add Homebrew to your PATH:
  - **If you use zsh (default on modern macOS):**
    ```bash
    echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
    eval "$(/opt/homebrew/bin/brew shellenv)"
    ```
  - **If you use bash:**
    ```bash
    echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.bash_profile
    eval "$(/opt/homebrew/bin/brew shellenv)"
    ```
  - *(If you use a different shell, add the eval line to your shell's profile file.)*
- If already installed, update it: `brew update`

## Step 2: Install Essential CLI Tools

Install the following tools via Homebrew (they will be installed in parallel automatically):

```bash
brew install gh ripgrep fd eza bat parallel hyperfine tokei jq yq
```

**Tools being installed:**
- `gh` - GitHub CLI
- `ripgrep` (rg) - ultra-fast grep alternative
- `fd` - fast find alternative
- `eza` - modern ls replacement
- `bat` - cat with syntax highlighting
- `parallel` - execute jobs in parallel
- `hyperfine` - command-line benchmarking
- `tokei` - code statistics
- `jq` - JSON processor
- `yq` - YAML processor

## Step 3: Update Global CLAUDE.md with Tool Preferences

Find the user's global CLAUDE.md file (typically at `~/.claude/CLAUDE.md`).

If found, check if it already contains tool preference guidance (search for keywords like "ripgrep", "eza", "bat", or "Tool Preferences").

If tool preferences are NOT already present, append a "Tool Preferences" section that lists all the tools installed in Step 2 and explains when Claude should use them instead of native alternatives:

**Tool preferences to add:**
- Use `eza` instead of `ls` for directory listings
- Use `bat` instead of `cat` for viewing files
- Use `rg` (ripgrep) instead of `grep` for searching
- Use `fd` instead of `find` for finding files
- Use `jq` for JSON processing
- Use `yq` for YAML processing
- Include any other tools from the installation list with appropriate usage guidance

If `~/.claude/CLAUDE.md` doesn't exist, create the directory and file first, then add the tool preferences.

**Important:** Keep the tool list synchronized - if you modify the tools installed in Step 2, update the CLAUDE.md guidance accordingly.
