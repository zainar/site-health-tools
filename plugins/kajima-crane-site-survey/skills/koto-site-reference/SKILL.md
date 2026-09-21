---
name: koto-site-reference
description: Ground truth for the Kajima Koto Pumping Station crane site (zlp-prd-jpn) — the reader, hub and tag health definitions, the site roster and its ghost records, the monitored tag set, and the known payload defects. Use when interpreting output from /koto-reader-check, /koto-hub-check, /koto-tag-check, /koto-tag-usage or /koto-crane-usage, when asked what counts as a healthy reader, hub or tag at Koto, when a Koto grade or threshold needs justifying or changing, when reconciling reader or tracker counts at that site, when working on crane load/usage instrumentation, or when adding an alert threshold or run frequency to any Koto monitor.
---

# Koto crane site — reference

Site `Koto_Pumping_Station_Crane` · `875ca2f3-64bf-4931-8d6a-fe0ea2b6c784` · account `Kajima-Koto` ·
env `zlp-prd-jpn` (WIFI_RT) · NID 16. Sister site: `Koto_Pumping_Station_Excavation`
(`e0a6a862-56cc-4281-ac18-0d3fa3cdc4bc`, NID 4) — **out of scope, and its Z expectations are
inverted**, so never copy a Crane Z rule to it.

**This is a safety system.** Red-zone and proximity alerts warn workers standing under crane loads.
Treat a silent site as urgent, not as a data-quality issue.

## Read this before quoting any number from it

Every threshold in this plugin comes from ZaiNar's documented SLAs, the 2026-08-25 registry export,
or the `postgres-zlp` Grafana datasource. **None has been validated against a live MCP payload.**
When a real payload contradicts a document here, the payload is right and the document needs
correcting. Say so rather than grading against a number the site has already disproved.

## The four rules that are wrong most often

State these whenever they bear on an answer. Each one has already produced a real error.

**1. ⚫ UNKNOWN is a grade, not a failure to grade.** Everything these tools see is cloud-side.
During the May 2026 outage all 30 readers were alive and answering local pings while every cloud
view was dark. A site UNKNOWN is a different fact from a site UNHEALTHY, and both differ from
HEALTHY. Conflating the first two is what let a 13-day outage stay hidden. A dark reader means
"not reaching the cloud" — never "the reader is dead". An unrun check is `Not checked`, never a pass.

**2. The denominators are 29 readers and 12 tags. Never the registry totals.** The registry returns
48 anchor records; 19 are ghosts with no surveyed x/y (`r20`–`r30`, `r33`–`r39`, `r67`). The filter
is **positional, not temporal** — a reader surveyed in and later gone dark keeps its coordinates.
Do not filter on `readers.status`: all 129 readers in the entire production database are `ACTIVE`
and the field is unmaintained. Spec expectation is 30 readers, so **one is unaccounted for** — say
so rather than rounding. For tags the monitored set is 12 named tags; the ZPS list's 31 and the
registry's 63 are hygiene lines, never denominators.

**3. Resolve tags by asset UUID, never by name prefix.** Prefixes mismatch the wearer for C05
(`A5_`), C06 (`B1_`), C08 (`B2_`) and C09 (`B3_`) — four of ten. Prefix matching mis-attributes
nearly half the monitored set, and a mis-attributed finding names the wrong person.

**4. `last_heartbeat` must not grade anything, on any device type.** It is a humanised string
(`"10h ago"`, `"51m ago"`, `"5.4h ago"`) with undocumented unit-boundary behaviour, and the tag tool
reference says outright *"Do NOT use for health checks"* — it shows stale values even while a tag is
actively reporting. Reader and hub stability come from `get_anchor_metric_history`; tag liveness
comes from the newest `locationTime`. If you are parsing prose to grade stability, you are on the
wrong source. A parse miss that defaults to healthy is the worst available failure here.

## Cadence and escalation are not the commands' job

**No command in this plugin decides when it runs, who hears about it, or what escalates.** They
grade, report, and hand back an explicit `Not checked` list. Cadence, delivery and any
confirm-before-paging ladder belong to whatever invokes them — the Agent Scheduler task, not these
files. The whole package is handed to the scheduler with no cadence baked in, so there is one place
to set it and no chance of a command and its scheduler disagreeing about what fires.

If asked to add an alert threshold or a run frequency to a command, put it in the scheduler task
instead and say why.

## Read-only, and three tools that look like diagnostics

Never call a write tool against this site, whatever the finding and however obvious the fix. The
production credential carries **write + admin**; `confirm: false` guards against a mistaken call,
not a wrongly-scoped key. Propose remediation and let a human who knows what is on site that hour
execute it — a crane may be under load.

- **`locate_anchor` / `locate_and_fetch` / `start_reader_survey`** are `engineering:write` and
  actively perturb the positioning system. They are what you would reach for to answer "can this
  reader still range?" Answer that from `get_anchor_ranging_history`.
- **`ping_tag`** transmits to the tag. It is what you would reach for to answer "is this tag alive?"
  Answer that from location history.
- **`reboot_tag_host`** reads like a tag tool and is not — it reboots tag management *on the hub*,
  taking out every tag on it at once. The Confluence spec files it under *Tag Actions*, which is how
  the name-similarity mistake happens.

Also never `set_tag_motion_sensitivity`: it is the natural-looking fix for a tag failing the movement
test, and it silently rewrites what every future run measures.

## Correlate before listing findings

The Actions backlog baseline is ~38 raw findings collapsing to ~10 distinct. Before writing a single
per-device finding, ask in order: is this the whole site · is this a cluster on one AP · is this the
afternoon signature (daytime JST, peaking ~16:00, cohorts of 10 then 3 then 11, nothing overnight →
name the cellular modem) · is this a hub finding wearing a reader costume (`0x0400 hub_not_found`) ·
is this a dead tag fleet rather than a hub fault. Correlated instability is **one** site finding.
Thirty findings is how the one that matters gets buried.

## Readers and hubs are separate checks, and the correlation between them is not automatic

`/koto-reader-check` and `/koto-hub-check` grade different layers of the same failure. **A dark site
presents as a dead hub; a hub fault presents as N silent readers.** Neither command can see the
other's data, so each names the other when a finding points across the boundary and says plainly that
it did not check it.

**`/koto-reader-check` is the one that runs the site sweep** (`get_site_last_locate`,
`get_site_health`, `list_stale_anchors`, the coherency audit) and writes `live_readers_reporting` and
`grade` to `.koto-reader-state.json`. The hub check reads that file for context. Run the reader check
first when anything looks wrong, and when interpreting a hub report, check whether it had site
context at all or ran blind.

## The reference files

Read the one that bears on the question; do not load all five.

| File | Read it when |
|---|---|
| `references/reader-health-definition.md` | Grading or justifying reader health — the five criteria, the 89-call budget, the three conflicting heartbeat thresholds |
| `references/hub-health-definition.md` | Grading hub health, or interpreting `/koto-hub-check` — and note criterion 5 (stability) **has no tool at all**; `get_hub_status` is point-in-time |
| `references/tag-health-definition.md` | Grading tags — the 12-tag roster, the two axes, the movement threshold problem, APPI |
| `references/get-site-reader-health-spec.md` | Asking the platform team for a site roll-up, or writing SQL against `device_metrics_history` — validated queries and field provenance |
| `references/crane-status-change-payload-review.md` | Anything about crane load/usage instrumentation, the Mixpanel event, or interpreting `/koto-crane-usage` output |

## Standing items — carry these until someone closes them

- **One reader unaccounted for** — 29 live vs 30 expected.
- **Hub roster: 5 counted, 0 enumerated.** `list_hubs` settles it in one call; check for `hub_b591`
  first. Until it resolves, the hub-silent P2 severity is conditional — the site demonstrably
  produces locations while its only named hub (`hub_b5c7`) has been dark 30+ days.
- **`r40_d23c` and `r41_d2d6` share one coordinate** (18760, 15040, 4500) — no geometric diversity
  for multilateration on that edge. Survey copy-paste or genuinely co-located, unchecked.
- **Three live readers report no firmware version** — `r45_b584`, `r53_da75`, `r59_d2d8`.
- **C05 (`A5_kps_ae92`) has never been tracked** — silent since 22 June 2026, two months before the
  crew was tagged. 10% of the monitored crew. It surfaced only because someone checked each worker
  individually, which is why these checks enumerate rather than sample.
- **Is Subcon C still on site?** As of 2026-09-08 all ten worker tags were dark 6–18 days while the
  crew showed 0 onsite. Demobilised, or ten workers untracked on a live crane site — opposite
  urgencies, not separable from this data. One question to a human settles it.
- **Grafana alert rules from incident Gap #1** — shipped, or still *Investigating*? These commands
  are checkpoints, not detectors; their value depends on the answer.
- **Working hours are settled: 08:00–18:00 JST, Mon–Fri** (Misha, 2026-09-16). One window across
  every check, replacing the three previously in play. **40 intervals of 15 min.** Weekends are
  outside it and are not graded; weekend activity is reported as an out-of-hours line rather than
  filtered away.
- **Saturday is excluded for now, and it is a deferral rather than a judgement** — crews commonly
  do work Saturdays on Japanese construction sites. It is blocked on one unanswered question: the
  device mode scheduler is documented as *"skipped on JST weekends"*, so on Saturday the fleet may
  sit in `on_motion`, whose tolerances are 4× looser than `continuous`. Grading a Saturday against
  the continuous tier would manufacture ERRORs on a healthy fleet every week. **Confirm with whoever
  owns `koto-crane-daily.md` §11 whether "weekends" means Sat+Sun or Sun only**, then widen the
  window across the package together. Readers and hubs are powered continuously and have no mode,
  so the blocker is entirely on the tag side. **When reading any report: a thin Saturday is not a
  finding.**
- **`KotoCrane_#1` was left flagged LOADED** by the 10 Sep test and never unloaded. Check whether
  that state still stands and whether it feeds red-zone alerting.

## Crane usage — `/koto-crane-usage`, and what it can honestly say

Crane usage is covered, but it answers a narrower question than its name suggests. It reads
`crane_status_change` from Mixpanel project `3369288` and **cross-checks against ZLP crane-tag
movement, which is required rather than optional** — without it, "nobody pressed the buttons" and
"the crane did not move" produce an identical row and carry opposite actions.

**The event records button presses, not crane state.** Presses are a lower bound that reads zero when
crews forget; movement reads high when a crane idles under load. Neither is a proxy for the other,
and nothing instruments the hook, so *was the crane actually carrying a load* has no answer at all.

**Every press in prod to date is a controlled test by Misha — zero operator presses since tracking
went live 2026-09-08.** So today the report's real output is *"the feature is unused"*, which is a
site conversation rather than an engineering one. Never report that zero as an idle crane.

Two defects shape every figure: the tracker fires **1–2 events per press with a nondeterministic
value** (so never count raw events and never divide by two — cluster within 10 s and take the first
of each cluster), and there is no `action` property, so direction is inferred. Dwell is only as good
as unload discipline; print unpaired loads beside any median.

## APPI

Per-person movement and usage output covers identifiable subcontractors. **Engineering-internal
only** — not for a customer-facing artifact, not the Kajima-facing report. Retention and access are
undecided; say so in every report that carries such a figure. A bench-tag claim needs ≥ 3 working
days at ≥ 80% coverage plus `r95` at the noise floor, and **BUG-432** (tags listed onsite with no
historical trail) produces a platform artifact indistinguishable from a bench tag in that view.
Telling Kajima a worker left their tag on a bench when the platform invented the record, or when the
battery simply died, is a serious and personal accusation to get wrong.
