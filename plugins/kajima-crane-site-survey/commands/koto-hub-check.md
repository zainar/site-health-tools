---
description: Koto Crane hub health check (zlp-prd-jpn). Read-only. Grades the site's hubs against the five-criterion definition — heartbeat, NTP, battery relay, RSSI relay, and tags reporting hub_not_found.
argument-hint: "[check | triage | weekly]  (default: check)"
allowed-tools: mcp__zlp-prd-jpn__resolver_status, mcp__zlp-prd-jpn__who_am_i, mcp__zlp-prd-jpn__list_hubs, mcp__zlp-prd-jpn__get_hub_status, mcp__zlp-prd-jpn__list_hub_tags, mcp__zlp-prd-jpn__get_hub_tag_health, mcp__zlp-prd-jpn__get_hub_logs, mcp__zlp-prd-jpn__check_tag_hub_connectivity, mcp__zlp-prd-jpn__get_node_config, mcp__zlp-prd-jpn__list_site_wifi_nodes, mcp__zlp-prd-jpn__get_site_firmware_inventory, mcp__zlp-prd-jpn__get_site_last_locate, mcp__zlp-prd-jpn__run_site_coherency_audit, Read, Write, Bash
---

# Koto Crane — hub health check — `$ARGUMENTS`

**Site**: `Koto_Pumping_Station_Crane` · `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`
**Account**: `Kajima-Koto` · **Env**: `zlp-prd-jpn` (WIFI_RT) · **NID** 16

This file is **self-contained**. It needs no other document to run.

> **This is a safety system.** Red-zone and proximity alerts warn workers standing under crane loads. A silent site is urgent, not a data-quality issue.

## Read `/koto-reader-check`'s last run before interpreting anything here

This command does **not** run the site sweep — `/koto-reader-check` owns it. That matters because **a dark site presents as a dead hub.** If the readers are also silent, the finding is one site failure, not a hub fault, and this command has no way to tell from its own tools. Read the reader check's `STATE` block first; if there isn't one, say so and grade accordingly.

---

## WHAT THIS IS, AND WHAT IT IS NOT

**This is a checkpoint, not a detector**, and it is the weakest of the checks in this plugin for a reason that no amount of care fixes: **criterion 5 has no tool.** See §7.5.

**Nothing in this command has been validated against a real payload.** Every threshold comes from ZaiNar's documented SLAs, the 2026-08-25 registry export, or Grafana observations. **When a payload contradicts what is written here, the payload is right and this file needs correcting.**

---

## INSTALL — handled by the plugin

This command ships in the **kajima-crane-site-survey** plugin. Installing the plugin gives you `/koto-hub-check` in every project — there is nothing to copy by hand.

**The plugin also registers the MCP server under the name `zlp-prd-jpn`, and that matters more than it looks.** Every tool in this file is prefixed `mcp__zlp-prd-jpn__`. If the server ends up registered under any other name, the `allowed-tools` list in the frontmatter **silently fails to match** and the command looks broken for no visible reason. Confirm before the first run:

```bash
claude mcp list
```

**Set `ZLP_PRD_JPN_API_KEY` to a key scoped `viewer` / `engineering:read`.** The credential this MCP normally carries has **write + admin on production**. `confirm: false` dry-run defaults protect against a mistaken call, not against a wrongly-scoped key.

---

## 0. PREFLIGHT — fail loud, never fall back

1. Call `resolver_status`.
2. On any failure → print exactly this and **STOP**:

```
⚫ KOTO CRANE HUBS — MONITORING UNAVAILABLE
Hub condition is UNKNOWN. Not healthy, not unhealthy.
Cause: <the error>
Check: claude mcp list  → confirm the server name matches the mcp__..__ prefix in this command.
```

Increment `unknown_runs` in state before stopping. **If `unknown_runs` was already ≥ 1, say prominently that monitoring has failed on two consecutive runs.**

3. Call `who_am_i`. **If it returns write or admin scope, warn at the top of the report and continue read-only.**

4. Get the current time in **both** zones and print them. JST = UTC+9.

```bash
date -u '+UTC %Y-%m-%d %H:%M'; TZ=Asia/Tokyo date '+JST %Y-%m-%d %H:%M (%a)'; TZ=America/Los_Angeles date '+PT  %Y-%m-%d %H:%M'
```

5. Read `.koto-hub-state.json`. If absent, seed it from **§Seed state**. Write it back at the end, **never before**.

6. **Read `.koto-reader-state.json` if it exists** — you need `live_readers_reporting` and `grade` for §8. If it is missing or stale, say so and treat the site-context gates as `unavailable`, never as "site is fine".

> **⚫ UNKNOWN is a grade, not a failure to grade.** Everything this command sees is cloud-side. During the May 2026 outage **all 30 readers were alive and answering local pings while every cloud view was dark.** A hub UNKNOWN is a different fact from a hub UNHEALTHY, and both differ from HEALTHY. **Conflating the first two is what let a 13-day outage stay hidden.**

---

## 1. ABSOLUTE RULE — read-only, no exceptions

Never call a write tool, regardless of what you find.

**Hub writes:** `configure_hub` · `set_hub_reassoc_timeout` · `set_hub_provisioning_mode` · `activate_hub_provisioning` · `create_hub` · `upgrade_hub_firmware` · `reboot_tag_host` · `reboot_node` · `set_node_name` · `update_node_config`.

> **`reboot_tag_host` reads like a tag tool and is not.** It reboots the tag-management process *on the hub* — it takes out tag handling for every tag on that hub at once. The Confluence spec files it under *Tag Actions*, which is how a name-similarity mistake happens.

**Dry-run is the default and it is the only guardrail.** Every write defaults to `confirm: false`, which protects against a mistaken call and not against a mistaken `confirm: true`. Propose remediation in the report. **A human who knows what is happening on site that hour executes it.**

---

## 2. SCOPE — the hub roster is unresolved, and it blocks this command's severity ceiling

The registry counts **5 hubs** at Crane and **enumerates none of them**. The only hub this project can name, `hub_b5c7`, has reported nothing for 30+ days and has no `sensors.tempC` — while the site graded 🟢 on 28 live readers over the same window and was producing locations.

Exactly one of these is true and nobody knows which:

1. `hub_b5c7` is a stale roster entry like the `r20`–`r37` block, and the live hub is one of the other four.
2. `hub_b5c7` is real and dead, and the Crane data path does not depend on it — in which case "the hub is a single point of failure" is wrong and hub-silent is **not** a P2 at this site.

`list_hubs` answers it in one call. **Every reader of this report should know the question is open until it does.** Check specifically whether **`hub_b591`** is present — every example in the hub tool reference that pairs a hub with the Crane site ID uses that name, which would explain how the site produces locations with `hub_b5c7` dark.

**Report the hub count you actually see. Never silently pick a denominator.**

**Readers are out of scope here** — `/koto-reader-check` grades them.

---

## 3. GRADES

| Grade | Meaning |
|---|---|
| 🟢 **HEALTHY** | All five criteria met over the evaluation window |
| 🟡 **WATCH** / 🟠 **AMBER** / 🔴 **RED** | Fails one or more — per-criterion thresholds below |
| ⚫ **UNKNOWN** | Could not observe it. Monitoring path down, hub roster unresolved, site uplink down. **Condition not established.** |

**Two evaluation windows, and they do different jobs. Never let the long one grade the site.**

| Window | Question | Use |
|---|---|---|
| **Newest-sample age** — 10 min warn / 30 min error | is the hub up *now* | **grading** |
| **Rolling 24 h**, JST working hours (08:00–18:00 Mon–Fri) | uptime, longest gap, flap count, trend | the report body |

*(A hub can be dead 23 hours and pass a 24-hour "any data" test. That is why the short window grades.)*

**Working hours are 08:00–18:00 JST, Mon–Fri** — settled 2026-09-16, the single window across every check in this package. Weekends are outside it and are not graded, but weekend activity is still reported as an out-of-hours line rather than filtered away. **Saturday is excluded for now, not on principle** — the blocker is on the tag side (see `/koto-tag-check` §5); hubs are powered continuously and have no reporting mode.

Both `get_hub_status` and `get_hub_tag_health` resolve a hub by name **only if `site` is also passed.** Always pass `site` = `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`.

---

## 4. MODES

| Mode | What runs |
|---|---|
| **`check`** (default) | §7.0–7.4 |
| **`triage`** | `check` + §7.6 narrowing tools, for a hub already flagged |
| **`weekly`** | `check` + §8 drift |

**The caller picks the mode; how often each runs is the scheduler's decision, not this file's.**

---

## 7. HUBS — the five criteria

*(Numbering kept aligned with `hub-health-definition.md` so the two read together.)*

### 7.0 Present

`list_hubs`. Report the count seen against the registry's claim of 5. Note whether `hub_b591` is present. **Compare the list against `hubs_seen` in state — any addition or disappearance is a finding.** Carry the §2 blocker until it is settled.

### 7.1 Alive

`get_hub_status` per hub, with `site`.

| Reading | Grade |
|---|---|
| Newest host-metrics sample > 30 min | **ERROR** |
| > 10 min | **WATCH** |
| NTP unsynced | **WATCH** |
| New `crash_report` | **WATCH** |
| `connected: true` while past a threshold | **contradiction finding — report both readings, resolve neither** |
| `disk_free` trending to zero, or `system_free_memory` falling day over day | **WATCH** |

**`tempC` is not required and its absence is not a fault.** `cpu_idle`, `system_free_memory`, `process_memory_usage.*` and `disk_free.*` are present for 63 devices at ~1 sample/min and are a de-facto 60 s heartbeat. That closes the missing-`tempC` gap with no new instrumentation — **do not report a hub as unmeasurable because it has no temperature sensor.**

**The contradiction case is more informative than either fact alone.** `R55_d299` showed `Active` while four days offline. A registry flag is a claim; a heartbeat is evidence. **Report the disagreement; do not silently pick a side.**

**NTP is a criterion, not a detail.** A hub whose clock has drifted timestamps everything downstream of it wrongly while looking perfectly alive.

> **`last_heartbeat` is a humanised string** (`"51m ago"`, `"16d ago"`), not a timestamp. **Parse it explicitly and state the parsed minutes in the report.** Behaviour at unit boundaries is undocumented — does 90 s render as `"1m ago"` or `"90s ago"`? **If you cannot parse it, that hub is ⚫ UNKNOWN — never healthy.** A parse miss that defaults to healthy is the worst available failure here.

### 7.2 and 7.3 — Relaying battery, and relaying RSSI

`list_hub_tags` for the coverage denominator, then `get_hub_tag_health` per hub, with `site`. One call returns `connected`, `last_seen`, `tte_seconds`, `rssi` and `flags` per tag, **with hub attribution done server-side.**

Per hub report: distinct tags returned · how many `connected` · how many with `tte_seconds` · how many with `rssi` · how many raising `0x0400`.

| Reading | Grade |
|---|---|
| ≥ 50 % of the hub's own trailing 28-day hour-of-day median | healthy |
| < 50 % | **WATCH** |
| < 20 % | **AMBER** |
| Zero across a working day with live tags present | **RED for that hub** |
| **Zero live tags in coverage** | **not evaluated — NOT failed** |

**Report battery and RSSI on separate lines. Never OR them into one "any data" boolean.** Both stopping together points at the hub or its uplink. One stopping while the other continues points at a specific field or pipeline stage — a much narrower diagnosis, and an OR destroys it.

**The denominator gate is load-bearing right now.** The 2026-08-25 baseline was 31 tags with **4 reporting live** and 11 stale over a month; the 2026-09-08 tag check found 9 of 12 monitored tags NEVER_SEEN. **With the fleet in that state, low relay volume is the expected reading and says nothing about the hub.** Without this gate the check blames the hub for a dead tag fleet. **Read `/koto-tag-check`'s last output before grading 7.2 or 7.3 at all.**

> The tool gives **current state per tag, not counts over a window.** Distinct-tag counts are computable directly; the volume-vs-median test needs repeated sampling or the metric-history endpoint. **Grade the distinct-tag part now and mark the volume test `not measured`** — do not estimate it.

### 7.4 Not being rejected by the tags — `hub_not_found`

Count of **distinct** tags raising `0x0400` in the flags bitmask over the window.

| Reading | Grade |
|---|---|
| Any tag | **WATCH** |
| Multiple distinct tags, or a count above `flag_0x0400_distinct_tags` in state | **AMBER** |

**This is the strongest hub signal on the site** — the tag telling us directly that it could not find its hub, rather than us inferring a fault from absent data.

**Deduplicate by tag before counting.** The alert stream here is heavily duplicated; the baseline for how bad it gets is 12 identical `A4_kps_6fe0` alerts stamped the same minute, and one tag with ~19 identical battery-low entries at a single timestamp. **Report distinct tags, raw count in parentheses.**

### 7.5 Stable — and this one has no tool

Uptime fraction · longest gap · flap count · recovery, same shape and thresholds as the reader definition's criterion 3.

> **There is no `get_hub_metric_history`.** `get_hub_status` is point-in-time. **A hub that flapped eleven times this morning and is up right now reads identical to one solid for a week.**
>
> **Say this in every report.** It is the largest hole in the hub definition and it does not close by being tactful about it. Criterion 5 must come from repeated sampling across runs, the ZLP `/query/device-metric-history/` endpoint, or a Grafana recording rule — none of which this command has.

**Recovery carries forward across runs.** The reconnect-after-ISP-recovery firmware bug applies to hubs too, and hubs run their own version line — `0.9.1.13`, not the anchors' `v1.7.8-0-g8cfcc1a`. **A hub that only came back because someone power-cycled it is not healthy today.**

### 7.6 Triage — only when 7.1–7.4 flag

`get_hub_logs` for the flagged hub · `check_tag_hub_connectivity` for flagged tags · `get_node_config` if a configuration question is open. **Logs are for diagnosis, not grading — do not pull them on every run.**

---

## 8. WEEKLY DRIFT

On `weekly`:

- `list_site_wifi_nodes` — hub / anchor / tracker counts. Compare to state.
- `get_site_firmware_inventory` — **state which layer you are reading.** Hubs run `0.9.1.13`, anchors run `v1.7.8-0-g8cfcc1a`, and `hub` is *also* an anchor firmware sub-component. **Reading the wrong layer reports drift that is not there and misses drift that is.**

---

## 9. CORRELATE BEFORE YOU LIST

**Before writing a single hub finding, ask in this order:**

1. **Is the site dark?** Call `get_site_last_locate` and read `/koto-reader-check`'s `STATE`. If readers are also silent, this is **one site failure**, not a hub fault — say so and stop grading the hub as the cause.
2. **Is the tag fleet dead?** Low relay volume with no live tags is **not evaluated**, per §7.2. Read `/koto-tag-check`'s last output first.
3. **Is this one hub or all of them?** Correlated failure across hubs is one finding naming the uplink, not one per hub.
4. **Is this actually a reader problem?** Ranging failures clustered in one hub's coverage may be reader calibration. `run_site_coherency_audit` gives the root-cause code — **quote it verbatim**: `wificloud_offline` · `zlp_pipeline_lag` · `inventory_mismatch` · `z_outlier` · `active_alerts` · `healthy`.

> **Hubs and readers are graded by separate commands, and the correlation between them is real.** When a finding points at gates 1, 2 or 4, name the other monitor in the report so whoever reads it knows which to run — and say plainly that this run did not check it.

**Everything here is cloud-side.** A silent hub means "not reaching the cloud", not "the hub is dead". Distinguishing them needs a local ping via the Netgear AP proxy on `192.168.179.x`, which this command cannot do.

---

## 10. HUB GRADE

| Level | Condition |
|---|---|
| ⚫ **UNKNOWN** | Monitoring path unavailable, or the hub roster could not be resolved. **Condition not established.** |
| 🔴 **P2 / RED** | Hub silent during working hours **with location output degraded** † |
| 🟠 **P3 / AMBER** | Relay volume < 20 % of baseline with live tags present · multiple distinct tags raising `0x0400` · correlated instability across hubs |
| 🟡 **WATCH** | Heartbeat > 10 min · NTP unsynced · new crash report · `connected`/heartbeat contradiction · memory or disk trending down · any single `0x0400` tag |
| 🟢 **GREEN** | Roster resolved, all five criteria met, nothing outstanding |

† **The hub-silent P2 is conditional on §2 resolving to "the hub is in the data path."** If Crane demonstrably works with its only named hub dark, hub-silent is not a P2 here and the incident register needs amending. **Say which reading you are applying.**

Reference thresholds: hub heartbeat ~60 s (warn > 10 min, error > 30 min).

### Alerting and cadence — not this command's job

**This command grades and reports. It does not decide when to run, who to tell, or what escalates.** Cadence, delivery and any confirm-before-paging ladder belong to whatever invokes it — the Agent Scheduler task, not this file.

One routing rule it still owes whatever does escalate, because it is analysis rather than scheduling: **if the readers are also red, this is not the monitor to act on.** The report must say so, so the site finding gets actioned rather than this one.

**Never put keys, tokens or dashboard credentials in any output.**

---

## 11. STANDING ITEMS — carry every run until closed

- **Hub roster: 5 counted, 0 enumerated** (§2). `list_hubs` settles it in one call; check for `hub_b591` first.
- **`hub_b5c7` silent 30+ days** while the site produced locations — which makes the single-point-of-failure assumption, and this command's severity ceiling, an open question.
- **Criterion 5 has no tool** (§7.5). Hub stability is not monitored by anything, here or elsewhere.
- **`last_heartbeat` is a humanised string.** Every threshold comparison is prose-parsing against an undocumented format.

## Open questions

1. **Which hub is the hub at Crane?** Five counted, zero enumerated. Blocks everything — but it is a five-second question, not a research project.
2. **A machine-readable heartbeat timestamp.** Criteria 7.1 and 7.5 cannot be graded reliably by parsing prose. Who owns the MCP tool surface?
3. **Hub metric history** — is the ZLP `/query/device-metric-history/` series populated for hub nodes? If yes, §7.5 is a query. If no, it is an instrumentation gap.
4. **Is the hub actually a single point of failure at Crane?** The spec says yes; current evidence says the site works with its only named hub dark. The answer sets this command's severity ceiling.
5. **Metric retention** — the trailing-median baselines in 7.2–7.3 need 28 days of history behind whatever backs question 3.
6. **Where "needed a manual power cycle" gets recorded** so 7.5's recovery attribute survives across runs.
7. **Fix the Confluence user guide.** Its 119-tool inventory omits the hub diagnostic tools entirely, and it already sent one hub-monitoring effort down a dead end.

---

## 12. REPORT

```
KOTO CRANE HUBS — <emoji + level> — <YYYY-MM-DD HH:MM JST / HH:MM PT> — mode: <check|triage|weekly>

One line: what is true right now.

Hubs        <n> seen (registry claims 5)  <names>   hub_b591 present: <y/n>
Roster      <resolved | UNRESOLVED — §2 blocker still open>
Site context <from /koto-reader-check STATE: readers <n>/29, grade <x>, run <when> | unavailable>
Change      <vs last run: hubs added/disappeared, 0x0400 count delta | unavailable — no prior state>

=== FIVE CRITERIA, PER HUB ===
<hub>   heartbeat <parsed minutes>   NTP <synced?>   crash <none|msg>   mem/disk <trend>
        battery relay: <n> distinct tags (<raw>)     rssi relay: <n> distinct tags (<raw>)
        0x0400 hub_not_found: <n> distinct tags (<raw>)   coverage gate: <evaluated|not evaluated — no live tags>
        contradiction: <none | connected:true while <n>m silent>
        stability: NOT MEASURED — no hub history tool exists

Findings
- <finding, with the number that triggered it — after §9 correlation>

Points at another monitor
- <e.g. "readers also silent — one site failure, run /koto-reader-check. NOT checked by this run.">

Standing items
- <from §11, unchanged until closed>

Recommended actions (for a human)
- <action> — <who>

Not checked
- Hub stability (uptime, longest gap, flap count) — no hub history tool exists.
- Readers — graded by /koto-reader-check.
- Tags — graded by /koto-tag-check.
- Local reachability — cloud-side only.
```

### Say these every run

1. **That hub stability is not covered at all**, and why.
2. **Whether the hub roster is still unresolved**, and therefore whether the P2 ceiling applies.
3. **That everything here is cloud-side.** A silent hub means "not reaching the cloud".
4. **⚫ UNKNOWN ≠ UNHEALTHY ≠ HEALTHY.** Never collapse them.
5. **Whether the site context was available**, or whether this ran blind to the reader state.

---

## Seed state — write to `.koto-hub-state.json` if absent

```json
{
  "last_run": null,
  "source": "seed — registry export 2026-08-25, unvalidated",
  "unknown_runs": 0,
  "grade": null,
  "hubs_seen": [],
  "hub_roster_resolved": false,
  "hub_b591_present": null,
  "flag_0x0400_distinct_tags": 0,
  "manual_power_cycles": {},
  "history": [],
  "standing_items_open": [
    "hub_count_5_vs_0_enumerated",
    "hub_b5c7_silent_30d",
    "criterion5_no_tool",
    "last_heartbeat_is_a_string"
  ],
  "context_2026_09_08": {
    "tag_fleet": "9 of 12 monitored tags NEVER_SEEN — the 7.2/7.3 coverage gate will bite",
    "infrastructure": "29 readers healthy, no reader-offline alerts",
    "blocking_question": "Is Subcon C still on site? Affects whether low relay volume means anything."
  }
}
```
