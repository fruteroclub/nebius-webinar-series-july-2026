# Webinar 1: Nebius Cloud Builder Environment

This directory contains the student-facing artifacts for Webinar 1 of the July
2026 Nebius FDE Trainer webinar series.

Webinar 1 provisions the cloud machine that will carry the rest of the series.
By the end, the student has a Nebius Cloud VM, SSH access, a basic builder
toolchain, and Pi Coding Agent configured to use Nebius Token Factory for
debugging and tech support.

Hermes and NemoClaw are not the teaching focus of Webinar 1. The guide includes
an optional live validation section because the July 5 dry run proved the Nebius
server can install Hermes with NemoClaw and route an NVIDIA Nemotron model
through Nebius Token Factory. The full Hermes/NemoClaw tutorial remains Webinar
2 and should reference:

https://github.com/fruteroclub/nebius-nemoclaw-tutorial

## Files

| File | Purpose | Use first when |
| --- | --- | --- |
| [written-guide.md](written-guide.md) | Student-facing companion guide for provisioning the Nebius Cloud server, configuring Pi with Token Factory, and optionally validating Hermes. | You want the actual tutorial steps. |
| [video-tutorial-script.md](video-tutorial-script.md) | Draft narration and recording flow for Webinar 1. | You are preparing or recording the video. |
| [local-testing-research/](local-testing-research/) | Evidence from the local laptop validation: Pi, Token Factory, NemoClaw, Hermes, and sandbox behavior. | You need to understand what was already tested locally. |

## Webinar 1 Outcome

The expected outcome is a working remote builder server:

- Nebius CLI configured on the student's laptop.
- Nebius Cloud VM created with SSH access.
- Basic server tools installed: Git, Docker, Node/npm, pnpm, jq, curl, unzip.
- Pi Coding Agent installed on the server.
- Pi configured to use Nebius Token Factory through an OpenAI-compatible model
  provider.
- API key handled without pasting secrets into chat or committing secrets to the
  repository.
- Optional proof that Hermes/NemoClaw can run on the VM and route through
  Nebius Token Factory.

## Boundary

Webinar 1 stops when the server is ready for Hermes/NemoClaw.

The optional validation proves readiness. Webinar 2 teaches the Hermes/NemoClaw
installation in full.

## Dry Run Status

Verified on July 5, 2026:

- Nebius CLI federation profile and VM provisioning.
- `cpu-e2` / `4vcpu-16gb` VM with a managed 50 GiB boot disk.
- SSH, Docker, Node 22, npm 10, pnpm, jq, and `binutils`.
- Nebius Token Factory direct inference with
  `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B`.
- Hermes/NemoClaw optional validation and terminal chat.
- VM deletion cleanup; instance and disk lists returned empty.

Not re-run before deleting the dry-run VM:

- Pi install/config on the Nebius VM.
- GitHub CLI auth.
- Vercel CLI auth.
