# Final Report Format

The report skeleton, the verdict vocabulary, how to fill the verdict cells, and the
honest-language discipline. The reading manifest lives at the step-3 pre-edit gate; the
step-5 reviewer records its evidence here through the Rule-file findings.

Always produce a final report, even on partial success or when blocked. The report is
where the checks are enforced: a check counts as met only when its receipt (a pasted
command output, tool verdict, or query result) appears in the report. "Reviewed" or
"done" without a real artifact counts as not met. Fill every row of the skeleton, and
don't overclaim.

## Skeleton

Reproduce this verbatim and fill every row; copy the structure exactly rather than
rewriting it.

````md
## Manual OpenTelemetry Instrumentation — Report

**Result:** Verified | Partially verified | Not verified — <one line: what was instrumented + which receipts passed>
**Intent:** <the why the change serves + what the example query proves>

### Signals added

| Signal (name)           | Type          | Key attributes      | Purpose               |
| ----------------------- | ------------- | ------------------- | --------------------- |
| e.g. `ai.chat.sessions` | UpDownCounter | `app.session.state` | count active sessions |

### Receipts

| Step                         | Receipt (pasted evidence)                                                                                                                                                                                       | Verdict                                    |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Telemetry unit tests         | <test file> + runner output ("N passed"); per signal: `<name>` → `<test fn>` (§section / name / key attrs / error)                                                                                              | Verified / Failed / Blocked                |
| Semantic review              | per-field convention review (matched convention / custom+justification)                                                                                                                                         | Verified / Failed                          |
| Sensitive data & cardinality | each label/attr + boundedness (enum/template/dropped); no denylisted IDs                                                                                                                                        | Verified / Failed                          |
| Review source                | `<agent id>` (`<scope reviewed>`) per reviewer, comma-separated; else "self-review (no subagent tool)"                                                                                                          | —                                          |
| Rule-file findings           | verbatim manifest cross-check: one line per step-3 manifest row — `§Section` + honored / violated / not applicable + why; a row the reviewer can't locate, or a routed file missing from the manifest, is a gap | Verified / Failed                          |
| Review verdict               | —                                                                                                                                                                                                               | Verified / Verified (self-review) / Failed |

### Example query

Question: <the question the query answers, or "does <signal> now exist as <shape>?">

```opal
<the query that answers the question, or confirms the signal's shape>
```

Reads as: <what one row/value means, or: rows present ⇒ change is live>

### Changes

One row per file touched; kind is `new`, `edited`, or `deleted`.

| File                         | Kind   | What changed                                                |
| ---------------------------- | ------ | ----------------------------------------------------------- |
| e.g. `src/otel/bootstrap.ts` | new    | SDK init: tracer + meter provider, OTLP exporter            |
| e.g. `src/server.ts`         | edited | wrapped outer handler; emits `http.server.request.duration` |

| Field        | Value                            |
| ------------ | -------------------------------- |
| Dependencies | added/changed, or none           |
| SDK setup    | where; confirm no duplicate init |

### Environment

| Field                | Value |
| -------------------- | ----- |
| Runtime              | <>    |
| Framework            | <>    |
| Package manager      | <>    |
| Module model         | <>    |
| Existing OTel/vendor | <>    |
| Observe/OTLP config  | <>    |

### Limitations & next steps

<limitations, gaps, and next steps; or none>
````

## The four verdict states

Every check resolves to exactly one — the report uses no other vocabulary:

- **Verified** — the check ran and passed.
- **Failed** — the check ran but did not pass (the signal was absent, or the shape /
  attributes were wrong).
- **Blocked** — could not run: a prerequisite or consent was missing (say which). If you can
  resolve it yourself, it isn't Blocked until you've tried. An unreachable agent, a missing
  datastream/token, or "no data in Observe" is a collection problem: deploy the agent with
  `deploy-*-explorer` from the current change and run `debug-*-collection` before reporting
  delivery as Blocked. Deliver through the observe-agent; don't bypass it with direct ingest.
- **Not run** — not executed (out of scope, or simply skipped).

"Not applicable" and "capability not implemented" are **scope notes**, not verification
outcomes; state them plainly.

## Filling the verdict cells

Every cell is one of the four states above. A claim is only as strong as the receipt
that proves it — a check proving `Verified` means the app code _emits_ the signal.
Prefer the strongest feasible proof, fall back down this ladder, and label honestly:

1. An **existing repo test** that exercises the real app code.
2. A **new/repaired repo test** exercising the real code path.
3. A **temporary app-code harness** driving the real function with fakes.
4. **Generated-SDK contract only** — manually creating an SDK span to show the exporter
   works. This proves the export/schema contract, **not** that application code emits
   telemetry — an exporter smoke test, last resort; label it contract-only.

A synthetic root span or focused call-site test cannot prove facts that only the real
process with its real bootstrap produces — the number, kind, name, or attributes of the
real server spans, framework-resolved route names, or duplicate-span suppression between
auto and manual instrumentation. Prove those with a test that drives the **real code
path**, or label them unproven.

## Example query

The "Example query" section is a required deliverable in every run. The OPAL must prove the
telemetry serves the intent (`../discovery/intent.md`): it either returns the answer to the
user's question, or confirms the signal now exists in the expected shape (right name,
attribute present, status set). Match the query to the intent you captured; if the intent
was to answer a question, a query that only shows the signal is present doesn't satisfy it.

Format it as three parts, so the query is self-explaining:

- `Question:` the user's question in plain English, or "does `<signal>` now exist as
  `<shape>`?" One line.
- the fenced ```opal block with the query itself.
- `Reads as:` what one returned row or value means (e.g. "one row per route, its p95
  request duration"), or "rows present ⇒ the change is live". One line.

## The Changes table

The `### Changes` section lists the code change as a table: one row per file touched,
nothing else. Keep it lightweight.

- `File` — the repo-relative path, in backticks.
- `Kind` — exactly one of `new`, `edited`, `deleted`.
- `What changed` — a short phrase (the instrument added, the seam wrapped, the init
  extended), not a diff or a prose paragraph.

Cross-cutting facts (dependencies, SDK-setup location) go in their own small table below
the Changes table, not as extra rows of it. A file that only gained a dependency still
gets a row.

## Language discipline

- Say telemetry "reached Observe" only when a query returned matching results.
- Claim the instrumented path "was hit" only with direct evidence.
- Report a skipped telemetry unit test as `Blocked` (a serious limitation), never as
  present.
- Prefer "verified that X" / "could not verify Y because Z" over a vague "done".
