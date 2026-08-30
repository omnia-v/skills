# errorbar agent skills

Skills that let a coding agent (Claude Code, Cursor, or anything that reads
the [SKILL.md convention](https://code.claude.com/docs/en/skills)) integrate a
repository with [errorbar](https://www.errorbar.ai) — LLM quality
measurement on your own traffic: calibrated judges, gates, monitoring.

## Install

```bash
npx skills add omnia-v/skills --skill errorbar-setup
```

Or manually: copy `skills/errorbar-setup/` into your repo's `.claude/skills/`.

## What `errorbar-setup` does

Inspects your repo and wires ONE of four integration doors:

| Door | For | Change |
| --- | --- | --- |
| **A — Gateway** | OpenAI-compatible apps using errorbar's catalog | base-URL swap |
| **B — BYOK** | Apps keeping GPT / Claude / Gemini | base-URL swap + provider key in workspace settings; provider keeps billing you |
| **C — OpenTelemetry** | Apps already emitting OTel | three env vars, zero code |
| **D — Tracing SDK** | No OTel, inference stays put | `@error-bar/tracing` / `errorbar-tracing`, one line |

Every run ends with the setup check — a receipt proving traffic actually
lands, not a claim that it should. The always-current instruction source is
[`agent-setup.md`](https://www.errorbar.ai/agent-setup.md); the skill
prefers it when it can fetch it, so installed copies don't rot.

## Renamed

The skill was `omnia-setup` until 2026-08-30. Re-run the install command above to pick up `errorbar-setup`; the old copy keeps working but no longer receives updates.

## Privacy

The skill never handles provider API keys (Door B routes them to workspace
settings, entered by the human), never prints `ERRORBAR_API_KEY`, and model-call
content is stored only if the workspace has request logging enabled — span
structure otherwise.
