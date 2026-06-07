# claude-statusline

A cross-platform [Claude Code](https://claude.com/claude-code) [status line](https://code.claude.com/docs/en/statusline) — context, cost,
rate-limit windows, git, and full config-scope breakdowns — rendered as an opinionated, multi-row
dashboard with Nerd Font glyphs. One zero-dependency [Bun](https://bun.sh) script.

It reads the JSON Claude Code feeds to a status-line command on stdin (model, context window,
cost, rate limits, cwd, git/PR, vim mode, …), enriches it with cheap local reads (git, config
files, `/etc/os-release`, etc.), and reflows everything to your terminal width.

## What it shows

Each row is width-aware: it packs onto one line when it fits, wraps to aligned continuation
lines when it doesn't, and gauges drop their progress bar before anything gets truncated.

- **Model** — model name (⚡ when fast mode is on), reasoning effort (styled to echo Claude
  Code's `/effort` menu), and a context-window gauge that warns early (yellow ≥50%, red ≥80%).
- **Limits** — the `5h` and `7d` rate-limit windows with usage bars, percentages, and reset
  times; plus the Sonnet weekly window (fetched from the OAuth usage API, cached with a TTL).
- **Usage** — total session cost, `$/h` burn rate, throughput (tok/s), API time, and wall-clock.
- **Turn** — the last call's token makeup: input / output / cache-write / cache-read.
- **Activity** — active slash command, todo progress, last tool call, sub-agents (each running
  agent with its own task count, plus a green ✔ tally of completed ones), and the session's edit
  volume (+added / −removed lines).
- **Repo** — *(only inside a git repo)* project name, worktree, branch with ahead/behind and
  staged/modified/untracked/conflict counts, the current branch's open PR (colored by review
  state), and the latest `v*` tag with commits-since.
- **Config** — what's actually loaded, each broken down by Claude Code's real config scopes
  **(managed / user / project / local / plugin)**, de-duped by precedence and gated on workspace
  trust: CLAUDE.md memory, agents, commands, skills (incl. output-styles), rules, MCP servers,
  claude.ai connectors, claude-in-chrome, hooks, plugins, and session-added dirs.
- **Host** — local clock, OS badge with real version, and `user@host`.
- **Info.** — current directory (home-relativized, leaf preserved when long), vim mode,
  output style, Claude Code version, agent name/type, and the session id (for `claude --resume`).

## Requirements

- [**Bun**](https://bun.sh) — the script runs under `bun` (uses `Bun.stdin` / `Bun.main`).
- A [**Nerd Font**](https://www.nerdfonts.com/) in your terminal — all icons are Nerd Font
  glyphs. Without one, icons render as tofu boxes (the text still works).
- A terminal with **truecolor (24-bit)** support for the smoothest effort-level gradients
  (256-color still looks fine).

## Install

Clone it anywhere you like:

```bash
git clone https://github.com/anthonybaldwin/claude-statusline.git
```

Then point Claude Code at it by adding a `statusLine` block to `~/.claude/settings.json`
(adjust the path to wherever you cloned it):

```json
{
  "statusLine": {
    "type": "command",
    "command": "bun \"/path/to/claude-statusline/statusline.js\"",
    "hideVimModeIndicator": true,
    "padding": 0,
    "refreshInterval": 60
  }
}
```

- `hideVimModeIndicator: true` — the status line renders vim mode itself, so this suppresses
  Claude Code's native `-- INSERT --` (avoids showing it twice).
- `padding` / `refreshInterval` — passed through by Claude Code; the script reads `padding` so
  wide rows don't overflow, and `refreshInterval: 60` keeps the clock current.

That's it — no separate install step and nothing to copy into `~/.claude`. The script reads
everything it needs (settings, transcript, project config) from their normal locations.

## Configuration

A couple of optional environment variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `CLAUDE_STATUSLINE_TRANSCRIPT_BYTES` | `1048576` (1 MiB) | How many trailing bytes of the transcript to parse per render. |
| `CLAUDE_STATUSLINE_MANAGED_DIR` | platform default | Override the enterprise/managed config dir (for testing/relocation). |

It also honors a few `~/.claude/settings.json` keys when present: `effortLevel`, `fastMode`,
and the `statusLine.padding` / `statusLine.refreshInterval` shown above.

## Customizing

It's a single readable file — the top of `statusline.js` defines every color and Nerd Font
glyph as a named constant, so swapping an icon or recoloring a widget is a one-line change. The
extensive inline comments explain *why* each threshold, scope rule, and color was chosen.

## License

[MIT](LICENSE) © Anthony Baldwin
