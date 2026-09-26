Category: Ideas
Title: [idea] Session-wide sidebar row for plugin status (not tied to a Space or agent)

I built a pomodoro plugin and wanted its countdown (`🍅 24:13`) in the sidebar. Custom `$name` tokens nearly get me there, but every token belongs to a Space or a pane, so session-wide status has nowhere to go:

- Reporting it on every Space repeats the same line under each one.
- Reporting it on one Space ties it to that Space. The plugin has to follow workspace create, close and reorder events, and the line reads as that Space's status.
- The agents panel only has rows while agents exist, and those rows move as agents come and go.

The same gap applies to any glanceable, session-wide status: usage or quota meters, CI state, a clock, a focus timer. `tab_bar_right` command entries cover some of this, but the sidebar is where I look for status.

What would help is one or two session-scoped rows, above the Spaces heading or at the bottom of the sidebar. They could use the existing token row syntax and styling rules, with values reported the same way `workspace report-metadata` works today but without a workspace id. Empty rows would disappear like they do now, so nothing changes for anyone who doesn't configure them.

Related: #4445 (a separate right sidebar for plugins) and #1608 (persistent plugin surfaces, closed) ask for richer surfaces. This asks only for a place to show text tokens that aren't tied to a Space.

Herdr 0.9.1, macOS.
