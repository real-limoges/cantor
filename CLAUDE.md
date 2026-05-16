# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Greenfield. Only `docs/`, `README.md`, `LICENSE`, `.gitignore` exist — no `src/`, no `deps.edn`, no `project.clj` yet. Treat `docs/architecture.md` and `docs/from-funktor.md` as the spec; they describe the intended namespace layering and design decisions that any new code must respect.

Cantor is the Clojure rewrite of [funktor](../funktor) (Haskell). The pattern algebra and scheduler design transfer; the Haskell plumbing (`foreign-store`, `fsnotify` reload watcher, `TVar`/STM, vendored PortMidi, cabal flags, coverage gates) deliberately does not. Before reproducing anything from funktor, check `docs/from-funktor.md` — it lists what to port, what to adapt, and what to delete from your mental model.

## Workflow

This is a REPL-first project. There is no build/test/run pipeline in the traditional sense:

- `clj` starts the REPL; `(use 'overtone.live)` boots `scsynth` on UDP 57110.
- Edit a file, eval the top-level form (Calva: `Ctrl+Alt+C Enter`), hear the change. The running scheduler thread keeps holding the same atom; redefining a var changes the next tick's behavior.
- Tests use `clojure.test` only. One spec per namespace at `test/cantor/.../X_test.clj`.
- Linting with `clj-kondo` is optional — don't gate commits on it.

The JVM is expected to stay up for the entire work session. Don't add features that require restarting it.

## Architecture invariants

These come from `docs/architecture.md`. Violating them will compound into the same friction the Clojure rewrite exists to escape.

- **Namespace layers are strictly downward-only.** `cantor.live` → `audio.scheduler` → `audio` → synthdefs; `live` → `hardware.midi`, `hardware.launchpad`, `grid.binding` → `grid`; everything sits on `core.stream` → `core.types`. Don't introduce upward or sideways deps.
- **Pure data flows down; side effects live only in `cantor.audio` and `cantor.hardware.*`.** Stream/grid/types are pure.
- **State is single-writer atoms, not STM.** The two top-level atoms are `scheduler-state` (current stream, tempo, transport beat, pending events) and `live-state` (session handle in `cantor.live`). Don't reach for refs/agents/STM "for safety" — single-writer `swap!` is sufficient and was load-bearing-tested in funktor.
- **`Beat` is `clojure.lang.Ratio`, not double.** Triplets are literally `1/12` in source. Convert to seconds only at the OSC boundary in the scheduler.
- **Streams are functions `arc -> [event]`, not lazy seqs.** The scheduler queries a window each tick; no materialization of unused future events. Smart constructors `periodic` / `cat` / `stack` / `slow` / `fast` mirror funktor's algebra.
- **Synthesis runs in `scsynth`.** Clojure never does sample-level DSP. A "timbre" is just an Overtone synth value plus a param map — do not introduce a `Timbre` record/type.
- **MIDI is out-of-band.** Input goes onto a `core.async` channel; a router thread `swap!`s into `scheduler-state`. The scheduler tick must never block on MIDI.
- **Monotonic time only.** `(System/nanoTime)`, never `(System/currentTimeMillis)`.
- **No reload machinery.** Clojure's REPL is the live image. Do not port `Funktor.Live.Reload`, file watchers, or `foreign-store` equivalents.
- **Boring Clojure.** Prefer `defn` + data over macros + protocols unless a macro genuinely earns its keep.

## Suggested build order

From `docs/from-funktor.md` — each step ends with something audible:

1. `cantor.core.types` + `cantor.core.stream` (Beat, Arc, Event, `periodic`)
2. Overtone boot + simple `definst` (confirm scsynth round-trip)
3. `cantor.audio` thin facade — `(play-stream stream)` schedules events
4. Hot-swap — re-evaluating the stream var changes playback without stopping the scheduler
5. `cantor.hardware.midi`
6. `cantor.hardware.launchpad` (Mk3 programmer mode)
7. `cantor.grid.binding` (sequencer mode commits hot-swap the stream)
8. `cantor.live` REPL helpers (`play`, `stop`, `set-tempo`, ...)
