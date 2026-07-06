# Local Testing Script: Pi, Token Factory, NemoClaw, and Hermes

Draft for Webinar 1 recording.

Working title:

```text
Webinar 1: Build the Local Server Environment for the FDE Trainer Agent
```

## Recording Goal

By the end of the video, the student should understand the environment shape
and have a working local sandbox:

- Pi Coding Agent is configured to use Nebius Token Factory.
- Docker is acting as the local server runtime.
- NemoClaw/NemoHermes is running Hermes Agent in a sandbox.
- The webinar repo is available inside the sandbox at `/sandbox/workspace`.
- The student knows not to trust agent output until tool execution is verified.

## Before Recording

Open:

- Terminal on the local machine.
- Hermes dashboard: `http://127.0.0.1:18789/`.
- This repo directory.

Confirm:

```bash
export PATH="$HOME/.local/bin:$PATH"
nemohermes fde-hermes-local status
```

Expected marker:

```text
Phase: Ready
```

## Segment 1: Opening

Say:

```text
In this first webinar, we are setting up the builder environment for the FDE
Trainer Agent. FDE means Forward Deployed Engineering. The goal of the series is
to launch an agent that helps builders train to become Forward Deployed
Engineers.
```

Say:

```text
Today is not a product tour. We are building the runtime that the rest of the
series depends on. Later we will move this shape to Nebius Cloud, but first we
prove it locally.
```

Show the environment map:

```text
Local laptop -> Docker -> NemoClaw/OpenShell sandbox -> Hermes Agent
Nebius Token Factory -> inference for Pi and Hermes
Webinar repo -> uploaded into /sandbox/workspace
```

## Segment 2: Tool Roles

Say:

```text
Each tool has a job. Pi is the coding agent we use for setup work and debugging.
Nebius Token Factory provides model inference. NemoClaw manages the Hermes
sandbox. Docker gives us the local server runtime. Nebius Cloud will replace the
laptop server later.
```

Say:

```text
GitHub, Vercel, Railway, and Cloudflare belong to the broader builder toolchain.
We will only install or configure them when the demo actually uses them.
```

## Segment 3: Nebius Token Factory Account and API Key

Say:

```text
Nebius appears twice in this series. Nebius Cloud is where the server will live.
Nebius Token Factory is where the model inference comes from. Today we use
Token Factory immediately, and we translate the server setup to Nebius Cloud
after the local path works.
```

Say:

```text
Token Factory exposes an OpenAI-compatible API. That means tools like Pi and
Hermes can use a base URL, a model name, and an API key. The base URL for Token
Factory is https://api.tokenfactory.nebius.com/v1/.
```

Show:

```text
Official docs:
- https://docs.tokenfactory.nebius.com/quickstart
- https://docs.tokenfactory.nebius.com/api-reference/introduction
- https://docs.tokenfactory.nebius.com/other-capabilities/billing-new
```

Say:

```text
In the Token Factory web app, create an account or log in, finish billing if
prompted, open API keys, create a key, name it for this webinar, and copy it
once. Token Factory will not show the same key again later.
```

Say:

```text
Do not paste this key into the repo, the guide, the chat, or the slides. For
this recording, I am storing it in macOS Keychain and then reading it from the
terminal only when a tool needs it.
```

Run:

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

Run:

```bash
security find-generic-password \
  -s nebius-token-factory-api-key \
  >/dev/null && echo "Token Factory key is stored"
```

Say:

```text
That verifies the key exists without printing it. On Linux or on the later
Nebius VM, we will use an environment variable, container secret, or VM secret
instead of macOS Keychain.
```

Run:

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

Say:

```text
This is the first real integration check. The key works, the Token Factory API
is reachable, and the model we plan to use is visible.
```

## Segment 4: Configure Pi for Nebius Token Factory

Say:

```text
The local shell may already have an OPENAI_API_KEY for other tools. For this
webinar, Pi must use Nebius Token Factory, not that ambient OpenAI key.
```

Say:

```text
Pi supports custom model providers through ~/.pi/agent/models.json. We will add
Token Factory as a provider, then pin Pi's default provider and model in
~/.pi/agent/settings.json.
```

Run:

```bash
mkdir -p ~/.pi/agent
chmod 700 ~/.pi/agent

timestamp="$(date +%Y%m%d%H%M%S)"

[ -f ~/.pi/agent/models.json ] && \
  cp ~/.pi/agent/models.json ~/.pi/agent/models.json.backup."$timestamp"

[ -f ~/.pi/agent/settings.json ] && \
  cp ~/.pi/agent/settings.json ~/.pi/agent/settings.json.backup."$timestamp"
```

Run:

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

Run:

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

Run:

```bash
pi --list-models nebius
```

Run:

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

Say:

```text
We do not ask the model what provider it is using. Models can guess. We verify
runtime metadata and behavior.
```

Run:

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

## Segment 5: Start From Docker as the Local Server

Say:

```text
For the local pass, Docker is our server. This gives us a reference setup before
we translate the same environment to a Nebius Cloud VM.
```

Run:

```bash
docker info --format 'Server={{.ServerVersion}} CPUs={{.NCPU}} MemBytes={{.MemTotal}}'
```

Say:

```text
NemoClaw's documented floor is 4 vCPU, 8 GB RAM, and 20 GB free disk. The
preferred target is closer to 16 GB RAM and 40 GB disk. If Docker has too little
memory, the sandbox can build slowly or fail.
```

## Segment 6: Install and Verify NemoClaw/Hermes

Say:

```text
NemoClaw uses the same Token Factory key. Pi uses it through models.json.
Hermes receives it during the NemoClaw setup as COMPATIBLE_API_KEY.
```

Show:

```text
Official docs:
- https://docs.nvidia.com/nemoclaw/user-guide/openclaw/home
- https://www.nvidia.com/nemoclaw.sh
```

Run:

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

Say:

```text
The important idea is that NemoClaw is not the model provider. It is the sandbox
and lifecycle layer. Hermes runs inside the sandbox, and Token Factory provides
the model responses.
```

Run:

```bash
export PATH="$HOME/.local/bin:$PATH"
nemohermes --version
nemoclaw --version
```

Expected:

```text
nemohermes v0.0.73
nemoclaw v0.0.73
```

Run:

```bash
nemohermes fde-hermes-local status
```

Say:

```text
This tells us whether the sandbox is alive. We are looking for Phase: Ready and
Hermes Agent running.
```

Show the dashboard:

```text
http://127.0.0.1:18789/
```

Say:

```text
The dashboard proves the service is reachable. It does not prove the agent is
safe to trust yet.
```

## Segment 7: Upload the Webinar Repo

Say:

```text
Now we put the webinar repo inside the sandbox. This lets the sandbox see the
same artifacts we are building locally.
```

Run:

```bash
cd /path/to/nebius-fde-trainer-webinar-series
REPO_PATH="$(pwd)"

nemohermes fde-hermes-local upload \
  "$REPO_PATH" \
  /sandbox/
```

Create the stable shortcut:

```bash
nemohermes fde-hermes-local exec -- \
  ln -sfn /sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series /sandbox/workspace
```

Verify with `find -L` because `/sandbox/workspace` is a symlink:

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

Say:

```text
From this point forward, /sandbox/workspace is the repo path we use in the
sandbox.
```

## Segment 8: Agent Trust Gate

Say:

```text
Before we let an agent touch files, we test whether it actually executes tools.
This is the first Forward Deployed Engineering habit: verify the environment
before acting.
```

Run:

```bash
nemohermes fde-hermes-local exec -- \
  sh -lc 'printf "real-%s\n" "$(date +%s)" > /tmp/hermes-ground-truth.txt && cat /tmp/hermes-ground-truth.txt'
```

Run:

```bash
nemohermes fde-hermes-local exec --workdir /sandbox -- \
  hermes chat \
    --yolo \
    -t terminal,file \
    -Q \
    -q "Use the terminal tool to run: cat /tmp/hermes-ground-truth.txt. Return only the observed command output."
```

Say if the local result is still `tool_turns=0`:

```text
This is the important debugging moment. Hermes is running, Token Factory is
connected, and the sandbox filesystem works. But the Hermes chat route is not
executing tools in this configuration. So we do not let the agent inspect or
edit files yet.
```

Say:

```text
When agent output conflicts with shell evidence, shell evidence wins.
```

## Segment 9: Environment Checkpoint

Run:

```bash
nemohermes fde-hermes-local status
```

Run:

```bash
nemohermes fde-hermes-local exec -- \
  pwd
```

Run:

```bash
nemohermes fde-hermes-local exec -- \
  sed -n '1,40p' /sandbox/workspace/README.md
```

Say:

```text
You are ready to continue when the sandbox is Ready, /sandbox/workspace exists,
and the README can be read from inside the sandbox.
```

## Close

Say:

```text
We now have the local builder environment. The next step is to move from setup
into the agent deployment path: Hermes running through NemoClaw, then the FDE
Trainer personality, skills, integrations, and eventually the paid product path.
```

Say:

```text
The unresolved item is Hermes tool execution through the custom provider route.
That is not hidden. It is part of the engineering work. We mark it, keep the
environment checkpoint, and do not trust agent actions until the gate passes.
```
