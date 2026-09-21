# `get_site_reader_health` — executable spec

**For:** the ZLP platform team (owners of the `zlp-prd-jpn` engineering-api MCP)
**From:** Kajima site survey project, 2026-09-01
**Companion docs:** `reader-health-definition.md` (what "healthy" means) · `crane-roster-ground-truth.md` (roster) · `grafana-reader-metrics-findings.md` (data source)

Every field below is **derived from a query that was run against `postgres-zlp` production on 2026-09-01** and returned the value quoted. Where something could not be validated, it says so explicitly.

---

## 1. Why this tool

The 32-tool reader/anchor surface is entirely per-node for diagnostics. Validating reader health across the Koto crane site currently costs **3N + 2 = 89 calls** for 29 readers, which rules out anything faster than a daily cadence.

Every field this tool would return already exists in `public.device_metrics_history`. What is missing is the aggregation. One call replaces 89.

**Secondary reason, and arguably the stronger one:** three of the fields below (`longest_gap_s`, `flap_count`, `ingest_skew_max_s`) **cannot be computed from the existing tools at all**, because `get_anchor_status.last_heartbeat` is a formatted string (`"10h ago"`), not a timestamp. Gap and density analysis is impossible from the current surface regardless of call budget.

---

## 2. Tool contract

```
name:   get_site_reader_health
group:  Site Health Roll-up
scope:  engineering:read
```

**Parameters**

| Name | Type | Default | Notes |
|---|---|---|---|
| `site` | string, required | — | Site name or wifi-cloud site ID |
| `range` | string | `last_24h` | Same shortcuts as `get_anchor_metric_history` |
| `start_time` / `end_time` | ISO 8601 | — | Explicit window, mutually exclusive with `range` |
| `include_unsurveyed` | boolean | `false` | When false, anchors with no surveyed x/y are excluded from `readers[]` and counted in `summary.unsurveyed` |

**Returns**

```jsonc
{
  "site": "Koto_Pumping_Station_Crane",
  "site_res_name": "875ca2f3-64bf-4931-8d6a-fe0ea2b6c784",
  "window": { "start": "...", "end": "...", "expected_samples": 1440 },
  "summary": {
    "roster_total": 48,        // all anchor records
    "unsurveyed": 19,          // ghost records, excluded from grading
    "monitored": 29,           // the denominator that matters
    "reporting": 28,
    "healthy": 0               // computed by the caller, not the tool
  },
  "readers": [
    {
      "name": "r55_d299",
      "node_id": "…",
      "mac": "c4ac59…",
      "surveyed": true,
      "position": { "x": 18760, "y": 15040, "z": 4500 },

      "samples": 1426,              // criterion 1a / 3
      "uptime_pct": 99.0,
      "longest_gap_s": 121,         // NOT derivable from existing tools
      "flap_count": 0,              // gaps > 600 s
      "last_sample_age_s": 47,      // integer seconds, NOT "10h ago"

      "temp_avg_c": 31.4,           // criterion 1b / 1c
      "temp_max_c": 38.9,
      "temp_sd_c": 2.31,            // stuck-sensor test
      "temp_peak_hour_jst": 16,

      "ranging_events": 5663,       // criterion 2b
      "trackers_max": 9,
      "aborts": 0,                  // criterion 2d

      "firmware": "v1.7.8-0-g8cfcc1a",
      "ntp_synced": true,
      "ingest_skew_max_s": 66       // backfill detector, see §5
    }
  ]
}
```

**The tool returns measurements, not grades.** Thresholds live in `reader-health-definition.md` and are contested (three different heartbeat numbers are in play). A tool that bakes in a grade forces a threshold decision that has not been made.

---

## 3. Field provenance — every field, with the metric it comes from and the evidence

All validated 2026-09-01 against site `875ca2f3-…`, 24 h window, 28 reporting readers.

| Field | Source | Validated evidence |
|---|---|---|
| `samples`, `uptime_pct`, `longest_gap_s`, `flap_count`, `last_sample_age_s` | `sensors.tempC` timestamps + `lag()` window | 2026-08-25: 28/28 readers ≥ 99 % uptime, **0** with a gap > 10 min, **0** silent > 10 min |
| `temp_avg_c`, `temp_max_c`, `temp_sd_c` | `sensors.tempC` value | 0 readers with σ < 0.1 °C |
| `ranging_events` | **`latency.count`, summed** | n=39,920 samples / 28 devices / 24 h · avg 3.97 · max 27 · **sum 158,579** → ≈ 5,663 events per reader per day |
| `trackers_max` | `trackers` | avg 2.11 · max 9 · sum 84,165 |
| `aborts` | `aborts` | **avg 0.00 · min 0 · max 0 · sum 0** — see §4 |
| `ingest_skew_max_s` | `stamp` (device epoch ms) vs `metricTime` | n=39,917 · min −65.5 s · **avg 0.0 s** · max +65.7 s · σ 37.8 s · **0 samples over 5 min** |
| `surveyed`, `position` | `readers.location` jsonb | Keys confirmed: **`x, y, z`**. Ghost filter is `location->>'x' IS NOT NULL` |
| `expected_samples` | `duration` metric | avg **60,234 ms** · min 60,000 · max 60,547 → the reporting interval is **exactly 60 s** |
| `firmware`, `ntp_synced` | Not in `device_metrics_history` | Must come from the same source `get_anchor_status` uses. **Not validated here.** |
| `temp_peak_hour_jst` | Derived from `sensors.tempC` bucketed by hour | Computable; not yet run |

### The 60-second finding settles an open question

`duration` averages 60,234 ms with a 547 ms spread. **The device reports every 60 seconds.** That means a `< 12 h` heartbeat threshold tolerates **720 consecutive missed reports**, and the tool reference's `> 1 h` warning tolerates 60. Both numbers should be revisited against this, not against the display format of `last_heartbeat`.

---

## 4. Two data-quality findings the platform team should know

**`aborts` is identically zero.** Across 39,920 samples and 28 devices over 24 h: avg 0, min 0, max 0. Either the site genuinely never aborts a ranging attempt, or the metric is not populated. **Criterion 2d (yield: accepted vs attempted) cannot be graded on this field until someone confirms which.** Worth checking against another site before building anything on it.

**`stamp` is an unadvertised device-side timestamp.** It is epoch milliseconds from the device's own clock. Comparing it against `metricTime` gives two things for free:

- **backfill detection** — a buffered hub flush would show `metricTime − stamp` in the hundreds or thousands of seconds
- **clock-sync validation** — a device with a drifted clock shows persistent skew

Measured today: **±66 s, mean 0, zero samples beyond 5 min.** So no backfill is occurring right now and device clocks are synced. This closes conflict #4 in `reader-health-definition.md` with evidence rather than assumption, and it gives the tool a cheap ongoing check that nothing in the current surface provides.

---

## 5. Reference implementation — validated SQL

**Run as two queries, not one.** See §6 for why.

### Q1 — heartbeat density, gaps, flaps

```sql
WITH raw AS (
  SELECT "deviceResName" AS dev, "metricTime" AS t,
         lag("metricTime") OVER (PARTITION BY "deviceResName" ORDER BY "metricTime") AS prev
  FROM public.device_metrics_history
  WHERE "siteResName" = $1::uuid
    AND "metricName" = 'sensors.tempC'
    AND "metricTime" > now() - $2::interval
)
SELECT dev,
       count(*)                                              AS samples,
       max(t)                                                AS last_sample,
       round(max(EXTRACT(EPOCH FROM (t - prev)))::numeric, 0) AS longest_gap_s,
       count(*) FILTER (WHERE EXTRACT(EPOCH FROM (t - prev)) > 600) AS flap_count
FROM raw
GROUP BY 1;
```

### Q2 — value aggregates

```sql
SELECT "deviceResName" AS dev,
  round(avg("metricValue")    FILTER (WHERE "metricName"='sensors.tempC')::numeric, 2) AS temp_avg_c,
  round(max("metricValue")    FILTER (WHERE "metricName"='sensors.tempC')::numeric, 2) AS temp_max_c,
  round(stddev("metricValue") FILTER (WHERE "metricName"='sensors.tempC')::numeric, 3) AS temp_sd_c,
  round(sum("metricValue")    FILTER (WHERE "metricName"='latency.count')::numeric, 0) AS ranging_events,
  round(max("metricValue")    FILTER (WHERE "metricName"='trackers')::numeric, 0)      AS trackers_max,
  round(sum("metricValue")    FILTER (WHERE "metricName"='aborts')::numeric, 0)        AS aborts,
  round(max(abs(EXTRACT(EPOCH FROM "metricTime") - "metricValue"/1000.0))
        FILTER (WHERE "metricName"='stamp')::numeric, 0)                               AS ingest_skew_max_s
FROM public.device_metrics_history
WHERE "siteResName" = $1::uuid
  AND "metricTime" > now() - $2::interval
  AND "metricName" IN ('sensors.tempC','latency.count','trackers','aborts','stamp')
GROUP BY 1;
```

### Q3 — roster and ghost filter

```sql
SELECT r."readerResName" AS dev, r.name, r."macAddress" AS mac,
       (r.location->>'x') IS NOT NULL AND (r.location->>'y') IS NOT NULL AS surveyed,
       r.location->>'x' AS x, r.location->>'y' AS y, r.location->>'z' AS z
FROM public.readers r
WHERE r."siteResName" = $1::uuid;
```

> **Do not filter on `readers.status`.** All 129 readers in the entire ZLP production database are `ACTIVE`; `INACTIVE`, `DELETED` and `ACCEPT_PENDING` have never been used. The field is unmaintained and cannot distinguish live hardware from inventory. The positional test above is the working ghost filter — validated: 48 records, 29 surveyed, 19 not, and connected status correlates perfectly with surveyed position.

---

## 6. Performance — the constraint that motivates a server-side tool

**Q1, Q2 and Q3 each complete individually within the Grafana ingress timeout.** Fusing them into one CTE chain with a final aggregate **returned `504 Gateway Time-out` twice**, including after narrowing the scan to the five metric names actually used.

The window function over ~40 k rows plus the per-row `EXTRACT`/`abs` on `stamp` puts the fused query at the boundary of the ~60 s ALB timeout.

Implications for whoever builds this:

1. **A naive single query will not hold at 30 readers, and the site is meant to grow.** Excavation has 37 roster records.
2. It needs a **covering index** on `device_metrics_history ("siteResName", "metricName", "metricTime")` if one does not exist, or a rollup/continuous aggregate.
3. Compute the gap analysis in the database (window function), not by shipping raw rows to the client.
4. This is exactly the argument for a **server-side tool** rather than leaving every consumer to write their own ad-hoc query against a table this shape.

---

## 7. Acceptance tests

Run against `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`, `range=last_24h`. Values from 2026-08-25 / 2026-09-01 production.

| # | Assertion | Expected |
|---|---|---|
| 1 | `summary.roster_total` | 48 |
| 2 | `summary.unsurveyed` | 19 |
| 3 | `summary.monitored` | 29 |
| 4 | `summary.reporting` | 28 (the gap is `r59_d2d8`) |
| 5 | `readers[].last_sample_age_s` | integer seconds, **not** a formatted string |
| 6 | Median `samples` | ≈ 1426–1440 (60 s cadence) |
| 7 | Readers with `longest_gap_s > 600` | 0 |
| 8 | Readers with `temp_sd_c < 0.1` | 0 |
| 9 | Median `ranging_events` | ≈ 5,663 — all readers well above the reference's 100/24 h floor |
| 10 | `sum(aborts)` across site | 0 — flag if non-zero, the metric may have started working |
| 11 | `max(ingest_skew_max_s)` | ≤ 70 (one reporting interval). > 300 means backfill |
| 12 | `include_unsurveyed=true` | returns 48 rows; ghosts have `surveyed:false` and null metrics |
| 13 | Latency, whole call | < 5 s for 48 readers / 24 h |
| 14 | Unknown site | structured error, not empty success — an empty roster must never read as "healthy" |

Test 14 matters most. The May 2026 outage stayed hidden for 13 days partly because "no data" and "no problem" were indistinguishable.

---

## 8. Two smaller asks against existing tools

**`get_anchor_ranging_history` needs an aggregate mode.** It returns raw UWB measurements. At ~5,663 events per reader per day, a 24 h site-wide pull is ~164,000 events — beyond the ~150,000-character hosted tool-result cap. A `granularity` parameter returning counts per bucket would make criterion 2 usable without the roll-up tool at all.

**`get_anchor_metric_history` should expose the ingestion timestamp** alongside the event timestamp. `stamp` gives this indirectly today, but only if the caller knows it exists and knows it is device epoch ms — neither is documented.

---

## 9. Validation status of the five-tool minimum set

**The five MCP tools have NOT been validated. They cannot be, from here.**

`ListConnectors` on 2026-09-01 returns the org's full connector list: Atlassian Rovo, Canva, Circleback, Figma, Gmail, Google Calendar, Google Drive, Gusto, Mixpanel, Slack. **There is no ZLP / engineering-api connector installed at the organization at all** — this is not "connected but disabled for this chat," it is absent.

So every statement about `list_readers`, `list_stale_anchors`, `get_anchor_status`, `get_anchor_metric_history` and `get_anchor_ranging_history` is read off `reader_anchor_mcp_tools_reference.md`, not off a live response. What *has* been validated is the underlying data those tools read — via the `postgres-zlp` Grafana datasource, which is the same database.

**To actually validate them**, per `eng-api-mcp-connector-setup.md`, either:

- **Claude Code on a networked machine** — `claude mcp add zlp-prd-jpn --transport http https://engineering-api-mcp.zlp-prd-jpn.zainar.net/mcp --header "Authorization: Bearer <viewer-scoped key>"`. Per-user credential, own audit identity, no admin needed, works today. **This is the fast path.**
- **An org custom connector** — needs an org Owner plus the `static_headers` beta. Only path that enables unattended scheduled runs.

**Validation checklist once connected** (read-only throughout):

1. `who_am_i` → confirm the key resolved to `viewer`, not write/admin.
2. `list_readers(site_res_name=875ca2f3-…, page_size=100)` → expect 48–49 records; check whether the response carries surveyed position (if it does, the ghost filter needs no second call).
3. `list_stale_anchors(site=…, stale_hours=0.5)` → expect 19–21; compare against the ghost list.
4. `get_anchor_status(node='r55_d299')` → **capture the literal `last_heartbeat` value** and confirm whether it is a string or a timestamp. This single field decides whether criterion 3 is possible from the MCP at all.
5. `get_anchor_status(node='r59_d2d8')` → the best test case on site: surveyed, connected in the registry, no firmware version, and silent on `sensors.tempC`.
6. `get_anchor_metric_history(node='r55_d299', metric_name='sensors.tempC', range='last_24h')` → count samples, expect ≈1440; confirm whether an ingestion timestamp is exposed.
7. `get_anchor_ranging_history(node='r55_d299', range='last_1h')` → **measure the payload size before trying 24 h.** Expect ~236 events/hour.
8. `generate_anchor_list_csv(site=…)` → inspect the columns. If it carries position and heartbeat age it collapses steps 2 and 3 into one call.

**Do not call any `engineering:write` tool during validation.** The reference states `zlp-prd-jpn` has write+admin permissions; `confirm:false` is a guardrail against accidents, not against a wrongly-scoped key.
