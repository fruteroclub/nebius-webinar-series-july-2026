# Nebius FDE Trainer Webinar Series

Working repository for the July 2026 Nebius webinar series artifacts.

For now, keep this repo guide-first and avoid adding structure before the
materials require it.

The public repository contains the student-facing artifacts and supporting
research notes. Internal planning lives in the private Nebius workstream.

## Current Files

| File | Purpose |
| --- | --- |
| [01-nebius-cloud-builder-environment/README.md](01-nebius-cloud-builder-environment/README.md) | Webinar 1 artifact index. |
| [01-nebius-cloud-builder-environment/written-guide.md](01-nebius-cloud-builder-environment/written-guide.md) | Student-facing Nebius Cloud server provisioning guide. |
| [01-nebius-cloud-builder-environment/video-tutorial-script.md](01-nebius-cloud-builder-environment/video-tutorial-script.md) | Draft Webinar 1 recording script. |
| [01-nebius-cloud-builder-environment/local-testing-research/](01-nebius-cloud-builder-environment/local-testing-research/) | Local validation notes for Pi, Token Factory, NemoClaw, Hermes, and sandbox behavior. |
| `.gitignore` | Protects local secrets and generated files. |

## Working Rules

- Do not commit secrets, API keys, private tokens, or real server credentials.
- Add directories only when a real artifact needs them.
- Optimize for participant time-to-smile: the guide should explain tools, not
  make students fight setup.
- Mark unverified commands as `TODO verify` until tested on the target Nebius
  environment.
