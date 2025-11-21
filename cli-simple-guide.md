# CLI Tools Explained Simply

## What it is
The backend ships with a small Typer-based command line app (`python -m app.cli`) that lets custodians check on Marova without using the admin dashboard. Think of it as a text console for quick diagnostics.

## How to run it
1. Open a terminal in `backend/`.
2. Activate the Python environment used for the API.
3. Launch the console:
   ```bash
   python -m app.cli --help
   ```
   You will see the available commands: `code-state`, `code-stats`, and `code-auto-sleep`.

## Command overview

### `code-state`
- Prints Marova’s current vitals (hunger, metabolism, dream energy, mood, sleep info) in a formatted table.
- Optional flags like `--field hunger --field dream_energy` let you focus on specific gauges.
- Timestamps are automatically converted to IST for easy reading.

### `code-stats`
- Shows a table of recent telemetry samples: hunger, dream energy, toxicity, dream debt, etc.
- Use `--limit 10` to control how many rows you see (latest first).
- Add `--metric dream_energy --metric toxicity_level` to only show the columns you care about.

### `code-auto-sleep`
- Displays whether auto-sleep is currently enabled and, if a nightly session is running, the start/end times.
- `--enable` turns the scheduler on, `--disable` turns it off. The command echoes the new status so you know the change stuck.

## Quality-of-life helpers baked in
- Tables are rendered with consistent widths so everything lines up, even in a narrow terminal window.
- ISO timestamps returned by the API are auto-detected and rewritten in IST format.
- Invalid field/metric names result in friendly error messages rather than silent failures.

## When to reach for the CLI
- You want a quick health check while the frontend is still building or inaccessible.
- You’re SSH’d into a server and need to confirm Marova’s state without opening the dashboard.
- You’re testing automation scripts and prefer command line output you can pipe into other tools.

## Tips for presentations
- `python -m app.cli code-state --field mood --field dream_energy` is a concise demo to show mood tracking.
- Run `code-stats --limit 5 --metric dream_debt --metric toxicity_level` before/after logging a memory to highlight the toxin/energy feedback loop.
- Toggle auto-sleep live with `code-auto-sleep --disable` followed by `--enable` to show how ops can override nightly cycles on demand.
