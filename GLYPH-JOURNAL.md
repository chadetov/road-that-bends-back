# Building a terminal game in Glyph — a field report

**What was built:** *The Road That Bends Back*, an interactive terminal
survival game. 1,430 lines of Glyph across 8 modules, emitting 2,179 lines of
TypeScript. Stateful turn loop, 20-node directed graph with cycles, resource
simulation, seeded RNG, JSON persistence, word-wrapped ANSI UI, 14 inline
`@example` tests.

**Toolchain:** written against `glyph 0.1.93` (npm `@glyphlang/glyph`), node
v26.3.0, tsc 6.0.3, tsx 4.22.4, macOS (darwin 25.2.0).

**Re-tested on `glyph 0.1.95`** (2026-08-30). The game builds and plays
unchanged; the project pin has been moved. Findings status below.

| # | Finding | 0.1.95 |
|---|---|---|
| 1 | `\u{..}` missing from the bootstrap; E0001 does not name it | unchanged |
| 2 | `std/random` has no documented signatures | unchanged |
| 2b | Discovery routes (E0105 exports, `.glyph-runtime`) undocumented | unchanged |
| 2c | `Rng.bool` "default 0.5" comment contradicts its type | unchanged |
| 3 | No string literal inside `${...}` | unchanged (known standing limitation) |
| 4 | The two `mut` doc statements still contradict | unchanged |
| 5 | Sibling-module imports still under-specified | unchanged |
| 6 | `glyph fix` still cannot prune one name from an import | unchanged |
| 7 | Statement-position `match` ceremony | unchanged (agreed no action) |
| — | Positional-payload record pattern leaks `{ tag, value }` | unchanged |
| **G142** | **Generic + imported missing arm was silent** | **FIXED** |

**The one real change is G142, and it is a good one.** On 0.1.93/0.1.94 a `match`
missing a variant compiled clean and threw at run time when — and only when — the
union was both generic and imported. On 0.1.95 all four cells of that matrix
report `E0200` at compile time:

```
non-generic + same-module   [E0200] missing variants `SLeaf`
generic     + same-module   [E0200] missing variants `Leaf`
non-generic + imported      [E0200] missing variants `SLeaf`
generic     + IMPORTED      [E0200] missing variants `Leaf`     <- was silent
```

That closes the only hole I found in the verifiability claim, which is the claim
the whole language is sold on. It is the right thing to have fixed first.

**Everything else is exactly as reported.** The bootstrap grew by one line
between 0.1.93 and 0.1.95 — a new `glyph --update` command — and `u{` still
appears zero times in it. So a first-time reader writing this game today would
walk into all six documentation-shaped walls in the same order, including
building the same eleven-line workaround for the escape that already works. The
compiler got more correct; the thing that misled me did not change.

**Author's position:** I came to Glyph with no prior knowledge of it — it was
not in my training data. Everything below was learned in one session from
`glyph llms`, the diagnostics, and probing. That makes this a reasonably clean
test of the "zero to correct in one read" claim. **Headline: the claim mostly
holds.** I wrote 1,430 lines with **four compile failures in the game code**
(plus three more in deliberate probes), every one of which the compiler
explained precisely enough to fix on the next attempt.

**One finding was wrong and has been corrected in place.** I originally reported
that Glyph could not express an ESC byte; it can, via `\u{1b}`. The reframed
finding #1 is about why I concluded otherwise, which turned out to be the more
useful result.

The findings are ordered by how much time they cost.

---

## 1. The working escape spelling is missing from the agent bootstrap, and E0001 does not name it

**Severity: high. Reclassified after review — see the correction note.**

> **Correction.** My first draft of this finding claimed Glyph could not express an
> ESC byte at all, and recommended adding escapes to the language. That was wrong,
> and the Glyph maintainers caught it. **`\u{1b}` works, and worked on 0.1.93.**
> I have verified it myself and deleted my workaround. I am leaving the finding in
> — reframed — because *how* I got it wrong is the actual result here.

Three spellings, one of which works, on the version I used:

```
\u{1b}    compiles; tsc --strict passes; emits a real 0x1B
\u001b    [E0001] parse: lex error: invalid escape sequence `\u` at offset 85
\x1b      [E0001] parse: lex error: invalid escape sequence `\x` at offset 85
```

Verified at the byte level on 0.1.93:

```
$ glyph run | od -c
0000000  033   [   3   1   m   R   E   D 033   [   0   m       d   o   n
```

**How a competent first-time user still gets this wrong.** I tried the two
spellings I knew from other languages, got a lexer error that said the escape was
*invalid* rather than *misspelled*, and concluded the feature did not exist. Then
I built an eleven-line workaround around that conclusion:

```glyph
// what I shipped, believing I had to
fn esc_byte() -> string {
  return match bytes.from_hex("1b") {
    Ok(b) => match bytes.to_text(b) { Ok(s) => s, Err(_) => "", },
    Err(_) => "",
  }
}
const ESC: string = esc_byte()

// what I should have written
const ESC: string = "\u{1b}"
```

That is a double-`Result` unwrap, an extra `std/bytes` import and a module-level
initialiser, all to reimplement a feature that already existed. It is now deleted.

**The evidence for why this happened, which I checked myself:**

- `\u{HEX}` **is** documented — D12 of `docs/language/spec.md` names it among the
  decoded escapes. But `u{` appears **0 times** in the 1,111-line `glyph llms`
  bootstrap, and zero times in all three of its copies (`llms.txt`, `AGENTS.md`,
  `web/llms.txt`) — the file the docs explicitly say is the one to read if you
  read only one, and the file an agent is pointed at first. Present where a human
  browsing the spec would find it; absent where a machine is told to look. That is
  a worse failure than "undocumented" and a more specific one to fix.
- The only escape-related lines in that whole document are the raw-string
  paragraph and the E0001 catalogue row, whose fix column reads *"Fix the
  string/escape/character"* — naming neither the supported spelling nor the fact
  that a supported spelling exists.
- The diagnostic itself says `invalid escape sequence \u`. To a reader who just
  wrote `\u001b`, "invalid" reads as *unsupported*, not *wrong syntax*.

**This is the cheapest fix in this report and I would still do it first.** Not
because ANSI output is blocked — it is not — but because the failure mode is a
first-time user confidently building a workaround for a solved problem, and then
writing it up as a language gap. A single line in the bootstrap and a `did you
mean \u{1b}?` on E0001 would have saved all of it.

**Asks:**
1. Carry `\u{..}` from `spec.md` D12 into the bootstrap's string-literal section
   (and its `AGENTS.md` / `web/llms.txt` copies).
2. Make E0001 name the supported spelling when it rejects `\uXXXX` or `\xXX` —
   the same `did you mean` treatment E0220 already gives variant typos.
3. Decide whether `\x1b` should also be accepted, or stay deliberately rejected.

**Generalisation worth more than the specific bug:** a diagnostic that says a
construct is invalid, without saying what to write instead, teaches an agent that
the capability is absent. E0105 already does the opposite — it lists the real
exports — and that listing is what let me discover `std/random` in finding 2. The
two diagnostics sit at opposite ends of the same design question.

## 2. `std/random` has no documented signatures

**Severity: high for anything simulation-shaped.**

`glyph llms` lists `std/random` under *"Not detailed below, but shipped and
importable"*, alongside 11 other modules. For a game, the RNG is not a
footnote — it is core, and I could not call it.

Two undocumented discovery routes got me there, and **both are genuinely good
and neither is written down**:

**(a) Ask for a name that doesn't exist and read the error.** This is a
brilliant affordance:

```
[E0105] Error: import: `nonexistent_probe` is not exported by `std/random`
        (exports: Rng, seeded)
```

**(b) Read the emitted runtime.** `glyph build` writes the full stdlib source
to `<out>/.glyph-runtime/std/*.ts`, so `random.ts` gave me the exact type:

```ts
export type Rng = {
  next: () => number;
  int: (lo: number, hi: number) => number;
  bool: (probability: number) => boolean;
  pick: <T>(items: ReadonlyArray<T>) => T;
};
```

**Sub-finding — a doc/impl mismatch inside that file.** The comment says
`bool` takes *"a boolean, true with the given probability (**default 0.5**)"*,
but the signature is `(probability: number) => boolean` with no default and no
optional marker. Calling `r.bool()` will not compile. One of the two is wrong.

**Asks:**
1. Give the twelve "shipped but not detailed" modules a signature block each,
   even a terse one. `std/random` is four lines.
2. Document the E0105-lists-exports trick and the `.glyph-runtime/` escape
   hatch in the bootstrap — they are the two best discovery tools in the
   toolchain and I found both by accident.
3. Fix the `bool` default comment.

---

## 3. No string literal inside `${...}` — highest-frequency friction

**Severity: medium. Correct behaviour, real cost, excellent error message.**

This is documented, and the diagnostic is close to perfect — it names the fix
inline:

```
[E0002] Error: parse: expected matching `}` for `${...}` (a nested string
        literal inside `${...}` is not supported in v1 — hoist it into a `let`)
```

But in a **text-heavy** program it is the restriction I hit most. I tripped it
twice in real code, and the natural shape for formatted output is exactly the
banned one:

```glyph
// what you want to write
io.println(" ${meter("health", g.health, false)}   ${meter("energy", g.energy, false)}")

// what you must write
let health_bar = meter("health", g.health, false)
let energy_bar = meter("energy", g.energy, false)
io.println(" ${health_bar}   ${energy_bar}")
```

The codebase carries **nine `let` bindings that exist only to escape this
rule** — four in `render.header`, five across `main.ending` and `main.notes`,
every one of them a call with a string-literal argument that would otherwise
have been written inline. Each is individually fine.
Collectively they are the main source of noise in the rendering code, which in
a terminal game is a third of the program.

I understand this is a v1 lexer simplification, not a design position. Flagging
it because "programs that print a lot of formatted text" is a large slice of
what people write CLIs for, and this is where they will feel Glyph.

---

## 4. Two documentation statements about `mut` contradict each other

**Severity: medium — it cost a probe and could easily cost an architecture.**

The syntax section says field assignment is legal:

> `mut` is legal in exactly four forms — `mut x = e`, **`mut x.field = e`**,
> `mut x[k] = e`, `mut x.method(args)`.

The gotchas section says the opposite:

> **`mut` is narrow.** It only enables reassignment and mutating method calls;
> there is no `mut` parameter, **field**, or other position.

Read together, the second sentence reads as *"you cannot mutate through a
parameter"* — which would force every state transition in a game to rebuild and
return a whole `Game` record. That is a materially worse architecture, and I
was about to write it.

I probed instead, and **in-place mutation of a parameter's field works**:

```glyph
fn drain(s: Stats, cost: number) -> void {
  mut s.energy = s.energy - cost
}
// after drain(s, 30): s.energy == 70
```

The entire game now threads one mutable `Game` by reference on the strength of
that probe. `mut g.path.push(id)` (a mutating method on a *field* of a
parameter) works too.

**Ask:** the gotcha means *"`mut` is not a declaration modifier — there is no
`mut` parameter or `mut` field **declaration**"*. Say that, and add one line
stating that a parameter's fields **are** mutable in place. This is a
load-bearing fact about how you structure a Glyph program.

---

## 5. Sibling-module imports are under-specified

**Severity: low, but it is the first thing anyone writing a second file hits.**

E0101's fix text reads:

> Use an absolute module path (`std/io`, `myapp/x`)

The `myapp/x` example implies a project module needs a package-name prefix. It
does not — a sibling in the same `src/` is imported by its **bare module name**:

```glyph
import world          // src/world.glyph, module world
import rules
```

I had to probe to establish this. One line in the bootstrap's "canonical
program shape" section would settle it.

**Related, and also worth one line:** to use both a namespace and bare
constructors from the same project module you need two import lines —

```glyph
import rules
import rules { Walked, Refused }
```

The bootstrap shows this pattern for `std/time` but presents it as a quirk of
that module rather than the general rule it is.

---

## 6. `glyph fix` stops short of partially-unused imports

**Severity: low.**

E0106 fires per-name, but `--help` says `fix` removes *"imports whose **every**
name is unused"*. So this:

```glyph
import types { Game, Place, PlaceKind, Wild, Market, Waystop, Shrine, Gate, Ending }
```

produced seven separate E0106 warnings and `glyph fix` could not touch any of
them, because `Game` and `Place` were still live. Hand-editing a nine-name
import list is exactly the mechanical edit an autofix should own. I hit this
twice (`economy.glyph`, `render.glyph`).

**Ask:** let `fix` prune individual names from a partially-used import list.

---

## 7. The `match`-only design has one repetitive shape

**Severity: low — a design cost, honestly incurred, reported for completeness.**

No `if` is a defensible choice and I do not want it reversed. But the
statement-position guard is genuinely noisy, and it recurs:

```glyph
match string.len(extra) > 0 {
  true => {
    render.blank()
    render.say(extra)
  },
  false => {},
}
```

That `false => {},` is pure ceremony. There are **10 such arms** in this
codebase, concentrated in the rendering and turn-loop code where the logic is
"sometimes print an extra thing".

Everywhere else `match` was a pleasure — including nested in a `match` arm,
which compiles cleanly and reads well:

```glyph
fn hunger_toll(hunger: number) -> number {
  return match hunger >= 100 {
    true => 12,
    false => match hunger >= 85 {
      true => 6,
      false => 0,
    },
  }
}
```

**Ask:** nothing urgent. If a sugar is ever considered, a statement-position
`when cond { ... }` with no else-arm requirement would absorb all 10 cases
without reintroducing `if`/`else` as an expression form.

---

## What worked well — and specifically why

These are not padding. Three of them changed how I worked.

**Exhaustiveness checking as a refactoring tool — the standout.** Halfway
through I realised place kinds were wrong: a bare gate in a field was offering
a market and paid labour. I added one variant, `Empty`, to `PlaceKind`. The
compiler responded with exactly four errors:

```
[E0200] non-exhaustive match on `PlaceKind`: missing variants `Empty`   ×4
```

— one for each site that needed a *decision*, across three modules, with no
grep and no guessing. Then it went silent the moment all four were answered.
This is the single best experience I had with the language. It is the argument
for the whole design, and it is worth putting front and centre in the pitch.

**Diagnostics.** Uniformly excellent: stable code, exact span, a `Help:` line
that states the fix, and a docs link. Across every failure in this build I
never once had to guess what the compiler meant. The E0105 "exports: Rng,
seeded" listing is a small masterpiece.

**`@example` on the declaration.** Tests that run on every `glyph check` with
zero configuration are the right default. They immediately caught an error —
mine, in a test I had just written:

```
glyph check: example failed: render example #0:
             (array.len(wrap("one two three four", 9))) != (2)
```

The answer was 3. The wrapper was right and I was wrong, and I knew within
seconds.

**`glyph fmt` is safe on prose.** This game is mostly text: 20 places of
multi-line literals, some with deliberate leading indentation (a word carved
into a door). `fmt` reformatted all 8 modules and changed not one character
inside a string. For a content-heavy program that trust matters a lot.

**`glyph llms` itself.** One offline document that took me from "I have never
heard of this language" to a compiling non-trivial program. The gotchas section
earned its keep — `bool` not `boolean`, trailing commas in every arm, no object
shorthand. I would put the two discovery tricks from finding #2 into it and
call it close to complete.

---

## Summary of asks

| # | Ask | Impact |
|---|---|---|
| 1 | Carry `\u{..}` from the spec into the bootstrap; make E0001 name it | Stops users building workarounds for solved problems |
| 2 | Signatures for the 12 "shipped, not detailed" modules | Removes guesswork from the stdlib |
| 2b | Document E0105-lists-exports and `.glyph-runtime/` | Two great tools nobody knows about |
| 2c | Fix `Rng.bool`'s "default 0.5" comment | Doc contradicts the type |
| 3 | Relax nested string literals in `${...}` | Main friction in text-heavy code |
| 4 | Reword the `mut` gotcha; state that parameter fields are mutable | Prevents a wrong architecture |
| 5 | One line on bare-name sibling imports | First wall on file #2 |
| 6 | `glyph fix` should prune single unused names | Mechanical edit, currently manual |
| 7 | (optional) statement-position guard sugar | Removes 10 `false => {},` arms |

**Overall:** eight compile failures in 1,400 lines of a language I had never
seen, every one self-explaining, and the finished program is genuinely more
refactor-safe than the TypeScript I would have written instead. The gaps above
are all narrow. The one I would fix first is the ESC byte — everything else has
a workaround that costs a line, and that one costs a category.
