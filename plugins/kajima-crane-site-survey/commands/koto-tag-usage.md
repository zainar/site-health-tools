---
description: Koto Crane tag USAGE report (zlp-prd-jpn). Read-only. Answers "were these tags actually used, by whom, when, and how much" over any date range — for the 12 monitored tags (Subcon C01–C10, Crane #1, Crane #5).
argument-hint: "[last_24h | last_7d | last_30d | YYYY-MM-DD:YYYY-MM-DD] [all | subcon-c | cranes]  (default: last_7d all)"
allowed-tools: mcp__zlp-prd-jpn__resolver_status, mcp__zlp-prd-jpn__who_am_i, mcp__zlp-prd-jpn__list_assets, mcp__zlp-prd-jpn__get_tag_by_asset, mcp__zlp-prd-jpn__list_assets_with_tags, mcp__zlp-prd-jpn__list_offline_tags, mcp__zlp-prd-jpn__list_continuous_mode_tags, mcp__zlp-prd-jpn__get_asset_history, mcp__zlp-prd-jpn__get_tag_location_history, mcp__zlp-prd-jpn__get_tag_status, mcp__zlp-prd-jpn__get_tag_battery, mcp__zlp-prd-jpn__get_tag_config, Read, Write, Bash(date:*), Bash(TZ=*)
---

# Koto Crane — tag usage — `$ARGUMENTS`

**Site**: `Koto_Pumping_Station_Crane` · `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`
**Account**: `Kajima-Koto` · **Env**: `zlp-prd-jpn` (WIFI_RT)

Self-contained. Companion to `koto-tag-check.md` (device health) — **run that for "is the fleet healthy right now", run this for "was this tag actually in service, and when did that stop".**

---

## THE THREE QUESTIONS — do not blur them

This file answers the middle one. Getting these confused is how a charging problem becomes an accusation about a worker.

| Question | Answered by | Evidence |
|---|---|---|
| **Is the tag working?** | `koto-tag-check.md` | coverage, battery, flags |
| **Was the tag in service?** | **this file** | did it produce location fixes, on which days, how many |
| **Was the tag actually worn?** | **nothing, reliably** | movement — and the threshold is unvalidated (§5) |

> **Reporting proves a tag was powered and in coverage. It does not prove a person was wearing it.** A tag in a site office drawer, powered, reports all day. Every usage figure in this report carries that ceiling, and the report must say so.

---

## INSTALL — handled by the plugin

This command ships in the **kajima-crane-site-survey** plugin. Installing the plugin gives you
`/koto-tag-usage` in every project — there is nothing to copy by hand.

**The plugin also registers the MCP server under the name `zlp-prd-jpn`, and that matters more than
it looks.** Every tool in this file is prefixed `mcp__zlp-prd-jpn__`. If the server ends up
registered under any other name, the `allowed-tools` list in the frontmatter **silently fails to
match** and the command looks broken for no visible reason. Confirm before the first run:

```bash
claude mcp list
```

If you already have that server registered under a different name, rename it rather than editing
prefixes across five files.

**Set `ZLP_PRD_JPN_API_KEY` to a key scoped `viewer` / `engineering:read`.** The credential this MCP
normally carries has **write + admin on production**. `confirm: false` dry-run defaults protect
against a mistaken call, not against a wrongly-scoped key.


**Read-only. Never call a write tool** — and note that **`ping_tag` is a write** (it transmits to the tag), as are `set_tag_tracking_mode`, `set_tag_motion_sensitivity`, `apply_tag_config`, `tag_locate_*`, `tag_sleep_mode_*`, `set_tag_param`, `set_tag_location`, `create_tag`, `update_tag`, `delete_tag`.
---

## 0. PREFLIGHT — fail loud

1. `resolver_status`. On any error → print and **STOP**:

```
⚫ KOTO CRANE TAG USAGE — UNAVAILABLE
Usage is UNKNOWN for this range. Not "no usage".
Cause: <error>
Check: claude mcp list  → confirm the server name matches the mcp__..__ prefix.
```

> **"Could not measure" and "was not used" are different findings.** Reporting the first as the second is how a tooling failure turns into a report that a crew did not work. Never collapse them.

**Scope check, before anything else runs.** Call `who_am_i`. **If it returns write or admin scope, print this and STOP:**

```
⚫ KOTO — REFUSING TO RUN ON A WRITE-SCOPED KEY
The key resolved to <scope>, not viewer / engineering:read.
Condition is UNKNOWN — nothing was checked.
Fix: issue a viewer-scoped key and set ZLP_PRD_JPN_API_KEY to it.
```

This halts rather than warning. The command can run shell commands (`date`, for the clocks), so a
write+admin production credential means the key's scope — not the `allowed-tools` allowlist — is what
stands between this monitor and live hardware. An unscoped key is a reason not to run.

2. Print both clocks. JST = UTC+9. The site runs JST; most tooling around it reports PT.

```bash
date -u '+UTC %Y-%m-%d %H:%M'
TZ=Asia/Tokyo date '+JST %Y-%m-%d %H:%M (%a)'
TZ=America/Los_Angeles date '+PT  %Y-%m-%d %H:%M'
```

3. Parse `$ARGUMENTS`. Default `last_7d all`. Explicit ranges are `YYYY-MM-DD:YYYY-MM-DD`, interpreted in **JST**.

4. Read `.koto-tag-usage-state.json`; seed from §9 if absent. Write at the end, never before.

---

## 1. SCOPE — 12 tags

| Worker | Tag | | Worker | Tag |
|---|---|---|---|---|
| C01 | `kps_289a` | | C06 | `B1_kps_d38f` |
| C02 | `kps_4cdd` | | C07 | `kps_df91` |
| C03 | `kps_cf91` | | C08 | `B2_kps_44f8` |
| C04 | `kps_c79b` | | C09 | `B3_kps_1a93` |
| C05 | `A5_kps_ae92` | | C10 | `kps_2c6d` |

| Equipment | Asset UUID | Tag |
|---|---|---|
| `KotoCrane_#1` | `8889c289-39bb-4b72-a2f9-0207e3cac8de` | `KotoCrane_#1_21ee_Magnet` |
| `KotoCrane_#5` | `f0d10e07-22c8-4809-b1f1-fbc3f8201109` | `KotoCrane_#5_3d9a_Magnet` |

**Resolve by asset UUID via `get_tag_by_asset` — never by name prefix.** Prefixes mismatch the wearer for C05 (`A5_`), C06 (`B1_`), C08 (`B2_`), C09 (`B3_`) — four of ten. Prefix-matching mis-attributes nearly half the set, and a mis-attributed usage finding names the wrong person.

**Working hours: Mon–Fri 08:00–18:00 JST.** Settled 2026-09-16; the single window across every check in this package. **Weekends are outside it and do not count toward days-used or coverage — but they are still reported**, as an out-of-hours line. Saturday working is common on Japanese construction sites, and a genuine Saturday shift must never become indistinguishable from an idle site. **Say in the report which part of the range fell outside working hours.**

> **Saturday is excluded for now, and the reason matters when you read a weekend line.** The device mode scheduler is documented as skipped on JST weekends, so tags may sit in `on_motion` on Saturdays — which produces a legitimately lower fix count than a weekday. **Do not read a thin Saturday as a tag stopping**, and do not pool Saturday fixes with weekday ones. See `/koto-tag-check` §5 for what has to be confirmed before Saturday becomes a graded day.

---

## 2. THE FOUR USAGE STATES

Assign one per tag per range. These are about service, not health.

| State | Definition |
|---|---|
| 🟢 **IN USE** | Produced fixes on the most recent working day in range |
| 🟡 **INTERMITTENT** | Fixes on some working days in range, none on the most recent |
| 🔴 **STOPPED** | Fixes earlier in range, then nothing — **report the stop date and time in JST** |
| ⚫ **NEVER USED** | No fixes anywhere in range. If also none in `last_30d`, say **"never used"** plainly |
| ⚫ **UNKNOWN** | Query failed for this tag. Not "unused" |

**A tag can be STOPPED and perfectly healthy hardware.** The crew may have left. State that possibility every time you report STOPPED.

---

## 3. PULL THE HISTORY — chunk it

Per tag: `get_asset_history(asset_res_name=<uuid>, range=…)` or explicit `start_time`/`end_time`.

> **`get_asset_history` has no pagination and no filters** — it returns the whole range in one response. There is no cursor to fall back on. At ~1000–2000 fixes per tag per day, a 30-day pull for 12 tags is ~500k points and **will** exceed the tool-result cap.

**So chunk by day, aggregate, and discard the raw fixes as you go:**

```bash
# one day at a time, per tag; keep only the daily rollup
```

Per tag per day retain only: fix count · first and last `locationTime` (JST) · intervals covered · movement figures (§5). **Never hold raw fixes across days.** If a single day still overflows, split into morning/afternoon.

**Do not sample and do not silently truncate.** At 12 tags completeness is affordable, and it is the whole point — C05 was found by checking every worker individually.

---

## 4. PER-TAG USAGE TIMELINE

For each tag, over working hours in range:

| Metric | Meaning |
|---|---|
| **Days used** | working days with ≥ 1 fix ÷ working days in range |
| **First / last fix** | JST, with date |
| **Stop time** | JST time of day of the final fix — **carry this, §6 depends on it** |
| **Daily coverage** | intervals with ≥1 fix ÷ 40, per day |
| **Fix count** | per day; a day with 12 fixes is not a day of use |
| **Longest gap** | consecutive working days with no fixes |

**Report every one of the ten workers individually, every run.** A distribution hides exactly what matters here — one worker who was never tracked at all.

---

## 5. MOVEMENT — the "worn" question, with its ceiling stated

Only for tag-days with coverage ≥ 80 %. Below that: **NOT EVALUATED**, and print no fraction.

**Positioning here is ±1–2 m.** A 500 mm threshold on raw fixes is below that floor, so a tag on a bench clears it in most intervals and the test that should detect an unworn tag instead certifies it as worn. Raising to 2 m inverts the error — a worker pacing a 1.5 m platform reads as a bench tag. **No threshold on raw per-fix displacement separates them.**

Noise is roughly zero-mean and uncorrelated; movement is not. **So aggregate before thresholding** — noise shrinks with averaging, movement does not.

```python
import math, statistics, itertools

SITE_DIAGONAL_MM = 45000        # 27.1 m × 34.9 m surveyed extent — hard sanity bound
THRESHOLDS_MM    = (500, 2000)  # dual-threshold on FILTERED positions, until the floor is measured
SUBWINDOW_S      = 180          # 3 min
MIN_FIXES        = 3            # per sub-window; below this the median is not trustworthy

def dist_mm(a, b):
    return math.sqrt((b[0]-a[0])**2 + (b[1]-a[1])**2 + (b[2]-a[2])**2)

def settled_positions(fixes, t0):
    out = []
    for _, grp in itertools.groupby(fixes, key=lambda f: int((f['t'] - t0) // SUBWINDOW_S)):
        g = list(grp)
        if len(g) < MIN_FIXES:
            continue                                   # dropped and counted, NOT stationary
        out.append(tuple(statistics.median(f[k] for f in g) for k in ('x', 'y', 'z')))
    return out

def interval_movement(fixes, t0):
    """(movement_mm, n_settled, n_discarded) or None when there is not enough data."""
    pts = settled_positions(fixes, t0)
    if len(pts) < 2:
        return None                                    # NO_DATA — never 0.0
    ds = [dist_mm(a, b) for a, b in itertools.combinations(pts, 2)]
    ok = [d for d in ds if d <= SITE_DIAGONAL_MM]      # discard positioning errors, count them
    return (max(ok) if ok else 0.0, len(pts), len(ds) - len(ok))

def day_metrics(settled):
    c = tuple(statistics.median(p[i] for p in settled) for i in range(3))
    dists = sorted(dist_mm(c, p) for p in settled)
    path  = sum(dist_mm(a, b) for a, b in zip(settled, settled[1:]))
    net   = dist_mm(settled[0], settled[-1])
    return {"r95_mm": dists[int(0.95 * (len(dists) - 1))],
            "path_m": path / 1000,
            "straightness": (net / path) if path else 0.0}
```

- **Do not take a max over raw fixes** — a max over dozens of noisy samples selects the largest noise excursion and inflates systematically.
- **`None` ≠ `0.0`.** `None` = could not evaluate; `0.0` = measurably still. Collapsing them turns a coverage failure into a statement about a person.
- **Report both thresholds side by side.** `500mm: 94% / 2000mm: 11%` means noise dominates and neither number describes the worker.
- **`r95` is the primary signal, not the movement fraction.** A bench tag stays inside a noise-sized circle all day however much its path wanders; a worn tag crosses a 27 × 35 m site. Report `r95`, `path_m`, `straightness` and **fix cadence** on every worker row — two tags at different cadences are not on the same scale.
- **Report metres, never kilometres.** On a 27 m site a figure in km is always wrong.

---

## 6. DROPOUT ANALYSIS — the part a health check does not do

Run whenever ≥ 2 tags are STOPPED in range. This is where usage data earns its keep.

**6a. Stop-time clustering.** Collect every STOPPED tag's final-fix **time of day in JST** and look for clustering, particularly **16:00–16:45 JST** — the documented cellular-modem signature at this site.

> **State the competing explanation in the same breath: end of shift produces an identical cluster.** This data cannot separate them, and naming the modem from clustering alone is a guess wearing a finding's clothes.

**6b. The equipment control — this is the discriminator.** For each day a worker tag stopped, check what the **crane tags** did on that same day.

| Reading | Interpretation |
|---|---|
| Crane tags stopped at the same time | the **site** went quiet — infrastructure or uplink, not the tag |
| Crane tags kept reporting to ~18:00 JST | the cloud path was up after the worker tag went silent — points at that tag, its battery, or its wearer leaving |

The scheduler switches tags off at **18:00 JST**, so a healthy equipment tag's last fix of the day lands at ~17:57–18:00. A worker tag stopping at 16:20 on a day the crane tags ran to 18:00 is a **100-minute** unexplained gap. Report that gap in minutes per stop event.

**6c. Battery correlation.** For each STOPPED tag pull `get_tag_battery` and check for a low/dead alert near the stop.

| Pattern | Reading |
|---|---|
| Low alert hours before the stop | battery explains it — a **charging** problem, not a discipline one |
| Dead alert shortly after the stop | consistent |
| Dead alert days after the stop | odd ordering — flag it, do not assume it explains the stop |
| No alert at all | **unexplained** — say so; do not infer battery |

**6d. Cascade shape.** Group stops by date. Cohorts dropping in ones and threes over days — rather than all at once — is the documented failure signature at this site. Report the sequence with dates.

---

## 7. REPORT

```
KOTO CRANE TAG USAGE — <range, JST> — run <YYYY-MM-DD HH:MM JST / HH:MM PT>

One line: were these tags in service, and what changed.

Scope      12 monitored (10 worker + 2 equipment)   Working hours: Mon–Fri 08:00–18:00 JST
States     IN USE <n> · INTERMITTENT <n> · STOPPED <n> · NEVER USED <n> · UNKNOWN <n>
Weekend    Sat <n> fixes across <n> Saturdays · Sun <n> — reported, NOT graded
           (a thin Saturday is not a finding — the fleet may be in on_motion)

=== SUBCON C — all ten, individually ===
C01  kps_289a
     State      🔴 STOPPED
     Used       <n>/<n> working days   fixes <n>
     First/last <date time JST>  →  <date time JST>
     Stop        <time JST>, <n> min before the 18:00 switch; crane tags that day ran to <time>
     Battery     <alert + timing, or "no alert — unexplained">
     Movement    <500mm pct>/<2000mm pct>  r95 <n>m  path <n>m  straight <n.nn>  cadence <n>s
                 (or NOT EVALUATED — coverage <pct>%)
...

=== EQUIPMENT — control group ===
#1  KotoCrane_#1_21ee_Magnet   used <n>/<n> days   last <time JST>
#5  KotoCrane_#5_3d9a_Magnet   used <n>/<n> days   last <time JST>

=== DROPOUT ===
Cascade     <date>: <tags> · <date>: <tags> · ...
Stop-time cluster   <n> of <n> stops fell between <HH:MM> and <HH:MM> JST across <n> days
                    Competing explanation: end of shift produces the same shape. Not separable here.
Equipment control   <site went quiet | cloud path was up — gap of <n> min>
Battery explains    <n> of <n> stops; <n> unexplained

Ceiling on every figure above
- Reporting proves powered and in coverage. It does NOT prove worn.
- Movement thresholds unvalidated; both shown; computed on filtered positions.
- Discarded <n> distances exceeding the 45 m site bound.

APPI
- Per-person data on identifiable subcontractors. Engineering-internal only.
- Not for a customer-facing artifact; not the Kajima-facing report.
- Retention and access undecided — this line stays until they are.

Not measured
- <what did not run, and why>
```

---

## 8. BEFORE THIS LEAVES ENGINEERING

The health check produces device facts. **This one produces statements about named people**, which is a different risk.

- **Never conclude a tag was unworn from one day**, or from a movement fraction alone. A bench-tag claim needs the reading on **≥ 3 working days at ≥ 80 % coverage**, plus `r95` at the noise floor. Noise does not repeat in the same direction; a bench tag does.
- **A platform defect produces the same shape.** Tags can appear onsite in Site Presence with no historical trail — indistinguishable from a bench tag in this view — and the "No data" message is inaccurate.
- **Battery-explained stops are not discipline findings.** Say which are which, explicitly.
- **Never escalate on anything in this report.** It goes to a human who decides what, if anything, reaches the customer. Cadence and delivery belong to the Agent Scheduler task that invokes this command, not to this file.

> Telling Kajima a worker left their tag on a bench when the platform invented the record, or when the battery simply died, is a serious and personal accusation to get wrong.

---

## 9. BASELINE — write to `.koto-tag-usage-state.json` if absent

Observed from ZPS **2026-09-08 09:25 PT / 2026-09-09 01:25 JST**. Last-activity only; movement did not run. Times JST. Use to compare rather than starting blind.

```json
{
  "observed": "2026-09-09T01:25:00+09:00",
  "source": "zps-browser-last-activity-only",
  "working_hours": "Mon-Fri 08:00-18:00 JST",
  "tags": {
    "kps_c79b":                 {"worker": "C04", "state": "STOPPED",    "last_fix": "2026-08-21T11:36:45+09:00", "battery": null},
    "B2_kps_44f8":              {"worker": "C08", "state": "STOPPED",    "last_fix": "2026-08-21T16:20:32+09:00", "battery": null},
    "kps_df91":                 {"worker": "C07", "state": "STOPPED",    "last_fix": "2026-08-21T16:21:05+09:00", "battery": null},
    "kps_289a":                 {"worker": "C01", "state": "STOPPED",    "last_fix": "2026-08-24T16:23:28+09:00", "battery": "low 2026-08-24T13:12+09:00"},
    "kps_4cdd":                 {"worker": "C02", "state": "STOPPED",    "last_fix": "2026-08-26T16:02:58+09:00", "battery": "dead 2026-09-01T02:27+09:00 (6d AFTER stop — odd ordering)"},
    "kps_cf91":                 {"worker": "C03", "state": "STOPPED",    "last_fix": "2026-08-27T09:57:32+09:00", "battery": null},
    "kps_2c6d":                 {"worker": "C10", "state": "STOPPED",    "last_fix": "2026-08-27T15:14:28+09:00", "battery": null},
    "B3_kps_1a93":              {"worker": "C09", "state": "STOPPED",    "last_fix": "2026-08-31T16:20:51+09:00", "battery": "dead 2026-09-01T11:04+09:00"},
    "B1_kps_d38f":              {"worker": "C06", "state": "STOPPED",    "last_fix": "2026-09-02T16:18:20+09:00", "battery": null},
    "A5_kps_ae92":              {"worker": "C05", "state": "NEVER_USED", "last_fix": "2026-06-23T14:51:35+09:00", "battery": null},
    "KotoCrane_#1_21ee_Magnet": {"asset": "KotoCrane_#1", "state": "IN_USE", "last_fix": "2026-09-08T17:57:55+09:00", "battery": null},
    "KotoCrane_#5_3d9a_Magnet": {"asset": "KotoCrane_#5", "state": "IN_USE", "last_fix": "2026-09-08T18:00:00+09:00", "battery": null}
  },
  "findings_2026_09_08": {
    "summary": "All 10 worker tags STOPPED. 9 were in service ~17-31 Aug then decayed to zero over 12 days. C05 never used.",
    "cascade": ["2026-08-21: C04, C07, C08", "2026-08-24: C01", "2026-08-26: C02", "2026-08-27: C03, C10", "2026-08-31: C09", "2026-09-02: C06"],
    "stop_time_cluster": "6 of 9 stops fell between 16:02 and 16:23 JST across 5 different days (C08, C07, C01, C02, C09, C06). Matches the documented 16:00-16:45 JST modem signature; end of shift produces an identical shape and this data cannot separate them.",
    "equipment_control": "On 2026-09-02 and 2026-09-08 both crane tags reported to 17:57-18:00 JST. C06 stopped 16:18 on 2026-09-02 — cloud path up ~100 min after it went silent.",
    "battery": "Explains C01 (low 3h before stop) and C09 (dead 19h after stop). C02 odd ordering. 7 tags have no battery alert — unexplained.",
    "infrastructure": "Not a reader or hub problem — 29 readers, no offline alerts, red-zone events fired normally through 2026-09-08.",
    "blocking_question": "Is Subcon C still on site? On-site activity on 2026-09-08 came from non-monitored TP_* tags and a named ZaiNar person. Crew showed 0 onsite. Demobilised (scope is stale) vs still working untracked — opposite urgencies, not separable from this data."
  }
}
```
