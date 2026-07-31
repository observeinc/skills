# Instrumentation Intent (fitness for purpose)

Telemetry must serve the reason for the change. A signal can pass every conformance check —
convention-clean, low-cardinality, privacy-safe, tested — and still not do what the user
wanted. Establish the intent before you choose a signal, and prove the telemetry serves it
before you call the task done.

**Every change has an intent; an action is not one.** "Rename this span" is an action. Its
intent is the why beneath it: "align the span name with semconv." Record the why, not the
action, and never call a change intent-free. A trivial intent is still an intent, and every
intent yields an example OPAL query. If you can't state the intent and write the query that
proves it, you don't understand the change yet.

## Capture the intent

Intent lives in what the user wants to do with the telemetry, not in the diff. The same code
(`tenant.id` on a span) is one-request debugging in one case and a per-tenant dashboard in
another, and that choice drives the signal, the instrument, and whether the value must be
bounded. Don't infer intent from the change; get it from the user.

- **Stated and Clear**: the user named the use case (a question to answer, a dashboard it feeds, a
  correctness fix). In this case we can directly use it.
- **Unstated or Ambiguous**: ask one question before picking a signal: **"What will you use this for: a
  question you want to answer or chart, or a correctness/shape fix?"** Don't guess. If the
  answer stays ambiguous (you can't tell whether the value must be bounded or which
  instrument it needs), ask again.

Write the intent as the why: "align span name with semconv", not "rename span".

## Prove the intent with one example OPAL

Every intent gets one example OPAL query showing the telemetry serves it. The query does
whichever the intent calls for, no query means you haven't pinned down the intent.
