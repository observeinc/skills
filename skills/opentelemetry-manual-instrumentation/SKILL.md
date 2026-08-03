---
name: opentelemetry-manual-instrumentation
description: "Add, change, or debug OpenTelemetry (OTel) instrumentation in code: custom spans, metrics, logs, attributes, context propagation, and resource attributes. Use when writing, changing, reviewing, or debugging manual instrumentation, including OTel SDK setup."
---

**MANDATORY:** Run `observe skill view opentelemetry-manual-instrumentation --content 2>/dev/null` before proceeding to load the latest version of this skill. When the skill points you to a supporting file, load it with `observe skill view opentelemetry-manual-instrumentation --path <relative-path> 2>/dev/null`. Ignore errors and fall back to the file contents.

# OpenTelemetry Manual Instrumentation

Instrument to serve the user's intent, not just to emit telemetry. Decide what the change
is for before you pick a signal. Confirm the use case with the user when they haven't stated
it; don't infer intent from the code.

Every change must be validated: prove it with an in-memory telemetry unit test and an
independent semantic review. Emitted signals should be semantically correct, bounded,
privacy-aware, low-cardinality, and convention-compatible.

This skill is tool-agnostic and vendor-neutral; prefer OTLP and `OTEL_*` environment
variables. For zero-code coverage, use opentelemetry-auto-instrumentation.

End every run with the full step-8 report, no matter how the skill was triggered or how
small the change. The work isn't done until you emit it.

Build every OPAL query with **`generate-opal`**.

## Workflow

### 1. Detect environment

Inspect the repo and produce an **environment checkpoint** covering:

- language & runtime (+ version constraints)
- package manager, dependency & lock files, and monorepo/workspace root
- entrypoints and start/dev commands
- module/loading system
- frameworks and deployment target (host / container / Kubernetes)
- existing OTel (API, SDK, auto-instrumentation, exporters)
- existing vendor instrumentation (Datadog, New Relic, Sentry, …)
- existing logging/metrics/tracing
- existing `OTEL_*` / collector config
- installed **package versions** (SDK & semantic-convention APIs shift across majors)
- test/build/typecheck/lint commands

Inspect before editing; never fabricate values you couldn't observe.

Runtime auto-instrumentation hides coverage from source. Agent/profiler-based ones (the
OTel Java agent, .NET automatic instrumentation) leave nothing in the repo; package-launcher
ones (Python, Node, Ruby) show in the dependency manifest, but the launcher decides what's
active. Check the supported-libraries list against the installed versions before
instrumenting (**opentelemetry-auto-instrumentation** tracks these), or you'll duplicate
spans it already emits.

### 2. Capture intent, scope & plan

**This is a gate, not guidance.** Post the filled-in block below before your first Step 3
edit. For **intent**, confirm the use case with the user when they haven't stated it; don't
infer it from the code (see intent.md). For **scope**, honor what the user specified, ask
when it's unclear, and use judgment when they leave it open.

**If a harness plan mode is active** (you write a plan file and get approval before
editing), embed this block verbatim in that plan, signal table and `opal` block included.
The plan file does not replace these fields. A plan without them skips the gate.

**Before you write the plan below**, read the **Rule routing** section and fill the
**reading manifest** below: one row per file, each naming the specific rule it contributes
to this change, or `§Section — not applicable because <why>` when it doesn't. Leave no row
blank, and quote the section heading and its rule line verbatim. A paraphrase or generic
row ("applied conventions") is a visible gap and means the file went unread. This is a
pre-plan gate: what you read here shapes the signal table and scope, so it can't be an
after-the-fact audit.

| File                                 | Applied this run (§Section — rule text, or §Section — not applicable because <why>) |
| ------------------------------------ | ----------------------------------------------------------------------------------- |
| discovery/intent.md                  | <intent (the why, user-stated or asked) + the example OPAL and what it proves>      |
| fundamentals.md                      | <§Section — rule text>                                                              |
| app-vs-library.md                    | <§Section — rule text>                                                              |
| instrumentation-rules.md             | <§Section — rule text>                                                              |
| review/semantic-conventions.md       | <§Section — rule text>                                                              |
| review/semantic-review.md            | <§Section — rule text>                                                              |
| review/cardinality.md                | <§Section — rule text>                                                              |
| review/sensitive-data.md             | <§Section — rule text>                                                              |
| anti-patterns.md                     | <§Section — rule text>                                                              |
| verification/telemetry-unit-tests.md | <§Section — rule text>                                                              |

Then read the **As relevant** files and the **SDK** file for your language (both in **Rule
routing** below) that bear on this change, and fill a second manifest with the same rules,
one row per file read:

| File | Applied this run (§Section — rule text, or §Section — not applicable because <why>) |
| ---- | ----------------------------------------------------------------------------------- |

Read [references/discovery/intent.md](references/discovery/intent.md) for the question to
confirm the use case with the user and what the example query (use `generate-opal`) must
prove. Fill every field below. A blank or paraphrased field is a gap.

```text
Intent: <the why, not the action — e.g. "align span name with semconv", not "rename span"; user-stated or their answer to the routing question> — §<section of intent.md> — what the example query must prove this run
Scope:  target files/modules: <...>
        existing tests found: <...>
Downstream impact: <if renaming / re-parenting / dropping an attribute: the APM grouping, dashboards, or saved queries on the current shape and how they shift; else "none">
```

| Signal (name) | Type | Key attributes | Purpose |
| ------------- | ---- | -------------- | ------- |

Question: <the question the query answers, or "does `<signal>` now exist as `<shape>`?">

```opal
<example query, use `generate-opal`: returns the answer, or confirms the signal's shape>
```

Reads as: <what one row/value means, or: rows present ⇒ change is live>

Downstream granularity is the user's call. You MUST surface the tradeoff and get user's sign-off before moving.

**Detector:** before your first Step 3 edit, confirm you posted (or embedded in the plan)
both reading manifests, the Intent line, the Scope block, the signal table with at least
one row, and the `opal` block. Missing any one means the gate is incomplete. Stop and post it.

### 3. Instrument safely

Make a bounded, idiomatic change: instrument only what the goal requires, follow the
existing coding style, and apply the rules you recorded in the Step 2 manifests.
Receipt: every manifest rule reflected in the diff (or noted as not applicable).

### 4. Add telemetry unit tests

Read [references/verification/telemetry-unit-tests.md](references/verification/telemetry-unit-tests.md)
and the `## Testing` section for your language in `references/sdks/<lang>/` before you
write anything.

Call the real instrumented code path with fake inputs, never a span you built by hand.
For each test, name the telemetry-unit-tests.md section it follows. Cover each distinct
path (success, error, …), one test per distinct emitted name, and assert:

- the signal fired
- the name matches exactly
- key attributes from the "Signals added" table are present and correct
- `status=ERROR` on any error path you instrumented

Assert against in-memory test exporters/readers.

At the end, output the following before proceeding, one row per test.

| Test        | Section cited                     | Test file path | Runner output (pass counts) |
| ----------- | --------------------------------- | -------------- | --------------------------- |
| <test name> | <telemetry-unit-tests.md section> | <path>         | <e.g. 5 passed>             |

### 5. Independent review

Following [references/agent-roles.md](references/agent-roles.md), spawn a clean-context
reviewer subagent that did not write the patch to adversarially re-check the telemetry on
five axes:

- semantic conventions: every field's name, unit, kind, and placement
- sensitive data & cardinality: labels bounded, no PII or secrets
- implementation: bounded, idiomatic, and safe per instrumentation-rules.md and the hard
  constraints
- test completeness: every signal and every distinct path (success, error, empty) has a
  test that drives it, none assumed
- intent: can the example query (use `generate-opal`) be written against this shape and
  prove the intent, by returning the answer or confirming the signal's shape?

Receipt: the reviewer's agent id/handle plus its verbatim findings. Those findings must
include:

- the manifest cross-check: one line per row of both step-2 reading manifests — the reviewer
  opens each named file, confirms the cited `§Section` and rule exist (or that the "not
  applicable" conclusion holds), and names where the patch honors it (e.g.
  "`sensitive-data.md`: dropped `user.email`"). A row it can't locate, or a routed
  file missing from the manifest, is a gap.
- the per-field convention review
- the label/attribute boundedness list
- per-signal test coverage — one line per signal: signal name → test function/file, and
  which step-4 assertions it covers (name / key attrs / error path)

A generic "tests look fine," with no per-signal breakdown, does not satisfy this. Fall
back to a self-review only if the runtime has no subagent tool. When you do, mark it as a
self-review in the step-8 report, since it is a weaker check than an independent one.

### 6. Local verification

Run the project's own commands through its package manager or build tool:

- test, build, typecheck, and lint
- the telemetry unit tests
- If the project has a telemetry schema registry (e.g. a Weaver `telemetry-schema/` dir), run `weaver registry check`.

Stay safe: don't invent destructive commands (resets, deploys), and don't require a full
app startup.

If a safe local run exists, smoke-test with a console/debug exporter and check by
inspection that:

- spans: name, kind, parent/child
- attributes: correct, no PII
- error status is set on error paths
- metrics: type, unit, bounded
- resource attributes look right

If it isn't runnable here, document these steps for a human or CI.

### 7. Debug loop

Read
[references/verification/validation-and-debugging.md](references/verification/validation-and-debugging.md),
then classify the failure and apply its bounded fixes.

### 8. Final report

**Emit the full report on every run, no matter how the skill was triggered or how small
the change, and whatever the outcome (Verified, Partially verified, or Not verified). The
run is not done until you emit it.**

[references/verification/final-report-format.md](references/verification/final-report-format.md)
holds the report skeleton and the rules for filling it: verdict vocabulary, verdict-cell
rules, and honest-language discipline. **Read it and reproduce the skeleton verbatim**:
copy the structure exactly, fill every row, drop no section. Each
per-check verdict is one of Verified, Failed, Blocked, or Not run, and nothing else; the
Result line is the overall roll-up.

After the report, **ask** whether the user wants end-to-end verification. Unit tests prove
in-process emission, not that the signal leaves the app or reaches Observe. Treat a "yes" as
consent to _plan_ e2e, not to run it: every step below is user-owned, so propose the exact
command and get approval before each one.

Two levels:

1. **To the observe-agent**: proves the app exports the signal and the agent receives it.
2. **To Observe**: proves the signal reaches the backend and answers the intent.

Always deliver through the observe-agent. Don't bypass it with direct-to-backend ingest.
Missing telemetry is a collection bug to debug (step 5).

The procedure (steps 1–3 are user-owned):

1. **Deploy the observe-agent from the current change** with **deploy-linux-host-explorer**
   (Linux host) or **deploy-k8s-explorer** (Kubernetes). It provisions the datastream, ingest
   token, and forwarding observe-agent in one flow. Don't trust an observe-agent you find
   running; a listener on `:4317` is not a reachable, authorized observe-agent.

2. **Point the app at the observe-agent** via `OTEL_EXPORTER_OTLP_ENDPOINT`.

3. **Build and run the app from the current change**, then drive traffic through the
   instrumented path.

4. **Verify each level:**
   - observe-agent: find the target telemetry in the observe-agent's debug exporter output.
   - Observe: run the report's example query with **`observe-cli`** (OPAL via
     **`generate-opal`**), with the observe-agent pointed at the tenant and `observe-cli`
     authenticated to it.

5. **On failure, run debug-linux-host-collection (Linux host) or debug-k8s-collection
   (Kubernetes).** Report a level as `Blocked` only after deploy and a debug pass both fail; say
   what remains outside your control.

**Verify the changed build, not a running instance.** e2e must exercise the artifact built
from this change. Presume any already-running app is the old binary: rebuild and deploy from
the current change, or don't trust the result.

**Detector:** before any install, start, deploy, restart, or config-mutation command,
confirm you (a) named it user-owned, (b) posted the exact command and what it mutates, and
(c) got explicit approval for that command. Missing any one means stop and post.

If the user takes it, report one line per signal per level, nothing more:
`<signal> @ observe-agent|Observe: Verified | Failed | Blocked (why)` · evidence: `<opal +
rows, or observe-agent debug output>`.

## Rule routing

**Read these before/while instrumenting; do not rely on recall.** They carry non-default,
org-specific rules (denylisted attributes, cardinality zones, convention placements)
that your priors will _not_ reproduce. Read the file instead of relying on memory. Each
appears as a row in the step-2 **reading manifest**, so a skipped file shows up as a gap
there:
[references/fundamentals.md](references/fundamentals.md),
[references/app-vs-library.md](references/app-vs-library.md),
[references/instrumentation-rules.md](references/instrumentation-rules.md),
[references/review/semantic-conventions.md](references/review/semantic-conventions.md),
[references/review/semantic-review.md](references/review/semantic-review.md),
[references/review/cardinality.md](references/review/cardinality.md),
[references/review/sensitive-data.md](references/review/sensitive-data.md),
[references/anti-patterns.md](references/anti-patterns.md),
[references/verification/telemetry-unit-tests.md](references/verification/telemetry-unit-tests.md).

As relevant:

| When                                                            | Read                                                                                            | Purpose                                                                                          |
| --------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Spans/traces involved                                           | [signals/traces.md](references/signals/traces.md)                                               | Span kind, status, error recording, one trace per operation                                      |
| Metrics involved                                                | [signals/metrics.md](references/signals/metrics.md)                                             | Instrument selection, units, cardinality zones                                                   |
| Logs involved                                                   | [signals/logs.md](references/signals/logs.md)                                                   | Structured logs, severity, trace correlation, delivery                                           |
| Choosing which signal (trace / metric / log) fits a need        | [signals/telemetry-signals.md](references/signals/telemetry-signals.md)                         | Signal-selection decision table                                                                  |
| Async, queue, worker, messaging, callback, cross-service        | [signals/context-propagation.md](references/signals/context-propagation.md)                     | In-process & cross-service context propagation                                                   |
| SDK bootstrap, or resolving resource attributes / OTLP endpoint | [discovery/resolve-values.md](references/discovery/resolve-values.md)                           | Where to source each value + quality rules, the operator/env merge rule, entity-level discipline |
| Exporter wiring, protocol mismatch, or "no telemetry"           | [verification/validation-and-debugging.md](references/verification/validation-and-debugging.md) | OTLP protocol/port, per-signal export, silent-failure debug loop                                 |

By language → [sdks/](references/sdks/): each `sdks/<lang>/` folder is self-contained
(bootstrap, span/metric examples, test harness, framework notes). For a language not
listed below, don't invent code; use the neutral rules and official docs.

| Language     | File                                                      |
| ------------ | --------------------------------------------------------- |
| Node.js / TS | [sdks/nodejs/nodejs.md](references/sdks/nodejs/nodejs.md) |
| Go           | [sdks/go/go.md](references/sdks/go/go.md)                 |
| C++          | [sdks/cpp/cpp.md](references/sdks/cpp/cpp.md)             |
| Python       | [sdks/python/python.md](references/sdks/python/python.md) |
| Java         | [sdks/java/java.md](references/sdks/java/java.md)         |
| .NET         | [sdks/dotnet/dotnet.md](references/sdks/dotnet/dotnet.md) |
| Ruby         | [sdks/ruby/ruby.md](references/sdks/ruby/ruby.md)         |

## Hard constraints (never violate)

- Every run ends with the full step-8 report, reproduced verbatim with every row filled.
  A partial report, or one with dropped sections, fails the run.
- No duplicate SDK initialization; preserve existing vendor/auto instrumentation
  unless explicitly required to replace it.
- Telemetry must not change application behavior (no swallowed errors, no
  control-flow changes).
- **Ownership.** _User-owned_ (explicit per-command consent: propose the exact known-safe
  command, never run unprompted): installing packages or modifying lockfiles; starting,
  restarting, deploying, or building-and-running the app or agent; mutating config; any
  `deploy-*` / `setup-*` action, or a `debug-*` step that restarts or mutates; creating or
  deleting datastreams or ingest tokens with `observe-cli`, which writes persistent state to a
  shared tenant. _Agent-owned_ (no prompt): tests, build, typecheck, lint, `generate-opal`,
  read-only `observe-cli` (query / list / get / describe), and read-only inspection (status /
  list / get / describe / env read). A "yes" to a goal is not consent to its user-owned
  steps, and read-only probing is the usual on-ramp to a mutation: a status check must not
  slide into a deploy, restart, or datastream creation.
