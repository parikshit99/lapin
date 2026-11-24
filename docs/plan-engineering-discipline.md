# Engineering discipline: what to take from a spec-first BIM kernel

A plan for the sessions that improve Lapin. It compares Lapin with Octopus, a separate BIM application in Rust built spec-first. Octopus is far behind Lapin in features but stricter in how it is built. Its owner has agreed that its ideas, rules and tooling may be used here under Lapin's MIT licence.

Every item below names the Octopus practice, what Lapin already has (with evidence from 24.11.2026), the gap, the change, and how to tell it is done. The gaps are taken mostly from Lapin's own audits (`docs/audit-2026-11-15.md`, `docs/audit-2026-11-22.md`). Their three recurring causes are exactly the problems Octopus's rules exist to prevent.

**How to use this file.**
- Take one item per branch and record it as a decision in `docs/decisions-pending.md`, with a paragraph in `docs/history.md` when it is done.
- Do the waves in order. Items inside a wave are independent.
- Each check here starts as a **ratchet**: a baseline file lists today's offenders, and the check fails only on new ones or when an offender gets worse. Nothing has to be fixed before a check lands, and the count can only go down.
- Do not change behaviour users see as a side effect. Where an item changes it, say so in its decision.

## What Lapin already has (do not redo)

Some of what looks like an Octopus strength is already in Lapin, sometimes done better:

| Octopus practice | Lapin today |
|---|---|
| Rebuild by recorded reads (Octopus B5) | `Session.Built.Reads` and the reverse index `_readers` in `Session.cs`; near reads by cell (`NearReads`, `PlaceCells`); earlier builds reused on undo (`Session.Builds.cs`) |
| Stable sub-element names (A12) | `ElementFace.Role` in `Lapin.Platform/ElementFaces.cs`: worked out from attributes, never from the B-rep |
| Import fidelity gate, 1 mm / 0.5 % (A11) | `RecognitionFidelity` |
| An edition is a list of modules; absent modules read-only (B10) | `products/*.json`, `scripts/product.mjs`, `AbsentModuleTests.cs` |
| Content-addressed package with a journal (C1) | `.lapin` format 3 (`docs/format-lapin.md`) |
| One transaction per command (B1, B2) | `IEdit` in `Lapin.Model/History.cs`, `edit.batch` |
| Schema migrations (C2) | `IProjectUpgrade` |
| Warnings denied, null safety | `engine/Directory.Build.props`: `TreatWarningsAsErrors`, `Nullable` |
| Licence allow-list | `pnpm check:licenses` |
| Module layering | Every module's `ProjectReference`s are within its `module.json` `requires` (checked by hand on 24.11.2026). It is not yet enforced: see 1.3 |

## Wave 1: cheap guards (days, no behaviour change)

### 1.1 One command that runs every check

**Octopus:** `tools/check_all.sh` runs format, lints, tests, every repository rule, docs and licences, and must print `all checks passed` before any push. Agents run it before each commit.

**Lapin:** the checks exist but are scattered: `pnpm typecheck`, `pnpm lint`, `pnpm test`, `check:licenses`, `scripts/check-commands.mjs`, `command-catalogue.mjs --check`, `check:web`. A session has to remember which to run.

**Change:** add `pnpm check` (`scripts/check-all.mjs`), which runs them all plus the new checks of this plan. It prints one line per check and ends with `all checks passed` or the list of failures. It should support `--fast` (no engine tests) for UI-only work. Add one line to `CLAUDE.md`: run `pnpm check` before every push.

**Done when:** `pnpm check` on a clean `main` prints `all checks passed`, and each sub-check fails it when broken on purpose.

### 1.2 A file size cap

**Octopus:** no source file over 800 lines (its decision H14). Small files are what make an AI session effective: it can read the whole file it changes.

**Lapin:** 42 engine `.cs` files and 18 UI `.ts`/`.tsx` files are over 800 lines. `Session.cs` is 1,497.

**Change:** add `scripts/check-file-size.mjs`, with a baseline in `scripts/file-size-baseline.json` (`path → lines today`). It fails when a file not in the baseline passes 800 lines, or a file in it grows. When a baselined file drops under 800, remove it from the baseline. Split files the way `Session.cs` already started (`Session.Builds.cs`, `Session.Step.cs`: `partial` classes by concern).

**Done when:** the check is in `pnpm check` and CI, and the baseline only shrinks.

### 1.3 Enforce module layering

**Octopus:** a crate depends only on its own layer or below, checked on every run. The kernel never knows what a wall is (its principle 3: modules use the same registry as extensions).

**Lapin:** the layering holds today but nothing keeps it.

**Change:** add `scripts/check-layering.mjs`. It fails when:
- a module's `.csproj` references another module that its `module.json` does not `require`, directly or transitively;
- a `using Lapin.<Module>` or a fully qualified `Lapin.<Module>.` in a module's `.cs` files names a module it does not require;
- `Lapin.Model`, `Lapin.Platform`, `Lapin.Package`, `Lapin.Protocol`, `Lapin.Drawing`, `Lapin.Mesh` or `Lapin.Geometry` references any module;
- a UI module under `ui/src/modules/<id>` imports from another UI module it does not require;
- an open module depends on a premium one (the rule in `CLAUDE.md`).

**Done when:** the check is in `pnpm check`, and a deliberate bad reference fails it.

### 1.4 One tolerance, one place

**Octopus:** one length tolerance for the whole model, carried in a context object, never a literal (its decision A10). Per-function epsilons are named there as "the classic source of geometry bugs".

**Lapin:** `engine/src` holds about 2,780 literal `1e-6`, `1e-7`, `1e-9` and `1e-12` values. The audit of 22.11 gives "the same rule written several times, and the copies diverged" as its first cause (text width measured 6 ways, over 60 number parsers in the UI, 3 chord rules).

**Change:**
1. Add `Lapin.Model/Tolerances.cs` with named values: `Length` (model metres), `Angle`, `Area`, `Parameter`, and `Screen` for display only. Each gets a doc comment saying what it is for.
2. Add `scripts/check-literals.mjs`, a ratchet on the count of tolerance literals per file. It fails when a file gains one.
3. Replace literals file by file while touching code for other reasons. Never mass-replace: two literals with the same value may mean different things.
4. Do the same for the UI's number parsing: one `parseNumber`/`formatNumber` module, and a ratchet on `parseFloat(`, `Number(` and hand-written regexps in `ui/src/modules`.

The same idea applies to every duplicated rule the audits list. Make one owner per rule, and lint against new copies. `Lapin.Geometry/Chords` is already the model for this.

**Done when:** the ratchets are in `pnpm check`, and the counts fall release by release.

## Wave 2: the audits' recurring causes (weeks)

### 2.1 One user action, one command

**Octopus:** all mutation goes through commands, and one user action is one command (its B1). Composite actions are a single command that does several edits in one transaction. There is no back door for the UI.

**Lapin:** audit 22.11, cause 2 (§3.1). Many UI handlers send several `execute` calls for one gesture, giving several undo steps and partial results when a middle one fails. Examples are Elevations (4 × `view.create`), "Clear all finishes", Studio arrays and "Place on sheet".

**Change:**
1. **Rule:** a UI event handler calls `execute` at most once. Several edits go through `edit.batch` or a dedicated engine command.
2. **Check:** `scripts/check-one-step.mjs` scans `ui/src` for functions that call `execute(` more than once, or inside a loop or `map`/`forEach`, with a baseline of today's offenders. Allow an explicit `// one-step: <reason>` marker for real multi-step flows such as wizards.
3. **Engine side:** a test in `Lapin.Engine.Tests` asserts that a failing step of `edit.batch` leaves the model exactly as before (diff of the store before and after is empty).
4. Fix the audit's list, then shrink the baseline.

**Done when:** the check is in `pnpm check`, and the §3.1 list is gone from the baseline.

### 2.2 No stale responses

**Octopus:** the interface never holds its own copy of model state fetched on the side. It reads the document and refreshes from the diff every transaction produces (B2: "undo, regeneration, UI refresh, autosave and network sync all consume the same diff").

**Lapin:** audit 22.11, cause 3. About 15 dialogs ask the engine and let a slow, old answer overwrite a newer one (G18: room B's panel shows room A's openings, then writes them to B).

**Change:**
1. One helper in `ui/src/engine/session.ts`, say `useEngineQuery(name, args, deps)`. It tags each request, drops any answer that is not the latest for that component, and asks again when the session's diff touches what the answer read. That makes it the `live` flag the audit asks for, built in rather than remembered.
2. **Check:** a ratchet on direct `query(` calls inside `useEffect` in `ui/src/modules` and `ui/src/shell`.
3. **Test:** a vitest case that delays the first answer and checks that the second wins.

**Done when:** the audit's list uses the helper, and the ratchet is in `pnpm check`.

### 2.3 Each invariant enforced at the one place every path goes through

**Octopus:** a rule is enforced where every path must pass. The command validates its arguments, not the panel that calls it, because scripts, extensions, tests and sync call the same command.

**Lapin:** two of the audit's worst bugs are invariants enforced on one path only:
- G2: `RefuseOverRead` is called only by `project.exportIfc`, so other exports and `file.write` can overwrite a linked IFC or DXF. That breaks "an existing IFC file is never written to".
- G16: `element.update` on a storey skips `MaxStoreyHeight` and `CheckLevel`, which Python, MCP and the assistant can reach.

**Change:**
1. **File writes:** every file the engine writes goes through one function (`AtomicFile` is the natural place), and that function refuses a path that a link or import reads. Add a test that enumerates every command in the catalogue that takes a path argument, points it at a linked file, and expects a refusal.
2. **Validation:** the checks of a class's attributes live in one validator per class, which `element.update`, `element.create` and every specialised command call. Add a test that runs each validated attribute through `element.update` and through the specialised command and expects the same answer.
3. **Rule in `CLAUDE.md`:** an invariant is enforced in the engine at the narrowest point all paths cross, never in the UI.

**Done when:** G2 and G16 have tests that fail on today's code and pass after the change.

## Wave 3: tests that find what reading does not (ongoing)

### 3.1 Tests written from the specification, not by the implementer

**Octopus:** an agent does not write the tests that decide whether its own work is accepted (`docs/orchestration.md` there, 17.11.2026). For every piece with numbered rules, a second agent writes one test per rule from the specification alone. It sees the public signatures but may not read the implementation. The implementer may not edit those tests. A disagreement between the two is a place where the spec was read two ways, and is resolved against the spec. In Octopus, every specification slip so far was found this way and none by the implementer's own tests.

**Lapin:** about 110 bugs were found by reading in the 22.11 audit, many of the kind such tests catch at once. G10: `treadTarget` is never read. G4: stirrup zones are ignored. G15: `B.Code` loses its set name.

**Change**, for engine work with numbered rules (`docs/spec-*.md`, and plan files that state rules):
1. Number the rules in the spec (`Rule 7. …`) when they are not already.
2. A second session (Sonnet is enough) writes `engine/tests/Lapin.Engine.Tests/Spec/<Area>SpecTests.cs`, one `[Fact]` or `[Theory]` per rule named `Rule07_…`, from the spec file and the command catalogue only.
3. The implementing session may not edit `Spec/` files. If it thinks one is wrong, it says so with the rule's text, and the user or a design session decides.
4. Rules that state a property ("volumes add up", "the joined wall's length equals …") get randomised tests, not single examples (3.2).

The same cases and numbers apply to the tests ported from the old Lapin: they are already specification tests in this sense. Keep them untouched.

UI feel, layout and pictures are exempt: the guide's scenes and the user's eye are the test there, as in Octopus.

### 3.2 Laws as randomised tests

**Octopus:** properties are tested over generated inputs, not examples. For example, "the result equals a full rebuild" and "save then load is the identity".

**Lapin:** about 12 test files use randomness. The laws below hold Lapin's architecture together and deserve generated tests (CsCheck or FsCheck, both permissive, test-only):

| Law | Why it matters here |
|---|---|
| **Incremental equals fresh.** After any random sequence of commands, every element's geometry in the live session equals a fresh session built from the same model | Rebuild by read sets is fast because it skips work. A read that is not recorded makes a stale wall that no example test sees |
| **Undo then redo is the identity**, and **undo of everything** gives the starting model | Each command is one undo step; the earlier-build reuse in `Session.Builds.cs` must not change results |
| **Save then load is the identity** (store and geometry), for `.lapin` format 3 | The package is the user's work |
| **IFC export then import** keeps every recognised element within the fidelity gate | Interop with BubbleBIM depends on it |
| **Quantities add up.** A wall's layer volumes sum to the wall's volume; cut pieces sum to the whole | Cost and estimates read them |
| **Determinism.** The same commands on the same model give byte-identical saves and meshes on desktop and in the browser engine | Shared sessions and the browser build |

Each law is one test class with a command generator over the catalogue (`catalogue.json` already describes the arguments). Keep the run time bounded with a fixed seed in CI and a long run by hand.

### 3.3 Hostile input for every reader

**Octopus:** every reader is tested with truncated, mutated and oversized files. A reader returns a typed error and never allocates without a bound.

**Lapin:** G19: a 200-byte `.bos` asks for 8 GB, and a `.lapin` can be a zip bomb. The user sees "Internal engine error". `BoundedRead` exists in `Lapin.Model` but is not used everywhere.

**Change:**
1. A list of every reader: `.lapin`, IFC/STEP, DXF, BOS/Thrift/Parquet, LandXML, PLY and Gaussian PLY, LAZ, SDNF, glTF/mesh, CSV, images.
2. A shared test harness, `HostileInput.Run(reader, sampleFiles)`. It truncates at random points, flips bytes, inflates length fields and nests zips. Each case must end with the reader's typed error or a valid result, within a time and memory budget, and never with an unhandled exception.
3. Every length or count read from a file goes through `BoundedRead` with a limit derived from the file's size.

**Done when:** every reader in the list has a hostile-input test class, and G19 is fixed.

### 3.4 Mutation testing on the engine

**Octopus:** after a package's tests pass, `cargo mutants` makes small deliberate errors in the changed code. Each survivor gets a test, or a written reason why it cannot matter. It costs machine time only.

**Change:** use Stryker.NET (Apache-2.0, a dev tool, not shipped) on the files a branch changed in `engine/src`, by hand or as a script (`node scripts/mutants.mjs <files>`). Start with the kernels whose mistakes are silent: `Lapin.Geometry`, `Lapin.Booleans`, the joins, quantities, `Lapin.Analysis` design checks, `Cpm.cs`, and `Lapin.Package`. A branch touching those reports its surviving mutants in its history paragraph.

## Wave 4: deeper contracts (when the waves above are in)

### 4.1 Names for every sub-element a dimension can hold

**Octopus:** every face, edge and vertex a generator makes carries a name derived from what made it (source element, profile segment, layer, operation), never an index (A12). Dimensions, tags, hosted elements and constraints refer to names, so they survive regeneration.

**Lapin:** faces have stable roles (`ElementFace.Role`), which covers most of it.

**Change:** find every place that refers to geometry by index or by coordinates: dimension end points, snaps saved into the model, painted finishes, hosted elements on Studio solids and free-form parts. Move each to a role or a derived name (`Edge[Side.Left∩Top]`, `Layer[2].Face[Exterior]`). Add one test per referrer: make the element, attach the reference, then move, stretch, cut and undo the element. The reference must still resolve to the same logical place.

### 4.2 A text form of the project for Git and for tests

**Octopus:** a lossless "exploded" form of the project file: a folder of sorted, pretty-printed JSON files, one per part. It is used for diffing in Git, golden tests and debugging (C3).

**Change:** `lapin explode <file.lapin> <folder>` and `lapin implode <folder> <file.lapin>`, where implode(explode(x)) is byte-identical to a canonical save of x, plus a round-trip test. Use it for golden tests of commands: a command's effect is the diff of the exploded folders, readable in a test failure and in review.

### 4.3 Results that do not depend on the platform

**Octopus:** results are identical on every platform. It avoids the system maths library and uses its own `trig` module, never iterates a hash set where order shows, and reads nothing from the clock or the environment.

**Lapin:** about 700 calls to `Math.Sin`, `Math.Cos` or `Math.Atan2` in `engine/src`. .NET's results can differ in the last bit between platforms and between desktop and the browser's WebAssembly engine. The determinism law in 3.2 tells whether this matters. If it fails, add a `Trig` module in `Lapin.Geometry` and a ratchet on direct calls. Do not do this before the law shows a difference.

## Architecture: what to keep, and three ideas to adopt

### The two stacks

| | Octopus | Lapin |
|---|---|---|
| Engine | Rust, with Manifold (C++) behind one FFI file | C# on .NET 10, with Manifold and Clipper2 through their C APIs |
| Interface | egui, drawn on the GPU in the same Rust process | React 18 + TypeScript + zustand, in a Tauri 2 web view |
| 3D view | Its own renderer on wgpu (Vulkan, Metal, DX12, WebGPU) | three.js, three-mesh-bvh, three-gpu-pathtracer |
| Engine ↔ interface | Function calls in one process; the UI reads the document directly | WebSocket JSON-RPC to the engine process; in the browser, a Web Worker |
| Data model | A generic entity–component store; IFC is a mapping on export | IFC entities and objectified relationships held as they are |
| Panels and serialisation | Generated from a derive macro on each component type | Written per module; the command catalogue is generated (`scripts/command-catalogue.mjs`) |
| Threads | Single-threaded by contract; parallel only inside rebuilds | Parallel element builds (`Parallel.For` in `Session.cs`), concurrent deliveries |

**Decide the stack on measured targets, and soon: the cost of a rewrite has fallen, and the cost of lock-in has not.** The engine–interface protocol is what gives Lapin its browser edition, shared sessions, Python, MCP and the assistant from one codebase. Those must survive any change. But "not yet" is the trap the incumbents fell into, and with AI writing most of the code, a rewrite is far cheaper than the incumbents' experience suggests. The case for moving is set out in full below (A.0), with the measurements that decide it and the work that makes the move safe (A.4). The protocol also has a price today: two of the audits' three recurring causes, stale answers (2.2) and several undo steps for one gesture (2.1), live on that boundary. A.1 to A.3 reduce that price without changing the stack.

### A.0 The case for another stack

**What the incumbents teach.** The established BIM tools started in the 1980s and 1990s. Years later they are still held back, and it is not by their language. It is by four early choices that everything else was built on:
- **The threading model.** Regeneration that was single-threaded by design is widely reported as the main limit on large models. Parallelising it afterwards means proving that no code anywhere relied on running alone.
- **The memory layout.** A model of many small heap objects with pointers between them cannot be streamed, mapped or shared, and becomes slow at scale.
- **The data model.** Whatever the in-memory model is becomes the API of every add-on and every file, so it can never be changed.
- **The interface toolkit.** An interface woven into the model code ages with it.

None of these can be fixed later without a rewrite, and for them a rewrite of a product with users rarely happened, because writing the code was the expensive part. The lesson is not "write it in Rust". It is: **decide these four while the code is young, on evidence, and keep the decision reversible until then.**

**Where Lapin's own numbers point the same way.** These are from Lapin's own measurements, not from Octopus:
- **Memory per element.** The builds of a model of 4,000 walls and 14,000 elements hold about 700 MB of managed heap (`SessionTuning.Desktop` in `Session.Builds.cs`). A 1.2 GB IFC needs 15.7 GB after import, and at first could not be read at all, because a .NET string holds at most 1 GB (`docs/plan-instancing.md`, 29.10.2026). Hospitals, airports and campuses are 10 to 100 times the 14,000-element model.
- **Allocation and the garbage collector.** On the generated 18,000-entity model `gen-10`, changing one wall type took 4.6 s and allocated 19.7 GB; opening it allocated 14 to 20 GB. The profile shows "a large share is garbage-collection suspension" (`docs/experiment-editing-speed.md`). The fixes taken (decisions 423–426) cut this by about a third. That helped, but the cost grows with the model, and a garbage-collected heap of millions of small objects is the memory-layout trap above.
- **The browser and the interface heap.** The browser edition has a 2 GB ceiling and one thread (`docs/plan-browser-engine.md`, `SessionTuning.Browser`). At a million instances, the interface's JavaScript heap is 1.07 GB, and turning the view hitches at 150 ms, "likely the garbage collector" (`docs/plan-instancing.md`). Every mesh crosses a process boundary and is held twice, once by the engine and once by the page.
- **Every feature twice.** A module has an engine half in C# and an interface half in TypeScript, kept in step by a protocol and by checks (`check-commands.mjs`, the catalogue). That is the source of the protocol-boundary bugs in 2.1 and 2.2, and it doubles the code an AI session must read to change one tool.
- **The data model.** Holding IFC entities as they are is Lapin's most incumbent-like choice. IFC is an exchange schema, and the in-memory model becomes the API of Python, MCP, extensions and the store. That is the lock-in described above, already taking shape: the extension store has been live since 12.11.2026.

**What a native single-process stack (Octopus's choice) would fix, and what it costs.**

| Fixes | Costs |
|---|---|
| Memory you lay out yourself: columns and arenas rather than an object per entity, no collector pauses, files can be mapped | The browser edition becomes a separate WebAssembly build of the same code, with its own renderer backend |
| One copy of each mesh, no serialisation between engine and screen, no stale answers | Python, MCP and shared sessions need a protocol added back, deliberately, at the edge |
| One language: a tool is one module, not two halves | The interface is rebuilt in a native toolkit (egui or similar), which is less rich than React for forms and documents |
| Threading checked by the compiler rather than by tests (A.3) | About 630,000 lines (C# engine and TypeScript UI) to port. With AI writing the code this is weeks to a few months, not years (below), but it must be verified, and both engines maintained, during the move |

**A rewrite when AI writes the code.** The old rule, "never rewrite a product with users", assumed that writing the code was the expensive part. Here it no longer is, and Lapin is the proof:
- Lapin itself is a rewrite. The transfer from the ifc-lite fork began on 23.10.2026 (`docs/transfer.md`). A month later the repository holds about 400,000 lines of C# and 230,000 of TypeScript, about 800 decisions and a shipping product. (The geometry kernel came from BubbleBIM Pro, so not all of it was written in that month.)
- Lapin has already swapped its whole engine once behind the same protocol. M0 ran on BubbleBIM's host, M1 on Lapin's: "the protocol with the interface stays the one from M0; only the engine behind it changes" (`docs/plan.md`). A native engine could replace the C# one the same way.
- Octopus wrote about 150,000 lines of Rust in its first two phases, in under two weeks after its Gate 0 (12.11.2026).

What AI does **not** make cheaper is what now decides the cost of a rewrite:

| Still expensive | Why | What makes it cheap |
|---|---|---|
| **Knowing the new code does what the old did** | AI can write a port in days, but cannot see that a wall join changed by 2 mm | The conformance suite: the spec tests (3.1) and laws (3.2), run through the protocol against both engines, plus golden outputs: exploded `.lapin` (4.2), drawings and IFC exports of real projects |
| **Users' files** | Every `.lapin` ever saved must open | `IProjectUpgrade` and the format's spec; a reader test over every saved file version |
| **Other people's code** | Python scripts, MCP clients, visual graphs and store extensions use the current API | The protocol and the command catalogue as the frozen API; extensions talk only to it |
| **The decisions** | About 800 decisions and the audits are the product's real knowledge | They are in `docs/` already, and they carry over to any stack unchanged |
| **Running two engines at once** | Bugs fixed in one must reach the other | A short move: freeze features in the old engine while the new one catches up |
| **Compute** | Agent runs and long test runs cost usage limits | Waves of at most two agents, as Octopus does |

So the incumbents' lesson cuts both ways here. The code is no longer what locks Lapin in. What locks it in is everything built on top of the code: users' files, scripts, extensions, and in-memory models that others depend on. All of that grows every week, and AI does not shrink it. **The cheapest time to change the stack is the earliest time the conformance suite can prove the change.**

**The decision this file asks for.**
1. **Build the conformance suite first** (3.1, 3.2, 4.2). It is worth having whatever the stack, and it is what turns a rewrite from a gamble into a measured port.
2. **Set the targets now** and measure them at 10 times today's largest test model. Suggested targets for a 150,000-element model on a mid-range laptop:
- opening it takes under 15 s;
- a small edit takes under 50 ms at p95;
- a type change across thousands of elements takes under 2 s;
- peak memory stays under 8 GB;
- turning the view has no p95 hitch over 33 ms.

3. **Prototype the other stack in parallel, now.** AI makes this cheap enough to settle the question with evidence rather than argument. On a branch or in its own repository, have agents build a native engine (Rust, as Octopus does) that serves **the same protocol** for a vertical slice. The slice is: the model store, storeys, layered walls with joins, openings with doors and windows, and `.lapin` format 3 read and write. Run the same benchmark (A.4.1) and the same conformance tests against both engines. Two to four weeks of agent time is enough to know.
4. **Decide from the two tables** (the targets, and the conformance results of the slice):
   - **The C# engine meets the targets**, or meets them after the cheap fixes (columnar storage for hot attributes, since Arrow is already the file format; pooled mesh buffers; structs in the geometry kernel; NativeAOT): keep it, and drop the prototype.
   - **The native slice is clearly better and passes the same tests:** move, and move fast, not over years.

**How to move, if the decision is to move.** Do it as Lapin did at M1: a whole engine swapped behind the protocol, not a slow migration through C APIs that keeps both alive for years.
1. **Freeze** new engine features for the move. Bug fixes go to both engines, and the conformance suite gets a test for each.
2. **Port module by module** with agents, at most two at a time, each module accepted only when its spec tests, the laws and the golden outputs pass through the protocol against the new engine. The decisions in `docs/` are the agents' specification, so port the tests before the code, as `CLAUDE.md` already requires for the ifc-lite transfer.
3. **Run both engines side by side** in development and in the guide's scenes (`scripts/user-docs/`). A diff of their answers to the same commands is a failing test.
4. **Switch** when every module passes, ship one release with the old engine still in the package behind a setting, then remove it.
5. **Keep the interface.** The React UI stays, so users see the same tool. A native viewport (wgpu in the Tauri window, fed by the engine rather than the page) is a separate, later decision on the same evidence, because it is where the 1M-instance hitches are.

Whichever way the measurements go, the work in A.4 keeps the move possible and cheap. Without it the incumbents' position is where Lapin ends up by default, AI or not.

### A.1 Types for the protocol, generated from the catalogue

**Octopus:** one description of each type (its derive macro) produces panels, serialisation and scripting bindings, so they cannot disagree.

**Lapin:** `engine/src/Platform/Lapin.Protocol/catalogue.json` already describes every command and query with its arguments. The UI still calls `execute('name', { … })` with an untyped object. A misspelt name is caught by `check-commands.mjs`, but a wrong or missing argument is only caught at run time.

**Change:**
1. Generate `ui/src/engine/protocol.gen.ts` from the catalogue: an interface of each command's arguments and answer, and a typed `execute<K extends CommandName>(name: K, args: Args[K])` overload beside the untyped one.
2. `command-catalogue.mjs --check` also checks that the generated file is current.
3. Move modules to the typed overload as they are touched, with a ratchet on untyped `execute(` calls in `ui/src/modules`.
4. Where the catalogue cannot work out an argument's type, it says `unknown`. Improve the catalogue there rather than hand-writing types.

The same file can give Python's `lapinbim` its type stubs, so the three clients (UI, Python, MCP) read one description.

**Done when:** a wrong argument name in a module fails `pnpm typecheck`.

### A.2 The interface reads the session, and the session is fed by diffs

**Octopus:** the interface never keeps a side copy of model state. It reads the document, and refreshes from the diff each transaction emits.

**Lapin:** half of this is built. `ui/src/engine/session.ts` folds every engine diff into the session's nodes (`applyDiff`, skipping diffs older than what is shown). The stale answers of audit cause 3 come from panels and dialogs that bypass it, asking the engine with `query(…)` and holding the answer in local state.

**Change:**
1. **Rule:** model data a panel shows comes from the session's nodes, or from a derived query that names what it reads and is asked again when a diff touches that.
2. Derived data that is too costly to ship in every diff (room openings, quantities, a schedule's rows) gets one engine-side query per kind. It returns the ids it read (the engine already records reads for builds), and the UI helper of 2.2 asks again only when a diff touches one of them.
3. 2.2's ratchet on `query(` inside `useEffect` measures progress.

**Done when:** the dialogs in the audit's list read the session or the helper, and none holds a model answer in `useState`.

### A.3 Threading rules, written and tested

**Octopus:** the core is synchronous and single-threaded by contract. Parallelism is applied only inside regeneration, over independent elements, each reading through its own view. No async below the IO layer.

**Lapin:** the session builds elements on every core (`Compute` in `Session.cs`: each reads through a view of its own) and accepts concurrent deliveries (`Deliver`, with `Interlocked`/`volatile` state). This is fine as built, but the rules are only in comments. A new module's generator that writes to a static cache, or reads the store outside its view, races silently.

**Change:**
1. Write the rules down in `docs/plan.md` or a short `docs/threading.md`:
   - What runs on worker threads: builds, emitted drawings.
   - What they may touch: their read view, immutable inputs, and thread-safe caches named in a list.
   - What runs only on the session's thread: commands, diffs, undo.
2. A test runs every module's generators over a generated model twice, once on one thread and once on all cores. The geometry must be identical. A difference means shared state.
3. A check flags new `static` mutable fields (non-`readonly` statics, and static `Dictionary`/`List`) in `engine/src/Modules`, with a baseline.

**Done when:** the rules are written, the one-thread/all-cores test runs in CI, and the static-field ratchet is in `pnpm check`.

### A.4 Keep the move possible

These cost little now and make every option in A.0 cheaper later.

1. **A scale benchmark, run monthly.** Generate the 150,000-element model (extend `gen-10`'s generator) and measure A.0's targets with `EditingSpeedBench.cs` and the instancing script. Record the results in a table in this file: date, commit, each target, pass or fail. That table decides the stack question.
2. **The protocol is the only contact between interface and engine.** Never share state any other way: no Tauri command that reaches the model, and no engine logic copied into the UI. Any part can then be replaced behind it.
3. **The geometry kernel's interface carries no .NET-only types.** It takes plain arrays and records in, and gives meshes and B-reps out. That way it can become a native library (A.0, step 2) without touching its callers. Add a layering rule for it (1.3).
4. **Hot paths over arrays, not object graphs.** New code that holds per-element or per-triangle data in bulk uses arrays or Arrow columns, not a list of small objects. Add this to the review checklist; measure it with the allocation numbers of `EditProf`.
5. **The laws of 3.2 and the spec tests of 3.1 are the conformance suite.** They are written against commands and results, never against C# internals, so any replacement engine, native or not, must pass them unchanged. This is what makes a move safe, gradual or whole. The incumbents never had such a suite, and it is a large part of why they could not move.
6. **The data model behind a mapping.** New Lapin-only concepts (intent, constraints, sub-element names) live in Lapin's own components beside the IFC entities, not inside IFC attributes. Then the in-memory model can drift from IFC where editing needs it, as Octopus's does, without breaking import and export.

## Community: a small core that others build on, as Blender is

**What makes Blender work for a community.** Being open source is not what does it. These things are:
1. **Everything is an operator.** Every action, built in or not, is a registered operator with typed properties, undo and a keymap entry. Add-ons call the same operators the interface does, and register new ones the same way.
2. **One reflection layer (RNA) describes all data.** From it come the Python API, the property panels, animation, drivers and the generated API reference. A type added by anyone gets all of these for free.
3. **The interface is written in the same API the community uses.** Blender's own panels and menus are Python, so an add-on can do anything the shipped interface does, and reading the shipped code is how people learn.
4. **One way in.** `register()` / `unregister()`, a manifest, reload without restarting, and since 4.2 an extensions platform with permissions.
5. **A way into the core, too.** A public tracker and code review, module owners who decide for their area, developer docs, and a fund that pays core developers. Most of Blender's features were once someone's add-on or patch.

Octopus takes the same line as its rule B8, "modules use the same registry as extensions; if a module needs a private hook, fix the registry". It also takes the reflection layer as B4 (a derive macro per component type).

**Where Lapin stands (24.11.2026).**

| Blender's piece | Lapin | Gap |
|---|---|---|
| Everything is an operator | Every tool is one engine command with JSON arguments and one undo step; the command catalogue describes them all (`catalogue.json`) | Small. 2.1 closes the multi-command gestures |
| Python drives the same operators | `lapinbim` drives the model through the protocol, typed from the catalogue (`docs/plan-python-scripting.md`) | Small. Python cannot yet add panels or tools to the interface (C.3) |
| One reflection layer | `ClassDescriptor` per IFC class in the engine; property sections written by hand per module in the UI | Medium (C.4) |
| Built-ins use the community's door | **No.** The UI's built-in modules make about 2,050 imports from shared code, and only about 315 of them come through `ui/src/platform/`, the contract. The rest reach straight into `shell/` (462), `app/` (365) and `engine/` (903). In the engine, modules see all 752 public types of `Lapin.Platform` | **Large** (C.1) |
| Load without rebuilding | Engine modules are loaded at run time from `artifacts/modules/<id>/` (`ModuleLoader`), into the default load context, with no isolation or unload. UI modules are compiled into the build (`virtual:lapin-modules`) | Large for the UI, medium for the engine (C.2) |
| Extensions platform | The store is live: content packs, Python scripts, signed packages, paid extensions at 0 % (`docs/plan-extensions.md`). JS plug-ins are planned "later", and C# modules "for partners, signed" | The platform is there; what it may carry is narrow |
| A way into the core | The repository is private; contributor-facing docs are half in Romanian; no CONTRIBUTING | Large (C.6) |

The core of the problem is the fourth row. Today a community developer would get a narrower and different door than Lapin's own modules use. Each time a built-in module reaches past the contract, the contract falls further behind what a real module needs. This is the "privileged first-party code path" that Octopus's B8 rejects.

### C.1 One door for built-in modules and the community

1. **The UI contract becomes a package.** `ui/src/platform/` grows into `@lapin/sdk`: the module contract, plus what the built-ins really use from `shell/` (cells, dialogs, prompts, colour fields, parameter editors), `app/store` (read-only selectors), and `engine/session` (typed `execute`/`query`, A.1). Built-in modules import only from it.
2. **A ratchet** (`scripts/check-module-imports.mjs`) on imports from `ui/src/modules/*` into anything except the SDK and the module's own folder. Today's count is about 1,730. It only goes down, and reaches zero.
3. **The engine contract becomes an assembly.** `Lapin.Sdk` holds `IModuleRegistry`, `ClassDescriptor`, the producers, `IEdit`, `IModelView`, and the geometry types modules need. Modules reference `Lapin.Sdk` and `Lapin.Model`, not `Lapin.Platform`. What a module needs from the platform's insides moves into the SDK on purpose, or becomes a service (`IModuleRegistry.Service<T>` already exists).
4. **The SDK is versioned.** It uses semver, with a public-API baseline file (`Microsoft.CodeAnalysis.PublicApiAnalyzers` for C#, `api-extractor` for TypeScript). Removing or changing a member fails the build unless the major version goes up.
5. **Rule:** if a built-in module needs something the SDK does not have, add it to the SDK. Never reach around it.

**Done when:** every built-in module, engine and UI, builds against the SDK alone, and the ratchet is at zero.

### C.2 Extensions loaded at run time, isolated

1. **UI modules from a folder.** A UI extension is an ES module bundle plus `module.json`, loaded with `import()` from the extensions folder on desktop. It gets the same `@lapin/sdk` instance as the built-ins, as a shared import map. A built-in module may also be shipped this way, which proves the path works.
2. **Engine extensions out of process, by default.** A community engine extension runs as its own process and speaks the protocol Lapin already has (JSON-RPC, as the host does for the UI). It registers its commands, queries and geometry producers by name through a `module.register` handshake. This has four advantages:
   - It works in any language: Python, C#, Rust, JS.
   - A crash or a hang cannot take the session down.
   - Permissions are enforced at the protocol (`EngineDispatcher.AllowedRemote` already does this for guests).
   - **It survives a change of engine stack (A.0).** An in-process C# DLL does not. Of everything in this section, this matters most: every in-process C# extension written now is a reason never to leave .NET.
3. **In-process C# modules** stay for Lapin's own modules and signed partners. Each gets its own collectible `AssemblyLoadContext`, so it can be unloaded and updated without a restart.
4. **Reload without restarting** for both kinds, in a developer mode. Blender's quick edit-and-reload loop is what keeps add-on authors working.

### C.3 Python and declarative UI: extend the interface without writing React

Blender's lowest step in is "a Python file that adds a panel". Lapin's equivalent:
1. **Declarative panels, tools and menu items**, described as data: fields, buttons, the command each one runs, and a gesture from `Gesture` in `module.ts`. The SDK renders them with the same components as the built-ins. A Python script or an out-of-process extension sends that description with `ui.register`, and the interface draws it on desktop and in the browser.
2. The **Script panel's script tools** (decision 489) become the first users of it.
3. Built-in modules move their simple property sections to the same description, which proves it is enough and keeps it honest.

### C.4 One description per class, many uses

As with Blender's RNA and Octopus's B4: extend `ClassDescriptor` (or a property catalogue beside `catalogue.json`) with each attribute's type, unit, range, label, group, and whether it is a type or an instance parameter. From it, generate:
- the default property panel;
- the Python attributes (`wall.height`);
- schedule columns, filters and formula fields (`FieldCatalog.cs` already lists fields);
- the API reference.

A community module that declares a class then gets a working inspector, Python access, schedules and docs without writing any of them. Hand-written sections stay where a module wants more.

### C.5 Developer experience

- `pnpm create lapin-extension <kind>`: templates for a content pack, a Python tool, a UI module and an out-of-process engine module, each with a test and a CI file for the catalogue's validator.
- **API reference generated** from the SDK, the command catalogue and the class descriptions (C.4), published beside the user guide.
- **Example extensions** in `lapin-extensions`, one per kind, kept building by its CI against each release.
- **A compatibility promise:** the SDK's major version and the protocol's version are in every manifest, and the store refuses an extension built for a version the app no longer serves.

### C.6 A way into the core

This is only for a part of Lapin the owner chooses to open. The premium modules stay private, as `CLAUDE.md` says.
1. **Contributor docs in English.** The forum is already English. `docs/plan.md`, the audits and parts of `docs/plan-extensions.md` are Romanian, which most contributors cannot read. Keep Romanian for the user guide (`docs/user/ro`).
2. **`CONTRIBUTING.md`:** how to build, `pnpm check` (1.1), the spec-test rule (3.1), the commit style, and the licence: inbound MIT, as outbound. Also say what will not be accepted: anything that makes an open module depend on a premium one.
3. **Areas with owners**, as Blender's modules have, mapped to `engine/src/Modules/<Group>`. Each area has a spec file and its numbered rules. A pull request is accepted by `pnpm check`, its spec tests and review (Claude, then the area's owner), which is the same mechanical acceptance as Octopus's `docs/orchestration.md`.
4. **Good first issues** taken from the audits' smaller bugs. There are about 110 confirmed in `docs/audit-2026-11-22.md`, and each comes with a file, a line and a reproduction.
5. **The open-core line in writing.** List what is open and stays open, what the hooks are for premium modules (`docs/overrides.md`), and the promise that a feature contributed to the open core will not be moved behind a subscription. Contributors need that answer before they give their time.

**What this changes for AI-run work.** A stable, small SDK is also what makes AI agents effective. A brief can say "read `Lapin.Sdk` and the module's own folder" instead of "the platform", and tests written against the SDK and the protocol hold across rewrites. That is the same door for people, agents and a future engine.

## Process: how the AI work itself is run

These are the parts of Octopus's process that pay off in an AI-run project, and fit Lapin's pace.

1. **Specification before code, for engine rules.** A feature with rules gets numbered rules in its `docs/spec-*.md` or plan file before code. The rules are what 3.1's tests are written from. Lapin's plan files already hold most of them, often unnumbered.
2. **Escalate rather than guess.** When a spec does not decide a case, the implementing session picks the most conservative reading, says so in its report, and adds the question to `docs/decisions-pending.md`. It does not choose silently.
3. **Briefs for sub-agents name their files.** A brief lists the files to read and the paths that may be touched, and says "do not edit `Spec/` tests or other modules". It never says "the repository". Ask for a report of ambiguities, defects found elsewhere, and anything not done.
4. **Accept by checks, not by reading.** A branch is accepted when `pnpm check` passes, the spec tests pass, and the mutants are explained. The orchestrating session reads the report, not every line of code.
5. **At most two sub-agents at a time**, each in its own worktree with its own build output (`engine/artifacts` and `bin/obj` are not shared between worktrees).
6. **Measured numbers are the agent's until re-run.** When a report gives timings or counts, re-run them before writing them into `docs/history.md`.
7. **An audit per month.** It should be read-only, use parallel sessions, and not repeat the previous one, as on 15.11 and 22.11. Its recurring causes turn into checks, not only fixes. That is the step that makes this plan pay.

## What not to take

- **A rewrite before the evidence.** Octopus's native stack answers real limits, set out in A.0, and AI makes a rewrite affordable. But a rewrite before the conformance suite exists cannot be verified, and one before the benchmark cannot be justified. Build both first, prototype in parallel, then decide.
- **Running CI only at gates.** Octopus does that to save runner minutes on a private repository. Lapin's CI on every push is better; keep it.
- **The full work-package ceremony everywhere.** Octopus itself scopes the second-agent tests and mutation runs to code whose mistakes are silent (numbers, solids, files). Do the same: engine kernels and readers yes, panels and ribbons no.
- **Proprietary licence rules.** Lapin stays MIT. Its own rule, permissive production dependencies, already covers what matters.

## Summary for a session picking up this plan

| Wave | Item | Size | Fixes |
|---|---|---|---|
| 1 | 1.1 `pnpm check` | S | checks forgotten |
| 1 | 1.2 file size ratchet | S | files too big to read whole |
| 1 | 1.3 layering check | S | module coupling, premium leaks |
| 1 | 1.4 one tolerance, one parser | M | audit cause 1 |
| 2 | 2.1 one action, one command | M | audit cause 2 |
| 2 | 2.2 no stale responses | M | audit cause 3 |
| 2 | 2.3 invariants at one point | M | G2, G16 |
| 3 | 3.1 spec tests by a second session | ongoing | bugs found only by audits |
| 3 | 3.2 laws as randomised tests | M | stale rebuilds, round trips |
| 3 | 3.3 hostile input | M | G19, crashes on bad files |
| 3 | 3.4 mutation testing | ongoing | code no test looks at |
| 4 | 4.1 names for sub-elements | L | dimensions lost on edit |
| 4 | 4.2 text form of `.lapin` | M | reviewable diffs, golden tests |
| 4 | 4.3 platform-independent maths | M (if needed) | desktop/browser drift |
| Arch. | A.0 conformance suite, targets, a native prototype, then the stack decision | weeks | the incumbents' lock-in |
| Arch. | A.1 protocol types from the catalogue | M | wrong arguments found at run time |
| Arch. | A.2 interface fed by diffs | M | stale answers, at the root |
| Arch. | A.3 threading rules and test | S–M | silent races in module generators |

| Arch. | A.4 keep the move possible | S each | a rewrite becoming the only way out |
| Comm. | C.1 one SDK for built-ins and extensions | L | built-ins using a door the community cannot |
| Comm. | C.2 run-time UI modules; out-of-process engine extensions | M–L | rebuilds to extend; extensions that tie Lapin to .NET |
| Comm. | C.3 declarative panels from Python | M | no low step into the interface |
| Comm. | C.4 one description per class | M | panels, Python and schedules written by hand per class |
| Comm. | C.5 templates, generated API reference | S–M | no starting point for authors |
| Comm. | C.6 contributor docs, owners, the open-core line | S, the owner's call | no way into the core |

A.1 and the benchmark of A.4 can go with wave 1. A.2 belongs with 2.2, and A.3 with 3.2. A.0 is the user's decision, made from A.4's table and the prototype's results. C.1's ratchet can start in wave 1; C.2's out-of-process design should be settled before the first community engine extension, so it does not tie Lapin to .NET; C.6 is the owner's decision on what to open.
