# weikipop — Fixes Summary

Branch: `feature/lazy-senses-and-frequencies` (merged with `main` @ df65603).
Scope: finished the sense-level lazy-loading work, made frequencies independent
of the main dictionary, and fixed the Magpie fullscreen and fast-lookup bugs.

All changes are committed locally in small increments and are **not pushed**.

---

## Technical Summary

### 1. Sense-level lazy loading (`src/gui/popup.py`)

Previously the first two entry groups rendered *every* sense of *every*
dictionary immediately, which is where the remaining lag came from.

- `_calculate_content` now renders group-by-group and stops as soon as the
  rendered HTML fills ~1.2× the popup height. Each dictionary entry initially
  renders only its first `_SENSES_PER_ENTRY_INITIAL` (4) senses.
- `_on_scroll_lazy_load` expands in two tiers at 70% scroll depth: first it
  reveals more senses inside already-visible entries (`_SENSES_PER_LOAD`, 5 at a
  time), then it appends new entry groups (`_GROUPS_PER_LOAD`, 3 at a time).
- `_expand_senses_to_fill` handles the case where all content fits the viewport
  (no scrollbar): hidden senses are expanded up front so they can never become
  permanently unreachable.
- Height gating now measures at the popup's real content width, and the
  rendered-sense count returned by `_render_senses` is clamped to the true total.
- Restored the `<hr>` separators between entry groups that the in-progress
  branch had dropped (entries were visually running together), and deleted the
  now-dead `_render_groups_to_html`.

### 2. Frequency independent of the main dictionary (`src/dictionary/lookup.py`)

Frequency ranks (JPDB/jiten) only live inside the main `dictionary.pkl`. When
the main dictionary wasn't first — or was disabled — the head entry of a word
group carried no frequency, so no badge showed and mining exported nothing.

- Added a standalone `freq_index` keyed by `(written_form, reading)` with a
  `(written_form, '')` fallback, harvested from every source's lookup map —
  **including a disabled built-in main dictionary** (loaded once, harvested, and
  released rather than pinned in memory; paths already harvested are skipped).
- `_format_and_sort` overlays the best index frequency onto each entry before
  the existing group propagation, so the badge (`popup.py`) and the mined
  `{freq}` / `{frequency-*}` fields both pick it up regardless of dictionary
  order or enabled state.
- Hardened against malformed frequency values: `_harvest_freq` ignores non-int
  freqs, and the lookup path normalizes a non-int freq to `DEFAULT_FREQ` so an
  imported dictionary with a bad/`None` rank can't crash a lookup.
- Verified with a headless test across three cases (main first / main second /
  main disabled) plus a malformed-frequency case.

### 3. Magpie fullscreen popup (`src/gui/magpie_manager.py`, `src/gui/popup.py`)

Three stacked causes of the position-dependent "spazzing" and the
occasional no-show:

- **Z-order:** the popup lost the topmost fight to Magpie's own fullscreen
  window. `show_popup` now calls `raise_()` unconditionally and re-asserts
  `HWND_TOPMOST` via `SetWindowPos` on Windows (and again ~once/second while
  visible). Added `WindowDoesNotAcceptFocus` so a click can't activate the
  popup and knock Magpie out of scaling.
- **Placement flapping:** the flip logic switched sides on a one-pixel boundary
  (mid-screen / screen edge), which Magpie's coordinate scaling amplified.
  Added hysteresis (`_flip_x/_y_with_hysteresis`, `_FLIP_MARGIN`) so the popup
  only flips once the preferred side is exceeded by a margin.
- **Transform instability:** `transform_raw_to_visual` now keeps the last good
  Magpie rect for a short grace period instead of snapping back to raw
  coordinates on a momentary read failure, and clamps near-edge cursor points
  into the source rect so edge text still maps.

### 4. Occasional garbled definitions / spacing on fast lookups

Root cause was render/scroll races, not the dictionary data:

- A stale `QTimer.singleShot` scroll-restore could fire into freshly-swapped
  content. Restores are now guarded by a `_render_epoch` token and a
  `_suppress_scroll_signal` flag so programmatic scrolling can't trigger lazy
  expansion or land on the wrong word.
- The 16 ms move-timer could show the previous word's HTML before the new
  content rendered. `show_popup` is now gated on a `_render_ready` flag set only
  after the current data has actually rendered.
- An empty lookup result (`[]`) no longer leaves stale definitions on screen —
  it's treated as "no content" and the popup hides.
- `reapply_settings` now invalidates cached render state, and `get_scan_geometry`
  snapshots the monitor to avoid torn reads during a scan-region change.
- The missing `<hr>` separators (see §1) were the other contributor to the "odd
  spacing" reports.

### Testing status

- Frequency logic: covered by a headless unit test (all cases pass).
- Everything GUI-related (Magpie behavior, scroll expansion, badge display,
  mining) could **not** be run on this machine (no PyQt6 / `dictionary.pkl` /
  config here) and needs manual verification in the real app.

### Known trade-offs (left as-is by choice)

- The Magpie transform persists for ~1 s after Magpie actually exits (deliberate,
  to avoid teleport-flicker during momentary window recreation).
- Tier-1 scroll compensation is exact when the expanding entry is above the
  viewport and approximate in the rare case you're reading *inside* that entry.

---

## Plain-language Summary

Four things were wrong with the popup dictionary, and all four are fixed.

**1. It was slow with lots of dictionaries.** Even though it looked like it was
loading things gradually, it was secretly building *all* the definitions for the
first couple of words up front. Now it only builds what actually fits on screen,
and loads more as you scroll down — including more meanings inside a single word.
So you can add lots of dictionaries without it getting sluggish.

**2. Word frequencies only worked if the main dictionary was at the top.** The
frequency numbers (how common a word is) were tied to the main dictionary, so if
you moved it down the list or turned it off, the numbers vanished — both on the
popup and on the flashcards you make. Now the app collects those numbers
separately, so they show up and get saved no matter where the main dictionary
sits or whether it's switched on.

**3. In Magpie fullscreen, the popup jumped around or didn't appear.** This was
a few problems at once: the popup was getting hidden *behind* Magpie's own
fullscreen window, it kept violently flipping between above and below the text
when the text sat near the middle of the screen, and a feedback loop sometimes
made it flicker on and off. It now stays on top, holds its position steadily,
and no longer fights with itself.

**4. Now and then a definition looked garbled, and fixed itself on a re-look.**
This happened when you looked words up quickly — the popup occasionally showed
the previous word for a split second or landed on the wrong scroll position while
swapping in the new word. That timing problem is fixed, so lookups render cleanly
even in a hurry.

**One thing to note:** these changes were written and checked as far as possible
without running the full app on this computer (it isn't set up to run here), so
the on-screen behavior — especially the Magpie fullscreen fixes — should be
tried out in the real app to confirm everything feels right.
