# KeepAlive 🟢

> **A tiny Windows tray app that keeps your PC awake and keeps Microsoft Teams
> showing you as "Available" while you step away from the keyboard — by
> pressing a keyboard key that does absolutely nothing: `F15`.**

No installer. No admin rights. No internet. No mouse moving on its own.
One small file, double-click, done.

---

## 💡 The idea in 30 seconds

You walk away from your PC for a few minutes — to make tea, read a printout,
think through a problem, or talk to someone in person. Behind your back,
two annoying things happen:

1. **Windows goes to sleep** (downloads pause, remote sessions drop,
   long tasks stop).
2. **Teams flips you to 🟡 "Away"**, even though you're working — just not
   typing.

KeepAlive quietly sits in the system tray and fixes both. Every 5 seconds
it asks Windows one question: *"Has the user really been idle, and is Teams
even running?"* Only if both answers are yes does it step in — it tells
Windows "stay awake" and, once you've been idle for 60 seconds, presses the
harmless **F15** key. Teams sees activity, stays 🟢 **Available**, and
nothing on your screen moves or changes. The moment you touch the mouse or
keyboard again, KeepAlive steps back and lets you take over.

**One-line pitch:**
> KeepAlive presses the most useless key on the keyboard (F15) once a
> minute — but only when you're idle and only while Teams is open — so your
> PC never sleeps and your status never goes Away.

---

## 😤 The problem

| Annoyance | Why it happens |
|---|---|
| PC sleeps mid-download / mid-task | Windows power settings turn the machine off after inactivity |
| Teams shows 🟡 Away too quickly | Teams watches keyboard/mouse **input idle time**, not whether you're actually busy |
| Can't just change the settings | On many work laptops, power settings are locked by IT policy |
| Even with sleep disabled, status still goes Away | Preventing sleep solves the *power* problem, not the *presence* problem — they're two separate systems |

Existing workarounds all have downsides: hardware mouse jigglers cost money
and visibly move your cursor, "wiggle the mouse" scripts fight with you
when you return, and pressing random keys like `Esc` or `Space` can close
dialogs or type into documents.

---

## ✅ The solution — three small tricks

### 1. Tell Windows "I'm still here" (power)
KeepAlive calls the Windows API `SetThreadExecutionState` with
`ES_SYSTEM_REQUIRED | ES_DISPLAY_REQUIRED`. This is the same official
mechanism video players use to stop your screen sleeping during a movie.

### 2. Watch *real* idle time, don't guess (timing)
Instead of pressing keys blindly on a schedule, KeepAlive asks Windows for
the time of your last real input using `GetLastInputInfo`. It only acts
when you have genuinely been idle for **60 seconds or more** — and any real
key press or mouse movement instantly resets that clock.

### 3. Press a key that does nothing (presence)
When idle, it injects a single **F15** key press (`VK_F15`, virtual-key
code `0x7E`) using `SendInput`. F15 exists in the Windows keyboard spec,
but almost no physical keyboard has one and almost no app binds any action
to it — so the press is invisible, yet Teams still counts it as activity.

### + One important rule: only while Teams is running (scoping)
If Teams isn't open, KeepAlive does **nothing at all** and lets your PC
sleep normally. No point burning power keeping a machine awake for nobody.

---

## ⌨️ Why F15 is the clever part

| Fake-activity method | Problem |
|---|---|
| Move the mouse | Cursor visibly jumps; can fight with the user; looks suspicious on screen-share |
| Press `Esc` | **Dangerous** — closes dialogs, cancels operations, exits full-screen |
| Press `Space` / letters | Types into documents, chats, search boxes |
| Press `Shift` / `Ctrl` | Mostly safe, but can trigger Sticky Keys prompts and interfere with shortcuts |
| **Press `F15`** ✅ | Nothing is bound to it, nothing reacts to it — but Windows and Teams still register "user activity" |

**Golden rule of this project: never, ever inject `Esc`.**

---

## ⚙️ How it works

A timer ticks every **5 seconds**. Each tick runs this decision flow:

```
Every 5 seconds:
│
├─ Paused by user? ────── YES ──► Let PC sleep normally        [⚪ grey]
│   NO
├─ Teams running? ─────── NO ───► Do nothing, allow sleep      [🟠 orange]
│   YES
├─ Keep OS + display awake (SetThreadExecutionState)
│
├─ Idle for 60s or more? ─ NO ──► User is active, hands off    [🟢 green]
│   YES
└─ Press F15 once ──────────────► Teams stays "Available"      [🟢 green]
```

Because an F15 press itself resets Windows' idle timer, the loop naturally
settles into **about one invisible key press per minute** — no more than
needed.

### Tray icon at a glance

| Icon | State | Meaning |
|---|---|---|
| 🟢 Green | Active | Target app is running; PC is being kept awake; presence protected |
| 🟠 Orange | Waiting | KeepAlive is running, but the target app (Teams) was not found — doing nothing |
| ⚪ Grey | Paused | You paused it; PC can sleep normally |

Hover the icon for a live status tooltip (e.g. *"Active - F15 sent after
64s idle"*). Right-click for **Pause / Resume / Exit**. Double-click also
toggles pause.

---

## 🌟 Features

- Keeps **system and display** awake while active
- Keeps **Teams presence Available** by simulating activity only when idle
- **Never interferes while you work** — real input always wins
- **Process-aware** — completely inactive when Teams is closed
- Works with **new Teams (`ms-teams`) and classic Teams (`Teams`)**, and
  can target any other app (Slack, etc.) via one settings line
- **Pause / Resume / Exit** from the tray; everything (including the
  keep-awake request) is cleaned up on exit
- **Single-instance** — double-clicking twice won't run two copies
- **Self-healing** — keep-awake is re-applied every tick and after
  wake-from-sleep
- **Portable & tiny** — the compiled program is ~12 KB; no installer, no
  registry changes, no admin rights, no internet access, no data collected

---

## 📦 The single-file design (the other clever part)

The main download is **one file: `KeepAlive.bat`** — and it isn't just a
launcher. The bottom half of that same file contains the complete C#
source code of the app.

On first double-click it:

1. Copies the C# program out of its own bottom half,
2. Compiles it with **`csc.exe`, the C# compiler already built into every
   Windows** (`C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe`) —
   no Visual Studio, no .NET SDK, no downloads, no NuGet packages,
3. Stores the resulting `KeepAlive.exe` in `%LOCALAPPDATA%\KeepAlive`, and
4. Starts it in the tray.

After that, double-clicking simply starts it instantly. And if you edit
the file (settings or even the C# code) and double-click again, it notices
the change, rebuilds, and restarts itself. **One file to copy, run, edit,
and manage.**

> Why build it this way? Because the machines that need this most — locked
> down work laptops — are exactly the machines where you can't install
> developer tools. The built-in compiler is always there.

---

## 🚀 Quick start

1. Download **`KeepAlive.bat`** and put it anywhere (Desktop is fine).
2. **Double-click it.**
   - First run only: a small window shows *"Building KeepAlive…"* for a
     few seconds, then closes.
3. Look in the **system tray** (click the `^` arrow near the clock if
   needed) for the coloured dot. Open Teams — within ~5 seconds the dot
   should turn 🟢 green.
4. To stop: right-click the tray icon → **Exit**.

### Settings (inside the same file)

Right-click `KeepAlive.bat` → **Edit**. Near the top, one line controls
which app(s) to watch (process names without `.exe`, space-separated):

```bat
set "TARGETS=ms-teams Teams"
```

| Example | Watches |
|---|---|
| `ms-teams Teams` | Microsoft Teams — new and classic (default) |
| `Slack` | Slack only |
| `ms-teams Teams Slack` | Any of them |
| `chrome` / `msedge` | Teams running in a browser tab |

Save with **Ctrl+S**, double-click the file again — it restarts with the
new setting. (Process names can be found in **Task Manager → Details**.)

### Start automatically with Windows (optional)

Press `Win + R`, type `shell:startup`, and place a **shortcut** to
`KeepAlive.bat` in that folder.

---

## 🛠️ For developers

The repo also contains the classic two-file version:

| File | Purpose |
|---|---|
| `KeepAlive.bat` | ⭐ Single-file version — settings + launcher + full C# source in one file |
| `Program.cs` | The complete application as a standalone C# file (same code as inside the `.bat`) |
| `build.bat` | Compiles `Program.cs` → `KeepAlive.exe` using the in-box `csc.exe` |
| `README.md` | This file |

Build manually: put `Program.cs` and `build.bat` together, run
`build.bat`. Language: **C# / .NET Framework 4.8**, UI:
**WinForms `NotifyIcon` + `ApplicationContext`** (a message loop with no
window). No external libraries whatsoever.

### Windows APIs used (P/Invoke)

| API | DLL | Job |
|---|---|---|
| `SetThreadExecutionState` | `kernel32.dll` | Keep system + display awake; cleared on pause/exit |
| `GetLastInputInfo` | `user32.dll` | Measure real user idle time |
| `SendInput` | `user32.dll` | Inject the F15 key-down / key-up pair |
| `GetTickCount` | `kernel32.dll` | Idle arithmetic (wraparound-safe unsigned math) |
| `Process.GetProcessesByName` | .NET | Check whether the target app is running |
| `SystemEvents.PowerModeChanged` | .NET | Re-apply keep-awake after wake-from-sleep |

⚠️ **C# 5 only:** the in-box `csc.exe` understands at most C# 5, so the
source deliberately avoids `$"..."`, `?.`, `nameof()`, expression-bodied
members, etc. Keep that in mind when editing — modern syntax will fail to
compile with confusing errors.

---

## 🔒 What it does *not* do

- Does not move your mouse or type visible characters
- Does not read your screen, files, keystrokes, or network
- Does not connect to the internet or collect/send any data
- Does not require admin rights and makes no registry changes
- Does not do anything while you are actively using the PC
- Does not keep the PC awake when Teams is closed (unless you change
  the target)

---

## ⚠️ Limitations

- **Locked PC (`Win + L`):** injected keys have no effect on the secure
  lock-screen desktop, so presence may still lapse while locked. Leave
  the session unlocked if you rely on the status.
- **Display stays on** while active (by design — `ES_DISPLAY_REQUIRED`).
- **Browser Teams** has no `ms-teams` process — set the target to your
  browser (`chrome` / `msedge`) as shown above.
- **SmartScreen** may warn on first run because the exe is home-compiled
  and unsigned: *More info → Run anyway*.
- Presence status is not productivity. Please use responsibly and check
  your workplace policies — this tool is meant for short breaks, reading,
  thinking, calls away from the desk, and long-running tasks, not for
  pretending to work.

---

## 🧰 Troubleshooting

**Icon stays 🟠 orange ("Waiting for ms-teams / Teams…")**
KeepAlive is alive, but it can't find a process named `ms-teams` or
`Teams`. Check: (1) Is the Teams *desktop app* actually open (not just a
browser tab, and not fully closed)? (2) In **Task Manager → Details**,
what is the exact process name on your machine? Put that name (without
`.exe`) in the `TARGETS` settings line, save, and double-click again.

**No tray icon visible**
Click the `^` arrow near the clock — Windows hides new icons there. You
can drag the dot out to the main tray to always see its colour.

**Build fails**
Exit KeepAlive from the tray before rebuilding (a running `.exe` can't be
overwritten). If you edited the C# code, remember: C# 5 syntax only.

---

## 🚧 Current status & next steps

- ✅ Core app builds and runs; tray icon, states, menu, and tooltips work
- 🟠 **Under investigation:** on the first test machine the icon stayed
  orange and did not turn green — the target-process detection needs
  on-device debugging (confirm exact Teams process name via Task Manager)
- 🔜 Planned fix/upgrade: a small diagnostic mode that lists all running
  processes containing "team" in the tray menu, so the correct name can
  be spotted instantly

### 💭 Future ideas

- Settings UI in the tray (pick target app from a list of running apps)
- Adjustable idle threshold (currently 60 s) from the settings line
- Working-hours schedule (e.g. active only 09:00–18:00, Mon–Fri)
- "Keep system awake but let the display sleep" mode
- Optional debug log file for diagnosing detection issues

---

*KeepAlive is a tiny personal utility provided as-is, with no warranty.
Use at your own risk — and go enjoy that cup of tea. ☕*
