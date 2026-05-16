# Clojure + VSCode (Calva) — getting un-rusty

Practical refresher aimed at someone who has written Clojure before but not recently. Covers the editor setup first because nothing else matters until eval-on-keystroke works, then a tour of the language features you'll actually hit in Cantor.

---

## 1. VSCode setup

### Extensions

Install one extension; the rest is configuration.

- **Calva** (`betterthantomorrow.calva`) — Clojure language support, REPL, structural editing, inline eval, debugger. Bundles `clj-kondo` for live linting.

Optional but worth it:

- **Calva Spritz** (`betterthantomorrow.calva-spritz`) — Joyride-powered extras (extra commands, keybinding helpers).
- **Rainbow Brackets** — color-matched parens. Vital when nesting gets deep.

### One-time configuration

Open settings (JSON) and add:

```json
{
  "calva.paredit.defaultKeyMap": "strict",
  "calva.evalOnSave": false,
  "calva.prettyPrintingOptions": { "enabled": true, "printEngine": "pprint" },
  "editor.formatOnType": true,
  "editor.autoClosingBrackets": "always"
}
```

`strict` paredit prevents you from deleting a single paren and unbalancing the form. Trust it — fighting paredit is the #1 source of Calva frustration for returning users.

### Jacking in

"Jack-in" = Calva starts the REPL with the right deps and connects to it. From a `.clj` file:

- `Ctrl+Alt+C Ctrl+J` — Jack-in. Pick `deps.edn` (or `Leiningen` if a `project.clj` exists). For Cantor: deps.edn.
- You'll get a prompt for aliases. None needed for a bare REPL; pick `:dev` once we have one.
- A REPL window opens. The status bar shows `cljREPL` and the current namespace.

Once connected, eval works in any `.clj` file:

| Action                          | Keybinding                  |
|---------------------------------|-----------------------------|
| Eval top-level form (most used) | `Ctrl+Alt+C Enter`          |
| Eval current form (under caret) | `Ctrl+Alt+C E`              |
| Eval and inline-show result     | `Ctrl+Alt+C V`              |
| Eval and replace with result    | `Ctrl+Alt+C R`              |
| Load current file               | `Ctrl+Alt+C Ctrl+L`         |
| Switch REPL namespace to file's | `Ctrl+Alt+C N`              |
| Interrupt running eval          | `Ctrl+Alt+C Ctrl+C`         |
| Clear REPL output               | `Ctrl+Alt+C Ctrl+R`         |

If you remember nothing else, remember `Ctrl+Alt+C Enter`. That's the entire workflow: edit a form, press it, hear the change.

### Structural editing (paredit)

Once you're in strict mode, edit by structure, not by character. The 80/20:

| Action                       | Keybinding             |
|------------------------------|------------------------|
| Slurp forward (pull next in) | `Ctrl+Right`           |
| Barf forward (push last out) | `Ctrl+Left`            |
| Slurp backward               | `Ctrl+Alt+Left`        |
| Barf backward                | `Ctrl+Alt+Right`       |
| Wrap selection in `()`       | `Ctrl+Alt+P Ctrl+P`    |
| Wrap selection in `[]`       | `Ctrl+Alt+P Ctrl+S`    |
| Splice (unwrap)              | `Ctrl+Alt+S`           |
| Raise (replace parent)       | `Ctrl+Alt+P R`         |
| Kill form forward            | `Ctrl+Alt+K`           |

`slurp` and `barf` are the verbs that distinguish Lisp editing from text editing. Practice them on a one-liner:

```clojure
;; cursor inside (+ 1 2), press Ctrl+Right →
(+ 1 2 3)
;; press Ctrl+Left →
(+ 1 2) 3
```

### Inline eval & inspecting results

`Ctrl+Alt+C V` prints results as a faded annotation next to the form. Hover over it for a copy/expand pop-up. For maps/seqs, click the result to open the Calva Inspector tree view.

For Cantor specifically: re-evaluating any `def` or `defn` redefines the var in-place. The scheduler thread keeps its atom; the next tick reads the new var. This is the whole point of using Clojure for live coding.

---

## 2. Clojure refresher

Just the parts you'll touch in Cantor. No exhaustive language tour.

### Reading Lisp again

`(f x y)` is a function call. Operators are functions: `(+ 1 2)`. Macros look the same: `(if pred then else)`. There are no statements, only expressions — everything returns a value.

Literals you'll see constantly:

```clojure
42           ; long
3.14         ; double
1/3          ; ratio — EXACT. Triplets in Cantor are this.
"hello"      ; string
:keyword     ; keyword — interned, ==-comparable, used as map keys
'sym         ; symbol — name of a thing, not the thing
nil          ; null/absent
true false   ; booleans
[1 2 3]      ; vector (indexed, O(log32 N) access)
'(1 2 3)     ; list (cons cell; mostly for code, not data)
{:a 1 :b 2}  ; map (hash-array-mapped-trie)
#{1 2 3}     ; set
```

Quoting (`'`) prevents evaluation. `'foo` is the symbol `foo`; `foo` would look up its value. Mostly used in macros and the REPL.

### def, defn, let

```clojure
(def tempo 120)                  ; module-level binding

(defn quarter-seconds [bpm]      ; function
  (/ 60.0 bpm))

(let [a 1                        ; local binding (block-scoped)
      b (+ a 1)]
  (* a b))                       ; → 2
```

`defn` is `def` + `fn`. The function body is a sequence of expressions; the last one's value is returned. No `return`.

### Anonymous functions

```clojure
(fn [x] (* x x))
#(* % %)        ; shorthand, % is the arg, %1 %2 for multiple
```

The `#()` reader macro is fine for one-liners; reach for `fn` once you need destructuring or multiple expressions.

### Sequences

`first`, `rest`, `cons`, `count`, `map`, `filter`, `reduce`, `take`, `drop` all work on any seqable (vectors, lists, lazy seqs, sets, map entries).

```clojure
(map inc [1 2 3])              ; (2 3 4)  — returns a lazy seq
(filter even? (range 10))      ; (0 2 4 6 8)
(reduce + 0 [1 2 3 4])         ; 10
(take 5 (iterate inc 0))       ; (0 1 2 3 4)
```

Lazy seqs matter: `(range)` is infinite; `(take 5 (range))` works. Don't `count` a lazy seq unless you know it terminates.

### Destructuring

The single feature you'll miss if you don't relearn it. Works in `let`, `fn` params, `defn`, `loop`.

```clojure
(let [[a b & rest] [1 2 3 4]]
  ;; a=1, b=2, rest=(3 4)
  )

(defn handle-event [{:keys [pitch velocity duration]
                     :or {duration 1}}]
  ;; pulls :pitch, :velocity, :duration from the map
  ;; defaults duration to 1
  ...)

(let [{:keys [start end]} arc] ...)   ; the common case in Cantor's stream code
```

You'll see `{:keys [...]}` everywhere in the codebase.

### Threading macros

Read-as-pipeline. `->` threads the result as the *first* argument; `->>` as the *last*.

```clojure
(-> {:a 1}
    (assoc :b 2)
    (assoc :c 3))
;; ⇒ (assoc (assoc {:a 1} :b 2) :c 3)
;; ⇒ {:a 1 :b 2 :c 3}

(->> [1 2 3 4]
     (map inc)
     (filter even?)
     (reduce +))
;; ⇒ (reduce + (filter even? (map inc [1 2 3 4])))
;; ⇒ 6
```

Convention: `->` for data updates (functions where the data is the first arg, like `assoc`/`update`/`conj`), `->>` for seq pipelines (functions where the seq is the last arg, like `map`/`filter`).

### State: atoms

```clojure
(def counter (atom 0))

@counter                    ; deref → 0
(swap! counter inc)         ; → 1
(swap! counter + 10)        ; → 11
(reset! counter 0)          ; → 0
```

`swap!` applies a pure function to the current value and atomically installs the result. It may retry, so the function must be side-effect-free. This is the only state primitive Cantor uses.

For `scheduler-state`:

```clojure
(swap! scheduler-state assoc :tempo 140)
(swap! scheduler-state update :pending conj evt)
```

### Namespaces

One namespace per file. The path mirrors the namespace: `src/cantor/core/stream.clj` → `cantor.core.stream`.

```clojure
(ns cantor.core.stream
  (:require [cantor.core.types :as t]
            [clojure.string :as str]))

(defn periodic [period value]
  ...)
```

In the REPL: `(in-ns 'cantor.core.stream)` switches to that namespace; `(use 'overtone.live)` pulls everything from `overtone.live` into the current ns (used for boot, not normal code).

### Records vs. maps

Default to plain maps. Records (`defrecord`) exist for performance + protocol dispatch, but for Cantor an "event" is `{:whole {...} :part {...} :value v}` — a map. Don't make types until you have a real reason.

### Macros

You will read a few (`defsynth`, `definst` from Overtone) but probably won't write any. If you find yourself writing a macro, ask whether a higher-order function would do the same job.

### Java interop

Used when the JVM is the only source of what you need:

```clojure
(System/nanoTime)                    ; static method
(.length "hello")                    ; instance method on String
(Math/PI)                            ; static field
(java.util.concurrent.LinkedBlockingQueue.)   ; constructor
```

Cantor needs `System/nanoTime` and possibly `Thread/sleep` in the scheduler loop. That's about it — Overtone wraps the rest.

### Errors and debugging

`(throw (ex-info "message" {:context :data}))` for raising. `(try ... (catch Exception e ...))` for catching. In the REPL, `*e` is the last exception; `(.printStackTrace *e)` shows the trace.

Calva's REPL prints stack traces inline. For atom inspection, just `@my-atom` in the REPL.

---

## 3. The Cantor-shaped REPL session

Typical session loop:

1. Open `src/cantor/live.clj`, jack in.
2. `Ctrl+Alt+C Enter` on `(use 'overtone.live)` — boots scsynth.
3. Open the namespace you're editing (e.g. `cantor.core.stream`); `Ctrl+Alt+C N` to switch the REPL into it.
4. Edit a form, `Ctrl+Alt+C Enter` to redefine.
5. In the REPL window, call into it: `(periodic 1 60)` and inspect the result.
6. Once it works in isolation, switch back to `cantor.live` (`Ctrl+Alt+C N` from that file) and `(play (periodic 1 60))`.
7. Hot-swap: edit `periodic`, eval, the running scheduler picks up the new behavior on the next tick.

When something goes wrong:

- Stuck note? `(stop)` should be in `cantor.live` for exactly this. Until it exists, `(overtone.live/stop)` cuts everything.
- REPL frozen on a runaway loop? `Ctrl+Alt+C Ctrl+C` interrupts the current eval. If that fails, restart the REPL — but losing the JVM means re-booting scsynth, so this hurts.
- "Var not found" after rename? `Ctrl+Alt+C Ctrl+L` reloads the whole file from disk.

---

## 4. Resources worth bookmarking

- **Clojure cheatsheet**: https://clojure.org/api/cheatsheet — every core function on one page.
- **ClojureDocs**: https://clojuredocs.org — same, but with community examples for each function.
- **Calva docs**: https://calva.io — keybindings, jack-in flags, paredit reference.
- **Overtone wiki**: https://github.com/overtone/overtone/wiki — synth examples and MIDI.
- **Brave Clojure** (free book): https://www.braveclojure.com — full refresher if you want longer-form.

Skip "Clojure for the Brave and True" Ch. 1–4 if you remember the syntax; jump to Ch. 5 (functional programming) and Ch. 10 (atoms/refs/agents).
