# team-update

A small, deterministic tool for a biweekly team status email to leadership. YAML in, Gmail-safe HTML out, every cycle committed to git. An assistant can help write the narrative; the tool renders it.

Designed for small platform or enabling teams that serve several partner groups and want to stay visible upward without maintaining a dashboard. Everything organization-specific lives in a local `config.yaml`, so this repo carries no team, employer, or partner detail.

## What an update looks like

Each cycle is a single email with six sections, in order:

- **Headline** — one sentence, the single most important thing this cycle
- **Shipped** — 1–3 outcomes, phrased as what changed for whom
- **Needs attention** — one sentence with a specific ask and date (omitted when there's nothing to ask)
- **Metrics** — four numbers in a 2×2 grid with deltas vs the previous cycle
- **Lines** — one row per partner group: stage dot, current status, next milestone
- **Initiatives** — cross-group work with progress and target date

The template is intentionally boring: fixed 600px table, one font, no background fills, no charts. Colors are used only where they carry meaning (stage dots, delta direction, the attention label). Full rules in [SPEC.md](SPEC.md).

## Why it exists

Small enabling teams get judged on outcomes their partners feel, not on activity. A cadence that consistently shows outcomes, in the same shape every two weeks, does more for visibility than any tracker. The tool exists to make that cadence cheap enough to keep.

Design choices and their rationale are in [SPEC.md §8](SPEC.md).

## Build

The spec is the contract. Code is generated from it, per-phase, with an AI coding agent working against [CLAUDE.md](CLAUDE.md) and [SPEC.md](SPEC.md):

- **Phase 1** — YAML → HTML render pipeline, CLI, tests. Enough to send the first update by pasting the rendered HTML into a compose window.
- **Phase 2** — collectors that fill metric values from Jira and GitHub.
- **Phase 3** — a narrative-drafting prompt printed to stdout.

Phase prompts are in [SPEC.md §6](SPEC.md).

## Layout

```
team-update/
├── SPEC.md               # the contract
├── CLAUDE.md             # guidance for the coding agent
├── config.yaml           # local, gitignored: team + groups + metric definitions
├── updates/              # local, gitignored: one YAML + rendered HTML per cycle
├── src/team_update/
├── templates/update.html.j2
└── tests/
```

`config.yaml`, `repos.yaml`, `updates/`, and `.cache/` are gitignored — this repo stays employer-agnostic.

## Stack

Python, [uv](https://github.com/astral-sh/uv), Jinja2, Pydantic, Typer, pytest. No LLM in the render path.

## License

MIT.
