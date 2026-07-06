# Nebius Webinar Series July 2026

Public artifact repository for the July 2026 Nebius webinar series.

The series is the opener for a Builder Growth content program focused on
Forward Deployed Engineering. The build-along project is an agent that helps
builders train to become Forward Deployed Engineers: set up infrastructure,
debug real environments, integrate tools, ship useful workflows, and eventually
monetize the result.

## Current Status

Webinar 1 is drafted and ready for Nebius DevRel review.

What is already in this repository:

- Student-facing written guide for provisioning a Nebius Cloud builder server.
- 15-20 minute video tutorial script with timing, architecture diagram,
  documentation/dashboard cues, narration beats, and editing notes.
- Local and cloud dry-run notes for Pi Coding Agent, Nebius Token Factory,
  NemoClaw/NemoHermes, and Hermes Agent readiness.
- Public-safe cleanup of live Nebius resource identifiers before publication.

Live dry run completed on July 5, 2026:

- Nebius CLI federation profile.
- Nebius Cloud VM provisioning on `cpu-e2` / `4vcpu-16gb`.
- SSH access and baseline toolchain verification.
- Nebius Token Factory direct inference with
  `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B`.
- Optional Hermes/NemoClaw bridge validation through Token Factory.
- Terminal chat with Hermes.
- VM cleanup verified with empty instance and disk lists.

## Series Arc

| Webinar | Working Title | Core Outcome | Nebius/Product Focus |
| --- | --- | --- | --- |
| 1 | Nebius Cloud Builder Environment | Provision the builder server and install Pi, Token Factory config, GitHub, and Vercel. | Nebius Cloud, Token Factory |
| 2 | Hermes with NemoClaw | Install and operate Hermes Agent inside the Nebius environment. | Nebius Cloud, Token Factory, NemoClaw |
| 3 | FDE Trainer Agent | Add personality, skills, search, and integrations to turn Hermes into an FDE trainer. | Token Factory, Tavily by Nebius |
| 4 | Money and Runtime | Add payments, automations, and worker/runtime deployment for the agent. | Nebius Cloud Sandboxes or worker runtime, payment stack TBD |

## Repository Map

| Path | Purpose |
| --- | --- |
| [01-nebius-cloud-builder-environment/README.md](01-nebius-cloud-builder-environment/README.md) | Webinar 1 overview for reviewers and contributors. |
| [01-nebius-cloud-builder-environment/written-guide.md](01-nebius-cloud-builder-environment/written-guide.md) | Student-facing companion guide with commands and verification steps. |
| [01-nebius-cloud-builder-environment/video-tutorial-script.md](01-nebius-cloud-builder-environment/video-tutorial-script.md) | 15-20 minute recording plan for the video tutorial. |
| [01-nebius-cloud-builder-environment/local-testing-research/](01-nebius-cloud-builder-environment/local-testing-research/) | Research notes and local validation history. Not the main student path. |

The repository is intentionally guide-first. New directories should be added
only when a real artifact exists.

## Review Path For Nebius DevRel

Start here:

1. Read [Webinar 1 README](01-nebius-cloud-builder-environment/README.md) for
   the current review snapshot.
2. Skim [video-tutorial-script.md](01-nebius-cloud-builder-environment/video-tutorial-script.md)
   to review pacing, positioning, and product explanations.
3. Review [written-guide.md](01-nebius-cloud-builder-environment/written-guide.md)
   for technical accuracy and command safety.

Specific feedback requested:

- Is the Nebius Cloud + Token Factory positioning accurate?
- Should the demo recommend a different VM shape, image, or setup path?
- Are there preferred Nebius docs, console screens, or calls to action to show?
- Are there any sponsor, credits, or Fellow-program requirements that should be
  reflected before recording?
- Should the optional Hermes/NemoClaw readiness proof stay in Webinar 1, or be
  moved entirely to Webinar 2?

## Technical Notes

- The Webinar 1 VM is CPU-only. Model inference goes through Nebius Token
  Factory, so a GPU server is not required for this first environment setup.
- The guide uses `pnpm` for project-level dependencies. `npm` is used only for
  global CLI installs when the tool's docs expect it.
- API keys are loaded into the current shell for the tutorial and are not
  committed to the repository.
- Live cloud resource IDs and public IP addresses are redacted from public docs.

## Related Links

- Nebius Fellows: https://nebius.com/fellows
- Nebius Fellows terms: https://nebius.com/fellows/terms-and-conditions
- Nebius Compute docs: https://docs.nebius.com/compute/
- Nebius Token Factory docs: https://docs.tokenfactory.nebius.com/quickstart
- Pi Coding Agent docs: https://pi.dev/docs/latest
- NemoClaw docs: https://docs.nvidia.com/nemoclaw/latest/
- Existing Hermes/NemoClaw tutorial reference:
  https://github.com/fruteroclub/nebius-nemoclaw-tutorial

## Security And Publishing Rules

- Do not commit secrets, API keys, private tokens, or real server credentials.
- Do not commit screenshots that expose account, billing, tenant, project, or
  token information.
- Keep generated videos, raw recordings, screenshots, exports, and local caches
  out of git unless explicitly reviewed for publication.
