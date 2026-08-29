# gbs-ThreadExPlugin

**Version 4.3.0. Requires GB Studio 4.3.0 or newer.**

Pauses a running script, or an actor's **On Update** script, and later starts it again from exactly where it stopped.

GB Studio can only stop a script, which throws away its position. This plugin freezes it instead. A half-finished wait keeps its remaining frames, an actor stopped mid-walk carries on from where it was, and nothing restarts from the top. That is what a pause menu, a cutscene that holds the world still, or a stun effect on one enemy needs.

It also lets a script ask whether another script is running, paused or gone, and how many script slots are still free.

No stock engine file is changed, so it works alongside every other plugin with no variants needed.

https://github.com/user-attachments/assets/96f36d35-0092-4c0c-95a2-a66232674233

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Engine Settings](#engine-settings)
4. [Size Limits and Restrictions](#size-limits-and-restrictions)
5. [Events Reference](#events-reference)
6. [FAQ](#faq)
7. [Media](#media)
8. [Memory Footprint](#memory-footprint)
9. [License](#license)
10. [Bank 0 (HOME) Usage](#bank-0-home-usage)
11. [Changelog](#changelog)

---

## Concepts

### Threads and handles

GB Studio runs scripts side by side as **threads**, and there are **16 slots** to share between them. Every actor update script and every attached input script takes one, so the pool fills faster than you might expect.

The stock **Thread Start** event puts a **handle** in a variable. That handle is what this plugin's thread events take, so anything you want to pause later has to be started into a variable you keep.

### What pausing keeps

The stock **Thread Stop** event destroys a thread. Its position is gone and starting it again begins from the top.

**Thread Pause** freezes the thread where it stands and keeps everything it was doing:

- A half-finished **Wait N frames** keeps its remaining frames, so resuming carries on counting.
- An actor paused mid-move continues the move from where it stopped.
- A thread can pause itself and be started again by another thread.

A paused thread keeps its slot and costs about one instruction per frame while frozen. Pausing uses no memory of its own and leaves nothing behind when a thread ends or the scene changes.

### Thread states

**If Thread State** branches on what a handle is currently doing:

| State | Meaning |
|---|---|
| **Running** | The thread exists and is executing. |
| **Paused** | The thread exists but was stopped by **Thread Pause**. |
| **Terminated** | The thread has finished, was stopped with **Thread Stop**, was lost to a scene change, or the variable never held a thread. |

Those three cover every case, so testing for **Terminated** is the same as asking whether the thread is gone.

---

## Project Setup

1. Copy `src/ThreadExPlugin` into your project's `plugins` folder, then restart GB Studio.
2. Under **Settings**, then **Engine**, then **Thread Ex**, turn on the features you plan to use. See [Engine Settings](#engine-settings).

### Pausing a thread

Start the thread with the stock **Thread Start**, storing its handle in a variable. Pass that same variable to **Thread Pause** and **Thread Resume** from any other script.

### Pausing an actor's update script

Use **Actor Pause Update Script** and **Actor Resume Update Script**, picking the actor from the scene. No handle is needed, because the events find the actor's script themselves, and it is neither stopped nor restarted.

For pooled or spawned actors with nothing to pick in the editor, use the **By Index** versions and give the actor number. 0 is the player, and the scene's actors start at 1.

### Checking before you spawn

**Get Thread Count** reports how many of the 16 slots are in use, so 16 minus that is how many more threads can start. Worth checking before starting several at once.

---

## Engine Settings

**Settings**, then **Engine**, then **Thread Ex** has one tickbox per feature, so you only pay for what you use. With all four unticked the plugin adds nothing at all.

| Setting | Events it enables | ROM |
|---|---|---|
| **Feature: Pause / resume threads** | Thread Pause, Thread Resume | 538 B |
| **Feature: Pause / resume actor update scripts** | Actor Pause and Resume Update Script, plus their By Index versions | 636 B |
| **Feature: Check thread state** | If Thread State | 192 B |
| **Feature: Read thread count** | Get Thread Count | 119 B |

Each size is for that feature on its own. The two pause features share code, and the first three share the lookup, so turning on several costs less than adding the numbers up. All four together is 997 B.

---

## Size Limits and Restrictions

### Sixteen thread contexts, shared

There are 16 slots in total, and **a paused thread keeps its own**. Starting a thread when none is free does nothing at all, so check **Get Thread Count** before starting several.

### Two cases do nothing on purpose

- Pausing a thread that is already paused, or resuming one that is not paused.
- Pausing a thread that currently holds the script lock, because nothing else would ever get to run and resume it.

### Thread handles do not survive a scene change

A handle from a previous scene reports **Terminated**. Start the thread again in the new scene.

### Disabling a feature is checked at build time

Unticking a feature removes its code from the ROM. An event that needs it stops the build with a message naming the setting to turn back on.

### No stock engine files are touched

The plugin adds only its own engine file, so it has no conflicts with any other plugin.

---

## Events Reference

---

### Thread Pause

Groups: **Control Flow**, under **Threads**, and **Misc**, under **Threads**.

Freezes a running thread where it stands, keeping its position and its state. Does nothing when the thread is already paused, is gone, or holds the script lock.

| Field | Description |
|-------|-------------|
| Thread handle variable | The variable holding the handle, as set by the stock **Thread Start**. |

---

### Thread Resume

Groups: **Control Flow**, under **Threads**, and **Misc**, under **Threads**.

Resumes a paused thread from the exact instruction where it stopped, including part-way through a wait. Does nothing if the thread is not paused.

| Field | Description |
|-------|-------------|
| Thread handle variable | The variable holding the handle. |

---

### If Thread State

Groups: **Control Flow**, under **Threads**, and **Misc**, under **Threads**.

Branches on whether a thread handle is currently running, paused or terminated. The Else branch can be collapsed away like any other conditional event.

| Field | Default | Description |
|-------|---------|-------------|
| Thread handle variable | none | The variable holding the handle. |
| Is | Running | The state to test for: **Running**, **Paused** or **Terminated**. |
| True | none | Script to run when the thread is in that state. |
| Else | none | Script to run otherwise. |

---

### Get Thread Count

Groups: **Control Flow**, under **Threads**, and **Misc**, under **Threads**.

Stores how many of the 16 slots are in use. Paused threads count, because they keep their slot, and so does the script running the event.

| Field | Description |
|-------|-------------|
| Store in | Variable that receives the count. |

---

### Actor Pause Update Script

Group: **Actor**, under **Script**.

Pauses an actor's **On Update** script without terminating or restarting it.

| Field | Description |
|-------|-------------|
| Actor | The actor whose update script to pause. |

---

### Actor Resume Update Script

Group: **Actor**, under **Script**.

Resumes a paused actor **On Update** script from where it stopped.

| Field | Description |
|-------|-------------|
| Actor | The actor whose update script to resume. |

---

### Actor Pause Update Script By Index

Group: **Actor**, under **Script**.

The same, with the actor given as a value or variable rather than picked from the scene. Use it for pooled or spawned actors.

| Field | Description |
|-------|-------------|
| Actor Index | Which actor's update script to pause. 0 is the player, and the scene's actors start at 1. |

---

### Actor Resume Update Script By Index

Group: **Actor**, under **Script**.

The by-index counterpart of **Actor Resume Update Script**.

| Field | Description |
|-------|-------------|
| Actor Index | Which actor's update script to resume. 0 is the player, and the scene's actors start at 1. |

---

## FAQ

**How do I build a pause menu that freezes the whole world?**
Pause each actor's update script with **Actor Pause Update Script**, and pause any background
threads by handle. Everything resumes mid-step when you unpause, so a walking NPC finishes the
step it was taking.

**How do I stun one enemy for a second?**
Call **Actor Pause Update Script** on it, **Wait 60 frames**, then **Actor Resume Update Script**.
It carries on from where it was rather than restarting its patrol.

**Why can I not just use Thread Stop and Thread Start?**
Stopping throws the script's position away, so starting it again begins from the top. A patrolling
guard would teleport back to the start of its route.

**Does a half-finished Wait survive a pause?**
Yes. A **Wait 60 frames** paused at frame 20 has 40 frames left when it resumes.

**My Thread Pause did nothing.**
Three usual reasons: the thread was already paused, the handle is from a previous scene so the
thread is gone, or the thread holds the script lock, in which case pausing it would leave nothing
running to resume it.

**How do I check whether something is still running?**
Use **If Thread State** with the handle. It branches on **Running**, **Paused** or **Terminated**,
and those three cover every case.

**My new threads silently fail to start.**
All 16 slots are in use. Check with **Get Thread Count** first. Remember that paused threads keep
their slots, and every actor update script and attached input script takes one.

**Do I need a handle to pause an actor's update script?**
No. The actor events find it themselves. Use the **By Index** versions for spawned or pooled actors
where there is nothing to pick in the editor.

**Can I keep a handle across a scene change?**
No. The thread is gone and the handle reports **Terminated**. Start the thread again in the new
scene.

**How do I keep the ROM cost down?**
Untick the features you do not use. All four together is 997 bytes, and each one on its own is
listed under [Engine Settings](#engine-settings).

**Does it clash with other plugins?**
No. It changes no stock engine files.

---

## Media

`ThreadExTest/` is a playable demo. Three actors pace back and forth across the scene, each driven differently:

| Actor | Driven by | Frozen by |
|---|---|---|
| Runner A | its own **On Update** script | Actor Pause Update Script |
| Runner B | a background thread started in On Init | Thread Pause, by handle |
| Runner C | its own **On Update** script | nothing, it never stops |

| Button | Does |
|---|---|
| **A** | Freezes A and B |
| **B** | Unfreezes both |
| **START** | Reports B's state (running / paused / stopped) and the thread count |
| **SELECT** | Terminates B's thread with the stock Thread Stop |

Runner C keeps moving while A and B are frozen, which shows that pausing acts on one script and not on the game as a whole. The actors are paused mid-slide, so resuming carries the move on from where it stopped.

Press SELECT then START to see the third state. Once a thread is gone, Thread Resume can do nothing for it. Watch the thread count move as well, since every actor update script and every button press takes a slot.

---

<!-- SETTINGCOST:BEGIN -->
### What each engine setting costs

Each setting changes what gets compiled. Figures are what you **get back by turning
the setting off**. Rows marked *off by default* show what turning it **on** costs, and
sliders show the cost per step. "none" means that budget does not move.

| Setting | Bank 0 | WRAM | Banked ROM |
|---|---|---|---|
| Feature: Pause / resume threads | none | none | **92 B** |
| Feature: Pause / resume actor update scripts | none | none | **190 B** |
| Feature: Check thread state | none | none | **150 B** |
| Feature: Read thread count | none | none | **119 B** |

Turning off every on-by-default switch above frees **551 B** of banked ROM. That is the
span between the plugin at its fullest and stripped to nothing, so treat it as a
ceiling. You keep whatever your game actually uses.

<details><summary>How these were measured</summary>

GB Studio 4.3.0-e1. This plugin's engine code was compiled with the toolchain and
flags GB Studio itself uses, and the size of each part of the result was read back and
sorted into the three budgets: the fixed bank 0, work RAM, and switchable ROM banks.

Two caveats. Only this plugin's own engine sources are measured, so a setting that also
changes a shared data structure can move a few more bytes elsewhere. And each setting is
toggled on its own, so a few measure slightly *negative* when enabling their code lets
the compiler drop a fallback path, and a setting that gates other settings shows only
its own contribution.

</details>
<!-- SETTINGCOST:END -->

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine at default engine settings, report of 2026-08-13. Figures are the difference against a stock project: a file that replaces a stock engine file counts only the change, which is why a plugin can come out negative. Each event you use also compiles a few bytes of script into your project, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | 0 bytes |
| WRAM | 0 bytes |
| Banked ROM | +997 bytes |

- **Bank 0:** nothing. Everything the plugin adds is compiled into a switchable ROM bank.
- **WRAM and SRAM:** none. A paused thread's resume point is kept on that thread's own stack, so pausing needs no memory of its own and leaves nothing behind when a thread ends or the scene changes.
- **Banked ROM:** 997 bytes with all four features on, and less with any of them turned off. See [What each engine setting costs](#what-each-engine-setting-costs).
- **Thread slots:** a paused thread keeps its slot, so it still counts against the 16.

---

## License

MIT

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB fixed ROM bank shared by the GB Studio engine core, the
interrupt handlers and the GBDK runtime. Extra banked ROM is cheap to add,
bank 0 is not, so bank 0 is usually the first thing a project runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |

**This plugin costs nothing in bank 0.** Everything it adds is compiled into a
switchable ROM bank.
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-08-03

- Initial release: pause and resume threads and actor update scripts without stopping them, plus a thread state check and a thread count reader.
