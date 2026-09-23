---
description: Koto Crane LOADED/UNLOADED button usage (Mixpanel prod + ZLP movement cross-check). Read-only. Answers "were the crane load buttons pressed, by whom, when, how often, for how long — and was the crane actually moving".
argument-hint: "[last_24h | last_7d | last_30d | YYYY-MM-DD:YYYY-MM-DD] [all | crane1 | crane5]  (default: last_7d all)"
allowed-tools: mcp__Mixpanel__Get-Projects, mcp__Mixpanel__Get-Events, mcp__Mixpanel__Get-Property-Values, mcp__Mixpanel__Run-Query, mcp__zlp-prd-jpn__resolver_status, mcp__zlp-prd-jpn__who_am_i, mcp__zlp-prd-jpn__list_assets, mcp__zlp-prd-jpn__get_tag_by_asset, mcp__zlp-prd-jpn__get_asset_history, Read, Write, Bash(date:*), Bash(TZ=*)
---

# Koto Crane — load-button usage — `$ARGUMENTS`

**Site**: `Koto_Pumping_Station_Crane` · `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`
**Account**: `Kajima-Koto` · `a6d42714-0493-4186-b2eb-4f97598269e8` · env `zlp-prd-jpn`
**Source A**: Mixpanel project **`3369288`** (`Zainar`, prod) · event **`crane_status_change`**
**Source B**: ZLP crane magnet tags — **required**, see §6
**App**: `https://zps.jp.zainartech.com` (ZPS v4.0.11) — **not** `zps.zainartech.com`, a
decommissioned host that stopped receiving deploys in Feb 2026 and still serves a working-looking
app.

Companion to `koto-tag-usage.md`, `koto-reader-check.md` and `koto-hub-check.md`. **This one answers "did anyone press the
buttons, and was the crane actually working".**

---

## THE THREE QUESTIONS — do not blur them

| Question | Answered by | Evidence |
|---|---|---|
| **Was the crane moving?** | §6 of this file (ZLP) | location fixes on the crane magnet tags |
| **Did anyone record a load?** | §5 of this file (Mixpanel) | `crane_status_change` events |
| **Was the crane actually carrying a load?** | **nothing, reliably** | nobody instruments the hook |

> **This event records BUTTON PRESSES, not crane state.** If crews do not press, the report reads
> zero however much lifting happened. **A zero here is not an idle crane.** Every figure carries
> that ceiling and the report must print it.

---

## INSTALL — handled by the plugin, except for Mixpanel

This command ships in the **kajima-crane-site-survey** plugin and gives you `/koto-crane-usage`.

**It is the one command that spans two MCP servers, and the plugin only supplies one of them.**
The plugin registers `zlp-prd-jpn`. **Mixpanel you must connect yourself** — it is an OAuth
connector, not something a plugin can register with an API key.

Both prefixes have to match exactly or the `allowed-tools` list in the frontmatter **silently fails
to match** and the command looks broken for no visible reason:

```bash
claude mcp list   # expect: zlp-prd-jpn   AND   Mixpanel (capital M)
```

If your Mixpanel server is registered under a different name, rename it rather than editing prefixes
here.

**Read-only on both sides.** Issue the ZLP key scoped `viewer` / `engineering:read`; the usual
credential carries write+admin on production.

---

## 0. PREFLIGHT — fail loud, on both sources

1. `Get-Projects`. Confirm `3369288` exists.
2. **`resolver_status` on `zlp-prd-jpn`. The movement cross-check is REQUIRED, not optional.**

On failure of **either**, print and **STOP**:

```
⚫ KOTO CRANE LOAD USAGE — UNAVAILABLE
Usage is UNKNOWN for this range. Not "no usage".
Failed source: <Mixpanel | ZLP>   Cause: <error>
Check: claude mcp list → confirm both server names match the mcp__..__ prefixes.
```

> **Why ZLP is a hard requirement rather than a nice-to-have.** Without it, a zero-press report is
> uninterpretable: "nobody pressed the buttons" and "the crane did not move" produce an identical
> row, and they carry opposite actions — one is a training conversation, the other is a site issue.
> **Better to fail loudly than to publish an unreadable zero.**

3. **Re-query before reporting any zero.** Observed 2026-09-16: a `Get-Property-Values` call with
   `from_date=2026-09-16, to_date=2026-09-17` returned **empty**, while the same query widened to
   `from_date=2026-09-14` returned the rows — and the narrow window had returned data minutes
   earlier. **A single empty result is not evidence of zero usage.** On any empty result, re-run
   with the window widened by ±2 days. If the wide query is also empty, report zero; if the two
   disagree, report ⚫ UNKNOWN and say the query was unstable.

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

4. Print both clocks. **Mixpanel timestamps are UTC.**

```bash
date -u '+UTC %Y-%m-%d %H:%M'
TZ=Asia/Tokyo date '+JST %Y-%m-%d %H:%M (%a)'
```

5. Parse `$ARGUMENTS`. Default `last_7d all`. Explicit ranges `YYYY-MM-DD:YYYY-MM-DD`, in **JST**,
   converted to UTC for the Mixpanel query.

6. Read `.koto-crane-usage-state.json`; seed from §8 if absent. Write at the end, never before.

**Never call a Mixpanel write tool** (`Create-*`, `Update-*`, `Delete-*`, `Edit-*`, `Bulk-Edit-*`).
On ZLP, **`ping_tag` is a WRITE** — it transmits to the tag — as are `set_tag_*`,
`apply_tag_config`, `tag_locate_*`, `reboot_node`.

---

## 1. TIMEZONE — the part that silently produces wrong answers

**Mixpanel project timezone is UTC.** Confirmed 2026-09-10 by a controlled press at 12:15 PT
stamped 19:16.

JST = UTC+9, **fixed — Japan observes no DST**. So:

- Working window **08:00–18:00 JST** = **23:00–09:00 UTC**, which **crosses midnight UTC**.
- **Mon–Fri JST** begins **Sunday 23:00 UTC** and ends **Friday 09:00 UTC**.

**Convert every event timestamp to JST before bucketing.** Do not filter on UTC hour and call it JST
working hours — the day label is wrong as often as it is right.

Sanity-check against a known event: `2026-09-16T18:08:49Z` = **17 Sep 03:08 JST** — a Wednesday-lunchtime press in California files as Thursday small hours in Japan, outside working hours.

**Working hours: Mon–Fri 08:00–18:00 JST.** Settled 2026-09-16; the single window across every check
in this package. **Presses outside the window are captured and reported as a separate out-of-hours
line, never discarded** — an early start, an overtime lift or a weekend shift is worth seeing rather
than filtering away. Saturday presses in particular should be reported with their count and times,
since Saturday is excluded for now rather than on principle.

> **Settled 2026-09-16 — one window, and the earlier three-way conflict is closed.** This command
> previously used 07:00–19:00 Mon–Fri, the health checks 08:00–18:00 Mon–Sat, and earlier scheduler
> work 09:00–18:00 Mon–Sat. All three are replaced by **08:00–18:00 JST, Mon–Fri**.
>
> One consequence worth keeping in view: the window is **narrower at both ends** than this metric
> originally had, so a genuine early start or an overtime lift now falls outside working hours —
> which is why the out-of-hours line is reported rather than filtered. The day range matches this
> command's original Mon–Fri, and the usage report and the health reports now describe **the same
> week**, which they did not before. Saturday is excluded for now across the package; see
> `/koto-tag-check` §5 for what would have to be confirmed to add it.

---

## 2. SCOPE — two cranes, and only two

**Monitored set: `KotoCrane_#1` and `KotoCrane_#5`. Nothing else.**

| Crane | `crane_res_name` / ZLP asset UUID | ZLP magnet tag |
|---|---|---|
| `KotoCrane_#1` | `8889c289-39bb-4b72-a2f9-0207e3cac8de` | `KotoCrane_#1_21ee_Magnet` |
| `KotoCrane_#5` | `f0d10e07-22c8-4809-b1f1-fbc3f8201109` | `KotoCrane_#5_3d9a_Magnet` |

**`#2`, `#3` and `#4` are out of scope** — confirmed by Misha 2026-09-16. Matches the ZLP side
exactly: `koto-tag-check.md` monitors the same two assets, `KotoCrane_#2_02f9_Magnet` has been
silent since 23 June 2026, and `#3`/`#4` carry no tags. **The denominator is 2.**

Key on **`crane_res_name`**, never `crane_name`. Display names here are actively edited — five
`equipment_edit` updates against `KotoCrane_#1`–`#5` in one 30-day window — and a rename mid-range
splits one crane across two breakdown rows, so the count looks halved.

> **`crane_res_name` equals the ZLP asset UUID.** That makes §6 a direct join rather than a name
> match — use it, and never match on tag-name prefix (DEF-062).

**Out-of-scope presses are a finding, not noise.** An event with a `crane_res_name` outside the two
above gets a **one-line hygiene note** — crane, id, count, first and last press — kept out of every
count, grade and denominator. Someone pressing buttons on an unmonitored crane means the scope is
stale, and that is worth knowing the day it happens. Same discipline as the ghost-record rule in
`koto-reader-check.md`.

---

## 3. DE-DUPLICATION — do this before counting anything

**The tracker fires 1–2 events per press, intermittently, with an unreliable value.** Observed
across 7 controlled presses: events per press = **2, 2, 2, 2, 1, 1, 1**. When a duplicate fires its
value varies — `(true,false)` three times, `(true,true)` once. Dev doubled in the reverse order.

| Rule | Why |
|---|---|
| **Never count raw events** | 4 of 7 presses emitted twice |
| **Never divide by 2** | 3 of 7 emitted once — a ÷2 rule is wrong in both directions |
| **Never trust the second event's value** | sometimes the new state, sometimes its opposite |
| **Cluster, then take the first event of each cluster** | correct crane, direction and timestamp on all 7 |

```python
CLUSTER_S = 10   # seconds; events inside this window on one crane collapse to one press

def presses(events):
    """events: [{t: epoch_s, crane: res_name, loaded: bool}] sorted by t."""
    out, last = [], {}
    for e in events:
        prev = last.get(e['crane'])
        if prev is None or e['t'] - prev > CLUSTER_S:
            out.append(e)                 # first of cluster = the press
        last[e['crane']] = e['t']
    return out
```

**State the caveat every run.** Validated on **7 presses by one user (`ae7a7683-…`) across three
sessions**. Dev showed the reverse order, where first-of-cluster picks the wrong direction. A
genuine rapid load→unload inside 10 s collapses into one press. **It is a workaround for an open
defect** — re-validate the moment real operator traffic appears, and delete this section once
`crane_status_change` emits one event per press with an explicit `action` property.

---

## 4. THE FOUR USAGE STATES

| State | Definition |
|---|---|
| 🟢 **IN USE** | ≥1 press on the most recent working day in range |
| 🟡 **INTERMITTENT** | presses on some working days, none on the most recent |
| 🔴 **STOPPED** | presses earlier in range, then nothing — **report stop date/time in JST** |
| ⚫ **NEVER USED** | no presses in range. If also none in `last_30d`, say **"never used"** plainly |
| ⚫ **UNKNOWN** | query failed or was unstable (§0.3). Not "unused" |

---

## 5. METRICS — per crane, over the range

`Get-Property-Values` on `crane_status_change`, `result_mode: expanded`, properties
`["crane_name","loaded","crane_res_name","user_res_name","role","site_name"]`. De-duplicate per §3,
convert to JST per §1.

| Metric | Definition |
|---|---|
| **Load presses** | first-of-cluster events with `loaded=true` |
| **Unload presses** | first-of-cluster events with `loaded=false` |
| **Days used** | working days with ≥1 press ÷ working days in range |
| **In-hours / out-of-hours** | split at the §1 window |
| **Cycles** | load press followed by an unload press on the **same crane** |
| **Dwell** | per cycle, unload − load. Report median and max |
| **Unpaired loads** | loads with no following unload — **the confidence measure** |
| **Pressed by** | `user_res_name` + `role`, distinct count |

**Two measured dwell figures, both controlled tests:** `#5` 74 s (10 Sep), `#1` 155 s (16 Sep). Use
them to validate the calculation, never as a baseline for real cycles.

> **Print unpaired loads next to median dwell, always.** An operator who presses *Loaded* and forgets
> *Unloaded* leaves a cycle open indefinitely, indistinguishable from a long lift. **If unpaired ≥ ⅓
> of loads, print `dwell NOT RELIABLE` instead of a median.**

---

## 6. MOVEMENT CROSS-CHECK — REQUIRED, every run

Join on `crane_res_name` = ZLP asset UUID (§2). `get_tag_by_asset` per crane, then
`get_asset_history` over the same range. **Chunk by day and keep only the daily rollup** —
`get_asset_history` has no pagination and returns the whole range in one response.

Per crane per day record: fix count · first/last `locationTime` (JST) · whether the tag moved beyond
the noise floor. **Positioning is ±1–2 m — do not threshold raw per-fix displacement.** Aggregate to
3-minute medians first, per `koto-tag-usage.md` §5, and report `r95` rather than a movement fraction.

| Buttons | Crane tag | Reading |
|---|---|---|
| zero presses | moving normally | **crew is not using the feature.** Not an idle crane — the expected finding today |
| zero presses | also still | crane genuinely idle, or the site is dark — run `/koto-reader-check` before concluding |
| presses present | still | presses without movement — test traffic, or someone recording from the office |
| presses present | moving | intended state. Compare press count against movement segments |

**Two shapes that are NOT faults:**

- Crane tags switch to motion/OFF at **18:00 JST**; their last fix lands ~17:57–18:00. Never read
  that as the crane stopping.
- A range covering hours before the crew arrives, or a weekend, will show zero presses and a still
  crane. **That is expected, not a finding.** Say which part of the range was outside working hours.

---

## 7. REPORT

```
KOTO CRANE LOAD USAGE — <range, JST> — run <YYYY-MM-DD HH:MM JST>

One line: were the load buttons used, and what changed.

Scope        2 cranes (#1, #5)   Working hours: Mon–Fri 08:00–18:00 JST  (same window as every other check)
States       IN USE <n> · INTERMITTENT <n> · STOPPED <n> · NEVER USED <n> · UNKNOWN <n>
Presses      <n> total  (in-hours <n> · out-of-hours <n> · weekend <n>)
Change       <vs the recorded baseline | unavailable — no prior state>
Operators    <n> distinct   roles: <manager n, crane n>
Raw events   <n>  → <n> presses after de-duplication (§3)

=== PER CRANE ===
KotoCrane_#1   8889c289-…
     State      ⚫ NEVER USED
     Presses    loads <n> · unloads <n>
     Days used  <n>/<n> working days
     First/last <date time JST>  →  <date time JST>
     Cycles     <n>   dwell median <n> max <n>   unpaired loads <n>
     Movement   fixes <n>  r95 <n>m  last fix <time JST>
     Verdict    <one of the four §6 readings>
     Pressed by <user_res_name> (<role>)
KotoCrane_#5   f0d10e07-…
     ...

Out of scope (hygiene only, never graded)
- <crane_res_name> <n> presses <first>–<last> JST   ← scope may be stale   (or "none")

Ceiling on every figure above
- This records BUTTON PRESSES, not crane state. A zero is not an idle crane.
- De-duplication is a workaround for an open defect; validated on 7 test presses by one user.
- Dwell is only as good as unload discipline — <n> unpaired loads this range.

Known defects affecting this report
- crane_status_change emits 1–2 events per press, intermittently (no ticket yet)
- no `action` property: LOADED vs UNLOADED inferred from `loaded` + cluster order
- no press id, no server-side duration

Not measured
- <what did not run, and why>
```

---

## 8. BASELINE — write to `.koto-crane-usage-state.json` if absent

As of **2026-09-16**. **Every event in prod to date is a controlled test by Misha
(`ae7a7683-c52f-4fd7-9887-ff6e155fc11a`, role `manager`). No operator has ever pressed a crane
button.** Use it to compare against rather than starting blind.

```json
{
  "observed": "2026-09-16T18:15:00Z",
  "source": "mixpanel-3369288-crane_status_change + zlp-prd-jpn",
  "project_timezone": "UTC",
  "working_hours": "Mon-Fri 08:00-18:00 JST",
  "tracking_live_since": "2026-09-08T22:29:54Z",
  "scope": {"monitored": 2, "note": "#2/#3/#4 out of scope per Misha 2026-09-16; matches ZLP monitored set"},
  "totals": {"raw_events": 11, "presses": 7, "operator_presses": 0},
  "test_sessions": [
    {"date": "2026-09-10", "utc": "19:16:12-19:35:21", "presses": 5, "events": 9,
     "note": "Test A loaded x3 no unload; Test B #5 load->unload, dwell 74s"},
    {"date": "2026-09-16", "utc": "18:08:49-18:11:24", "presses": 2, "events": 2,
     "note": "Test C #1 load, held 20s untouched (persisted), then unload. Dwell 155s. No duplicates."}
  ],
  "cranes": {
    "8889c289-39bb-4b72-a2f9-0207e3cac8de": {"name": "KotoCrane_#1", "state": "NEVER_USED_BY_OPERATOR"},
    "f0d10e07-22c8-4809-b1f1-fbc3f8201109": {"name": "KotoCrane_#5", "state": "NEVER_USED_BY_OPERATOR"}
  },
  "usage_context": {
    "crane_tab_opens_90d": 130, "crane_tab_opens_koto_90d": 87,
    "crane_tab_opens_by_crane_role_90d": 28, "crane_tab_opens_since_10_sep": 1
  },
  "standing_items_open": [
    "zero_operator_presses",
    "duplicate_emission_defect",
    "no_action_property",
  ],
  "blocking_question": "Do the Koto operators know the LOADED/UNLOADED buttons exist? Zero operator presses since tracking went live. No instrumentation fix produces data until the crew uses the feature — this is a site conversation, not an engineering one."
}
```

**If the state file is absent or unparseable, say so and report the `Change` line as `unavailable` —
never as "no change."** Those are different facts.

---

## 9. BEFORE THIS GOES ANYWHERE NEAR KAJIMA

- **Never report a zero as an idle crane.** Say "no load presses recorded" and print the ceiling.
- **Never escalate on this.** It goes to a human who decides what reaches the customer.
- Press data is attributable to named ZaiNar and Kajima accounts via `user_res_name`. Same APPI
  handling as `koto-tag-usage.md` §8 — engineering-internal, not a customer-facing artifact.
- **While operator presses are zero, this report's real output is "the feature is unused",** not a
  usage figure. Say that plainly rather than printing zeros that look like measurement.
