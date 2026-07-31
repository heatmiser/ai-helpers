# gemini-ai-helpers Setup Guide

Step-by-step setup for RHEL/Fedora users. This guide documents the full authentication
journey — including the dead ends — so you do not repeat them.

## Prerequisites

- `podman` installed and running (rootless is fine)
- A Google account with access to [Google Cloud Console](https://console.cloud.google.com)
- `gh` CLI authenticated (optional — only needed if you want GitHub context in sessions)

## Pull the image

```bash
podman pull quay.io/s4v0/gemini-ai-helpers:latest
```

The image is `x86_64` and `aarch64`. No Red Hat subscription required — it is based on
UBI 10.

---

## Authentication

Gemini CLI supports Google account OAuth (`gemini auth`) and API key authentication. On
RHEL/Fedora inside a container, **most paths fail**. Here is what was tried and why each
outcome happened.

### What does not work

#### Google OAuth (`gemini auth`) inside the container

Running `gemini auth` in the container launches a loopback OAuth flow that requires a
keyring daemon to persist the token. UBI-based containers do not run a D-Bus session bus
or GNOME Keyring, so even though `libsecret` is present in the image, it cannot store the
token. The result is an infinite auth loop — the browser step completes but the token is
never saved.

`--network=host` does not help — the loopback server bind is a secondary issue; the
keyring failure is the blocker.

#### Bind-mounting `~/.gemini/` from the host

This works **only if** you have an existing, valid `~/.gemini/` on the host — meaning you
have already completed `gemini auth` successfully outside a container. On a fresh install
or when the host also lacks a keyring daemon, there is nothing to mount.

#### AI Studio API key (`aistudio.google.com/apikey`)

Blocked by org policy for Red Hat Workspace Google accounts. You will see an access
denied page. Personal Google accounts can use this path; Red Hat Workspace accounts
cannot.

---

### What works: GCP Console API key

Create an API key directly in Google Cloud Console. This requires a GCP project with
the Generative Language API enabled.

#### 1. Enable the Generative Language API

In [Google Cloud Console](https://console.cloud.google.com), select or create a project,
then navigate to **APIs & Services → Library** and enable **Generative Language API**.

> **Generative Language API vs Vertex AI API:** These are distinct APIs. Enabling the
> Vertex AI API (common in enterprise GCP environments) does not cover the Generative
> Language API. The `gemini-cli` API key path requires `generativelanguage.googleapis.com`
> specifically — enable it even if Vertex AI is already active on the project.

> **Cost:** `gemini-cli` defaults to **Auto** mode, which routes between
> `gemini-3.5-flash` (free tier available; data used to improve Google products) and
> `gemini-3.1-pro-preview` (no free tier — $2.00/$12.00 per 1M tokens) depending on task
> complexity. Rates shown are standard tier for personal GCP accounts. Enable billing on
> the GCP project before use; complex prompts will route to Pro. To stay on the free tier,
> pin the model explicitly: `gemini --model gemini-3.5-flash` or set
> `GEMINI_MODEL=gemini-3.5-flash` in your environment. See the
> [Gemini API pricing page](https://ai.google.dev/gemini-api/docs/pricing) for current rates.
>
> **Enterprise pricing:** Organizations with existing Google Cloud or Workspace agreements
> can negotiate custom per-token rates directly with Google Cloud sales; AI API usage is
> often bundled into Enterprise License Agreements (ELAs). While each deal is unique,
> negotiated rates are lower than the standard single-account rate card.

#### 2. Create an API key

Navigate to **APIs & Services → Credentials → Create Credentials → API key**.

> **Red Hat Workspace accounts:** org policy may require the key to be restricted to a
> service account. If prompted, bind it to the default Compute Engine service account
> (`PROJECT_NUMBER-compute@developer.gserviceaccount.com`) — this is sufficient for
> personal/dev use.

#### 3. Store the key

```bash
mkdir -p ~/.config/gemini
echo "AIzaSy..." > ~/.config/gemini/api-key
chmod 600 ~/.config/gemini/api-key
```

The key never touches the container image — it is injected at runtime via environment
variable read from this file.

#### 4. Configure `~/.gemini/settings.json`

Gemini CLI reads `~/.gemini/settings.json` for user preferences. When the container mounts
`~/.gemini`, this file controls the auth method. Without it (or with the default
`oauth-personal`), the container prompts for browser-based OAuth on every launch — which
fails in containerised environments because there is no keyring daemon.

Create `~/.gemini/settings.json` on the host:

```json
{
  "security": {
    "auth": {
      "selectedType": "gemini-api-key"
    }
  },
  "ui": {
    "useAlternateBuffer": false
  }
}
```

---

## Running the container

### Shell function (add to `~/.bashrc`)

```bash
function gemini-ai() {
    local tty_args=""
    [ -t 0 ] && tty_args="--tty"
    podman run -i ${tty_args} --rm \
        --pull newer \
        --userns=keep-id \
        -e GEMINI_API_KEY="$(cat ~/.config/gemini/api-key)" \
        -e GEMINI_CLI_TRUST_WORKSPACE=true \
        -e GEMINI_CLI_SYSTEM_SETTINGS_PATH=/etc/gemini/settings.json \
        -e GH_TOKEN=$(gh auth token) \
        -e TERM="${TERM:-xterm-256color}" \
        -e COLORTERM=truecolor \
        -v ~/.gemini:/home/gemini/.gemini:z \
        -v ~/gemini-cli-cntr-includes/etc-gemini-settings.json:/etc/gemini/settings.json:ro,z \
        -v "$(pwd):$(pwd):z" \
        -w "$(pwd)" \
        quay.io/s4v0/gemini-ai-helpers:latest "$@"
}
```

Key flags:

| Flag | Purpose |
|------|---------|
| `--userns=keep-id` | Maps your UID into the container — files you create are owned by you |
| `GEMINI_CLI_TRUST_WORKSPACE=true` | Suppresses the workspace trust prompt on every launch |
| `GEMINI_CLI_SYSTEM_SETTINGS_PATH` | Points gemini-cli to the system-level settings override file |
| `GH_TOKEN` | Passes your GitHub auth into the container for `gh` CLI access |
| `-v ~/.gemini:...` | Persists conversation history and reads `~/.gemini/settings.json` for auth type |
| `-v ...etc-gemini-settings.json:...` | Injects the model routing override (see below); `:ro,z` = read-only + SELinux label |
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
gemini-ai
gemini-ai -p "Summarise the last 5 commits in this repo"
```

---

## Optional: model routing override

Gemini CLI's **Auto** mode uses a numerical classifier (running on `gemini-3.1-flash-lite`)
to score each prompt and route it to either a flash or pro model. The bundled
`etc-gemini-settings.json` in this directory overrides the flash target to
`gemini-3.6-flash` (the latest GA flash model as of mid-2025) without rebuilding the
container image.

### Setup

Create the includes directory and copy the settings file:

```bash
mkdir -p ~/gemini-cli-cntr-includes
cp /path/to/ai-helpers/images/gemini/etc-gemini-settings.json \
   ~/gemini-cli-cntr-includes/
```

The shell function above already includes the two flags that activate it:

```
-e GEMINI_CLI_SYSTEM_SETTINGS_PATH=/etc/gemini/settings.json
-v ~/gemini-cli-cntr-includes/etc-gemini-settings.json:/etc/gemini/settings.json:ro,z
```

### How it works

`GEMINI_CLI_SYSTEM_SETTINGS_PATH` points to a system-level settings file that is merged
with the highest priority (above user and workspace settings). The file enables
`experimental.dynamicModelConfiguration`, which activates a settings-driven model
resolution path. It overrides two distinct model resolution paths:

**`classifierIdResolutions.flash`** — controls the main chat flash tier (Auto mode).
Setting `"default": "gemini-3.6-flash"` with `"contexts": []` redirects all
flash-tier chat to `gemini-3.6-flash`. The empty `contexts` array is required
to prevent the built-in conditional (`useGemini3_5Flash: true`) from firing and
returning `gemini-3.5-flash` before the default is evaluated.

**`customAliases`** — controls subagent model selection. Subagents (web-search,
web-fetch, loop-detection, etc.) resolve their model via the alias inheritance
chain, not `classifierIdResolutions`. Overriding `gemini-3-flash-base` in
`customAliases` redirects all seven subagent aliases that extend it. The
`agent-history-provider-summarizer` alias is also overridden directly, as it
hard-codes `gemini-3-flash-preview` without inheriting from `gemini-3-flash-base`.

### Verified model usage (Auto mode with override active)

```
gemini-3.1-flash-lite   utility_router     complexity classifier (unchanged)
gemini-3.1-flash-lite   utility_summarizer
gemini-3.6-flash        main               flash-tier tasks
gemini-3.6-flash        utility_tool       subagents (web-search, web-fetch, loop-detection, etc.)
```

`gemini-3-flash-preview` no longer appears anywhere in the session summary.

---

## Optional: building your own image

The image is built from
[heatmiser/ee-builds](https://github.com/heatmiser/ee-builds) via GitHub Actions.
Source files:

```
ee-builds/
├── gemini-ai-helpers/
│   ├── Containerfile
│   ├── gemini-entrypoint.sh
│   └── README.md
└── .github/workflows/
    ├── generate_image_matrix.py
    ├── pr-image-build.yml
    └── push-image-build.yml
```

Fork `heatmiser/ee-builds`, configure your own quay.io push target in the workflow
secrets, and the existing CI will build and publish on merge to main.
