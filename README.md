# Omnia agent skills

Skills that let a coding agent (Claude Code, Cursor, or anything that reads
the [SKILL.md convention](https://code.claude.com/docs/en/skills)) integrate a
repository with [Omnia](https://platform.omnia-voice.com) — the AI gateway
with judge-calibrated evals.

## Install

```bash
npx skills add omnia-v/skills --skill omnia-setup
```

Or manually: copy `skills/omnia-setup/` into your repo's `.claude/skills/`.

## What `omnia-setup` does

Inspects your repo and wires ONE of four integration doors:

| Door | For | Change |
| --- | --- | --- |
| **A — Gateway** | OpenAI-compatible apps using Omnia's catalog | base-URL swap |
| **B — BYOK** | Apps keeping GPT / Claude / Gemini | base-URL swap + provider key in workspace settings; provider keeps billing you |
| **C — OpenTelemetry** | Apps already emitting OTel | three env vars, zero code |
| **D — Tracing SDK** | No OTel, inference stays put | `@omnia/tracing` / `omnia-tracing`, one line |

Every run ends with the setup check — a receipt proving traffic actually
lands, not a claim that it should. The always-current instruction source is
[`agent-setup.md`](https://platform.omnia-voice.com/agent-setup.md); the skill
prefers it when it can fetch it, so installed copies don't rot.

## Privacy

The skill never handles provider API keys (Door B routes them to workspace
settings, entered by the human), never prints `OMNIA_API_KEY`, and model-call
content is stored only if the workspace has request logging enabled — span
structure otherwise.
