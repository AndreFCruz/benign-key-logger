## Goal

Add an opt-in way to count *bare* modifier presses — a `Shift`, `Command`,
`Control`, or `Option` pressed and released on its own, with no accompanying
key. The existing logger only ever records a modifier as part of a combo
(`<cmd> + c`), and folds `Shift` into the character it produces (`Shift+a` →
`A`), so a lone modifier tap was invisible. A user wanted those standalone taps
counted for keyboard-layout analysis.

Scope is intentionally narrow: surface bare modifier taps in the existing
aggregate-count sink without changing any default behavior or the existing
typed-character statistics.

## Files Changed And Why

### `key_logger.py`

- New `DEFAULT_COUNT_MODIFIER_TAPS = False` constant, `Config.count_modifier_taps`
  field, and `--modifier-taps` / `--no-modifier-taps` CLI flags (off by default),
  threaded through `parse_args()` and the startup config summary.
- New `KeyLoggerApp.consumed_modifiers` set tracks which currently-held modifier
  keys have already been folded into a logged keystroke. `log()` marks every held
  modifier consumed when a real keystroke fires (including `<shift>` when it is
  absorbed into a capitalized character), so those never double-count as bare.
- Extracted the sink fan-out tail of `log()` into a new `record_log_entry(entry,
  include_in_ngrams=True)` helper. `log()` now calls it with the default; the new
  `log_bare_modifier()` calls it with `include_in_ngrams=False`.
- `key_up()` gained an `else`/`finally` on its existing try: on a genuine modifier
  release that was never consumed, it calls `log_bare_modifier()`; `finally`
  discards the key from `consumed_modifiers` so the next press starts clean. The
  locked-in garbage-collection sweep also clears `consumed_modifiers` to stay in
  sync.

### `README.md`

Documents the flag in the storage section, adds a usage example, and adds an
audit-checklist entry making clear the feature is opt-in and only adds aggregate
counts.

### `launchd/` (LaunchAgent)

The background agent is opted into bare-modifier counting too. Because
`install.sh` rebuilds `ProgramArguments` wholesale (`plutil -replace … -json`)
and ignores the template's array, the flag is added in **two** places that must
agree: the `--modifier-taps` element in the template's `ProgramArguments` (for
documentation) and the `PA_JSON` list `install.sh` actually generates. Adding it
to the template alone would have no effect — the generated plist is what
launchd runs. It stays a benign, counts-only invocation (no `--stdout`,
`--raw-events`, or `--trigrams`).

## Behavior Changes

- Default behavior is unchanged: with the flag off, bare modifier presses record
  nothing, exactly as before.
- With `--modifier-taps`, a modifier pressed and released without any other key is
  counted under its canonical name (e.g. `<shift>`, `<cmd>`) in whatever sinks are
  enabled (counts, `--raw-events`, `--file`, `--stdout`).
- Bare taps are kept out of the bigram/trigram chain, so enabling the flag does
  not perturb existing typed-character adjacency stats.
- Startup logging now reports the active `modifier_taps` setting.

## Approach

Detect "bare" at key release rather than guessing at press time: a modifier is a
bare tap exactly when it comes up having never contributed to a logged keystroke.
"Contributed" is tracked with a per-physical-key `consumed_modifiers` set —
populated in `log()` (a real keystroke is firing, so all held modifiers are now
used) and cleared per key on release. This reuses the existing key-down/key-up
state machine instead of adding a parallel timer or heuristic.

Routing bare taps through a shared `record_log_entry()` keeps them honoring the
same owner-only file hardening, batched commits, and sink selection as every other
entry, with the single difference of skipping the n-gram chain.

## Alternatives Considered

### Make bare-modifier counting the default

Rejected. The project keeps invasive/behavior-changing capabilities opt-in and
avoids silently changing what gets logged or the meaning of existing data. An
explicit flag preserves stable defaults and lets the user enable it where wanted
(including the background LaunchAgent, which `install.sh` runs with `--modifier-taps`).

### Count bare taps on key-down with a timer

Rejected. You cannot know at press time whether a modifier will be part of a combo
until another key arrives (or doesn't). Deciding on release is exact and needs no
timer.

### Let bare taps participate in the bigram/trigram chain

Rejected. A lone modifier is not an adjacency-relevant keystroke, and including it
would change existing typed-character n-gram counts the moment the flag is enabled.
Skipping the chain makes the flag purely additive.

### Reuse `--full-events` instead of a new flag

Rejected. `--full-events` already logs every individual modifier up/down, but into
a separate verbose table that stores exact ordered events. The request was for
bare presses to appear in the default aggregate `key_counts` view, which this flag
provides without the privacy cost of full event capture.

## Assumptions

- Counting by canonical modifier name (default) is what the user wants; under
  `--modifier-sides` bare taps are reported per side like the rest of the app.
- Existing users rely on current defaults and on the existing n-gram stats not
  shifting underfoot.

## Shortcuts Or Tradeoffs

- Holding two keys that canonicalize to the same modifier (e.g. both Shift keys)
  and releasing both with no intervening keystroke counts as two bare taps. This
  is a rare gesture and arguably correct (two physical presses occurred).

## Known Limitations

- If both physical Shift keys are held while a keystroke fires and then released
  one at a time, the consumed-state is tracked per physical key, so a stale
  release is not mis-counted in the normal case; the only edge is the
  simultaneous-both-shifts-no-key case noted above.
- Bare taps are still subject to the same macOS Secure Input Mode and Accessibility
  constraints as all other capture.

## Key Decisions

- Keep the behavior opt-in and off by default.
- Detect bare taps on release via a consumed-modifier set, reusing the existing
  state machine.
- Keep bare taps out of the n-gram chain so the flag is purely additive.
- Factor sink fan-out into `record_log_entry()` so bare taps share all existing
  hardening and batching.
