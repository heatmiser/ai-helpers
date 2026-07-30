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

---

## Running the container

### Interactive session

```bash
podman run -it --rm \
    --pull newer \
    --userns=keep-id \
    -e GEMINI_API_KEY="$(cat ~/.config/gemini/api-key)" \
    -e GEMINI_CLI_TRUST_WORKSPACE=true \
    -e GH_TOKEN=$(gh auth token) \
    -e COLORTERM=truecolor \
    -v "$(pwd):$(pwd):z" \
    -w "$(pwd)" \
    quay.io/s4v0/gemini-ai-helpers:latest
```

Key flags:

| Flag | Purpose |
|------|---------|
| `--userns=keep-id` | Maps your UID into the container — files you create are owned by you |
| `GEMINI_CLI_TRUST_WORKSPACE=true` | Suppresses the workspace trust prompt on every launch |
| `GH_TOKEN` | Passes your GitHub auth into the container for `gh` CLI access |
| `-v "$(pwd):$(pwd):z"` | Mounts your current directory at the same path; `:z` applies the SELinux shared label |
| `-w "$(pwd)"` | Sets the working directory inside the container to match the host |

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
        -e GH_TOKEN=$(gh auth token) \
        -e COLORTERM=truecolor \
        -v "$(pwd):$(pwd):z" \
        -w "$(pwd)" \
        quay.io/s4v0/gemini-ai-helpers:latest "$@"
}
```

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
