# Duration and Time Operations in OPAL

## Duration Creation

### Constructor functions

-   `duration_hr(N)` — hours (e.g., `duration_hr(24)` → 24h)
-   `duration_min(N)` — minutes (e.g., `duration_min(15)` → 15m)
-   `duration_sec(N)` — seconds (e.g., `duration_sec(30)` → 30s)
-   `duration_ms(N)` — milliseconds (e.g., `duration_ms(500)` → 500ms)

### Duration literals

Literals express any time unit inline without a constructor function:

-   `1ns` — nanosecond
-   `1us` — microsecond
-   `1ms` — millisecond
-   `1s` — second
-   `1m` — minute
-   `1h` — hour
-   `1d` — day
-   `1w` — week

### Day and week durations

Use duration literals for day and week thresholds — they are the simplest and most readable approach:

    filter (now() - flag_changed) > 30d               // 30 days
    filter (now() - last_seen) > 7d                    // 1 week
    filter (now() - last_deploy) > 90d                 // 90 days
    make_col retention:1w                              // 1 week literal

Equivalently, multiply hours with a constructor:

    make_col month_threshold:duration_hr(24 * 30)      // ~30 days
    filter (now() - last_seen) > duration_hr(24 * 7)   // 7 days

### From strings and timestamps

    make_col d:parse_duration("2h30m")
    make_col elapsed:row_end_time() - row_start_time()

---

## Duration Extraction (to numbers)

-   `to_nanoseconds(d)` → int64
-   `to_milliseconds(d)` → float64
-   `to_seconds(d)` → float64
-   `to_minutes(d)` → float64
-   `to_hours(d)` → float64

---

## Constructors vs Extractors

`duration_*()` functions CREATE a duration from a number. `to_*()` functions EXTRACT a number from a duration.

| Create duration from number         | Extract number from duration   |
| :---------------------------------- | :----------------------------- |
| `duration_sec(30)` → 30s duration   | `to_seconds(d)` → float64      |
| `duration_min(15)` → 15m duration   | `to_minutes(d)` → float64      |
| `duration_hr(24)` → 24h duration    | `to_hours(d)` → float64        |
| `duration_ms(500)` → 500ms duration | `to_milliseconds(d)` → float64 |

Computing age as a number:

    // CORRECT — extract hours from a duration, then divide:
    make_col days_since_change:to_hours(now() - flag_changed) / 24

    // CORRECT — compare duration to duration:
    filter (now() - flag_changed) > duration_hr(24 * 30)

    // WRONG — duration_hr() takes a number, not a duration:
    make_col days_since_change:duration_hr(now() - flag_changed) / 24

---

## Worked Examples

### Stale feature flags (not updated in 30 days)

On Resource datasets, do NOT filter on valid-to (`filter is_null(row_end_time())`) — it usually returns zero rows. Use `coalesce(row_end_time(), now())` to measure age so no rows are dropped:

    make_col age:coalesce(row_end_time(), now()) - row_start_time()
    filter age > 30d
    make_col age_days:to_hours(age) / 24
    pick_col @."Valid From":row_start_time(), @."Valid To":row_end_time(), name, age_days
    topk 100, max(age_days)

### Elapsed time since last update

    make_col age_hours:to_hours(now() - row_start_time())
    make_col is_stale:(now() - row_start_time()) > 7d

### SLA compliance

    make_col response_time:row_end_time() - row_start_time()
    make_col within_sla:response_time < duration_ms(500)
    statsby total:count(), sla_met:sum(case(within_sla, 1, true, 0)), group_by(service_name)
    make_col sla_rate:round(100.0 * sla_met / total, 2)

### Time bucketing by hour of day

    make_col hour:format_time(row_start_time(), "HH24")
    statsby events:count(), group_by(hour)
    sort asc(hour)

### Time between events

    make_col prev_time:window(lag(row_start_time(), 1), group_by(service_name), order_by(row_start_time()))
    make_col gap_sec:float64(to_seconds(row_start_time() - prev_time))

---

## now() Function

    make_col age_hours:float64(to_hours(now() - row_start_time()))
    make_col is_stale:(now() - row_start_time()) > 24h

---

## Timestamp Formatting

    make_col time_str:format_time(row_start_time(), "YYYY-MM-DD HH24:MI:SS")

Format tokens use **Snowflake conventions** (NOT strftime): `YYYY`, `MM`, `DD`, `HH24`, `MI`, `SS`, `AM`/`PM`, `DY`, `MON`.

## Timestamp Parsing

    make_col ts:parse_isotime(string(timestamp_str))                        // ISO 8601
    make_col ts:parse_timestamp(string(time_col), "YYYY-MM-DD HH24:MI:SS") // custom format

## Epoch-to-Timestamp Conversion

    make_col ts:from_nanoseconds(epoch_ns)       // int64 nanoseconds → timestamp
    make_col ts:from_milliseconds(epoch_ms)      // int64 milliseconds → timestamp
    make_col ts:from_seconds(epoch_sec)          // int64/float64 seconds → timestamp

---

## Common Pitfalls

-   Using strftime format (`%Y-%m-%d`) → use Snowflake format: `YYYY-MM-DD`
-   Comparing duration to number → compare durations: `elapsed > duration_sec(5)` not `elapsed > 5`
-   Forgetting `string()` in parse_isotime → `parse_isotime(string(col))`
-   Passing a duration into a constructor → `duration_hr(now() - ts)` is wrong; use `to_hours(now() - ts)` to extract
