# herdr-pomodoro

A pomodoro timer for [herdr](https://herdr.dev) that lives in the sidebar, with
a 🍅 like [tmux-pomodoro-plus](https://github.com/olimorris/tmux-pomodoro-plus).

```
🍅 24:13 2/4        focusing, 2nd pomodoro of 4
☕ 04:59 2/4        on a break
⏸ 🍅 12:03 2/4      paused
🍅 --:--            idle
```

Plain bash, no build step. The timer needs only herdr; the work log needs
`jq`, and the log browser needs `fzf`.

## How it works

herdr doesn't let plugins draw arbitrary sidebar sections, but Space rows can
show custom `$name` metadata tokens, and a row with no values disappears.

- `SCOPE=global` (default): one timer. The plugin reports the `$pomodoro`
  token on the **first workspace only**, so a `$pomodoro` row placed first in
  `ui.sidebar.spaces.rows` renders as a single line above your spaces list.
- `SCOPE=space`: every space has its own timer, shown on that space's row.
  Actions and the monitor act on the focused space. Idle spaces show nothing
  unless `SHOW_IDLE=1`.

The timer's truth is an end timestamp in a state file. A ticker process runs
only while the timer is running, refreshing the token every second (with a
5s TTL so a dead ticker never leaves a frozen clock). A startup hook revives
it after herdr restarts, and workspace events move the token when the first
workspace changes.

## Install

```sh
herdr plugin link ~/code/herdr-pomodoro
```

Then add the row to `~/.config/herdr/config.toml` and reload (`prefix+r`):

```toml
[ui.sidebar.spaces]
rows = [
  [{ token = "$pomodoro", fg = "#fb4934", bold = true, rules = [
    { starts_with = "⏸", fg = "#a89984", bold = false },
    { starts_with = "☕", fg = "#b8bb26" },
    { contains = "--:--", fg = "#a89984", bold = false },
  ] }],
  ["state_icon", "workspace"],
  ["branch", "git_status"],
]
```

The last two rows are herdr's defaults; keep whatever you had.

## Controls

| Action | Does |
|---|---|
| `nkampa.pomodoro.toggle` | start / stop |
| `nkampa.pomodoro.start` | start or resume |
| `nkampa.pomodoro.stop` | pause |
| `nkampa.pomodoro.reset` | stop and clear the session |
| `nkampa.pomodoro.skip` | jump to the next phase |
| `nkampa.pomodoro.monitor` | popup with a progress bar and cycle count |
| `nkampa.pomodoro.scope-toggle` | switch between one global timer and one per space |
| `nkampa.pomodoro.refresh` | re-read config and re-sync the sidebar |
| `nkampa.pomodoro.log` | browse the work log (fuzzy search by year / month / day) |

In the monitor: `space` start/stop, `r` reset, `n` skip, `q` close.

Bind keys in `config.toml`:

```toml
[[keys.command]]
key = "prefix+t"
type = "plugin_action"
command = "nkampa.pomodoro.toggle"
description = "pomodoro start/stop"
```

## Work log

Every focus period that ends (completed, skipped, or reset after at least
`MIN_LOG_MINUTES`) is logged with:

- start and end time, minutes focused, and how it ended
- the space: name, folder, and git branch
- the space's agents and their terminal titles (Claude titles its terminal
  with a summary of the session, so this doubles as a "what was I doing")
- an optional note

When a pomodoro completes, a popup shows that context and asks what you got
done. Enter saves the note; leaving it empty keeps the agent titles as the
summary. The break keeps running meanwhile (`NOTE_PROMPT=required` holds the
break until you answer, `off` skips the popup).

Entries go to `~/.local/share/herdr-pomodoro/log.jsonl`, one JSON object per
line, and (with `LOG_JOURNAL=1`) to a markdown file per day in
`~/.local/share/herdr-pomodoro/journal/`:

```markdown
- 11:49–12:14 🍅 25m **chesscom** (`main`) — cache-first service worker
```

### Browsing the log

The `log` action opens a popup to browse it (needs `fzf`):

```
all › sw
  2026-09-25 Fri 11:49–12:14  🍅 25m  chesscom (feat/sw)  cache-first service worker
  2026-08-03 Mon 09:00–09:25  🍅 25m  chesscom (feat/sw)  Service worker cache
```

Type to fuzzy-search every entry (space, branch, note, agent titles, dates).
`ctrl-y` / `ctrl-o` / `ctrl-d` / `ctrl-e` switch to years, months, days, or
all entries; `enter` drills into the selected year, month, or day; backspace
on an empty query goes back up. The preview shows totals for a period
(pomodoros, time, days, per-space breakdown) or the full entry.

`bin/pomodoro log` prints today's entries (`log all`, `log 2026-09-25`).
Logging needs `jq`.

## Tab bar (optional)

`bin/pomodoro status` prints the same line (the focused space's timer with
`SCOPE=space`), so it also works as a tab bar entry:

```toml
[ui]
tab_bar_right = [
  { type = "command", command = "~/code/herdr-pomodoro/bin/pomodoro status", interval_seconds = 1, timeout_seconds = 1 },
]
```

## Config

Optional. Create `config` in the plugin's config dir
(`herdr plugin config-dir nkampa.pomodoro`); it's sourced as bash:

```sh
SCOPE=global          # global | space (one timer per space)
WORK_MINUTES=25
SHORT_BREAK_MINUTES=5
LONG_BREAK_MINUTES=15
LONG_BREAK_EVERY=4
AUTO_START_BREAKS=1   # break starts on its own when a pomodoro ends
AUTO_START_WORK=0     # focus waits for you after a break
ICON_WORK="🍅"
ICON_BREAK="☕"
ICON_PAUSED="⏸"
SHOW_IDLE=            # 1 shows "🍅 --:--" when idle; default 1 global, 0 space
NOTIFY=herdr          # herdr | system | both | off

# Work log
LOG=1                 # 0 turns logging off
LOG_FILE="$HOME/.local/share/herdr-pomodoro/log.jsonl"
LOG_JOURNAL=1         # also write a markdown file per day
JOURNAL_DIR="$HOME/.local/share/herdr-pomodoro/journal"
NOTE_PROMPT=optional  # optional | required | off
NOTE_TIMEOUT=300      # seconds before the note popup gives up
MIN_LOG_MINUTES=1
```

After editing, run the `nkampa.pomodoro.refresh` action. `scope-toggle` flips
`SCOPE` for you. Switching scope pauses the old scope's running timers; they
pick up where they were when you switch back.

`NOTIFY=herdr` uses `herdr notification show`, which respects
`ui.toast.delivery` (off by default). Use `system` for a macOS/Linux desktop
notification regardless.
