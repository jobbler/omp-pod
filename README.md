# omp-pod

Minimal secure podman sandbox for running [omp](https://github.com/can1357/oh-my-pi)
(oh-my-pi) inside a rootless container, isolated to a specific working directory.

## What it does

- Runs omp in a hardened rootless podman container (`--cap-drop=ALL`,
  `no-new-privileges`, `--userns=keep-id`, pids/memory/cpu limits)
- Mounts your current directory at `/home/agent/workspace` — files the agent
  writes appear on the host owned by you
- Shares your host `~/.omp` config (models, auth, sessions) — sessions created
  inside are visible to host omp and vice versa; resume with omp's own `/resume`
- Mounts `~/.config/gh` **read-only** when it exists, so `git push` / private
  HTTPS clones work inside without installing gh (a git credential helper reads
  the OAuth token from `hosts.yml`)
- Unrestricted network egress (model APIs, package installs)

## AI Disclosure

Created with assistance of Oh My Pi running the GLM 5.3 model.


## Install

```bash
curl -fsSL https://raw.githubusercontent.com/jobbler/omp-pod/main/omp-pod-install | bash
```

Or from a checkout:

```bash
./omp-pod-install            # clone-less: link + build
./omp-pod-install --no-build # link only
```

Requires: rootless podman (tested 5.8.7 on Fedora), `~/.omp` already set up
(run host omp once to configure models/auth).

## Updating the omp container

This updates the `omp` version running inside the container.

```bash
./omp-pod-build                     # default: newest omp release
OMP_VERSION=18.11.1 ./omp-pod-build # pin a specific release
```

Latest: https://github.com/can1357/oh-my-pi/releases

## Use

```bash
cd <project-dir>
omp-pod                 # interactive TUI
omp-pod "fix the bug"   # one-shot prompt (goes to the model, not a shell)
```

All arguments after `omp-pod` are forwarded verbatim to omp. The host `omp`
binary stays untouched for unsandboxed use; alias `omp='omp-pod'` in your
shell rc if you want the sandbox by default.

Containers are `--rm`: exiting ends them. There is no attach/resume lifecycle —
use omp's `/resume` against the shared `~/.omp`.

## Files

| File | Purpose |
|---|---|
| `omp-pod-containerfile` | ubuntu:24.04 + git, ripgrep, fd, uv, bun, omp prebuilt binary |
| `omp-pod-init` | entrypoint: sets up gh credential helper (if mounted), cds to workspace, execs omp |
| `omp-pod` | launcher: computes container name from dir, applies podman hardening flags |
| `omp-pod-build` | builds the image; `OMP_VERSION=<ver>` pins, `OMP_VERSION=latest` resolves newest |
| `omp-pod-install` | curl-able installer: clones, links `~/.local/bin/omp-pod`, builds image |

## Security model & tradeoffs

- **Boundary**: userns mapping (host you → container uid 1000 `agent`),
  all capabilities dropped, no-new-privileges, pid/memory/cpu caps.
  Host filesystem outside the mounts is invisible to the container.
- **gh token exposure (accepted)**: when `~/.config/gh` exists it is mounted
  read-only, but its plaintext OAuth token is readable by anything in the
  sandbox, and egress is unrestricted — a compromised agent could exfiltrate
  it. Revoke with `gh auth logout` if suspected.
- **Unrestricted egress (accepted)**: anything inside can talk to any host.
- No LSP servers and no browser/Chromium in the image: omp's LSP tool stays
  dormant, and the browser tool fails if invoked. Add the relevant RUN lines
  and rebuild if you need them.

## Double-launch guard

The container is named after the working directory (`omp-<dirname>`).
Running `omp-pod` twice in the same dir refuses with an error instead of
attaching (attaching a TUI container from a second terminal garbles both).
Exit the first session or `podman rm -f omp-<dirname>`.


## SELinux (Fedora)

Both mounts use `:z` so podman relabels content as needed; works with
SELinux enforcing.


