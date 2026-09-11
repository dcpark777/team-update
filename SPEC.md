# team-update — build kit

A small, deterministic tool for a biweekly team status email to leadership. YAML in, Gmail-safe HTML out, every sent cycle committed to git. No LLM in the render path; an assistant helps write the narrative, the tool renders it.

Designed for small platform/enabling teams that serve several partner groups and need to stay visible upward without maintaining a dashboard. Everything team-specific lives in `config.yaml`, so the repo carries no organization details.

---

## 1. Project structure

```
team-update/
├── CLAUDE.md                  # guidance for an AI coding agent (section 5)
├── SPEC.md                    # sections 2–4 of this file, kept in-repo
├── config.yaml                # team name, partner groups, metric definitions, links (section 2a)
├── pyproject.toml             # uv-managed; deps: jinja2, pyyaml, pydantic, typer
├── README.md                  # how to run a cycle (section 6)
├── templates/
│   └── update.html.j2         # the email (section 4 has the rules)
├── updates/
│   ├── 2026-09-24.yaml        # one file per cycle — the source of truth
│   ├── 2026-09-24.html        # rendered output, committed after send
│   └── ...
├── src/team_update/
│   ├── __init__.py
│   ├── cli.py                 # typer: init, new, render, delta, collect, preview
│   ├── config.py              # load + validate config.yaml
│   ├── model.py               # pydantic schema for a cycle YAML
│   ├── render.py              # YAML → HTML via Jinja2
│   ├── delta.py               # compute Δ vs previous cycle
│   └── collectors/
│       ├── __init__.py
│       ├── base.py            # Collector protocol: fill(metric) -> int | None
│       ├── jira.py            # issue counts via JQL (Phase 2)
│       └── github.py          # issue counts, releases, file-pattern checks (Phase 2)
└── tests/
    ├── test_render.py
    ├── test_delta.py
    └── fixtures/              # config + two sample cycle YAMLs
```

Dependencies: `jinja2`, `pyyaml`, `pydantic`, `typer`. Dev: `pytest`. No charting library — the biweekly has no chart by design.

---

## 2a. config.yaml — the only place with team-specific content

```yaml
team:
  name: Team Name              # used in subject line and header
  subject_prefix: "Team Name update"

recipients:                    # informational; the tool never sends mail
  to: [manager, director, vp, partner-directors]

groups:                        # partner groups you serve, in the fixed display order
  - Group A
  - Group B
  - Group C
  - Group D

metrics:                       # exactly 4; keys are stable across cycles
  - key: open_requests
    label: Open partner requests
    good_direction: down       # down | up | neutral
    source: jira               # jira | github | manual
    query: 'project = KEY AND type = Request AND status != Done'
  - key: critical_vulns
    label: Critical vulns (code + container)
    good_direction: down
    source: github
    query: { kind: issues, labels: [security, critical], state: open }
  - key: compliant_image
    label: Repos on compliant base image
    good_direction: up
    source: github
    query: { kind: file_match, path: Dockerfile, pattern: '^FROM registry\.example/base:' }
    total_from: repo_count
  - key: releases
    label: Releases this cycle
    good_direction: neutral
    source: github
    query: { kind: releases_in_window }

repos_file: repos.yaml         # list of {repo, group, kind} the github collector iterates

links:
  - { label: Request board, url: "https://..." }
  - { label: Past updates,  url: "https://..." }
  - { label: "#team-channel", url: null }
```

Metric ideas by what they signal (pick one per category for balance):
demand (open requests, requests received), responsiveness (median turnaround, % closed within SLA), risk (critical vulns, repos past EOL), adoption (repos on standard, models on platform, groups onboarded).

## 2b. Cycle YAML schema (`updates/<date>.yaml`)

```yaml
cycle:
  start: 2026-09-10
  end: 2026-09-24
  send_date: 2026-09-24

headline: >
  One sentence: the single most important thing this cycle.

shipped:                       # 1–3 items; outcome for whom, not activity
  - text: Outcome shipped, stated as what changed for which group
    owner: null                # name only for a specific callout
  - text: Another outcome
    owner: Teammate

attention: >                   # null when there is nothing to ask
  The blocker and the specific ask, with a date.

metrics:                       # values only; labels/direction come from config, Δ from delta.py
  open_requests: 9
  critical_vulns: 3
  compliant_image: 18
  releases: 14

lines:                         # one row per group, order from config
  - group: Group A
    stage: active              # active | onboarding | support | none  → dot color
    status: What we are doing with them · progress
    next: next dated milestone
  - group: Group B
    stage: support
    status: Support only · nothing open
    next: none scheduled

initiatives:                   # cross-group work; 1–3 rows
  - name: Initiative name
    progress: 6/16 items
    groups: [Group A, Group B]  # or [all]
    target: Nov 15
    rag: green                 # green | amber | red
    owner: null

notes: >                       # free text for you / the drafting assistant; never rendered
  Anything to remember when writing the narrative.
```

Validators: exactly the metric keys from config; `shipped` 1–3; `attention` optional; deltas never hand-entered.

---

## 3. CLI

```
team-update init             # scaffold config.yaml + first cycle from prompts
team-update new              # next cycle file from the last one: keeps lines/initiatives, blanks narrative
team-update collect          # fill metrics whose source != manual (Phase 2)
team-update delta            # each metric with Δ vs previous cycle and good/bad/neutral
team-update render           # write updates/<date>.html
team-update preview          # render and open in the default browser
team-update draft            # print a narrative-drafting prompt to stdout (Phase 3)
```

---

## 4. Template rules

Layout, in order: header line · headline · Shipped · Needs attention · metrics 2×2 · Lines · Initiatives · links footer.

- Fixed 600px table, centered. Nested `<table>`s for the metrics grid and any two-column rows. No flex, no grid, no `<style>` block — inline `style` attributes only, for Gmail and Outlook.
- Font: `Arial, Helvetica, sans-serif`. Body 14px, section labels 11px muted, headline 18px medium. One size hierarchy.
- No background fills. Structure via 1px `#E5E5E0` hairlines and weight.
- Colors, only where they mean something:
  - stage dots: active `#1D9E75`, onboarding `#EF9F27`, support `#B4B2A9`, blocked/red `#E24B4A`
  - deltas: good `#1D9E75`, bad `#E24B4A`, neutral `#6B6B66`
  - "Needs attention" label: `#BA7517`; the sentence itself is normal text
  - links: `#185FA5`
- Dots are small round `<td>` cells or a colored `●`, not emoji.
- Metric cell: label (11px muted), value (20px medium), delta inline right after (12px). `total_from` renders as `18 / 40`.
- `attention` null → paragraph omitted, no spacer. `owner` null → nothing appended.
- Lines table: group column `white-space:nowrap`; status wraps; "next:" muted.
- Subject line: `<subject_prefix> · Sep 10–24`.

Density guard: readable in full on one desktop Gmail screen. If not, cut content, not padding.

---

## 5. CLAUDE.md

```markdown
# team-update

Biweekly team status email for leadership.
Pipeline: updates/<date>.yaml + config.yaml → templates/update.html.j2 → updates/<date>.html.

## Rules
- The render path is deterministic. No LLM calls in src/.
- SPEC.md is the contract. Spec change first, then code.
- Everything organization-specific lives in config.yaml and repos.yaml.
  Never hardcode team names, group names, URLs, or queries in src/ or templates/.
- Metric deltas are computed by delta.py from the previous cycle, never hand-entered.
- Gmail-safe HTML only: 600px table layout, inline styles, no fills,
  no flex/grid, no <style>. See SPEC.md §4.
- Keep the template boring. A visual addition must answer
  "what does a reader do differently because of it."
- uv for everything. pytest before claiming done. Justify new deps.
- Commits: one per logical change. Do not push or open PRs.
```

---

## 6. Phased prompts for an AI coding agent

Run each as a fresh session in the repo with SPEC.md and CLAUDE.md present.

### Phase 1 — render pipeline (enough for the first send)

```
Read CLAUDE.md and SPEC.md. Build Phase 1 of team-update:

1. uv project per SPEC §1. Deps: jinja2, pyyaml, pydantic, typer; dev: pytest.
2. config.py: load and validate config.yaml (SPEC §2a). Exactly 4 metrics.
3. model.py: pydantic models for the cycle YAML (SPEC §2b); metric keys
   must match config.
4. delta.py: delta per metric vs the previous cycle file; classify
   good/bad/neutral from config good_direction. No previous cycle → None,
   rendered as "—".
5. render.py + templates/update.html.j2 implementing SPEC §4 exactly.
6. cli.py: init, new, delta, render, preview.
7. tests with fixture config + two cycles: delta math, null attention
   omitted, owner rendering, total rendering, and that HTML contains no
   <style>, no background-color, no display:flex/grid, and no string
   from config appears hardcoded in the template.
8. Render the fixture and print the path. Do not push.

Plan first, show the plan, then build.
```

Then manually: `team-update preview`, open in a browser, select-all, copy, paste into a Gmail compose window. Check on the Gmail mobile app before sending. Pasting from a browser preserves inline styles and avoids needing mail API access.

### Phase 2 — collectors

```
Read CLAUDE.md and SPEC.md. Implement `team-update collect`.

- collectors/base.py: Collector protocol; a registry keyed by source.
- collectors/jira.py: count issues for a JQL string. Auth via env
  JIRA_URL / JIRA_TOKEN. --dry-run prints JQL and count.
- collectors/github.py: iterate repos_file; support query kinds
  `issues` (open issues matching labels), `releases_in_window`
  (releases published inside the cycle), `file_match` (fetch a file at
  the default branch and regex-match). Auth via env GH_URL / GH_TOKEN
  (GitHub Enterprise base URL supported).
- Cache raw responses under .cache/<date>/ so re-runs are free and
  auditable. Log every query and raw count before writing to the YAML.
  Never overwrite a manual metric.
- Tests with recorded fixture responses; no network in tests.

Plan first. Do not push.
```

### Phase 3 — narrative assist (optional)

```
Implement `team-update draft`: print a prompt containing this cycle's
delta output, the previous cycle's headline and shipped items, and the
`notes:` field. The user pastes it into an assistant to get a proposed
headline and shipped bullets, then edits the YAML. The tool itself makes
no LLM calls.
```

---

## 7. Cadence

| When | What |
|---|---|
| Two days before send | Ask teammates for inputs (template below). `team-update new`. |
| Send-day morning | Fill YAML. `collect` or enter metrics by hand. `delta`, `preview`. |
| Send time (fixed) | Paste into Gmail. Same recipients, subject from the tool. Send. |
| After send | Commit `updates/<date>.yaml` + `.html`. |

Teammate ask, identical every cycle:

> Team update goes out <day>. By <day-2> EOD can you send me:
> – 1–3 shipped items, phrased as outcome for whom (not "merged X")
> – status of your track: the metric values you own, anything blocked
> – anything you want leadership to know or decide

First send only: one sentence at the top saying what this is and its cadence. Consider showing the first one to your manager before it goes to the full list.

---

## 8. Design rationale (why it looks like this)

- Wins first, then the ask, then numbers: a reader who stops after one screen still gets the whole picture.
- Same four metrics in the same order every cycle; the delta column is only meaningful when the metrics don't move.
- Organized by initiative and partner group, not by person. Names appear only on specific callouts, so the sender is seen as running a team rather than reporting on individuals.
- No chart in the biweekly. Deltas carry the trend; save charts for a quarterly review.
- Bad numbers stay red and unexplained unless the headline covers them. Over-explaining a bad number draws more attention than the number.
- Partner rows must say something concrete even when there is no active engagement. "Support only · nothing open" is a status; "N/A" is an afterthought.
