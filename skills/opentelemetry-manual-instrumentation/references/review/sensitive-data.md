# Sensitive Data

Leaking sensitive data causes real harm. This file governs _what must never be
recorded_ and _how to sanitize what might be_. It complements
`cardinality.md` (which owns the cardinality math). Every emitted field —
span name, attribute, event, log body/field, metric label, resource attribute —
passes this review.

## Never-instrument list (any signal, no exceptions)

Never record these in telemetry, even in hashed form:

- **Credentials** — passwords, API keys, bearer/session tokens, private keys.
- **Full auth headers** — `Authorization`, `Cookie`, `Set-Cookie`.
- **Financial** — full card numbers (PAN), CVV, bank account numbers.
- **Government IDs** — SSN, passport, tax IDs.
- **Health / biometric** — medical records, biometric identifiers.

If a user insists a credential-adjacent value is needed, offer a masked or hashed
form and document it.

## High-risk fields (record only under conditions)

| Field                            | Condition to record                                             |
| -------------------------------- | --------------------------------------------------------------- |
| `user.id` / `enduser.id`         | Only if **opaque** (an internal id) — never a username or email |
| IP address (`client.address`)    | Only if needed; truncate or hash                                |
| email / username                 | Never as an attribute                                           |
| `url.full`                       | Strip the query string (or redact its values) before recording  |
| request / response body          | Never as an attribute — record a **size** and/or content type   |
| `db.statement` / `db.query.text` | Only if parameterized (no inlined literals)                     |

## Sanitization — decision order

Apply the **earliest** option that works; later options are progressively wider
safety nets, each best paired with source-level care:

1. **Library / instrumentation option** — many instrumentations expose a sanitizer
   or "capture" toggle (redact URL query, disable body capture, parameterize SQL).
   Prefer it.
2. **A span/log processor** — redact or drop sensitive attributes in `onEnd`/on
   emit, before export. Register it **before** the export processor.
3. **Collector redaction** — a central `redaction`/`transform` stage, applied after
   data has left the process and crossed the network.
4. **Disable capture** — if none of the above can make a field safe, don't capture
   it.

**Derived substrings.** Deriving a safe substring from PII (email → domain via
`split('@').last`) needs a guard: require the delimiter; if it's absent, omit the
attribute. Without the guard, the fallback returns the raw value.

## Regex safety-net (defence in depth, not exhaustive)

As a last line, redact obvious secrets in free-form text (log messages, error
strings): credit-card and SSN patterns, emails, JWTs (`eyJ…`), and API-key prefixes
(`sk_`, `pk_`, `key_`, `token_`).

## Privacy spans the whole pipeline

A **formatter**, logging **adapter**, **MDC / context var**, framework **access
log**, or **exception renderer** can re-add an ID downstream of the `logger.*` call
that dropped it. Review each signal's _final rendered output_. See
`../signals/logs.md`.
