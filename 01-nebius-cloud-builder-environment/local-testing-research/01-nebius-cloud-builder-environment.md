# 1. Nebius Cloud Builder Environment

## Goal

Provision the Nebius Cloud server and install the builder environment for the
rest of the series.

## What Participants Should Understand

- What the Nebius Cloud server is for.
- Why Pi Coding Agent is part of the environment.
- How Nebius Token Factory powers the coding agent.
- Where GitHub, Vercel, Railway, and Cloudflare fit in the build.
- Why the setup should be packaged and repeatable.

## Target Environment

NemoClaw requirements are the floor:

- Minimum: 4 vCPU, 8 GB RAM, 20 GB free disk.
- Preferred: 4+ vCPU, 16 GB RAM, 40 GB free disk.

Recommended Nebius VM starting point:

- VM type: regular, non-GPU Compute VM.
- Platform: `cpu-d3` where available, or `cpu-e2` in `eu-north1`.
- Preset: `4vcpu-16gb`.
- Boot image family: `ubuntu24.04-driverless`.
- Boot disk: 50 GiB `network_ssd` to leave room for Docker images and install
  caches.
- Network: dynamic public IPv4 for the webinar demo so participants can SSH in
  directly. For production-like variants, remove the public IP and use a jump
  server or VPN.

Before finalizing the guide, verify availability in the target project:

```bash
nebius compute platform list --parent-id <project_ID>
```

## Current Implementation Mode

For the first local pass, the laptop is the server.

We are starting at Step 4 because the cloud server provisioning steps are not
needed until we translate the working local setup to Nebius Cloud.

Cloud provisioning still belongs in the final webinar guide:

- Nebius docs support VM creation through the web console, CLI, and Terraform.
- The CLI path is reproducible enough for a written guide and scriptable enough
  for a finished artifact.
- `cloud-init` can create the SSH user and bootstrap Docker, Node.js, Pi, and
  the builder tooling on first boot.
- Managed boot disks can be created together with the VM and deleted with the VM,
  which fits the disposable recording server.

Use the web console as a fallback teaching path, not the canonical artifact.
Terraform and custom VM images are useful later, but they add setup overhead
before we know the exact environment.

## Package Manager Rule

- Use `pnpm` for project-level dependencies and repository-local installs.
- Use `npm` for global CLI tools.
- Pi Coding Agent is a global CLI tool, so the guide should reference the
  official `npm install -g` path.

When referencing install commands, include the official documentation link next
to the command.

## Tool Log

These are the tools used during the local setup. Add new tools here as
they appear in the implementation.

| Tool | Local status | Why it matters for the Nebius VM tutorial |
| --- | --- | --- |
| Pi Coding Agent | `0.80.3` | Agent harness for local setup and later VM/container work. |
| Node.js | `v22.21.0` | Required by Pi and NemoClaw. |
| npm | Available through Node.js | Used for global CLI installs such as Pi. |
| jq | Available | Used to inspect Token Factory model metadata and pricing. |
| curl | Available | Used for Token Factory API smoke tests. |
| macOS Keychain `security` CLI | Available locally | Used locally so the Token Factory API key is not stored in plaintext. Replace with env vars or container secrets on Linux/Nebius. |
| Docker | `28.1.1`, 12 CPUs, ~7.7 GiB memory visible to Docker | Required because NemoClaw uses Docker/OpenShell for sandboxed Hermes. Increase Docker Desktop memory if sandbox builds or runtime work fail. |
| NemoHermes CLI | `0.0.73` | NemoClaw-managed Hermes Agent onboarding and sandbox control. |
| NemoClaw CLI | `0.0.73` | Installed as a shim with NemoHermes. |
| OpenShell | `0.0.71` | Sandbox gateway used by NemoClaw on local Docker. |
| Hermes sandbox base image | `ghcr.io/nvidia/nemoclaw/hermes-sandbox-base:v0.0.73`, 2.47 GB | Base image pulled by NemoClaw before building the local Hermes sandbox. |

## Step 4: Pi Coding Agent

Pi is installed locally and available as:

```bash
pi --version
```

Official docs:

- [Pi quickstart](https://pi.dev/docs/latest)

If installing from scratch on the Nebius VM, use the canonical install sequence
in [`../written-guide.md`](../written-guide.md), Step 12. It configures npm to
install global CLIs into the user's home directory before installing Pi.

## Step 5: Configure Nebius Token Factory

Pi supports custom providers in:

```text
~/.pi/agent/models.json
```

Official docs:

- [Pi custom models](https://pi.dev/docs/latest/models)
- [Nebius Token Factory quickstart](https://docs.tokenfactory.nebius.com/quickstart)
- [Token Factory model list API](https://docs.tokenfactory.nebius.com/api-reference/models/list-models.md)

Local secret storage:

- The local macOS setup stores the Token Factory API key in Keychain under
  service `nebius-token-factory-api-key`.
- `models.json` references the key through Pi's documented command-based
  `apiKey` resolution.
- The API key is not written into the repository or into `models.json`.
- In the later Docker/Nebius VM path, use an environment variable or container
  secret instead of macOS Keychain.

Default provider selection:

- The local shell exports `OPENAI_API_KEY` for other tooling.
- To prevent Pi from using OpenAI by default, pin the Pi default provider and
  model in `~/.pi/agent/settings.json`.

```json
{
  "defaultProvider": "nebius-token-factory",
  "defaultModel": "google/gemma-3-27b-it",
  "defaultThinkingLevel": "off",
  "enabledModels": [
    "nebius-token-factory/google/gemma-3-27b-it"
  ]
}
```

Current local Pi provider config:

```json
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
```

Model selection note:

- Token Factory's authenticated `/v1/models?verbose=true` endpoint returns
  pricing metadata.
- The selected first-test model is `google/gemma-3-27b-it`.
- It was chosen because the live metadata listed it as an active text model with
  tool support and low prompt/completion prices.

Validation commands:

```bash
pi --list-models nebius
```

Do not ask the model which provider or model it is using. The model can guess or
hallucinate that answer. Use Pi's JSON event stream instead:

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

Expected metadata:

```text
nebius-token-factory	google/gemma-3-27b-it	openai-completions
```

```bash
pi --provider nebius-token-factory \
  --model google/gemma-3-27b-it \
  --no-tools \
  --no-context-files \
  --no-session \
  -p "Reply with exactly: pi token factory ok"
```

Expected output:

```text
pi token factory ok
```

Verify Pi is not using ambient OpenAI credentials:

```bash
OPENAI_API_KEY="not-a-real-key" \
  pi --no-skills \
  --no-tools \
  --no-context-files \
  --no-session \
  -p "Reply with exactly: still nebius"
```

Expected output:

```text
still nebius
```

For the first repo-inspection smoke test, disable skills so duplicate global
skill installs do not add noisy startup warnings:

```bash
pi --no-skills \
  --tools read,ls,find \
  --no-session
```

If Pi prints `Skill conflicts`, it means the same skill name exists in multiple
skill roots. In the local setup, duplicates exist under both `~/.pi/agent/skills`
and `~/.agents/skills`. Pi chooses one copy and skips the other, so this is not
an execution failure.

## Step 6: Run NemoClaw/Hermes Locally With Docker

This step runs the same agent environment that we will later move to a Nebius
Cloud server. For the local pass, the laptop is the server and Docker provides
the sandbox.

Official docs:

- [NemoClaw OpenClaw docs](https://docs.nvidia.com/nemoclaw/user-guide/openclaw/home)
- [NemoClaw installer](https://www.nvidia.com/nemoclaw.sh)

Local preflight:

```bash
docker info --format 'Server={{.ServerVersion}} CPUs={{.NCPU}} MemBytes={{.MemTotal}}'
```

Local observed result:

```text
Server=28.1.1 CPUs=12 MemBytes=8217751552
```

This is about 7.7 GiB of Docker-visible memory. NemoClaw can start on this, but
the recommended local target is closer to 16 GiB. If the sandbox build fails,
hangs, or gets killed under load, increase Docker Desktop's memory allocation.

Run the NemoClaw installer for Hermes Agent with Nebius Token Factory:

```bash
TOKEN="$(security find-generic-password -w -s nebius-token-factory-api-key)"

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

Why these settings:

- `NEMOCLAW_AGENT=hermes` selects Hermes Agent for the webinar path.
- `NEMOCLAW_PROVIDER=custom` maps Token Factory to NemoClaw's OpenAI-compatible
  provider path.
- `NEMOCLAW_ENDPOINT_URL` points at the Token Factory OpenAI-compatible API.
- `NEMOCLAW_MODEL` uses the same cheap first-test model as Pi.
- `NEMOCLAW_WEB_SEARCH_PROVIDER=none` keeps search disabled for this setup step.
  Tavily belongs later, when we wire search intentionally.
- `COMPATIBLE_API_KEY` is populated from Keychain, not pasted into the guide or
  stored in plaintext.

The installer writes local CLIs to:

```text
~/.local/bin/nemohermes
~/.local/bin/nemoclaw
~/.local/bin/openshell
```

For the current shell:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

If the shell cannot find `nemohermes` after install, add that PATH line to the
shell profile used in the workshop environment.

### Observed Local Install Notes

The first local Docker pull was slow and retried the largest layer. This is not
necessarily a failure. Docker may print a retry countdown and then resume with a
smaller remaining chunk.

If Docker appears stuck before the image pull starts, restart Docker Desktop and
confirm a small pull works:

```bash
docker pull alpine:latest
```

The local run pulled this base image:

```bash
docker pull ghcr.io/nvidia/nemoclaw/hermes-sandbox-base:v0.0.73
```

Observed image:

```text
ghcr.io/nvidia/nemoclaw/hermes-sandbox-base   v0.0.73   2.47GB
```

The first successful onboarding built the sandbox image in about 42 seconds
after the base image was available.

### Verify NemoClaw/Hermes

Check CLI availability:

```bash
export PATH="$HOME/.local/bin:$PATH"

nemohermes --version
nemoclaw --version
```

Expected local output:

```text
nemohermes v0.0.73
nemoclaw v0.0.73
```

Check sandbox status:

```bash
nemohermes fde-hermes-local status
```

Expected status markers:

```text
Sandbox: fde-hermes-local
Phase: Ready
Harness: Hermes Agent (gateway)
Hermes Agent: running
Model: google/gemma-3-27b-it
```

Check local HTTP endpoints:

```bash
curl -sS -o /tmp/nemoclaw-dashboard-check.txt -w '%{http_code}\n' \
  http://127.0.0.1:18789/

curl -sS -o /tmp/nemoclaw-api-health.txt -w '%{http_code}\n' \
  http://127.0.0.1:8642/health
```

Expected output:

```text
200
200
```

Check the local OpenAI-compatible API path end to end:

```bash
TOKEN="$(nemohermes fde-hermes-local gateway-token --quiet)"

HTTP_CODE=$(curl -sS -o /tmp/nemoclaw-hermes-chat.json -w '%{http_code}' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"model":"google/gemma-3-27b-it","messages":[{"role":"user","content":"Reply exactly: hermes-ready"}],"max_tokens":20,"temperature":0}' \
  http://127.0.0.1:8642/v1/chat/completions)

unset TOKEN

printf 'http=%s\n' "$HTTP_CODE"
jq -r '.choices[0].message.content' /tmp/nemoclaw-hermes-chat.json
```

Expected output:

```text
http=200
hermes-ready
```

Local access URLs:

```text
Hermes Agent Dashboard: http://127.0.0.1:18789/
Hermes Agent API:       http://127.0.0.1:8642/v1
```

Management commands:

```bash
nemohermes fde-hermes-local status
nemohermes fde-hermes-local logs --follow
nemohermes fde-hermes-local connect
nemohermes fde-hermes-local gateway-token --quiet
```

For the Nebius VM tutorial, replace macOS Keychain with a Linux-safe secret
path. The minimum acceptable path is an environment variable set outside the
repository. A better path is a container secret or VM bootstrap secret, depending
on the final provisioning approach.

## Step 7: Tool-Calling Gate

The dashboard loading is not enough. For this webinar series, Hermes must be
able to execute tools. If it only chats, it cannot be trusted to inspect files,
fix bugs, or work like an agent.

Create a ground-truth file inside the sandbox:

```bash
export PATH="$HOME/.local/bin:$PATH"

nemohermes fde-hermes-local exec -- \
  sh -lc 'printf "real-%s\n" "$(date +%s)" > /tmp/hermes-ground-truth.txt && cat /tmp/hermes-ground-truth.txt'
```

Ask Hermes to read that file through its terminal tool:

```bash
nemohermes fde-hermes-local exec --workdir /sandbox -- \
  hermes chat \
    --yolo \
    -t terminal,file \
    -Q \
    -q "Use the terminal tool to run: cat /tmp/hermes-ground-truth.txt. Return only the observed command output."
```

Expected result:

```text
real-<timestamp>
```

Verify the session actually used a tool:

```bash
nemohermes fde-hermes-local exec -- \
  hermes logs --since 2m | grep -E 'tool_turns|conversation turn' | tail -n 8
```

Expected marker:

```text
tool_turns=1
```

Current local result:

```text
tool_turns=0
```

In the current local setup, Token Factory itself returns structured
`tool_calls`, but Hermes sessions through the custom provider choose plain text
instead of tool calls. This means the Hermes dashboard and Hermes chat should
not be used yet for file-inspection tasks. Treat any response that prints
literal `tool_code`, invents files, or summarizes unseen files as a failed
tool-calling test.

This is not a prompt-design problem. It is a provider/tool-call compatibility
gate that must be resolved before the agent can be trusted to act.

Video framing:

```text
The first agent lesson is not prompt engineering. It is verification. If the
agent cannot prove it executed a tool, we do not let it touch the project. We
fall back to shell commands, capture the evidence, and mark the tool-calling
route as unresolved.
```

## Step 8: First Repo Inspection With Shell Evidence

Until the Hermes tool-calling gate passes, use shell commands as the source of
truth and use the model only to summarize evidence that is pasted into the
prompt.

Upload the local repo into the sandbox from the local machine:

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /path/to/nebius-fde-trainer-webinar-series
REPO_PATH="$(pwd)"

nemohermes fde-hermes-local upload \
  "$REPO_PATH" \
  /sandbox/
```

Verify the sandbox path from the local machine:

```bash
nemohermes fde-hermes-local exec -- \
  find /sandbox/nebius-fde-trainer-webinar-series -maxdepth 3 -type f | sort
```

Expected files:

```text
/sandbox/nebius-fde-trainer-webinar-series/.gitignore
/sandbox/nebius-fde-trainer-webinar-series/01-nebius-cloud-builder-environment/01-nebius-cloud-builder-environment.md
/sandbox/nebius-fde-trainer-webinar-series/01-nebius-cloud-builder-environment/README.md
/sandbox/nebius-fde-trainer-webinar-series/01-nebius-cloud-builder-environment/video-tutorial-script.md
/sandbox/nebius-fde-trainer-webinar-series/01-nebius-cloud-builder-environment/written-guide.md
/sandbox/nebius-fde-trainer-webinar-series/README.md
```

If a previous upload created a nested directory, inspect the real nested path.
Example:

```text
/sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series
```

Earlier local observed path before the guide/script files were added:

```text
/sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series/.gitignore
/sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series/01-nebius-cloud-builder-environment/01-nebius-cloud-builder-environment.md
/sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series/README.md
```

Confirmed read test:

```bash
nemohermes fde-hermes-local exec -- \
  sed -n '1,120p' /sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series/01-nebius-cloud-builder-environment/01-nebius-cloud-builder-environment.md
```

Expected first line:

```text
# 1. Nebius Cloud Builder Environment
```

Confirmed README read test:

```bash
nemohermes fde-hermes-local exec -- \
  sed -n '1,120p' /sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series/README.md
```

Expected first line:

```text
# Nebius FDE Trainer Webinar Series
```

Create a stable workspace shortcut so the rest of the tutorial does not depend
on the accidental nested upload path:

```bash
nemohermes fde-hermes-local exec -- \
  ln -sfn /sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series /sandbox/workspace
```

Verify the shortcut:

```bash
nemohermes fde-hermes-local exec -- \
  ls -la /sandbox/workspace
```

Expected output shape:

```text
/sandbox/workspace -> /sandbox/nebius-fde-trainer-webinar-series/nebius-fde-trainer-webinar-series
```

Because `/sandbox/workspace` is a symlink, use `find -L` to follow it:

```bash
nemohermes fde-hermes-local exec -- \
  find -L /sandbox/workspace -maxdepth 3 -type f | sort
```

Expected files:

```text
/sandbox/workspace/.gitignore
/sandbox/workspace/01-nebius-cloud-builder-environment/01-nebius-cloud-builder-environment.md
/sandbox/workspace/01-nebius-cloud-builder-environment/README.md
/sandbox/workspace/01-nebius-cloud-builder-environment/video-tutorial-script.md
/sandbox/workspace/01-nebius-cloud-builder-environment/written-guide.md
/sandbox/workspace/README.md
```

After refreshing the upload, `find -L /sandbox/workspace -maxdepth 3 -type f |
sort` should show those files. Use `/sandbox/workspace` as the canonical
sandbox repo path from this point forward.

## Step 9: Environment Checkpoint

Before moving on, confirm the local server environment is usable.

Check the sandbox status:

```bash
nemohermes fde-hermes-local status
```

Expected marker:

```text
Phase: Ready
```

Check the default sandbox working directory:

```bash
nemohermes fde-hermes-local exec -- \
  pwd
```

Expected output:

```text
/sandbox
```

Check that the webinar repo is readable from inside the sandbox:

```bash
nemohermes fde-hermes-local exec -- \
  sed -n '1,40p' /sandbox/workspace/README.md
```

Expected first line:

```text
# Nebius FDE Trainer Webinar Series
```

Confirmed local result: the sandbox is ready, `/sandbox/workspace` resolves to
the uploaded webinar repo, and the README is readable from inside the sandbox.

Audience checkpoint:

```text
You are ready to continue when the sandbox is Ready, /sandbox/workspace exists,
and the README can be read from inside the sandbox.
```

Video framing:

```text
This is the end of the local environment setup. We have Docker running the
NemoClaw/Hermes sandbox, Nebius Token Factory connected for inference, and the
webinar repo mounted into a clean workspace path. The only unresolved item is
Hermes tool execution through the custom provider, so we do not let the agent
modify files yet.
```

Run the repo inspection with shell commands:

```bash
nemohermes fde-hermes-local exec -- \
  find /sandbox/nebius-fde-trainer-webinar-series -maxdepth 3 -type f | sort

nemohermes fde-hermes-local exec -- \
  sed -n '1,80p' /sandbox/nebius-fde-trainer-webinar-series/README.md

nemohermes fde-hermes-local exec -- \
  sed -n '1,140p' /sandbox/nebius-fde-trainer-webinar-series/01-nebius-cloud-builder-environment/01-nebius-cloud-builder-environment.md
```

Video framing:

```text
The dashboard proves the service is up. Shell evidence proves what the
environment actually contains. When an agent output disagrees with shell
evidence, the shell wins.
```

## Source Notes

- [Nebius Compute quickstart](https://docs.nebius.com/compute/quickstart) uses
  the Nebius CLI, `jq`, SSH keys, `cloud-init`, boot disks, VM creation, and SSH
  connection.
- [Nebius VM creation docs](https://docs.nebius.com/compute/virtual-machines/manage)
  support web console, CLI, and Terraform, including custom `cloud-init`, public
  IP settings, and managed boot disks created with the VM.
- [Nebius VM types docs](https://docs.nebius.com/compute/virtual-machines/types)
  list `4vcpu-16gb` on non-GPU platforms, which satisfies NemoClaw's preferred
  RAM target.
- [Nebius platform-list docs](https://docs.nebius.com/compute/virtual-machines/list-platforms)
  show how to verify platforms and presets available in a project.
- [Nebius boot image docs](https://docs.nebius.com/compute/storage/boot-disk-images)
  recommend `ubuntu24.04-driverless` for non-GPU VMs.
- [Nebius Containers over VMs](https://docs.nebius.com/compute/virtual-machines/containers)
  supports custom Docker images from public registries.
- [Nebius custom-container tutorial](https://docs.nebius.com/compute/virtual-machines/applications-containers)
  shows custom Docker image deployment from a public registry.
- [NemoClaw prerequisites](https://docs.nvidia.com/nemoclaw/user-guide/hermes/get-started/prerequisites.md)
  require Node.js 22.16+, npm 10+, Docker access, and recommend Ubuntu 24.04 on
  Linux.

## Cleanup

The recording server can be destroyed after the webinar recording.
