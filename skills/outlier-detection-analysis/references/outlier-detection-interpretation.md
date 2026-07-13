# Outlier Detection — Interpreting Results

The pipeline returns up to 50 rows, one per `(attribute, value)` candidate, sorted by `phi` descending. Your job is to translate this ranked table into actionable root-cause insight.

## Phi coefficient bands

Phi measures the statistical correlation between a field value and the bad cohort. Range `[-1, +1]`; values closer to ±1 mean stronger correlation. The bands below apply to the absolute value of phi — apply the same thresholds to negative phi values, treating them as "correlation with the good cohort" (see note below).

| abs(phi) | Strength     | Interpretation                                                |
| :------- | :----------- | :------------------------------------------------------------ |
| > 0.5    | Very strong  | Highly likely root cause or major contributing factor         |
| > 0.3    | Strong       | Probable contributing factor — investigate                    |
| > 0.1    | Moderate     | Worth investigating, especially with high `bad_count`         |
| < 0.1    | Weak / noise | Usually safe to ignore unless paired with very high frequency |

A negative phi means the value is correlated with the **good** cohort (i.e., it protects against bad behavior). Surface these too — they are useful when comparing service variants or canary deployments.

## Reading bad% alongside phi

Always read the phi coefficient together with `frequency_in_bad_cohort` (a.k.a. "bad %"):

| Phi  | Bad % | What it means                                                                                                  |
| :--- | :---- | :------------------------------------------------------------------------------------------------------------- |
| High | High  | Strong, consistent correlation — top candidate                                                                 |
| High | Low   | Strong but rare — value is highly predictive when present, but only present occasionally (look at `bad_count`) |
| Low  | High  | Common but weak — appears in most bad rows but also in most good rows                                          |
| Low  | Low   | Background noise                                                                                               |

## Picking semantic root causes

Not every high-phi value is meaningful. Filter down to values that are semantically interpretable in observability:

- Error / exception types (`TimeoutException`, `NullPointerException`, `500`, `503`)
- Service / component names that suggest a failing service
- Resource attributes (host, container, pod, region, AZ) that suggest infra issues
- Database operations / endpoints / route patterns
- Feature flags, deployment IDs, build versions

Skip very high cardinality unique-per-request identifiers — they are rarely root causes:

- `request_id`, `trace_id`, `span_id`, `correlation_id`
- `user_id`, `session_id` (unless the question is about a single user)
- UUIDs, hashes, signed-URL tokens

## Recommended response format

Structure the response as follows.

### 1. Headline summary

One sentence stating the most likely root cause and the supporting evidence.

> The strongest correlation with slow checkout requests is `service.name = payment-api` (phi=0.71, 88% of slow requests).

### 2. Ranked table (top 5–10 candidates)

| Rank | Attribute    | Value            | Phi  | Bad % | Notes                                           |
| :--- | :----------- | :--------------- | :--- | :---- | :---------------------------------------------- |
| 1    | error_type   | TimeoutException | 0.82 | 95%   | Very strong; almost exclusive to bad cohort     |
| 2    | service.name | payment-api      | 0.71 | 88%   | Strong; consistent with a payment-service issue |
| 3    | region       | us-west-2        | 0.45 | 72%   | Strong regional correlation                     |

### 3. Root-cause narrative

Two or three sentences synthesizing the table into a hypothesis.

> Timeouts originating from `payment-api` in `us-west-2` account for the majority of slow checkout requests. The combination of `error_type`, `service.name`, and `region` correlations suggests a localized timeout in the payment service's us-west-2 deployment, rather than a global slowdown.

### 4. Recommended next steps

Concrete actions, each tied to an investigation a human can run:

- Drill into `service.name = payment-api` traces in `us-west-2` for the last hour.
- Check recent deployments / config changes for the payment service in that region.
- Compare `payment-api` p99 latency in `us-west-2` vs other regions.

If the host environment provides capabilities to run these follow-ups (e.g., creating query cards, running additional traces), surface them as actionable suggestions.

## Common pitfalls when interpreting

- **All values come from one field** — if every top-N row is `attributes.<something>`, propose narrowing future analyses with a `fieldFilter` to surface secondary correlations.
- **Empty / `__empty__` paths** — these come from the empty-fix step and indicate that complex field was null/empty for some rows. Treat as a real signal: "missing X correlates with bad behavior."
- **Identical phi across many rows** — usually means those values co-occur perfectly (single underlying event); collapse them in the narrative.
- **Tiny `total_bad` or `total_good`** — the cohort is too small for stable statistics. Tell the user and suggest broadening the time range.
- **All values have phi near 0** — there is no statistical correlation; the bad behavior is uniformly distributed. Recommend changing the threshold or the time window.

## Worked example

Input: dataset = OTel spans for the checkout flow, threshold = `duration > 482ms` (P95).

Output (top rows):

| Rank | Attribute                    | Value              | Phi  | Bad % |
| :--- | :--------------------------- | :----------------- | :--- | :---- |
| 1    | `attributes.error.type`      | `TimeoutException` | 0.82 | 95%   |
| 2    | `service.name`               | `payment-api`      | 0.71 | 88%   |
| 3    | `resource_attributes.region` | `us-west-2`        | 0.45 | 72%   |

Response:

> **Likely root cause:** TimeoutException errors from `payment-api` in `us-west-2` (phi=0.82, present in 95% of slow requests).
>
> **Evidence chain:** The timeout error type (phi=0.82) co-occurs with `payment-api` (phi=0.71) and is concentrated in `us-west-2` (phi=0.45). All three correlations are above the "strong" threshold (|phi| > 0.3), and the timeout error appears in 95% of slow requests vs the global baseline.
>
> **Next steps:** Inspect `payment-api` traces in `us-west-2`, check for recent payment-service deployments, compare regional p99 latency.
