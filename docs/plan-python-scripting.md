# Python scripts in Lapin: plan

The user's decision (12.11.2026): "Python scripts in Lapin, tied to Python scripts that drive the model through the protocol — Lapin stays C# and MIT, with no dependencies — delivered as a plug-in, even in the default free edition, in the first phase." Asked for on the forum: an API for add-ons, a Python console that shows the interface's steps as code (as in Bonsai/IfcOpenShell), a developer switch (as in Blender).

**State (12.11.2026):** the user answered every question of §11 with the recommendations ("yes to all").
- **Stage 1 is done:**
  - the `lapinbim` package in `python/`;
  - the command catalogue at build time (`scripts/command-catalogue.mjs` → `session.catalogue`, shared with the MCP bridge);
  - `edit.batch {label}`;
  - the tests (`pnpm test:python`);
  - publishing to PyPI prepared (`scripts/publish-python.mjs`; the user publishes it with their token).

  The developers' guide: `docs/python.md`.
- **Stage 2 is done** (12.11.2026):
  - the open module `scripting` in `products/lapin.json` (the interface half `ui/src/modules/scripting`; the engine half the host methods `script.*` in `Lapin.Host/ScriptHost.cs`);
  - the user's Python found or chosen in Preferences, with the kernel `lapinbim.kernel` (`python -I -B -u`) shipped with the application in `resources/python`;
  - `doc.begin/commit/rollback` in the engine (`Session.Step.cs`, `History.Begin`);
  - the Script panel (console, CodeMirror 6 editor with completion from the catalogue, Run/Stop/Restart, Mine, external editor) behind Developer extras.

  The user's decisions of 12.11.2026:
  - `lapinbim` is published to PyPI **only together with a version of the application** (`docs/python.md`);
  - the premium repos make their own catalogue (`command-catalogue.mjs --root … --out …`, embedded as `Lapin.catalogue.json`; `docs/overrides.md`);
  - a lighter read of the model (entities without meshes) comes later (§8).
- **Stage 3 is done** (12.11.2026):
  - the recorder: the console's Recorder tab, caught at `EngineClient.onCall`. Undo removes the last step from the script, instead of `lapin.undo()`, which a script cannot do within the run's step;
  - `LapinScript` in the engine module `scripting`, with `scripts/<name>.py` in the package (other tools' parts in `scripts/` stay foreign);
  - trust by SHA-256, `lapin.selection()`, the element proxies;
  - the three points approved from stage 2's questions: the same kernel's connections join the step (a token per process, `script.join`), the engine's and the console's messages translated, and `lapin.step()`.

  The details are in `docs/python.md`.
- **Stage 4 is done** (12.11.2026):
  - `lapin.ui.ask`: a window dialog made of fields (a number with a unit, text, a choice, a checkbox, an element from the model or the selection, a storey, a type), through the host methods `script.ask`/`script.answer`; Stop takes it back and stops the script;
  - `lapin.ui.tool` and the plug-in header (`# lapin: tool=…, icon=…, run=…, requires=…`);
  - the tools on My tools, in the "Scripts" group, and in Commands…. They come from Mine; from the project's scripts only with trust; and from Extensions, with trust asked at the first click. One tool = one run = one undo step;
  - `script.untrust`;
  - the approved answer to stage 3: a guest does not delete, rename, group, copy or move a project script (`ScriptingModule.Before`).

The plan is complete; what is left are the "Later" points of §9.

## 1. What Lapin has today

- **Every tool is an engine command** with JSON arguments and one undo step (`doc.execute {command, args}`); questions that change nothing are `doc.query`. `edit.batch {steps}` runs several commands in a single step (`CoreModule`, its label is "Edit").
- **The protocol**: JSON-RPC over WebSocket on `127.0.0.1`, answers in binary `LPF1` frames (a JSON header + raw arrays, `BinaryFrame.cs`); diffs are sent to every connected interface. The port is in `<user-dir>/host.json` (`docs/mcp.md`).
- **The MCP bridge** (`scripts/lapin-mcp.mjs`, Node 22, no dependencies) already does what the Python client will do: it finds the engine, reads `session.info`, executes and asks. It takes the commands' arguments from the engine's sources next to it (`argsFromSource`), so in an installed application, without the sources, it does not have them.
- **`session.info`**: the product, the modules, the commands (name, label, no arguments), the classes with their fields, the unit `m`.
- **The caller**: a connection from the computer is `Caller.Local` (it may do anything); a guest of a shared session goes only through `EngineDispatcher.AllowedRemote`; the host's methods (`share.*`, `cancel.long`) are refused to guests.
- **The recovery journal** holds transactions as entity changes (before/after), not as commands with arguments: the macro recorder cannot use it directly (§5).
- **The editions' override points** (`docs/overrides.md`, decision 344), **the foreign parts** of `.lapin` (`dag/`, `source/`… kept byte for byte, decision 345), **an absent module's content** kept (decision 484), **the My tools tab** (decision 479), **later deliveries** (`ctx.Deliveries`).

## 2. A single API: the Python package `lapin`

**Recommendation:** a pure Python package, written by Lapin, MIT, **standard library only** (no `websockets`, `numpy` or other wheels). The same package is used by outside scripts and by the console in the application, so a script written in the console runs unchanged from a terminal, and the other way round.

- **The transport**: a minimal WebSocket client on `socket` (the handshake with `hashlib`/`base64`, masked frames, fragments, ping/pong; ~300 lines), and the `LPF1` frame reader with `struct` and `memoryview` (arrays come as `array.array`/`memoryview`; whoever has `numpy` does `numpy.frombuffer` themselves, without the package requiring it). The port comes from `host.json`, as for MCP, or `LAPIN_PORT`. A separate transport for the browser (§3) sits under the same interface.
- **Layer 1, raw**: `lapin.execute("architecture.addWall", {...})`, `lapin.query("schedule.csv", {...})`, `lapin.undo()`, `lapin.redo()`, `lapin.save()`, `lapin.info()`.
- **Layer 2, typed, generated**: `lapin.architecture.add_wall(x1=0, y1=0, x2=5, y2=0, height=2.8)` (the command's name in snake_case, named arguments only).
  - It is generated at run time from the catalogue (`__getattr__`), plus `.pyi` files for completion in an editor (`python -m lapin stubs`).
  - The classes: `lapin.model.walls`, `lapin.model.of("IfcSlab")`, `lapin.model["e41"]`. These are proxies that read the fields from the catalogue.
  - An assignment (`wall.height = 3`) sends the class's edit command, where one exists (to be settled per class; at first, reading only, and `element.update`/`setParam`).
- **Units**: metres and degrees, as in the engine (decision 112); no conversion layer in stage 1.
- **Steps**: `with lapin.batch("Façades"):` gathers the commands and sends them on exit as one `edit.batch` (one undo step, the results on exit). Later `with lapin.step("Façades"):`, a step opened on the server (§8), in which each command returns its result at once.
- **Events**: `lapin.on("diff", f)`, `lapin.on("selection", f)`. Diffs already come on every connection. The selection belongs to the interface, so the interface announces it to the host (a local event, like `presence`), and `lapin.selection()` reads it.
- **The argument catalogue**: for layer 2 to work in an installed application too, the catalogue (commands, the arguments each handler reads, queries) is generated **at build time** from the sources and shipped with the engine. The `argsFromSource` code moves into a shared script (`scripts/command-catalogue.mjs`), and `session.info` sends the catalogue (or a `session.catalogue` query does). A side gain: the MCP bridge gets the arguments in the installed application too. In the long term, arguments could be declared in C# at `r.Command(...)` (large: every module), but not now.
- **The package name**: `lapin` is already taken on PyPI (checked 12.11.2026); `lapin-bim` and `lapinbim` are free. Recommendation: the distribution and the import are `lapinbim` (like the domain lapinbim.com), and the console binds the name `lapin` (`import lapinbim as lapin`). The guide's examples use `lapin.`.
- **Python versions**: 3.11 and newer (3.10 left support in October 2026); tested on 3.11–3.14 in CI.
- **Delivery**: `pip install lapinbim` and, without internet, the same file in the application (`<app>/python/lapinbim/`), which the console puts on `sys.path`; Help → Python → "Copy the lapin package…" writes it where the user wants.

## 3. Python in Lapin, without shipping it

| Option | What it is | Licence | For |
|---|---|---|---|
| **A. The user's Python** (recommended, stage 1) | The host (`Lapin.Host`) starts the CPython it finds (`python3` on PATH, `py -3` on Windows, python.org, Homebrew) or the one chosen in Preferences, as a child process; it connects back on the same WebSocket | the user's; nothing shipped | desktop; any package of theirs (ifcopenshell, pandas) works in the console |
| **B. "Python for Lapin", an optional download** | python.org's *embeddable package*, unpacked into `<user-dir>/python/` on request | PSF, permissive | **Windows only**: python.org has no embeddable package for macOS/Linux; there, *python-build-standalone* would be needed, whose distributions include libraries with their own licences (to be checked one by one) |
| **C. Pyodide in the browser** | CPython in WebAssembly, loaded into a worker on request, talking to the engine's worker through `postMessage` (the same protocol, another transport) | MPL-2.0 (weak copyleft, per file) | the web edition |

**Recommendation:**
- **A in stage 1**, with no dependency at all: Lapin neither ships nor downloads Python. If it finds none, the panel says where to get it from python.org.
- **B** is not a dependency of Lapin's code, but it is a third-party component distributed by us. Do it only if Windows users without Python ask, and only downloaded from python.org on a click, not in the installer.
- **C** goes into no build: the "permissive production dependencies" rule excludes it from the package. Loaded on request from Pyodide's CDN, unmodified, it would not be "shipped", but that is the user's decision (§11). Until then, the web edition shows the panel with "Python works in the desktop application".

- **The process**: one per session, started at the first Run with `python -I -u -m lapinbim.kernel` (isolated mode, without the current folder on `sys.path`). It receives the code from the host and puts its output out as JSON lines (stdout, stderr, result, an error with its traceback). It stays alive between runs (the console's variables are kept) and is restarted with "Restart".
- **The host**: the methods `script.run {code, name?}`, `script.stop`, `script.restart`, `script.status` and the event `script.output`, in `EngineHub` (like `cancel.long`). They belong to the host and are refused to guests. They are not model commands (they do not change the model; the script changes the model through ordinary commands).

## 4. The plug-in's shape: the open module `scripting`

- **In `products/lapin.json`** (the free edition), with an engine half and an interface half, only through the modules' hooks (nothing in the platform knows about Python, apart from the host methods of §3 and the catalogue of §2, which serve MCP too).
- **The engine** (`engine/src/Modules/Scripting/Lapin.Scripting`):
  - the class `LapinScript` (name, description, code, language `python`, hash);
  - the commands `scripting.save`, `scripting.rename`, `scripting.delete`, each one undo step. So the project's scripts take part in undo, versions (`docs/versions.md`), shared sessions and the absent-module rule (decision 484) with nothing new;
  - in the package, each script's code is also written as text, `scripts/<name>.py` (a part of the module), so it can be read without Lapin. To be checked that the package takes a module's text parts; otherwise the `code` column in Arrow.
- **The interface** (`ui/src/modules/scripting/`):
  - the **Script** panel: the console (an input line with history, the output, the recorder, §5);
  - the editor: Run, Stop, Restart, the output below, errors with their line;
  - the library: **Mine** (`<user-dir>/scripts/*.py`, ordinary files) and **The project's** (`LapinScript`);
  - "Open in external editor": the file is watched, so an editor such as VS Code can be used.
- **The developer switch**: Preferences → **Developer extras** (off by default, as in Blender). It shows the Script panel, the recorder, "Copy as Python" in a tool's context menu and the command's name in tooltips. Off, nothing in the interface changes.
- **The code editor**: a text area with indentation and a colouring written by us (no dependency) in stage 1; CodeMirror 6 (MIT) is an alternative, loaded only by the panel (§11).

## 5. The recorder: the interface's steps as code

- **Where**: in the interface, at `EngineClient.execute`, not in the journal: the journal holds entities, not commands. It is the only place every tool goes through, overridden or not, and in the browser too. Queries are not recorded.
- **What it writes**: one Python line per command, from the same catalogue: `lapin.architecture.add_wall(x1=0.0, y1=0.0, x2=5.0, y2=0.0, height=2.8)`. A tool's `edit.batch` becomes a `with lapin.batch("…"):` block. Undo/redo appear as `lapin.undo()`.
- **Ids**: a command on existing elements writes the id (`lapin.model["e41"]`). The ids a command returns are bound to variables (`w1 = lapin.architecture.add_wall(...)`, then `w1.id` in the next steps), so a copied script works on a new project. The current selection as an argument (`lapin.selection()`) is an option for stage 3.
- **Shown** live in the console (a "Recorder" tab with Start/Stop), with "Copy as script" and "Save as script". A step made by a script is not recorded a second time.

## 6. Tools and dialogs from scripts

- **Tools**: `lapin.ui.tool("Perimeter walls", run=f, icon="square")` registers a tool that appears on the **My tools** tab (decision 479), in a "Scripts" group; a click runs the function in the Python process. A tool that asks for points (`gesture="points"`) gets the points picked in the plan, like an ordinary tool's argument.
- **Panels, menus, palettes** (decision 528, at the user's request of 13.11.2026), all declarative JSON drawn by the window (docs/python.md):
  - `lapin.ui.panel("Walls")` puts the script's tools in a group of their own in My tools, instead of "Scripts"; `size="small"` makes them small;
  - `lapin.ui.menu(…)` is a button with a list of tools (each stays a tool in its own right);
  - `lapin.ui.palette(…)` is a small floating panel of the window (titles, texts, the fields of `ask`, buttons), opened by its button. A button is a Run and one undo step, and receives the palette's values.
- **Simple dialogs**: `lapin.ui.ask("Walls", fields={"height": 2.8, "type": lapin.ui.choice([...])})` gives an interface dialog made of fields (a number with a unit, text, a choice, a checkbox, an element picked from the model). It returns a dict with the values. Nothing HTML or JS comes from the script: the interface draws its own fields.
- **A start for plug-ins and the store**: a script with a header (`# lapin: name, version, requires`) is the first plug-in; later, signed plug-ins and a store. The order from the discussion about the API: **A** the Python client (this plan) → **B** interface plug-ins in JS (on the `ui/src/platform/module.ts` contract) → **C** C# modules (the engine's contract, `module.json`). All three speak the same command catalogue.

## 7. Safety

- **On the desktop, a script can do anything** the user can (a CPython process): Lapin does not sandbox it, and says so in the panel.
- **The project's scripts never run on opening**, and never without asking. The first run of a script that came with the project (or changed since) asks for trust: "This script came with project X. It can do anything the user can do on this computer." — Run once / Trust / Cancel. Trust is kept on the computer, by the code's hash (`<user-dir>/scripting-trust.json`). In stages 1–3 there are no "on open" or "on save" events that start scripts by themselves.
- **Guests of a shared session** do **not** run scripts on the host: `script.*` belong to the host and are refused to guests, and the Python client connects only to the engine on its own computer (127.0.0.1). The host's script, while sharing, edits as the host (its author, its undo); guests see the diffs as usual. The `scripting.*` commands are not in `AllowedRemote` (a guest cannot put a script into the host's project).
- **Signed plug-ins** later, with the store.

## 8. Speed

- **One call** on loopback takes ~0.2–1 ms plus the command. A script that places 5,000 elements one by one makes 5,000 undo steps and takes a few seconds. Hence `lapin.batch()` (chunks of a few hundred commands, so as not to keep the session busy for long) and, for a Run from the panel, **one run = one undo step**, named after the script (with `step` opened on the server, below).
- **The step opened on the server** (stage 2, an engine change): `doc.begin {label}` / `doc.commit` / `doc.rollback`, tied to the connection.
  - Commands go into the same transaction, and the diff leaves at commit.
  - A closed connection or a Stop rolls back.
  - While a step is open, the interface's edits are refused ("A script is editing the model"); reads work.
  - Plus an optional `label` on `edit.batch`.
- **A lighter read of the model** (decided 12.11.2026, to be done later): `lapin.model` reads `doc.snapshot` today, which brings the rendering too; a query giving only the entities and relationships (no meshes, no rendering) would make the first read of a large model much faster.
- **Bulk reading**: `doc.snapshot` (JSON, no meshes) is enough up to tens of thousands of elements; meshes come as `LPF1` arrays, read without copying. Arrow cannot be read with the standard library: JSON in stage 1; a columnar query (`model.columns {class, fields}` → raw arrays) if the need is measured.
- **The interface does not block**: the script is another process, and the engine makes one call at a time, so the interface waits at most for one command (or one chunk of a batch).
- **Stop**: first `KeyboardInterrupt` (SIGINT / CTRL_BREAK); after 2 s the process is killed. A batch in progress either finishes or is not done at all (atomic); an open `step` rolls back.
- **Undo**: what the script did is undone with Ctrl+Z like any step.

## 9. Stages

| Stage | What it brings | Effort |
|---|---|---|
| **1. The client** | the `lapinbim` package: WebSocket, `LPF1`, `execute`/`query`/`undo`/`save`, `batch`, `on("diff")`; the argument catalogue at build time (shared with MCP); the dynamic typed layer + `.pyi`; examples; tests | ~1 week |
| **2. The console** | the `scripting` module in the `lapin` edition; the user's Python found/chosen; the host's process, `script.*`; the Script panel (console, editor, Run/Stop/Restart), the Mine library; Developer extras; `doc.begin/commit` and one run = one step | ~1.5–2 weeks |
| **3. The recorder and the project** | the recorder (live, Copy/Save as script), `LapinScript` in the project with `scripts/*.py`, trust, `lapin.selection()`, the element proxies | ~1 week |
| **4. Tools and dialogs** | `lapin.ui.tool` on My tools, `lapin.ui.ask`, the plug-in header | ~1 week |
| **Later** | a lighter read of the model (entities without meshes, §8), Pyodide in the web edition (if decided), "Python for Lapin" on Windows, signed plug-ins, the store, JS (B), C# (C) | — |

- **Tests**:
  - the package has its own tests with the standard library's `unittest` (no pytest), run in CI on Python 3.11 and 3.14 against an engine started on a free port (like `scripts/lapin-mcp-check.mjs`): connecting, a wall, batch = one step, undo, binary frames, a command error, Stop in the middle of a script;
  - xUnit for `script.*`, `doc.begin/commit/rollback` (including a lost connection) and refusing guests;
  - vitest for the recorder's command → Python line transformation;
  - `pnpm test` runs them all (with Python missing, the Python tests skip, with a message).
- **The guide**:
  - `docs/user/scripting.md` and `docs/user/ro/scripting.md` (the console, the recorder, trust, the first script);
  - a `scripting` scene in `scripts/user-docs/scenes.mjs` (the build machine's Python);
  - a page for developers, `docs/python.md` (the API, like `docs/mcp.md`).

  New strings go in all the languages of `ui/src/i18n`.
- **The MCP parallel**: the MCP bridge and the Python client read the same catalogue (§2) and the same `host.json`/protocol; what is added to a command appears in both. An AI agent can write a script through MCP, and the console can run it.

## 10. The Lite-STEP edition

The Lite-STEP proposal (Ginnung Technologies, https://www.lite-step.org) is a declarative Python DSL.
- **Its licence** was not stated on the site when the plan was written. Since 13.11.2026 it is known: **MIT** (the public repo `ginnung-tech/lite-step`, Copyright (c) 2026 David Arbirk).
- **It depends at run time** on ifcopenshell (LGPL-3.0), manifold3d, numpy, shapely, pydantic and libcst, so it stays an **external process**: Lapin depends neither on it nor on ifcopenshell.
- **The integration** approved by the user (13.11.2026) goes through the open Python API, in four steps: `docs/plan-external-geometry.md`.
- **Its code was read** so that the API would be easy for it to use (whole millimetres, N×3 meshes, canonical names → stable GlobalIds). Small pieces could be taken now (MIT), crediting the author, but Lapin's code stays its own.
- **An edition** (decision 344, `docs/overrides.md`) could put a declarative layer in its style over `lapinbim` (a model described in `source/model.py`, kept as a foreign part, decision 345), with overrides of commands and tools. The licence (MIT) allows it now; the decision is still to be taken.

The `lapinbim` package stays imperative and Lapin's own.

## 11. Questions for the user

The answers (12.11.2026, "yes to all"):
1. `lapinbim`, bound as `lapin` in the console;
2. on PyPI from stage 1 (the version = Lapin's, `0.1.<commit count>`; the user publishes it);
3. only the user's Python;
4. Python on the desktop only (no Pyodide);
5. minimum 3.11;
6. the Script panel behind "Developer extras", off by default (stage 2);
7. CodeMirror 6 (stage 2);
8. one run = one undo step through `doc.begin/commit` (stage 2);
9. `LapinScript` in the model plus `scripts/*.py` (stage 3);
10. guests do not run scripts;
11. tools and dialogs from scripts in the free edition (stage 4);
12. the catalogue generated at build time from the sources, now.

The questions as they were asked:
1. The package name: is `lapinbim` (PyPI and import, `lapin` in the console) good? (`lapin` is taken on PyPI.)
2. Do we publish to PyPI from stage 1, or only the file shipped with the application at first?
3. Only the user's Python (A) in stage 1 — agreed? Do you ever want "Python for Lapin" on Windows (B)?
4. The web edition: is Pyodide (MPL-2.0, loaded from its CDN on request, not shipped) acceptable under the permissive-licence rule, or does Python stay desktop-only?
5. The minimum version: Python 3.11?
6. "Developer extras" off by default (the panel hidden), or the Script panel visible to everyone?
7. The code editor: written by us (no dependency) or CodeMirror 6 (MIT, loaded only by the panel)?
8. One run from the panel = one undo step (it needs `doc.begin/commit` in the engine, and the interface does not edit while the script runs) — agreed?
9. The project's scripts: in the model (`LapinScript`, with undo and versions) plus `scripts/*.py` as text in `.lapin` — or only files in `scripts/`, without undo?
10. Guests of shared sessions do not run scripts on the host — confirmed?
11. Do tools and dialogs from scripts (stage 4) go into the free edition, like the rest?
12. The argument catalogue generated at build time from the sources (recommended now) or the arguments declared in C# at each command (cleaner, a lot of work)?
