# Python adapter reference

This file is the Python transport contract for `llm-obs-experiment-bootstrap`. Load it only when the Python SDK is the transport or when an HTTP artifact is being emitted in Python.

## Source of truth

The public `dd-trace-py` repository exposes the experiment API on its `main` branch. The syntax below was checked against:

- https://github.com/DataDog/dd-trace-py/blob/main/ddtrace/llmobs/_llmobs.py
- https://github.com/DataDog/dd-trace-py/blob/main/ddtrace/llmobs/_experiment.py
- https://github.com/DataDog/dd-trace-py/blob/main/ddtrace/llmobs/__init__.py
- https://github.com/DataDog/dd-trace-py/blob/main/tests/llmobs/test_experiments.py

Use public imports only. Never import `ddtrace.llmobs._experiment`, `_llmobs`, or other private modules in generated artifacts.

## Setup

```python
import os
from ddtrace.llmobs import LLMObs

LLMObs.enable(
    api_key=os.getenv("DD_API_KEY"),
    app_key=os.getenv("DD_APPLICATION_KEY") or os.getenv("DD_APP_KEY"),
    site=os.getenv("DD_SITE", "datadoghq.com"),
    project_name="<project>",
    agentless_enabled=True,
)
```

`LLMObs.enable` accepts `project_name`, `site`, `api_key`, `app_key`, `agentless_enabled`, and the tracing/service options shown in `_llmobs.py`. The generated artifact should read credentials from the environment or an approved `.env` loader and must never embed literal keys.

## Dataset API

Use keyword arguments so generated code is resilient to positional-order mistakes.

```python
records = [
    {
        "input_data": {"prompt": "What is 2 + 2?"},
        "expected_output": "4",
        "metadata": {"source": "synthetic"},
        "tags": ["split:eval"],
    },
]

dataset = LLMObs.create_dataset(
    dataset_name="<dataset>",
    project_name="<project>",
    description="<description>",
    records=records,
    bulk_upload=False,
    deduplicate=True,
)

pinned = LLMObs.pull_dataset(
    dataset_name="<dataset>",
    project_name="<project>",
    version=4,
    tags=["split:eval"],
)
```

Current signatures:

```python
LLMObs.pull_dataset(
    dataset_name: str,
    project_name: str | None = None,
    version: int | None = None,
    tags: list[str] | None = None,
) -> Dataset

LLMObs.create_dataset(
    dataset_name: str,
    project_name: str | None = None,
    description: str = "",
    records: list[DatasetRecordNew] | None = None,
    bulk_upload: bool = False,
    deduplicate: bool = True,
) -> Dataset
```

New records use `input_data`, optional `expected_output`, `metadata`, `tags`, and optional user-defined `id`. Records returned from a remote dataset use `record_id`; do not fabricate that field for new records. Tags must be non-empty `key:value` strings.

`Dataset` exposes `append(record)`, `extend(records)`, `push(deduplicate=True, create_new_version=True, bulk_upload=None)`, and tag/update/delete helpers. `push()` mutates the remote dataset and may create a new version. Explain that operation before generating it.

For CSV input, use the actual helper rather than inventing a generic CSV loader:

```python
LLMObs.create_dataset_from_csv(
    csv_path="./data/eval.csv",
    dataset_name="<dataset>",
    input_data_columns=["prompt"],
    expected_output_columns=["answer"],
    metadata_columns=["source"],
    csv_delimiter=",",
    description="<description>",
    project_name="<project>",
    deduplicate=True,
    id_column=None,
)
```

`expected_output_columns`, `metadata_columns`, and `id_column` are optional. Preserve a runtime CSV path rather than embedding CSV contents unless the user explicitly requests conversion.

## Experiment API

The synchronous factory has this public shape:

```python
from ddtrace.llmobs import LLMObs


def task(input_data, config, metadata=None):
    return run_application(input_data, config, metadata)


def exact_match(input_data, output_data, expected_output):
    return output_data == expected_output


def aggregate(inputs, outputs, expected_outputs, evaluators_results):
    return sum(bool(value) for value in evaluators_results["exact_match"]) / len(outputs)

experiment = LLMObs.experiment(
    name="<experiment>",
    task=task,
    dataset=dataset,
    evaluators=[exact_match],
    description="<purpose>",
    project_name="<project>",
    tags={"adapter": "python", "generated_by": "claude-code"},
    config={"purpose": "<purpose>", "model": "<model>"},
    summary_evaluators=[aggregate],
    runs=1,
)

result = experiment.run(
    jobs=10,
    raise_errors=False,
    sample_size=None,
    max_retries=0,
)
print(result)
```

Current factories:

```python
LLMObs.experiment(
    name, task, dataset, evaluators, description="", project_name=None,
    tags=None, config=None, summary_evaluators=None, runs=1,
) -> SyncExperiment

LLMObs.async_experiment(
    name, task, dataset, evaluators, description="", project_name=None,
    tags=None, config=None, summary_evaluators=None, runs=1,
) -> Experiment
```

Function evaluators receive `(input_data, output_data, expected_output)`. Summary evaluators receive `(inputs, outputs, expected_outputs, evaluators_results)`. Class evaluators must use the public `BaseEvaluator`, `BaseAsyncEvaluator`, `BaseSummaryEvaluator`, or `BaseAsyncSummaryEvaluator` contracts from `ddtrace.llmobs`.

The sync wrapper exposes `run(jobs=1, raise_errors=False, sample_size=None, max_retries=0, retry_delay=None)`. Async `Experiment.run` defaults to `jobs=10`; both accept a callable `retry_delay(attempt)`. Do not map Node's sequential run to Python's `jobs` behavior without stating the difference.

Task and evaluator errors remain distinct from false values. `raise_errors=False` preserves row-level errors in the result; use `raise_errors=True` only when the caller requests fail-fast behavior.

## Evaluator results and remote evaluators

For diagnostic context, use the public `EvaluatorResult`:

```python
from ddtrace.llmobs import EvaluatorResult

def quality(input_data, output_data, expected_output):
    score = 1.0 if output_data == expected_output else 0.0
    return EvaluatorResult(
        value=score,
        reasoning="Exact comparison",
        assessment="pass" if score else "fail",
        metadata={"criterion": "exact_match"},
        tags={"category": "accuracy"},
    )
```

`RemoteEvaluator`, `EvaluatorContext`, `LLMJudge`, and provider-specific classes are public exports, but load their dedicated reference only when selected. Do not claim a remote evaluator exists by name without checking the organization/backend.

## Python compatibility rules

Preserve the legacy behavior of this skill:

- default adapter is Python;
- default format is `.py`, with optional `.ipynb`;
- `--jobs` maps only to `experiment.run(jobs=N)`;
- local JSON is validated and PII-scrubbed before embedding;
- `--dataset` and `--dataset-name` are mutually exclusive;
- no dataset produces a small runnable inline sample;
- task discovery is bounded to `--app-root`, excludes tests/build/vendor trees, and never silently replaces a discovered callable with a placeholder; and
- the generated artifact retains the purpose, provider, evaluator style, dataset identity, and next-step sections used by the legacy Python workflow.
