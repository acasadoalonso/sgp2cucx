# sgp2cucx

Generate a **SeeYou Competition (`.cucx`)** file from a **Sailplane Grand Prix
(SGP)** competition on [crosscountry.aero](https://www.crosscountry.aero).

Given an SGP competition id, `sgp2cucx` pulls the competition, pilots, tasks, and
results and assembles a `.cucx` you can open in SeeYou Competition. The generated
`.cup` waypoint file carries both the **task definitions** and their
**turnpoints**, so tasks resolve back to the waypoint database.

## Install

```bash
pip install -r requirements.txt   # httpx
```

## Usage

```bash
# all scored days
python3 tools/make_cucx.py --comp-id 93

# a single day (1-based, as shown in the results) with an explicit output name
python3 tools/make_cucx.py --comp-id 93 --day 3 --out norway_day3.cucx
```

- `--comp-id` — the SGP competition id (required).
- `--day` — `ALL` (default) or a 1-based task day number. This controls **which
  task lines go into the `.cup` Related Tasks section**; the `contest.db` inside
  the `.cucx` always contains every day's task, pilots, and results.
- `--out` — output path (defaults to a name derived from the competition).

The script prints `wrote <path>` on success.

For a full walkthrough of the pipeline (fetch → normalize → build `contest.db` →
build the `.cup` → assemble the ZIP → verify), see [docs/PROCESS.md](docs/PROCESS.md).

## What's inside a `.cucx`

A `.cucx` is a ZIP containing:

| Member | Contents |
|--------|----------|
| `contest.db` | SQLite database (`application_id=1668637560`) — contest, classes, contestants, pilots, tasks, task points, results |
| `waypoint/<id>.cup` | Active waypoint file: the turnpoint list plus a `-----Related Tasks-----` section with one line per selected day |
| `uv.meta` | Class/aircraft metadata (TSV) |
| `tmptasks.meta` | CUP-format scratch tasks |

## Project layout

```
tools/
  make_cucx.py        CLI entry point
  cucx_bundle.py      fetch + normalize an SGP comp into a bundle dict
  cucx_db.py          build contest.db from the bundle
  cucx_package.py     build the .cup and assemble the .cucx ZIP
  cucx_geo.py         coordinate conversions (radians <-> DDMM.mmm)
  cucx_hash.py        content hash for contest_file registration
  cucx_schema.sql     seed schema (regenerate via gen_cucx_schema.py)
  gen_cucx_schema.py  one-shot schema derivation from pavullo.cucx
  pavullo.cucx        reference template produced by SeeYou
  tests/              unit + integration tests with captured comp-93 fixtures
src/SGP/sgp_api.py    crosscountry.aero fetchers (httpx)
.claude/skills/gen_cucx_from_sgp/
                      Claude Code skill that drives make_cucx.py interactively
```

## Tests

```bash
python3 -m pytest tools/tests/ -q
```

Tests use captured comp-93 fixtures (`tools/tests/fixtures/comp93/`) via a fake
fetcher, so they run offline with no network access.

## License

[MIT](LICENSE)
