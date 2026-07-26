# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A [pi](https://pi.dev) extension that sandboxes LLM coding agents inside a [Gondolin](https://github.com/earendil-works/gondolin) micro-VM. The extension routes all of pi's built-in tools through an Alpine VM, so the LLM only sees and can modify files within the sandbox.

The extension is TypeScript loaded directly by pi (no build step). The `package.json` build/check/clean scripts are stubs.

**Based on:** the [Gondolin extension example](https://github.com/earendil-works/pi) in the official pi repository (`packages/coding-agent/examples/extensions/gondolin`).

## Commands

```sh
# Install dependencies
npm install --ignore-scripts

# Launch pi with the extension (from the project directory for development)
pi -e .                            # interactive session
pi -e . --model opus               # pass pi options
pi -e . -m "hello"                 # one-shot message
```

## Architecture

### `index.ts` — the extension

A pi extension (declared via `"pi": { "extensions": ["./index.ts"] }` in `package.json`). Dependencies:
- `@earendil-works/gondolin` — VM creation, filesystem operations, `ShadowProvider`
- `@earendil-works/pi-coding-agent` — tool factories and type definitions
- `yaml` — parsing `pi-gondolin-mount-config.yml`

**VM lifecycle:**
- Created lazily on first `session_start` via `VM.create()`, binding CWD as a `RealFSProvider` mount at `/workspace`
- The CWD provider is wrapped in a `ShadowProvider` that masks `pi-gondolin-mount-config.yml` from the VM (ENOENT on reads, omitted from directory listings)
- Reused across all tool calls (singleton via `ensureVm()`)
- Destroyed on `session_shutdown`

**Path translation (`toGuestPath`):**
- Relative paths → `/workspace/<relative>`
- Absolute paths inside the project → `/workspace/<relative>` (writes through to host)
- Absolute paths matching a configured additional mount → translated to that mount's guest path
- Other absolute paths → resolved against `/` (isolated within VM)

**Tool routing pattern:** Each native pi tool is registered with an override `execute` that:
1. Ensures the VM is running
2. Creates a new tool instance pointed at `GUEST_WORKSPACE` with VM-backed operations
3. Delegates to the guest-based tool

The ops factory functions (`createGondolinReadOps`, `createGondolinWriteOps`, etc.) translate paths and delegate to `vm.fs.*` or `vm.exec()`.

**Events:**
- `session_start` — warms up the VM
- `session_shutdown` — tears down the VM
- `user_bash` — intercepts user-initiated bash to run inside the VM
- `before_agent_start` — rewrites the system prompt to show `/workspace` as CWD
- Registered command `/gondolin` — shows VM status

### `pi-gondolin-mount-config.yml`

Additional directory mounts in docker-compose volume syntax. Read from CWD (host filesystem) before VM creation. Masked from the VM via `ShadowProvider` — the LLM cannot see or access it.

```yaml
mounts:
  - /home/user/data:/mnt/data       # read-write
  - /home/user/readonly:/mnt/ro:ro  # read-only
```

### `sandbox-claude-template.md`

Reference template for the `CLAUDE.md` that end-users should place in their sandboxed project directories. Written from the VM's perspective (`/workspace`, Alpine Linux, ephemeral filesystem).

# Prompting Workflow

1. Use your tools to read relevant files, check dependencies, and map out the current architecture. Do not guess.
2. If any part of a request is ambiguous or underspecified, you MUST ask targeted clarifying questions before taking action.
3. Write the plan to `~/.claude/plans/` (using EnterPlanMode). The plan file is automatically opened in Neovim for review via a file-system watcher. **Stop and wait for my approval before modifying any files.**
4. Once approved, execute the code changes exactly as outlined in the plan.
