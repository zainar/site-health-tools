# What counts as a healthy reader — Koto crane site

**Status:** revision 2, 2026-09-01. Supersedes the placeholder in `kajima-site-monitor-spec.md` §3.
**Scope:** WiFi R3 readers at `Koto_Pumping_Station_Crane` (`875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`). Same structure should transfer to Excavation (`e0a6a862-56cc-4281-ac18-0d3fa3cdc4bc`).

**Revision 2 changes:** mapped to the real MCP tool surface (`reader_anchor_mcp_tools_reference.md`, 2026-09-01); minimum tool set defined; heartbeat moved from criterion 1 to criteria 0 and 3; roster resolved against `crane-roster-ground-truth.md`; three conflicts with the tool reference's own criteria recorded in §Conflicts.

> **Terminology:** reader and anchor are the same device. "Reader" is the ZLP/Prisma term, "anchor" the WiFi-RT term. The tools use both.

---

## The frame

Misha's starting proposal was three criteria:

1. Historical temperature continuous over 24h → hardware was ON and the temperature sensor works
2. Reader is participating in locationing
3. Reader has a stable internet connection

That is the right skeleton. Three things need fixing before it becomes a definition you can grade against.

### Problem 1 — all criteria are cloud-side, and none can tell you *where* the fault is

Temperature continuity, locationing participation and connection stability are all observed from the cloud. During the May 2026 outage **all 30 readers were alive and answering local pings while every cloud view was dark.** All three checks would have failed, and all three would have been failing for a reason that had nothing to do with the readers.

So the definition must be stated honestly: *healthy as observed from the cloud.* And it needs a grade that is neither healthy nor unhealthy:

| Grade | Meaning |
|---|---|
| 🟢 **HEALTHY** | Meets all criteria below over the evaluation window |
| 🟡 / 🟠 / 🔴 | Fails one or more criteria — see per-criterion thresholds |
| ⚫ **UNKNOWN** | We could not observe it. API down, site uplink down, roster unresolved |

**A site-wide UNKNOWN is a different fact from a site-wide UNHEALTHY, and both are different from healthy.** Conflating the first two is what let a 13-day outage stay hidden. Never collapse them.

### Problem 2 — the definition is missing the dangerous state

A reader can be powered, reporting temperature every 60 s, and contributing ranging measurements — while being **wrong**. Mis-calibrated position, drifted Z, wrong LPS address, firmware out of step with the rest of the roster. Crane #5's anchor tag has already physically shifted during operation once.

That reader passes all three of the proposed criteria and quietly poisons the location solution that keeps workers out from under crane loads. It is worse than an offline reader, because an offline reader is visibly absent and this one is not. **Trustworthiness has to be a criterion, not an afterthought.**

### Problem 3 — heartbeat is not a criterion-1 signal

Per Misha, a reader heartbeat indicates: the reader communicated with the backend/hub, the connection was active at that timestamp, and the network path is working.

That is a statement about **the path**, not about the device. It belongs to criterion 3 (stable connection) and criterion 0 (reachable at all). Temperature stays criterion 1's signal because it proves something heartbeat cannot: that the sensor works and the box is thermally sane. The two are complementary, not redundant.

It also means **heartbeat can never satisfy criterion 2.** A reader can heartbeat every minute while contributing zero ranging measurements — unsynced, or wrong LPS address.

---

## The definition

A reader is **healthy** when all five hold over the evaluation window.

Evaluation window: **rolling 24 h, weighted to JST working hours (08:00–18:00 Mon–Sat).** Overnight quiet is not evidence of anything at this site — the documented failure mode is daytime-only, peaking ~16:00 JST.

---

### 0. Present — it is on the roster, and reachable at all

**Roster of record** (from `crane-roster-ground-truth.md`, 2026-08-25 registry export): **48 anchor records, 29 real.** The 29 are `r31`, `r32`, and the contiguous run `r40`–`r66`. The other 19 (`r20`–`r30`, `r33`–`r39`, `r67`) are ghost records.

**The ghost filter is positional, not temporal.** Connected status and surveyed position correlate perfectly across all 48 records: every connected anchor has full x/y/z, every disconnected one has a z-only placeholder or nothing. A reader that is surveyed in and later goes dark **keeps its coordinates**. So:

> **An anchor with no surveyed x/y is inventory, not a monitored reader.**

This matters: it is a property of the record, not of the signal being monitored. An earlier proposal to exclude readers silent for >100 days was rejected because the exclusion criterion would have been the same signal being graded — a permanently dead reader would age out of the denominator and the site would return to green. The positional test has no such feedback loop.

**Do not use `list_readers(status=...)` as the ghost filter.** Every reader in the ZLP production database — all 129, every site — is `ACTIVE`. `INACTIVE`, `DELETED` and `ACCEPT_PENDING` have never been used. The field is unmaintained.

**Reachability gate: last heartbeat within 12 h.** Beyond that, the reader drops off the monitored roster and onto the inventory review list — it is not graded as unhealthy, it is graded as *not currently a monitored reader*, and the count is reported separately. This is the correct home for a coarse threshold (see §Conflicts on the specific value).

Ghost count and gated-out count are **hygiene numbers, reported every run, never folded into the health score.** A tool that reports "48 anchors, 46 stale" every run gets muted within a week — the exact failure the incident report's lesson #1 warns about.

### 1. Alive — powered, and the sensor works

**1a. Continuity.** `sensors.tempC` samples present with no gap exceeding the documented SLA: **~60 s expected · warn > 10 min · error > 30 min.**

- Measured 2026-08-25: all 28 live readers reported ~1/min; longest single gap across the whole roster in 24 h was under 10 min. The hardware comfortably meets this.
- Continuity proves **power AND the metrics path**, jointly. It cannot separate them. A gap means "not reaching the cloud" — never report it as "the reader is dead."

**1b. Validity — continuity alone is not enough.** Production has already produced the counter-examples:

| Test | Threshold | Why |
|---|---|---|
| Not stuck | σ > 0.1 °C over the window | A gapless series with zero variance is a cached value or a dead sensor |
| Not zeroed | not all-zero | The tool reference names **temperature sensor I2C failure** as a critical fault, and all-zeros as its signature |
| In range | within the R3's rated operating range | `r47_b51b` reported avg 561,445 hPa / max 5,369,470 on the pressure channel — continuous, and nonsense |

**1c. The temperature value is a leading indicator and is currently discarded.** The documented root cause at this site is thermal — cellular modem, peaking 16:00 JST, construction-equipment noise. Today only the gaps are used. Add per-reader absolute temperature, daily peak and its hour, day-over-day drift in that peak, and cross-reader spread (readers heating faster than the rest localises the problem physically). **Grade WATCH on a rising afternoon peak before it fails, not after.** This is the one thing that could move this site from reactive to predictive.

### 2. Contributing — it is participating in locationing

**2a. Synced.** In the LPS timing group with a valid address (`check_anchor_lps_address`), and system clock NTP-synced (`get_anchor_status.system`). An unsynced reader is online and contributes nothing — it looks alive and is functionally absent.

**2b. Contributing measurements.** Ranging events present over the window (`get_anchor_ranging_history`).

**2c. Normalised to the reader's own baseline, not to the site.** A corner reader legitimately participates far less than a central one. Absolute thresholds either cry wolf on the perimeter or miss a failure in the middle.

> **Healthy = ranging rate ≥ 50 % of that reader's own trailing 28-day median for the same hour-of-day**, evaluated only in periods with ≥ 1 live tag in coverage. < 50 % = WATCH · < 20 % = AMBER · zero across a working day with live tags present = RED for that reader.
>
> **Floor, regardless of baseline: < 100 ranging events in 24 h = RED.** (The tool reference's own criterion — kept as an absolute floor, not as the grading rule. See §Conflicts.)

`get_anchor_metric_history(range="last_30d")` is available, so the 28-day baseline is buildable — this was an open question in revision 1 and is now closed.

**Gate on the denominator.** Zero contribution at 03:00 with nobody on site is not a fault: no live tags in coverage → the criterion is *not evaluated*, not *failed*.

**2d. Yield, not just volume.** Accepted vs attempted. A reader throwing ranging errors at high volume looks busy and is failing. The `aborts` and `latency.*` metrics exist per reader; `get_tag_status` decodes the tag-side flags bitmask, where `0x0100 ranging_error` and `0x0400 hub_not_found` change the diagnosis.

### 3. Stable — connection quality over time, not a binary

Four numbers per reader per window:

| Metric | Definition | Healthy |
|---|---|---|
| **Uptime fraction** | samples received ÷ expected, over JST working hours | ≥ 99 % |
| **Longest single gap** | max continuous silence in the window | < 10 min |
| **Flap count** | distinct disconnect→reconnect events | ≤ 2 / day |
| **Recovery** | returned without human intervention, and how fast | self-recovered, < 5 min |

**Grade on density, not on age.** A heartbeat proves the path worked *at that timestamp*. What matters over a window is how many arrived versus how many should have. Age-of-last only catches a reader dark *right now* — and runs currently land ~08:00 JST, so a reader dark from 08:30 to 17:30 the previous working day has a fresh heartbeat and reads healthy. **The failure heals before the check looks at it.** This is not hypothetical: daytime-only failure with overnight recovery is this site's documented signature.

Uptime alone is misleading: 99 % with one 20-minute hole during a lift is worse than 99 % spread over a hundred 5-second blips. **Report the longest gap alongside the fraction, always.**

**Recovery deserves its own weight.** There is a documented firmware bug where readers do not reconnect after the ISP recovers. **A reader that only came back because someone power-cycled it is not healthy today, even though it is up today.** That attribute must be carried forward across runs, not read point-in-time.

**Three failure domains, one metric.** The path is reader → Netgear AP (`192.168.179.x`) → site router → cellular modem → ISP → cloud. A per-reader number cannot separate them; **the shape across readers can, and the tool must compute it:**

- one reader flapping, others steady → that reader or its AP
- a cluster flapping together → that AP or that part of the site
- the whole roster degrading together in the afternoon → the modem/uplink, the documented signature

Correlated instability is **one site-level finding, not thirty reader findings.** Thirty findings is how the one that matters gets buried (Actions backlog baseline: ~38 raw, ~10 distinct).

### 4. Trustworthy — it is telling the truth

- **Calibration:** `check_anchor_calibration` within tolerance; no unexplained position change since last check. Known open defect: `r40_d23c` and `r41_d2d6` are recorded at the identical point (18760, 15040, 4500) — two readers at one coordinate contribute no geometric diversity to multilateration.
- **Z position:** inside the expected band. All Crane z are positive (900 mm → 5.0 m); Excavation inverts this, where a positive z on a caisson tag is a ranging failure.
- **LPS address:** valid and unique (`check_anchor_lps_address`).
- **Firmware:** 44 anchors on `v1.7.8-0-g8cfcc1a`; **4 report no version** — `r45_b584`, `r53_da75`, `r59_d2d8` (all live and surveyed) and `r20_d112` (ghost). A drift check must know whether it is reading the anchor layer (`v1.7.8`) or the hub layer (`0.9.1.13`), or it will report drift that is not there and miss drift that is.
- **Cross-system coherency:** WiFi-Cloud and ZLP agree on identity and state (BUG-227 is open on exactly this for Crane #5).

Grade **WATCH** on any single finding here. It is rarely urgent within the hour, and it is exactly the class of fault that never becomes urgent on its own — it just degrades every fix the site produces until someone looks.

---

## Site grade from reader grades

| Site level | Condition |
|---|---|
| ⚫ **UNKNOWN** | Monitoring path unavailable, or roster unresolved. Condition not established. |
| 🔴 **P2** | All-or-near-all readers failing criterion 1 or 3 · zero location events during working hours · hub silent |
| 🟠 **P3** | **≥ 3 readers** failing criterion 1 or 2 · correlated site-wide instability under criterion 3 |
| 🟡 **WATCH** | 1–2 readers failing · any criterion 4 finding · any rising thermal trend under 1c |
| 🟢 **GREEN** | Full roster present, all five criteria met, nothing outstanding |

Denominators are graded against the **29 connected-and-surveyed** readers, never the 48-record registry.

---

## Minimum tool set

From the 32 tools in `reader_anchor_mcp_tools_reference.md`, **5 tools validate reader health.** Two are site-wide and run once; three run per reader.

### Daily run — 2 site calls + 3 per reader

| # | Tool | Scope | Criteria served | Why it cannot be dropped |
|---|---|---|---|---|
| 1 | `list_readers` | site ×1 | 0 | The roster. Use `site_res_name` + `page_size=100`. Ignore the `status` filter (see §0). |
| 2 | `list_stale_anchors` | site ×1 | 0, 3 | One call gives the whole stale set at a chosen `stale_hours`. Replaces 29 status calls for the reachability gate. |
| 3 | `get_anchor_status` | per reader | 1, 2a, 3, 4 | Only source of `connected`, firmware, NTP sync, `crash_report`, and `ranging.trackers_currently_seen`. |
| 4 | `get_anchor_metric_history` | per reader | **1a, 1b, 1c, 3** | The only time series. Everything density-based — uptime, longest gap, flap count, thermal trend — comes from here and nowhere else. |
| 5 | `get_anchor_ranging_history` | per reader | **2b, 2c, 2d** | The only per-anchor participation data. This is what revision 1 said did not exist. |

**Cost: 3N + 2 = 89 calls** for 29 readers. That is heavy for an hourly cadence — see §Sequencing.

### Weekly drift pass — adds 2 per reader

| Tool | Criteria | Notes |
|---|---|---|
| `check_anchor_calibration` | 4 | Validates rather than dumps. Would catch the r40/r41 duplicate-coordinate defect. |
| `check_anchor_lps_address` | 2a, 4 | Validates rather than dumps. |

### Worth evaluating — may collapse the site tier to one call

`generate_anchor_list_csv` is described as "CSV of all anchors at a site **with config and status**." If that CSV carries surveyed position and heartbeat age, it replaces tools 1 and 2 *and* supplies the ghost filter in a single call. **Worth ten minutes to check the actual columns** before building anything.

### Explicitly excluded, and why

| Tool | Why not |
|---|---|
| `list_anchors` | Redundant with `list_readers`. Pick one — `list_readers` takes the site UUID directly and paginates. |
| `get_anchor_calibration`, `get_anchor_lps_address` | Dump the config; the `check_*` variants validate it. Use the checkers, keep these for diagnosis. |
| `get_anchor_sensors` | Baro offsets — a Z-diagnosis tool, not a health signal. |
| **All 20 write tools** | Standing rule, no exceptions. |

**On the write tools specifically.** `locate_anchor`, `locate_and_fetch`, `locate_and_fetch_w11` and `start_reader_survey` are scoped `engineering:write` — they actively perturb the positioning system. They are tempting for "actively test whether this reader can range." **They must never appear in a monitor at this site.** Nor may `cycle_anchor`, `reset_anchor`, `upgrade_anchor_firmware`, or any bulk operation.

The tool reference states plainly: *"Production environment: zlp-prd-jpn has write+admin permissions."* That confirms as live the risk flagged in `eng-api-mcp-connector-setup.md` — a shared org connector credential carrying `engineering:write` would let anyone in the ZaiNar org reboot a reader at a live crane safety site. **Issue the monitor's key scoped `viewer` / `engineering:read`.** `confirm: false` defaulting to dry-run is a guardrail against accidents, not against a wrongly-scoped key.

---

## Sequencing — 89 calls is too many to run hourly

Two-stage, so the expensive pass is earned rather than routine:

**Stage 1 — site sweep, 2 calls, every run.** `list_readers` + `list_stale_anchors(stale_hours=0.5)`. If the stale set is empty and the roster count matches 29, the site is provisionally green and the run can stop with a one-line result.

**Stage 2 — per-reader, only for:** (a) every reader in the stale set, (b) a rotating slice of the healthy roster so the full set is covered across a day, and (c) all 29 once daily in the morning JST pass.

`get_anchor_ranging_history` returns **raw** measurements, not aggregates. Twenty-four hours × 29 readers may exceed the ~150,000-character tool-result cap noted in the MCP collateral. **Test the payload size on one reader before assuming a site-wide 24 h pull is possible** — if it is too large, use shorter windows and aggregate, or sample.

---

## Conflicts to resolve before wiring alerts

**1. Heartbeat threshold — three different numbers are in play.**

| Source | Healthy | Warning | Critical |
|---|---|---|---|
| Misha, 2026-08-31 | < 12 h | — | — |
| `reader_anchor_mcp_tools_reference.md` | "within threshold" | > 1 h | > 1 day |
| ZaiNar heartbeat SLA (MCP collateral §2) | ~60 s expected | > 10 min | > 30 min |

These differ by a factor of 720. The tool reference is *stricter* than the 12 h proposal and closer to the SLA. **Recommendation:** keep 12 h as the criterion-0 reachability gate, and grade criterion 3 on working-hours density against the SLA. Both come from the same data at no extra cost. Someone needs to sign off on which number the alert fires against.

**2. `get_anchor_status.last_heartbeat` is a formatted string** (e.g. `"10h ago"`), not a timestamp. Consequences, and this one is load-bearing:

- You cannot compute gaps, uptime or flap count from it. Criterion 3 **must** come from `get_anchor_metric_history`.
- Its granularity is coarse and unstated — at a 10-minute threshold it is unusable.
- This may be *why* the reference's thresholds are 1 h / 1 day: those are the granularities a relative string can express. The threshold may have been chosen by the display format rather than by the failure mode.

**3. "100+ ranging events in 24 h" is an absolute threshold.** It will misgrade perimeter readers in both directions. Adopted here as a RED floor only; the grading rule is baseline-relative (§2c).

**4. Backfill.** If heartbeats or metrics are buffered at the hub and flushed when the uplink returns, a multi-day outage backfills and gap analysis afterwards shows a continuous series. **Check whether `get_anchor_metric_history` exposes both an event timestamp and an ingestion timestamp.** If only one, live reporting cannot be distinguished from a backfilled flush, and density silently stops being trustworthy during exactly the incident it is meant to catch.

---

## Open questions

1. **Which heartbeat threshold fires the alert** (§Conflicts 1).
2. **Heartbeat cadence** — what interval does the device actually emit at? Unanswered; `last_heartbeat`'s string format may make it unknowable from this API.
3. **Backfill semantics** (§Conflicts 4).
4. **`generate_anchor_list_csv` columns** — does it collapse the site tier to one call?
5. **`get_anchor_ranging_history` payload size** over 24 h for one reader.
6. **Coverage map** — criterion 2c's "tags within its coverage" needs a nominal radius or static neighbour list per reader. Does one exist, or must it be derived from observed history?
7. **R3 rated operating temperature range** — needed to make 1b/1c gradeable rather than descriptive.
8. **Recovery history** — where "needed a manual power cycle" gets recorded so it survives across runs.
9. **`r59_d2d8`** — surveyed and connected per the registry, but reported no `sensors.tempC` in the 2026-08-25 24 h window and carries no firmware version. Best available first test case for whether these tools reveal anything the Grafana path does not.

---

## Validation status

**None of the above has been run against the live MCP.** The engineering-api MCP is not connected to any Claude session yet (see `eng-api-mcp-connector-setup.md`); everything measured so far came from the `postgres-zlp` Grafana datasource. Every threshold here is derived from ZaiNar's documented values, the registry export, or observed Grafana data — **not** from real MCP payloads. Step 5 of the spec's next-steps list still stands in full.
