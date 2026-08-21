---
name: agent-observability-local-experiment
description: Plan, generate, validate, run, compare, and optionally publish a reproducible LLM Observability experiment from a local coding-agent workflow. Use for exploration, offline paired comparisons, regression gates, production-failure reproduction, cost/performance benchmarks, or preparing a validated harness for iterative optimization.
---

# LLM Observability Local Experiment

Turn a local LLM application or code change into a reproducible experiment with an explicit goal, versioned cases, task adapter, evaluators, metrics, provenance, and next actions.

This skill is an orchestration and design contract. It should prevent experiments that run successfully but cannot answer a meaningful product or engineering question.

Use the generalized `agent-observability-experiment-bootstrap` skill to emit Python or Node adapter-specific experiment artifacts. Do not duplicate SDK API contracts here; this skill selects the adapter and defines the experiment plan around it. HTTP task endpoints may be used when an existing local runner is supplied, but HTTP artifact construction is deferred from this first iteration.

## Core rules

1. Establish the decision the experiment must support before generating code.
2. Select an explicit experiment profile; do not default to an unbounded benchmark.
3. Use the same immutable CaseSet and evaluator suite for comparable variants.
4. Treat `expected_output` as optional and only use it when the expected behavior is known.
5. Keep evaluator measurements separate from gate policies.
6. Collect operational metrics—errors, duration, tokens, and cost—whenever available.
7. Record Git, dataset, evaluator, model, configuration, and adapter provenance.
8. Run a single-case smoke test before the full suite.
9. Default to local execution. Publishing or production-data access requires confirmation.
10. Never silently modify application source code, checkout over the current branch, or write secrets into artifacts.

## When to use this skill

Use it when the user asks to:

- create an experiment from a local code change;
- compare a branch, prompt, model, or agent configuration;
- build an offline evaluation suite;
- create a regression or CI gate;
- reproduce a production LLM failure;
- benchmark cost, latency, token use, or reliability;
- explore production behavior and identify follow-up hypotheses; or
- prepare a reliable harness for iterative optimization.

Use `agent-observability-auto-experiment` for the iterative hill-climb itself after this skill has produced and validated a fixed benchmark and scoring contract.

## First iteration scope

This first iteration captures all six RFC experiment profiles as explicit planning and execution policies:

| Profile | First-iteration behavior |
| --- | --- |
| Exploration / discovery | Run a representative sample and return findings, clusters, and hypotheses rather than one pass/fail score. |
| Offline paired comparison | Run baseline and candidate against the same immutable cases and report per-case and aggregate deltas. |
| Regression gate | Require a pinned CaseSet, deterministic checks, thresholds, and a CI-friendly decision. |
| Failure reproduction | Preserve trace/annotation provenance, reproduce the failure, and optionally compare a fix. |
| Optimization | Produce a fixed benchmark and scoring contract, then hand off iterations to `agent-observability-auto-experiment`. |
| Cost/performance benchmark | Measure latency, tokens, cost, throughput, and reliability with quality guardrails. |

Python and Node artifact construction is delegated to `agent-observability-experiment-bootstrap`. Local execution is the default. Datadog publication, production-data access, and autonomous optimization remain explicit confirmation points.

## Workflow

### Phase 0: Discover the repository and execution context

Inspect the repository before asking the user to provide information that can be derived locally:

- project instructions (`AGENTS.md`, `CLAUDE.md`, and equivalent files);
- Git branch, commit, merge base, and working-tree diff;
- application entry points and existing tests;
- package manifests and installed SDKs;
- existing datasets, fixtures, evaluators, and experiment scripts; and
- local commands for running the application.

Infer the candidate from the current worktree. Select a baseline from an explicit user reference, `HEAD`, or the merge base. If the baseline must be executed, use a temporary worktree or isolated checkout; never replace files in the user’s current worktree merely to run the baseline.

If there is no executable candidate or baseline, report the limitation and ask for an execution command or task adapter. Do not claim that an experiment is comparable when only one side ran.

### Phase 1: Build the experiment plan

Create an `ExperimentPlan` before generating the suite:

```yaml
experiment_type: exploration | paired_comparison | regression_gate | failure_reproduction | optimization | cost_performance
goal: <decision this run supports>
hypothesis: <testable expected change>

candidate:
  source: working_tree | git_ref | command | module
  revision: <commit or working-tree digest>

baseline:
  source: none | git_ref | prior_run | command
  revision: <commit or run ID>

case_source:
  kind: dataset | local_fixture | trace_sample | annotations | prior_run | synthetic
  identity: <path, name/id, or trace selection>
  version: <pinned version when applicable>

task:
  adapter: python | node | http
  source: <module:function, command, endpoint, or explicit placeholder>

metrics:
  primary: [<metrics>]
  secondary: [<metrics>]
  operational: [error_rate, duration, token_usage, estimated_cost]

acceptance_policy:
  enabled: true | false
  min_cases: <number>
  thresholds: <explicit policy>

permissions:
  local_write: approved | ask
  datadog_publish: approved | ask
  production_data: approved | ask
```

The plan must answer:

- What changed?
- What decision should the result support?
- What is the comparison, if any?
- Which cases represent the decision?
- What constitutes improvement, regression, or useful discovery?
- What is allowed to be written or published?

If the user has not supplied a meaningful goal, ask them to choose a profile and provide a short goal. Do not invent a pass/fail criterion for exploration.

### Phase 2: Select the experiment profile

Choose exactly one primary profile. A profile may have secondary metrics or follow-up actions, but the primary decision must remain clear.

#### Exploration / discovery

Use when the goal is to understand behavior or find opportunities rather than prove a predetermined hypothesis.

Defaults:

- use a diverse, trace-derived, annotation-derived, or representative sample;
- keep the CaseSet small enough to inspect;
- allow missing expected outputs;
- use operational metrics, semantic evaluators, error categories, and novelty/diversity signals;
- use summary evaluators to cluster or prioritize findings; and
- return representative examples, failure modes, hypotheses, and proposed next experiments.

Do not report a single score as the outcome unless the user explicitly requests one.

#### Offline paired comparison

Use when comparing a baseline and candidate on the same cases. Call this an offline paired comparison rather than an online A/B test.

Defaults:

- same CaseSet version for both variants;
- same evaluator names, versions, and inputs;
- explicit baseline and candidate provenance;
- per-case paired deltas where available;
- quality, error, duration, token, and cost comparisons; and
- a stated rule for acceptable tradeoffs.

Reject or pause if the two variants cannot run on comparable cases.

#### Regression gate

Use when the result should protect a release or CI pipeline.

Defaults:

- pinned canonical CaseSet version;
- deterministic checks for critical contracts;
- explicit minimum case count;
- thresholds and required metrics;
- failure on task errors, missing required metrics, or incomplete runs; and
- machine-readable result and exit status.

A regression evaluator measures behavior. The acceptance policy decides whether CI passes.

#### Failure reproduction

Use to reproduce a known production failure, validate a fix, or promote an incident into a regression case.

Defaults:

- preserve source trace, span, annotation, and incident provenance;
- tag failure mode, severity, source, and reproduction status;
- treat expected behavior as a contract rather than copying the observed output as truth;
- compare before and after behavior when validating a fix; and
- include the original trace IDs in the report without putting them into secrets or arbitrary tags.

#### Optimization

Use to prepare a fixed benchmark for iterative prompt, model, retrieval, or agent improvement.

Defaults:

- fixed CaseSet and scoring contract;
- baseline run before the first change;
- one focused change per iteration;
- Git revision or working-tree digest per candidate;
- explicit keep/revert decisions; and
- handoff to `agent-observability-auto-experiment` for the loop.

Do not start an autonomous hill-climb without a validated baseline and user approval.

#### Cost/performance benchmark

Use when operational behavior is the primary decision.

Defaults:

- representative CaseSet;
- repeated runs if variance matters;
- duration, tokens, estimated cost, throughput, and error rate as primary metrics;
- quality evaluators as guardrails; and
- no required expected output unless quality is also being evaluated.

### Phase 3: Select the CaseSet

Map the profile to an appropriate source:

| Profile | Preferred source |
| --- | --- |
| Exploration | diverse traces, annotations, or sampled production cases |
| Paired comparison | immutable dataset snapshot or local fixture set |
| Regression gate | pinned canonical dataset version |
| Failure reproduction | failing traces, annotations, or incident fixtures |
| Optimization | stable benchmark dataset |
| Cost/performance | representative dataset with enough volume for variance |

Supported sources:

- existing Datadog Dataset;
- local JSON/CSV fixtures;
- trace IDs or trace samples;
- annotation exports;
- prior experiment runs; and
- deliberately generated synthetic cases.

For each source, record:

- source kind and identity;
- dataset or CaseSet version;
- sampling query or selection rule;
- stable case IDs;
- expected-output status; and
- provenance and privacy constraints.

Do not use observed production outputs as ground truth without an explicit validation or labeling step. If expected behavior is unknown, leave `expected_output` absent and choose evaluators that do not require it.

### Phase 4: Define primitives and evaluation

#### Case records

A record should contain:

```text
stable ID
input
optional expected_output
metadata
low-cardinality tags
provenance
```

Use tags for queryable slices such as `source:production`, `risk:high`, `failure_mode:tool_retry`, `variant:baseline`, or `domain:billing`. Use metadata for trace IDs, annotation IDs, sampling context, and case-specific configuration. Never put secrets or large prompts in tags.

#### Task adapter

The task is a thin adapter around the application under test. It should:

- accept the record input, configuration, and metadata;
- invoke the candidate implementation;
- return structured output and useful diagnostic metadata;
- avoid evaluator logic;
- use bounded timeouts;
- make external side effects explicit; and
- preserve candidate and trace provenance.

The task may use an importable function, shell/JSONL command, local HTTP endpoint, existing CLI, or approved trace replay adapter. If the source cannot be found, emit a clearly marked placeholder and stop before claiming a valid run.

#### Evaluators

Use deterministic evaluators for exact contracts and semantic judges only where deterministic checks are insufficient. Choose evaluators based on the profile and available signals:

- output/schema correctness;
- tool-call correctness;
- retrieval/groundedness;
- safety or policy compliance;
- semantic quality;
- error taxonomy;
- latency, token, and cost metrics; and
- aggregate deltas or distributions.

Every evaluator must have a stable name, clear input contract, and explicit missing-data/error behavior. Evaluator failures must not become passing values.

#### Summary evaluators and gates

Use summary evaluators for aggregate quality, paired deltas, distributions, and cost/quality tradeoffs. Keep row-level failures visible.

Represent acceptance separately:

```yaml
acceptance_policy:
  primary_metric: semantic_recall
  minimum: 0.80
  max_regression:
    estimated_cost: 0.10
  required:
    - no_critical_case_failure
    - task_error_rate < 0.02
```

Exploration profiles may have no acceptance gate. Regression profiles must have one.

### Phase 5: Select the construction adapter

Use the generalized `agent-observability-experiment-bootstrap` skill for Python and Node artifact generation.

| Situation | First-iteration path |
| --- | --- |
| Python application or existing Python workflow | `python` bootstrap adapter |
| JavaScript/TypeScript application with local `dd-trace` support | `node` bootstrap adapter |
| Existing local HTTP endpoint or externally driven workflow | use an explicitly supplied `http` task runner; do not generate an HTTP client |
| User explicitly requests an unsupported construction path | explain the boundary and ask for an existing runner or a supported adapter |

The adapter must not change the experiment semantics. It changes only how the task, dataset, experiment, events, and metrics are constructed. HTTP artifact construction is intentionally deferred until a verified public contract exists.

Use the bootstrap skill with the selected adapter and pass through:

- purpose;
- project name;
- dataset source and version;
- task source;
- evaluator style or labels;
- output path; and
- provenance metadata.

Do not mix APIs in one generated artifact. Do not manually call HTTP endpoints from a Python or Node SDK artifact to work around an unverified SDK feature.

### Phase 6: Generate the suite

Default ephemeral output:

```text
.ddog/experiments/<slug>/
  experiment.yaml
  plan.md
  generated/
  artifacts/
  README.md
```

A durable, tracked regression suite may use:

```text
experiments/<slug>/
  experiment.yaml
  plan.md
  generated/
  evaluators/
  fixtures/
  README.md
```

Only create a tracked suite after the user approves repository changes. Do not overwrite an existing experiment directory; choose a new slug or ask before replacing it.

The generated manifest must include:

- profile, goal, and hypothesis;
- candidate and baseline identity;
- CaseSet identity/version;
- adapter and task source;
- evaluator labels and versions;
- primary, secondary, and operational metrics;
- acceptance policy;
- permission decisions;
- Git and environment provenance; and
- local artifact paths and eventual Datadog IDs.

### Phase 7: Validate before running

Run local validation before network publication or a full execution.

Checklist:

- [ ] profile and decision are explicit;
- [ ] candidate task source is real or clearly marked placeholder;
- [ ] baseline is identified or intentionally absent;
- [ ] case IDs are stable and unique;
- [ ] inputs are non-empty and schema-valid;
- [ ] expected-output semantics are documented;
- [ ] comparable variants use the same CaseSet and evaluators;
- [ ] evaluator labels are unique and versioned;
- [ ] missing values and evaluator errors are distinct from false/pass;
- [ ] required metrics and acceptance thresholds are explicit;
- [ ] timeouts, retries, and concurrency are bounded;
- [ ] secrets are not present in generated files or artifacts;
- [ ] Git, dataset, model, and configuration provenance is recorded;
- [ ] local output paths are safe and non-destructive; and
- [ ] the adapter artifact passes its syntax/dry-run checks.

Use the bootstrap skill’s adapter-specific validation commands. For local fixtures, scrub obvious PII and credential-like values before embedding or publishing.

If validation fails, fix the plan or ask the user for a decision. Do not continue to a full run with known comparability or provenance gaps.

### Phase 8: Smoke test

Run one representative case first. Report:

- task output shape;
- evaluator results and reasoning;
- operational metrics;
- trace/span IDs when available;
- artifact paths; and
- errors, missing data, or side-effect warnings.

Ask for confirmation before a costly or production-connected full run if the smoke test reveals unexpected behavior.

### Phase 9: Full local run

Run the full suite with:

- bounded concurrency;
- per-case timeout;
- cancellation handling;
- per-case task and evaluator error capture;
- deterministic case ordering or a recorded sampling seed; and
- atomic artifact writes.

The runner should support these conceptual operations:

```text
list       show cases, evaluator labels, and configuration
check      validate without executing tasks
run-case   execute one case
run        execute the selected suite
compare    compare two local or published results
```

Do not mark a run successful when some rows failed unless the report explicitly distinguishes partial completion and the profile permits it.

### Phase 10: Optional Datadog publication

Publication requires an explicit user decision. Before publishing, state:

- project and dataset to be created or updated;
- dataset version and record count;
- whether prompts, outputs, traces, or annotations leave the local environment;
- expected evaluator/API cost;
- credentials and site configuration required; and
- whether a baseline or comparison run will also be published.

Use the selected bootstrap adapter. Never write API keys into generated files. If publication partially fails, preserve the project/dataset/experiment IDs, failed row IDs, and a reconciliation command; do not claim completion.

### Phase 11: Handoff and next actions

Always return a concise handoff containing:

```text
Profile:
Goal:
Hypothesis:
Candidate:
Baseline:
CaseSet:
Adapter:
Evaluators:
Primary metrics:
Gate decision:
Local artifacts:
Datadog IDs/URL:

Next actions:
1. ...
2. ...
3. ...
```

Recommend actions based on the profile:

- Exploration: inspect clusters, label cases, and select a follow-up comparison.
- Paired comparison: inspect biggest wins/regressions and decide whether to promote the candidate.
- Regression gate: add the pinned suite to CI or update the gate policy.
- Failure reproduction: preserve resolved cases as regression fixtures.
- Optimization: invoke `agent-observability-auto-experiment` with the validated harness.
- Cost/performance: review quality guardrails before adopting the cheaper/faster candidate.

## Permissions and safety

Use this permission model:

### Inspect

Read repository files and Git state. No network or mutations.

### Local execution

Write only approved generated files and artifacts. Execute local commands. No Datadog writes.

### Datadog publication

Create or update Datadog projects, datasets, experiments, spans, or metrics. Requires explicit confirmation.

### Production or external data

Query production traces, annotation queues, external services, or sandboxes. Requires explicit confirmation and secret-safe handling.

The skill must not:

- checkout over the current worktree;
- modify application source code without explicit approval;
- publish production prompts or outputs silently;
- invent dataset IDs, trace IDs, evaluator names, or API endpoints;
- treat observed outputs as verified ground truth;
- put credentials in manifests or reports; or
- start autonomous optimization without a user-approved scoring contract.

## Composition with other skills

Use the smallest existing skill that supplies missing expertise:

- `agent-observability-experiment-bootstrap`: generate Python or Node artifacts;
- `agent-observability-eval-bootstrap`: generate evaluator candidates from traces or failure hypotheses;
- `agent-observability-trace-rca`: diagnose failures before creating a reproduction profile;
- `agent-observability-session-classify`: classify sampled production sessions;
- `agent-observability-experiment-analyzer`: analyze one or more published results;
- `agent-observability-auto-experiment`: iterate after a validated baseline; and
- `agent-observability-eval-pipeline`: orchestrate broad production-trace-to-experiment flows.

Do not regenerate these skills’ outputs unless the user explicitly requests a standalone alternative.

## Completion criteria

A local experiment is ready to hand off when:

1. Its profile and decision are explicit.
2. Its CaseSet and expected-output policy are documented.
3. Its task adapter is real, reproducible, or clearly blocked.
4. Its evaluators and operational metrics are defined.
5. Its acceptance policy is explicit when gating is requested.
6. Its adapter artifact passes validation.
7. A smoke test has run successfully or the blocker is reported.
8. Full-run artifacts preserve per-case results and provenance.
9. Publication, if any, was explicitly approved.
10. The user receives a comparison link or a clear local-only handoff with next actions.

## Initial implementation boundary

The first version implements planning, all six RFC profile policies, manifest requirements, validation, local smoke/full-run orchestration, and delegation to supported adapters. It does not introduce a second experiment SDK or a new HTTP client. Adapter-specific Python and Node construction belongs to `agent-observability-experiment-bootstrap`; this skill supplies the goal-oriented workflow and safety policy.
