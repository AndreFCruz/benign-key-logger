# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file macOS key logger (`key_logger.py`) for analyzing *your own* keyboard usage (intended for keyboard-layout optimization). The entire project is one Python script plus a README. There is no package, no build step, and **no automated test suite or linter config** — do not go looking for them.

The defining constraint is the project's "benign" ethos: a moderately experienced programmer must be able to read the script and be convinced nothing nefarious happens. Honor this when changing code:

- **Never add network behavior or new third-party dependencies.** The only non-stdlib import is `pynput`. Data stays local, always.
- **Keep invasive capabilities opt-in.** Stdout key echo, plaintext file logging, physical-key logging, and full-event capture all default off (see `DEFAULT_*` constants at the top). Don't flip a privacy-affecting default to "on" without an explicit request.
- **Keep it a single auditable file.** Splitting into modules has been repeatedly considered and rejected; preserve that unless asked otherwise.
- Output files holding keystrokes are sensitive and are forced to owner-only `0600` (`OWNER_ONLY_FILE_MODE`); preserve that hardening for any new file the logger writes.

## Environment & running

Use the existing conda env `keylogger` (Python 3.11, `pynput` 1.8.2) for anything that runs the code:

```bash
conda run -n keylogger python key_logger.py            # default: SQLite logging
conda run -n keylogger python key_logger.py --help     # all flags + current defaults
conda run -n keylogger python key_logger.py --stdout    # echo captured keys live
conda run -n keylogger python key_logger.py --debug     # internal state, no key echo
```

**macOS gotcha:** the listener needs Accessibility permission (System Settings → Privacy & Security → Accessibility) for *the app running it* (Terminal, editor, etc.). Without it the script runs but silently receives no events. macOS Secure Input Mode (password fields) suppresses logging at the OS level — there is no password filtering in this code, so don't claim there is.

To smoke-test parsing/setup logic without a live keyboard, exercise the pure helpers and `parse_args(...)` directly rather than running the listener.

## Architecture

The script splits cleanly into three layers — understanding the boundary between them is the key to navigating it:

1. **`Config` (dataclass)** — immutable user choices derived from CLI flags. `build_parser()` / `parse_args()` turn argv into a `Config`. `--full-events` requires SQLite and errors otherwise.
2. **`KeyLoggerApp`** — all mutable runtime state and side effects: SQLite lifecycle, file-permission hardening, the `keys_currently_down` list, and event handling.
3. **Top-level pure functions** — key canonicalization/normalization/stringification (`canonicalize_key`, `key_to_str`, `normalize_ctrl_character`, `normalize_shifted_key_for_physical_logging`, `format_logged_key`, `equivalent_shifted_keys`). These have no state and are the safest things to change/test.

### Event flow

`pynput.Listener` (runs in its own thread — hence `check_same_thread=False` on the SQLite connection) → `preprocess()` (remap L/R modifiers, then drop ignored keys) → `key_down` / `key_up`.

The central, non-obvious design decision: **logging happens on key-DOWN of non-modifier keys only.** Modifiers are accumulated in `keys_currently_down` and only combined into a log entry when a real key fires. Read the long docstrings in `log()`, `key_down()`, and `key_up()` before touching this — they encode behavior that looks like bugs but isn't:

- **Shift + symbol** is logged as the resulting character (`Shift+a` → `A`), *not* as a combo — unless `--physical-keys` is set. Other modifiers (Ctrl/Cmd/Alt) are always kept as explicit combos.
- **Sticky-key suppression:** a key already in `keys_currently_down` is ignored on repeat.
- **Orphaned key-up events** (an up with no matching down) genuinely occur — from out-of-order shift+symbol release and from Secure Input Mode transitions. `key_up()` handles them via `reconcile_shift_mismatch()` and a "locked-in" garbage-collection sweep gated on `LOCKED_IN_GARBAGE_COLLECTION_LIMIT` and no modifiers being held.

### Storage

SQLite is the default sink. `setup_sqlite_database()` creates the `key_log` table (and `full_key_log` when `--full-events`) plus three convenience views (`key_counts`, `bigram_counts`, `trigram_counts`) defined in the `*_VIEW_SQL` constants. Writes are **batched** — committed every `SQLITE_COMMIT_EVERY_N_EVENTS` (50) events or `SQLITE_COMMIT_EVERY_SECONDS` (5s), with a forced final flush via an `atexit` handler. Data is appended across runs (`CREATE TABLE IF NOT EXISTS`); to start fresh, delete/rename the DB file. `--wal` is optional and off by default.

## Repo conventions

- **Indentation is 2 spaces** throughout (unusual for Python) — match it exactly.
- After a meaningful change, this repo records a rationale doc at `.agents/commits/<YYYY-MM-DD>_<kebab-slug>.md`. Follow the existing files' structure (Goal / Files Changed And Why / Behavior Changes / Approach / Alternatives Considered / Assumptions / Tradeoffs / Known Limitations / Key Decisions). Keep changes small and single-purpose, as those docs do.
- Keep README defaults, examples, and the "Audit Checklist" in sync whenever you change a flag or a default.
