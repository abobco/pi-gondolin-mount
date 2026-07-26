 # pi-gondolin-mount

A [pi](https://pi.dev) extension that sandboxes LLM coding agents inside a [Gondolin](https://github.com/earendil-works/gondolin) micro-VM with **configurable additional directory mounts**. All of pi's built-in tools (Read, Write, Edit, Bash, etc.) are routed through an Alpine VM, so the LLM only sees and can modify files within the sandbox.

## Installation

### Global install (recommended)

```sh
pi install npm:pi-gondolin-mount
```

Or install directly from git:

```sh
pi install git:https://github.com/abobco/pi-gondolin-mount.git
```

## Usage

Once installed globally, just run `pi` normally:

```sh
pi                                  # interactive session
pi -m "list files"                  # one-shot message
pi --model opus                     # pass pi options
```


The current working directory is mounted at `/workspace` inside the VM. File changes under `/workspace` write through to the host; other guest filesystem changes are isolated to the VM and lost when the session ends.

## Additional mounts

Create a `pi-gondolin-mount-config.yml` in your project directory to grant the VM access to directories outside the project. This file follows a docker-compose like syntax for mapping host directories to the VM. `<host-dir>:<guest-dir>:<privileges>`. 

Example:
```yaml
mounts:
  - /home/user/data:/mnt/data        # read-write
  - /home/user/readonly:/mnt/ro:ro   # read-only
```

**Note: `pi-gondolin-mount-config.yml` is automatically masked from the VM**. This stops the sandboxed LLM from editing its own permissions

## Sandboxed CLAUDE.md

See [`sandbox-claude-template.md`](sandbox-claude-template.md) for a template system prompt written from the VM's perspective (`/workspace`, Alpine Linux, ephemeral filesystem).

### Local development

```sh
git clone https://github.com/abobco/pi-gondolin-mount.git
cd pi-gondolin-mount
npm install --ignore-scripts
```

Then load it with the `-e` flag:

```sh
pi -e /path/to/pi-gondolin-mount
```

**Requirements:**
- Node.js >= 23.6.0 (for `@earendil-works/gondolin`)
- QEMU (`sudo pacman -S qemu` on Arch, `sudo apt install qemu-utils qemu-system-x86` on Debian)
## Credit

This extension is derived from the [Gondolin extension example](https://github.com/earendil-works/pi) in the official pi repository (`packages/coding-agent/examples/extensions/gondolin`). It differs from that extension in that it allows additional directory mounts to be configured via the `pi-gondolin-mount-config.yml` file.

## Architecture

```
pi-gondolin-mount/
├── index.ts                    # pi extension entry point
├── package.json
├── pi-gondolin-mount-config.yml                  # additional mount config (masked from VM)
└── sandbox-claude-template.md  # reference for sandboxed CLAUDE.md
```

The extension intercepts all of pi's built-in tools and routes them through a Gondolin micro-VM:

| Tool | How it's routed |
|------|----------------|
| Read, Write, Edit | VM filesystem operations with path translation |
| Bash | `vm.exec()` with CWD translated to `/workspace` |
| Ls, Find, Grep | VM filesystem traversal with glob/pattern matching |

VM lifecycle is managed automatically: created lazily on first `session_start`, reused across all tool calls, and destroyed on `session_shutdown`.
