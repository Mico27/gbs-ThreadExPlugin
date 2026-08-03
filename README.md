# GB Studio — Thread Ex Plugin

Extra control over GB Studio's VM threads: pause a running thread — or an actor's
**On Update** script — without terminating it and resume it from the exact
instruction where it stopped, branch on whether a thread is running, paused or
gone, and read how many thread contexts are still free.

* **Type:** engine plugin
* **Target:** GB Studio 4.3.0+
* **Stock engine files touched:** none
* **RAM used:** 0 bytes (WRAM / HRAM / SRAM)
* **ROM used:** 997 bytes with every feature enabled (individually toggleable in Settings → Engine → Thread Ex)

Because it neither replaces nor patches any stock engine file, this plugin has
no `engineAlt/` variants and composes with every other plugin.

---

## Installation

Copy `src/ThreadExPlugin` into your project's `plugins/` folder, then restart
GB Studio.

`ThreadExTest/` is a playable example project - see below.

---

## Events

### Thread Pause / Thread Resume
*Control Flow → Threads, and Misc → Threads*

Takes a thread handle variable, as set by the stock **Thread Start**.

* The thread keeps its VM stack, including a half-finished `Wait N frames`,
  which resumes with the remaining frames intact.
* A thread can pause itself; another thread resumes it later.

### If Thread State
*Control Flow → Threads, and Misc → Threads*

Branches on what a thread handle is currently doing, with a dropdown for which
state to test:

| State | Meaning |
|---|---|
| Running | The thread exists and is executing |
| Paused | The thread exists but was stopped by **Thread Pause** |
| Terminated | The thread has finished, was stopped with **Thread Stop**, was lost to a scene change, or the variable never held a thread |

The three states are exhaustive, so testing for *Terminated* is the same as
asking whether the thread is gone. The Else branch can be collapsed away like
any other conditional event.

### Get Thread Count
*Control Flow → Threads, and Misc → Threads*

Stores how many of the engine's 16 thread contexts are currently in use. Paused
threads are counted - they keep their context - and so is the script running the
event, since it is itself a thread. `16 - count` is how many more threads can be
started right now, which is worth checking before spawning several at once:
`script_execute` silently does nothing when no context is free.

### Actor Pause Update Script / Actor Resume Update Script
*Actor → Script*

The same thing applied to an actor's **On Update** script. The script is not
terminated and not restarted.

Both take an actor picker, and each has a **By Index** sibling (*Actor Pause /
Resume Update Script By Index*) that takes a raw actor index as a script value
instead, for addressing pool or spawned actors that have no design-time actor to
pick. Index 0 is the player, scene actors are 1-based.

---

## How pausing works

Pausing swaps the thread's program counter and bank for a tiny stub that idles
and jumps to itself:

```
loop:  VM_IDLE
       VM_JUMP loop
```

so the paused context stays in the VM runner's list and costs about one
instruction per frame. The resume point is pushed onto the paused thread's *own*
VM stack — three words above its stack pointer, deliberately skipping the first
slot because `VM_INVOKE` handlers such as `wait_frames` keep their counter there
without reserving it. That is why pausing needs no RAM of its own and leaves
nothing behind when a thread is terminated or the scene changes.

Two cases do nothing on purpose:

* pausing a thread that is already paused, or resuming one that is not paused;
* pausing a thread that currently holds the VM lock — the VM would never switch
  away from it and nothing could resume it.

---

## Example project

`ThreadExTest/` is a playable demo of the plugin. Three actors pace back and
forth across the scene, each driven differently:

| Actor | Driven by | Frozen by |
|---|---|---|
| Runner A | its own **On Update** script | Actor Pause Update Script |
| Runner B | a background thread started in On Init | Thread Pause, by handle |
| Runner C | its own **On Update** script | nothing - it never stops |

Controls:

| Button | Does |
|---|---|
| **A** | Freezes A and B |
| **B** | Unfreezes both |
| **START** | Reports B's state (running / paused / stopped) and the thread count |
| **SELECT** | Terminates B's thread with the stock Thread Stop |

Runner C is the control: it keeps moving while A and B are frozen, which shows
that pausing acts on one thread and not on the VM as a whole. Because the actors
are paused *mid-slide*, resuming continues the move from wherever it stopped
rather than restarting it.

Press SELECT then START to see the third state: once a thread is really gone,
Thread Resume can do nothing for it. Watch the thread count change too - every
actor update script and every button press is itself a thread.

## Engine settings

Settings → Engine → **Thread Ex** has one checkbox per feature. Unchecking one
removes its code from the ROM; any event that needs it then fails the build with
a message naming the setting, rather than producing a linker error. With all four
unchecked the plugin compiles to nothing at all.

| Setting | Events | ROM |
|---|---|---|
| Pause / resume threads | Thread Pause, Thread Resume | 538 B |
| Pause / resume actor update scripts | Actor Pause Update Script, Actor Resume Update Script | 636 B |
| Check thread state | If Thread State | 192 B |
| Read thread count | Get Thread Count | 119 B |

Sizes are for that feature on its own; the two pause features share the
pause/resume helpers and all three of the first ones share the context lookup, so
enabling several costs less than the sum (all four together is 997 B).

## Layout

```
src/ThreadExPlugin/
  plugin.json
  engine/
    engine.json
    include/thread_ex.h
    src/thread_ex.c
  events/
    eventThreadPause.js
    eventThreadResume.js
    eventIfThreadState.js
    eventGetThreadCount.js
    eventActorUpdateScriptPause.js
    eventActorUpdateScriptPauseByIndex.js
    eventActorUpdateScriptResume.js
    eventActorUpdateScriptResumeByIndex.js
    eventInternalPauseStub.js     <- internal, hidden from the event menu
```

The internal event exists only so the pause stub can be emitted as a regular
compiled script. It is marked deprecated so it never shows up in the Add Event
menu.

## History

This plugin started out as "Actor Tools", then briefly "Pause Resume Thread". The actor-side events — iterating
actors in an area, extended actor property get/set, forcing an actor script to
run, and the actor activated / deactivated callbacks — now live in the
[Dynamic Actor plugin](https://github.com/Mico27/gbs-DynamicActorPlugin).

## License

MIT
