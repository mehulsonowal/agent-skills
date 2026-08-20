---
name: llm-obs-experiment-bootstrap
description: Bootstraps a reproducible LLM Observability experiment using the Python ddtrace SDK, the Node dd-trace SDK, or the Datadog HTTP control-plane API. The legacy /llm-obs-experiment-py-bootstrap path and Python defaults remain supported. Use for experiment, dataset, evaluator, benchmark, regression, or LLM-as-a-judge scaffolding.
---

# LLM Observability Experiment Bootstrap

Generate one self-contained experiment artifact. The artifact evaluates a task over a versioned dataset, records outputs and evaluator metrics, carries configuration and provenance, and prints a link or identifiers when possible. Choose one adapter:

- **Python SDK** (`python`, default): `ddtrace.llmobs`; preserves the historical behavior of this skill.
- **Node SDK** (`node`): `tracer.llmobs.experiments` from the local `dd-trace` package.
- **HTTP API** (`http`): direct JSON:API calls to the LLM Observability control plane. The task and evaluator run in the generated client; HTTP only stores datasets, experiment metadata, spans, and metrics.

Do not modify `dd-trace-js` or `dd-source`. Do not create another builder skill. This file is the implementation contract for all three adapters.

## Invocation and compatibility

The installed skill directory remains `.agents/skills/llm-obs-experiment-py-bootstrap`, and the old invocation remains valid:

```text
/llm-obs-experiment-py-bootstrap [--purpose TEXT] [--format py|ipynb]
  [--dataset PATH | --dataset-name NAME] [--dataset-version N]
  [--project-name NAME] [--evaluator-style function|class|remote]
  [--jobs N] [--output PATH] [--task-source module:function]
  [--placeholder-task] [--app-root PATH] [--env-file PATH]
```

General invocation adds:

```text
--adapter python|node|http       # default: python
--format py|ipynb|mjs            # python defaults to py; node defaults to mjs; HTTP defaults to py
--dataset-id UUID                # HTTP only; mutually exclusive with --dataset-name
--site SITE                      # otherwise DD_SITE or datadoghq.com
--concurrency N                  # HTTP only; do not translate to Node's run options
```

`--language typescript` and other unsupported language requests must fail clearly; use `--adapter node` for a JavaScript/ESM artifact. Keep accepting every legacy Python flag. For Python, default `--format py`, `--jobs 10`, output `./experiments/experiment.py`, and preserve the existing dataset, purpose, bounded introspection, provider-key, PII-scrub, evaluator-style, `.env` loading, syntax-check, and next-steps behavior. `--adapter node` and `--adapter http` do not use Python-only evaluator classes or `jobs`.

Never block on optional defaults. Resolve a non-empty purpose: use `--purpose`, extract a clear purpose from the request, or ask the existing purpose question. Use purpose as reasoning context for task selection and evaluator semantics, not as a rigid taxonomy.

## Shared experiment model

Every adapter must model these concepts:

1. **Project**: a named LLM Observability project. Resolve `--project-name`, then package/service metadata, then `experiment-<slug>`. Never silently use an unrelated project.
2. **Dataset**: records with `input`, optional `expected_output`, optional `metadata`, and optional tags. For local JSON, validate and scrub PII before embedding. For local CSV, preserve an absolute runtime path and document the dependency. For a remote dataset, pin `dataset_version` when supplied.
3. **Task**: a deterministic adapter from the dataset input to the application under test. Default Python behavior introspects the bounded app tree and wires a real function; `--placeholder-task` is the explicit fallback. Node and HTTP generation should use `--task-source` when available and otherwise emit a clearly marked task function, never claim that an invented import is real.
4. **Evaluators**: named row metrics; use deterministic checks for regression and inline judges/rules for quality. HTTP metrics must use one of backend-supported metric types: `boolean`, `score`, `categorical`, `json`, or `text`.
5. **Run state**: record each task outcome, evaluator outcome/error, and final completion/failure. A task failure must not be mistaken for a passing evaluator result.
6. **Configuration and provenance**: include model/provider/configuration plus `generated_by: claude-code`, `purpose`, `adapter`, `skill`, source revision/path when known, dataset name/id/version, and generation timestamp. Put provenance in the API's configuration and tags/metadata paths where those paths exist.
7. **Reproducibility**: preserve the exact dataset source, project, purpose, task source, evaluator names/rubrics, adapter, site, and pinned version. Do not put API keys in generated files. Do not fabricate dataset IDs, evaluator names, or backend features.

## Backend selection and decision rules

Use Python SDK when the application is Python, the user wants the existing notebook/script style, or no adapter was specified. Use Node SDK when the application is JavaScript/TypeScript and the installed `dd-trace` exposes `tracer.llmobs.experiments`. Use HTTP when the user explicitly requests HTTP, needs a language-neutral client, or needs externally-driven task execution. Prefer an SDK over HTTP when it covers the requested behavior: SDKs handle project resolution, dataset operations, experiment event formation, and SDK-specific failures.

Never mix APIs in one generated artifact. In particular, do not call HTTP endpoints to compensate for missing Python/Node SDK features without stating the adapter switch. Do not use private Python or Node modules. Do not assume a `jobs`/parallelism option exists across adapters.

## Python SDK adapter (legacy default)

### Public surface and canonical shape

Use only public imports from `ddtrace.llmobs`:

```python
import os
from ddtrace.llmobs import LLMObs, EvaluatorResult

LLMObs.enable(
    api_key=os.getenv("DD_API_KEY"),
    app_key=os.getenv("DD_APPLICATION_KEY") or os.getenv("DD_APP_KEY"),
    site=os.getenv("DD_SITE", "datadoghq.com"),
    project_name="<project>",
    agentless_enabled=True,
)

dataset = LLMObs.create_dataset(
    dataset_name="<dataset>",
    description="<description>",
    records=[{"input_data": {"prompt": "..."}, "expected_output": "...", "metadata": {}, "tags": ["split:eval"]}],
)

experiment = LLMObs.experiment(
    name="<name>", dataset=dataset, task=task_fn, evaluators=[exact_match],
    config={"model": "<model>", "temperature": 0.0,
            "generated_by": "claude-code", "skill": "llm-obs-experiment-py-bootstrap",
            "adapter": "python", "purpose": "<purpose>"},
    description="<purpose>",
    tags={"generated_by": "claude-code", "skill": "llm-obs-experiment-py-bootstrap",
          "adapter": "python", "purpose": "<purpose>"},
)
experiment.run(jobs=10)
print(experiment.url)
```

The public Python surface includes `LLMObs.enable`, `create_dataset`, `create_dataset_from_csv`, `pull_dataset`, `experiment`, `async_experiment`, `RemoteEvaluator`, `EvaluatorContext`, `BaseEvaluator`, `EvaluatorResult`, and `LLMJudge` as available in the installed SDK. Keep the prior `--evaluator-style function|class|remote` router and load only the selected reference under `references/evaluator-styles/`. Rich evaluators should return `EvaluatorResult(value=..., reasoning=..., assessment=..., metadata=..., tags=...)`; bare `bool` is acceptable only for trivial checks.

Required Datadog credentials are `DD_API_KEY` and either `DD_APPLICATION_KEY` or `DD_APP_KEY`; `DD_SITE` defaults to `datadoghq.com`. Load `.env` without requiring `python-dotenv` using `scripts/env_setup_template.py`; shell values win. Assert only provider keys needed by the discovered task, using the matching provider reference. Preserve the existing provider references and generated section sequence (header, env, enable, dataset, task, evaluators, experiment, run, result inspection).

### Legacy Python compatibility contract

The old Python workflow remains normative: `--purpose` is resolved before task/evaluator generation; absent a purpose, use the existing five seed choices (output accuracy, tool-call correctness, structured output/schema, retrieval/faithfulness, regression/refactor) plus Other. Purpose biases candidate ranking, wrapper return shape, evaluator semantics, header, config, tags, and next steps. Bounded introspection searches Python under the resolved `--app-root` while excluding tests, `.gitignore` paths, environments, build artifacts, and vendor trees; it recognizes OpenAI, Anthropic, LiteLLM, LangChain, LlamaIndex, Gemini/Vertex, Bedrock, and existing LLMObs/workflow/agent decorators. Present the top three candidates and wire the confirmed function; do not replace a discovered candidate with a placeholder silently.

Keep the old wrapper adaptation rules: map one input key to a one-argument function, map named keys for multi-argument functions, pass a dict through for dict/`**kwargs` functions, pass config values when accepted, and wrap async functions with `asyncio.run` unless the whole artifact is async. Preserve side-effect warnings for external HTTP, databases, environment reads, and file writes. Read only the selected provider and evaluator-style reference. Emit provider-specific asserts from the `{{PROVIDER_ASSERTS}}` substitution in `scripts/env_setup_template.py`, not a re-derived loader. Python JSON records are PII-scrubbed and embedded; CSV remains a runtime file; remote datasets use public `LLMObs.pull_dataset` by name/version. Python SDK artifacts must not contain private imports, literal credentials, generated record IDs, or direct HTTP requests.

Dataset rules remain backward compatible: `--dataset` and `--dataset-name` are mutually exclusive; JSON is validated, scrubbed, and embedded; CSV uses `create_dataset_from_csv`; `--dataset-name` uses `LLMObs.pull_dataset(dataset_name=..., project_name=..., version=...)`; no source means the runnable three-record sample. Per-record tags are `key:value` strings, never bare strings. Do not generate `record_id`/`canonical_id` fields or direct HTTP calls. Keep the existing app-root/introspection blocklist and side-effect warnings. `--jobs` maps to `experiment.run(jobs=N)`.

### Python workflow examples

- **Inline accuracy**: default adapter, no dataset flags, `exact_match` plus a richer deterministic evaluator, then `experiment.run(jobs=10)`.
- **Pinned remote regression**: `--dataset-name qa_v3 --dataset-version 4 --purpose "regression test prompt v3"`; call `pull_dataset(..., version=4)` and prefer deterministic/near-match evaluators.
- **Notebook**: `--format ipynb --evaluator-style remote`; one markdown/code pair per existing section and `RemoteEvaluator` names only after confirming they exist in the organization.

## Node SDK adapter (dd-trace-js)

This section is grounded in the local source, not inferred API names. Consult these evidence paths before emitting or changing Node code:

- `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/src/llmobs/experiments/index.js`
- `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/src/llmobs/experiments/dataset.js`
- `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/src/llmobs/experiments/experiment.js`
- `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/src/llmobs/experiments/client.js`
- `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/test/llmobs/experiments/example.js`
- `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/src/llmobs/sdk.js`

The actual public entry point is `tracer.llmobs.experiments`. Configure the tracer with `tracer.init({ llmobs: { mlApp: projectName } })`; `mlApp` or `service` is used as the project name. The source's `llmobs.enable()` is deprecated for dd-trace 7 and is not the preferred generated setup. Experiments require `DD_API_KEY`, `DD_APP_KEY`, a configured site, and LLM Observability enabled. The local example reads `DD_API_KEY`, `DD_APP_KEY`, and optional `DD_SITE`, then calls `tracer.init`.

The supported calls and argument shapes are:

```js
const tracer = require('dd-trace')
tracer.init({ llmobs: { mlApp: PROJECT_NAME } })
const { experiments } = tracer.llmobs

const dataset = experiments.createDataset(DATASET_NAME, {
  description: '...',
  records: [{ inputData: { prompt: '...' }, expectedOutput: '...', metadata: {}, tags: ['split:eval'] }],
})
// or use the returned Dataset's fluent addRecord(input, expectedOutput, metadata, tags)

const pulled = await experiments.pullDataset(DATASET_NAME, {
  version: DATASET_VERSION, expectedRecordCount: N, tags: ['split:eval'], maxWaitMs: 30000,
})

const experiment = experiments.experiment({
  name: EXPERIMENT_NAME,
  dataset,
  task: async (input, config, metadata) => ({ output: await runTask(input), metadata }),
  evaluators: {
    exact_match: (input, output, expected) => output === expected,
  },
  summaryEvaluators: { /* optional functions over rows */ },
  description: PURPOSE,
  config: { model: MODEL, generated_by: 'claude-code', adapter: 'node', purpose: PURPOSE },
  tags: { generated_by: 'claude-code', adapter: 'node', purpose: PURPOSE },
})
const result = await experiment.run({ maxRetries: 0, retryDelay: attempt => 100 * (attempt + 1), throwOnErrors: false })
console.log(result.url, result.experimentId, result.rows)
```

Important Node details evidenced by source:

- `createDataset(name, descriptionOrOptions)` accepts a string description or an options object with `description` and `records`. Record properties are camelCase (`inputData`, `expectedOutput`, `metadata`, `tags`); the backend payload becomes `input`, `expected_output`, `metadata`, and optional `tags`.
- `Dataset.addRecord` is fluent and accepts `(input, expectedOutput?, metadata?, tags?)`; `push`, `id`, `url`, `records`, `recordIds`, and tag/update/delete helpers are available on the Dataset.
- `pullDataset(name, options)` accepts `version`, `tags`, `expectedRecordCount`, and `maxWaitMs`, paginates records, and retries eventually-consistent reads. It takes a dataset **name**, not a dataset ID.
- `experiment(options)` requires `name`, `dataset`, and `task` for local runs. `evaluators` is an array or an object keyed by evaluator name. Evaluators receive `(record.input, output, record.expectedOutput)`; task receives `(record.input, config, record.metadata)`.
- `run` is sequential in this local source and accepts `maxRetries`, `retryDelay`, and `throwOnErrors`; it does **not** accept Python's `jobs` option. Preserve `--jobs` only for Python, and warn/omit it for Node.
- `config`, `tags`, `metadata`, `description`, `ensure_unique`, and `run_count` are distinct experiment options. The source sends config as `config` and object tags as metadata tags; keep provenance in both `config` and `tags`.
- Dataset record tags must be non-empty `key:value` strings (`validateTagsList`). Do not supply bare labels. Dataset record IDs are generated by the Node DatasetRecord when omitted; do not invent IDs for normal SDK input.
- The Node SDK client uses `https://api.<site>`, `DD-API-KEY`, and `DD-APPLICATION-KEY` headers, with dashboard links derived from the site. Do not manually duplicate this HTTP client in a Node SDK artifact.

### Node workflow examples

- **JavaScript task**: `--adapter node --task-source app/answer.js:answer`; import the task in the generated `task` wrapper, create records with `inputData`, and run `await experiment.run()`.
- **Pinned dataset**: `await experiments.pullDataset('qa_v3', { version: 4 })`; pass the resulting Dataset into `experiment`.
- **Externally driven execution**: if the user's framework owns task execution, the local source exposes `startExperiment(options)`, then `submitSpan(input)`, `submitEvaluationMetrics(span, metrics)`, and `close(options)`. Use this only when explicitly requested; it is not the normal local `run` path.

## HTTP API adapter

This section is grounded in these local source paths:

- `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/src/llmobs/experiments/client.js` (wire format, host/site mapping, headers, endpoint paths)
- `/Users/mehul.sonowal/dd/dd-source/domains/ml-observability/apps/apis/llm-obs/bootstrap.go` (registered `/api/v2/llm-obs/v1` routes)
- `/Users/mehul.sonowal/dd/dd-source/domains/ml-observability/apps/apis/llm-obs/internal/adapters/handlersv2/http/experiment.go` (create and event request fields/validation)
- `/Users/mehul.sonowal/dd/dd-source/domains/ml-observability/apps/apis/llm-obs/internal/adapters/handlersv2/http/dataset.go` (dataset/record request fields)
- `/Users/mehul.sonowal/dd/dd-source/domains/ml-observability/apps/apis/llm-obs/internal/core/domain/experiment.go` and `eval_metric.go` (span and metric fields/types)

### HTTP authentication and host

Use the explicit site, defaulting to `DD_SITE` or `datadoghq.com`, and send both headers on every control-plane request:

```text
DD-API-KEY: ${DD_API_KEY}
DD-APPLICATION-KEY: ${DD_APPLICATION_KEY:-$DD_APP_KEY}
Content-Type: application/json
```

The local Node client maps `datadoghq.com` to `https://api.datadoghq.com`; it uses `https://api.<site>` for other sites and derives the UI host separately. Do not use `DD_APP_KEY` as the literal header name. Do not hardcode keys. Validate credentials before making a request and redact them from error output.

### HTTP routes and JSON:API envelope

The public v2 routes use `/api/v2/llm-obs/v1`, JSON:API `{ "data": { "type": ..., "attributes": ... } }`, and UUID project/dataset/experiment path parameters. The actual route set includes:

```text
POST /api/v2/llm-obs/v1/projects
POST /api/v2/llm-obs/v1/{project_id}/datasets
GET  /api/v2/llm-obs/v1/{project_id}/datasets?filter[name]=...
POST /api/v2/llm-obs/v1/{project_id}/datasets/{dataset_id}/records
GET  /api/v2/llm-obs/v1/{project_id}/datasets/{dataset_id}/records
POST /api/v2/llm-obs/v1/experiments
POST /api/v2/llm-obs/v1/experiments/{experiment_id}/events
PATCH /api/v2/llm-obs/v1/experiments/{experiment_id}
GET  /api/v2/llm-obs/v1/experiments/{experiment_id}/events
```

Create a project with attributes `{ "name": "<project>" }`. Create a dataset with `{ "name": "<dataset>", "description": "...", "metadata": {...} }`. The source handler creates a project-scoped dataset at the project route. Append records with `{ "records": [{ "input": ..., "expected_output": ..., "metadata": ... }] }`; `id` is optional and the backend generates record IDs when omitted. The v2 source dataset handler models record `tags` and `tag_operations`; preserve tags as `key:value` strings and do not claim support for any other tag syntax.

Create an experiment with attributes matching `CreateExperimentRequest`:

```json
{
  "data": {
    "type": "experiments",
    "attributes": {
      "project_id": "<project-uuid>",
      "dataset_id": "<dataset-uuid>",
      "dataset_version": 4,
      "name": "qa-v3",
      "description": "<purpose>",
      "ensure_unique": true,
      "run_count": 1,
      "config": {"model": "...", "adapter": "http", "purpose": "...", "generated_by": "claude-code"},
      "metadata": {"tags": ["adapter:http", "generated_by:claude-code", "purpose:<slug>"]}
    }
  }
}
```

The server can generate the experiment ID/name and validates `name`; `ensure_unique` controls unique run naming. The handler's `IngestExperimentEventsRequest` accepts top-level `spans` and/or `metrics` and rejects a request with neither. A generated HTTP client should execute each task locally, then post one span and its metrics (or batches) to the events endpoint. It must preserve `dataset_record_id` on the span input; the handler remaps it to a tag.

A minimal span uses the domain fields evidenced in `experiment.go`:

```json
{
  "spans": [{
    "span_id": "<client-generated-span-id>",
    "trace_id": "<client-generated-trace-id>",
    "dataset_record_id": "<record-id>",
    "name": "task",
    "start_ns": 1700000000000000000,
    "duration": 1234567,
    "status": "ok",
    "meta": {
      "span": {"kind": "experiment"},
      "input": {"prompt": "..."},
      "output": "...",
      "expected_output": "...",
      "metadata": {"source": "local"}
    },
    "tags": ["adapter:http"]
  }],
  "metrics": [{
    "timestamp_ms": 1700000000123,
    "trace_id": "<same-trace-id>",
    "span_id": "<same-span-id>",
    "metric_type": "boolean",
    "label": "exact_match",
    "boolean_value": true,
    "assessment": "pass",
    "reasoning": "...",
    "tags": ["adapter:http"],
    "metric_source": "custom"
  }]
}
```

`ExperimentSpanEvent` requires `span_id`, `start_ns`, `duration`, `status` (`ok` or `error`), and `meta`; `meta.span.kind` is set to `experiment` by the handler. `EvalMetricBase` requires `timestamp_ms`, `metric_type`, and `label`; use exactly one typed value field matching the metric type. Use `status`/`error` for evaluator failures rather than silently posting a false value. Generate span/trace correlation IDs only for event correlation; never put generated IDs in dataset records or claim they are canonical dataset IDs.

HTTP has no local SDK `run()` that executes tasks, no Python `jobs`, and no Node evaluator function serialization. Evaluators must run in the generated client, and only metric values/reasoning/assessment are sent. Managed/custom evaluator configuration and inference endpoints exist in the source under `/api/unstable/llm-obs/...`, but they are separately gated and their request shapes are not interchangeable with v2 experiment events. Do not emit those endpoints unless the user explicitly asks for them and the exact handler/request type has been inspected.

### HTTP workflow examples

- **Language-neutral local run**: create/get project, create dataset, append records, create experiment, execute each task, post spans + boolean/score metrics, PATCH status if needed, then print `https://<app-site>/llm/experiments/<id>`.
- **Existing dataset**: use `--dataset-id` and optionally `--dataset-version`; fetch records through the project-scoped records route before execution. If only a name is known, list datasets with `filter[name]` and verify exactly one accessible dataset; do not guess an ID.
- **External framework**: create the experiment and use its event endpoint as the sink. Preserve one stable dataset record ID per row and correlate every metric to the same span/trace IDs.

## Generation workflow (all adapters)

1. Parse adapter and legacy flags. Reject unsupported format/adapter combinations and mutually exclusive dataset flags.
2. Resolve project and purpose. Record the resolution source in the generated header.
3. Resolve dataset. For local JSON, require a top-level array and `input_data`/`input` shape as appropriate; scrub emails, phones, SSNs, and API-key-like strings and report affected record indices. Preserve dataset version/name/id in provenance.
4. Resolve task. For Python, run the existing bounded introspection and candidate ranking; for Node/HTTP, use explicit `module:function` when possible. Never emit a fake “wired” source. If no source exists, emit a placeholder with a prominent replacement warning.
5. Select 2–3 evaluators based on purpose. Accuracy: exact match plus a richer rule/judge. Tool-use: inspect structured tool calls when the task returns them; otherwise explain the limitation. RAG: groundedness rubric only when retrieved context is available. Schema: parse/validate shape. Regression: deterministic exact/near-match; avoid an LLM judge.
6. Emit the adapter-specific artifact with stable section banners and provenance. Include the exact source path, generation timestamp, dataset version, evaluator labels/rubrics, and configuration. Never embed secrets.
7. Validate locally before presenting output:
   - Python: `python -m py_compile <path>`; notebook: parse JSON and require code/markdown cells.
   - Node: `node --check <path>` for CommonJS/ESM syntax; do not require network credentials for syntax validation.
   - HTTP Python: `python -m py_compile <path>` and a dry-run JSON/envelope validation that performs no network call.
   - For all adapters, grep generated output for private imports, literal keys, malformed tags, missing provenance, and mismatched dataset version.
8. Failure handling: fail fast on missing credentials/invalid input before writes; preserve per-row task/evaluator errors; use SDK retry options only where evidenced; for HTTP retry only idempotent reads and use an explicit request timeout. On partial event failure, report experiment ID and failed row IDs, do not mark completion, and provide a rerun/reconciliation step.
9. Print next steps including install, credentials, run command, experiment URL/ID, task source, dataset provenance, evaluator caveats, and unresolved uncertainties.

## Generated artifact invariants

Every generated artifact must:

- Keep `purpose`, adapter, project, dataset identity/version, source, evaluator labels, and provenance visible.
- Read credentials from environment or a discoverable `.env`; never include actual secrets.
- Preserve input/expected output/metadata without inventing dataset IDs. For Python/Node SDK records, respect each SDK's actual camelCase/raw record shape and tag validation.
- Distinguish missing expected output from an empty output. Do not treat task errors as evaluator passes.
- Print or return an experiment URL/ID and a concise result summary.
- Avoid private imports (`ddtrace.llmobs._...` or Node internal modules), manual JSON:API calls in SDK adapters, and guessed symbols/endpoints.
- Keep Python provenance key `skill: llm-obs-experiment-py-bootstrap` for legacy Python output so existing downstream searches remain compatible. New Node/HTTP artifacts use `skill: llm-obs-experiment-bootstrap`; if a caller explicitly asks for legacy provenance, preserve it while also setting `adapter`.

## Evidence and uncertainty discipline

Consult local source before documenting a symbol or wire field. Current evidence confirms Node SDK project/dataset/experiment APIs, v2 HTTP project/dataset/record/experiment/event routes, and v2 HTTP record tag fields listed above. It does **not** establish a public HTTP endpoint that executes arbitrary task functions, a universal HTTP dataset-name pull endpoint, or cross-adapter evaluator serialization. Treat those as unresolved unless the target deployment/source is inspected. If a local source snapshot and installed package disagree, report both paths and generate against the selected version; do not silently merge contracts.

## Completion output

Use this concise structure after generation:

```text
Generated LLM Observability experiment: <adapter>/<format>
Path: <path>
Purpose: "<purpose>"
Project: <project>
Dataset: <local path | name | id>, version=<version or latest>
Task: <wired source | placeholder>
Evaluators: <labels>
Provenance: generated_by=claude-code, adapter=<adapter>, skill=<skill>
Validation: <commands and pass/fail>
Result link: <URL or pending until run>

Next steps:
1. Verify the task source and evaluator semantics.
2. Set DD_API_KEY and DD_APPLICATION_KEY/DD_APP_KEY plus DD_SITE when non-US1.
3. Install the adapter dependency and run the generated artifact.
4. Review per-row errors before treating metrics as a successful run.
```

Do not commit changes. Do not modify `dd-trace-js` or `dd-source`.

## Evidence paths consulted for this refactor

- Existing skill: `/Users/mehul.sonowal/.agents/skills/llm-obs-experiment-py-bootstrap/SKILL.md`
- Node SDK implementation/example: `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/src/llmobs/experiments/{index.js,dataset.js,experiment.js,client.js}`, `/Users/mehul.sonowal/go/src/github.com/DataDog/dd-trace-js/packages/dd-trace/test/llmobs/experiments/example.js`, and `src/llmobs/sdk.js`.
- HTTP routes/handlers/domain: `/Users/mehul.sonowal/dd/dd-source/domains/ml-observability/apps/apis/llm-obs/bootstrap.go`, `internal/adapters/handlersv2/http/{experiment.go,dataset.go}`, `internal/adapters/handlers/unstable/http/evaluator_config.go`, `internal/core/domain/{experiment.go,eval_metric.go}`.
