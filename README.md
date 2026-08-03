# gbs-ThreadExPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

Extra control over GB Studio's VM threads: pause a running thread — or an actor's **On Update** script — without terminating it, and resume it from the exact instruction where it stopped. It also lets scripts branch on whether a thread is running, paused or gone, and read how many thread contexts are still free.

The plugin neither replaces nor patches any stock engine file, so it needs no compatibility variants and composes with every other plugin.

https://github.com/user-attachments/assets/96f36d35-0092-4c0c-95a2-a66232674233

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Engine Settings](#engine-settings)
4. [Size Limits and Restrictions](#size-limits-and-restrictions)
5. [Events Reference](#events-reference)
6. [Media](#media)
7. [Memory Footprint](#memory-footprint)
8. [History](#history)
9. [License](#license)

---

## Concepts

### Threads and thread handles

GB Studio runs scripts as concurrent **threads**, and the engine has **16 thread contexts** to share between them. Every actor update script and every attached input script is itself a thread, so the pool is busier than it looks.

The stock **Thread Start** event stores a **thread handle** in a variable. That handle is what this plugin's thread events take, so anything you want to pause later has to be started into a variable you keep.

### Pausing is not stopping

The stock **Thread Stop** event destroys a thread: its position in the script is gone, and starting it again begins from the top.

**Thread Pause** instead freezes the thread exactly where it is and keeps everything it was doing:

- The thread keeps its own state, including a half-finished *Wait N frames* — resuming continues with the remaining frames intact.
- An actor paused mid-move resumes the move from wherever it stopped, rather than restarting it.
- A thread can pause itself and be resumed later by another thread.

A paused thread still holds its context and costs about one instruction per frame while frozen. Pausing needs no RAM of its own and leaves nothing behind when a thread is terminated or the scene changes.

### Thread states

**If Thread State** branches on what a handle is currently doing:

| State | Meaning |
|---|---|
| **Running** | The thread exists and is executing. |
| **Paused** | The thread exists but was stopped by **Thread Pause**. |
| **Terminated** | The thread has finished, was stopped with **Thread Stop**, was lost to a scene change, or the variable never held a thread. |

The three states are exhaustive, so testing for *Terminated* is the same as asking whether the thread is gone.

---

## Project Setup

1. Copy `src/ThreadExPlugin` into your project's `plugins/` folder, then restart GB Studio.
2. Under **Settings → Engine → Thread Ex**, enable the features you intend to use — see [Engine Settings](#engine-settings).

### Pausing a thread

Start the thread with the stock **Thread Start**, storing its handle in a variable. Pass that same variable to **Thread Pause** and **Thread Resume** from any other script.

### Pausing an actor's update script

Use **Actor Pause Update Script** and **Actor Resume Update Script** with an actor picker. No handle is involved — the events find the actor's update thread themselves, and the script is neither terminated nor restarted.

For pool or spawned actors that have no design-time actor to pick, use the **By Index** siblings and supply the actor index as a value. Index 0 is the player; scene actors are 1-based.

### Checking before you spawn

**Get Thread Count** reports how many of the 16 contexts are in use, so `16 − count` is how many more threads can start right now. Worth checking before spawning several at once.

---

## Engine Settings

**Settings → Engine → Thread Ex** has one checkbox per feature, so you only pay for what you use. With all four unchecked the plugin compiles to nothing at all.

| Setting | Events it enables | ROM |
|---|---|---|
| **Feature: Pause / resume threads** | Thread Pause, Thread Resume | 538 B |
| **Feature: Pause / resume actor update scripts** | Actor Pause / Resume Update Script (and the By Index variants) | 636 B |
| **Feature: Check thread state** | If Thread State | 192 B |
| **Feature: Read thread count** | Get Thread Count | 119 B |

Sizes are for that feature on its own. The two pause features share the pause and resume helpers, and the first three share the context lookup, so enabling several costs less than the sum — all four together is 997 B.

---

## Size Limits and Restrictions

### Sixteen thread contexts, shared

The engine has 16 contexts in total, and **paused threads keep theirs**. Starting a thread when none is free silently does nothing, so check **Get Thread Count** before spawning several at once.

### Two cases do nothing on purpose

- Pausing a thread that is already paused, or resuming one that is not paused.
- Pausing a thread that currently holds the VM lock — the VM would never switch away from it, so nothing could ever resume it.

### Thread handles do not survive a scene change

A handle from a previous scene reports **Terminated**. Re-start the thread in the new scene rather than trying to resume the old handle.

### Disabling a feature is checked at build time

Unchecking a feature removes its code from the ROM. Any event that needs it then fails the build with a message naming the setting to re-enable, rather than producing an obscure link error.

### No stock engine files are touched

The plugin adds only its own engine file, so it has no compatibility conflicts with any other plugin.

---

## Events Reference

---

### Thread Pause

**`EVENT_THREAD_EX_THREAD_PAUSE`** — groups: **Control Flow → Threads**, **Misc → Threads**

Freezes a running thread where it stands, keeping its position and its state. Does nothing if the thread is already paused, is gone, or currently holds the VM lock.

| Field | Description |
|-------|-------------|
| Thread handle variable | The variable holding the handle, as set by the stock **Thread Start**. |

---

### Thread Resume

**`EVENT_THREAD_EX_THREAD_RESUME`** — groups: **Control Flow → Threads**, **Misc → Threads**

Resumes a paused thread from the exact instruction where it stopped, including part-way through a wait. Does nothing if the thread is not paused.

| Field | Description |
|-------|-------------|
| Thread handle variable | The variable holding the handle. |

---

### If Thread State

**`EVENT_THREAD_EX_IF_THREAD_STATE`** — groups: **Control Flow → Threads**, **Misc → Threads**

Branches on whether a thread handle is currently running, paused or terminated. The Else branch can be collapsed away like any other conditional event.

| Field | Default | Description |
|-------|---------|-------------|
| Thread handle variable | — | The variable holding the handle. |
| Is | Running | The state to test for: **Running**, **Paused** or **Terminated**. |
| True | — | Script to run when the thread is in that state. |
| Else | — | Script to run otherwise. |

---

### Get Thread Count

**`EVENT_THREAD_EX_GET_THREAD_COUNT`** — groups: **Control Flow → Threads**, **Misc → Threads**

Stores how many of the engine's 16 thread contexts are currently in use. Paused threads are counted, since they keep their context, and so is the script running the event, since it is itself a thread.

| Field | Description |
|-------|-------------|
| Store in | Variable that receives the count. |

---

### Actor Pause Update Script

**`EVENT_THREAD_EX_UPDATE_SCRIPT_PAUSE`** — group: **Actor → Script**

Pauses an actor's **On Update** script without terminating or restarting it.

| Field | Description |
|-------|-------------|
| Actor | The actor whose update script to pause. |

---

### Actor Resume Update Script

**`EVENT_THREAD_EX_UPDATE_SCRIPT_RESUME`** — group: **Actor → Script**

Resumes a paused actor **On Update** script from where it stopped.

| Field | Description |
|-------|-------------|
| Actor | The actor whose update script to resume. |

---

### Actor Pause Update Script By Index

**`EVENT_THREAD_EX_UPDATE_SCRIPT_PAUSE_BY_INDEX`** — group: **Actor → Script**

The same, with the actor given as a value or variable instead of an actor picker — for pool or spawned actors that have no design-time actor to pick.

| Field | Description |
|-------|-------------|
| Actor Index | Index of the actor whose update script to pause. Index 0 is the player; scene actors are 1-based. |

---

### Actor Resume Update Script By Index

**`EVENT_THREAD_EX_UPDATE_SCRIPT_RESUME_BY_INDEX`** — group: **Actor → Script**

The by-index counterpart of **Actor Resume Update Script**.

| Field | Description |
|-------|-------------|
| Actor Index | Index of the actor whose update script to resume. Index 0 is the player; scene actors are 1-based. |

---

## Media

`ThreadExTest/` is a playable demo. Three actors pace back and forth across the scene, each driven differently:

| Actor | Driven by | Frozen by |
|---|---|---|
| Runner A | its own **On Update** script | Actor Pause Update Script |
| Runner B | a background thread started in On Init | Thread Pause, by handle |
| Runner C | its own **On Update** script | nothing — it never stops |

| Button | Does |
|---|---|
| **A** | Freezes A and B |
| **B** | Unfreezes both |
| **START** | Reports B's state (running / paused / stopped) and the thread count |
| **SELECT** | Terminates B's thread with the stock Thread Stop |

Runner C is the control: it keeps moving while A and B are frozen, which shows that pausing acts on one thread and not on the VM as a whole. Because the actors are paused *mid-slide*, resuming continues the move from wherever it stopped rather than restarting it.

Press SELECT then START to see the third state: once a thread is really gone, Thread Resume can do nothing for it. Watch the thread count change too — every actor update script and every button press is itself a thread.

---

## Memory Footprint

| | Cost |
|---|---|
| WRAM | +0 bytes |
| SRAM | +0 bytes |
| ROM | +997 bytes with every feature enabled |

- **WRAM / HRAM / SRAM:** none. A paused thread's resume point is kept on that thread's own stack, so pausing needs no memory of its own and leaves nothing behind when a thread ends or the scene changes.
- **ROM:** 997 bytes with all four features on, and less with any of them unchecked — see [Engine Settings](#engine-settings) for the per-feature figures. Using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks.
- **Thread contexts:** a paused thread keeps its context, so it still counts against the engine's 16.

---

## License

MIT

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |

**This plugin costs nothing in bank 0.** Every one of its functions is compiled
into a switchable ROM bank; nothing it adds is resident in bank 0.
<!-- BANK0:END -->
