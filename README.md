# ZaiNar Claude Code plugins

A plugin marketplace for internal Claude Code tooling. Add it once; install what you need.

## Install

```bash
# in Claude Code — add the marketplace (once)
/plugin marketplace add zainar/zainar-claude-plugins

# then install
/plugin install kajima-crane-site-survey@zainar-tools
```

Replace `zainar/zainar-claude-plugins` with the real `owner/repo` if it differs.

**Private repo?** That works — it uses your existing git credentials. If `/plugin marketplace add`
can't reach it, check `gh auth status` or that your SSH key is loaded, then retry.

**Updating.** There is no auto-update. When a new version lands:

```bash
/plugin marketplace update zainar-tools
```

---

## Available plugins

### `kajima-crane-site-survey` — v0.4.1

Five read-only checks for the Koto Pumping Station crane site
(`875ca2f3-64bf-4931-8d6a-fe0ea2b6c784`, account `Kajima-Koto`, env `zlp-prd-jpn`):

| Command | Answers |
|---|---|
| `/koto-reader-check` | Are the 29 commissioned readers healthy? **Runs the site sweep.** |
| `/koto-hub-check` | Are the hubs healthy — heartbeat, NTP, battery/RSSI relay, `hub_not_found`? |
| `/koto-tag-check` | Are the 12 monitored tags working now — battery, flags, coverage? |
| `/koto-tag-usage` | Were those tags in service, when did that stop, and why? |
| `/koto-crane-usage` | Were the LOADED/UNLOADED buttons pressed — and was the crane moving? |

Plus a `koto-site-reference` skill carrying the health definitions, the platform roll-up spec and the
crane-event payload review, loaded on demand.

> **This is a safety system.** Red-zone and proximity alerts warn workers standing under crane loads.
> A silent site is urgent, not a data-quality issue.

#### Before your first run — three things

**1. Your own ZLP API key, scoped `viewer` / `engineering:read`.**

```bash
export ZLP_PRD_JPN_API_KEY="zainar-..."
```

The plugin ships no credential and registers the engineering API as `zlp-prd-jpn`, reading the key
from that variable. **Scope matters more than it looks**: the credential this MCP normally carries has
**write + admin on production**, and the commands' read-only design is enforced by their
`allowed-tools` whitelists — which stop Claude calling a write tool, but do nothing about a
wrongly-scoped key being used elsewhere. The reader and hub checks warn in preflight if the key
resolves to write or admin.

**2. Mixpanel, if you want `/koto-crane-usage`.** It is the one command spanning two servers, and a
plugin cannot register an OAuth connector — add Mixpanel yourself, registered as `Mixpanel` (capital
M). The other four do not need it.

**3. Confirm both server names.**

```bash
claude mcp list   # expect: zlp-prd-jpn   AND (for crane usage) Mixpanel
```

Tools are prefixed `mcp__zlp-prd-jpn__` and `mcp__Mixpanel__`. **Registered under a different name,
the `allowed-tools` frontmatter silently fails to match and the command looks broken for no visible
reason** — that is the single most likely first-run problem. Rename the server rather than editing
prefixes across five files.

#### Two things that will surprise you

**State files are per-working-directory.** `.koto-reader-state.json`, `.koto-hub-state.json` and the
rest are written wherever you run the command. If more than one person runs these, you each build a
separate history and the `Change` lines are not comparable between you. **Agree on who runs them, or
on a shared working directory, before you start.** They are gitignored here deliberately — do not
commit them.

**Run `/koto-reader-check` first when anything looks wrong.** It is the only check that runs the site
sweep. A dark site presents as a dead hub, twelve dead tags, or an idle crane — and none of those is
the real finding.

#### Known limits, stated up front

- **Nothing has been validated against a live MCP payload.** Thresholds come from ZaiNar's documented
  SLAs, the 2026-08-25 registry export, and the `postgres-zlp` Grafana datasource. The first few runs
  test the thresholds as much as the site. **When a payload contradicts a file, the payload is right.**
- **These are checkpoints, not detectors.** The Grafana `device.reader.offline` rules from incident
  Gap #1 are the detector, last seen listed as *Investigating*.
- **Hub stability is not covered at all** — there is no `get_hub_metric_history`.
- **`/koto-crane-usage` will report zero.** Every `crane_status_change` event in prod so far is a
  controlled test; no operator has pressed the button since tracking went live 2026-09-08. A zero is
  **not** an idle crane — the event records presses, not crane state.

See the plugin's own `README.md` for the full picture, including the working-hours window and the
Saturday question.

---

## Adding a plugin to this marketplace

1. Drop the plugin directory under `plugins/`.
2. Add an entry to `.claude-plugin/marketplace.json` with `name`, `source` (the relative path),
   `description` and `version`.
3. Commit and push. Users pick it up with `/plugin marketplace update zainar-tools`.

**Bump `version` on every change.** Omit it and installs pin to the commit SHA, which makes "are we
on the same version?" unanswerable.
