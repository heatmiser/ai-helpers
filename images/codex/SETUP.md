# codex-ai-helpers Setup Guide

Step-by-step setup for RHEL/Fedora users. Codex CLI authenticates via a plain API
key — there is no browser OAuth flow or keyring daemon involved, making the container
setup considerably simpler than OAuth-based CLIs.

## Prerequisites

- `podman` installed and running (rootless is fine)
- An OpenAI account with a funded API key from [platform.openai.com](https://platform.openai.com)
- `gh` CLI authenticated (optional — only needed if you want GitHub context in sessions)

## Pull the image

```bash
podman pull quay.io/s4v0/codex-ai-helpers:latest
```

The image is `x86_64` and `aarch64`. No Red Hat subscription required — it is based on
UBI 10.

---

## Authentication

### Get an API key

1. Log in to [platform.openai.com](https://platform.openai.com)
2. Navigate to **API keys** → **Create new secret key**
3. Copy the key — it is shown only once

> **Cost:** Codex defaults to `codex-mini-latest` (OpenAI's task-optimised model).
> Pricing is per token — see the
> [OpenAI pricing page](https://openai.com/api/pricing/) for current rates. You can
> override the model at runtime with `codex --model gpt-4o` or by setting
> `OPENAI_MODEL` in your environment.

### Store the key on the host

```bash
mkdir -p ~/.config/openai
echo "sk-..." > ~/.config/openai/api-key
chmod 600 ~/.config/openai/api-key
```

The key never touches the container image — it is read from this file and injected at
runtime via environment variable.

---

## Running the container

### Shell function (add to `~/.bashrc`)

```bash
function codex-ai() {
    local tty_args=""
    [ -t 0 ] && tty_args="--tty"
    podman run -i ${tty_args} --rm \
        --pull newer \
        --userns=keep-id \
        -e OPENAI_API_KEY="$(cat ~/.config/openai/api-key)" \
        -e GH_TOKEN="$(gh auth token)" \
        -e TERM="${TERM:-xterm-256color}" \
        -e COLORTERM=truecolor \
        -v ~/.codex:/home/codex/.codex:z \
        -v "$(pwd):$(pwd):z" \
        -w "$(pwd)" \
        quay.io/s4v0/codex-ai-helpers:latest "$@"
}
```

Key flags:

| Flag | Purpose |
|------|---------|
| `--userns=keep-id` | Maps your UID into the container — files you create are owned by you |
| `OPENAI_API_KEY` | Injects your key from the host file; never stored in the image |
| `GH_TOKEN` | Passes your GitHub auth into the container for `gh` CLI access |
| `-v ~/.codex:...` | Persists conversation history and reads `~/.codex/instructions.md` for custom system prompt |
| `-v "$(pwd):$(pwd):z"` | Mounts your current directory at the same path |
| `-w "$(pwd)"` | Sets the working directory inside the container to match the host |

The `[ -t 0 ]` check detects whether stdin is a terminal and conditionally adds `--tty`,
so the function works in both interactive and piped/scripted contexts.

After adding it, reload your shell:

```bash
source ~/.bashrc
```

Then invoke from any directory:

```bash
codex-ai
codex-ai -q "Summarise the last 5 commits in this repo"
```

---

## Optional: custom system instructions

Codex CLI reads `~/.codex/instructions.md` as a system-level prompt that is prepended
to every session. Place any standing context there — preferred coding style, project
conventions, or role definitions — and it will apply automatically when the
`~/.codex/` mount is in use.

```bash
mkdir -p ~/.codex
cat > ~/.codex/instructions.md << 'EOF'
You are a senior Red Hat platform engineer. Prefer dnf over apt, podman over docker,
and OpenShift-native patterns over vanilla Kubernetes where applicable.
EOF
```

The shell function above already mounts `~/.codex` into the container at
`/home/codex/.codex`, so the file is picked up without any additional flags.

---

## Model selection

### Discovering available models

Codex has a built-in command that lists the models your account can access:

```bash
# Live list — requires OPENAI_API_KEY to be set
codex-ai debug models

# Bundled list — compiled into the binary, no auth required
codex-ai debug models --bundled
```

Both commands output JSON. To extract just the model slugs:

```bash
codex-ai debug models --bundled | python3 -c "
import sys, json
for m in json.load(sys.stdin)['models']:
    if m.get('visibility') == 'list':
        print(f\"{m['slug']:30}  {m.get('display_name','')}: {m.get('description','')}\")"
```

### Available models (as of 0.153.4)

The models currently visible in the Codex model picker, ordered by priority:

| Model slug | Display name | Description |
|------------|-------------|-------------|
| `gpt-6-astra` | GPT-6-Astra | Most capable model for complex, demanding work |
| `gpt-5.6-sol` | GPT-5.6-Sol | Latest frontier agentic coding model |
| `gpt-5.6-terra` | GPT-5.6-Terra | Balanced agentic coding model for everyday work |
| `gpt-5.6-luna` | GPT-5.6-Luna | Fast and affordable agentic coding model |
| `gpt-5.5` | GPT-5.5 | Frontier model for complex coding and research |
| `gpt-5.2` | GPT-5.2 | Optimised for professional work and long-running agents |

The authoritative and up-to-date list is at
[developers.openai.com/api/docs/guides/latest-model](https://developers.openai.com/api/docs/guides/latest-model).
Model availability and slugs change over time — always run `codex debug models` to
confirm what your account can access.

### Persistent model selection via `config.toml`

Set the default model for all sessions by creating `~/.codex/config.toml` on the host
(the shell function above mounts `~/.codex` into the container):

```toml
model = "gpt-5.6-terra"

# Consider setting [mcp_servers] here!
```

This file is read at startup. The `model` key accepts any valid model slug.

### Per-session model override

Override the configured model for a single invocation:

```bash
codex-ai --model gpt-6-astra
codex-ai --model gpt-5.6-luna -q "Summarise the last 5 commits"
```

---

## Optional: building your own image

The image is built from
[heatmiser/ee-builds](https://github.com/heatmiser/ee-builds) via GitHub Actions.
Source files:

```
ee-builds/
├── codex-ai-helpers/
│   ├── Containerfile
│   ├── codex-entrypoint.sh
│   ├── image.yaml
│   └── README.md
└── .github/workflows/
    ├── generate_image_matrix.py
    ├── pr-image-build.yml
    └── push-image-build.yml
```

Fork `heatmiser/ee-builds`, configure your own quay.io push target in the workflow
secrets (`QUAY_USERNAME`, `QUAY_PASSWORD`), and the existing CI will build and publish
on merge to main.

Override the codex version at build time:

```bash
podman build \
  --build-arg CODEX_VERSION=0.153.4 \
  --build-arg AI_HELPERS_REF=main \
  --tag codex-ai-helpers:dev \
  ee-builds/codex-ai-helpers/
```
