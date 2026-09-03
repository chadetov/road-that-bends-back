# Building a terminal game in Glyph — a field report

**What was built:** *The Road That Bends Back*, an interactive terminal
survival game. 1,430 lines of Glyph across 8 modules, emitting 2,179 lines of
TypeScript. Stateful turn loop, 20-node directed graph with cycles, resource
simulation, seeded RNG, JSON persistence, word-wrapped ANSI UI, 14 inline
`@example` tests.

**Toolchain:** written against `glyph 0.1.93` (npm `@glyphlang/glyph`), re-tested
on 0.1.95 and 0.1.106. node v26.3.0, tsc 6.0.3, tsx 4.22.4, macOS (darwin 25.2.0).

**Re-tested on `glyph 0.1.106`** (2026-09-04), eleven releases on. The game
builds and plays unchanged and the pin has been moved. Three findings are
genuinely fixed, one was implemented and shipped broken, and one improvement
quietly made a still-open finding harder to work around.

| # | Finding | 0.1.95 | 0.1.106 |
|---|---|---|---|
| 1 | `\u{..}` missing from bootstrap; E0001 does not name it | unchanged | **FIXED** |
| 2 | `std/random` has no documented signatures | unchanged | **FIXED** |
| 2b | Discovery routes undocumented | unchanged | unchanged, and *harder* |
| 2c | `Rng.bool` "default 0.5" contradicts its type | unchanged | **FIXED** in code, doc residual |
| 3 | No string literal inside `${...}` | unchanged | unchanged |
| 4 | The two `mut` doc statements contradict | unchanged | unchanged |
| 5 | Sibling-module imports under-specified | unchanged | unchanged |
| 6 | `glyph fix` cannot prune one name from an import | unchanged | **implemented, but it corrupts the file** |
| 7 | Statement-position `match` ceremony | unchanged | unchanged (agreed no action) |
| — | Positional-payload pattern leaks `{ tag, value }` | unchanged | unchanged |
| G142 | Generic + imported missing arm was silent | FIXED | still fixed |

### Fixed, and fixed well

**Finding 1, both asks.** The bootstrap now documents the escape at lines
315-317 — *"`\u{HEX}` for an arbitrary Unicode code point by its hex value,
`\u{1b}` for ESC among them"* — and E0001 now says what to write instead:

```
Help: Check for an unterminated string, an invalid escape
      (only \n \t \r \" \\ \u{HEX} are allowed), or a stray character.
```

That is the exact change this report argued for, and it is the one that would
have saved me the eleven-line workaround. The general principle held: the
diagnostic is now closed under *"what do I write instead"*.

**Finding 2.** `std/random` has a real signature block — `type Rng`,
`seeded(seed)`, and the four methods. No probing required any more.

**Finding 2c, in the code.** `bool` is now `(probability?: number)` with a real
`= 0.5` default, and `r.bool()` compiles and runs. Verified, not read.
**Residual:** the new bootstrap block writes `rng.bool(probability: number)`
with no `?`, so a reader still cannot tell the argument is optional. The
contradiction moved rather than closing: the code is now right and the
signature under-reports it.

### Shipped broken

**Finding 6 was implemented, and the implementation corrupts source.** The new
prune-names path drops the newline that terminates the import:

```
before   import helper { one, two, three }
         fn main(argv: Array<string>) -> number {

after    import helper { one }fn main(argv: Array<string>) -> number {

glyph fix: removed 2 unused import(s) across 1 file(s).      exit 0
glyph check: [E0002] expected newline after import, found Fn  exit 1
```

A project that compiled before `glyph fix` does not compile after it, and `fix`
exits 0 claiming success. `glyph fmt` cannot repair it (`1 failed`) and also
exits 0. Characterised across four shapes:

| case | shape | result |
|---|---|---|
| 1 | partial prune, declaration on the next line | **breaks** |
| 2 | partial prune, blank line after the import | ok |
| 3 | whole unused import removed (the old path) | ok |
| 4 | partial prune, another import on the next line | **breaks** |

It is the new path only, and it survives only when a blank line happens to
absorb the lost newline — which is why it would pass any test written against
the old behaviour. Both breaking shapes are idiomatic Glyph.

### An improvement that cost something

0.1.106 prunes the emitted runtime to what a project actually imports: this
codebase now emits 13 `std` modules instead of all 36. Good for output size,
and undocumented — I found it by counting.

But it weakens finding 2b. Reading `.glyph-runtime/std/*.ts` was one of the two
ways to discover an undocumented module's signatures, and it now only shows the
modules you already import. To read `std/websocket`'s API you must first import
`std/websocket`. The workaround for a still-open documentation gap got narrower
because of an unrelated optimisation — which is an argument for closing 2b
properly rather than leaving people to find their own routes in.

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

> **Resolved in 0.1.106.** The bootstrap now documents `\u{HEX}` and names
> `\u{1b}` for ESC, and E0001's help text lists the allowed escapes. Both asks
> below are done. The finding is kept for the record and for the generalisation
> at the end of it, which is the part that outlived the bug.

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

> **Mostly resolved in 0.1.106.** `std/random` now has a full signature block,
> and `Rng.bool` has a real default so `r.bool()` compiles. Two things remain:
> the new block writes `rng.bool(probability: number)` without the `?`, so the
> default is still invisible to a reader; and the two discovery routes below are
> still undocumented — one of them now less useful, since the emitted runtime is
> pruned to the modules a project already imports.

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

**Severity: was low. Now high — the fix shipped broken.**

> **Implemented in 0.1.106, and it corrupts source.** `glyph fix` now prunes
> individual names, but drops the newline terminating the import, so a project
> that compiled before the fix does not compile after it — while `fix` exits 0
> reporting success. Full characterisation in the 0.1.106 status section above.
> The ask below stands, plus a second one: `fix` should re-parse every file it
> rewrites before claiming success.

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
| 1 | ~~Carry `\u{..}` into the bootstrap; make E0001 name it~~ | **Done in 0.1.106** |
| 2 | Signatures for the "shipped, not detailed" modules | **`std/random` done in 0.1.106**; others unverified |
| 2b | Document E0105-lists-exports and `.glyph-runtime/` | Two great tools nobody knows about |
| 2c | `Rng.bool` default | **Fixed in code**; bootstrap signature still omits the `?` |
| 3 | Relax nested string literals in `${...}` | Main friction in text-heavy code |
| 4 | Reword the `mut` gotcha; state that parameter fields are mutable | Prevents a wrong architecture |
| 5 | One line on bare-name sibling imports | First wall on file #2 |
| 6 | `glyph fix` pruning single names | **Shipped in 0.1.106 but corrupts the file** — and it should re-parse what it rewrites |
| 7 | (optional) statement-position guard sugar | Removes 10 `false => {},` arms |

**Overall:** eight compile failures in 1,400 lines of a language I had never
seen, every one self-explaining, and the finished program is genuinely more
refactor-safe than the TypeScript I would have written instead. The gaps above
are all narrow. The one I would fix first is the ESC byte — everything else has
a workaround that costs a line, and that one costs a category.
