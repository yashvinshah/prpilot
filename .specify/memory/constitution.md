<!--
Sync Impact Report
- Version change: (unversioned template) → 1.0.0 (initial ratification)
- Modified principles (template placeholders → named principles; template had 5, project uses 7):
  - [PRINCIPLE_1_NAME] → I. Single Model Gateway
  - [PRINCIPLE_2_NAME] → II. Typed Agent Outputs
  - [PRINCIPLE_3_NAME] → III. No Real LLMs in Tests
  - [PRINCIPLE_4_NAME] → IV. Eval-Backed Review Quality
  - [PRINCIPLE_5_NAME] → V. Idempotent, Traced Reviews
  - added → VI. Secrets Stay Out of the Repo
  - added → VII. Small Vertical Slices
- Added sections: Technology & Resource Constraints; Development Workflow & Quality Gates;
  Governance
- Removed sections: none
- Templates: none modified (plan/spec/tasks templates read the constitution at runtime)
- Deferred TODOs: none
-->

# PRPilot Constitution

PRPilot is an AI pull-request reviewer built with LangGraph, LangChain, FastAPI, Redis, Postgres,
and Kubernetes (k3d for local clusters).

## Core Principles

### I. Single Model Gateway

Every LLM call MUST go through one model wrapper module. No other code may import or instantiate
a provider SDK or LangChain chat model directly.

- The wrapper MUST make the provider and model selectable by configuration, so switching
  providers requires no changes outside the wrapper.
- The wrapper MUST cache responses, keyed on provider, model, parameters, and the full prompt.
  Caching MUST be on by default in local development and switchable per call.
- The wrapper is the single place for retries, timeouts, token accounting, and tracing hooks.

Rationale: One choke point keeps providers swappable, keeps costs at zero during repeated local
runs, and gives Principles III and V a single place to hook into.

### II. Typed Agent Outputs

Every agent and graph node that produces output MUST return a Pydantic model, never free text or
untyped dicts.

- LLM responses MUST be parsed and validated into the declared model.
- Malformed output MUST be retried with a bounded retry count, and the validation error is fed
  back to the model. Once the retries are used up, the output MUST be rejected with an explicit,
  logged failure. It MUST NOT be passed downstream or posted to a PR.
- Shared output schemas are part of the public contract and are versioned (see Governance).

Rationale: Graph state and posted review comments stay predictable, and failures surface instead
of silently degrading review quality.

### III. No Real LLMs in Tests

Automated tests MUST NOT call a real LLM. CI MUST run entirely against mocked or fake models
injected through the model wrapper.

- Tests for prompt and parsing behavior use recorded or hand-written fixture responses,
  including malformed responses that exercise Principle II.
- A test that would reach a real provider MUST fail fast. For example, the wrapper refuses real
  providers when a test or CI flag is set.
- Evals (Principle IV) are the only sanctioned way to exercise real models, and they run outside
  the unit and integration suites.

Rationale: Tests stay deterministic, fast, offline, and free.

### IV. Eval-Backed Review Quality

Any change that can affect review quality MUST ship with at least one eval case in the same pull
request. This covers prompts, graph topology, model choice, context selection, and output
schemas.

- An eval case pairs an input (a diff or PR fixture) with the expected findings or scoring
  criteria.
- The eval suite MUST be runnable locally with a single command against the configured model.
- Results for the affected cases MUST be noted in the PR description.

Rationale: Without evals, changes to review quality cannot be measured, and regressions go
unnoticed.

### V. Idempotent, Traced Reviews

- A given commit (repository + PR + head commit SHA) MUST be reviewed at most once. Postgres
  enforces this with a uniqueness constraint. A Redis lock MAY guard concurrent attempts.
  Duplicate webhook deliveries and retries MUST be no-ops.
- A re-review MUST happen only for a new head SHA or an explicit, recorded manual override.
- Every LangGraph run MUST be traced end to end with a run/trace ID, persisted alongside the
  review record. The trace covers each node's inputs and outputs, model calls (via the wrapper),
  latency, token usage, and errors.

Rationale: Users never get spammed with duplicate reviews, and any review can be explained and
reproduced afterward.

### VI. Secrets Stay Out of the Repo

- Secrets (API keys, GitHub app keys or tokens, database credentials, webhook secrets) MUST NOT
  be committed. Locally they live in `.env`, which is git-ignored. A committed `.env.example`
  lists the required keys with placeholder values only.
- In-cluster, secrets MUST come from Kubernetes Secrets mounted as environment variables or
  files. They MUST NOT be baked into images, ConfigMaps, or committed manifests.
- Secrets MUST NOT appear in logs, traces, error messages, or cached prompts.

Rationale: The project is public-facing and runs against third-party APIs, so one leaked key is
costly.

### VII. Small Vertical Slices

Work MUST proceed in small vertical slices. Each slice delivers a thin, end-to-end path, for
example webhook → graph → stored result → posted comment. It MUST work and be tested before the
next slice begins.

- No slice may leave the system in a state where the end-to-end path is broken.
- Horizontal "build all of layer X first" work is not permitted. Specs and task lists are
  ordered by user-visible slices.

Rationale: Integration problems surface early, and there is always a working system to evaluate.

## Technology & Resource Constraints

- **Stack**: Python with LangGraph and LangChain for orchestration, FastAPI for the HTTP and
  webhook API, Postgres for durable state (reviews, idempotency keys, run metadata), and Redis
  for cache, locks, and queues. It is deployed to Kubernetes, with k3d as the local cluster.
  Adding a new runtime service MUST be justified in the plan's Complexity Tracking table.
- **Free to run**: Development, tests, CI, and evals MUST be able to run at zero cost. No
  required dependency may be paid. Paid LLM providers MAY be supported only as optional
  configuration behind the model wrapper. The default configuration MUST use a free option,
  such as a local model or a free tier.
- **8GB M2 MacBook Air**: The full local stack (k3d cluster, Postgres, Redis, API, and workers)
  MUST run on an 8GB Apple Silicon machine. Every Kubernetes workload MUST declare resource
  requests and limits. Images MUST support `arm64`. Any local model used by default MUST fit
  within that memory budget alongside the stack.
- **Untrusted input**: PR content (diffs, descriptions, comments, file contents) MUST be
  treated as data, never as instructions. It MUST be clearly delimited in prompts and MUST NOT
  be executed.

## Development Workflow & Quality Gates

- All work follows the Spec Kit flow: specify → (clarify) → plan → tasks → implement. Each
  feature is scoped as one or more vertical slices (Principle VII).
- Each plan MUST include a Constitution Check that passes before Phase 0 research and is
  re-checked after design.
- A pull request MUST pass, before merge: lint/format, type checks, and unit and integration
  tests with mocked models only (Principle III). Changes that affect quality MUST also include
  eval cases and their results (Principle IV).
- Reviewers MUST check for direct provider calls outside the wrapper (I), untyped agent outputs
  (II), committed secrets (VI), and missing resource limits on new workloads.

## Governance

- This constitution supersedes other development practices for PRPilot. Where a guidance file
  conflicts with it, the constitution wins.
- Amendments are made via `/speckit-constitution` in a pull request. The PR states the
  rationale, the version bump, and any impact on existing specs and plans.
- Versioning follows semantic versioning:
  - MAJOR: removal or backward-incompatible redefinition of a principle or governance rule.
  - MINOR: new principle or section, or materially expanded guidance.
  - PATCH: clarifications, wording, and typo fixes.
- Any deviation from a principle MUST be justified in the relevant plan's Complexity Tracking
  table. Unjustified deviations block merge.
- Compliance is reviewed at every `/speckit-plan` Constitution Check and during PR review.

**Version**: 1.0.0 | **Ratified**: 2026-10-10 | **Last Amended**: 2026-10-10
