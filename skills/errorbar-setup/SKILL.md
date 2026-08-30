---
name: errorbar-setup
description: Wire this repository to errorbar (AI gateway + judge-calibrated evals). Inspects the repo, picks the right integration door (gateway, BYOK, OpenTelemetry env vars, or the tracing SDK), makes the edits, and verifies with a receipt. Use when the user wants to connect an app to errorbar, capture LLM traffic for evaluation, or compare models on production traffic.
---

# errorbar setup

You are integrating this repository with errorbar (LLM quality measurement:
calibrated judges, gates and monitoring on your own traffic, captured through
an OpenAI-compatible gateway or telemetry). Ask the user only for the API key if it is
not already in the environment as `ERRORBAR_API_KEY`.

The canonical, always-current version of these instructions is:

```
curl -fsSL https://www.errorbar.ai/agent-setup.md
```

If you can fetch it, prefer it over the copy below (this skill may lag it).

## 1. Inspect the repo, then pick ONE door

Look at which LLM provider(s) and frameworks this repo uses, then choose:

| This repo… | Door |
| --- | --- |
| Calls an OpenAI-compatible API and can use errorbar's model catalog | **A — gateway** |
| Uses a closed provider (OpenAI / Anthropic / Gemini / xAI / Mistral) it must KEEP | **B — gateway with the provider's own key (BYOK)**, or **C/D** if adding a request-path hop is unacceptable |
| Already emits OpenTelemetry traces (e.g. Vercel AI SDK telemetry, existing OTel SDK) | **C — OTLP env vars** |
| No OTel, and inference must not move | **D — errorbar tracing SDK** |

Whichever door: never hardcode the key. Add `ERRORBAR_API_KEY` to the
environment/secrets the same way this repo handles its existing provider keys,
and add it to `.env.example` if one exists.

## Door A — route inference through the gateway

- Point the OpenAI-compatible client at errorbar:
  - base URL: `https://gateway.errorbar.ai/v1`
  - API key: from `ERRORBAR_API_KEY` (starts with `sk_`)
- Model names pass through unchanged (e.g. `openai/gpt-oss-120b`).
- If the code sets per-request headers, add on inference calls:
  - `X-Errorbar-Tag`: short stable label for this traffic
    (`[a-zA-Z0-9_:.-]`, max 64 chars). One tag per distinct kind of traffic —
    tags become evaluation populations; mixing unrelated traffic blends them.
  - `X-Errorbar-Trace-Id`: only if the repo has a per-run id and makes multiple
    model calls per run — same id on every call of one run. Skip it if W3C
    `traceparent` propagation is already active; the gateway reads that.

## Door B — keep the current model, via BYOK

The repo keeps its exact provider and model; only the base URL changes. The
provider bills the customer's own provider key; errorbar bills zero for those
tokens and captures the traffic for measurement.

- Ask the user to paste the provider's API key into **errorbar → Settings →
  Workspace → Provider keys**. Do NOT handle that key yourself, and never put
  it in the repo.
- Point the client at `https://gateway.errorbar.ai/v1` with
  `ERRORBAR_API_KEY` (as Door A), and prefix the model with its provider:
  `openai/gpt-4o`, `anthropic/claude-sonnet-4-5`, `gemini/gemini-2.5-pro`.
- Add the same `X-Errorbar-Tag` / trace headers as Door A.
- A gateway 401 `provider_key_rejected` means the provider key in workspace
  settings is bad/rotated — that is the user's to fix; say so.

## Door C — already speaking OpenTelemetry: three env vars

No packages, no code. Add to the deployment environment:

```
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://gateway.errorbar.ai/v1/traces
OTEL_EXPORTER_OTLP_TRACES_HEADERS=Authorization=Bearer ${ERRORBAR_API_KEY}
OTEL_RESOURCE_ATTRIBUTES=errorbar.tag=<short-stable-label>
```

Vercel AI SDK repos: also ensure telemetry is on
(`experimental_telemetry: { isEnabled: true }`). errorbar ingests OTLP/HTTP in
protobuf or JSON and understands the official `gen_ai.*` conventions plus the
Traceloop, Vercel `ai.*`, and OpenInference attribute dialects.

## Door D — no OTel yet: the errorbar tracing SDK

One package, one line, standard OpenTelemetry underneath (eject documented —
removing it later loses nothing):

- Node (CommonJS): `npm install @error-bar/tracing`, then
  `require("@error-bar/tracing").setup()` once at startup, BEFORE any LLM client
  is constructed.
- Node (pure ESM): same install, no code — start the app with
  `node --import @error-bar/tracing/register`.
- Python: `pip install errorbar-tracing`, then
  `from errorbar_tracing import setup; setup()` at startup before clients.
- Set `ERRORBAR_API_KEY` and `ERRORBAR_TAG=<short-stable-label>` in the environment.
- Captures OpenAI, Anthropic, Gemini (Python), and LangChain automatically —
  only libraries actually installed are instrumented.
- **If the registry doesn't have the package yet** (rollout, or a registry
  outage): do NOT stop — wire the identical setup from standard ecosystem
  packages. Node: install `@opentelemetry/sdk-node`,
  `@opentelemetry/exporter-trace-otlp-proto`, and the
  `@traceloop/instrumentation-{openai,anthropic,langchain}` packages, then
  start a `NodeSDK` with an `OTLPTraceExporter` pointed at
  `https://gateway.errorbar.ai/v1/traces` with header
  `Authorization: Bearer ${ERRORBAR_API_KEY}` and
  `OTEL_RESOURCE_ATTRIBUTES=errorbar.tag=<label>`. Python: install
  `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-http`, and the
  `opentelemetry-instrumentation-{openai,anthropic,google-generativeai,langchain}`
  packages, build a `TracerProvider` + `BatchSpanProcessor(OTLPSpanExporter)`
  with the same endpoint/header, and call each installed library's
  `Instrumentor().instrument(tracer_provider=...)`. Identical spans either
  way — the package is convenience, not a dependency.

For Doors C and D, note in your summary: model-call CONTENT
(prompts/completions) is stored only if the workspace has request logging
enabled; span structure is kept either way. Batches are capped at 4 MB — keep
the exporter's `max_export_batch_size` default.

## 2. Verify — never report success without this receipt

```
ERRORBAR_API_KEY=$ERRORBAR_API_KEY sh -c "$(curl -fsSL https://www.errorbar.ai/setup.sh)"
```

It proves the key works, reports logging/traffic/grades/judge progress, and
names the ONE next step. For Doors A/B it makes one tiny test completion
(fractions of a cent); add `-- --no-inference` if this key must not spend.

## Optional: the platform as tools (MCP)

If the user asks for their agent to run bake-offs, read gates, or inspect
judge alignment from the editor, register the errorbar MCP server — every
platform API operation becomes a tool:

```
claude mcp add errorbar -e ERRORBAR_API_KEY=$ERRORBAR_API_KEY -- npx -y @error-bar/mcp
```

(Any MCP client: command `npx`, args `["-y", "@error-bar/mcp"]`, env
`ERRORBAR_API_KEY`.) Prefer a key with only the scopes the agent needs; add
`--read-only` or `--no-spend` to keep it from starting billable work. Only on
request — it is not part of capturing traffic.

## 3. After traffic flows

Do these in order, each only once the previous one has something to work on.
Every call below is the management API at `https://gateway.errorbar.ai/v1`
with `Authorization: Bearer $ERRORBAR_API_KEY`
(reference: https://docs.errorbar.ai/api-reference/introduction).

1. If the setup check reported request logging OFF, ask the user to turn it
   on (errorbar → Settings → Workspace → Request logging). Without it only
   structure is kept and there is nothing to grade; do not try to change
   this setting yourself.
2. Once real traffic has been logged, run a screening: `POST /v1/evals` with
   `{ "screening": true }` and nothing else. It takes the dominant model in
   recent traffic as the incumbent, its stored answers as the baseline, and
   the cheapest model of each family as candidates. Report the verdict per
   candidate (verified-better / tied-but-cheaper / inconclusive / worse) with
   the win-rate interval — never recommend on a point estimate.
3. Create one judge: `POST /v1/criteria` with ONE binary question about the
   single quality property that matters most in this codebase. Not a rubric,
   not a score.
4. Once about 30 real exchanges are graded (the user's job, in the dashboard;
   tell them when the queue is ready), calibrate it:
   `POST /v1/criteria/{id}/align`. Report the true-positive rate, the
   true-negative rate, whether the judge is trustworthy — and if not, how many
   more grades of which class it needs.

Do not report any pass rate without the interval around it.

## 4. Report

Tell the user: which door you chose and why, every file you changed, the tag
you picked, and the setup check's reported "next step". Do not print the API
key anywhere. If anything failed, say exactly which step and stop rather than
improvising around it.
