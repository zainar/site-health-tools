---
description: Koto Crane reader health check (zlp-prd-jpn). Read-only. Grades the 29 commissioned readers against the five-criterion definition, and runs the site sweep the other checks read for context.
argument-hint: "[check | deep | weekly | triage]  (default: check)"
allowed-tools: mcp__zlp-prd-jpn__resolver_status, mcp__zlp-prd-jpn__who_am_i, mcp__zlp-prd-jpn__get_site_last_locate, mcp__zlp-prd-jpn__get_site_health, mcp__zlp-prd-jpn__get_site_last_metrics, mcp__zlp-prd-jpn__get_site_summary, mcp__zlp-prd-jpn__list_readers, mcp__zlp-prd-jpn__list_stale_anchors, mcp__zlp-prd-jpn__generate_anchor_list_csv, mcp__zlp-prd-jpn__get_anchor_status, mcp__zlp-prd-jpn__get_anchor_metric_history, mcp__zlp-prd-jpn__get_anchor_ranging_history, mcp__zlp-prd-jpn__check_anchor_calibration, mcp__zlp-prd-jpn__check_anchor_lps_address, mcp__zlp-prd-jpn__list_site_wifi_nodes, mcp__zlp-prd-jpn__run_site_coherency_audit, mcp__zlp-prd-jpn__check_wificloud_health, mcp__zlp-prd-jpn__check_pipeline_coherency, mcp__zlp-prd-jpn__check_inventory_coherency, mcp__zlp-prd-jpn__explain_zlp_issue, mcp__zlp-prd-jpn__get_site_firmware_inventory, mcp__zlp-prd-jpn__list_assets_with_tags, Read, Write, Bash
---

# Koto Crane — reader health check — `$ARGUMENTS`

**Site**: `Koto_Pumping_Station_Crane` · `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`
**Account**: `Kajima-Koto` · **Env**: `zlp-prd-jpn` (WIFI_RT) · **NID** 16

This file is **self-contained**. It needs no other document to run.

> **This is a safety system.** Red-zone and proximity alerts warn workers standing under crane loads. A silent site is urgent, not a data-quality issue.

## This check runs the site sweep. The others depend on it.

`/koto-hub-check`, `/koto-tag-check` and `/koto-tag-usage` all need to know whether the site itself is up before they can interpret their own findings — a dark site presents as a dead hub, twelve dead tags, or an idle crane, and none of those is the real finding. **This is the only check that runs §5.** Run it first when anything else looks wrong, and write its `STATE` block where the others can read it.

---

## WHAT THIS IS, AND WHAT IT IS NOT

**This is a checkpoint, not a detector.** It runs when something runs it. The primary detection mechanism for this site is the Grafana `device.reader.offline` (> 30 min) and active-count-below-threshold rules routed to Slack — corrective action Gap #1 from the May 2026 incident, owner Jake/DevOps, **last seen listed as *Investigating*.**

**Confirm those rules shipped.** A checkpoint sitting on top of working alerting is a good design. A checkpoint standing in *place* of alerting is slower than what the platform already does for free, on a site where workers stand under crane loads. Say in every report whether that question is still open.

**Nothing in this command has been validated against a real payload.** Every threshold comes from ZaiNar's documented SLAs, the 2026-08-25 registry export, or Grafana observations. The registry export carries no timestamps of any kind. **The first few runs are as much a test of these thresholds as of the site** — when a payload contradicts what is written here, the payload is right and this file needs correcting.

---

## INSTALL — handled by the plugin

This command ships in the **kajima-crane-site-survey** plugin. Installing the plugin gives you `/koto-reader-check` in every project — there is nothing to copy by hand.

**The plugin also registers the MCP server under the name `zlp-prd-jpn`, and that matters more than it looks.** Every tool in this file is prefixed `mcp__zlp-prd-jpn__`. If the server ends up registered under any other name, the `allowed-tools` list in the frontmatter **silently fails to match** and the command looks broken for no visible reason. Confirm before the first run:

```bash
claude mcp list
```

If you already have that server registered under a different name, rename it rather than editing prefixes across five files.

**Set `ZLP_PRD_JPN_API_KEY` to a key scoped `viewer` / `engineering:read`.** The credential this MCP normally carries has **write + admin on production**. `confirm: false` dry-run defaults protect against a mistaken call, not against a wrongly-scoped key. The preflight checks this and refuses to continue on a write-scoped key.

---

## 0. PREFLIGHT — fail loud, never fall back

Run in order. **Do not proceed past a failure and do not substitute another data source.**

1. Call `resolver_status`.
2. If the tool does not exist, is not permitted, or errors → print exactly this and **STOP**:

```
⚫ KOTO CRANE READERS — MONITORING UNAVAILABLE
Site condition is UNKNOWN. Not healthy, not unhealthy.
Cause: <the error>
Check: claude mcp list  → confirm the server name matches the mcp__..__ prefix in this command.
```

Increment `unknown_runs` in state before stopping. **If `unknown_runs` was already ≥ 1, say prominently that monitoring has now failed on two consecutive runs** — that is its own finding and it means nobody has had eyes on this site since the last successful run.

3. Call `who_am_i`. **If it returns write or admin scope, print a warning at the top of the report and continue read-only.** Do not stop — a wrongly-scoped key is a governance finding, not a monitoring failure — but it must not go unremarked.

4. Get the current time in **both** zones and print them. JST = UTC+9; the site runs on JST and most tooling around it reports PT.

```bash
date -u '+UTC %Y-%m-%d %H:%M'; TZ=Asia/Tokyo date '+JST %Y-%m-%d %H:%M (%a)'; TZ=America/Los_Angeles date '+PT  %Y-%m-%d %H:%M'
```

5. Read `.koto-reader-state.json` from the working directory. If absent, seed it from **§Seed state** at the end of this file. Write it back at the end of the run, **never before** — a run that dies mid-way must not leave a state file claiming it succeeded.

> **⚫ UNKNOWN is a grade, not a failure to grade.** Everything this command sees is cloud-side. During the May 2026 outage **all 30 readers were alive and answering local pings while every cloud view was dark.** A site UNKNOWN is a different fact from a site UNHEALTHY, and both differ from HEALTHY. **Conflating the first two is what let a 13-day outage stay hidden.** Never collapse them, and never upgrade an unrun check to a pass — an unrun check is `Not checked`.

---

## 1. ABSOLUTE RULE — read-only, no exceptions

Never call a write tool, regardless of what you find or how obvious the fix looks.

**Anchor / reader writes:** `locate_anchor` · `locate_and_fetch` · `locate_and_fetch_w11` · `start_reader_survey` · `cycle_anchor` · `reset_anchor` · `upgrade_anchor_firmware` · `reboot_node` · `trigger_firmware_upgrade` · any `set_*` / `apply_*` / `configure_*` / bulk operation.

**Two traps, named because each one is what you would naturally reach for:**

- **`locate_anchor` / `locate_and_fetch` / `start_reader_survey` are scoped `engineering:write` and actively perturb the positioning system.** They are exactly what you would call to answer "can this reader still range?" Answer that from `get_anchor_ranging_history` instead. **They must never appear in a monitor at this site.**
- **`cycle_anchor` looks like the fix** for a reader that will not reconnect after an ISP recovery. There is a documented firmware bug that produces exactly that state. Propose the power-cycle; do not perform it. A crane may be under load.

Propose remediation in the report. **A human who knows what is happening on site that hour executes it.**

---

## 2. SCOPE — 29 readers

### The ghost-record rule — apply before any counting

The registry returns **48 anchor records. Only 29 are commissioned readers**: `r31`, `r32`, and the contiguous run `r40`–`r66`. The other 19 (`r20`–`r30`, `r33`–`r39`, `r67`) are ghost records.

> **An anchor with no surveyed x/y is inventory, not a monitored reader.**

**The filter is positional, not temporal.** Verified across all 48 records with no exceptions: every `connected` anchor has full x/y/z; every disconnected one has a z-only placeholder (`z = 600`) or nothing at all. A reader that is surveyed in and later goes dark **keeps its coordinates**. These 19 never had any.

An earlier proposal to exclude readers silent > 100 days was rejected because the exclusion criterion would be the same signal being graded — a permanently dead reader would age out of the denominator and the site would return to green. **The positional test has no such feedback loop. Use it.**

**Do not use `list_readers(status=...)` as the ghost filter.** Every reader in the ZLP production database — all 129, every site — is `ACTIVE`. `INACTIVE`, `DELETED` and `ACCEPT_PENDING` have never been used. The field is unmaintained.

**Denominator is 29.** Spec expectation is 30, so **one reader is unaccounted for** — say so every run; do not quietly round. Report the ghost count as a **one-line hygiene note, outside the health score**. Counting them yields "46 of 48 stale, health 4%" on a site that is probably fine, and a report that cries wolf gets ignored inside a week — the incident's own lesson #1 in a different costume.

**Do not quote the platform's own health percentage.** Recompute against 29.

**Hubs are out of scope here** — `/koto-hub-check` grades them. §9 says when a reader finding is actually a hub finding.

---

## 3. GRADES

| Grade | Meaning |
|---|---|
| 🟢 **HEALTHY / GREEN** | Meets all criteria over the evaluation window |
| 🟡 **WATCH** / 🟠 **AMBER / P3** / 🔴 **RED / P2** | Fails one or more — per-criterion thresholds below |
| ⚫ **UNKNOWN** | Could not observe it. API down, site uplink down, roster unresolved. **Condition not established.** |

**Evaluation window: rolling 24 h, weighted to JST working hours (08:00–18:00 Mon–Fri).** Overnight quiet is not evidence of anything at this site — the documented failure mode is daytime-only, peaking ~16:00 JST.

> **Settled 2026-09-16: 08:00–18:00 JST, Mon–Fri, everywhere in this package.** One window, replacing the three previously in play. **Weekends are outside it and are not graded** — but weekend activity is still reported as an out-of-hours line, never filtered away. Saturday working is common on Japanese construction sites, and a genuine Saturday shift must not become indistinguishable from an idle site.
>
> **Saturday is excluded for now rather than on principle**, and the package is built so it can be added back in one edit. See §Open questions for what to confirm first — the blocker is on the tag side, not here. Readers are powered continuously and have no reporting mode, so Saturday would need no special handling in this file.

---

## 4. MODES AND THE CALL BUDGET

A full per-reader pass is **3N + 2 = 89 calls** for 29 readers. That is too many to run casually, so the expensive pass is earned rather than routine.

| Mode | What runs | Rough calls |
|---|---|---|
| **`check`** (default) | §5 site sweep + per-reader only for readers in the stale set | ~10–20 |
| **`deep`** | §5 + full §6 per-reader pass on all 29 | ~95 |
| **`weekly`** | `deep` + §6.4 drift pass (+2 per reader) + §8 weekly drift | ~155 |
| **`triage`** | §5 + §9 narrowing tools, for a site already known to be flagged | ~10 |

**The caller picks the mode; how often each depth runs is the scheduler's decision, not this file's.** One property of that choice is worth stating so it is made deliberately: **a reader that is never in the stale set is never actually examined by `check`.** Whatever invokes this therefore needs `deep` often enough to cover the full roster, and `check` runs should rotate a slice of the healthy roster through the per-reader pass.

> **`get_anchor_ranging_history` returns raw measurements, not aggregates.** 24 h × 29 readers may exceed the ~150,000-character tool-result cap. **Test the payload size on one reader before attempting a site-wide 24 h pull.** If it is too large, use shorter windows and aggregate — **do not silently truncate and do not sample without saying so.**

---

## 5. STAGE 1 — site sweep. Runs every mode, and the other checks read it.

Run in this order. Stop where told.

1. `get_site_last_locate` — **the primary signal.** Age of the most recent position fix.
2. `get_site_health` — **recompute against 29.** Do not quote the platform's number.
3. `list_stale_anchors` with **`stale_hours=0.5`** — *not* the 24 h default. A crane site does not have 24 hours to spare. **Drop any result with no surveyed position.**
4. `get_site_last_metrics`
5. `get_site_summary` — compare the live reader count against 29.
6. `run_site_coherency_audit` — run it **every time**, not only on a flag. The root-cause code is the most useful single line in the report. **Quote it verbatim**: `wificloud_offline` · `zlp_pipeline_lag` · `inventory_mismatch` · `z_outlier` · `active_alerts` · `healthy`.

**`connected` is not a heartbeat.** It is a registry flag. The site can show 29 connected and produce no location fixes at all — that is exactly what May 2026 looked like. **Step 1 outranks step 5.**

**Expect `inventory_mismatch` every run** from the 19 ghost records. Compare against `coherency_code` and `coherency_is_baseline_only` in state. A mismatch that is only the known 19 is **not a finding** — set `coherency_is_baseline_only: true` and write "standing baseline, unchanged". A mismatch that is *more* than the 19 **is** a finding.

**If the stale set is empty and the roster count matches 29**, the site is provisionally green: on `check` mode you may skip §6 per-reader entirely and say so in `Not checked`.

> **Worth ten minutes once, then never again:** `generate_anchor_list_csv` is documented as "CSV of all anchors at a site **with config and status**." If that CSV carries surveyed position and heartbeat age, it replaces steps 3 and 5 *and* supplies the ghost filter in a single call. **Check the actual columns on your first run** and record the answer in `csv_columns_checked` in state.

---

## 6. READERS — the five criteria

Skip on `check` when §5 came back clean, except for readers in the stale set. Run all 29 on `deep` and `weekly`.

### 6.0 Present — on the roster, and reachable at all

`list_readers` with `site_res_name` and `page_size=100`. Apply the ghost rule from §2.

**Reachability gate: last heartbeat within 12 h.** Beyond that the reader **drops off the monitored roster and onto the inventory review list** — it is not graded unhealthy, it is graded *not currently a monitored reader*, and that count is reported separately.

Ghost count and gated-out count are **hygiene numbers, reported every run, never folded into the health score.**

### 6.1 Alive — powered, and the sensor works

Source: `get_anchor_metric_history`. This is the **only** time series available; everything density-based comes from here and nowhere else.

**1a. Continuity.** `sensors.tempC` samples present with no gap exceeding the SLA: **~60 s expected · warn > 10 min · error > 30 min.**

Measured 2026-08-25: all 28 live readers reported ~1/min; longest single gap across the whole roster in 24 h was under 10 min. **The hardware comfortably meets this**, so a breach is a real signal, not a tuning artifact.

**Continuity proves power AND the metrics path, jointly. It cannot separate them.** A gap means "not reaching the cloud" — **never report it as "the reader is dead."**

**1b. Validity — continuity alone is not enough.** Production has already produced every counter-example:

| Test | Threshold | Why |
|---|---|---|
| Not stuck | σ > 0.1 °C over the window | A gapless series with zero variance is a cached value or a dead sensor |
| Not zeroed | not all-zero | Temperature-sensor I2C failure is a documented critical fault; all-zeros is its signature |
| In range | within the R3's rated operating range | `r47_b51b` reported avg 561,445 hPa / max 5,369,470 on the pressure channel — continuous, and nonsense |

> **The rated operating range is an open question.** Until someone supplies it, report the observed temperature range descriptively and say the in-range test could not be applied.

**1c. Thermal trend — the leading indicator, and it is currently discarded.** The documented root cause at this site is thermal: cellular modem, peaking 16:00 JST, construction-equipment noise. Today only the gaps get used.

**Report per reader: absolute temperature, daily peak and its hour, day-over-day drift in that peak, and cross-reader spread.** Readers heating faster than the rest localises the problem physically.

> **Grade WATCH on a rising afternoon peak before it fails, not after.** This is the one thing that could move this site from reactive to predictive, and it costs nothing extra — the data is already in the call you made for 1a.

### 6.2 Contributing — participating in locationing

**2a. Synced.** `check_anchor_lps_address` (valid, unique) and `get_anchor_status.system` (NTP synced). **An unsynced reader is online and contributes nothing** — it looks alive and is functionally absent.

**A heartbeat can never satisfy this criterion.** A reader can heartbeat every minute while contributing zero ranging measurements — unsynced, or wrong LPS address.

**2b. Contributing measurements.** `get_anchor_ranging_history` over the window.

**2c. Normalised to the reader's own baseline, never to the site.** A corner reader legitimately participates far less than a central one. Absolute thresholds either cry wolf on the perimeter or miss a failure in the middle.

> **Healthy = ranging rate ≥ 50 % of that reader's own trailing 28-day median for the same hour-of-day**, evaluated only in periods with ≥ 1 live tag in coverage.
> **< 50 % = WATCH · < 20 % = AMBER · zero across a working day with live tags present = RED for that reader.**
> **Absolute floor regardless of baseline: < 100 ranging events in 24 h = RED.**

`get_anchor_metric_history(range="last_30d")` is available, so the 28-day baseline is buildable. **Until you have 28 days of it, say the baseline is unavailable and grade on the absolute floor alone** — do not silently substitute a site-wide average, which is the exact error 2c exists to prevent.

**Gate on the denominator.** Zero contribution at 03:00 with nobody on site is not a fault: **no live tags in coverage → the criterion is *not evaluated*, not *failed*.** `/koto-tag-check`'s last output tells you whether there were live tags at all.

**2d. Yield, not just volume.** Accepted vs attempted. **A reader throwing ranging errors at high volume looks busy and is failing.** Use the `aborts` and `latency.*` metrics per reader; `0x0100 ranging_error` and `0x0400 hub_not_found` in the tag-side flags bitmask change the diagnosis.

### 6.3 Stable — connection quality over time, not a binary

Four numbers per reader per window, all from `get_anchor_metric_history`:

| Metric | Definition | Healthy |
|---|---|---|
| **Uptime fraction** | samples received ÷ expected, over JST working hours | ≥ 99 % |
| **Longest single gap** | max continuous silence in the window | < 10 min |
| **Flap count** | distinct disconnect→reconnect events | ≤ 2 / day |
| **Recovery** | returned without human intervention, and how fast | self-recovered, < 5 min |

**Grade on density, not on age.** A heartbeat proves the path worked *at that timestamp*. Age-of-last only catches a reader dark *right now* — and a morning run means **a reader dark from 08:30 to 17:30 the previous working day has a fresh heartbeat and reads healthy. The failure heals before the check looks at it.** This is not hypothetical; daytime-only failure with overnight recovery is this site's documented signature.

**Report the longest gap alongside the fraction, always.** 99 % with one 20-minute hole during a lift is worse than 99 % spread over a hundred 5-second blips.

**Recovery carries forward across runs.** There is a documented firmware bug where readers do not reconnect after the ISP recovers. **A reader that only came back because someone power-cycled it is not healthy today, even though it is up today.** Carry that attribute in state; it cannot be read point-in-time.

> **`get_anchor_status.last_heartbeat` is a formatted string** (e.g. `"10h ago"`), not a timestamp. **You cannot compute gaps, uptime or flap count from it.** Criterion 3 must come from `get_anchor_metric_history`. If you find yourself parsing prose to grade stability, stop — that is the wrong source.

**Three failure domains, one metric.** The path is reader → Netgear AP (`192.168.179.x`) → site router → cellular modem → ISP → cloud. A per-reader number cannot separate them; **the shape across readers can, and you must compute it:**

- one reader flapping, others steady → **that reader or its AP**
- a cluster flapping together → **that AP or that part of the site**
- the whole roster degrading together in the afternoon → **the modem/uplink, the documented signature**

**Correlated instability is ONE site-level finding, not thirty reader findings.** Thirty findings is how the one that matters gets buried.

### 6.4 Trustworthy — it is telling the truth

Runs on `weekly`; runs on `deep` for any reader already flagged.

- **Calibration** — `check_anchor_calibration` within tolerance; no unexplained position change since last check.
- **Z position** — inside the expected band. All Crane z are positive (900 mm → 5.0 m). *(Excavation inverts this — a positive z on a caisson tag there is a ranging failure. Different site; noted so the rule does not get copied wrong.)*
- **LPS address** — valid and unique (`check_anchor_lps_address`).
- **Firmware** — `get_site_firmware_inventory`. **State which layer you are reading**: anchors run `v1.7.8-0-g8cfcc1a`, hubs run `0.9.1.13`, and `hub` is *also* an anchor firmware sub-component. **Reading the wrong layer reports drift that is not there and misses drift that is.**
- **Cross-system coherency** — WiFi-Cloud and ZLP agree on identity and state. BUG-227 is open on exactly this for Crane #5.

**Grade WATCH on any single finding here.** It is rarely urgent within the hour, and it is exactly the class of fault that never becomes urgent on its own — **it just degrades every fix the site produces until someone looks.**

### 6.5 A reader can pass 1 and 3 and still be poisoning the solution

A reader can be powered, reporting temperature every 60 s, and contributing ranging measurements — while being **wrong**. Mis-calibrated position, drifted Z, wrong LPS address, firmware out of step. **Crane #5's anchor tag has already physically shifted during operation once.**

That reader passes continuity, participation and stability, and quietly poisons the location solution that keeps workers out from under crane loads. **It is worse than an offline reader, because an offline reader is visibly absent and this one is not.** Never let a green on criteria 1–3 stand in for criterion 4.

---

## 7. WEEKLY DRIFT

On `weekly`, after §6:

- `list_site_wifi_nodes` — hub / anchor / tracker counts. Compare to state.
- `get_site_firmware_inventory` — **state which layer.**
- `check_anchor_calibration` across all 29 live anchors.
- `check_anchor_lps_address` across all 29.
- `list_assets_with_tags` — mapping still correct after crew changes.
- `check_inventory_coherency` — against the ghost baseline, not raw.

**A reader trending down over three weeks is the pattern; one bad day is not.** Compare against `history` in state.

---

## 8. CORRELATE BEFORE YOU LIST

This is the step that decides whether the report is usable. The Actions backlog baseline is ~38 raw findings collapsing to ~10 distinct.

**Before writing a single per-reader finding, ask in this order:**

1. **Is this the whole site?** Zero location events + all-or-near-all readers silent → **one P2 finding**, not 29.
2. **Is this a cluster?** Readers flapping together, or on the same AP subnet → **one finding naming the AP or the site segment.**
3. **Is this the afternoon signature?** Failures concentrated in daytime JST peaking ~16:00, readers dropping in cohorts (10, then 3, then 11), nothing overnight → **name the cellular modem as prime suspect**, one finding. **Compare the live count against `live_readers_reporting` in state — a 29 → 24 → 18 trend across runs is that pattern, and the delta is the only way you will see it.**
4. **Is this a hub finding wearing a reader costume?** Multiple readers showing `0x0400 hub_not_found`, or ranging failures clustered in one hub's coverage → **that is a hub finding.** This command does not grade hubs. **Say so explicitly and point at `/koto-hub-check`** rather than listing N reader findings.
5. **Is this a tag-fleet problem?** Zero ranging with no live tags in coverage is **not evaluated**, per §6.2c. Cross-check `/koto-tag-check`'s last output before calling a reader silent.

**Only what survives all five gets listed individually.**

> **Readers and hubs are graded by separate commands, and the correlation between them is real.** When a finding here points at gates 4 or 5, name the other monitor in the report so whoever reads it knows which one to run — and say plainly that this run did not check it.

**And state the limitation plainly, every time the site looks dark:** every tool here reads **cloud-side** state. During the May 2026 outage all 30 readers were alive and answering local pings while every cloud view showed them dark. **A dark site means "not reaching the cloud", not "readers are dead".** Distinguishing them needs a local ping via the Netgear AP proxy on `192.168.179.x`, which this command cannot do. **Escalate to someone who can reach the site network.**

---

## 9. SITE GRADE

Denominators are graded against the **29 connected-and-surveyed** readers, never the 48-record registry.

| Level | Condition |
|---|---|
| ⚫ **UNKNOWN** | Monitoring path unavailable, or roster unresolved. **Condition not established.** |
| 🔴 **P2 / RED** | Zero location events during working hours · all-or-near-all live readers failing criterion 1 or 3 |
| 🟠 **P3 / AMBER** | **≥ 3 readers** failing criterion 1 or 2 · correlated site-wide instability under criterion 3 · location pipeline lag > 120 s |
| 🟡 **WATCH** | 1–2 readers failing · any criterion 4 finding · any rising thermal trend under 1c · metrics lag > 60 s · firmware spread · any coherency finding beyond the ghost baseline |
| 🟢 **GREEN** | Full roster present, all five criteria met, nothing outstanding |

*(The register also lists **hub silent** as a P2 condition. That is `/koto-hub-check`'s call, not this one's — and it is conditional on the unresolved hub roster question. Do not grade it from here.)*

Reference thresholds: anchor heartbeat ~60 s (warn > 10 min, error > 30 min) · metrics lag warn > 60 s / error > 300 s · location lag warn > 30 s / error > 120 s.

### Alerting and cadence — not this command's job

**This command grades and reports. It does not decide when to run, who to tell, or what escalates.** Cadence, delivery and any confirm-before-paging ladder belong to whatever invokes it — the Agent Scheduler task, not this file. Keeping them there means one place to change them, and no chance of this file and the scheduler disagreeing about what fires.

What this command owes whatever invokes it is an honest grade, an explicit `Not checked` list, and a `Change` line against the previous run. **Never put keys, tokens or dashboard credentials in any output.**

---

## 10. STANDING ITEMS — carry every run until someone closes them

- **One reader unaccounted for** — 29 live vs 30 expected.
- **`r40_d23c` and `r41_d2d6` sit at the identical coordinate** (18760, 15040, 4500). Two readers at one point add no geometric diversity to multilateration on that edge. Survey copy-paste, or genuinely co-located — unchecked.
- **Three live readers report no firmware version** — `r45_b584`, `r53_da75`, `r59_d2d8`.
- **`r59_d2d8`** — surveyed and connected per the registry, but reported no `sensors.tempC` in the 2026-08-25 24 h window and carries no firmware version. **Best available first test case for whether these tools reveal anything the Grafana path does not.**
- **Crane #5's tag has physically shifted during operation before** — calibration drift there is not hypothetical. BUG-227 open on WiFi-Cloud/ZLP coherency for it.
- **Grafana alert rules from incident Gap #1** — shipped, or still *Investigating*? This command's value depends on the answer.

## Open questions — carry until answered

1. **Which heartbeat threshold fires the alert.** Three numbers are in play and they differ by a factor of 720: Misha's 12 h reachability gate · the tool reference's 1 h warn / 1 day critical · the ZaiNar SLA's 10 min / 30 min. This command uses 12 h as the criterion-0 gate and the SLA for criterion 3 density. **Someone needs to sign off.**
2. **Backfill semantics.** If heartbeats are buffered at the hub and flushed when the uplink returns, a multi-day outage backfills and gap analysis afterwards shows a continuous series. **Does `get_anchor_metric_history` expose both an event timestamp and an ingestion timestamp?** If only one, density silently stops being trustworthy **during exactly the incident it exists to catch.**
3. **Coverage map** — §6.2c's "tags within its coverage" needs a nominal radius or static neighbour list per reader. Does one exist?
4. **R3 rated operating temperature range** — needed to make 1b/1c gradeable rather than descriptive.
5. **`generate_anchor_list_csv` columns** — does it collapse the site tier to one call?
7. **Where "needed a manual power cycle" gets recorded** so the recovery attribute survives across runs.
8. **Saturday — excluded for now, and re-adding it is blocked on one question.** The tag mode scheduler is documented as *"skipped on JST weekends"*, so on Saturdays the fleet may sit in `on_motion`, whose tolerances are 4× looser than `continuous`. That would misgrade the tag checks, not this one — readers have no mode. **Confirm with whoever owns `koto-crane-daily.md` §11 whether "weekends" means Sat+Sun or Sun only**, then widen the window across the package together.

---

## 11. REPORT

Print to the conversation. On `weekly`, also `Write` it to `koto-readers-YYYY-MM-DD.md` in the working directory.

```
KOTO CRANE READERS — <emoji + level> — <YYYY-MM-DD HH:MM JST / HH:MM PT> — mode: <check|deep|weekly|triage>

One line: what is true right now.

Readers     <n>/29 live reporting        (silent: <names>)
Gated out   <n> past the 12h reachability gate — inventory review, not graded
Ghosts      19 uncommissioned records (excluded — hygiene only)
Last fix    <age>
Pipeline    metrics <lag> / location <lag>
Coherency   <root-cause code verbatim>  <(standing baseline, unchanged) | (NEW — beyond baseline)>
Change      <vs last run: +/- readers, who recovered, who newly went silent | unavailable — no prior state>

=== FIVE CRITERIA ===
0 Present       <n>/29 on roster and inside the 12h gate
1 Alive         continuity <n> ok / <n> gap>10m / <n> gap>30m   validity <n> stuck/zeroed/out-of-range
1c Thermal      peak <°C> at <hh:mm JST>   day-over-day <+/- °C>   spread <°C>   trend: <flat|rising>
2 Contributing  <n> synced   <n> below 50% baseline   <n> below the 100-event floor   baseline: <available|unavailable>
3 Stable        uptime <pct>%   longest gap <n>m   flaps <n>   shape: <isolated|cluster|site-wide>
4 Trustworthy   calibration <n findings>   z <ok|outliers>   lps <ok|dupes>   firmware <spread, and which layer>

Findings
- <finding, with the number that triggered it — after §8 correlation>

Points at another monitor
- <e.g. "4 readers raising 0x0400 — hub finding, run /koto-hub-check. NOT checked by this run.">

Standing items
- <from §10, unchanged until closed>

Suspected cause
- <only if evidence supports it — otherwise "insufficient evidence from cloud-side data">

Recommended actions (for a human)
- <action> — <who>

Not checked
- Hubs — graded by /koto-hub-check.
- Tags — graded by /koto-tag-check.
- Local reachability — cloud-side only.
- <whatever else this mode did not cover, and why>
```

### Say these every run

1. **Which mode ran, and what it therefore did not check.** An unrun check is `Not checked`, never a pass.
2. **That everything here is cloud-side.** A dark reader means "not reaching the cloud", never "the reader is dead".
3. **That hubs and tags are not covered by this command.**
4. **⚫ UNKNOWN ≠ UNHEALTHY ≠ HEALTHY.** Never collapse them.
5. **Whether the Grafana detection rules are confirmed live.** This command is a checkpoint, not a detector.

**Be blunt about uncertainty.** A confident wrong diagnosis on this site costs more than an honest "cloud says dark, cannot tell from here whether the readers are alive." The May outage meandered for 13 days across nine people partly because findings arrived without their caveats.

---

## Seed state — write to `.koto-reader-state.json` if absent

Values from the 2026-08-25 registry export and the 2026-09-08 tag check context. **Nothing here has been confirmed against a live MCP payload** — correct it on the first successful run.

```json
{
  "last_run": null,
  "source": "seed — registry export 2026-08-25, unvalidated",
  "unknown_runs": 0,
  "grade": null,
  "live_readers_reporting": null,
  "silent_readers": [],
  "gated_out_readers": [],
  "coherency_code": "inventory_mismatch",
  "coherency_is_baseline_only": true,
  "csv_columns_checked": false,
  "baseline_28d_available": false,
  "manual_power_cycles": {},
  "history": [],
  "standing_items_open": [
    "reader_29_vs_30",
    "r40_r41_same_coord",
    "three_readers_no_fw",
    "r59_d2d8_no_temp_no_fw",
    "crane5_tag_shift_bug227",
    "grafana_gap1_rules_unconfirmed"
  ],
  "context_2026_09_08": {
    "infrastructure": "29 readers, no reader-offline alerts; red-zone events fired 12:46/14:20/15:18 JST on 08 Sep",
    "tag_fleet": "9 of 12 monitored tags NEVER_SEEN — the §6.2c coverage gate will bite",
    "blocking_question": "Is Subcon C still on site? Affects whether zero ranging means anything."
  }
}
```
