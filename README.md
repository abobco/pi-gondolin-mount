 # pi-gondolin-mount

A basic extension for [pi](https://pi.dev) that allows mounting multiple drives into a sandboxed   [Gondolin](https://github.com/earendil-works/gondolin) micro-VM, using a familiar docker-compose-like syntax

## Why?

I found myself working on a project where I needed to copy a large number of files from a slow network drive. I didn't like any of the existing options for sandboxing an agent to do this for me, so I decided to extend the [Gondolin extension example](https://github.com/earendil-works/pi) from the pi repository to meet my needs

## Installation

**Requirements:**
- Node.js >= 23.6.0 (for `@earendil-works/gondolin`)
- QEMU (`sudo pacman -S qemu` on Arch, `sudo apt install qemu-utils qemu-system-x86` on Debian)

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

## Custom images

By default the sandbox boots Gondolin's stock guest image. To give the agent extra language runtimes or tools (Rust, Go, extra system packages, …), build a custom image with `gondolin build` and point the extension at it. See [`custom-images.md`](custom-images.md) for the full build reference.

**Build requirements** (host): `lz4`, `cpio`, `e2fsprogs`. On Debian/Ubuntu: `sudo apt install lz4 cpio e2fsprogs`.

This repo ships [`rust-image-config.json`](rust-image-config.json) as a worked example — an `x86_64` Alpine image with `rustc`, `cargo`, and the common C/build deps (`gcc`, `musl-dev`, `git`, `openssl-dev`, `pkgconf`). Build and verify it:

```sh
npm install --ignore-scripts
npx gondolin build --config rust-image-config.json --output ./rust-assets
npx gondolin build --verify ./rust-assets

# sanity-check the toolchain inside the image
GONDOLIN_GUEST_DIR=./rust-assets npx gondolin exec -- sh -lc "rustc --version && cargo --version"
```

> Set `arch` in the config to match your host (`x86_64` or `aarch64`). Build outputs (`./*-assets/`) are gitignored.

**Rolling your own:** copy `rust-image-config.json`, keep the default `rootfsPackages` (the sandbox helpers and the extension's `bash` probe depend on `bash`/`openssh`/etc.), and add your packages. Search [pkgs.alpinelinux.org](https://pkgs.alpinelinux.org/packages) for package names.

### Using a custom image

Point the extension at the built asset directory. Two ways, config wins over the env var:

```yaml
# pi-gondolin-mount-config.yml — relative paths resolve against the project dir
image: ./rust-assets
```

```sh
# or, at launch time
GONDOLIN_GUEST_DIR=./rust-assets pi
```

Run `/gondolin` in a session to confirm which image is active. If the path isn't a valid build output (no `manifest.json`), the extension warns and falls back to the default image.

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

## Credit

This extension is derived from the [Gondolin extension example](https://github.com/earendil-works/pi) in the official pi repository (`packages/coding-agent/examples/extensions/gondolin`). It differs from that extension in that it allows additional directory mounts to be configured via the `pi-gondolin-mount-config.yml` file.

## Architecture

```
pi-gondolin-mount/
├── index.ts                    # pi extension entry point
├── package.json
├── pi-gondolin-mount-config.yml                  # additional mount + image config (masked from VM)
├── rust-image-config.json      # example custom image build config (Rust toolchain)
└── sandbox-claude-template.md  # reference for sandboxed CLAUDE.md
```

The extension intercepts all of pi's built-in tools and routes them through a Gondolin micro-VM:

| Tool | How it's routed |
|------|----------------|
| Read, Write, Edit | VM filesystem operations with path translation |
| Bash | `vm.exec()` with CWD translated to `/workspace` |
| Ls, Find, Grep | VM filesystem traversal with glob/pattern matching |

VM lifecycle is managed automatically: created lazily on first `session_start`, reused across all tool calls, and destroyed on `session_shutdown`.
