# What counts as a healthy tag — Koto crane site

**Status:** draft for review, 2026-09-02. **Rev 3** — corrected against `tag-tools-reference.md` (2026-09-02) and `koto-tag-movement-check.md`, both in `~/Documents/zlp_references/`. Three rev-2 statements were wrong and are fixed below: the monitored set is **12, not 13** (Crane **#1** and **#5**, two named assets — not three magnet tags); criterion 1b and criterion 4 **do** have data sources; and reporting mode **can be read**, so "assumed-continuous" is a fallback rather than the rule.

> **The runnable version of this definition is `koto-tag-check.md`** in `~/Documents/zlp_references/`, which supersedes `koto-tag-movement-check.md`. This file is the reasoning; that file is the procedure. Keep them in step.

Rev 2 — Misha's movement-based frame, rewritten with the recommendations folded in, then with the open conflicts resolved into decisions. Completes the trio with `reader-health-definition.md` and `hub-health-definition.md`, and answers the tag half of open decision §7.2 in `kajima-site-monitor-spec.md`.
**Scope:** **the crane magnet tags and the Subcon C01–C10 worker tags** at `Koto_Pumping_Station_Crane` (`875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`) — 12 tags, named in §0. Every other tracker at the site is inventory and is reported as a hygiene count only. Structure deliberately mirrors the reader and hub definitions so the three read as a set.

**Revision 2 changes.** Rev 1 named five conflicts and eight open questions and resolved none of them, which makes a definition that reads well and grades nothing. This revision decides the ones that can be decided from evidence already in hand, and says plainly which of them a human still has to sign off:

| Rev 1 left open | Rev 2 decision | Where |
|---|---|---|
| Which tier the 15-minute number fires at | Grade against the **continuous** tier inside working hours; do not grade outside them | §1a |
| Reporting mode unknown → UNKNOWN | **Assumed-continuous inside working hours**, labelled as assumed, with a mode-contradiction check | §1a |
| Movement threshold *d* undefined | Interim *d* = **2.0 m**, explicitly a placeholder, plus the one-pull measurement that replaces it | §4a |
| Denominator unknown (31 vs 63) | **Superseded.** Scope narrowed 2026-09-02 to an explicit, named set of **12** — 10 Subcon C worker tags + 2 crane equipment tags. The 31-vs-63 gap becomes a hygiene line, not a blocker | §0 |
| Expected-headcount source for the roster backstop | The **10 named Subcon C workers**. Weekly reconciliation against `list_assets_with_tags`, with its age reported every run | §0 |
| No alert design | Two alerts only, mirroring `koto-hub-check-command.md` | §Alerts |

Still needs a human: the alert tier sign-off (§Alerts), APPI retention and access (§Conflicts 4), and whether the discipline output shares an artifact with the engineering report.

> **Terminology:** tag, tracker and node are the same object across the three systems. ZPS says *tag*, the registry says *tracker*, the WiFi-RT tools say *node*. Three physically different populations wear the name — worker tags, crane magnet tags, and the anchor tags fixed to readers — and they have different healthy behaviour. §0 splits them.

---

## The frame

The starting definition was:

> Movement every 15 minutes in the last 24 hours during a work shift, according to the working hours by Japanese time.

The instinct behind it is right and is the one the other two definitions do not cover: a tag exists to produce *positions of a thing that moves*, so evidence of movement is closer to the point than a heartbeat is. A reader can heartbeat and contribute nothing; a tag can heartbeat and be sitting in a drawer.

Four things have to change before it can grade anything.

### Problem 1 — read literally, it fails the entire fleet every day

"Movement every 15 minutes" over a ten-hour shift is 40 consecutive intervals, each of which must contain movement. Nobody satisfies that. A worker at a control panel, a toolbox talk, lunch, a crane waiting on a load — every one produces a 15-minute interval with no movement, and under an all-of rule every one is a failure.

So the criterion has to be a **fraction, not an all-of**: intervals containing qualifying movement ÷ intervals in which movement was expected. That single change is what makes it gradeable, and it forces the two questions in Problems 2 and 4 to be answered — what counts as movement, and what counts as expected.

### Problem 2 — movement and health are two different questions, and this conflates them

*Is the device working* and *is the tag being worn* are separate, and the same silence answers both. A worker tag that reports every minute from a bench is perfectly healthy hardware. A crane magnet tag on a crane that has not moved since Tuesday is perfectly healthy hardware. Neither is unhealthy, and grading them unhealthy trains people to ignore the report — the incident's lesson #1 in a third costume.

The reverse matters more. **"Present but not moving" is already a named deliverable at this site**, and it is not an engineering one: Kajima was formally asked to ensure staff wear charged tags, and `koto-crane-daily.md` §8 carries the bench-tag flag as the compliance signal the site asked about. Folding it into a device-health grade sends a personal accusation about an identifiable worker up an engineering channel, under APPI, with **BUG-432** (tags listed onsite with no historical trail) and **BUG-431** (inaccurate "No data") both open on exactly the shape that produces it.

So movement stays — as criterion 4, on its own line, with its output routed to the discipline report and never to the device grade. **The device grade never says "unhealthy" because a tag did not move.**

### Problem 3 — 15 minutes is a real threshold, but it belongs to reporting, and it is mode-dependent

The site's documented tag SLAs:

| Mode | Expected | Warn | Error |
|---|---|---|---|
| `on_motion` | ~5 min | > 15 min | > 60 min |
| `continuous` | more frequent | > 5 min | > 15 min |

Fifteen minutes is the **warn** tier for `on_motion` and the **error** tier for `continuous` — a 4× difference in what the same number means. And at this site the mode is not a per-tag constant: the Slack scheduler switches the fleet daily — `23:00 UTC` = 08:00 JST puts tags into **continuous** at the start of the working day, `09:00 UTC` = 18:00 JST puts them into **motion / OFF** at the end, skipped on JST weekends (`koto-crane-daily.md` §11). A fixed 15-minute rule is therefore wrong for part of every single day, and wrong in the dangerous direction overnight.

There is also a trap inside the switch itself: through 08-20 → 08-25, `#1_21ee` reported disconnected at nearly every job window and reconnected ~30 min later. **The first ~30 minutes either side of a mode switch is a known artifact window.** Do not grade in it without saying so.

### Problem 4 — "during a work shift" is not the same as site working hours

Site hours at Crane are **08:00–18:00 JST Mon–Sat** (the window both sibling definitions use). A worker's shift is not that window: across the ten Subcon C workers, last-report times landed on six different days and at markedly different hours, and presence status flipped Offsite within ~2 min of each tag's last report for 9 of 10. **The presence logic is sound, and shifts genuinely end at different times.**

Grading a tag against 08:00–18:00 fails every early finisher. Grading it only against its own observed presence window creates the failure the reader definition already rejected once: **the exclusion criterion becomes the signal being graded.** A tag that is dead never shows presence, is therefore never evaluated, drops out of the denominator, and the site returns to green with a dead fleet. That is precisely how the 100-day-silence exclusion was rejected in `reader-health-definition.md` §0.

The resolution is that both are needed and they answer different questions — see criterion 4's denominator, and the roster backstop in §0 that catches a tag with no presence *at all*.

### Problem 5 — everything here is observed cloud-side

Same as the other two. In the May outage all 30 readers were alive and answering local pings while every cloud view was dark. ⚫ **UNKNOWN** is a grade.

---

## The definition

A tag is **healthy** when criteria 0–3 and 5 hold. **Criterion 4 (movement) is measured and reported but does not set the device grade** — see Problem 2.

**Evaluation windows — both, doing different jobs. Never let the long one grade the site.**

| Window | Question | Use |
|---|---|---|
| **Newest-report age**, mode-aware (see 1) | is the tag up *now* | **grading and alerting** |
| **The JST working day just ended** (08:00–18:00 Mon–Sat), with per-tag presence sub-windows | shift coverage, movement fraction, battery trajectory | the report body |
| **Rolling 24 h** | overnight behaviour, mode-switch artifacts, cohort-decay clustering | context only, never the grade |

---

### 0. Present — it is on the roster, and it is the kind of tag you think it is

**The monitored set is explicit, named, and small — decided 2026-09-02.** Only two populations are in scope: the **crane magnet tags** and the **Subcon C01–C10 worker tags**. This is the single most useful decision on this page, because it dissolves the problem rev 1 called its largest gap. The registry counts **63 trackers** at Crane and enumerates none; the ZPS Devices list showed **31**; neither number has ever been reconciled. **Neither is the denominator any more.**

**Worker tags — 10, named** (mapping from `koto-crane-24h-movement-review.md`, verified worker-by-worker 2026-08-27):

| Worker | Tag | Note |
|---|---|---|
| C01 | `kps_289a` | |
| C02 | `kps_4cdd` | |
| C03 | `kps_cf91` | |
| C04 | `kps_c79b` | |
| **C05** | **`A5_kps_ae92`** | **Silent since 22 June. Has never been tracked.** |
| C06 | `B1_kps_d38f` | prefix mismatch — DEF-062 |
| C07 | `kps_df91` | |
| C08 | `B2_kps_44f8` | prefix mismatch — DEF-062 |
| C09 | `B3_kps_1a93` | prefix mismatch — DEF-062 |
| C10 | `kps_2c6d` | |

**Crane equipment tags — 2, by asset UUID** (corrected in rev 3; rev 2's "3 magnet tags" came from the roster count, not from what is monitored):

| Asset | Asset UUID | Tag |
|---|---|---|
| `KotoCrane_#1` | `8889c289-39bb-4b72-a2f9-0207e3cac8de` | resolve via `get_tag_by_asset` |
| `KotoCrane_#5` | `f0d10e07-22c8-4809-b1f1-fbc3f8201109` | resolve via `get_tag_by_asset` |

> **Monitored denominator: 12 — 10 worker tags + 2 crane equipment tags.**

The roster's third magnet tag and the untagged `KotoCrane_#2`/`#3`/`#4` remain open: `#1_21ee` and `#2_02f9` are the two magnet tags named anywhere in this project, on 2026-08-27 only two were visible where the roster says three, and the monitored set names #1 and #5. **Five cranes, two monitored tags** — whether that is intended is a question for Simon, and it is a coverage gap the 63-tracker view hid entirely.

Three consequences of scoping this tightly, and they all cut in the same direction:

- **The scale changes what a single failure means.** C05 is not one stale tag in a fleet of 63 — it is **10 % of the monitored worker population, and it has never worked**. At fleet scale that reads as noise. At this scale it is the headline.
- **DEF-062 is now load-bearing, not a footnote.** Prefix mismatches hit **3 of the 10 in-scope worker tags** (C06, C08, C09 — and C05's `A5_` prefix makes four names that mislead). Eye-reconciling roster against tag list will mis-attribute nearly half the monitored set, and a mis-attributed compliance finding is worse than no finding. **Match on tag ID, never on prefix.**
- **12 tags is cheap enough to check individually, every run.** No sampling, no rotating slice, no two-stage sequencing of the kind the 29-reader definition needs. The per-worker pass that found C05 *is* the routine now, rather than the thing nobody has time for.

**Everything else at Crane is inventory for now.** The other ~20–50 trackers, whatever the true count, are reported as **one hygiene line — count only, never in the health score, never in a denominator**. Rev 1's assignment rule (`list_assets_with_tags`: a tracker with no current person or asset mapping is inventory) is what sorts them, and it stays as the rule for *adding* a tag to the monitored set later. **Anchor tags are out of scope here** — a fixed tag that moves is a calibration finding and belongs to `reader-health-definition.md` §4, not to this document.

**Two classes in scope, two different healthy behaviours.** Classify before grading, or criterion 4 is meaningless:

| Class | Count | Movement expectation |
|---|---|---|
| **Worker tag** | 10 | Moves during the wearer's presence window |
| **Crane equipment tag** | 2 | Motionless is normal. Movement is the *event*, not the health signal — it is crane utilisation, and it is what answers "who left crane #2 loaded" |

**One thing this scoping exposes that the fleet view hid: there are five cranes and three magnet tags.** `KotoCrane_#1`–`#5` appear in the ZPS crane selector. If the monitored set is deliberately three, two cranes are untagged and the report should say which. If it is meant to be five, two tags are missing. Either way it is now a visible question rather than a rounding error inside 63.

**The roster backstop — this is the one that catches what movement cannot.** For each tag mapped to a person expected on site, no reports *at all* across the working day is not "not evaluated". It is its own state and it must be named in every run:

> `A5_kps_ae92` (worker C05) has been silent since **22 June** — two months before the crew was tagged — while sitting on the roster indistinguishable from nine tracked colleagues. It surfaced only because someone checked each worker individually. **A fleet check that gates on presence would never have found it.**

**How the backstop runs — decided, and the narrowed scope makes it trivial.** Rev 1 left "expected-headcount source" open, which meant the backstop was a principle with no owner and would never have run. The expected set is now simply **the 10 named Subcon C workers**:

> **Every run: for each of the 12 monitored tags, report last-ever activity. Any tag with zero reports in the preceding seven days is named individually, with its date.**
>
> **Weekly, Monday JST:** re-reconcile the tag↔worker mapping against `list_assets_with_tags`, so a crew change cannot silently retire a tag from the monitored set.

Seven days rather than one, because a worker off for two days is not a finding and a tag silent for a week is. At 12 tags this runs on every pass rather than weekly — the cost argument that would have made it a weekly-only check at fleet scale does not apply.

**Every run reports the age of the last mapping reconciliation.** A backstop whose staleness is invisible is not a backstop. More than 14 days old is itself a 🟡 WATCH.

### 1. Reporting — it is talking, at the cadence its current mode requires

This is the corrected form of the original criterion, and it is the one that decides the grade.

**1a. Mode — decided.** Rev 1 said an undetermined mode makes the tag ⚫ UNKNOWN, on the grounds that neither default is safe: `on_motion` quadruples every tolerance and hides real silence, `continuous` cries wolf every evening. **That dilemma only exists if you try to grade all 24 hours.** Once grading is confined to the window where the scheduler puts tags into continuous, it disappears:

> **Grade criterion 1 only inside JST working hours (08:00–18:00 Mon–Sat), against the `continuous` tier — warn > 5 min, error > 15 min. Outside that window, criterion 1 is not evaluated.** Overnight coverage comes from the §0 roster backstop instead, which is the check that actually catches a dead tag.

The residual risk is now one-directional and it is the safe direction: if a tag is actually in `on_motion` during working hours, the stricter tier produces a false WATCH, not a hidden outage.

Two things this does **not** license:

- **Label it.** Every graded tag carries `mode: assumed-continuous (scheduler)` in the report. An assumption stated is a finding; an assumption buried is a bug.
- **Check the assumption against behaviour.** A tag reporting steadily at ~5-minute intervals through working hours is behaving `on_motion`, which contradicts the scheduler. That is a **mode-contradiction finding → ⚫ UNKNOWN for that tag**, not an unhealthy grade — the same shape as the readers' `Active`-while-offline contradiction check. Rev 1's UNKNOWN survives here, where it is earned by evidence rather than used to avoid a decision.

Per-tag mode remains open decision §7.2 and reading it properly still closes this for good. This is what to do until someone does.

**1b. Grade on density across the shift, not on age-of-last.** Identical argument to `reader-health-definition.md` §3, and it bites harder here: runs land ~08:00–09:00 JST, so a tag dark from 08:30 to 17:30 the previous day has a fresh report and reads healthy. **The failure heals before the check looks at it**, and daytime-only failure with overnight recovery is this site's documented signature.

| Metric | Definition | Healthy | WATCH | AMBER |
|---|---|---|---|---|
| **Report fraction** | reports received ÷ expected at the continuous cadence, over the tag's shift window | ≥ 95 % | 80–95 % | < 80 % |
| **Longest single gap** | max continuous silence inside the shift window | < 5 min | 5–15 min | > 15 min |
| **Flap count** | distinct disconnect→reconnect events | ≤ 2 / day | 3–5 | > 5 |
| **Newest-report age** | at run time, during working hours | < 5 min | 5–15 min | > 15 min |

**Two exclusions, applied before any of the four are computed:**

1. **The ±30 min windows around each mode switch** (08:00 and 18:00 JST) — the documented artifact window, §Problem 3.
2. **Intervals with no live reader in the tag's area** — no coverage, no expected report. Not evaluated, not failed.

Report the longest gap next to the fraction, always. 98 % with one 40-minute hole during a lift is worse than 98 % of five-second blips.

**A single tag hitting AMBER is a tag finding. Three or more hitting it together is one site finding** — same rule as the readers' correlated-instability shape. Thirty findings is how the one that matters gets buried.

**1c. Cohort decay is a site finding, not 30 tag findings.** Collect the last-ever activity of every stale tag, convert to JST, and look for clustering in **16:00–16:45 JST** across different days — the documented modem signature; the 2026-08-25 baseline found seven tags clustered there. **State the competing explanation in the same breath:** end of shift produces exactly the same shape. This data cannot separate them, and naming the modem as cause from it alone is a guess wearing a finding's clothes.

### 2. Powered — battery, as a trajectory and not a threshold

Absent from the original frame, and it is the dominant tag failure mode. It is also the **leading** indicator for the silence criterion 1 detects only after the fact.

Bands are ZLP's own active rules — use these, not an invented 20 %:

| State | Rule |
|---|---|
| Low | 10–29 % and discharging |
| Critical | < 10 % and still reporting |
| Dead | ≤ 5 % and silent 15+ min |

**Deduplicate by tag before reporting any count.** The alert stream is heavily duplicated — baseline: 12 identical `A4_kps_6fe0` alerts stamped the same minute. Distinct tags affected, raw count in parentheses.

**The number worth adding: discharge rate and time-to-dead against the remaining shift.** A tag entering the shift at 12 % will die around lunch. That is actionable at 08:00 and useless at 16:00, and it is the tag's counterpart to the readers' thermal-trend prediction — the one thing that moves this site from reactive to predictive. **Grade WATCH on a tag projected to die inside the shift, before it does.**

### 3. Located — it is producing position fixes, not just reports

A tag can report on schedule and contribute no location. The affirmative evidence is already in the data:

- `0x0400 hub_not_found` — the tag saying directly it could not reach its hub
- `0x0100 ranging_error` — ranging attempted and failing

Any tag raising either in the window = **WATCH**; multiple distinct tags, or a count rising day over day = **AMBER**. Deduplicate by tag first.

**Gate on coverage.** A tag off site produces no fixes and that is not a fault. No live readers in its area → the criterion is **not evaluated**, not failed.

### 4. Moving — measured, reported, and kept out of the device grade

**4a. Define movement as displacement, and set the floor above the noise.** This is the unsolved mechanical problem in the original definition. Position output at this site jitters, and **BUG-304** — last-known-location triggering red-zone alerts — is open evidence that stationary tags emit position events. A naive "position changed" test is satisfied by multipath noise on a bench tag, which quietly destroys the exact compliance signal criterion 4 exists to feed.

> **Movement = displacement between consecutive fixes exceeding *d*, where *d* is set from the observed still-tag noise distribution — not chosen.**

**Rev 3: run dual-threshold until *d* is measured.** `koto-tag-movement-check.md` proposed **500 mm** while its own limitations section states positioning is **±1–2 m** in obstructed areas. Those cannot both stand: at ±1–2 m error, a tag on a bench clears 500 mm in most intervals, and **the test meant to detect an unworn tag instead certifies it as healthy** — the compliance signal inverts. So compute the fraction at **both 500 mm and 2000 mm and report both**; the gap between them says how much of the "movement" is noise. `500mm: 94% / 2000mm: 11%` means neither number describes the worker.

**Interim value: *d* = 2.0 m. This is a placeholder, not a measurement**, and the report must say so wherever a movement figure appears. It is set above the metre-scale error typical of WiFi-RTT positioning so that ordinary jitter cannot manufacture movement; it has never been checked against this site's actual noise, and the site is only 27.1 × 34.9 m, so 2 m is a meaningful fraction of it. Too high and a worker pacing a control platform reads as a bench tag — which is the expensive error here, because that is the one that reaches Kajima.

**The measurement that replaces it — one pull, no new tooling:**

1. Pick a tag known to be stationary: a crane magnet tag on a crane that did not move, or any worker tag between 20:00 and 06:00 JST.
2. Pull 12 hours of position fixes for it.
3. Compute displacement between each consecutive pair.
4. **Set *d* to the 99th percentile.** That is, by construction, the displacement a still tag exceeds 1 % of the time.
5. Repeat on a second tag in a different part of the site. If the two differ by more than ~50 %, *d* is position-dependent and needs to be per-zone rather than site-wide — which is itself a finding worth having.

**Until step 4 is done, criterion 4 is descriptive only:** it may produce candidate lists in the engineering-internal brief, and it must not appear in any customer-facing artifact or be quoted to Kajima in any form.

**4b. Score as a fraction over the tag's own presence window.**

> Movement fraction = 15-minute intervals containing ≥ 1 qualifying displacement ÷ intervals inside the tag's observed presence window.

Presence window = first to last report of the day for that tag, which the Subcon C data shows tracks the real shift within ~2 minutes. Intervals outside it are not counted and not failed.

**4c. Per class, and only worker tags produce a discipline signal.**

| Class | Reading | Meaning |
|---|---|---|
| Worker tag | fraction ≥ 60 % | consistent with a worn tag |
| Worker tag | fraction < 20 % with presence recorded | **bench-tag candidate** — discipline output, with the caveat below |
| Worker tag | presence recorded, zero qualifying displacement all day | strongest bench-tag candidate, same caveat |
| Crane magnet tag | any reading | not a health signal. Report movement *events* with times — that is crane utilisation, and it is what answers "who left crane #2 loaded" |

Anchor tags are out of scope (§0): a fixed tag that moves is a calibration finding and belongs to `reader-health-definition.md` §4.

**At ten workers, name them individually.** A fleet-scale report would give a distribution; this one gives ten rows, C01 to C10, each with its fraction. That is the format that made C05 visible, and it costs nothing at this size.

**4d. The caveat is mandatory and goes in every report that carries a bench-tag candidate.** **BUG-432** (High, In Progress) is open on tags listed onsite in Site Presence with no historical trail — a platform artifact **indistinguishable from a bench tag in this view**. **BUG-431** adds that the "No data" message is inaccurate. And if the reading came through the ZPS Reports UI, **DEF-059** (Confirm reverts the date range to a 1-hour default) and **DEF-060** (Reset clears the range) both silently produce a wrongly-scoped query, **and a wrongly-scoped query looks exactly like a motionless worker**.

> Report the observation, name BUG-432, and state that the two cannot be separated from the Reports UI alone. Telling Kajima a worker left their tag on a bench when the platform invented the presence record is a serious and personal accusation to get wrong.

### 5. Trustworthy — it is the tag you think it is, and its position is sane

- **Mapping current:** `list_assets_with_tags` still correct after crew changes. Any tag→person change since the last run is reported.
- **Naming coherent:** DEF-062 prefix mismatches carried until normalised.
- **Z sane:** Crane z is positive (900 mm → 5.0 m). Crane #5 parked high is the known z-outlier case; **BUG-227** is open on WiFi-Cloud and ZLP disagreeing in the Location Engine for exactly that.
- **Cross-system coherent:** WiFi-Cloud and ZLP agree on identity and state.

Any single finding here = **WATCH**. This is the class of fault that never becomes urgent on its own and degrades every fix the site produces until someone looks.

---

## ⚫ UNKNOWN is a grade

| Grade | Meaning |
|---|---|
| 🟢 **HEALTHY** | Criteria 0–3 and 5 met over the evaluation window |
| 🟡 / 🟠 / 🔴 | Fails one or more — per-criterion thresholds above |
| ⚫ **UNKNOWN** | Could not observe it. Monitoring path down, tag roster unresolved, **reporting mode undetermined**, site uplink down |

A tag UNKNOWN is a different fact from a tag UNHEALTHY, and both differ from healthy. Conflating the first two is what let a 13-day outage stay hidden.

**Mode-undetermined maps to UNKNOWN, not to a default.** Picking `on_motion` "to be safe" quadruples every tolerance and hides real silence; picking `continuous` cries wolf every evening. Neither is a safe default, so neither is used.

---

## Site grade from tag grades

Uses the register's existing language (`kajima-site-monitor-spec.md` §3) — no parallel scale:

| Site level | Condition |
|---|---|
| ⚫ **UNKNOWN** | Monitoring path unavailable, or the monitored set could not be resolved |
| 🔴 **P2** | Zero location events from any monitored tag during working hours *(this is what 13 days dark looked like)* |
| 🟠 **P3** | **≥ 3 of the 10 worker tags** silent across a working day with the crew on site · ≥ 3 monitored tags at criterion-1 AMBER together · cohort-decay cluster confirmed across multiple days |
| 🟡 **WATCH** | Any monitored tag battery critical · any projected to die inside the shift · any `0x0400`/`0x0100` tag · any criterion 5 finding · mapping reconciliation > 14 days old · the third magnet tag still unnamed · a crane with no magnet tag |
| 🟢 **GREEN** | All 12 accounted for and reporting to mode through the working day, no battery or flag findings |

**The denominator is 12 — 10 worker tags + 2 crane equipment tags — and every count in the report is against it.** Not 31, not 63, and never the platform's own tag total. The ≥ 3 P3 threshold is deliberately the same absolute number the register uses for readers, and at 10 worker tags it is a far heavier proportion: **three silent tags is 30 % of the monitored crew, not 10 % of a roster.** That is the correct severity for a site where the tags are what keep people out from under crane loads, and it should not be softened to a percentage to make the number look smaller.

**Report the inventory count on its own line, always, and never inside the grade:** `Inventory: <n> other trackers at site, not monitored`.

**Movement never sets the site grade.** A day where every tag reported perfectly and nobody moved is a green site with a compliance finding, and that is the honest reading.

---

## Which tools check this definition

**Mapped against `tag-tools-reference.md`, 2026-09-02** — 27 WiFi-Diagnostics tag tools (15 read) plus 6 ZLP-Core CRUD. Rev 2 said this reference did not exist and that two criteria had no data source. **Both statements were wrong**, and the correction runs in the good direction: every criterion in this document now has a source.

| Tool | Criteria served |
|---|---|
| `list_offline_tags` | 0 — site sweep in one call |
| `list_continuous_mode_tags` | **1a — reads the mode instead of assuming it** |
| `get_tag_by_asset` / `list_assets` | 0, 5 — resolve asset → tag **by UUID**, which is what makes DEF-062 harmless |
| `list_assets_with_tags` | 0, 5 — the mapping check |
| `get_asset_history` | **1b, 4 — position fixes over the window. The time series rev 2 said did not exist.** |
| `get_tag_location_history` | 1b, 4 — alternate source; may fail for some tags |
| `get_continuous_tag_health` | 1 — health summary for continuous-mode tags |
| `get_tag_status` | 2, 3 — battery, RSSI, flags |
| `get_tag_battery` / `get_tag_fuel_gauge` | **2 — time-to-empty directly, no need to derive a discharge rate** |
| `get_tag_events`, `get_tag_config`, `check_tag_hub_connectivity` | triage |
| `list_trackers` | inventory hygiene count |

**Three corrections the reference forces:**

1. **`get_tag_status().last_heartbeat` must not grade anything.** The reference is explicit: it returns formatted strings (`"5.4h ago"`), *"often shows stale values even when the tag is actively reporting"*, and **"Do NOT use for health checks."** This is the same defect the reader definition flagged as its conflict #2, now confirmed and stronger. **Liveness comes from the newest `locationTime`.**
2. **Criterion 1a's "assumed-continuous" is a fallback, not the rule.** `list_continuous_mode_tags` reads mode directly. Keep the fallback for when it cannot be reached; delete the word "assumed" from the report the moment it can.
3. **Criterion 2 gets simpler.** `get_tag_battery` returns time-to-empty, so the projected-death-inside-shift test needs no derivation.

**`ping_tag` is a write tool.** It reads like the obvious way to answer "is this tag alive?" and it transmits to the tag. So does `set_tag_motion_sensitivity`, which is the natural-looking fix for a tag failing criterion 4 and would silently rewrite what every future run measures. Neither belongs in a health check.

**Cost, at 12 tags: 1 site call + 3 per tag = 37 calls.** That is small enough to run every pass with no sampling, no rotating slice, and no two-stage sequencing — unlike the 29-reader definition, which needs all three to stay affordable. **The narrowed scope buys completeness**, and completeness is what would have caught C05.

### Write tools a health check never calls

Standing rule, no exceptions. **`reboot_tag_host` reads like a tag tool and is not** — it reboots the tag-management process on the *hub*, taking out tag handling for every tag on that hub at once. Dry-run (`confirm: false`) is the default and is the only guardrail; it protects against a mistaken call, not a mistaken `confirm: true`. The credential in use has write+admin on `zlp-prd-jpn`. **Issue the monitor's key scoped `viewer` / `engineering:read`.**

---

## Alerts — two only

> **Superseded as a build instruction, kept as reasoning.** `/koto-tag-check` no longer implements
> any of this: cadence, delivery and the confirm-before-paging ladder are the Agent Scheduler's to
> set, and nothing in the shipped package schedules or pages. The design below still records *why*
> two alerts rather than ten, and why criterion 4 must never page — carry that reasoning into
> whatever the scheduler ends up doing, but do not read this section as describing the command's
> current behaviour.

Everything not listed here goes in the daily report, never to a channel. Same design as `koto-hub-check-command.md`, deliberately, so the three monitors behave identically when they fire.

### Alert 1 — page, `@here`

Fire only when **both** hold, **confirmed on two consecutive runs**:

1. **≥ 3 of the 10 worker tags** past the criterion-1 error tier during JST working hours with the crew on site, **or** zero location events from any monitored tag during working hours, and
2. The reader and hub checks are **not** simultaneously red — if they are, this is one site failure and the reader or hub monitor pages, not this one.

```
🔴 KOTO CRANE TAGS — <n>/12 monitored tags not reporting — <HH:MM JST / HH:MM PT>
Workers affected: <C0n list>   Cranes: <#n list>
Mode graded: assumed-continuous (scheduler)
Not checked: <criteria with no data source this run>
```

### Alert 2 — no `@here`, monitor owner only

The run did not complete, or the monitored set could not be resolved, twice in a row. Says *condition unknown*, never *healthy*.

**Never alert on criterion 4.** A movement finding is a compliance observation about a named person; it goes to the engineering-internal brief and a human decides whether it reaches Kajima. Paging a channel about a worker is not a trade worth making, and with BUG-432 open it could be wrong.

**Suppression:** two consecutive confirming runs before any page · no re-page for the same tag within 12 h · correlated failure across tags → one page, not one per tag · recovery in the original thread.

---

## Conflicts — what rev 2 decided, and what still needs a human

**Decided in rev 2:**

| Was | Now |
|---|---|
| Which tier the 15-minute number fires at | Continuous tier inside working hours; not evaluated outside (§1a) |
| Mode unknown → everything UNKNOWN | Assumed-continuous, labelled, with a behavioural contradiction check (§1a) |
| Movement threshold undefined | Interim *d* = 2.0 m as a stated placeholder + the pull that replaces it (§4a) |
| Denominator unresolved | 12 named tags (§0) |
| Backstop had no expected-set source | The 10 named Subcon C workers, checked every run (§0) |
| No alert design | Two alerts, above |

**Still open, and each needs a person:**

**1. Alert tier sign-off.** The tiers above are defensible, not authorised. Someone owns the decision that ≥ 3 of 10 worker tags silent is worth a page at this site.

**2. Per-tag reporting mode** — open decision §7.2 since the spec was written. §1a is what to do until it is read; reading it closes the question properly and removes the word "assumed" from every report.

**3. Backfill semantics.** If tag reports are buffered at the hub and flushed when the uplink returns, a multi-day outage backfills and gap analysis afterwards shows a continuous series. Does the source expose both an event timestamp and an ingestion timestamp? If only one, density silently stops being trustworthy during exactly the incident it is meant to catch.

**4. APPI — retention and access, still undecided.** Criterion 4's output is per-person movement data on identifiable workers and subcontractors. Say so in every report until it is decided, and keep it out of any customer-facing artifact. Narrowing scope to ten named subcontractors makes this **more** pointed, not less: the data is now about a specific, small, identifiable group.

---

## Next steps, cheapest first

Each of these is short, and each one removes a caveat that currently has to be printed in every report.

| # | Do | Closes | Cost |
|---|---|---|---|
| 1 | **`get_tag_by_asset` on Crane #1 and #5**; write the tag names into §0. Settle whether a third magnet tag exists and whether #2/#3/#4 are deliberately untagged | The unnamed-crane-tag caveat, and a possible coverage gap | one call |
| 2 | **`list_continuous_mode_tags`** — read mode rather than assuming it | Removes "assumed-continuous" from every report; closes spec §7.2 for the tags that matter | one call |
| 3 | **Measure *d*.** One 12-hour pull on a stationary tag, 99th-percentile displacement (§4a) | Makes criterion 4 gradeable and — with APPI settled — showable | one pull |
| 4 | **Test payload size** on one 24 h `get_asset_history` pull before running all twelve | Whether a full-fleet 24 h pull is possible at all | one query |
| 5 | ~~Generate a tag tool reference~~ — **done**, `tag-tools-reference.md`, 2026-09-02 | — | — |
| 6 | **Resolve `#1_21ee`'s mode-switch disconnects.** Compare 14 days of disconnects against the two job windows: all within ±30 min = scheduler artifact; any outside = tag fault | The ±30 min exclusion in §1b, which currently hides a possible real fault | one query |
| 7 | **Five cranes, three magnet tags** (§0). Which two are untagged, and is that intended? | A coverage gap that the fleet view hid | ask Simon |
| 8 | **Settle APPI retention and access**, and where the discipline output goes (spec §7.4) | The standing caveat on criterion 4 | ask Simon |

Items 1–4 are the ones that change what the monitor can grade. Items 7–8 are the ones only a human can answer.

---

## Validation status

**None of the above has been run against the live MCP.** The engineering-api MCP is not connected to any Claude session (`eng-api-mcp-connector-setup.md`); everything measured so far came from the `postgres-zlp` Grafana datasource, the 2026-08-25 registry export, and browser reads of ZPS. Every threshold here is derived from ZaiNar's documented values or observed data — **not** from real tag payloads.

**Rev 3 closes the data-source gap.** `tag-tools-reference.md` supplies a source for every criterion here, including the two rev 2 called sourceless. What remains unvalidated is the *values*: **no real tag payload has been seen**, so the first run of `koto-tag-check.md` is a validation run, not a health check — compare actual payloads against the thresholds and correct both files before anyone acts on a grade.

**Two things in this document are stated placeholders and must be labelled as such in any report that uses them:** the movement threshold (§4a — dual-threshold, neither value validated) and the reporting mode where `list_continuous_mode_tags` could not be reached (§1a). A third, the crane tag names, resolves with one call.

**The scope narrowing is the largest single improvement in rev 2**, and not because it saves calls. A definition that graded 63 unenumerated trackers could produce a number every morning and still miss that one of ten workers had never been tracked at all. Twelve named tags cannot hide that.
