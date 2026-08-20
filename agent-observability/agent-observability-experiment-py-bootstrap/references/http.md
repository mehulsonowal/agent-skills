# HTTP adapter reference

This file is the HTTP transport contract for `llm-obs-experiment-bootstrap`. Load it for `--adapter http`. Also load `python.md` or `nodejs.md` when the generated HTTP client is executable Python or Node.js.

## Scope and source of truth

The HTTP adapter executes the task and evaluators locally, then stores the dataset, experiment metadata, spans, and evaluation metrics through the Datadog LLM Observability control plane. HTTP does not execute arbitrary task functions or serialize evaluator functions on behalf of the client.

The wire contract is checked against these public sources:

- https://github.com/DataDog/dd-trace-js/blob/master/packages/dd-trace/src/llmobs/experiments/client.js — control-plane host, authentication, JSON:API envelope, routes, span/metric submission, and dashboard URLs;
- https://github.com/DataDog/dd-trace-js/blob/master/packages/dd-trace/src/llmobs/experiments/experiment.js — experiment attributes and event span/metric fields;
- https://github.com/DataDog/dd-source/blob/main/domains/ml_observability/apps/apis/lux/internal/adapters/llmobs/projects.py — current project read model and filters; and
- https://github.com/DataDog/dd-source/blob/main/domains/ml_observability/apps/apis/lux/internal/adapters/llmobs/datasets.py — current dataset/record read and create model.

The LUX adapter uses an internal forwarded JWT transport and is not a public generated-client contract. Do not import or call that internal Python adapter from a user artifact. Use the API-key control-plane contract below.

## Authentication and hosts

Read credentials from the environment:

```text
DD_API_KEY
DD_APPLICATION_KEY or DD_APP_KEY
DD_SITE (default: datadoghq.com)
```

For the public control-plane client, send both headers on every request:

```text
DD-API-KEY: <DD_API_KEY>
DD-APPLICATION-KEY: <DD_APPLICATION_KEY or DD_APP_KEY>
Content-Type: application/json when a body is present
```

The current Node client maps a site to:

```text
API: https://api.<site>
UI:  https://app.<site> for single-level sites; the site itself for regional sites
```

Follow the site mapping in the selected public client rather than hardcoding `datadoghq.com`. Use a request timeout, redact credentials from errors, retry only idempotent reads, and do not blindly retry dataset/experiment writes.

## JSON:API envelope

Control-plane requests use:

```json
{
  "data": {
    "type": "<resource-type>",
    "attributes": { }
  }
}
```

The current Node client sends JSON:API envelopes for project, dataset, experiment, event, and status operations. Preserve the exact `type` and attributes shape; do not send a bare attributes object.

## Stable project and dataset routes

Base path:

```text
/api/v2/llm-obs/v1
```

Project operations:

```text
POST /api/v2/llm-obs/v1/projects
```

Body:

```json
{
  "data": {
    "type": "projects",
    "attributes": {"name": "<project>"}
  }
}
```

The current SDK uses this operation as get-or-create and caches the resulting project ID. Do not assume a project name is a dataset name.

Dataset operations:

```text
POST /api/v2/llm-obs/v1/{project_id}/datasets
GET  /api/v2/llm-obs/v1/{project_id}/datasets?filter[name]=...
GET  /api/v2/llm-obs/v1/{project_id}/datasets/{dataset_id}/records
POST /api/v2/llm-obs/v1/{project_id}/datasets/{dataset_id}/records
POST /api/v2/llm-obs/v1/{project_id}/datasets/{dataset_id}/batch_update
```

Create dataset attributes include `name` and optional `description`. The record-create request uses JSON:API type `datasets` and attributes such as:

```json
{
  "records": [
    {
      "input": {"prompt": "..."},
      "expected_output": {"answer": "..."},
      "metadata": {"source": "local"},
      "tags": ["split:eval"],
      "id": "optional-client-record-id"
    }
  ],
  "create_new_version": true
}
```

The current `dd-source` main model confirms `input`, optional `expected_output`, optional `metadata`, `tags`, optional `id`, `valid_from_version`, and `valid_to_version`. Record listing supports `filter[id]`, repeated `filter[tags]`, `filter[canonical_id]`, `filter[version]`, `page[limit]`, and `page[cursor]`. Dataset writes are versioned by default; report the resulting version.

## Experiment routes and attributes

Create an experiment:

```text
POST /api/v2/llm-obs/v1/experiments
```

Current SDK attributes:

```json
{
  "name": "qa-v3",
  "project_id": "<project-uuid>",
  "dataset_id": "<dataset-uuid>",
  "dataset_version": 4,
  "description": "<purpose>",
  "ensure_unique": true,
  "run_count": 1,
  "config": {
    "adapter": "http",
    "purpose": "<purpose>",
    "generated_by": "claude-code"
  },
  "metadata": {
    "tags": ["adapter:http", "generated_by:claude-code"]
  }
}
```

Use exact UUIDs returned by project/dataset lookup or creation. Never infer them from names or generate them as dataset record IDs.

Post experiment events:

```text
POST /api/v2/llm-obs/v1/experiments/{experiment_id}/events
```

The JSON:API attributes contain `spans` and/or `metrics`; do not send an empty event request:

```json
{
  "data": {
    "type": "experiments",
    "attributes": {
      "spans": [ ... ],
      "metrics": [ ... ]
    }
  }
}
```

Update status:

```text
PATCH /api/v2/llm-obs/v1/experiments/{experiment_id}
```

Attributes are `{ "status": "completed" }` or `{ "status": "failed", "error": "..." }`. Preserve partial-write IDs and failed rows when a status update or event post fails.

## Span and metric fields

A span represents one task row. The current Node client emits fields such as:

```json
{
  "span_id": "<client-generated-span-id>",
  "trace_id": "<client-generated-trace-id>",
  "project_id": "<project-uuid>",
  "dataset_id": "<dataset-uuid>",
  "name": "task",
  "start_ns": 1700000000000000000,
  "duration": 1234567,
  "status": "ok",
  "meta": {
    "input": {"prompt": "..."},
    "output": {"answer": "..."},
    "expected_output": {"answer": "..."},
    "metadata": {"source": "local"}
  },
  "tags": [
    "experiment_id:<experiment-uuid>",
    "dataset_record_id:<record-id>"
  ]
}
```

For task errors, use `status: "error"` and `meta.error` with type/message/stack where available. Keep generated span/trace IDs only for event correlation; they are not canonical dataset IDs.

A metric correlates to the same span and trace:

```json
{
  "metric_source": "custom",
  "label": "exact_match",
  "span_id": "<same-span-id>",
  "trace_id": "<same-trace-id>",
  "timestamp_ms": 1700000000123,
  "experiment_id": "<experiment-uuid>",
  "metric_type": "boolean",
  "boolean_value": true,
  "tags": ["adapter:http"]
}
```

Use exactly one typed value field:

- `boolean` → `boolean_value`
- numeric score → `score_value`
- structured value → `json_value`
- categorical/text value → `categorical_value`

For evaluator failure, send an error-bearing metric or preserve the local error and omit the metric according to the current client behavior. Never send a false value merely because the evaluator failed.

## HTTP execution rules

1. Resolve project and dataset IDs explicitly.
2. Create or select a pinned dataset version.
3. Create the experiment with config and metadata provenance.
4. Execute each task locally, recording start time, duration, output, and error.
5. Execute evaluators locally and correlate each metric to the row span/trace.
6. Post spans and metrics in bounded batches.
7. Patch completion or failure status.
8. Print the UI URL using the configured site and experiment ID.

The HTTP adapter must not:

- call internal LUX adapters or forward a JWT as if it were a public API key;
- assume a server-side task executor exists;
- serialize Python/Node evaluator functions into JSON;
- send top-level `spans`/`metrics` when the selected client expects JSON:API attributes;
- retry non-idempotent writes without an idempotency strategy; or
- hide failed rows behind a successful HTTP status.
