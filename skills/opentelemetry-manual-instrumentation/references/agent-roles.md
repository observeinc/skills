# Independent reviewer (tool-agnostic)

This file defines the independent adversarial review from SKILL.md step 5, which owns the
receipt rule. A fresh-context reviewer that did not write the patch catches what the
author's context rationalized, and returns only findings to keep the main context lean.
When your runtime exposes no subagent tool, run it as a self-review in the same context and
mark it as such in the report — a weaker check than an independent one.

## Independent adversarial reviewer — required for any change that emits/modifies fields

Run it as a separate subagent with full reasoning capability (same model tier as the main
agent) and fresh context that did not write the patch. It adversarially refutes the six
checks in the review prompt below: semantic conventions, sensitive data & cardinality, that the
signal fires, test completeness, safe implementation, and intent.

The receipt is the subagent's id/handle plus its verbatim findings, including one line per
routed rule file the reviewer checked, naming the specific rule and whether the patch honors
it (e.g. "`sensitive-data.md`: `user.email` dropped"). A valid review cites per-file
rules and comes from an independent context; a self-review has no agent id to paste. It
validates the Semantic review and Sensitive data & cardinality checks and verifies both step-3
reading manifests row by row. The reviewer opens every file in the prompt's **Read first**
list, plus every As-relevant and SDK file named in the second manifest.

**Review prompt (hand this to the subagent verbatim):**

> You did not write this patch — review it adversarially. Try to **refute** each check
> below; report PASS/FAIL with the specific defect, and default to FAIL when unsure.
>
> **Read first — cite the specific rule from each; do not rely on memory:**
> `review/semantic-conventions.md`, `review/semantic-review.md`,
> `review/cardinality.md`, `review/sensitive-data.md`,
> `instrumentation-rules.md`, `anti-patterns.md`, `discovery/intent.md`.
>
> **Checks to refute:**
>
> 1. **Semantic conventions**: every emitted field's name, unit, span kind, and placement
>    matches the convention (or is a justified, namespaced custom field).
> 2. **Sensitive data & cardinality**: every metric label is bounded; no PII, secret, or
>    high-cardinality value is used as a label.
> 3. **It fires**: the span/metric is emitted by the real code path and proven by a test
>    that drives it, not assumed.
> 4. **Tests are complete**: every signal and every distinct path (success, error, empty)
>    has a test driving it, not just the happy path.
> 5. **Implementation is safe**: the instrumenting code is bounded, idiomatic, and safe
>    per `instrumentation-rules.md` and `anti-patterns.md`; it changes no application
>    behavior and adds no duplicate SDK init.
> 6. **It serves the intent**: the example query can be written against this shape and
>    proves the intent, by returning the answer the user asked for or confirming the signal
>    now exists in the expected shape (`discovery/intent.md`).
>
> **Output:**
>
> - One line per check above: PASS/FAIL + the specific defect.
> - One line per file in "Read first": `§Section` + whether the patch honors or violates
>   it, or `§Section — not applicable because <why>`. Every file gets a line.
> - Manifest check: for each row of both step-3 reading manifests, open the named file,
>   find the cited `§Section`, and flag any row where the section or rule can't be
>   located — including "not applicable" rows, which must also cite a `§Section`
>   confirming the conclusion.

## Rules

- The reviewer is **read-only**; route any fix it prompts through the normal edit path.
- The reviewer never invents rules — it reads the referenced files and applies them.
- The reviewer obeys the SKILL.md hard constraints and the routed rule files.
