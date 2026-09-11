Read CLAUDE.md and SPEC.md. Build Phase 1 of team-update.

templates/update.html.j2 already exists — do not modify it. Its top
comment block is the contract for the view model render.py must build.

1. uv project per SPEC §1. Deps: jinja2, pyyaml, pydantic, typer; dev: pytest.
2. config.py: load and validate config.yaml (SPEC §2a). Exactly 4 metrics.
3. model.py: pydantic models for the cycle YAML (SPEC §2b); metric keys
   must match config.
4. delta.py: delta per metric vs the previous cycle file; classify
   good/bad/neutral from config good_direction. No previous cycle → None,
   rendered by omitting the delta span in the template.
5. render.py: load config + current cycle YAML + optional previous cycle
   YAML, build the view model exactly as documented at the top of
   templates/update.html.j2, and render to updates/<send_date>.html.
   Stage colors, RAG colors, delta signs/colors, and groups_display are
   render.py's job.
6. cli.py: init, new, delta, render, preview.
7. tests against updates/2026-09-11.yaml + one fabricated previous cycle:
   delta math, null attention omitted, owner rendering, total rendering,
   and that HTML contains no <style>, no background-color, and no
   display:flex/grid.
8. Render 2026-09-11 and print the path. Do not push.

Plan first, show me the plan, then build.
