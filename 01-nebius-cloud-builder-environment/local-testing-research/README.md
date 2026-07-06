# Local Testing Research: Pi, Token Factory, NemoClaw, and Hermes

This directory preserves the local validation work completed before the
student-facing Webinar 1 path was moved back to Nebius Cloud provisioning.

This is research evidence, not the public Webinar 1 sequence. The public Webinar
1 files live one directory up and now focus on Nebius Cloud server provisioning,
Pi Coding Agent, and Nebius Token Factory.

## Files

| File | Purpose | Use first when |
| --- | --- | --- |
| [written-guide.md](written-guide.md) | Local companion guide covering Token Factory key setup, Pi provider config, NemoClaw/Hermes, and the sandbox checks. | You need the local test procedure. |
| [video-tutorial-script.md](video-tutorial-script.md) | Local narration and recording flow for explaining the tech while building. | You need to reconstruct what was tested locally. |
| [01-nebius-cloud-builder-environment.md](01-nebius-cloud-builder-environment.md) | Working implementation notes and verified command log. | You need source details, exact observed outputs, or unresolved setup notes. |

## Current Status

Confirmed locally:

- Pi Coding Agent is installed and configured for Nebius Token Factory.
- Docker is running.
- NemoClaw/NemoHermes installed Hermes in sandbox `fde-hermes-local`.
- Hermes dashboard is reachable at `http://127.0.0.1:18789/`.
- Hermes OpenAI-compatible API health returns `200`.
- The webinar repo is uploaded into the sandbox.
- `/sandbox/workspace` is the canonical sandbox repo path.
- The sandbox can read `README.md` and `01-nebius-cloud-builder-environment.md`.

Known unresolved issue:

- Hermes chat through the current custom provider route does not execute tools
  reliably. It returns plain text with `tool_turns=0`.
- Until that gate passes, do not ask Hermes to inspect or edit files. Use
  `nemohermes ... exec -- <command>` for shell evidence.

## Recording Order

1. Open `video-tutorial-script.md`.
2. Keep `written-guide.md` open beside the terminal.
3. Use `01-nebius-cloud-builder-environment.md` only when you need the raw
   command history or troubleshooting detail.

## Canonical Sandbox Path

Use this path in the rest of Webinar 1:

```text
/sandbox/workspace
```

Verify it:

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
