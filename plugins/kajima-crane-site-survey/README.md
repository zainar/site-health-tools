# Kajima crane site survey

Read-only checks for the Kajima **Koto Pumping Station crane site**
(`875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`, account `Kajima-Koto`, env `zlp-prd-jpn`), plus the
definitions behind every grade they produce.

> **This is a safety system.** Red-zone and proximity alerts warn workers standing under crane loads.
> A silent site is urgent, not a data-quality issue.

## The six checks

| Command | Answers | Arguments |
|---|---|---|
| `/koto-site-check` | **All three health checks in order, cross-checked into one site grade.** Start here. | `check` · `deep` · `weekly` |
| `/koto-reader-check` | Are the 29 commissioned readers healthy? **Runs the site sweep.** | `check` · `deep` · `weekly` · `triage` |
| `/koto-hub-check` | Are the site's hubs healthy — heartbeat, NTP, battery and RSSI relay, `hub_not_found`? | `check` · `triage` · `weekly` |
| `/koto-tag-check` | Are the 12 monitored tags working right now — battery, flags, coverage? | `check` · `triage` · `weekly` |
| `/koto-tag-usage` | Were those tags in service, when did that stop, and why? | any date range |
| `/koto-crane-usage` | Were the LOADED/UNLOADED buttons pressed — and was the crane moving? | any date range |

Plus a `koto-site-reference` skill Claude loads on demand: the three health definitions, the platform
roll-up spec, and the crane-event payload review.

**Use `/koto-site-check` for a whole-site picture.** It runs the three health checks in the order
their dependencies require — reader, then tag, then hub — and applies the five cross-gates none of
them can apply alone, because none can see another's data. It defines no thresholds of its own.

**Running one on its own is still fine**, but mind the order: `/koto-reader-check` is the only check
that runs the site sweep, and a dark site presents as a dead hub, twelve dead tags, or an idle crane —
none of which is the real finding.

## Nothing here is scheduled

**No command decides when it runs, who hears about it, or what escalates.** They grade, report, and
hand back an explicit `Not checked` list. There are no alert ladders, no suppression rules, no
paging, and no cadence prescriptions anywhere in the package.

That is deliberate: **cadence and delivery are the Agent Scheduler's to set**, so there is one place
to change them and no chance of a command and its scheduler disagreeing about what fires. Hand the
whole package over and let the scheduler decide how often each check runs and where findings go.

What each command does provide, because the scheduler needs it: a `STATE` block for run-to-run
deltas, a `Change` line, and an explicit statement of what it did *not* check.

## Readers and hubs are separate, and the correlation is manual

They grade different layers of the same failure. A dark site presents as a dead hub; a hub fault
presents as N silent readers. Neither command can see the other's data.

So each one names the other when a finding crosses the boundary, and says plainly that it did not
check it. `/koto-reader-check` writes `live_readers_reporting` and `grade` to
`.koto-reader-state.json`; `/koto-hub-check` reads that file for site context and reports when it had
none. **If you schedule them, run the reader check first.**

## Setup

**1. A viewer-scoped ZLP API key.** The plugin registers the engineering API as `zlp-prd-jpn` and
reads the key from an environment variable:

```bash
export ZLP_PRD_JPN_API_KEY="zainar-..."
```

**Scope it `viewer` / `engineering:read`, not the default.** The credential this MCP normally carries
has **write + admin on production**, and `confirm: false` dry-run defaults protect against a mistaken
call, not against a wrongly-scoped key. The reader and hub checks verify scope in preflight and warn
if it resolved to write or admin.

**2. Mixpanel, connected yourself.** `/koto-crane-usage` is the one command spanning two servers, and
the plugin can only supply one. Mixpanel is an OAuth connector, not something a plugin registers with
an API key — add it separately, registered as `Mixpanel` (capital M). The other four do not need it.

**3. Confirm both server names.**

```bash
claude mcp list   # expect: zlp-prd-jpn   AND (for crane usage) Mixpanel
```

Tools are prefixed `mcp__zlp-prd-jpn__` and `mcp__Mixpanel__`. Registered under a different name, the
`allowed-tools` frontmatter **silently fails to match** and the command looks broken for no visible
reason. Rename the server rather than editing prefixes across five files.

## Read-only — what actually enforces it

Each command declares an explicit `allowed-tools` whitelist containing only read-scoped MCP tools,
and that is why these ship as commands rather than skills. **But the allowlist is not the whole
guardrail, and an earlier version of this README overstated it.**

An `allowed-tools` list constrains which *MCP tools* Claude may call. It does not constrain a shell.
These commands need `date` to print the JST/PT clocks, so `Bash` is whitelisted — **scoped to
`Bash(date:*)` and `Bash(TZ=*)`**, precisely so it cannot be used to reach the engineering API
directly with an HTTP method the allowlist would have refused. Unscoped `Bash` would have made the
read-only claim decorative.

**The other half is the key's scope**, which is why every command now calls `who_am_i` in preflight
and **halts** on a write or admin key rather than warning. A shell plus an admin production
credential is not a governance note to carry in the report; it is a reason not to run.

Three tools look like diagnostics and are writes: `locate_anchor` / `start_reader_survey` perturb the
positioning system, `ping_tag` transmits to the tag, and `reboot_tag_host` reads like a tag tool but
takes out tag handling for every tag on a hub. None appears in any whitelist, and no Mixpanel
`Create-*` / `Update-*` / `Delete-*` / `Edit-*` tool does either. Remediation is proposed in the
report; a human who knows what is on site that hour executes it.

## Before you act on a grade

**Nothing in this plugin has been validated against a live MCP payload.** Every threshold comes from
ZaiNar's documented SLAs, the 2026-08-25 registry export, or the `postgres-zlp` Grafana datasource.
The registry export carries no timestamps at all. **The first few runs test these thresholds as much
as they test the site** — when a payload contradicts a file here, the payload is right and the file
needs correcting.

Four more things worth knowing before the output surprises you:

- **These are checkpoints, not detectors.** The Grafana `device.reader.offline` rules from incident
  Gap #1 are the detector, and were last seen listed as *Investigating*. A checkpoint on top of
  working alerting is a good design; one standing in place of it is slower than what the platform
  already does for free.
- **Hub stability is not covered at all.** There is no `get_hub_metric_history`. A hub that flapped
  eleven times this morning and is up right now reads identical to one solid for a week.
  `/koto-hub-check` prints that in `Not checked` every run rather than quietly scoring it green.
- **Everything is cloud-side.** In the May 2026 outage all 30 readers answered local pings while
  every cloud view was dark. ⚫ UNKNOWN ≠ UNHEALTHY ≠ HEALTHY.
- **`/koto-crane-usage` will report zero.** Every `crane_status_change` event in prod to date is a
  controlled test by Misha — zero operator presses since tracking went live on 2026-09-08. Today the
  report's honest output is *"the feature is unused"*, which is a site conversation, not an
  engineering one. **A zero is never an idle crane**: the event records button presses, not crane
  state, which is why the ZLP movement cross-check is required rather than optional.

## Working hours — one window

**08:00–18:00 JST, Monday–Friday.** Settled 2026-09-16; it replaces the three windows previously in
play (08:00–18:00 Mon–Sat, 07:00–19:00 Mon–Fri, 09:00–18:00 Mon–Sat) and applies to every check in
the package. 40 intervals of 15 minutes. **Weekends are outside it and are not graded**, but weekend
activity is reported as an out-of-hours line rather than filtered away.

### Saturday is excluded for now — and it is one edit to add back

**This is a deferral, not a judgement about whether crews work Saturdays.** They commonly do on
Japanese construction sites, and excluding Saturday means a genuine Saturday shift shows up as an
out-of-hours line rather than as coverage. That is why nothing filters weekend activity away.

It is deferred because of one unanswered question, and the blocker sits entirely on the tag side:

**The device mode scheduler is documented as *"skipped on JST weekends"*.** It puts tags into
`continuous` at 08:00 JST and `motion/OFF` at 18:00 JST on weekdays. If "weekends" includes Saturday
— and the phrase conventionally does — then on Saturday the fleet sits in `on_motion`, whose
tolerances are **4× looser**: warn > 15 min and error > 60 min, against continuous's 5 and 15.
Grading a Saturday against the continuous tier would manufacture ERRORs on a perfectly healthy fleet,
every Saturday, forever.

**To add Saturday back**, confirm with whoever owns `koto-crane-daily.md` §11 whether "weekends"
means Sat+Sun or Sun only. Then:

- **Scheduler does run Saturdays** → change `weekday() > 4` to `> 5` in `/koto-tag-check` §6 and
  update the window text across the package. Nothing else is needed.
- **Scheduler skips Saturdays** → Saturday additionally needs the `on_motion` tier applied to it, and
  `list_continuous_mode_tags` becomes mandatory rather than a fallback on Saturday runs.

Readers and hubs are powered continuously and have no reporting mode, so neither would need changing.

**Meanwhile, when reading any report: a thin Saturday is not a finding.**

## Still open

The **hub roster is unresolved** — 5 counted, 0 enumerated, and the only nameable hub has
been dark 30+ days while the site produced locations. Until `list_hubs` settles it, the hub-silent P2
severity is conditional. And **`KotoCrane_#1` was left flagged LOADED** by the 10 Sep test and never
unloaded; worth clearing if that state feeds red-zone alerting.

## Contents

```
kajima-crane-site-survey/
├── .claude-plugin/plugin.json
├── .mcp.json                      # registers zlp-prd-jpn; Mixpanel is connected separately
├── commands/
│   ├── koto-reader-check.md       # 29 readers, five criteria, + the site sweep
│   ├── koto-hub-check.md          # hubs, five criteria (criterion 5 has no tool)
│   ├── koto-tag-check.md          # tag device health and activity, two axes, never merged
│   ├── koto-tag-usage.md          # in-service timeline and dropout analysis
│   └── koto-crane-usage.md        # load-button presses + required movement cross-check
└── skills/koto-site-reference/
    ├── SKILL.md                   # site facts, the rules that get broken most, standing items
    └── references/
        ├── reader-health-definition.md          # rev 2, 2026-09-01
        ├── hub-health-definition.md             # rev 3, 2026-09-01
        ├── tag-health-definition.md             # rev 3, 2026-09-02
        ├── get-site-reader-health-spec.md       # platform ask: 89 calls → 1
        └── crane-status-change-payload-review.md # prod-verified, defect open
```

**Superseded, and not included** — do not install these from anywhere else:
`koto-crane-check-command.md` and `koto-hub-check-command.md` (their content is now in
`/koto-reader-check` and `/koto-hub-check`), `koto-tag-check-command.md` (replaced by
`/koto-tag-check`), and `crane-load-event-tracking-spec.md` (written before instrumentation existed;
the event now behaves differently than proposed).

---

Source: the *Kajima site survey* project. Site facts current to 2026-09-16. Version 0.6.0.
