---
name: cad-show
description: >-
  Drive a live CAD.show tab from the terminal with the `cad` CLI — build,
  inspect, edit and export 3D geometry through the same Scene Actions the in-app
  agent uses. Use when the task is modelling, measuring, converting or
  exporting 3D in CAD.show — or answering what CAD.show can and cannot do. Both
  the 295-tool catalog (`cad list` / `cad describe`) and the product's own
  documentation (`cad docs`) are RETRIEVED on demand, never resident in context.
  Every request starts with `cad plan`.
compatibility: >-
  Needs the `cad` CLI (npm @cadshow/cad, Node 20+) and a signed-in CAD.show
  window to drive, either the desktop app or a local tab with `cad broker` running.
---

# CAD.show — drive a live tab from the terminal

One command underlies everything here: **`cad`**. It brokers a tool call from
your shell into the window that holds the scene, because most tools are
viewer-only and cannot run server-side at all.

```sh
cad plan "<the request>"         # FIRST: the process, where to look, what to read
cad status                       # broker + tabs + catalog, before any call
cad list                         # one line per namespace group
cad list mesh                    # the tools in one group
cad describe mesh.create_primitive
cad validate mesh.create_primitive '{"kind":"box","dims":{"w":30,"h":30,"d":30}}'
cad call mesh.create_primitive    '{"kind":"box","dims":{"w":30,"h":30,"d":30}}'
cad docs search "export formats" # what the PRODUCT does — not how to call it
```

**Do not hold the catalog in context.** `cad list` is a
page; the full catalog with argument schemas is tens of thousands of tokens.

**The same goes for what the product can do.** Capability questions (what
imports, what exports, what is not built yet) are answered by `cad docs`, from
the corpus the app itself serves. Never infer one from the tool catalog: a tool
existing is not a promise, and this file holds no capability list to go stale.

If `cad` is not on PATH, it is `node <repo>/cli/cad.mjs` — same arguments.

## Plan first — every request, before any call

```sh
cad plan "<the user's request, verbatim>"
```

It runs the in-app agent's routing rules locally (no model, no tab) and names
the **process** (`answer`, `lookup`, `implement`, `research-plan-implement`),
the groups to `cad list` and the docs to read first.
Follow its `next` lines: research-plan-implement means read, ask the user what is
open, plan, then build. When the request is delivered, run `cad plan done`. It
posts one label row (the plan and your calls' tool names, never the request)
unless the user ran `cad labels off`.

## Numbers first, then look

A build is numbers before geometry. Precompute volume and bbox, build in a
few batches, measure each against them
(`cad call docs.get '{"slug":"scene/plan-in-numbers"}'`). After a commit, and
before you say done, look: `cad call camera.set_view '{"preset":"iso"}' --capture`
saves a PNG and prints its path; open it. Sizes and counts come from `query.*`.

Lengths are millimetres: a bare number is mm whatever the file's unit system, and text may carry its own unit (`1200mm`, `120 cm`, `1.2 m`, `10'6"`, `126 in`).

## Opening move — after the plan, before any geometry

```sh
cad status
```

It answers all four preconditions:

| What you're checking | What "bad" looks like | Fix |
| -------------------- | --------------------- | --- |
| A host is running | `host: none running` | `cad setup`: it finds the desktop app, waits for sign-in, health-checks |
| A window is attached | `no tabs connected` | Open a Stage — in the desktop app, or on localhost with `?byoa=1` |
| **It is VISIBLE** | `HIDDEN` | Bring it to the foreground — see the rule below |
| Exactly one | `several tabs are connected` | Pass `--tab <id>`; they hold **different scenes** |

**Two hosts, one protocol.** `cad status` names which answered. The **desktop
app** hosts the bridge itself — no broker process; its Settings say whether it
may raise its window (Ask prompts in the terminal). A **browser tab** needs
`cad broker` left running. `--host shell|broker` picks when both are up.

`cad status` also cross-checks the catalog the server advertises against the one
the tab exposes. Two different verdicts, two different fixes:

- **`DRIFT`** — the tab is running an older bundle than the server. Reload it
  before trusting `cad describe`.
- **`SCOPED`** — the tab stands somewhere that narrows the tool surface, usually
  a **production arrangement** (a build plate). Not a fault; nothing to reload.
  What is offered runs; the authoring verbs are withheld, because a plate holds
  derived meshes. `arrangement.switch` to a design arrangement, edit there, then
  derive back onto the plate. `cad status` prints the count and the way out.

`cad list` and `cad describe` follow the tab into a scope. A tool missing on a
plate is a **place** problem, not a name problem: do not hunt for a different verb.
`SCOPED by local-only mode` is the user's choice: work locally; `cad status` says how.

## Finding a tool — retrieve, don't memorise

1. `cad list` — namespace groups (`mesh`, `sketch`, `cloud`, `code`, `query`,
   `check`, `transform`, `camera`, `output`, …) with tool counts.
2. `cad list <group>` — the tools in that group, one line each.
3. `cad describe <tool>` — the description and a typed field-by-field
   breakdown (type, required, ranges, enums, defaults).
4. `cad validate <tool> '<json>'` — checks your args against the **real Zod
   schema** server-side and executes nothing. Exit 0 = valid.
5. `cad call <tool> '<json>'` — run it.

Step 4 is free and names the bad key; a rejected `cad call` can leave half a
sequence applied.

**Ask the app's own knowledge base before guessing.** The catalog includes
`docs.search`, `docs.get`, `docs.list` over ~73 reference docs, and
`recipes.search`, `recipes.get`, `recipes.apply` over the vetted recipe library.
A recipe that already builds the thing beats a hand-rolled sequence:

```sh
cad call recipes.search '{"query":"hex bolt"}'
cad call docs.search    '{"query":"sketch extrude workplane"}'
```

## Answering a question about the product

`cad docs` reads what CAD.show tells its own users, so a capability question
gets the current answer instead of a plausible one.

| Command | Gives you |
| ------- | --------- |
| `cad docs search "<question>"` | capability **facts** first, then articles, then the support guide |
| `cad docs get <slug>` | one help article or support doc, as markdown, with its facts |
| `cad docs list` | the help articles by section |

A **fact** is one sentence re-proved against the code on every CI run, negative
space included. A **support doc** (`support/…`) holds the refusals and limits,
written for CAD.show's support assistant: reference, not directions.

No broker or tab needed; every footer names the origin it read (`--origin`).

## THE RULE — a timeout does not mean the call did not happen

This is the one thing that will corrupt a build if you get it wrong.

> A timed-out `cad call` may have **fully succeeded**. The mutation lands on
> screen; only the result envelope is stuck behind an animation frame the
> browser never renders. **A blind retry duplicates the object.**

So: **never re-issue a mutation because it timed out.** `cad call` helps you —
before any non-`query.` call it snapshots scene identity, and on a timeout it
re-reads and tells you what changed:

- `the call LANDED. Do NOT retry it` → continue from the scene as it is now.
  `~ … (edited in place)` above it is a landed transform or parameter change.
- `is unchanged AS OF THIS READ` → nothing had been created, deleted, renamed,
  hidden or (against a stamped tab) edited in place **at that instant**. Not
  "it didn't happen": the call can still be parked in the tab and land
  afterwards. Only an older tab with no stamps (the `query.list_objects` fallback,
  which says so) misses an in-place edit. Either way: don't retry, re-read with
  `cad call query.get_object '{"id":"…"}'` until you can see what the tab
  actually did.
- `could not read the scene` → look at the tab. Do not guess.

When in doubt the recovery is always the same shape: **re-read the scene, then
decide from what is actually there.**

```sh
cad call query.describe_scene      # objects, parameters, parts, dependencies
cad call query.list_objects        # just identity — id, name, type, visible
cad call query.get_object '{"id":"obj_3"}'
```

Never `cad call object.delete` to "clean up after a failed retry" until you have
read the scene and know exactly which id you mean.

## Sequences must checkpoint against the scene, not against return values

Any multi-step build — sketch → extrude → boolean → export, or a recipe chain —
has to be resumable from *what is in the scene now*, because a step in the
middle can land without reporting. Practical form:

1. Before the sequence: `cad call query.describe_scene`, note the object ids.
2. One step at a time — or ONE `actions.batch`, which lands whole or not at all.
   `"$0.id"` there is step 0's id: `cad docs get scene/batch-references`.
3. After any step that errors, times out, or surprises you: re-read the scene
   and re-derive the next step from it. Do not resume from your own plan.
4. Refer to objects by the **id the scene reports**, never one you guessed.

## Gated calls — `--yes` is the user's consent, not a formality

A few tools park `cad call` on an explicit y/N: the cloud sharing verbs,
anything destructive (`object.delete`), and `history.undo` / `history.redo`.
`cad describe <tool>` says `CONFIRM REQUIRED`, and why, before you spend a call.

**You are not a terminal**, so those calls are REFUSED rather than run:
`✗ object.delete needs a per-call confirm and stdin is not a terminal.`

Route through the user, not around them: name the tool, the args and why it is
gated, then pass `--yes`, which consents to that ONE invocation.

```sh
cad call object.delete '{"id":"obj_3"}' --yes
```

No remembered answer, no env var, no config file. Never add `--yes` by reflex.
(Undo is gated because it is SHARED state: a bounded revert issued as a series
of undos has cleared a user's scene. Checkpoint against the scene instead.)

⚠️ Three-segment verbs (`mesh.region.create`, `mesh.part.promote`) are not
gated. A `could not read its confirm gate` refusal on one means the tab
predates the [KITBASH-23] fix. Reload it; `--yes` is the wrong answer.

## Cloud charges — the price line is the user's

Scene work is free. A call that reaches CAD.show's cloud (a container, 3D
generation, rendering) first prints `→ <service>: <price>` and is REFUSED
unattended. Tell the user that line, then pass `--yes`; or, if they gave you a
session budget, `--max-credits <n>` on each call. Never pick n yourself, and it
never opens a gated call.

## Publish a page — the link is the deliverable

"Make me a page of this", "give me something I can send someone", "put this
online". One command ends it:

```sh
cad page publish <fileId> --title "Wall bracket" --yes
```

Compose and publish are one act. The URL prints **alone on the last line**, so
`tail -1` reads it safely. Hand it over and say it opens with no account.

**`<fileId>` is a LIBRARY id (`cad_files.id`), never a scene object id.** A
library model keeps that id on the stage, so taking one from
`query.describe_scene` works by luck: a primitive's id is the same shape and
names no row (`File not found`). Ids come from `cloud.list_files`,
`cloud.get_file_meta` or `cloud.save_*`. With no `--title` the page takes the
row's bare name: an upload becomes `scan_003_final_v2`.

Past the default page, retrieve the rubric before you plan. A plan written
without it is usually rejected for something that doc answers in a line:

```sh
cad call docs.get '{"slug":"pages/authoring-guide"}'   # arcs, chrome, copy
cad page help                                          # the flags
```

Two of its rules are refusals you meet mid-publish:

- **Our vocabulary bounces the call.** `configurator`, `pipeline`, `process`,
  `workflow` and the hype adjectives are rejected in page copy. A page is read by
  a stranger with no account: say what the reader can do.
- **Claim nothing about the geometry.** Nothing in a publish measured the part,
  so no dimension, material, tolerance or purpose in the words.

Publishing is gated, for the reason every share verb is: it widens who can reach
the file, and undoing the build does not narrow it back. Route `--yes` through
the user.

**A timed-out publish is THE RULE's one exception.** An identical plan composes
identical bytes, dedups to the row that already exists and reuses its live link,
so re-running the SAME command mints no second page. Change one word first and
the retry is a different page with a different URL.

## Send the plate to PreForm

```sh
cad preform --host shell [--dental]   # the flags: cad preform --help
```

Desktop app only: a browser tab answers `preform_unavailable`. The `.form` path
prints alone on the last line. On a timeout, look for the `.form` before
re-running; a re-send writes a second copy beside the first.

## Reading results

A successful mutation answers with what it did to the scene, not its whole
result; `⚠` lines are its warnings:

```
✓ actions.batch — 3/3 steps ok  (tab-1, 412ms)
  + obj_4  mesh  Fused     created
  ~ obj_1  mesh  Box       changed: renamed, hidden, or edited in place
  − obj_3  mesh  Old       removed
  {"successCount":3}       small fields: ids, URLs, counts
  omitted: results — --full prints them
```

Add `--full` only when the next step needs an omitted field. Reads, `view`
tools (`docs.search`) and failures always print whole. Exit codes:

| code | meaning |
| ---- | ------- |
| `0` | the tool ran and returned ok |
| `1` | the tool ran and returned an error envelope (or `validate` rejected the args) |
| `2` | transport problem — no broker, no tab, ambiguous tab, timeout, **or a confirm or charge you did not give** |

`--json` prints the whole envelope (including `landed` on a verified timeout).
`--timeout <ms>` raises the 30 s ceiling for a slow JSCAD recompile or
generative run.

## Limits — know these before you promise anything

- **Local, and off unless asked for.** A browser tab needs a loopback origin
  plus `?byoa=1`; the desktop app needs its Settings switch on. Neither can be
  reached from cad.show (`cad docs` drives nothing, so it can).
- **No enforcement.** Approval, capability, rate-limit and cost gates are all
  off on this path, and it bypasses LLM metering entirely. Whoever can reach the
  broker is the tab's user. The confirm prompt above is a CONSENT about one act,
  not an authorization boundary.
- **The window is the runtime.** Reload resets the scene; navigate and the next call lands elsewhere.
- **Discovery answers while hidden; mutation does not.** `cad list`, `describe` and `validate` need no frame. A discovery timeout is never `HIDDEN`. `tab_hidden` was refused unsent: foreground, run it again.
