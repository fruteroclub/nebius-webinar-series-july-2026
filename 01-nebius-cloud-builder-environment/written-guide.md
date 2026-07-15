# Webinar 1 Written Guide: Nebius Cloud Builder Server

## Goal

Deploy a Nebius Cloud server and install the first builder tools:

- Pi Coding Agent, connected to Nebius Token Factory.
- GitHub CLI, for repo/auth workflow.
- Vercel CLI, for frontend/app deployment workflow.

Stop after this guide. Hermes and NemoClaw begin in Webinar 2.

## Draft Status

This draft is based on a live Nebius run on July 5, 2026.

Verified live:

- Nebius CLI install and federation profile.
- Nebius VM provisioning with a managed 50 GiB boot disk.
- SSH access to the VM.
- Cloud-init baseline: Docker, Git, Node 22, npm 10, pnpm, jq, curl, unzip,
  wget, and `binutils`.
- Nebius Token Factory direct inference with an NVIDIA Nemotron model.
- Optional Hermes/NemoClaw validation on the same VM.
- Terminal chat with Hermes.
- VM deletion cleanup, with instance and disk lists returning empty.

Still official-doc-backed but not re-run before the VM was deleted:

- Pi Coding Agent install/config on the Nebius VM.
- GitHub CLI auth on the Nebius VM.
- Vercel CLI auth on the Nebius VM.

## Official Docs

Keep these open. If a command changes, the official docs win.

- Nebius CLI install: https://docs.nebius.com/cli/install
- Nebius CLI quickstart: https://docs.nebius.com/cli/quickstart
- Nebius VM management: https://docs.nebius.com/compute/virtual-machines/manage
- Nebius VM platforms: https://docs.nebius.com/compute/virtual-machines/list-platforms
- Nebius boot disk images: https://docs.nebius.com/compute/storage/boot-disk-images
- Token Factory quickstart: https://docs.tokenfactory.nebius.com/quickstart
- Pi docs: https://pi.dev/docs/latest
- GitHub CLI Linux install: https://github.com/cli/cli/blob/trunk/docs/install_linux.md
- GitHub CLI auth: https://cli.github.com/manual/gh_auth_login
- Vercel CLI: https://vercel.com/docs/cli

## What Runs Where

Run Nebius provisioning commands on your laptop.

Run Pi, GitHub CLI, and Vercel CLI setup inside the Nebius Cloud server after
you SSH into it.

## Prerequisites

You need:

- Nebius account with credits/billing available.
- Nebius project ID.
- Nebius Token Factory API key.
- Local terminal with `curl`, `ssh`, and `jq`.
- GitHub account.
- Vercel account.

Do not paste API keys into chat, screenshots, GitHub issues, commits, or slides.

## Step 1: Install Nebius CLI on Your Laptop

Run locally:

```bash
curl -sSL https://storage.eu-north1.nebius.cloud/cli/install.sh | bash
exec -l "$SHELL"
nebius version
```

## Step 2: Configure Nebius CLI

Run locally:

```bash
export NEBIUS_PROJECT_ID="<your-nebius-project-id>"
nebius profile create --parent-id "$NEBIUS_PROJECT_ID"
nebius profile list
```

Use these prompt values:

- Profile name: `fde-webinar`
- API endpoint: `api.nebius.cloud`
- Authorization type: `federation`
- Federation endpoint: `auth.nebius.com`

Complete the browser login. If prompted for a tenant, choose the tenant that
contains the webinar project. Confirm that the new profile is marked as default.

## Step 3: Choose the VM Shape

Run locally:

```bash
nebius compute platform list --parent-id "$NEBIUS_PROJECT_ID"
```

Pick the smallest available CPU VM with at least:

- 4 vCPU
- 16 GB RAM
- 50 GB boot disk
- Ubuntu 24.04

Set these variables locally:

```bash
export NEBIUS_VM_NAME="fde-builder-01"
export NEBIUS_VM_USER="fde"
export NEBIUS_PLATFORM="cpu-e2"
export NEBIUS_PRESET="4vcpu-16gb"
```

From the current project platform list, `cpu-e2` is the non-GPU Intel Ice Lake
platform and `4vcpu-16gb` is the smallest preset that satisfies the Webinar 1
target. Avoid GPU platforms for Webinar 1 because model inference comes from
Token Factory, not from a local GPU.

If `cpu-e2` hits a quota/capacity issue, use `cpu-d3` with the same
`4vcpu-16gb` preset.

## Step 4: Confirm Your SSH Public Key

Run locally.

You need an SSH public key because Step 5 puts that key into the VM's
cloud-init file. That is what allows your laptop to SSH into the Nebius VM after
it is created.

If you already have an SSH key, this step is only a quick confirmation. In the
live dry run, the local machine already had `~/.ssh/id_ed25519.pub`, so this
step took only a few seconds.

Run this check:

```bash
export NEBIUS_SSH_PUBLIC_KEY="${NEBIUS_SSH_PUBLIC_KEY:-$HOME/.ssh/id_ed25519.pub}"

if [ -r "$NEBIUS_SSH_PUBLIC_KEY" ]; then
  echo "OK: using SSH public key: $NEBIUS_SSH_PUBLIC_KEY"
  head -n 1 "$NEBIUS_SSH_PUBLIC_KEY"
else
  echo "No SSH public key found at: $NEBIUS_SSH_PUBLIC_KEY"
  echo "Create one with:"
  echo 'ssh-keygen -t ed25519 -C "nebius-fde-webinar"'
  echo 'export NEBIUS_SSH_PUBLIC_KEY="$HOME/.ssh/id_ed25519.pub"'
fi
```

If the command prints `OK`, continue to Step 5.

If it prints `No SSH public key found`, run the two commands it prints, then run
the check again. The public key output should start with `ssh-ed25519`,
`ssh-rsa`, or `ecdsa-`.

## Step 5: Create Cloud-Init User Data

Run locally.

Cloud-init is the setup script Nebius runs inside the VM the first time the VM
boots.

In this guide, cloud-init does four things:

- Creates the Linux user you will SSH into, using `NEBIUS_VM_USER` from Step 3.
- Adds your SSH public key from Step 4 so you can connect without a password.
- Installs the baseline server packages: Docker, Git, jq, curl, unzip, wget,
  and `binutils`.
- Installs Node.js 22 and pnpm for the builder toolchain.

You do not run this file manually inside the VM. You create it locally, pass it
to Nebius when creating the VM, and Nebius applies it during first boot.

First run this preflight check. It verifies that the VM user and SSH public key
from the previous steps are present. It prints `OK` when the values are ready
and `FAIL` when something needs to be fixed.

```bash
if [ -z "${NEBIUS_VM_USER:-}" ]; then
  echo "FAIL: NEBIUS_VM_USER is not set. Run: export NEBIUS_VM_USER=\"fde\""
elif [ -z "${NEBIUS_SSH_PUBLIC_KEY:-}" ]; then
  echo "FAIL: NEBIUS_SSH_PUBLIC_KEY is not set. Run: export NEBIUS_SSH_PUBLIC_KEY=\"$HOME/.ssh/id_ed25519.pub\""
elif [ ! -r "$NEBIUS_SSH_PUBLIC_KEY" ]; then
  echo "FAIL: SSH public key not found or not readable: $NEBIUS_SSH_PUBLIC_KEY"
else
  NEBIUS_SSH_PUBLIC_KEY_CONTENT="$(head -n 1 "$NEBIUS_SSH_PUBLIC_KEY")"

  case "$NEBIUS_SSH_PUBLIC_KEY_CONTENT" in
    ssh-*|ecdsa-*) echo "OK: cloud-init inputs are ready" ;;
    *) echo "FAIL: file does not look like an SSH public key: $NEBIUS_SSH_PUBLIC_KEY" ;;
  esac
fi
```

Do not continue until this prints `OK: cloud-init inputs are ready`.

Then create the cloud-init file:

```bash
if [ -z "${NEBIUS_SSH_PUBLIC_KEY_CONTENT:-}" ]; then
  echo "FAIL: NEBIUS_SSH_PUBLIC_KEY_CONTENT is not set. Run the Step 5 preflight check first."
else
  cat > /tmp/nebius-fde-cloud-init.yaml <<EOF
#cloud-config
users:
  - name: ${NEBIUS_VM_USER}
    groups: sudo
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ${NEBIUS_SSH_PUBLIC_KEY_CONTENT}
package_update: true
packages:
  - ca-certificates
  - curl
  - git
  - jq
  - binutils
  - unzip
  - wget
  - docker.io
runcmd:
  - systemctl enable --now docker
  - usermod -aG docker ${NEBIUS_VM_USER}
  - curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
  - apt-get install -y nodejs
  - npm install -g pnpm
EOF
  echo "OK: wrote /tmp/nebius-fde-cloud-init.yaml"
fi
```

Quickly inspect the file before continuing:

```bash
sed -n '1,120p' /tmp/nebius-fde-cloud-init.yaml
```

You should see:

- `#cloud-config` at the top.
- A user named whatever `echo "$NEBIUS_VM_USER"` prints, usually `fde`.
- Your SSH public key under `ssh_authorized_keys`.
- Package names like `docker.io`, `jq`, `binutils`, and `git`.

Do not continue if the SSH key line is empty.

Now convert the file into a single JSON string for the Nebius CLI:

```bash
export NEBIUS_CLOUD_INIT="$(jq -Rrs '.' < /tmp/nebius-fde-cloud-init.yaml)"
```

Why this conversion exists:

The next command passes cloud-init through the `--cloud-init-user-data` CLI
flag. The file has multiple lines, so `jq -Rrs '.'` safely turns the whole file
into one escaped string that the CLI can accept.

Verify that the variable is loaded:

```bash
test -n "$NEBIUS_CLOUD_INIT" && echo "Cloud-init user data is loaded"
```

Why Node 22: Webinar 2 will install NemoClaw, and the current NemoClaw
prerequisites require Node.js 22.16+ and npm 10+.

## Step 6: Use the Default Subnet

A subnet is the Nebius network segment where the VM will be attached. Nebius
requires a `subnet_id` in the VM network interface, so the create command needs
one.

For the webinar path, use the project's default subnet. Most projects have one
ready-to-use subnet, so this should be automatic.

Run locally:

```bash
export NEBIUS_SUBNET_ID="$(
  nebius vpc subnet list --format json \
    | jq -r '.items[] | select(.status.state == "READY") | .metadata.id' \
    | head -n 1
)"

if [ -n "$NEBIUS_SUBNET_ID" ]; then
  echo "OK: using subnet $NEBIUS_SUBNET_ID"
else
  echo "FAIL: no READY subnet found in this Nebius project"
fi
```

If this prints `OK`, continue to Step 7.

If it prints `FAIL`, inspect the project subnets:

```bash
nebius vpc subnet list --format json | jq
```

Look for a subnet with:

- `metadata.id` - this is the value you need.
- `metadata.name` - a human-readable name, often something like
  `default-subnet-...`.
- `status.state` - use a subnet in `READY` state.

If there are multiple READY subnets and you need to choose a specific one, copy
its `metadata.id`. It usually starts with `vpcsubnet-`:

```bash
export NEBIUS_SUBNET_ID="<subnet-id>"
```

Verify the value before creating the VM:

```bash
echo "$NEBIUS_SUBNET_ID"
```

If the output is empty, stop and fix the subnet value before continuing. The VM
creation command in Step 7 will fail without a valid subnet.

If `nebius vpc subnet list` returns no subnets, the Nebius project does not have
networking ready yet. Create or request a VPC subnet in the Nebius console
before continuing.

Instructor verification note: the live run used the project's default subnet in
`READY` state.

## Step 7: Create the VM

Run locally.

This creates the VM and its boot disk in one command.

If this command fails during a workshop, stop and inspect the error before
changing flags. The exact available platforms, presets, images, and subnets are
project-specific.

```bash
export NEBIUS_INSTANCE_ID="$(
  nebius compute instance create \
    --name "$NEBIUS_VM_NAME" \
    --parent-id "$NEBIUS_PROJECT_ID" \
    --resources-platform "$NEBIUS_PLATFORM" \
    --resources-preset "$NEBIUS_PRESET" \
    --boot-disk-managed-disk-name "${NEBIUS_VM_NAME}-boot" \
    --boot-disk-managed-disk-type network_ssd \
    --boot-disk-managed-disk-size-gibibytes 50 \
    --boot-disk-managed-disk-source-image-family-image-family ubuntu24.04-driverless \
    --boot-disk-attach-mode READ_WRITE \
    --cloud-init-user-data "$NEBIUS_CLOUD_INIT" \
    --network-interfaces "[{\"name\":\"eth0\",\"ip_address\":{},\"public_ip_address\":{},\"subnet_id\":\"$NEBIUS_SUBNET_ID\"}]" \
    --format json | jq -r '.metadata.id'
)"

echo "$NEBIUS_INSTANCE_ID"
```

Wait a few minutes for the VM and cloud-init setup to finish.

## Step 8: Get the Public IP and SSH In

Run locally.

```bash
nebius compute instance get "$NEBIUS_INSTANCE_ID" --format json \
  > /tmp/nebius-fde-instance.json

jq . /tmp/nebius-fde-instance.json
```

Try to extract the public IP:

```bash
export NEBIUS_VM_IP="$(
  jq -r '.. | objects | .public_ip_address? // empty | .address? // empty' \
    /tmp/nebius-fde-instance.json | head -n 1
)"
export NEBIUS_VM_IP="${NEBIUS_VM_IP%/*}"

echo "$NEBIUS_VM_IP"
```

If that prints an empty line, find the public IP manually in the JSON output,
remove any CIDR suffix such as `/32`, and set it:

```bash
export NEBIUS_VM_IP="<public-ip-address>"
```

Connect:

```bash
ssh "${NEBIUS_VM_USER}@${NEBIUS_VM_IP}"
```

You are now inside the Nebius Cloud server.

## Step 9: Verify the Server Baseline

Run inside the server:

```bash
whoami
uname -a
git --version
docker --version
node --version
npm --version
pnpm --version
jq --version
```

Check Node and npm versions:

```bash
node -e 'const [M,m]=process.versions.node.split(".").map(Number); process.exit(M>22 || (M===22 && m>=16) ? 0 : 1)' \
  && echo "Node version OK"

npm -v | awk -F. '{ exit ($1 >= 10 ? 0 : 1) }' \
  && echo "npm version OK"
```

If Docker says permission denied:

```bash
exit
ssh "${NEBIUS_VM_USER}@${NEBIUS_VM_IP}"
docker ps
```

The fresh SSH session picks up the Docker group membership.

Live install note: NemoClaw also requires `strings`, which is provided by
`binutils` on Ubuntu. If the VM was created before `binutils` was added to
cloud-init, install it manually:

```bash
sudo apt-get update
sudo apt-get install -y binutils
command -v strings
```

## Step 10: Create a Workspace Directory

Run inside the server:

```bash
mkdir -p "$HOME/code"
cd "$HOME/code"
```

## Step 11: Load the Token Factory API Key

Create the API key in the Token Factory console:

https://tokenfactory.nebius.com

Then run inside the server:

Copy this command exactly. Do not replace `NEBIUS_API_KEY` with your actual
key. After you run the command, the terminal will show a prompt; paste the key
at that prompt and press Enter. The key will not be visible as you type.

```bash
read -r -s -p "Paste Nebius Token Factory API key, then press Enter: " NEBIUS_API_KEY
printf "\n"
export NEBIUS_API_KEY

if [ -n "$NEBIUS_API_KEY" ]; then
  echo "OK: NEBIUS_API_KEY is loaded for this shell"
else
  echo "FAIL: NEBIUS_API_KEY is empty"
fi
```

Do not paste the key into the command itself. The command ends with the
variable name `NEBIUS_API_KEY`; the actual key is pasted only after the prompt
appears.

This keeps the key in the current shell session only. It does not write the
secret to the repository.

Choose the NVIDIA model we will use for Pi and Hermes:

```bash
export NEBIUS_TF_MODEL="nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B"
```

This model was selected from the live Token Factory model list because it is the
smallest obvious NVIDIA/Nemotron text model available in the project. Larger
options such as `nvidia/nemotron-3-super-120b-a12b`,
`nvidia/Llama-3_1-Nemotron-Ultra-253B-v1`, and
`nvidia/Nemotron-3-Ultra-550b-a55b` are better candidates for quality demos
after the basic install path is proven.

## Step 12: Install Pi Coding Agent

Run inside the server.

Pi is a VM-level CLI tool, so we use npm here. For project dependencies, use
pnpm.

Official Pi docs:

https://pi.dev/docs/latest

Because this is a disposable Nebius Cloud VM, install global CLI tools at the VM
level with `sudo npm install -g`. Do not use `sudo` for project dependencies.

Do not run `sudo apt install pi`; Ubuntu's `pi` package is unrelated to the Pi
Coding Agent.

```bash
sudo npm install -g --ignore-scripts @earendil-works/pi-coding-agent
hash -r
pi --version
```

## Step 13: Configure Pi for Token Factory

Run inside the server:

```bash
mkdir -p "$HOME/.pi/agent"
```

Create `models.json`:

```bash
cat > "$HOME/.pi/agent/models.json" <<'EOF'
{
  "providers": {
    "nebius-token-factory": {
      "baseUrl": "https://api.tokenfactory.nebius.com/v1/",
      "api": "openai-completions",
      "apiKey": "$NEBIUS_API_KEY",
      "models": [
        {
          "id": "nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B",
          "name": "NVIDIA Nemotron 3 Nano 30B via Nebius Token Factory",
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
EOF

chmod 600 "$HOME/.pi/agent/models.json"
```

Create `settings.json`:

```bash
cat > "$HOME/.pi/agent/settings.json" <<'EOF'
{
  "defaultProvider": "nebius-token-factory",
  "defaultModel": "nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B",
  "defaultThinkingLevel": "off",
  "enabledModels": [
    "nebius-token-factory/nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B"
  ]
}
EOF

chmod 600 "$HOME/.pi/agent/settings.json"
```

Smoke test:

```bash
pi --no-tools --no-context-files --no-session -p \
  "Say the provider and model you are configured to use, then stop."
```

Expected: Pi reports the Nebius Token Factory provider/model. If it reports an
OpenAI model, check for an ambient `OPENAI_API_KEY` and re-check
`~/.pi/agent/settings.json`.

## Step 14: Install GitHub CLI

Run inside the server.

Official install docs:

https://github.com/cli/cli/blob/trunk/docs/install_linux.md

```bash
(type -p wget >/dev/null || (sudo apt update && sudo apt install wget -y)) \
  && sudo mkdir -p -m 755 /etc/apt/keyrings \
  && wget -nv -O- https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
  && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && sudo mkdir -p -m 755 /etc/apt/sources.list.d \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
    | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
  && sudo apt update \
  && sudo apt install gh -y
```

Verify:

```bash
gh --version
```

## Step 15: Authenticate GitHub CLI

Run inside the server:

```bash
gh auth login --web
gh auth status
```

Follow the device/browser login instructions. For the Git protocol, choose SSH
if you want GitHub CLI to help manage SSH auth.

Set Git identity:

```bash
git config --global user.name "<your-name>"
git config --global user.email "<your-email>"
git config --global init.defaultBranch main
```

## Step 16: Install Vercel CLI

Run inside the server.

Official docs:

https://vercel.com/docs/cli

Vercel documents global CLI install options. For this webinar environment, use
npm for VM-level CLI tools and pnpm for project dependencies.

```bash
sudo npm install -g vercel
hash -r
vercel --version
```

## Step 17: Authenticate Vercel CLI

Run inside the server:

```bash
vercel login --github
vercel whoami
```

If browser/device login is awkward on the server, use a temporary token from
your Vercel account:

```bash
read -r -s -p "Paste Vercel token, then press Enter: " VERCEL_TOKEN
printf "\n"
export VERCEL_TOKEN
vercel whoami
```

Do not write the Vercel token into the repository.

## Step 18: Create the Webinar Checkpoint

Run inside the server:

```bash
mkdir -p "$HOME/fde-webinar-checkpoints"
{
  date
  whoami
  hostname
  uname -a
  git --version
  gh --version
  docker --version
  node --version
  npm --version
  pnpm --version
  pi --version
  vercel --version
} > "$HOME/fde-webinar-checkpoints/webinar-1-builder-server.txt"

cat "$HOME/fde-webinar-checkpoints/webinar-1-builder-server.txt"
```

## Done

At this point the Nebius Cloud builder server is ready for the next session.

Webinar 2 starts here and installs Hermes with NemoClaw:

https://github.com/fruteroclub/nebius-nemoclaw-tutorial

## Optional Live Validation: Hermes with NemoClaw

Use this only when validating the server before destroying it.

Run inside the Nebius Cloud server:

```bash
docker ps
command -v strings >/dev/null || {
  sudo apt-get update
  sudo apt-get install -y binutils
}
test -n "$NEBIUS_API_KEY" && echo "NEBIUS_API_KEY is loaded"
```

Install Hermes through NemoClaw:

```bash
TOKEN="${NEBIUS_API_KEY:-}"

if [ -z "$TOKEN" ]; then
  echo "FAIL: NEBIUS_API_KEY is not loaded. Run Step 11 before installing Hermes."
else
  curl -fsSL https://www.nvidia.com/nemoclaw.sh | \
    NEMOCLAW_AGENT=hermes \
    NEMOCLAW_NON_INTERACTIVE=1 \
    NEMOCLAW_ACCEPT_THIRD_PARTY_SOFTWARE=1 \
    NEMOCLAW_REASONING=true \
    NEMOCLAW_PROVIDER=custom \
    NEMOCLAW_ENDPOINT_URL=https://api.tokenfactory.nebius.com/v1/ \
    NEMOCLAW_MODEL="${NEBIUS_TF_MODEL:-nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B}" \
    NEMOCLAW_SANDBOX_NAME=fde-hermes-nebius \
    NEMOCLAW_WEB_SEARCH_PROVIDER=none \
    COMPATIBLE_API_KEY="$TOKEN" \
    NEMOCLAW_YES=1 \
    bash
fi

unset TOKEN
```

Verify:

```bash
export PATH="$HOME/.local/bin:$PATH"

nemohermes --version
nemoclaw --version
nemohermes fde-hermes-nebius status

curl -sS -o /tmp/nemoclaw-api-health.txt -w '%{http_code}\n' \
  http://127.0.0.1:8642/health
```

Verified on the Nebius VM:

- Sandbox: `fde-hermes-nebius`
- Model: `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B`
- Provider: custom OpenAI-compatible endpoint through Nebius Token Factory
- Install duration: 747 seconds
- Installer note: refresh shell PATH before using `nemohermes`:

```bash
source "$HOME/.bashrc"
export PATH="$HOME/.local/bin:$PATH"
```

### Validate Hermes and Token Factory

Run inside the Nebius VM.

Test Token Factory directly:

```bash
curl -sS https://api.tokenfactory.nebius.com/v1/chat/completions \
  -H "Authorization: Bearer $NEBIUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$NEBIUS_TF_MODEL\",\"messages\":[{\"role\":\"user\",\"content\":\"Reply exactly: token-factory-ready\"}],\"max_tokens\":512,\"temperature\":0}" \
  | jq -r '.choices[0].message.content'
```

For NVIDIA/Nemotron reasoning models, a low `max_tokens` value can be exhausted
by the reasoning trace before final content is produced. If
`.choices[0].message.content` is `null`, inspect the full response:

```bash
jq '{finish_reason: .choices[0].finish_reason, content: .choices[0].message.content, reasoning: .choices[0].message.reasoning}' /tmp/tokenfactory-direct.json
```

Verified on the Nebius VM with
`nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B`:

```text
http=200
finish=stop content=token-factory-ready reasoning_chars=285
```

The reasoning trace confirms this is a reasoning model. Keep validation requests
at `max_tokens: 512` or higher unless a non-reasoning mode is available.

Test Hermes status and health:

```bash
nemohermes fde-hermes-nebius status

curl -sS -o /tmp/nemoclaw-api-health.txt -w '%{http_code}\n' \
  http://127.0.0.1:8642/health
```

Test the Hermes OpenAI-compatible gateway:

```bash
HERMES_TOKEN="$(nemohermes fde-hermes-nebius gateway-token --quiet)"

HTTP_CODE=$(curl -sS -o /tmp/nemoclaw-hermes-chat.json -w '%{http_code}' \
  -H "Authorization: Bearer $HERMES_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$NEBIUS_TF_MODEL\",\"messages\":[{\"role\":\"user\",\"content\":\"Reply exactly: hermes-ready\"}],\"max_tokens\":512,\"temperature\":0}" \
  http://127.0.0.1:8642/v1/chat/completions)

unset HERMES_TOKEN

printf 'http=%s\n' "$HTTP_CODE"
jq -r '.choices[0].message.content' /tmp/nemoclaw-hermes-chat.json
```

Verified on the Nebius VM:

```text
http=200
finish=stop content=hermes-ready reasoning_chars=0
```

This validates the Hermes gateway path:

```text
Hermes gateway -> Nebius Token Factory -> NVIDIA Nemotron model -> Hermes response
```

### Chat With Hermes

Terminal chat from inside the Nebius VM:

```bash
source "$HOME/.bashrc"
export PATH="$HOME/.local/bin:$PATH"
nemohermes fde-hermes-nebius connect
```

Browser dashboard from your laptop:

```bash
ssh -L 18789:127.0.0.1:18789 -L 8642:127.0.0.1:8642 "${NEBIUS_VM_USER}@${NEBIUS_VM_IP}"
```

Keep that SSH tunnel open, then open:

```text
http://127.0.0.1:18789/
```

Test sandbox command execution:

```bash
nemohermes fde-hermes-nebius exec -- pwd
nemohermes fde-hermes-nebius exec -- python3 --version
```

## Cleanup: Delete the VM

Official docs:

- Instance delete: https://docs.nebius.com/cli/reference/compute/instance/delete
- Disk delete: https://docs.nebius.com/cli/reference/compute/disk/delete

Nebius CLI docs state that deleting a VM instance also deletes all managed disks
declared in the instance spec. This VM uses a managed boot disk, so deleting the
instance should also delete `fde-builder-01-boot`.

Close SSH sessions and tunnels first. Then run locally:

```bash
export NEBIUS_PROJECT_ID="$(nebius config get parent-id)"

nebius compute instance delete "$NEBIUS_INSTANCE_ID"
```

Verify that no `fde-builder-01` instance remains:

```bash
nebius compute instance list --parent-id "$NEBIUS_PROJECT_ID" --format json \
  | jq -r '.items[]? | [.metadata.id, .metadata.name, (.status.state // "unknown")] | @tsv'
```

If any unmanaged disk was created manually, list disks and delete only the
webinar disk after confirming its ID:

```bash
nebius compute disk list --parent-id "$NEBIUS_PROJECT_ID" --format json \
  | jq -r '.items[]? | [.metadata.id, .metadata.name, (.status.state // "unknown")] | @tsv'
```

## Instructor Verification Log

The July 5, 2026 live run used:

- Profile: `fde-webinar`
- VM name: `fde-builder-01`
- Instance ID: verified, redacted from public guide
- Public IP: verified, redacted from public guide
- Platform/preset: `cpu-e2` / `4vcpu-16gb`
- Subnet: default project subnet, ID redacted from public guide
- Managed boot disk: `fde-builder-01-boot`, 50 GiB

Verified server baseline:

- User: `fde`
- Kernel: Ubuntu 24.04 with `6.11.0-1016-nvidia`
- Git: `2.43.0`
- Docker: `29.1.3`
- Node.js: `22.23.1`
- npm: `10.9.8`
- pnpm: `11.10.0`
- jq: `1.7`
- Node and npm version gates passed.

Verified Hermes/NemoClaw bridge:

- Sandbox: `fde-hermes-nebius`
- Model: `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B`
- Provider: custom OpenAI-compatible endpoint through Nebius Token Factory
- Install duration: 747 seconds
- Direct Token Factory check: `http=200`, `content=token-factory-ready`
- Hermes gateway check: `http=200`, `content=hermes-ready`
- Terminal chat through `nemohermes fde-hermes-nebius connect` worked.

Verified cleanup:

- `nebius compute instance delete "$NEBIUS_INSTANCE_ID"` completed.
- Instance list returned empty.
- Disk list returned empty.
