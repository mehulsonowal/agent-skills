---
name: llm-obs-experiment-bootstrap
description: Bootstrap a reproducible LLM Observability experiment through the Python ddtrace SDK, the Node dd-trace SDK, or the Datadog HTTP control-plane API. Use for experiment, dataset, evaluator, benchmark, regression, or LLM-as-a-judge scaffolding. The legacy Python invocation remains supported.
---

# LLM Observability Experiment Bootstrap

Generate one reproducible experiment artifact. The artifact evaluates a task over a versioned dataset, records outputs and evaluator metrics, carries configuration and provenance, and prints a result link or identifiers when possible.

This skill is intentionally adapter-independent. The adapter-specific API contract lives in `references/` and must be loaded selectively.

## Invocation and compatibility

The installed directory and legacy invocation remain valid:

```text
/agent-observability-experiment-py-bootstrap [--purpose TEXT] [--format py|ipynb]
  [--dataset PATH | --dataset-name NAME] [--dataset-version N]
  [--project-name NAME] [--evaluator-style function|class|remote]
  [--jobs N] [--output PATH] [--task-source module:function]
  [--placeholder-task] [--app-root PATH] [--env-file PATH]
```

General options:

```text
--adapter python|node|http       # default: python
--format py|ipynb|mjs            # Python: py/ipynb; Node: mjs; HTTP: py or mjs
--dataset-id UUID                # HTTP dataset lookup; mutually exclusive with --dataset-name
--site SITE                      # otherwise DD_SITE or datadoghq.com
--concurrency N                  # HTTP client concurrency; never translate to Python jobs or Node run options
```

Do not prompt for optional defaults. Resolve a non-empty purpose from `--purpose`, the request, or a focused question. Keep the purpose as reasoning context, not a fixed taxonomy.

## Mandatory context loading

Load context in this order:

1. Parse the adapter and artifact language.
2. Read exactly one transport reference:
   - Python SDK → `references/python.md`
   - Node SDK → `references/nodejs.md`
   - HTTP API → `references/http.md`
3. If HTTP is the transport, also read the artifact-language reference (`python.md` or `nodejs.md`) when generating executable client code.
4. Read only the selected provider reference under `references/providers/` for Python task generation.
5. Read only the selected evaluator reference under `references/evaluator-styles/`.

Do not load all provider, evaluator, Python, Node, and HTTP references “for completeness.” The selected reference is the source of truth for syntax and API behavior.

## Adapter selection

Use Python when the application or requested artifact is Python, or when no adapter is specified. Use Node when the application is JavaScript/TypeScript and the local `dd-trace` package exposes `tracer.llmobs.experiments`. Use HTTP when the user explicitly requests HTTP, needs a language-neutral transport, or owns task execution outside the SDK.

The transport adapter and artifact language are separate decisions. For example, `--adapter http --format py` means a Python client that executes its task/evaluators locally and sends experiment data through HTTP.

Never mix SDK and HTTP calls in one generated artifact. Do not use private SDK modules or invent a missing symbol/endpoint. If local source and an installed package disagree, report the discrepancy and generate against the selected version.

## Shared experiment model

Every adapter must represent the following concepts:

1. **Project** — resolve an explicit project name, configured service metadata, or a clearly documented generated fallback. Never silently use an unrelated project.
2. **Dataset** — records with input, optional expected output, optional metadata, and tags. Pin a remote dataset version when supplied.
3. **Task** — a deterministic adapter from record input to the application under test. Keep evaluation logic outside the task.
4. **Evaluators** — named row-level or summary-level metrics. Use deterministic checks for contracts and judges only where semantic evaluation is needed.
5. **Run state** — preserve task errors, evaluator errors, completion state, result rows, and partial failures separately.
6. **Provenance** — include purpose, adapter, skill name/version, project, dataset identity/version, task source, evaluator labels/rubrics, model/configuration, Git revision, and generation timestamp.

`expected_output` is optional and must not be synthesized from an observed production output without explicit validation. Distinguish a missing value from an intentionally empty object. Dataset tags must use the backend’s validated `key:value` form where the selected reference requires it.

## Generation workflow

### 1. Resolve purpose and project

Derive the purpose and project without guessing across product boundaries. A project is not automatically the same as an `ml_app`, service, dataset, or repository name. Record how each value was resolved.

### 2. Resolve the dataset

Support:

- inline records;
- local JSON or CSV;
- a named remote dataset and optional version;
- an HTTP dataset ID and optional version; and
- an explicitly approved trace/annotation export.

For local JSON, require a top-level array, validate the selected adapter’s record shape, scrub obvious PII and credential-like values, and report affected record indices. Do not invent canonical or remote record IDs.

For CSV, preserve the runtime path and document the dependency. Use the Python CSV column contract from `references/python.md`; Node and HTTP clients must not pretend that a Python-only CSV helper exists.

### 3. Resolve the task

Use `--task-source` when provided. Otherwise use the selected language’s bounded application discovery rules:

- Python: inspect the resolved app root and rank real callable candidates.
- Node: prefer an explicit import/module function and emit a clearly marked placeholder when absent.
- HTTP: execute the task in the generated client or call an explicitly supplied local endpoint; HTTP does not execute arbitrary task functions for the caller.

Never claim that an invented import or endpoint is wired. Preserve side-effect warnings for network, database, filesystem, environment, or tool calls.

### 4. Select evaluators

Select two or three evaluators based on purpose and available signals. Keep labels unique and stable.

- Accuracy: exact/near match plus a richer rule or judge when needed.
- Tool use: inspect structured tool calls; state the limitation when the task does not expose them.
- Structured output: parse and validate the schema.
- Retrieval: evaluate groundedness only when retrieved context is available.
- Regression: prefer deterministic checks and explicit thresholds.
- Exploration: include diagnostics or taxonomy metrics, not only a pass/fail score.

Evaluator failures must not become passing values. Summary evaluators must remain distinct from row evaluators.

### 5. Emit the artifact

Use the selected adapter reference for the exact generated code. Include:

- purpose and project resolution;
- dataset source and version;
- real task source or a prominent placeholder warning;
- evaluator labels and rubrics;
- configuration and provenance;
- credential instructions without literal secrets; and
- a result URL/ID placeholder and next steps.

Preserve the historical Python section ordering and evaluator/provider reference behavior when using the Python adapter.

### 6. Validate locally

Before presenting the artifact:

- Python `.py`: `python -m py_compile <path>`.
- Python `.ipynb`: parse JSON and require code/markdown cells.
- Node `.mjs`: `node --check <path>`.
- HTTP Python: Python syntax check plus no-network JSON/envelope validation.
- HTTP Node: Node syntax check plus no-network JSON/envelope validation.

For every adapter, check for private imports, literal credentials, malformed tags, missing provenance, mismatched dataset versions, fabricated IDs, and task/evaluator errors that were collapsed into false or pass.

### 7. Report completion

Use this compact structure:

```text
Generated LLM Observability experiment: <adapter>/<format>
Path: <path>
Purpose: "<purpose>"
Project: <project>
Dataset: <local path | name | id>, version=<version or latest>
Task: <wired source | placeholder>
Evaluators: <labels>
Provenance: generated_by=claude-code, adapter=<adapter>, skill=llm-obs-experiment-bootstrap
Validation: <commands and pass/fail>
Result link: <URL or pending until run>

Next steps:
1. Verify the task source and evaluator semantics.
2. Set the credentials required by the selected reference.
3. Install the selected SDK or run the HTTP artifact.
4. Review per-row errors before treating metrics as a successful run.
```

## Safety and uncertainty

- Do not modify application source code unless explicitly asked.
- Do not write credentials into generated files or artifacts.
- Do not publish prompts, outputs, traces, datasets, or evaluations without explicit user approval.
- Do not use production data as ground truth without labeling and validation.
- Do not use an internal `dd-source` service adapter as if it were a public client contract.
- Do not retry non-idempotent writes automatically unless the selected reference explicitly supports it.
- On partial publication, preserve IDs and failed rows and provide a reconciliation path.

## Reference maintenance

Each adapter reference must identify the local source files and branch used to verify it. Re-check the reference when the SDK/API version changes. The local `dd-trace-py` and `dd-source` checkouts currently use `main` rather than a local `master` ref; use their default `main` branch as the source of truth. `dd-trace-js` currently exposes the experiment API on its local `master` branch.

Keep shared workflow guidance here and adapter syntax in the references. If a detail is only true for one transport, do not duplicate it in this file.

## Existing references

- `references/python.md` — Python `ddtrace.llmobs` API and legacy Python compatibility.
- `references/nodejs.md` — Node `tracer.llmobs.experiments` API.
- `references/http.md` — HTTP control-plane wire contract and limitations.
- `references/providers/` — provider-specific task/environment guidance; load one as needed.
- `references/evaluator-styles/` — function, class, or remote evaluator guidance; load one as needed.

Do not modify `dd-trace-py`, `dd-trace-js`, or `dd-source` while updating this skill.
