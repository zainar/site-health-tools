---
description: Koto Crane tag health check (zlp-prd-jpn). Read-only. Grades device health and activity on two separate axes for the 12 monitored tags — Subcon C01–C10, Crane #1, Crane #5.
argument-hint: "[check | triage | weekly] [all | subcon-c | cranes]  (default: check all)"
allowed-tools: mcp__zlp-prd-jpn__resolver_status, mcp__zlp-prd-jpn__who_am_i, mcp__zlp-prd-jpn__list_assets, mcp__zlp-prd-jpn__get_tag_by_asset, mcp__zlp-prd-jpn__list_assets_with_tags, mcp__zlp-prd-jpn__list_trackers, mcp__zlp-prd-jpn__list_offline_tags, mcp__zlp-prd-jpn__list_continuous_mode_tags, mcp__zlp-prd-jpn__get_continuous_tag_health, mcp__zlp-prd-jpn__get_asset_history, mcp__zlp-prd-jpn__get_tag_location_history, mcp__zlp-prd-jpn__get_tag_status, mcp__zlp-prd-jpn__get_tag_battery, mcp__zlp-prd-jpn__get_tag_fuel_gauge, mcp__zlp-prd-jpn__get_tag_events, mcp__zlp-prd-jpn__get_tag_config, mcp__zlp-prd-jpn__check_tag_hub_connectivity, Read, Write, Bash(date:*), Bash(TZ=*)
---

# Koto Crane — tag health check — `$ARGUMENTS`

**Site**: `Koto_Pumping_Station_Crane` · `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`
**Account**: `Kajima-Koto` · **Env**: `zlp-prd-jpn` (WIFI_RT)

This file is **self-contained**. It needs no other document to run.

> **This is a safety system.** Red-zone and proximity alerts warn workers standing under crane loads. A silent tag is not a data-quality issue.

---

## INSTALL — handled by the plugin

This command ships in the **kajima-crane-site-survey** plugin. Installing the plugin gives you
`/koto-tag-check` in every project — there is nothing to copy by hand.

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
---

## 0. PREFLIGHT — fail loud, never fall back

Run in order. **Do not proceed past a failure and do not substitute another data source.**

1. Call `resolver_status`.
2. If the tool does not exist, is not permitted, or errors → print exactly this and **STOP**:

```
⚫ KOTO CRANE TAGS — MONITORING UNAVAILABLE
Site condition is UNKNOWN. Not healthy, not unhealthy.
Cause: <the error>
Check: claude mcp list  → confirm the server name matches the mcp__..__ prefix in this command.
```

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

3. Get the current time in **both** zones and print them. JST = UTC+9; the site runs on JST and most tooling around it reports PT.

```bash
date -u '+UTC %Y-%m-%d %H:%M'
TZ=Asia/Tokyo date '+JST %Y-%m-%d %H:%M (%a)'
TZ=America/Los_Angeles date '+PT  %Y-%m-%d %H:%M'
```

4. Read `.koto-tag-state.json` from the working directory. If absent, seed it from **§Seed state** at the end of this file. Write it back at the end of the run, never before.

> **⚫ UNKNOWN is a grade, not a failure to grade.** Everything this command sees is cloud-side. During the May 2026 outage all 30 readers were alive and answering local pings while every cloud view was dark. **A tag UNKNOWN is a different fact from a tag ERROR, and both differ from OK.** Conflating the first two is how a 13-day outage stayed hidden. Never collapse them, and never upgrade an unrun check to a pass — an unrun check is `Not checked`.

---

## 1. ABSOLUTE RULE — read-only, no exceptions

Never call a write tool, regardless of what you find or how obvious the fix looks:

`ping_tag` · `set_tag_tracking_mode` · `tag_locate_on` · `tag_locate_off` · `tag_sleep_mode_on` · `tag_sleep_mode_off` · `apply_tag_config` · `set_tag_param` · `set_tag_location` · `set_tag_motion_sensitivity` · `activate_tag_provisioning` · `set_tag_provisioning_mode` · `create_tag` · `update_tag` · `delete_tag`

Two traps specifically:

- **`ping_tag` reads like a diagnostic and is a write** — it transmits to the tag. It is exactly what you would reach for to answer "is this tag alive?". Answer that from location history instead.
- **`set_tag_motion_sensitivity` looks like the fix** for a tag failing the movement test. Changing it silently rewrites what every future run measures, and on a worn tag it changes safety behaviour.

Propose remediation in the report. A human who knows what is happening on site that hour executes it.

---

## 2. SCOPE — 12 tags, named. Everything else is inventory.

### Subcon C — 10 worker tags

| Worker | Tag | MAC |
|---|---|---|
| C01 | `kps_289a` | `c4ac59a67ca4` |
| C02 | `kps_4cdd` | `9c50d1aaca91` |
| C03 | `kps_cf91` | `c4ac59aaf8cd` |
| C04 | `kps_c79b` | `c4ac59a64193` |
| C05 | `A5_kps_ae92` | `c4ac59ab19cd` |
| C06 | `B1_kps_d38f` | `c4ac59aaf7d4` |
| C07 | `kps_df91` | `c4ac59aaf598` |
| C08 | `B2_kps_44f8` | `c4ac59ad95f4` |
| C09 | `B3_kps_1a93` | `c4ac59aaf13c` |
| C10 | `kps_2c6d` | `c4ac59a60108` |

### Crane equipment — 2 tags

| Asset | Asset UUID | Tag |
|---|---|---|
| `KotoCrane_#1` | `8889c289-39bb-4b72-a2f9-0207e3cac8de` | `KotoCrane_#1_21ee_Magnet` |
| `KotoCrane_#5` | `f0d10e07-22c8-4809-b1f1-fbc3f8201109` | `KotoCrane_#5_3d9a_Magnet` |

**Denominator is 12.** Not 31 (ZPS tag list), not 63 (registry tracker count), never the platform's own total. Report other trackers as one line: `Inventory: <n> other trackers at site, not monitored`.

**Resolve tags by asset UUID via `get_tag_by_asset` — never by name prefix.** Prefixes mismatch the wearer for C05 (`A5_`), C06 (`B1_`), C08 (`B2_`) and C09 (`B3_`) — four of ten. Matching on prefix mis-attributes nearly half the set, and a mis-attributed compliance finding is worse than no finding.

**Out of scope:** anchor tags fixed to readers (a fixed tag that moves is a calibration finding), `KotoCrane_#2_02f9_Magnet` (exists, silent since 23/06/2026), and every other tracker.

---

## 3. THE THREE CORRECTIONS — read before trusting any number

### 3.1 `get_tag_status().last_heartbeat` must not grade anything

It returns formatted strings (`"5.4h ago"`) and shows stale values **even when the tag is actively reporting**. The vendor tool reference says outright: *"Do NOT use for health checks."*

> **Liveness comes from the newest `locationTime`** in `get_asset_history` / `get_tag_location_history`. `get_tag_status` is still called — for battery, RSSI and flags — but its heartbeat field is not read.

A tag graded OFFLINE from `last_heartbeat` may be reporting normally. That error looks like diligence and reads as a fault report about a worker.

### 3.2 Movement — aggregate before you threshold

Positioning here is **±1–2 m** in obstructed areas. A 500 mm movement threshold on raw fixes is below that floor, so a tag on a bench clears it in most intervals and **the test meant to detect an unworn tag certifies it as worn.** Raising the threshold to 2 m just inverts the error — a worker pacing a 1.5 m control platform then reads as a bench tag. **No threshold on raw per-fix displacement separates them.**

Noise is roughly zero-mean and uncorrelated between samples; movement is not. So average first — noise shrinks, movement does not:

> **Split each 15-minute interval into 3-minute sub-windows. Take the component-wise median of each. Measure between medians, never between raw fixes.**

A median over *n* fixes cuts noise by ~√*n*. At 3 s cadence (~60 fixes/sub-window) a ±1–2 m floor becomes ~0.2 m. At 60 s cadence it is ~3 fixes and the gain is small — so **cadence must be reported beside every movement figure, and two tags at different cadences are not on the same scale.**

Also: **do not take a max over raw fixes.** A maximum over dozens of noisy samples selects the largest noise excursion and inflates systematically, in the same direction as the threshold error.

### 3.3 Sanity-bound every distance — the site is 27.1 × 34.9 m

Largest possible displacement is about **45 m**. A prior version of this check reported "moved 1.2km" in 15 minutes and "15.3km over 10 hours" — the site diagonal 340 times.

> **Discard any distance > 45 000 mm as a positioning error. Count the discards and report the count.** A run discarding many is reporting on the positioning system, not on a worker.

**Report metres, never kilometres.** On a 27 m site a figure in km is always wrong.

---

## 4. TWO AXES, NEVER MERGED

**Axis A — device health.** Is the tag working? Reporting, powered, locating.
**Axis B — activity.** Is it moving? Worn, or in use.

Every combination occurs:

| | Moving | Not moving |
|---|---|---|
| **Reporting** | worker at work · crane in use | **bench-tag candidate** (worker) · parked crane (normal) |
| **Not reporting** | impossible — investigate the data | dead tag, dead battery, or off site |

> **Axis A sets the grade. Axis B is reported, never escalated, and never called "unhealthy."** A day where all 12 reported perfectly and nobody moved is a green site with a compliance observation — and that observation goes to a human, not a channel.

### Axis A grades

| Grade | Condition |
|---|---|
| 🟢 **OK** | Coverage ≥ 95 % of expected working intervals · longest gap < 15 min · battery not critical · no error flags |
| 🟡 **WATCH** | Coverage 80–95 % · gap 15–30 min · battery critical or projected to die inside the shift · any `0x0400`/`0x0100` flag · mode contradiction |
| 🟠 **ERROR** | Coverage < 80 % during working hours · gap > 30 min · no fix > 60 min during working hours |
| ⚫ **STALE** | No location fix in 24 h |
| ⚫ **NEVER SEEN** | No location fix in 7 days |
| ⚫ **UNKNOWN** | Could not observe — API down, tag unresolvable, mode undetermined |

### Axis B readings

| Reading | Worker tag | Crane tag |
|---|---|---|
| Movement fraction ≥ 60 % | ACTIVE | in use |
| 20–60 % | PARTIAL — breaks, stationary work | intermittent |
| < 20 % **and** r95 at the noise floor, on **≥ 3 working days** | **bench-tag candidate** → §7 caveats | **normal** — a parked crane is expected to be still |
| coverage < 80 % | **NOT EVALUATED** | NOT EVALUATED |

A crane tag never produces a bench-tag finding. For cranes report movement **events** with times — that is utilisation, and it answers "who left crane #2 loaded".

---

## 5. STEP 1 — resolve the set and read the mode

1. `list_offline_tags(site=<uuid>)` — one call, the whole offline set. Any of the 12 here is an ERROR candidate; confirm in step 2 before grading.
2. `list_continuous_mode_tags(site=<uuid>)` — **read the mode, do not assume it.**
3. `get_tag_by_asset(asset_res_name=<uuid>)` per monitored asset → tag name + node ID.
4. Compare against §2. **Report any addition, disappearance or remapping by name.** Never silently substitute.

**Thresholds depend on mode:**

| Mode | Expected | Warn | Error |
|---|---|---|---|
| `continuous` | more frequent | > 5 min | > 15 min |
| `on_motion` | ~5 min | > 15 min | > 60 min |

If `list_continuous_mode_tags` is unreachable, fall back to **assumed-continuous inside working hours**, label every grade `mode: assumed`, and treat a tag reporting steadily at ~5-minute intervals through working hours as a **mode contradiction → ⚫ UNKNOWN for that tag**, not an ERROR.

**Scheduler:** `23:00 UTC` = 08:00 JST → continuous ON. `09:00 UTC` = 18:00 JST → motion/OFF. Skipped on JST weekends. **Exclude ±30 min around each switch from all metrics** — tags routinely report a disconnect at the switch and reconnect ~30 min later, and that artifact otherwise shows as a daily flap.

---

## 6. STEP 2 — Axis A: coverage, reporting, battery

`get_asset_history(asset_res_name=<uuid>, range="last_24h")` per tag.

> **Test one tag first.** ~1000–2000 fixes per tag per 24 h; 12 tags is up to 24 000 points and may exceed the tool-result cap. If too large, use shorter windows and aggregate. **Do not silently truncate, and do not sample** — at 12 tags completeness is affordable and it is the entire point.

> **`get_asset_history` has no pagination and no filters** — it returns the whole range in one response (confirmed in the `get_asset_event_history` spec's comparison table, 2026-09-02). There is no cursor to fall back on, so **shortening the window is the only mitigation** if a pull is too large. Budget for that before the first full-fleet run.


**Working-hours filter:** JST = UTC+9, keep **Mon–Fri 08:00 ≤ hour < 18:00**, 40 intervals of 15 min.

**Settled 2026-09-16: 08:00–18:00 JST, Mon–Fri is the single window across every check in this package.** Weekends are outside it and are not graded; weekend activity is reported as an out-of-hours line rather than filtered away.

> ### Why Saturday is excluded, and what to confirm before adding it
>
> **This is a deferral, not a judgement about whether crews work Saturdays** — they commonly do on Japanese construction sites. It is here because *this file* is where adding Saturday would go wrong.
>
> **The mode scheduler is documented as "skipped on JST weekends."** It puts the fleet into `continuous` at 08:00 JST and `motion/OFF` at 18:00 JST on weekdays. If "weekends" includes Saturday — and the phrase conventionally does — then on Saturday the tags sit in `on_motion`, whose tolerances are **4× looser**: warn > 15 min and error > 60 min, against continuous's 5 and 15.
>
> **Grading a Saturday against the continuous tier would manufacture ERRORs on a perfectly healthy fleet, every Saturday.** That is the cry-wolf failure this package exists to avoid, and it would arrive weekly.
>
> **To add Saturday back:** confirm with whoever owns `koto-crane-daily.md` §11 whether "weekends" means Sat+Sun or Sun only. Then either (a) if the scheduler *does* run Saturdays, change `weekday() > 4` to `> 5` here and update the window text across the package — nothing else is needed; or (b) if it does not, Saturday additionally needs the on_motion tier applied to it, and `list_continuous_mode_tags` must be mandatory rather than a fallback on Saturday runs.
>
> **Until then, Saturday activity is reported, never graded.** A Saturday with thin coverage is not a finding and must not be reported as one.

```python
from datetime import datetime, timedelta, timezone
JST = timezone(timedelta(hours=9))

def to_jst(loc):
    return datetime.fromisoformat(loc['locationTime'].replace('Z', '+00:00')).astimezone(JST)

def interval_bucket(t):                 # 0..39, or None outside working hours
    if t.weekday() > 4 or not (8 <= t.hour < 18):   # Mon=0 .. Fri=4; weekends excluded
        return None                                  # -> '> 5' adds Saturday, see the note in §5
    return (t.hour - 8) * 4 + t.minute // 15        # 40 intervals of 15 min
```

Per tag compute **coverage** (intervals with ≥1 fix ÷ 40), **longest gap** (max minutes between consecutive fixes in working hours), **newest fix age**, and **flap count** (silence→report transitions, excluding switch windows). Grade against §4.

**Report the longest gap next to coverage, always.** 96 % with one 35-minute hole during a lift is worse than 96 % of five-second blips.

**Exclusions before computing:** the ±30 min switch windows, and intervals with no live reader in the tag's area (no coverage → *not evaluated*, not failed).

**Correlate before listing.** One tag at ERROR is a tag finding. **Three or more together is one site finding** — check readers and hub before reporting twelve separate faults.

### Battery

`get_tag_status` (battery, RSSI, flags) and `get_tag_battery` (time-to-empty, voltage). `get_tag_fuel_gauge` only for already-flagged tags.

| State | Rule |
|---|---|
| Low | 10–29 % and discharging |
| Critical | < 10 % and still reporting |
| Dead | ≤ 5 % and silent 15+ min |

**Use time-to-empty against the remaining shift.** A tag entering the shift at 12 % dies around lunch — actionable at 08:00, useless at 16:00. **WATCH any tag projected to die inside the shift, before it does.** This is the only predictive signal here.

**Decode the flags bitmask and name these explicitly:** `0x0400 hub_not_found` (the tag saying it could not reach its hub) and `0x0100 ranging_error`. Any tag raising either = WATCH. Multiple distinct tags, or a count rising vs last run = ERROR, and it becomes a **hub** finding, not a tag finding.

**Deduplicate by tag before any count.** The alert stream is heavily duplicated — one tag has ~19 identical battery-low entries at a single timestamp. Report distinct tags, raw count in parentheses.

---

## 7. STEP 3 — Axis B: movement

Only for tags with coverage ≥ 80 %. Below that: **NOT EVALUATED**. Do not print a fraction.

```python
import math, statistics, itertools

SITE_DIAGONAL_MM = 45000        # 27.1 m × 34.9 m surveyed extent — hard sanity bound
THRESHOLDS_MM    = (500, 2000)  # dual-threshold on FILTERED positions, until the floor is measured
SUBWINDOW_S      = 180          # 3 min
MIN_FIXES        = 3            # per sub-window; below this the median is not trustworthy

def dist_mm(a, b):
    return math.sqrt((b[0]-a[0])**2 + (b[1]-a[1])**2 + (b[2]-a[2])**2)

def settled_positions(fixes, t0):
    """Median-filter time-ordered fixes (each with .t seconds) into per-sub-window positions."""
    out = []
    for _, grp in itertools.groupby(fixes, key=lambda f: int((f['t'] - t0) // SUBWINDOW_S)):
        g = list(grp)
        if len(g) < MIN_FIXES:
            continue                                    # dropped and counted, NOT stationary
        out.append(tuple(statistics.median(f[k] for f in g) for k in ('x', 'y', 'z')))
    return out

def interval_movement(fixes, t0):
    """(movement_mm, n_settled, n_discarded), or None when there is not enough data."""
    pts = settled_positions(fixes, t0)
    if len(pts) < 2:
        return None                                     # NO_DATA — never 0.0
    ds = [dist_mm(a, b) for a, b in itertools.combinations(pts, 2)]
    ok = [d for d in ds if d <= SITE_DIAGONAL_MM]
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

> **`None` and `0.0` must never be conflated.** `None` = could not evaluate. `0.0` = measurably still. Collapsing them turns a coverage failure into a statement about a worker.

**Movement fraction = intervals MOVED ÷ intervals with data**, reported at 500 mm and 2000 mm side by side. The gap is the diagnostic: `500mm: 94% / 2000mm: 11%` means noise dominates and neither number describes the worker.

**Denominator is the tag's own presence window** — first to last fix of the day, which tracks the real shift within ~2 minutes. Intervals outside it are not counted and not failed.

**`r95` is the primary bench-tag signal**, not the movement fraction. A bench tag stays inside a noise-sized circle all day however much its path wanders; a worn tag crosses the site. Report `r95`, `path_m` and `straightness` on every worker row, and **name all ten workers individually every run** — that format is what surfaced C05.

### Caveats — mandatory wherever a bench-tag candidate appears

- **One day is never enough.** Requires ≥ 3 working days at ≥ 80 % coverage. Noise does not repeat in the same direction; a bench tag does. This is the guard that prevents a wrong accusation, and it is not optional.
- **The threshold is unvalidated.** Until the floor is measured (§9), a candidate is a candidate.
- **Platform defect:** tags can appear onsite in Site Presence with no historical trail — an artifact **indistinguishable from a bench tag in this view** — and the "No data" message is inaccurate.
- **APPI:** per-person movement data on identifiable subcontractors. **Engineering-internal only.** Not for a customer-facing artifact and not the Kajima-facing report. **Retention and access are undecided — say so in every report until they are.**

---

## 8. TRIAGE · WEEKLY · SITE GRADE

**Triage** (`triage` mode, or anything above WATCH): `get_tag_events` · `get_tag_config` (a low motion-sensitivity setting explains a low movement fraction with no worker behaviour involved) · `check_tag_hub_connectivity` · `get_tag_fuel_gauge`.

**Before blaming a tag, check whether it is the site.** If ≥ 3 tags flag together, verify readers and hub for the same window. Tags cannot produce fixes without readers, and a dark site presents as twelve dead tags.

**Weekly:** `list_assets_with_tags` (mapping still correct after crew changes — report any tag↔worker change) · `list_trackers` (inventory count) · compare coverage and both movement fractions against last week in state. **A tag trending down over three weeks is the pattern; one bad day is not.**

**Cohort-decay check:** convert every stale tag's last-ever activity to JST and look for clustering in **16:00–16:45 JST** across different days — the documented cellular-modem signature. **State the competing explanation in the same breath:** end of shift produces the identical shape. This data cannot separate them.

### Site grade

| Level | Condition |
|---|---|
| ⚫ **UNKNOWN** | Monitoring down, or the set could not be resolved |
| 🔴 **P2** | Zero location fixes from any of the 12 during working hours |
| 🟠 **P3** | ≥ 3 of the 10 worker tags at ERROR across a working day **with the crew on site** · ≥ 3 tags at ERROR together · cohort-decay cluster confirmed across multiple days |
| 🟡 **WATCH** | Any battery critical or projected to die inside the shift · any `0x0400`/`0x0100` · 1–2 tags at ERROR · mapping reconciliation > 14 days old · mode undetermined · a crane with no tag |
| 🟢 **GREEN** | All 12 resolved and reporting to mode through the working day, no battery or flag findings |

**≥ 3 of 10 worker tags is 30 % of the monitored crew**, not 10 % of a roster. The absolute number deliberately matches the register's reader threshold — do not soften it into a percentage to make it look smaller.

**Movement never sets the site grade.**

### Alerting and cadence — not this command's job

**This command grades and reports. It does not decide when to run, who to tell, or what escalates.**
Cadence, delivery and any confirm-before-paging ladder belong to the Agent Scheduler task that
invokes it.

Two routing rules this command still owes whatever does escalate, because both are analysis rather
than scheduling:

- **Never escalate on Axis B.** A movement finding is a compliance observation about a named person;
  a human decides whether it reaches the customer. Say so in the report so nothing downstream treats
  it as an engineering signal.
- **Say when this is not the monitor to act on.** If readers and hub are simultaneously red, twelve
  dead tags are one site failure — the report must name that, so the infra finding gets actioned
  rather than this one.

**Never put keys or tokens in any output.**
---

## 9. OPEN — carry these in the report until closed

1. **Is Subcon C still on site?** As of 2026-09-08 all ten worker tags were dark 6–18 days while the crew showed 0 onsite and the red-zone events came from non-monitored `TP_*` tags and a named ZaiNar person. **Two readings with opposite urgency:** the crew demobilised and this scope is stale, or ten workers are on a live crane site untracked. **This data cannot separate them — one question to a human can.** Until answered, do not grade above or below P3.
2. **Measure the movement floor.** Run a known-stationary tag (parked crane, or any tag 20:00–06:00 JST) and a known-active tag through **the same median-filter pipeline**; floor = 99th percentile of consecutive settled-position distance. **Report the separation between the two distributions — if they overlap, the method does not work here and no threshold rescues it.** Collapses dual-threshold to one number.
3. **Cranes #2/#3/#4 are untagged** — intended, or a coverage gap? `#2`'s magnet tag has been dead since 23/06/2026.
4. **APPI retention and access** for the per-person output.
5. **31 tags in ZPS vs 63 trackers in the registry** — unreconciled. Hygiene only; not a denominator.
6. **Does the mode scheduler's "skipped on JST weekends" include Saturday?** Saturday is excluded from grading for now precisely because this is unanswered — it decides which threshold tier a Saturday would be graded against. One question to whoever owns `koto-crane-daily.md` §11 closes it and unblocks adding Saturday across the package. See §5.

---

## 10. REPORT

```
KOTO CRANE TAGS — <emoji + level> — <YYYY-MM-DD HH:MM JST / HH:MM PT>

One line: what is true right now.

Monitored   <n>/12 resolved      Mode: <read | assumed-continuous>
Axis A      OK <n> · WATCH <n> · ERROR <n> · STALE <n> · NEVER SEEN <n> · UNKNOWN <n>
Readers/hub <ok | degraded — see that monitor>
Inventory   <n> other trackers at site, not monitored
Mapping     reconciled <date> (<n> days ago)

=== SUBCON C — 10 workers, named individually ===
C01  kps_289a   A: <grade>  cov <pct>%  gap <n>m  batt <pct>%/<h>h  cadence <n>s
                B: move <500mm>/<2000mm>  r95 <n>m  path <n>m  straight <n.nn>  days <n>/3
...

=== CRANE EQUIPMENT — 2 ===
#1  KotoCrane_#1_21ee_Magnet   A: <grade>  cov <pct>%   B: movement events <times>
#5  KotoCrane_#5_3d9a_Magnet   A: <grade>  cov <pct>%   B: movement events <times>

Findings
- <finding, with the number that triggered it>

Movement caveats
- Thresholds unvalidated; both figures shown; filtered positions.
- Bench-tag candidates require ≥3 working days. Platform artifact is indistinguishable in this view.
- APPI: per-person data, engineering-internal. Retention and access undecided.

Open items carried
- <from §9>

Suspected cause
- <only if evidence supports it — otherwise "insufficient evidence from cloud-side data">

Recommended actions (for a human)
- <action> — <who>

Not checked
- <what did not run, and why>

Discards    <n> distances exceeded the 45 m site bound
```

### Say these every run

1. **Which mode was graded against**, and whether it was read or assumed.
2. **Both movement thresholds**, that both are on filtered positions, and that neither is validated.
3. **That everything here is cloud-side.** A dark tag means "not reaching the cloud", never "the tag is dead". Separating those needs someone at the site.
4. **The APPI status** of any per-person figure.
5. **Anything not checked, and why.**

---

## Seed state — write to `.koto-tag-state.json` if absent

Observed from ZPS on **2026-09-08 09:25 PT / 2026-09-09 01:25 JST**. Axis A only; movement did not run. Use as the baseline for the first comparison.

```json
{
  "last_run": "2026-09-09T01:25:00+09:00",
  "source": "zps-browser-partial",
  "window": "last_24h",
  "unknown_runs": 0,
  "mode_source": "not_read",
  "d_mm": null,
  "mapping_reconciled": "2026-09-02",
  "tags": {
    "kps_289a":                 {"asset": "Subcon, C01", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-08-24T16:23:28+09:00", "note": "battery low alert 2026-08-24 13:12 JST"},
    "kps_4cdd":                 {"asset": "Subcon, C02", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-08-26T16:02:58+09:00", "note": "battery DEAD alert 2026-09-01 02:27 JST"},
    "kps_cf91":                 {"asset": "Subcon, C03", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-08-27T09:57:32+09:00", "note": null},
    "kps_c79b":                 {"asset": "Subcon, C04", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-08-21T11:36:45+09:00", "note": null},
    "A5_kps_ae92":              {"asset": "Subcon, C05", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-06-22T14:51:35+09:00", "note": "never tracked — standing item"},
    "B1_kps_d38f":              {"asset": "Subcon, C06", "axis_a": "STALE",       "newest_fix": "2026-09-02T16:18:20+09:00", "note": "crosses to NEVER_SEEN 2026-09-09"},
    "kps_df91":                 {"asset": "Subcon, C07", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-08-21T16:21:05+09:00", "note": null},
    "B2_kps_44f8":              {"asset": "Subcon, C08", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-08-21T16:20:32+09:00", "note": null},
    "B3_kps_1a93":              {"asset": "Subcon, C09", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-08-31T16:20:51+09:00", "note": "battery DEAD alert 2026-09-01 11:04 JST"},
    "kps_2c6d":                 {"asset": "Subcon, C10", "axis_a": "NEVER_SEEN",  "newest_fix": "2026-08-27T15:14:28+09:00", "note": null},
    "KotoCrane_#1_21ee_Magnet": {"asset": "KotoCrane_#1", "axis_a": "OK",         "newest_fix": "2026-09-08T17:57:55+09:00", "note": "reported to the 18:00 JST switch"},
    "KotoCrane_#5_3d9a_Magnet": {"asset": "KotoCrane_#5", "axis_a": "OK",         "newest_fix": "2026-09-08T18:00:00+09:00", "note": "reported to the 18:00 JST switch"}
  },
  "summary": {"monitored": 12, "ok": 2, "watch": 0, "error": 0, "stale": 1, "never_seen": 9, "unknown": 0},
  "context_2026_09_08": {
    "site_grade": "P3",
    "infrastructure": "healthy — 29 readers, no reader-offline alerts, red-zone events fired 12:46/14:20/15:18 JST on 08 Sep",
    "blocking_question": "Is Subcon C still on site? Crew showed 0 onsite; on-site activity came from non-monitored TP_* tags and a named ZaiNar person."
  }
}
```
