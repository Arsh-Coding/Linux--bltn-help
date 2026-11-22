# Linux Doubt Sessions

Questions I ask while learning, with the answers. Any new command from here also goes into [linux-commands.md](linux-commands.md).

| # | Date | Topic | Phase |
|---|---|---|---|
| 1 | 2026-11-22 | `upower -b` vs `upower -i $(upower -e \| grep BAT)` | Saved commands (battery) |

---

## 1. Why does `upower -b` give the same result as the long battery command?

Question: I use `upower -i $(upower -e | grep BAT)` for battery status, but `upower -b` shows the same output. What's the difference?

Answer: They end up at the same information, they just get there differently.

- `upower -b` (`--battery`) is a built-in option that prints full details for every battery device. One step, no searching.
- The long version does it manually in three steps: list all power devices (`-e`), keep only the battery line (`grep BAT`), then show details for that one device (`-i`).

| | `upower -i $(upower -e \| grep BAT)` | `upower -b` |
|---|---|---|
| How it works | You find the battery path, then ask about it | upower finds batteries itself |
| Battery not named `BAT...` (e.g. `CMB0`) | Breaks, `grep` finds nothing | Still works |
| Laptop with two batteries | `-i` expects one device path | Shows all of them |
| Availability | Works on any upower version | Newer versions only (older ones say "Unknown option") |

Verdict: use `upower -b` day to day. Keep the long version in your head anyway, because the pattern inside it is used everywhere in Linux:

- `|` (pipe) sends one command's output into another
- `grep` filters lines
- `$( ... )` runs a command and pastes its result into another command

Related options:

```bash
upower -e        # list every power device path (battery, AC adapter, etc.)
upower -d        # dump details for ALL devices, not just the battery
upower -v        # show upower version
upower --help    # list every option (this is how you discover things like -b)
```

Lesson: I found `-b` myself by reading `man upower`. When you find a long command online, check `man command` or `command --help`. There's often a built-in shortcut.

Source: [upower(1) man page](https://man.archlinux.org/man/upower.1)
