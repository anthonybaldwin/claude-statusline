# claude-statusline

A cross-platform [Claude Code](https://claude.com/claude-code) [status line](https://code.claude.com/docs/en/statusline) — context, cost,
rate-limit windows, git, and full config-scope breakdowns — rendered as an opinionated, multi-row
dashboard with Nerd Font glyphs. Zero-dependency [Bun](https://bun.sh) scripts, no build step.

It reads the JSON Claude Code feeds to a status-line command on stdin (model, context window,
cost, rate limits, cwd, git/PR, vim mode, …), enriches it with cheap local reads (git, config
files, `/etc/os-release`, etc.), and reflows everything to your terminal width.

## What it shows

Each row is width-aware: it packs onto one line when it fits, wraps to aligned continuation
lines when it doesn't, and gauges drop their progress bar before anything gets truncated.
Height adapts to your terminal, taking at most a third of it — Claude Code needs the rest for
the conversation, prompt, and its own footer. On a tall window rows wrap freely (nothing
truncated); as the window shortens, wrapped detail is cut with a trailing `…` first, then whole
rows are shed (least-essential first, the Model/Limits gauges last) so the dashboard never
pushes Claude Code's own footer off the bottom. Within one window size the height only ratchets
up, never down, so Claude Code's repaint never leaves ghost rows behind.

- **Model** — model name (⚡ when fast mode is on), reasoning effort (styled to echo Claude
  Code's `/effort` menu) with a 💡 lamp when extended thinking is on, and a context-window gauge
  that warns early (yellow ≥50%, red ≥80%) plus a red `200K+` marker once you cross the fixed
  200k long-context pricing line on an extended-context (>200k) model.
- **Limits** — the `5h` and `7d` rate-limit windows with usage bars, percentages, and reset
  times; plus any model-scoped weekly caps from the OAuth usage API's `limits` array (cached
  with a TTL) — e.g. `7dF` for the Fable weekly allowance, which the API normalizes to its own
  0–100% scale. Each windowed gauge carries a signed pace balance vs the even-consumption budget
  line: `+N%` = quota in hand (green), `-N%` = burning ahead of pace (yellow). A `Cr` gauge
  (monthly usage-credit spend, `$used/$limit`) appears only once credits actually start being
  consumed. Behind a [Claude apps gateway](https://code.claude.com/docs/en/claude-apps-gateway-spend-limits),
  an `Sp` gauge shows your spend-limit usage (`rate_limits.spend_limit`, Claude Code ≥ 2.1.251) —
  it keeps counting past 100% once you're over the cap, with the period's reset date.
- **Usage** — total session cost, `$/h` burn rate, throughput (tok/s), API time, and wall-clock.
- **Turn** — the last call's token makeup: input / output / cache-write / cache-read; then the
  session's prompt-cache health (`prompt_cache`, Claude Code ≥ 2.1.251): hit ratio, `warm` with
  time left on the TTL or `cold` with what the next request will re-cache, and a yellow miss
  count with the likely cause of the last miss (e.g. `tools changed`) when there's room.
- **Activity** — active slash command, todo progress, last tool call, sub-agents (each running
  agent with its own task count, plus a green ✔ tally of completed ones), and the session's edit
  volume (+added / −removed lines).
- **Repo** — *(only inside a git repo)* project name, origin-remote identity (host-branded
  GitHub/GitLab/Bitbucket badge + `owner`, or `owner/name` when the remote name differs from the
  folder), worktree, branch with ahead/behind and staged/modified/untracked/conflict counts, the
  current branch's open PR — or GitLab merge request, shown as `!N` — colored by review state,
  and the latest `v*` tag with commits-since.
- **Config** — what's actually loaded, each broken down by Claude Code's real config scopes
  **(managed / user / project / local / plugin)**, de-duped by precedence and gated on workspace
  trust: instruction files (CLAUDE.md, plus AGENTS.md wherever Claude Code ≥ 2.1.277 loads it
  natively — honoring the *Project instructions* setting), auto-memory files (this project's `~/.claude/projects/<slug>/memory/`),
  agents, commands, skills (incl. output-styles), workflows, routines, rules, MCP servers,
  claude.ai connectors, claude-in-chrome, hooks, plugins, themes (plugin or your own
  `~/.claude/themes/`), and session-added dirs.
- **Exts.** — plugin-*exclusive* component types, each shown with its count (`0` included, the
  same as Config): LSP servers, background monitors, `bin/` executables, and message channels.
  Counts include components declared inline in a plugin's **marketplace entry**, not just in the
  plugin's own files (which is how the official LSP plugins ship them).
- **Host** — local clock, OS badge with real version, `user@host`, and a fleet count of *other*
  live Claude Code sessions on this machine (the same population as the `← N agent` count in
  Claude Code's own footer; hidden at 0).
- **Info.** — current directory (home-relativized, leaf preserved when long), vim mode,
  output style, Claude Code version, agent name, a `remote` cloud marker for remote-attached
  sessions, and the session identity: its custom/AI-generated name when one exists, with the
  id (for `claude --resume`) in dim parens.

### Reading the `(0/0/1/0/-)` scope breakdown

Most Config items render as `icon total (m/u/p/l/x)` — a count, then a dim parenthetical that
says *where* those items come from. The five slots are fixed and always in the same order, from
broadest to narrowest scope:

| Slot | Scope | Where it lives |
| --- | --- | --- |
| `m` | **managed** | Enterprise/org config: `managed-settings.json`, `managed-mcp.json`, or the Windows registry policy. Only non-zero on managed machines. |
| `u` | **user** | Your `~/.claude/` (settings, agents, skills, …) and top-level `~/.claude.json` — loads in every project. |
| `p` | **project** | Committed in the repo: `.claude/`, `.mcp.json`, `CLAUDE.md`. Shared with collaborators. |
| `l` | **local** | Your uncommitted per-project overrides: `.claude/settings.local.json`, `CLAUDE.local.md`, `~/.claude.json` `projects[cwd]`. |
| `x` | **plugin** | Components bundled by enabled plugins. Lowest precedence. |

Each slot is one of two things:

- **A number** (including `0`) — this item type *can* come from that scope, and this is how many
  currently do. A `0` means "nothing from here right now".
- **`-`** — this item type can *never* come from that scope, so the slot is not applicable.
  Agents, commands, and skills have no managed or local form; rules and routines have no plugin
  form; and so on.

So `(0/0/1/0/-)` next to the CLAUDE.md icon reads: no managed memory file, none in `~/.claude/`,
**one `CLAUDE.md` in this repo**, no `CLAUDE.local.md`, and plugins can't provide one. Likewise
`(-/3/0/-/2)` on skills means three of your own in `~/.claude/skills/`, none in this repo, two shipped by
plugins — and the dashes because skills have no managed or local form.

Which scopes each item can have:

| Item | Slots that can hold a count |
| --- | --- |
| CLAUDE.md, Plugins | `m` `u` `p` `l` |
| MCP servers, Hooks | `m` `u` `p` `l` `x` |
| Agents, Commands, Skills, Workflows | `u` `p` `x` |
| Rules, Routines | `u` `p` |
| Themes, Channels | `u` `x` |

Items counted at a single location (auto-memory, connectors, chrome, session dirs, and the
LSP/monitor/bin entries on **Exts.**) show a bare count with no parenthetical. Items whose total is
`0` also skip it — an all-zero breakdown is just noise. On a narrow terminal the parenthetical is
the first thing dropped to reclaim width; the total always stays visible.

## Requirements

- [**Bun**](https://bun.sh) — the script runs under `bun` (uses `Bun.stdin` / `Bun.main`).
- A [**Nerd Font**](https://www.nerdfonts.com/) installed — all icons are Nerd Font glyphs.
  It doesn't have to be your primary terminal font: some terminals fall back to installed fonts
  automatically (e.g. Ghostty on macOS), while others need it listed as an explicit fallback —
  in Windows Terminal, set the font face to e.g. `MonoLisa, SymbolsNerdFont`. Patching your
  existing font with the Nerd Fonts patcher is also an option. Without a Nerd Font anywhere in
  the chain, icons render as tofu boxes (the text still works).
- A terminal with **truecolor (24-bit)** support for the smoothest effort-level gradients
  (256-color still looks fine).

## Install

Clone it anywhere you like:

```bash
git clone https://github.com/anthonybaldwin/claude-statusline.git
```

Then run the installer — it configures `~/.claude/settings.json` for you. It's a Bun script (Bun is
required to run the status line anyway), so it works the same on Windows, macOS, and Linux:

```bash
cd claude-statusline
bun install.js          # add --print to preview the changes without writing
```

It points Claude Code's `statusLine` and `subagentStatusLine` at these scripts, backs up any
existing `settings.json`, and preserves your other keys. Restart Claude Code to see it.

### Manual setup

Prefer to wire it up by hand? Add a `statusLine` block to `~/.claude/settings.json`
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

## Subagent status line

`subagent-statusline.js` implements Claude Code's
[`subagentStatusLine`](https://code.claude.com/docs/en/statusline#subagent-status-lines) setting —
the per-subagent row in the agent panel below the prompt. It renders each row as
`name · description · model effort · ● · NNk tok`, colored by status (running / completed / error)
and clamped to the row width, matching the main dashboard. Wire it up alongside `statusLine`:

```json
{
  "subagentStatusLine": {
    "type": "command",
    "command": "bun \"/path/to/claude-statusline/subagent-statusline.js\""
  }
}
```

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
