# CLAUDE.md

## Output preferences

**Deliverables are local files in this repo. Do not publish Artifacts.**

- Reports, write-ups and any visual deliverable go in this folder as `.html`
  files, committed alongside the code. Not hosted, not online, no links.
- Never call the Artifact tool for this project, including for "just a preview".
- If a deliverable needs to be shared, it is shared as a file or via the public
  GitHub repo, never as a hosted page.

## What this is

*The Road That Bends Back* — a terminal survival game written in
[Glyph](https://www.npmjs.com/package/@glyphlang/glyph), plus a field report on
writing it. Public repo: `github.com/chadetov/road-that-bends-back`.

```
src/
  types.glyph     core types and a fresh Game
  world.glyph     the road graph — 20 places, all the prose
  rules.glyph     energy, hunger, health, time, arrival
  economy.glyph   hunting, buying, working, resting
  puzzle.glyph    the three words and the door
  render.glyph    word-wrap, meters, colour
  save.glyph      the ledger that survives death
  main.glyph      the turn loop
docs/
  screenshot.svg  captured from a live pty run, not a mock-up
GLYPH-JOURNAL.md  the field report (markdown)
field-report.html the field report (styled, local)
```

## Commands

```sh
glyph check src            # type-check + run the 14 @example tests
glyph run                  # play it
glyph run -- --seed 7      # different dice
glyph run -- --forget      # burn the ledger
glyph fmt src              # format
glyph --update             # move the installed compiler to the newest release
```

Pin lives in `package.json` and is exact on purpose — a 0.1.x release can add a
diagnostic that rejects code which compiled before. Move it with `glyph upgrade`
or by hand, never widen it to a range.

**Do not run `glyph fix` on this repo while on 0.1.106.** It corrupts source:
pruning a name from an import list drops the newline terminating the import, so
the file stops parsing while `fix` exits 0 reporting success. There are currently
no `E0106` warnings here so it has nothing to prune, but don't invite it.

## Game design invariants

Break these and it stops being this game:

- **The world is a directed graph, not a tree.** Cycles are the point. Three hubs
  (milestone, ferry landing, nine stones) yield a word only on the *third* visit,
  and the door wants all three in a stated order. Looping back is the source of
  the answer, not a punishment.
- **Knowledge is the save file.** Death clears found words but the door accepts a
  correct *typed* answer regardless. Never gate the door on stored state.
- **Every loop stays escapable and branches stay two-way.** An early bug made the
  two halves of the map meet one-way, so no single life could reach all three
  cycles.
- Hunt odds scale off *current* energy — a tired hunter is a bad hunter. That is
  deliberate; it is what turns one bad watch into a bad week.
- The ferry fare has a free alternative (wading the mire), so an empty purse never
  strands the player.

Content is data: a new place is an entry in `world.glyph`'s `PLACES`; a fourth
cycle is one more entry in `puzzle.glyph`'s `FRAGMENTS`.

## Glyph facts that cost real time

Verified on 0.1.93–0.1.106. Re-check before relying on them; the language moves
fast.

- **Escapes:** only the braced form works — `"\u{1b}"` for ESC. The four-hex form
  and `\x` are both `E0001`.
- **No string literal inside `${...}`** — hoist it into a `let` first. This is the
  most frequent friction in text-heavy code.
- **Parameter fields are mutable in place.** `fn f(s: Stats) { mut s.energy = 0 }`
  works, as does `mut g.path.push(x)`. The "no `mut` field" gotcha in the docs
  means there is no `mut` *declaration* modifier. This is why the game threads one
  mutable `Game` by reference instead of rebuilding it every turn.
- **Sibling modules import by bare name:** `import world`, not `myapp/world`. To
  use both a namespace and bare constructors you need two lines: `import rules`
  plus `import rules { Walked, Refused }`.
- **A positional payload binds a name, not a record pattern.** `Wrapped(p) => p.name`,
  never `Wrapped({ name: n })` — the latter leaks `{ tag, value }` through a raw
  `TS2339`.
- **Two undocumented discovery tricks:** importing a bogus name makes `E0105` list
  a module's real exports; `glyph build` dumps stdlib source to
  `<out>/.glyph-runtime/std/*.ts` (0.1.106 prunes this to modules you already
  import).

## Field report workflow

`GLYPH-JOURNAL.md` and `field-report.html` are the same content in two formats.
When re-testing against a new Glyph release:

1. Re-run every finding rather than assuming it carried over. Several have moved
   independently, and one was fixed then shipped broken.
2. Verify every number before writing it down. This goes to people who can check
   it; two figures were overstated on first draft and one claim was simply wrong.
3. Quote diagnostics verbatim. It is what makes the report actionable.
4. Keep both formats in step — the HTML is not a stale copy.
