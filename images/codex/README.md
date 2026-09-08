# codex-ai-helpers Container Image

A UBI 10-based container image that pairs OpenAI's Codex CLI with the same rich
tooling layer used by the [Claude ai-helpers image](../claude/) — giving Codex
agents a capable, Red Hat-compatible environment to operate in.

## Quick start

```bash
podman pull quay.io/YOUR_ORG/codex-ai-helpers:latest
```

Run an interactive session:

```bash
podman run -it --rm \
  --env OPENAI_API_KEY=your-key-here \
  quay.io/YOUR_ORG/codex-ai-helpers:latest
```

## What's included

| Tool | Version | Install method | Purpose |
|------|---------|----------------|---------|
| codex | 0.153.4 | npm (system-wide) | OpenAI Codex AI agent (model configurable at runtime) |
| jq | latest (UBI) | dnf | JSON processing |
| yq | 4.53.3 | binary + SHA256 | YAML processing |
| ripgrep (`rg`) | 15.2.0 | binary + SHA256 | Fast code search |
| jc | latest | uv pip | Converts CLI output to JSON |
| shellcheck | 0.11.0 | binary + SHA256 | Shell script linting |
| gh | 2.89.0 | binary + SHA256 | GitHub CLI |
| glab | 1.91.0 | binary + SHA256 | GitLab CLI |
| oc | stable | binary | OpenShift client |
| uv | latest | installer | Python package manager |
| Python 3 | system | dnf | Scripting runtime |
| Python packages | — | uv pip | pytest, requests, pyyaml, ruff, tox, tox-uv, jc |
| Node.js | system | dnf | Runtime for codex |

All binaries are verified with pinned SHA256 checksums at build time.
Supports `x86_64` and `aarch64`.

The full [ai-helpers](https://github.com/opendatahub-io/ai-helpers) skill and agent
library is cloned to `/opt/ai-helpers` at image build time and available on disk for
all sessions.

## Authentication

Codex CLI authenticates exclusively via `OPENAI_API_KEY`. There is no browser OAuth
flow — pass the key as an environment variable or bind-mount `~/.codex/` from a host
directory that already contains it.

### Option 1: Environment variable (`OPENAI_API_KEY`)

```bash
podman run -it --rm \
  --env OPENAI_API_KEY=sk-... \
  quay.io/YOUR_ORG/codex-ai-helpers:latest
```

To avoid putting the key in your shell history, read it from a file at run time:

```bash
podman run -it --rm \
  --env OPENAI_API_KEY="$(cat ~/.config/openai/api-key)" \
  quay.io/YOUR_ORG/codex-ai-helpers:latest
```

**Choose this method when:**
- Running in CI/CD pipelines (inject via a pipeline secret — Tekton, GitHub Actions,
  GitLab CI, Jenkins all handle this natively)
- Deploying to Kubernetes or OpenShift, where a `Secret` resource is the idiomatic
  way to deliver credentials to a container
- The container is ephemeral and short-lived — the key is injected per-run and never
  persists anywhere in the image or on the host
- You are scripting or automating non-interactive workloads and want a clean,
  stateless credential model

**Why not always use this?**
The key is visible in `podman inspect` on the running container and appears in the
container's `/proc/<pid>/environ`. On a shared host or in an environment with broad
container-inspect permissions, this is a meaningful exposure.

---

### Option 2: Bind-mount `~/.codex/`

Bind-mounting `~/.codex/` from the host lets the container pick up a pre-stored API
key and any custom `instructions.md` without passing credentials via environment
variables. The directory must already exist on the host.

```bash
podman run -it --rm \
  --env OPENAI_API_KEY="$(cat ~/.config/openai/api-key)" \
  --volume "${HOME}/.codex:/home/codex/.codex:z" \
  quay.io/YOUR_ORG/codex-ai-helpers:latest
```

> The `:z` flag relabels the mount for SELinux on RHEL/Fedora. The primary benefit
> of the mount is persisting conversation history and custom instructions across
> sessions — not credential storage (the key is still passed via env var).

**Choose this method when:**
- You want conversation history to persist across container runs
- You have a custom `instructions.md` on the host that you want to reuse across
  sessions without rebuilding the image
- You are doing interactive development on a workstation and want a consistent
  per-user configuration

**Why not always use this?**
It creates a coupling between the host filesystem and the container. In CI/CD or
Kubernetes, there is no persistent home directory to mount from, so this method does
not apply.

---

### Kubernetes / OpenShift

Use a `Secret` to deliver the API key as an environment variable:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openai-api-key
type: Opaque
stringData:
  OPENAI_API_KEY: "sk-..."
```

```yaml
env:
  - name: OPENAI_API_KEY
    valueFrom:
      secretKeyRef:
        name: openai-api-key
        key: OPENAI_API_KEY
```

## Running examples

Pass a one-shot prompt:

```bash
podman run --rm \
  --env OPENAI_API_KEY="$(cat ~/.config/openai/api-key)" \
  quay.io/YOUR_ORG/codex-ai-helpers:latest \
  -q "Summarise the key changes in the Linux 6.10 release notes"
```

Mount a local repository for analysis:

```bash
podman run -it --rm \
  --env OPENAI_API_KEY="$(cat ~/.config/openai/api-key)" \
  --volume "$(pwd):/workspace:ro,Z" \
  --workdir /workspace \
  quay.io/YOUR_ORG/codex-ai-helpers:latest
```

Use a persistent config directory and a local repo together:

```bash
podman run -it --rm \
  --env OPENAI_API_KEY="$(cat ~/.config/openai/api-key)" \
  --volume "${HOME}/.codex:/home/codex/.codex:z" \
  --volume "$(pwd):/workspace:ro,Z" \
  --workdir /workspace \
  quay.io/YOUR_ORG/codex-ai-helpers:latest
```

## Build information

The image is built and published from
[heatmiser/ee-builds](https://github.com/heatmiser/ee-builds) via GitHub Actions:

- **Pull requests** → `ghcr.io/heatmiser/codex-ai-helpers:pr-<number>-<sha>`
- **Merge to main** → `quay.io/YOUR_ORG/codex-ai-helpers:latest`

The base image is `registry.access.redhat.com/ubi10/ubi:10.2` (public, no Red Hat
subscription required).
