# `crane_status_change` — verified in prod; duplication is nondeterministic, but dwell is derivable

Dev review 2026-09-09. **Prod verified against two controlled tests, 2026-09-10**, both by Misha on
`https://zps.jp.zainartech.com`:

- **Test A** — Loaded ×3: once on KotoCrane_#1, twice on KotoCrane_#5. No unload.
- **Test B** — Loaded on KotoCrane_#5, held ~1 minute, then Unloaded.

## Status: live, delivering, correct identity

Tracking is live in prod-jpn (`index.html` Last-Modified 8 Sep 2026 22:29:54 GMT). Events reach
Mixpanel project `3369288` within seconds.

| | |
|---|---|
| `crane_name` | `KotoCrane_#1`, `KotoCrane_#5` — correct |
| `crane_res_name` | `8889c289-…` (#1), `f0d10e07-…` (#5) — distinct, stable |
| `site_name` | `Koto_Pumping_Station_Crane` — correct |
| `role` / `user_res_name` | `manager` / `ae7a7683-…` (Misha) |
| Latency | seconds |

## The full prod record (9 events, 5 known presses)

```
19:16:12  #1  true      ┐ press 1 — Loaded, crane 1
19:16:14  #1  false     ┘
19:16:18  #5  true      ┐ press 2 — Loaded
19:16:22  #5  false     ┘
19:16:26  #5  true      ┐ press 3 — Loaded
19:16:31  #5  false     ┘
19:34:07  #5  true      ┐ press 4 — Loaded   ← both events true this time
19:34:09  #5  true      ┘
19:35:21  #5  false       press 5 — Unloaded ← single event, no duplicate
```

## 🔴 Revised defect: the tracker fires 1–2 events per press with a nondeterministic value

**This supersedes the earlier "every press emits two self-cancelling events" model, which Test B
disproved.** Events per press across the five known presses: **2, 2, 2, 2, 1**. Value pattern:
`(true,false)` ×3, then `(true,true)`, then `(false)`.

So neither the count nor the value is reliable per press:

- **Count is not consistently doubled.** Dividing total events by 2 is wrong — the unload press
  emitted once. Any "÷2" rule (including one this doc previously recommended) is unsafe.
- **The duplicate's value is not fixed.** Sometimes it repeats the new state (`true,true`),
  sometimes it reports the opposite (`true,false`). The earlier dev test doubled in the reverse
  order again (`false,true`).
- **Confirmed not a persistence fault.** Misha watched the UI: state stuck on *Loaded*. This is a
  logging defect, not a failing save.

Likely cause for whoever fixes it: two async paths firing the tracker — an optimistic local update
plus a server/websocket confirmation — racing, each reading the state variable at send time, so both
the number of emissions and the snapshot each captures vary.

### What to ask for

1. **One event per press.** The blocker.
2. **An explicit `action` property naming the button pressed** — `LOADED` / `UNLOADED` — separate
   from the resulting state. Survives an accidental duplicate in a way a bare state field cannot.
3. **A per-press id**, so duplicates are detectable rather than silent.
4. **`loaded_duration_seconds` on the unload event**, computed server-side. Removes client-side
   pairing from the dwell calculation entirely.

## ✅ Dwell time is derivable — and Test B measured it

**KotoCrane_#5: loaded 19:34:07 UTC → unloaded 19:35:21 UTC = 1 min 14 s.** Misha reported holding
it "about a minute", so the figure is correct. Taking the duplicate at 19:34:09 instead gives 72 s —
the duplication costs a couple of seconds of precision, not the measurement.

This revises the earlier finding that dwell was underivable. It was underivable from Test A because
that test contained **no unload press at all** — the apparent 2–5 s "dwells" there were the
duplicate artifact, not cycles.

### Interim rule that works on all five presses

**Cluster events per crane within a ~10 s window; take the first event of each cluster as the
press.** Applied to the record above it yields exactly the ground truth: 4 Loaded + 1 Unloaded,
correct crane, correct direction, correct timestamps. Dwell = interval between consecutive
first-of-cluster events of opposite value on the same crane.

**Caveats, and they matter before this goes into a report:**

- Validated on **5 presses by one user in one session**. Not a proven rule.
- **Dev showed the opposite order** (`false` then `true`), so first-of-cluster would have picked the
  wrong direction there. Either the builds differ or the race resolves differently under load.
  Treat the rule as provisional until it is checked against real operator traffic.
- A rapid genuine load→unload inside the cluster window would be collapsed into one press.
- It is a workaround for a bug, not a substitute for fixing it.

### The dwell risk that instrumentation cannot fix

Dwell requires crews to press **both** buttons. An operator who presses *Loaded* and forgets
*Unloaded* leaves the cycle open indefinitely — indistinguishable from a genuinely long lift. Test A
is a live example: crane #1 was loaded and never unloaded. **Publish unpaired-load count alongside
every dwell figure**; it is the honest confidence measure for the whole metric.

**Operational note:** #5 is now correctly unloaded, but **KotoCrane_#1 has been flagged LOADED since
19:16:12 and was never unloaded.** Worth clearing if that state feeds red-zone alerting.

## ✅ Resolved: the Mixpanel project timezone is UTC

Misha pressed at **~12:15 PT** (UTC−7); Mixpanel stamped **19:16**. 12:15 PT = 19:15 UTC. **Project
timezone is UTC.**

For the agreed **07:00–19:00 JST, Mon–Fri** window:

- JST = UTC+9, fixed — **Japan observes no DST**.
- 07:00–19:00 JST = **22:00–10:00 UTC**, crossing midnight UTC.
- Mon–Fri JST begins **Sunday 22:00 UTC**, ends **Friday 10:00 UTC**.
- Worked example: Test B's 19:34 UTC = **11 Sep 04:34 JST** — a Thursday-lunchtime press in
  California files as Friday small hours in Japan, outside working hours.

Because the offset is fixed, the JST split needs no client change: map UTC hour buckets by hand, or
create a Mixpanel **custom property** computing JST hour and weekday from `$time` + 9h. So
`site_local_hour` / `site_local_dow` are now nice-to-have, not blocking.

## Full property payload

**Identity:** `crane_res_name`, `crane_name`
**State:** `loaded` (boolean)
**Context:** `feature: cranes` · `site_name` · `site_res_name` · `account_name` · `acct_res_name` ·
`user_res_name` · `role` · `platform: web`
**Mixpanel standard:** `$user_id`, `$current_url`, `$browser`, `$os`, `$device_id`, `$lib_version`,
`$city`, `$insert_id`, `$mp_event_size`

**Absent:** `action`, any press id, any duration, `site_local_hour`, `site_local_dow`,
`site_timezone`, `crane_category`, `crane_firm`.

**Cosmetic:** `feature: "cranes"` is plural where every other value is singular (`map`, `actions`,
`equipment`, `people`, `settings`). Align before it is baked into saved reports.

## Usage context

This week (7–10 Sep) every `crane_status_change` event in prod is one of Misha's 9 test events.
Nobody opened the crane tab in prod between 3 Sep and the tests — see
`claude/zps-prod-stale-cloudfront-deploy.md`. Over 90 days the tab saw 130 opens, 87 at Koto
(59 manager, 28 `crane` role), concentrated in late June then tapering.

**A zero here is not an idle crane.** The event records button presses, not crane state. If crews do
not press, the metric reads zero however much lifting happened. Every report built on this must say
so on its face.
