---
description: Koto Crane whole-site health check (zlp-prd-jpn). Read-only. Runs the reader, hub and tag checks in order, applies the cross-checks none of them can do alone, and returns one site grade.
argument-hint: "[check | deep | weekly]  (default: check)"
allowed-tools: mcp__zlp-prd-jpn__resolver_status, mcp__zlp-prd-jpn__who_am_i, mcp__zlp-prd-jpn__get_site_last_locate, mcp__zlp-prd-jpn__get_site_health, mcp__zlp-prd-jpn__get_site_last_metrics, mcp__zlp-prd-jpn__get_site_summary, mcp__zlp-prd-jpn__list_readers, mcp__zlp-prd-jpn__list_stale_anchors, mcp__zlp-prd-jpn__generate_anchor_list_csv, mcp__zlp-prd-jpn__get_anchor_status, mcp__zlp-prd-jpn__get_anchor_metric_history, mcp__zlp-prd-jpn__get_anchor_ranging_history, mcp__zlp-prd-jpn__check_anchor_calibration, mcp__zlp-prd-jpn__check_anchor_lps_address, mcp__zlp-prd-jpn__list_site_wifi_nodes, mcp__zlp-prd-jpn__run_site_coherency_audit, mcp__zlp-prd-jpn__check_wificloud_health, mcp__zlp-prd-jpn__check_pipeline_coherency, mcp__zlp-prd-jpn__check_inventory_coherency, mcp__zlp-prd-jpn__explain_zlp_issue, mcp__zlp-prd-jpn__get_site_firmware_inventory, mcp__zlp-prd-jpn__list_hubs, mcp__zlp-prd-jpn__get_hub_status, mcp__zlp-prd-jpn__list_hub_tags, mcp__zlp-prd-jpn__get_hub_tag_health, mcp__zlp-prd-jpn__get_hub_logs, mcp__zlp-prd-jpn__check_tag_hub_connectivity, mcp__zlp-prd-jpn__get_node_config, mcp__zlp-prd-jpn__list_assets, mcp__zlp-prd-jpn__get_tag_by_asset, mcp__zlp-prd-jpn__list_assets_with_tags, mcp__zlp-prd-jpn__list_trackers, mcp__zlp-prd-jpn__list_offline_tags, mcp__zlp-prd-jpn__list_continuous_mode_tags, mcp__zlp-prd-jpn__get_continuous_tag_health, mcp__zlp-prd-jpn__get_asset_history, mcp__zlp-prd-jpn__get_tag_location_history, mcp__zlp-prd-jpn__get_tag_status, mcp__zlp-prd-jpn__get_tag_battery, mcp__zlp-prd-jpn__get_tag_fuel_gauge, mcp__zlp-prd-jpn__get_tag_events, mcp__zlp-prd-jpn__get_tag_config, Read, Write, Bash(date:*), Bash(TZ=*)
---

# Koto Crane — whole-site check — `$ARGUMENTS`

**Site**: `Koto_Pumping_Station_Crane` · `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`
**Account**: `Kajima-Koto` · **Env**: `zlp-prd-jpn` (WIFI_RT) · **NID** 16

> **This is a safety system.** Red-zone and proximity alerts warn workers standing under crane loads. A silent site is urgent, not a data-quality issue.

## What this adds, and what it deliberately does not

It runs the three health checks in the order their dependencies require, then applies the
**cross-checks that none of them can do alone** — because none of them can see another's data.

**It does not define a single threshold of its own.** Every grade comes from the three command files;
this one only sequences them and resolves the boundary cases between them. If you find yourself
inventing a number here, it belongs in the file that owns that layer, and this file should cite it.

**It is not the usage reports.** `/koto-tag-usage` and `/koto-crane-usage` answer "was this in
service", which is a different question from "is this healthy" and carries APPI weight this command
deliberately does not touch. Run those separately.

---

## INSTALL — handled by the plugin

Ships in the **kajima-crane-site-survey** plugin as `/koto-site-check`. Every tool is prefixed
`mcp__zlp-prd-jpn__`; confirm with `claude mcp list` that the server is registered under that exact
name, or the `allowed-tools` list silently fails to match.

**`ZLP_PRD_JPN_API_KEY` must be scoped `viewer` / `engineering:read`.** The preflight halts otherwise.

---

## 0. PREFLIGHT — once, for all three

Run this yourself rather than letting each sub-check repeat it.

1. `resolver_status`. On failure → print and **STOP**:

```
⚫ KOTO CRANE SITE — MONITORING UNAVAILABLE
Site condition is UNKNOWN. Not healthy, not unhealthy. Nothing was checked.
Cause: <the error>
```

2. **`who_am_i`. If it returns write or admin scope, print this and STOP:**

```
⚫ KOTO — REFUSING TO RUN ON A WRITE-SCOPED KEY
The key resolved to <scope>, not viewer / engineering:read.
Site condition is UNKNOWN — nothing was checked.
```

A shell plus a write+admin production credential means the key's scope, not the `allowed-tools`
allowlist, is what stands between this monitor and live hardware. That is a reason not to run.

3. Print both clocks:

```bash
date -u '+UTC %Y-%m-%d %H:%M'
TZ=Asia/Tokyo date '+JST %Y-%m-%d %H:%M (%a)'
TZ=America/Los_Angeles date '+PT  %Y-%m-%d %H:%M'
```

4. Read `.koto-reader-state.json`, `.koto-hub-state.json` and `.koto-tag-state.json` from the working
   directory. Any that are missing → seed from the relevant command file, and report that run's
   `Change` line as `unavailable`, **never as "no change"**.

---

## 1. RUN THE THREE, IN THIS ORDER

The order is a dependency, not a preference. Read each file and follow its procedure for the mode in
`$ARGUMENTS`:

| # | Follow | Mode | Why this position |
|---|---|---|---|
| 1 | `${CLAUDE_PLUGIN_ROOT}/commands/koto-reader-check.md` | as given | **Only check that runs the site sweep.** Everything below needs to know whether the site itself is up. |
| 2 | `${CLAUDE_PLUGIN_ROOT}/commands/koto-tag-check.md` | as given | Gives the live-tag count the hub's coverage gate needs. |
| 3 | `${CLAUDE_PLUGIN_ROOT}/commands/koto-hub-check.md` | as given | Last, because its two relay criteria are **not evaluated** without a live tag denominator. |

**This reorders the pair.** `/koto-hub-check` tells you to read the reader state first; it says
nothing about tags because, run alone, it has no way to get them. Run as a set, the tag check comes
second so the hub check has a real denominator instead of a gate it can only mark `not evaluated`.

**Do not skip a check because an earlier one was clean.** A green reader pass does not imply healthy
hubs or tags — that inference is exactly what §2 exists to prevent.

**Follow each file's own read-only rules.** They are absolute and this command does not relax them.

---

## 2. CROSS-CHECKS — the part no single command can do

Apply these **before** writing any finding. Each one collapses what would otherwise be several
reports' worth of separate findings into one true statement.

**Gate 1 — is the site dark?**
Zero location events during working hours **and** all-or-near-all readers silent **and** hub silent
**and** tags silent → **one 🔴 P2: the site is not reaching the cloud.** Not four findings. Report it
once and stop attributing it to any layer.

> And say the limit out loud: every tool here reads **cloud-side** state. In the May 2026 outage all
> 30 readers answered local pings while every cloud view was dark. **A dark site means "not reaching
> the cloud", never "the hardware is dead"** — separating them needs a local ping via the Netgear AP
> proxy on `192.168.179.x`, which none of these commands can do. Escalate to someone on the site
> network rather than diagnosing further.

**Gate 2 — is a reader finding actually a hub finding?**
Multiple readers showing `0x0400 hub_not_found`, or ranging failures clustered in one hub's coverage
→ **hub finding.** The reader check can only point at this; you have the hub data, so resolve it.

**Gate 3 — is a hub finding actually a dead tag fleet?**
Low or zero battery/RSSI relay **with no live tags in coverage** → the hub criteria are
**not evaluated, not failed.** The 2026-09-08 baseline was 9 of 12 monitored tags NEVER_SEEN; with
the fleet in that state, low relay volume is the expected reading and says nothing about the hub.
**You have the tag count from step 1.2 — use it rather than marking the gate unresolved.**

**Gate 4 — is a tag finding actually the site?**
≥ 3 tags at ERROR **with readers or hub also degraded** → one site finding. Tags cannot produce fixes
without readers, and a dark site presents as twelve dead tags.

**Gate 5 — is this the afternoon signature?**
Failures concentrated in daytime JST peaking ~16:00, readers dropping in cohorts, nothing overnight →
**name the cellular modem as prime suspect, once.** Compare live counts against the three state files:
a 29 → 24 → 18 trend across runs is that pattern, and the delta is the only way to see it.

**Only what survives all five gets listed individually.** Baseline for how bad this gets unfiltered:
~38 raw findings collapsing to ~10 distinct.

---

## 3. SITE GRADE

Take the worst of the three, then apply the gates. **Do not average them** — a healthy reader roster
does not offset a dead tag fleet; they answer different questions.

| Level | Condition |
|---|---|
| ⚫ **UNKNOWN** | Preflight failed, or any layer could not be observed. **Condition not established** — different from healthy and from unhealthy, and conflating the first two is what let a 13-day outage stay hidden. |
| 🔴 **P2** | Gate 1, or any single layer at its own P2 |
| 🟠 **P3** | Any layer at its own P3, after gates 2–4 have reassigned what belongs elsewhere |
| 🟡 **WATCH** | Any layer at WATCH · any unresolved cross-gate · any state file missing so a `Change` line is unavailable |
| 🟢 **GREEN** | All three green **and** every gate resolved **and** nothing in any `Not checked` list that a grade depended on |

**GREEN has a specific meaning here and it is narrower than it looks.** It cannot mean "everything is
fine" while hub criterion 5 has no tool at all. **Say what was not measured in the same line that
reports GREEN**, every run, or the grade is a claim the data does not support.

### Alerting and cadence — not this command's job

It grades and reports. When it runs, who hears about it and what escalates belong to the Agent
Scheduler task that invokes it. **Never put keys, tokens or dashboard credentials in any output.**

---

## 4. REPORT

One report, not three. On `weekly`, also `Write` it to `koto-site-YYYY-MM-DD.md`.

```
KOTO CRANE SITE — <emoji + level> — <YYYY-MM-DD HH:MM JST / HH:MM PT> — mode: <check|deep|weekly>

One line: what is true right now, across the whole site.

Readers   <emoji> <n>/29 reporting  (<n> unobserved — UNKNOWN, still in the denominator)
Tags      <emoji> <n>/12 resolved · OK <n> · ERROR <n> · NEVER SEEN <n>
Hubs      <emoji> <n> seen (registry claims 5)   stability: NOT MEASURED — no tool exists
Last fix  <age>        Pipeline  metrics <lag> / location <lag>
Coherency <root-cause code verbatim>  <(standing baseline) | (NEW)>
Change    <vs last run, per layer | unavailable — no prior state>

=== CROSS-CHECKS ===
1 Site dark?           <no | YES — one P2, see findings>
2 Reader → hub?        <n> readers raising 0x0400 → <reassigned to hub | none>
3 Hub relay evaluable? <yes, <n> live tags | NOT EVALUATED — <n> live tags>
4 Tags → site?         <independent | reassigned to site finding>
5 Afternoon signature? <no | YES — modem prime suspect>

Findings  (after correlation — only what survived all five gates)
- <finding, with the number and the layer that produced it>

Standing items
- <carried from all three, deduplicated>

Suspected cause
- <only if evidence supports it — otherwise "insufficient evidence from cloud-side data">

Recommended actions (for a human)
- <action> — <who>

Not checked
- Hub stability — no hub history tool exists, on any run.
- Local reachability — cloud-side only.
- Tag and crane usage — separate commands, not run here.
- <whatever this mode did not cover, and why>
```

### Say these every run

1. **Which mode ran**, and what it therefore did not check. An unrun check is `Not checked`, never a pass.
2. **That everything is cloud-side.** Dark means "not reaching the cloud".
3. **That hub stability is not covered at all**, whatever the hub grade says.
4. **⚫ UNKNOWN ≠ UNHEALTHY ≠ HEALTHY.**
5. **Which cross-gates resolved and which did not** — an unresolved gate is why a finding might be
   filed against the wrong layer.

---

## Cost, and why `check` is the default

Roughly **60–70 calls** on `check` (reader sweep + 12 tags × 3 + hubs), and **150+** on `deep`, where
the reader pass alone is 3N + 2 = 89 for 29 readers. `weekly` adds both drift passes.

`get_anchor_ranging_history` returns raw measurements and `get_asset_history` has no pagination —
both can exceed the tool-result cap. **Test payload size on one reader and one tag before a full
pass**, and if you shorten a window say so rather than silently truncating.

---

## What this command cannot fix

**Readers and hubs are graded by separate commands and the correlation is still manual** — this file
automates the gates, but it is applying judgement after the fact, not sharing data between checks.
A finding filed against the wrong layer is still possible when a gate cannot resolve.

**Nothing here has been validated against a live MCP payload.** Every threshold in the three files it
runs comes from ZaiNar's documented SLAs, the 2026-08-25 registry export, or the `postgres-zlp`
Grafana datasource. **The first runs test the thresholds as much as the site** — when a payload
contradicts a file, the payload is right and that file needs correcting, not this one.
