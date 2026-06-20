# claude-statusline

An opinionated multi-row statusline dashboard for Claude Code. Single-file Bun scripts, no
dependencies, no build step.

## Files

- `statusline.js` — the main dashboard. Reads Claude Code's statusline JSON from stdin, writes
  ANSI-styled rows to stdout. Everything lives in this one file, ordered roughly: ANSI/color
  constants → glyphs → formatting helpers → config-counting subsystem (scopes, plugins, MCP) →
  stdin parse → per-row segment building → `packSection` layout → output.
- `subagent-statusline.js` — the per-subagent row for Claude Code's agent panel
  (`subagentStatusLine` in settings). Same conventions, much smaller.
- `install.js` — points `~/.claude/settings.json` at the scripts in this checkout (merges,
  backs up first). `bun install.js --print` for a dry run.

## Running / verifying changes

Requires **Bun** (`Bun.stdin` / `Bun.main`). There is no test suite. Verify by piping a sample
payload:

```sh
echo '{"session_id":"x","model":{"display_name":"Opus"},"cwd":"C:/some/repo","cost":{},
  "context_window":{"context_window_size":200000,"current_usage":{"input_tokens":50000}},
  "rate_limits":{"five_hour":{"used_percentage":42,"resets_at":"2026-06-12T20:00:00Z"}}}' \
  | COLUMNS=140 bun statusline.js
```

The user's live statusline runs `statusline.js` **directly from this working tree** (settings
point at the repo file, not a copy) — edits take effect on the next render, including broken
ones. Don't leave the file in a non-running state between edits.

## Critical invariants

These exist to dodge real Claude Code rendering bugs; violating them corrupts the user's
terminal, not just the aesthetics.

1. **Height must never shrink between renders.** CC reserves vertical space from the previous
   render's line count; if the count drops, stale rows stack (under-clear, fixed upstream in CC
   v2.1.170 but kept defensive). Hence: `sections` is a fixed slot list, empty sections backfill
   with blank rows to `HEIGHT = sections.length`, plus one trailing gap row — total is always
   `HEIGHT + 1`. **Adding a row = adding one `packSection(...)` entry to `sections`**; never
   conditionally omit a slot.
2. **Lines must never exceed the terminal width.** CC counts logical lines; a terminal
   hard-wrap desyncs its repaint. `packSection` measures with `vlen` (visible width, ANSI- and
   wide-glyph-aware) and degrades per item: widest alt → narrowest alt → wrap to a fresh row.
   Width comes from `COLUMNS || stdout.columns || 80` — never a huge fallback (a `1e9` fallback
   once corrupted the terminal).
3. **U+2800 (Braille blank), not spaces, for indentation and blank rows.** CC strips leading
   whitespace and trailing blank rows; U+2800 renders blank but isn't whitespace.
4. **Never block the render on the network.** Slow data (OAuth usage API) is served from a
   TTL'd temp-file cache; a stale cache triggers a detached self-respawn
   (`CLAUDE_STATUSLINE_USAGE_REFRESH=1`) that refreshes out-of-band and exits before the stdin
   read. Cross-render state lives in `$TEMP/sl-*.json` files (usage cache,
   per-session accumulators).

## Layout conventions

- A row's content is a list of **segments**: plain strings, or `{ alts: [wide, narrow] }` where
  the packer uses `alts[0]` when the row fits and `alts[alts.length - 1]` when space is tight
  (bars and parenthesized detail go in the wide form only).
- `packSection(label, segments, lead, leadWidth, preferDetail)` does the layout: all-widest on
  one line → all-narrowest on one line (skipped when `preferDetail`) → greedy per-item wrap.
  An empty segment list emits nothing (the slot backfills blank — rows like Turn/Activity
  disappear after `/clear`).
- Row **lead** glyphs render once before the first segment; pass the lead's true cell width —
  `vlen` guesses wrong for some glyphs in the user's terminal font.
- Section labels pad to `SECTION_WIDTH` (8) and render bold-italic-SOFT.

## Style conventions

- Colors are 256-color ANSI constants at the top of the file. `SOFT` (245) for labels/punctuation,
  `DIM` (244) for annotations and predictions, `VAL`/`WHITE` (97) for primary values; semantic
  colors (GREEN/YELLOW/RED, usageColor, contextColor) carry meaning — don't reuse them
  decoratively. Each row lead has its own hue; before introducing a color, grep `c256(` to
  confirm it's unused.
- Glyphs are Nerd Font codepoints declared as `g*` constants with the NF name in a comment.
  On the user's Windows font some Font Awesome glyphs render as tofu — Material Design (`nf-md-*`,
  `0xf0xxx`) codepoints are the safer choice; comments by existing glyphs note known exceptions.
- Comments throughout the file explain *why* (CC bugs, font quirks, API gotchas), and several
  encode hard-won diagnoses (e.g. the API-duration laptop-sleep artifact). Preserve and imitate
  that register; don't strip them when refactoring.

## Pace balance

Each Limits gauge appends a signed pace balance vs the even-consumption budget line
(`(elapsed/window)·100`, window start = `resets_at` − window length): `+N%` = quota in hand
(green), `-N%` = burning ahead (yellow), SOFT within ±2%, shown as soon as any time has elapsed
in the window (suppressed only for a degenerate just-reset window — matches usage-buttons'
PaceMetric). Kept to a few chars on purpose — the user wants it terse; don't expand it into words
("reserve"/"over pace") or give it a dedicated row.

## Workflow notes

- Update the README row list when rows change — it enumerates every row in render order.
- Commit when a change is done and verified; **do not push without being asked.**
- Test renders share the real `$TEMP/sl-*.json` caches — clean up anything you seed with fake
  data.
- **Subprocess hygiene:** pass secrets (the OAuth token) to `curl` via its stdin config
  (`-K -`), never a `-H`/CLI arg — args are visible in process listings. Add `windowsHide: true`
  to every `spawn`/`spawnSync` so console/detached children don't flash a terminal on Windows.
