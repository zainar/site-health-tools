# What counts as a healthy hub — Koto crane site

**Status:** draft for review, 2026-09-01. Rev 3 — Misha's two-criterion definition, rewritten with the recommendations folded in, then mapped onto the real hub MCP tool surface from `hub_mcp_tools_reference.md`. Answers step 3 of `reader-health-check-2026-08-25.md` ("decide how `hub_b5c7` gets health-checked, since it has no `sensors.tempC`") and the open hub item in `reader-health-definition.md`.
**Scope:** hubs at `Koto_Pumping_Station_Crane`. Excavation's `hub_b5b3_caisson` is out of scope — known-broken, pending physical relocation, possibly behind the Kajima firewall (`hub-monitor-export-assessment.md` §4). Structure deliberately mirrors `reader-health-definition.md` so the two read as a pair.

---

## The frame

The starting definition was:

> Hub connected, and whether it received any tag battery info or RSSI info in the last 24 hours.

That frame is kept: for a hub, **"is data passing through me"** is the health question. It is the right instinct, and it sidesteps the fact that `hub_b5c7` reports no `sensors.tempC` rather than waiting for a sensor the hub does not have.

What changed in this revision, and why:

| Original | This revision | Reason |
|---|---|---|
| "connected" | heartbeat age, with `connected` carried as a separate contradiction check | A registry flag is a claim, not evidence (`kajima-site-monitor-spec.md` §2). `R55_d299` showed `Active` while four days offline. |
| "last 24 hours" | grade on newest-sample age (10/30 min); characterise over 24 h | A hub can be dead 23 hours and pass a 24-hour any-of test. The documented failure mode is daytime-only, peaking ~16:00 JST. |
| "any … info" | volume and distinct-tag count vs the hub's own hour-of-day baseline | "Any" passes on one report from one tag. Same trap `reader-health-definition.md` §2c solved for readers. |
| battery **or** RSSI | battery **and** RSSI, reported as separate lines | Different fields, potentially different paths. If one stops and the other continues, that is the diagnosis — an OR destroys it. |
| — | `0x0400 hub_not_found` added as its own criterion | The tag saying directly that it could not find its hub. Affirmative evidence, not inference from absence. Already in the data, free to query. |
| healthy / unhealthy | ⚫ UNKNOWN as a third state | All of this is cloud-side. In the May outage all 30 readers answered local pings while every cloud view was dark. |

---

## The definition

A hub is **healthy** when all five hold.

**Evaluation windows — both, and they do different jobs.** Never let the long one grade the site.

| Window | Question | Use |
|---|---|---|
| **Newest-sample age** — 10 min warn / 30 min error | is the hub up *now* | **grading and alerting** |
| **Rolling 24 h**, weighted to JST working hours (08:00–18:00 Mon–Sat) | uptime fraction, longest gap, flap count, trend | the report body |

---

### 0. Present — it is the hub of record

On an authoritative hub roster for the site, with identity matching across WiFi-Cloud and ZLP.

**Blocked today, and this is the first thing to fix.** The registry counts **5 hubs** at Crane and enumerates none of them — there is no hub list anywhere in the export. The only hub we can name, `hub_b5c7`, sits in the silent-21 group of `public.readers`: nothing reported for 30+ days, no `sensors.tempC` — while the site graded 🟢 on 28 live readers over the same window and locations were being produced.

So exactly one of these is true and we do not know which:

1. `hub_b5c7` is a stale roster entry like the `r20`–`r37` block, and the live hub is one of the other four.
2. `hub_b5c7` is real and dead, and the Crane data path does not depend on it — in which case "the hub is a single point of failure" needs revisiting, because it demonstrably is not one right now.

A check pointed at a ghost record produces a permanent red, and a permanent red is muted within a week. **Settle which hub is the hub before automating anything below.**

### 1. Alive — its own heartbeat reaches the cloud

Newest host-metrics sample within **10 min** (warn) / **30 min** (error), against the documented hub SLA of ~60 s expected (`kajima-site-monitor-spec.md` §3).

- **`tempC` is not required and its absence is not a fault.** `cpu_idle`, `system_free_memory`, `process_memory_usage.*` and `disk_free.*` are present for 63 devices at ~1 sample/min and are a de-facto 60 s heartbeat. That closes the missing-`tempC` gap with no new instrumentation.
- **Carry `connected` alongside, as its own field.** A hub reporting `connected: true` while silent past the warn threshold is a **contradiction finding** — more informative than either fact alone, and the same class of finding as the `R55_d299` case. Report it explicitly; do not resolve it silently in favour of one source.
- **Grade host resource health while reading those fields.** `disk_free` trending toward zero, or `system_free_memory` falling day over day, fails later today. That is prediction rather than detection, and it is the hub's equivalent of the readers' thermal trend.
- **NTP sync is a criterion, not a detail.** `get_hub_status` returns it. A hub whose clock has drifted timestamps everything downstream of it wrongly while looking perfectly alive — the hub's counterpart to the readers' LPS-sync criterion. Any hub reporting NTP unsynced = WATCH.
- **Crash reports** come back in the same call. Any new crash report = WATCH, regardless of what the heartbeat says.
- A gap means **"not reaching the cloud"** — never report it as "the hub is dead."

### 2. Relaying — tag battery data is passing through it

- **Distinct tags** reporting `charge` via this hub in the window, and the **volume** of those reports.
- Healthy = each **≥ 50 % of this hub's own trailing 28-day median for the same hour-of-day.** < 50 % = WATCH · < 20 % = AMBER · zero across a working day with live tags present = RED.
- **Gate on the denominator.** ≥ 1 live tag in the hub's coverage, or the criterion is **not evaluated** — not failed. The 2026-08-25 baseline was 31 tags with **4 reporting live** and 11 stale over a month; with the fleet in that state, low volume is the expected reading and says nothing about the hub. Without this gate the check blames the hub for a dead tag fleet.

### 3. Relaying — RSSI is passing through it

Same test, same thresholds, same denominator gate, applied to `rssi`. **Reported on its own line, never OR'd with criterion 2.**

Battery and RSSI stopping together points at the hub or its uplink. One stopping while the other continues points at a specific field or pipeline stage, and is a much narrower diagnosis. Collapsing them into a single "any data" boolean throws that away.

### 4. Not being rejected by the tags — `hub_not_found`

Count of **distinct** tags raising `0x0400 hub_not_found` in the flags bitmask over the window.

| Reading | Grade |
|---|---|
| Any tag | WATCH |
| Multiple distinct tags, or a count rising day over day | AMBER |

This is the strongest hub signal on the site: it is the tag telling us directly that it could not find its hub, rather than us inferring a fault from absent data. `flags` is already in `device_metrics_history` for 53 devices.

**Deduplicate by tag before counting.** The alert stream here is heavily duplicated — baseline: 12 identical `A4_kps_6fe0` alerts stamped the same minute. Report distinct tags affected, raw count in parentheses.

### 5. Stable — connection quality over time, not a binary

Four numbers per hub per window, same shape as the reader definition:

| Metric | Definition | Healthy |
|---|---|---|
| **Uptime fraction** | received heartbeats ÷ expected, over JST working hours | ≥ 99 % |
| **Longest single gap** | max continuous silence in the window | < 10 min |
| **Flap count** | distinct disconnect→reconnect events | ≤ 2 / day |
| **Recovery** | returned without human intervention, and how fast | self-recovered, < 5 min |

**Report the longest gap next to the fraction, always.** 99 % with one 20-minute hole during a lift is worse than 99 % spread over a hundred 5-second blips.

**Recovery carries forward across runs.** The reconnect-after-ISP-recovery firmware bug applies to hubs too, and hubs run their own version line — `0.9.1.13`, not the anchors' `v1.7.8-0-g8cfcc1a`. State which layer you are reading or you will report drift that is not there. **A hub that only came back because someone power-cycled it is not healthy today, even though it is up today.**

---

## ⚫ UNKNOWN is a grade, not a failure

Every criterion above is observed **cloud-side**. During the May outage all 30 readers were alive and answering local pings while every cloud view was dark.

| Grade | Meaning |
|---|---|
| 🟢 **HEALTHY** | All five criteria met over the evaluation window |
| 🟡 / 🟠 / 🔴 | Fails one or more — per-criterion thresholds above |
| ⚫ **UNKNOWN** | Could not observe it. Monitoring path down, hub roster unresolved, site uplink down. Condition not established. |

**A hub UNKNOWN is a different fact from a hub UNHEALTHY, and both differ from healthy.** Conflating the first two is what let a 13-day outage stay hidden. Never collapse them.

**Hub state feeds the site grade using the register's existing language** (`kajima-site-monitor-spec.md` §3): hub silent is one of the three 🔴 P2 conditions, alongside all-or-near-all readers silent and zero location events during working hours. Do not invent a parallel scale — but note that this severity ceiling is only correct if criterion 0 resolves to *the hub is in the data path*. If Crane works with its named hub dark, hub-silent is not a P2 at this site and the register needs amending.

---

## Which tools check this definition

**Source: `hub_mcp_tools_reference.md` (generated 2026-09-01, zlp-prd-jpn) — 18 hub-related tools, 8 of them `engineering:read`.**

**This supersedes what the Confluence pages say, and it corrects two things in this project's own docs.** The [Engineering API MCP User Guide](https://zainar.atlassian.net/wiki/spaces/SD/pages/436404230) tool inventory (119 tools) does not list the hub diagnostic tools at all, and the [WiFi Diagnostics Engineering Spec](https://zainar.atlassian.net/wiki/spaces/SD/pages/374603778) still carries `list_hub_tags`, `get_hub_tag_health` and `get_hub_logs` as unchecked TODO items. All three are registered and read-scoped in production. **Treat the reference guide as current and both Confluence pages as stale** — and get the user guide fixed, because it is the page people onboard from.

> **Correction to `hub-monitor-export-assessment.md` §1.** That doc's secondary claim — *"`get_hub_status` is not in the MCP tool inventory"* — is **wrong**. The tool exists, read-scoped, with the payload shape that script assumed. Its primary finding stands unchanged: `claude mcp call` is not a real subcommand, so the script still cannot work and still must not be scheduled. The tool name was the one thing it got right.

### Coverage, criterion by criterion

| Criterion | Tool | Field it grades on | Status |
|---|---|---|---|
| **0** Present | `list_hubs` · `list_site_wifi_nodes` | hub IDs and names at site | ✅ **covered** |
| **1** Alive | `get_hub_status` | `connected`, `last_heartbeat`, `system` (memory, IP, MAC, **NTP sync**), `crash_report` | ✅ **covered** |
| **2** Battery relay | `get_hub_tag_health` | `tte_seconds` per tag, `last_seen` | ✅ **covered** |
| **3** RSSI relay | `get_hub_tag_health` | `rssi` per tag | ✅ **covered** |
| **4** `hub_not_found` | `get_hub_tag_health` · `check_tag_hub_connectivity` | `flags` per tag · per-tag hub connection | ✅ **covered** |
| **5** Stable over time | — | — | ❌ **no tool. Point-in-time only** |
| Coverage gate (2–3) | `list_hub_tags` | `tag_node_ids`, `count` | ✅ **covered** |

**Four of five criteria are servable today by two read calls.** That is a materially better position than the definition assumed, and it changes the build scope for the third time — so verify against the live server before planning anything on it.

### `get_hub_tag_health` is the tool this definition was waiting for

It returns, per tag on a named hub: `connected`, `last_seen`, `tte_seconds`, `rssi`, `flags`. That is criteria 2, 3 and 4 in a single call **with hub attribution built in**.

**This closes open questions 2 and 6 outright.** The attribution problem was the largest unknown in rev 2 — whether battery/RSSI rows could be joined to a receiving hub at all. They can: the tool does the join server-side, and `list_hub_tags` supplies the denominator (which tags belong to this hub) that criteria 2–3's coverage gate needs. Neither needs a new query, a baseline table, or a Grafana rule.

### Recommended run order

The reference guide's own hub health workflow maps onto the criteria almost exactly. Use it:

1. `list_hubs` → criterion 0. **Settles the 5-hubs-0-enumerated blocker in one call.**
2. `get_hub_status` per hub → criterion 1 (heartbeat, NTP, memory, crash report).
3. `list_hub_tags` → the coverage denominator for the next step.
4. `get_hub_tag_health` → criteria 2, 3 and 4 in one call.
5. `get_hub_logs` **only when steps 2–4 flag something.** Logs are for diagnosis, not grading — do not pull them on every run.

Both `get_hub_status` and `get_hub_tag_health` resolve a hub by name only if `site` is also passed. Always pass `site` = `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`.

### Three problems in the payloads, and they affect grading

**1. `last_heartbeat` is a humanised string, not a timestamp.** The documented examples are `"51m ago"` and `"16d ago"`. Criterion 1 grades against 10-minute and 30-minute thresholds, and criterion 5 needs arithmetic — both require parsing prose, with no stated behaviour at the unit boundaries (does 90 s render as `"1m ago"` or `"90s ago"`? what rounds to `"1h ago"`?). **Ask for a raw epoch or ISO field alongside it.** Until then, any threshold comparison is string-parsing against an undocumented format, and a parse miss reads as healthy. Same problem for `last_seen` per tag.

**2. No history on any hub read tool.** `get_hub_status` is point-in-time; there is no `get_hub_metric_history` to match the anchors' `get_anchor_metric_history`. So **criterion 5 stays where rev 2 put it** — uptime fraction, longest gap, flap count and recovery must come from repeated sampling, the ZLP `/query/device-metric-history/` endpoint, or a Grafana recording rule. A single `get_hub_status` call cannot tell a hub that has been solid for a week from one that flapped eleven times this morning and happens to be up right now.

**3. `get_hub_tag_health` reports `connected` per tag but the criteria need volume.** The tool gives current state per tag, not counts of reports received over a window. Distinct-tag counts are computable from it directly; the *volume vs trailing-median* test in criteria 2–3 still needs either repeated sampling or the metric-history endpoint. Grade the distinct-tag part from the tool now, and treat the volume test as the part that waits on history.

### Safety — sharper than the earlier note

The reference guide states plainly: **"Production environment: zlp-prd-jpn has write+admin permissions."** So the credential in use can execute every write tool below against live hardware at a site where crews work under crane loads. This is exactly the risk `eng-api-mcp-connector-setup.md` flagged as hypothetical; it is not hypothetical.

Hub write tools, none of which a health check ever calls: `configure_hub` · `set_hub_reassoc_timeout` · `set_hub_provisioning_mode` · `activate_hub_provisioning` · `create_hub` · `upgrade_hub_firmware` · `reboot_tag_host` · `reboot_node` · `set_node_name` · `update_node_config`.

Two specific traps:

- **`reboot_tag_host` reads like a tag tool and is not.** It reboots the tag-management process *on the hub* — it takes out tag handling for every tag on that hub at once. The Confluence spec even files it under *Tag Actions*, which is how a name-similarity mistake happens.
- **Dry-run is the default, and it is the only guardrail.** Every write defaults to `confirm: false`. That protects against a mistaken call and not against a mistaken `confirm: true`. Keep the read-only rule from `koto-crane-check-command.md` absolute: propose remediation in the report, let a human who knows what is happening on site that hour execute it.

Read-scoped and safe: `list_hubs`, `get_hub_status`, `list_hub_tags`, `get_hub_tag_health`, `get_hub_logs`, `get_node_config`, `list_site_wifi_nodes`, `check_tag_hub_connectivity`.

### One lead on criterion 0

Every example in the reference guide that pairs a hub with the **Crane** site ID uses **`hub_b591`** — not `hub_b5c7`, the hub this project has assumed throughout. It is only an example (`hub_b4e6_office` also appears), so it is a lead and not a finding. But it is a specific name to check against `list_hubs` first, and if `hub_b591` is live at Crane it would explain how the site produces locations while `hub_b5c7` has been silent 30+ days.

---

## Open questions

1. **Which hub is the hub at Crane?** 5 counted, 0 enumerated. `list_hubs` answers it in one call, and `hub_b591` is the name to check first. Still blocks everything — but it is now a five-second question, not a research project.
2. **A machine-readable heartbeat timestamp.** `last_heartbeat` / `last_seen` are humanised strings. Criteria 1 and 5 cannot be graded reliably without a raw epoch field. Who owns the MCP tool surface?
3. **Hub metric history.** No hub equivalent of `get_anchor_metric_history` exists, so criterion 5 has no tool. Is the underlying ZLP `/query/device-metric-history/` series populated for hub nodes? If yes, criterion 5 is a query. If no, it is an instrumentation gap.
4. **Is the hub actually a single point of failure at Crane?** The spec says yes; current evidence says the site works with its only named hub dark. The answer sets this definition's severity ceiling.
5. **Metric retention** — the trailing-median baselines in criteria 2–3 need 28 days of history behind whatever series backs question 3.
6. **Where "needed a manual power cycle" gets recorded** so criterion 5's recovery attribute survives across runs.
7. **Fix the Confluence user guide.** Its 119-tool inventory omits the hub diagnostic tools entirely, and it already sent one hub-monitoring effort down a dead end.

> Still worth someone's attention: the engineering spec's example reports **6 hubs** at Koto Excavation while the registry export counts **5** at Crane. Different sites, so not a contradiction — but no expected hub count has been stated for either, and criterion 0 needs a denominator the same way the reader roster did.
