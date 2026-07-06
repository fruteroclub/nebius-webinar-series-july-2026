# How to Set Up the Local Nebius Builder Environment

This is the student-facing companion guide for Webinar 1.

The goal is to set up the builder environment used by the rest of the FDE
Trainer series. In this local pass, your laptop is the server and Docker runs
the sandbox. Later, we will move the same shape to Nebius Cloud.

Use this guide while recording from
[video-tutorial-script.md](video-tutorial-script.md). Use
[01-nebius-cloud-builder-environment.md](01-nebius-cloud-builder-environment.md)
only when you need raw implementation notes or troubleshooting detail.

## What You Will Have

By the end:

- You know what Nebius Token Factory does in this series.
- You have created a Token Factory API key and stored it outside the repo.
- Pi Coding Agent is configured to use Nebius Token Factory.
- NemoClaw/NemoHermes is installed locally.
- Hermes Agent is running in a Docker/OpenShell sandbox.
- The Hermes dashboard opens at `http://127.0.0.1:18789/`.
- The webinar repo is visible inside the sandbox at `/sandbox/workspace`.
- You know how to verify whether the agent is actually using tools.

## What Runs Where

```text
Local laptop
  Docker
    NemoClaw/OpenShell sandbox
      Hermes Agent
      /sandbox/workspace

Nebius Token Factory
  inference endpoint for Pi and Hermes
```

## Tool Roles

| Tool | Role in Webinar 1 |
| --- | --- |
| Pi Coding Agent | Coding agent for setup, debugging, and later project work. |
| Nebius Token Factory | Model inference provider. |
| Docker | Local server runtime for the first pass. |
| NemoClaw/NemoHermes | Installs and manages Hermes Agent inside a sandbox. |
| Hermes Agent | Agent harness we will later turn into the FDE Trainer. |
| GitHub | Source control and collaboration. |
| Vercel | Future frontend/app deployment path when needed. |
| Railway | Future services, jobs, databases, or backend deployment path when needed. |
| Cloudflare | Future DNS, tunnels, routing, and edge/security path when needed. |

## What Nebius Does in Webinar 1

Nebius has two jobs in this series:

- Nebius Token Factory provides the model API. Pi and Hermes both send requests
  to Token Factory instead of using a local model or an OpenAI key.
- Nebius Cloud will provide the server. In this local pass, Docker on the
  laptop stands in for that server so we can verify the environment before
  translating it to a VM.

Token Factory exposes an OpenAI-compatible API. In practice, that means the
important connection details are:

```text
base URL: https://api.tokenfactory.nebius.com/v1/
auth:     Authorization: Bearer <your-token-factory-api-key>
model:    google/gemma-3-27b-it
```

We use the same Token Factory key twice:

- Pi uses it through `~/.pi/agent/models.json`.
- NemoClaw/Hermes uses it during sandbox setup through `COMPATIBLE_API_KEY`.

Package manager rule:

- Use `pnpm` for project-level dependencies and local repo installs.
- Use `npm` for global CLI tools, including Pi, unless a tool's docs require a
  different path.

## Prerequisites

Local requirements:

- Docker installed and running.
- Node.js 22+.
- npm available for global CLI installs.
- `jq` and `curl`.
- A Nebius Token Factory account with billing completed.
- Pi Coding Agent installed.

NemoClaw requirements:

- Minimum: 4 vCPU, 8 GB RAM, 20 GB free disk.
- Preferred: 4+ vCPU, 16 GB RAM, 40 GB free disk.

Official docs:

- [Pi docs](https://pi.dev/docs/latest)
- [Nebius Token Factory quickstart](https://docs.tokenfactory.nebius.com/quickstart)
- [Nebius Token Factory API authentication](https://docs.tokenfactory.nebius.com/api-reference/introduction)
- [Nebius Token Factory billing](https://docs.tokenfactory.nebius.com/other-capabilities/billing-new)
- [NemoClaw docs](https://docs.nvidia.com/nemoclaw/user-guide/openclaw/home)
- [Nebius Compute docs](https://docs.nebius.com/compute/)

## Step 1: Confirm Docker Capacity

Run:

```bash
docker info --format 'Server={{.ServerVersion}} CPUs={{.NCPU}} MemBytes={{.MemTotal}}'
```

Local observed result:

```text
Server=28.1.1 CPUs=12 MemBytes=8217751552
```

That is about 7.7 GiB of Docker-visible memory. It worked locally, but it is
below the preferred 16 GB target. If builds fail or get killed, increase Docker
Desktop memory.

## Step 2: Create and Store the Token Factory API Key

Official docs:

- [Nebius Token Factory quickstart](https://docs.tokenfactory.nebius.com/quickstart)
- [Nebius Token Factory API authentication](https://docs.tokenfactory.nebius.com/api-reference/introduction)
- [Nebius Token Factory billing](https://docs.tokenfactory.nebius.com/other-capabilities/billing-new)

In the browser:

1. Open [tokenfactory.nebius.com](https://tokenfactory.nebius.com/).
2. Create an account or log in with Google/GitHub.
3. Complete billing setup if prompted. Nebius Token Factory requires billing
   during onboarding.
4. Open the API keys section.
5. Click `Create API key`.
6. Give the key a name, for example `fde-trainer-webinar-local`.
7. Copy the displayed key immediately. Token Factory does not show it again
   later.

Do not paste the key into the repo, chat, slides, or markdown files.

For this verified local path on macOS, store the key in Keychain:

```bash
read -rsp "Paste Nebius Token Factory API key: " NEBIUS_API_KEY
printf '\n'

security add-generic-password \
  -a "$USER" \
  -s nebius-token-factory-api-key \
  -w "$NEBIUS_API_KEY" \
  -U

unset NEBIUS_API_KEY
```

Verify that the key exists without printing it:

```bash
security find-generic-password \
  -s nebius-token-factory-api-key \
  >/dev/null && echo "Token Factory key is stored"
```

If you are on Linux or on the later Nebius VM path, use a current-shell
environment variable or a secret manager instead:

```bash
read -rsp "Paste Nebius Token Factory API key: " NEBIUS_API_KEY
printf '\n'
export NEBIUS_API_KEY
```

That env-var path does not write the key to disk, but you must set it again in
new terminal sessions unless you move it into a proper secret manager.

Smoke-test the key against Token Factory without printing it:

```bash
TOKEN="$(security find-generic-password -w -s nebius-token-factory-api-key)"

curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "https://api.tokenfactory.nebius.com/v1/models?verbose=true" \
| jq -r '.data[] | select(.id == "google/gemma-3-27b-it") | .id'

unset TOKEN
```

Expected:

```text
google/gemma-3-27b-it
```

## Step 3: Install Pi and Configure the Nebius Provider

Pi is a global CLI, so install it using the official npm path if needed:

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

Official docs:

- [Pi quickstart](https://pi.dev/docs/latest)
- [Pi custom models](https://pi.dev/docs/latest/models)
- [Pi settings](https://pi.dev/docs/latest/settings)

Create the Pi config directory:

```bash
mkdir -p ~/.pi/agent
chmod 700 ~/.pi/agent
```

Back up existing Pi config if present:

```bash
timestamp="$(date +%Y%m%d%H%M%S)"

[ -f ~/.pi/agent/models.json ] && \
  cp ~/.pi/agent/models.json ~/.pi/agent/models.json.backup."$timestamp"

[ -f ~/.pi/agent/settings.json ] && \
  cp ~/.pi/agent/settings.json ~/.pi/agent/settings.json.backup."$timestamp"
```

Create `~/.pi/agent/models.json` with a custom Token Factory provider:

```bash
cat > ~/.pi/agent/models.json <<'JSON'
{
  "providers": {
    "nebius-token-factory": {
      "baseUrl": "https://api.tokenfactory.nebius.com/v1/",
      "api": "openai-completions",
      "apiKey": "!security find-generic-password -w -s nebius-token-factory-api-key",
      "models": [
        {
          "id": "google/gemma-3-27b-it",
          "name": "Gemma 3 27B via Nebius Token Factory",
          "input": ["text"],
          "reasoning": false,
          "contextWindow": 110000,
          "maxTokens": 16384,
          "cost": {
            "input": 0.1,
            "output": 0.3,
            "cacheRead": 0,
            "cacheWrite": 0
          },
          "compat": {
            "supportsReasoningEffort": false
          }
        }
      ]
    }
  }
}
JSON

chmod 600 ~/.pi/agent/models.json
```

If you are using the Linux/current-shell env-var path, change the `apiKey` line
above to:

```json
"apiKey": "$NEBIUS_API_KEY"
```

Pin Pi to the Nebius provider so it does not silently use another ambient API
key:

```bash
cat > ~/.pi/agent/settings.json <<'JSON'
{
  "defaultProvider": "nebius-token-factory",
  "defaultModel": "google/gemma-3-27b-it",
  "defaultThinkingLevel": "off",
  "enabledModels": [
    "nebius-token-factory/google/gemma-3-27b-it"
  ]
}
JSON

chmod 600 ~/.pi/agent/settings.json
```

## Step 4: Verify Pi Uses Nebius Token Factory

Validate the configured model list:

```bash
pi --list-models nebius
```

Then verify Pi does not fall back to an ambient `OPENAI_API_KEY`:

```bash
OPENAI_API_KEY="not-a-real-key" \
  pi --no-skills \
  --no-tools \
  --no-context-files \
  --no-session \
  -p "Reply with exactly: still nebius"
```

Expected:

```text
still nebius
```

For a stronger check, use Pi's JSON event stream:

```bash
OPENAI_API_KEY="not-a-real-key" \
  pi --mode json \
  --no-tools \
  --no-context-files \
  --no-session \
  -p "Reply with exactly: json smoke" \
| jq -r '
    select(.type == "message_end" and .message.role == "assistant")
    | [.message.provider, .message.model, .message.api]
    | @tsv
  '
```

Expected:

```text
nebius-token-factory	google/gemma-3-27b-it	openai-completions
```

Do not ask the model what provider it is using. Models can guess. Use runtime
metadata or controlled behavior checks.

Local secret note:

- The local setup stores the Token Factory API key in macOS Keychain under
  `nebius-token-factory-api-key`.
- The key is not written into the repo.
- On Linux/Nebius, replace this with an environment variable, container secret,
  or VM bootstrap secret.

## Step 5: Run NemoClaw/Hermes

Official docs:

- [NemoClaw OpenClaw docs](https://docs.nvidia.com/nemoclaw/user-guide/openclaw/home)
- [NemoClaw installer](https://www.nvidia.com/nemoclaw.sh)

Run the installer:

```bash
TOKEN="$(security find-generic-password -w -s nebius-token-factory-api-key)"

[ -n "$TOKEN" ] || {
  echo "Missing Nebius Token Factory API key"
  exit 1
}

curl -fsSL https://www.nvidia.com/nemoclaw.sh | \
  NEMOCLAW_AGENT=hermes \
  NEMOCLAW_NON_INTERACTIVE=1 \
  NEMOCLAW_ACCEPT_THIRD_PARTY_SOFTWARE=1 \
  NEMOCLAW_PROVIDER=custom \
  NEMOCLAW_ENDPOINT_URL=https://api.tokenfactory.nebius.com/v1/ \
  NEMOCLAW_MODEL=google/gemma-3-27b-it \
  NEMOCLAW_SANDBOX_NAME=fde-hermes-local \
  NEMOCLAW_WEB_SEARCH_PROVIDER=none \
  COMPATIBLE_API_KEY="$TOKEN" \
  NEMOCLAW_YES=1 \
  bash

unset TOKEN
```

If you are using the Linux/current-shell env-var path, replace the first line
with:

```bash
TOKEN="${NEBIUS_API_KEY:?Set NEBIUS_API_KEY first}"
```

The first image pull can be slow. Docker may retry a large layer and resume with
a smaller remaining chunk. That is not necessarily a failure.

If Docker appears stuck before image pull progress starts, restart Docker
Desktop and confirm a small pull works:

```bash
docker pull alpine:latest
```

## Step 6: Verify Hermes

Add local CLI shims to the current shell:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Check CLI versions:

```bash
nemohermes --version
nemoclaw --version
```

Expected local versions:

```text
nemohermes v0.0.73
nemoclaw v0.0.73
```

Check the sandbox:

```bash
nemohermes fde-hermes-local status
```

Expected markers:

```text
Sandbox: fde-hermes-local
Phase: Ready
Harness: Hermes Agent (gateway)
Hermes Agent: running
Model: google/gemma-3-27b-it
```

Open the dashboard:

```text
http://127.0.0.1:18789/
```

Check the local API health:

```bash
curl -sS -o /tmp/nemoclaw-api-health.txt -w '%{http_code}\n' \
  http://127.0.0.1:8642/health
```

Expected:

```text
200
```

## Step 7: Upload the Webinar Repo

Upload the repo into the sandbox:

```bash
cd /path/to/nebius-fde-trainer-webinar-series
REPO_PATH="$(pwd)"

nemohermes fde-hermes-local upload \
  "$REPO_PATH" \
  /sandbox/
```

If you already uploaded the repo before these guide files existed, run the
upload again so the sandbox gets the latest files.

The current local upload created a nested path:

```text
/sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series
```

Create a clean shortcut:

```bash
nemohermes fde-hermes-local exec -- \
  ln -sfn /sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series /sandbox/workspace
```

Verify it. Use `find -L` because `/sandbox/workspace` is a symlink:

```bash
nemohermes fde-hermes-local exec -- \
  find -L /sandbox/workspace -maxdepth 3 -type f | sort
```

Expected:

```text
/sandbox/workspace/.gitignore
/sandbox/workspace/01-nebius-cloud-builder-environment/01-nebius-cloud-builder-environment.md
/sandbox/workspace/01-nebius-cloud-builder-environment/README.md
/sandbox/workspace/01-nebius-cloud-builder-environment/video-tutorial-script.md
/sandbox/workspace/01-nebius-cloud-builder-environment/written-guide.md
/sandbox/workspace/README.md
```

## Step 8: Read the Repo From Inside the Sandbox

Run:

```bash
nemohermes fde-hermes-local exec -- \
  sed -n '1,40p' /sandbox/workspace/README.md
```

Expected first line:

```text
# Nebius FDE Trainer Webinar Series
```

This proves the sandbox can see the project.

## Step 9: Tool-Calling Gate

The dashboard proves Hermes is reachable. It does not prove Hermes can act.

Create a ground-truth file:

```bash
nemohermes fde-hermes-local exec -- \
  sh -lc 'printf "real-%s\n" "$(date +%s)" > /tmp/hermes-ground-truth.txt && cat /tmp/hermes-ground-truth.txt'
```

Ask Hermes to read it with tools:

```bash
nemohermes fde-hermes-local exec --workdir /sandbox -- \
  hermes chat \
    --yolo \
    -t terminal,file \
    -Q \
    -q "Use the terminal tool to run: cat /tmp/hermes-ground-truth.txt. Return only the observed command output."
```

Then check whether a tool was actually used:

```bash
nemohermes fde-hermes-local exec -- \
  hermes logs --since 2m | grep -E 'tool_turns|conversation turn' | tail -n 8
```

Expected for a trusted agent path:

```text
tool_turns=1
```

Current local result:

```text
tool_turns=0
```

That means Hermes chat is not executing tools through the current custom
provider route. Until this passes, do not ask Hermes to inspect or edit files.
Use shell commands for evidence.

## Step 10: Environment Checkpoint

You are ready to continue when all of these are true:

```bash
nemohermes fde-hermes-local status
```

Shows:

```text
Phase: Ready
```

```bash
nemohermes fde-hermes-local exec -- \
  pwd
```

Shows:

```text
/sandbox
```

```bash
nemohermes fde-hermes-local exec -- \
  sed -n '1,40p' /sandbox/workspace/README.md
```

Shows:

```text
# Nebius FDE Trainer Webinar Series
```

## What We Built

We now have:

- A local server environment running through Docker.
- Hermes running inside a NemoClaw/OpenShell sandbox.
- Nebius Token Factory connected for inference.
- The webinar repo available inside the sandbox at `/sandbox/workspace`.
- A clear trust gate for whether Hermes can execute tools.

## Known Issue

Hermes tool execution through the current custom provider path is unresolved.
Token Factory can return structured `tool_calls` directly, but Hermes sessions
through the current route return plain text and log `tool_turns=0`.

For now:

- Do not let Hermes edit files.
- Do not trust Hermes file summaries unless they are grounded in shell output.
- Continue using `nemohermes fde-hermes-local exec -- <command>` for evidence.

## Next Webinar Step

The next step is the cloud translation:

- Provision the Nebius Cloud VM.
- Install the same builder environment there.
- Decide whether the packaging path is `cloud-init`, Docker image, install
  script, template repo, or another Nebius-supported flow.
