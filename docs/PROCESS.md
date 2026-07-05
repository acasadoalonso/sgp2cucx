# How `gen_cucx_from_sgp` works — the full process

This document walks through the entire pipeline the `gen_cucx_from_sgp` skill
drives, from an SGP competition id to a SeeYou-importable `.cucx`. It is the
human companion to `.claude/skills/gen_cucx_from_sgp/SKILL.md` (which tells
Claude how to run it) and `.claude/skills/gen_cucx_from_sgp/references/cucx_format.md`
(the on-disk format reference).

## Two entry points

The same code path is reached two ways:

- **The skill (interactive).** In Claude Code, the `gen_cucx_from_sgp` skill
  prompts for the competition number and the day, then runs the CLI and verifies
  the output. This is the intended day-to-day path.
- **The CLI (direct).** `python3 tools/make_cucx.py --comp-id <ID> [--day ALL|N]
  [--out FILE]`. The skill is a thin, guided wrapper over exactly this command.

## Inputs

| Input | Meaning | Default |
|-------|---------|---------|
| **competition number** (`--comp-id`) | the SGP competition id (e.g. `93`) | required |
| **day** (`--day`) | `ALL`, or a 1-based day number matching the "Day N" in the results | `ALL` |
| output (`--out`) | output path | derived from the competition short name |

The **day** selection controls **only which task lines are written into the
`.cup` Related Tasks section**. The `contest.db` inside the `.cucx` always holds
every scored day's task, contestants, and results — narrowing to one day never
drops data from the database, it just focuses the waypoint file on that day.

## The pipeline

```
comp-id ─▶ sgp_api ─▶ cucx_bundle ─▶ ┌─ cucx_db ──────▶ contest.db ─┐
 (SGP)     (fetch)   (normalize)     │                              ├─▶ cucx_package ─▶ .cucx (ZIP)
                                     └─ cucx_package.build_cup ▶ .cup┘
```

`tools/make_cucx.py` is the orchestrator: `generate()` calls
`cucx_bundle.build_bundle()`, then `cucx_package.assemble_cucx()` (which itself
calls `cucx_db.build_contest_db()` and `build_cup()`).

### Step 1 — Fetch from SGP (`src/SGP/sgp_api.py`)

`build_bundle` pulls the raw data straight through `sgp_api` (plain `httpx`
calls to crosscountry.aero — no MCP runtime needed):

- `fetch_competition(comp_id)` — competition metadata + the day index (each day
  has a `day_id`, `date`, and `type_label`).
- `fetch_pilots(comp_id)` — the pilot field.
- `fetch_task(comp_id, day_id)` — per day: name, type, length, altitudes,
  airfield, and decoded turnpoints.
- `fetch_day_results` / `fetch_total_results` — per day and cumulative scoring.

### Step 2 — Normalize into a bundle (`cucx_bundle.build_bundle`)

Raw SGP JSON is normalized into one `bundle` dict — `{comp, airfield, pilots,
tasks}` — that the rest of the pipeline consumes. This is also where SGP's data
quirks are absorbed so downstream code stays simple:

- **Empty/future days** (0 turnpoints, `length=None`) are **skipped entirely** —
  they never become tasks.
- **`task_number`** is assigned 1-based over the days that *do* have a task, so
  it matches the "Day N" a user sees in the results. This is the number `--day`
  filters on.
- **Ranks** that aren't integers (e.g. `"DNS"`) are coerced to `None`.
- **Result status** is mapped to `official` / `preliminary`, or `practice` for
  days flagged as practice.
- The **airfield** is captured once, from the first task's start turnpoint.
- Distances are converted to metres; turnpoint lat/lon stay in degrees at this
  stage.

### Step 3 — Build `contest.db` (`cucx_db.build_contest_db`)

A fresh SQLite database is created from the seed schema (`tools/cucx_schema.sql`,
which carries `application_id=1668637560`, `user_version=3`, and the shared seed
tables including the SGP scoring script). Rows are inserted following the
foreign-key chain:

1. **location** → **contest** (dates, country, time zone).
2. **class** (+ a class-level `warning` row and the `class_meta` scoring
   defaults copied from the reference template).
3. **contestant** + **pilot** per pilot, keyed by competition number so results
   can link back. FLARM ids, registration, and FAI ranking id are carried
   through.
4. Per task: a **task** row (referencing the `Sailplane_Grand_Prix` scoring
   script), then for each turnpoint a **point** row and a **task_point** link.
   - `point.latitude/longitude` are stored in **radians**.
   - Per-leg **distance** (haversine) and **course_in/course_out** bearings are
     computed and stored.
   - Observation-zone fields (`type`, `oz_type`, `oz_line`, `oz_angle1`) are
     derived from each turnpoint's role (start / point / finish) and zone shape.
5. Per result: a **result** row (points, totals, rank, times converted from
   millis to UTC datetimes, speed, distance). Results whose competition number
   has no matching contestant — e.g. a stray practice-day entrant — are skipped.

Ids come from a single monotonic allocator; they only need to be unique within
the file.

### Step 4 — Build the `.cup` waypoint file (`cucx_package.build_cup`)

The `.cup` has two parts:

- **Waypoint list** — every turnpoint across all tasks, de-duplicated by name,
  each written in `DDMM.mmm` + hemisphere format (converted from degrees).
- **`-----Related Tasks-----`** — one line per *selected* day (per `--day`):
  the task label followed by its ordered turnpoint names. Because every name in
  a task line also appears in the waypoint list above, SeeYou resolves the task
  geometry back to the waypoint database. **This is the "tasks + waypoints in the
  `.cup`" guarantee.**

`--day ALL` emits every day's task line; `--day N` emits only day N's.

### Step 5 — Assemble the `.cucx` (`cucx_package.assemble_cucx`)

The `.cucx` is a ZIP with four members:

| Member | Built by | Contents |
|--------|----------|----------|
| `contest.db` | Step 3 | the SQLite competition database |
| `waypoint/<id>.cup` | Step 4 | the active waypoint file |
| `uv.meta` | this step | TSV `<class_id>\t<N>\t<aircraft_type>` |
| `tmptasks.meta` | this step | CUP-format scratch tasks header |

The `.cup` is also **registered in the `contest_file` table** with `active=1`,
`format='waypoint/cup'`, its byte `size`, and its content **hash =
`base64(sha256(bytes))` with `=` padding stripped**. This registration is what
makes SeeYou treat the `.cup` as the contest's active waypoint file.

### Step 6 — Verify

The generated file is a ZIP, so verification is cheap and should always be run
before claiming success:

- unzip and confirm the four members are present;
- open `contest.db` and check `PRAGMA integrity_check` → `ok` and
  `PRAGMA application_id` → `1668637560`;
- confirm the task count and the `.cup` Related Tasks lines match the `--day`
  selection.

The one thing not machine-checkable is whether **SeeYou Competition** actually
opens the file — opening it in SeeYou closes that loop.

## Example

```bash
# every scored day of SGP competition 93
python3 tools/make_cucx.py --comp-id 93
# → wrote norway_sgp.cucx   (contest.db has all days; .cup lists every day's task)

# just day 3 (Race1), custom name
python3 tools/make_cucx.py --comp-id 93 --day 3 --out norway_day3.cucx
# → contest.db still has all days; .cup Related Tasks holds only Race1
```

## Tests

`python3 -m pytest tools/tests/ -q` runs the whole pipeline offline against
captured comp-93 fixtures (`tools/tests/fixtures/comp93/`) via a fake fetcher,
covering the bundle normalization, DB build, `.cup` waypoint + task-line
emission, day filtering, the content hash, the schema, and an end-to-end
`generate()` integration test.

## Regenerating the schema

`tools/cucx_schema.sql` is derived once from the reference template
`tools/pavullo.cucx` (a real `.cucx` produced by SeeYou) via
`python3 tools/gen_cucx_schema.py`. Re-run that only if the SeeYou schema needs
re-deriving from a newer template.
