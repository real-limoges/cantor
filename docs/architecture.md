# Architecture

Reference doc for Cantor's design. Ported from Funktor's `docs/architecture.md` with Clojure/Overtone idioms substituted. Read this before any structural change.

Synthesis runs in `scsynth`. Clojure owns the pattern DSL, scheduler, REPL session, MIDI input, and Launchpad grid. Sample-level DSP is `scsynth`'s job.

## Namespace Layers

Each layer depends only on what's below it.

```
cantor.live                 REPL interface: play / stop / set-tempo / start-midi / start-launchpad
  │
  ├── cantor.audio.scheduler     wall-clock event scheduler + hot-swap
  │     │
  │     └── cantor.audio         facade: open-device / note-on / note-off (wraps Overtone)
  │           │
  │           └── synthdefs in   src/cantor/synthdefs.clj — defsynth forms
  │
  ├── cantor.hardware.midi       MIDI input via overtone.midi + router (core.async chan)
  │
  ├── cantor.hardware.launchpad  Mk3 SysEx; midi ↔ pad translation (pure)
  │
  ├── cantor.grid.binding        mode dispatcher: :sequencer / :instrument / :scene
  │     │
  │     └── cantor.grid          pad / color / pad-action — pure data
  │
  └── cantor.core.stream         arc -> [event]; periodic / cat / stack / slow / fast
        │
        └── cantor.core.types    beat (ratio), arc, event, pitch, velocity
```

## Runtime Model

A `(play stream)` call boots one scheduler thread that talks to scsynth via Overtone's OSC layer.

```
REPL                       Scheduler thread          scsynth (OS process)
─────────────────────────────────────────────────────────────────────────
(play stream)
  ├─ ensure scsynth via Overtone ────UDP socket────> 127.0.0.1:57110
  ├─ initial scheduler state                             │
  └─ (future ...) ────> scheduler loop                   │
                            │                            │
                            │  every ~10ms:              │
                            │   1. (System/nanoTime)     │
                            │   2. pop due events ───────┼── /s_new freq amp ...
                            │   3. query stream window   │   /n_set id gate 0
                            │   4. enqueue new           │   /n_free id (steal)
                            │   5. park                  │
                            ▼                            ▼
                        atom scheduler-state         scsynth voice pool
                                                     EnvGen :action FREE
```

State coordination uses **atoms**, not Haskell TVars. STM-grade transactions weren't load-bearing in Funktor — single-writer atoms with `swap!` are simpler and equally correct here.

- **`scheduler-state` (atom)** — current stream, tempo, transport beat, pending events. Written by `play` / `set-tempo` (hot-swap), grid commits, MIDI router, and the scheduler loop.
- **`live-state` (atom in `cantor.live`)** — session handle. Survives REPL evaluations natively; no `foreign-store` equivalent needed.

`cantor.audio` owns its own per-session state (active-voice map by pitch). Overtone gives us node objects directly; we may not even need a manual Pitch→NodeId map (TBD during implementation).

### Process lifecycle (TBD)

`cantor.audio` must register a JVM shutdown hook that stops scsynth cleanly:

```clojure
(.addShutdownHook (Runtime/getRuntime)
  (Thread. #(overtone.core/kill-server)))
```

Without it, a JVM crash (not a clean exit) leaves an orphaned scsynth holding the audio device and port 57110, and the next jack-in fails with a confusing "address in use" error. `pkill scsynth` is the manual recovery.

Decide during step 3 of the build order (`cantor.audio` facade) whether the hook lives in `cantor.audio` (registered on first `(open!)`) or in `cantor.live` (registered on first `(play ...)`). The first is more honest about ownership; the second is simpler to reason about.

## Key Design Decisions

### Native ratios for beats

Clojure has `clojure.lang.Ratio` built-in. Triplets are literally `1/3` in source code:

```clojure
(def triplet-eighth 1/12)   ;; reads as a Ratio, not a Double
```

Convert to seconds only at the audio boundary (scheduler → OSC `bundle-at`).

### Stream as an arc-indexed query function

```clojure
;; stream is just a function: arc -> seq of events
(defn periodic [period value]
  (fn [{:keys [start end]}]
    (for [n (range (Math/ceil start) end)]
      {:whole {:start n :end (+ n period)}
       :part  {:start n :end (+ n period)}
       :value value})))
```

Not a lazy seq. The scheduler asks "events whose `:part :start` falls in this arc" each tick and gets back exactly those. No materialization of unused future events.

Smart constructors `periodic`, `cat`, `stack`, `slow`, `fast` mirror funktor's `Funktor.Core.Stream`. Port them; the algebra is the gem and it's language-independent.

### Synthesis lives in scsynth

Voice pool, oscillators, envelopes, filters all run inside `scsynth`. Overtone's `definst` / `defsynth` compiles Clojure → SuperCollider bytecode and ships it to the server. Cantor sends `(synth :freq f :amp a)` to spawn and `(ctl node :gate 0)` to release; `EnvGen :action FREE` runs the release tail and frees the node.

We do *not* maintain Haskell-style `Funktor.Audio.Timbre` records — Overtone synth defs are first-class values, so a "timbre" is just a synth + a map of param overrides.

### Out-of-band MIDI routing

MIDI input goes onto a `core.async` channel. A router thread takes from the channel and uses `swap!` on `scheduler-state` to enqueue immediate events. The scheduler tick never blocks on MIDI.

Equivalent to Funktor's `PortMidi → TQueue → enqueueImmediate` pattern, but with `core.async/chan` instead of `STM TQueue`.

### Monotonic clock

`(System/nanoTime)` for scheduling. Immune to wall-clock jumps (NTP, DST).

### No reload machinery

This is the big one. Funktor needed `foreign-store` + `fsnotify` to survive GHCi `:reload`. **Clojure's REPL IS the live image** — `(defn play [...] ...)` redefines the var, the running scheduler thread keeps holding the same atom, and the next tick reads the new behavior. No persistence layer needed.

The entire `Funktor.Live.Reload` namespace simply does not have a Cantor equivalent.

## Constraints

- Boring Clojure: prefer `defn` + data over macros + protocols unless a macro genuinely earns its keep.
- One REPL session per work session — design assumes a long-running JVM. Don't add features that require restarting the JVM.
- Pure data flows downward; side effects only in `cantor.audio` and `cantor.hardware.*`.
- Every namespace gets a `test/cantor/.../X_test.clj` spec. `clojure.test` is enough — no need for kaocha/midje unless it earns its keep.
