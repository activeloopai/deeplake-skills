---
name: deeplake-memory
description: SDK and CLI for Deeplake persistent memory — a cloud-backed filesystem for AI agents. Use when users want to mount, manage, or use persistent memory that survives across sessions, or run any `deeplake` CLI command.
---

# Deeplake Memory — Persistent Filesystem for AI Agents

Deeplake Memory is a FUSE-mounted folder backed by Deeplake Cloud. Everything written here survives across sessions and syncs in real-time to all connected AI agents (Claude Code, Cursor, Copilot, etc.).

## Memory Location

Your persistent memory is mounted at `~/.deeplake/memory/`.

- **Read/write files** with standard commands (`cat`, `ls`, `echo >`)
- **Sessions persist** — files are shared across all Claude Code sessions
- **Multi-agent** — other AI assistants see the same files in real-time
- **Cloud-backed** — data stored in Deeplake Cloud, accessible from any machine

## CLI Reference

### Getting Started

```bash
# First time setup (interactive)
deeplake init

# Login to Deeplake
deeplake login

# Mount all registered filesystems
deeplake mount --all

# Check what's running
deeplake list
```

### Commands

| Command | Description |
|---|---|
| `deeplake init [options]` | Initialize a new mount (interactive setup) |
| `deeplake mount <path\|--all>` | Mount a specific path or all registered mounts |
| `deeplake umount <path\|--all>` | Unmount a specific path or all mounts |
| `deeplake list` | Show all registered mounts and their status |
| `deeplake login` | Authenticate with Deeplake |
| `deeplake logout` | Clear stored credentials |
| `deeplake organization list` | List all organizations |
| `deeplake organization set <name\|id>` | Switch active organization |
| `deeplake workspace list` | List all workspaces |
| `deeplake workspace set <name\|id>` | Switch active workspace |
| `deeplake member invite <email\|user>` | Invite a user to your organization |
| `deeplake member remove <email\|user>` | Remove a user from your organization |
| `deeplake import <source-dir> <table>` | Bulk import a directory into a table |
| `deeplake version` | Show CLI version |
| `deeplake help` | Show help message |

### Aliases

| Alias | Expands to |
|---|---|
| `org` | `organization` |
| `ws` | `workspace` |
| `stop` | `umount` |
| `status` | `list` |
| `--version` | `version` |
| `-h`, `--help` | `help` |

### Init Options

```bash
deeplake init --org my-org --workspace prod --table my_project --path /path/to/mount
```

| Flag | Description | Default |
|---|---|---|
| `--org <name\|id>` | Organization name or ID | from credentials |
| `--workspace <name>` | Workspace name | `default` |
| `--table <name>` | Table name | `memory` |
| `--path <path>` | Mount path (skips interactive prompt) | — |
| `--hook` | Auto-install memory sync hook (skips prompt) | — |
| `--skill <local\|global>` | Install Deeplake skill for Claude Code (skips prompt) | — |
| `--auto` | Use all defaults, skip all prompts (implies --hook --skill global) | — |

### Member Invite Options

```bash
deeplake member invite user@example.com --role writer
```

| Flag | Description | Default |
|---|---|---|
| `--role <admin\|writer\|reader>` | Access level for the invited user | `writer` |

### Global Options

| Flag | Description |
|---|---|
| `-v`, `--verbose` | Enable verbose logging |

### Environment Variables

| Variable | Description | Default |
|---|---|---|
| `DEEPLAKE_API_URL` | Deeplake API URL | `https://api.deeplake.ai` |
| `DEEPLAKE_TOKEN` | API token (for managed service) | — |
| `DEEPLAKE_ORG_ID` | Organization ID | — |
| `DEEPLAKE_WORKSPACE_ID` | Workspace ID | — |
| `DEEPLAKE_MOUNT_PATH` | Default mount path | — |

## Examples

```bash
# First time setup (interactive)
deeplake init

# Fully automatic setup — all defaults, no prompts
deeplake init --auto

# Custom org and workspace
deeplake init --org my-org --workspace my-ws --table my_project

# Non-interactive with custom path + hooks + skill
deeplake init --org my-org --path ~/my-memory --hook --skill global

# Install skill only for this project
deeplake init --hook --skill local

# Switch organizations (re-points all memory mounts)
deeplake organization list
deeplake org set my-org

# Switch workspaces
deeplake workspace list
deeplake ws set production

# Invite a teammate
deeplake member invite user@example.com --role writer

# Revoke access
deeplake member remove user@example.com

# Unmount everything
deeplake umount --all

# Bulk import a directory (~400x faster than cp)
deeplake import ./my-data my_table
```
