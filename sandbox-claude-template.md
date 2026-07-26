# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment

**You are running inside a sandboxed [Gondolin](https://github.com/earendil-works/gondolin) micro-VM.** The host project directory is mounted at `/workspace` inside the VM. All tool operations (Read, Write, Edit, Bash, Ls, Find, Grep) execute within the VM against the guest filesystem.

- **Current working directory:** `/workspace`
- **Guest workspace root:** `/workspace` (writes through to the host project directory)
- **Paths:** Use `/workspace`-relative paths. Relative paths are resolved to `/workspace/<relative>`. Absolute paths outside `/workspace` are isolated within the VM and do not reach the host.
- **Environment:** Alpine Linux — use `apk` for package management if needed. The VM is ephemeral; installed packages and files outside `/workspace` (and configured mounts) are lost when the session ends.

### Additional mounts

If configured by the host, additional directories may be mounted at guest paths like `/mnt/data`. Some mounts may be read-only. Check the system prompt for details.

## Commands

All commands run inside the VM, targeting the guest filesystem:

```sh
ls /workspace
node --version
cat /workspace/CLAUDE.md
```

# Prompt Protocol

## Mandatory Planning Workflow

- **NEVER** modify, create, or delete any files before presenting a complete step-by-step implementation plan.
- For any task requiring more than a single file change or architectural choice, you must invoke `/plan` or outline your strategy in a Markdown file under `.agents/plans/` first.
- Wait for explicit user approval of the written plan before executing any mutating tools (`write`, `edit`, or destructive shell commands).

# Prompting Workflow: 

Whenever you are given a task, you must strictly follow this sequence:
1. Use your tools to read relevant files, check dependencies, and map out the current architecture. Do not guess.
2. If any part of a request is ambiguous or underspecified, you MUST ask targeted clarifying questions before taking action.
3. Write the plan to `/workspace/plans/`. **Stop and wait for my approval before modifying any files.**
4. Once approved, execute the code changes exactly as outlined in the plan.
