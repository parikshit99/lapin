# Lapin on the BubbleBIM Pro engine (plan, 23.10.2026)

Lapin is being rewritten without ifc-lite. The engine becomes BubbleBIM Pro (C#, .NET 10), and what we built in Lapin is kept as functionality. The ifc-lite fork (`Ciuntuc-BIM-Studio/lapin`) was backed up in full before anything was deleted.

## Decisions

1. **Desktop first.** A Tauri application, with the C# engine started locally, as in BubbleBIM-Pro. The web version comes later.
2. **A separate repo.** Lapin has its own repo and uses the BubbleBIM Pro engine as a dependency. It is not a product profile inside BubbleBIM-Pro.
3. **MIT licence**, MIT only.
4. **The `.lapin` format is kept.** Format 3 is a zip of content-addressed parts, with the model in Arrow tables (`docs/format-lapin.md`). Reading old projects (format 2) comes at M6.
5. **The Unity / Unreal model.** The application works locally. The cloud brings accounts, updates and libraries (types, materials, joinery, hatches). Later it also brings synchronisation, versions and teamwork. Modelling is not done on the web.
6. **Editions are combinations of modules** (24.10.2026). Modules are not necessarily "premium": the aim is that a client gets, cleanly in production, exactly the combination they need. Examples: only architecture and structure, only what civil work needs, only what MEP needs.
   - A product (`products/<id>.json`) names the modules. Each module declares its dependencies (`requires` in `module.json`), and those come automatically.
   - A module the product does not have does not exist in the package, neither in the engine nor in the interface.
7. **Reused code** (24.10.2026). Code from BubbleBIM Pro and from Atrium, the user's C# projects, may be used. The platform patterns (modules loaded from folders, the session, undo, the protocol, packaging per product) come from BubbleBIM Pro, adapted to the IFC model.

## Backup (done)

In `~/Documents/GitHub/_backups/lapin-2026-10-23/`:

- `lapin.git`: a full mirror from GitHub (8 branches, one tag);
- `lapin-all.bundle`: the same repo in a single file;
- `lapin-local-all.bundle`: the local repo, including the branches that are not on GitHub (`client/ifcsheet`, `plugins-bubbles`);
- `issues.json` (6 issues), `pulls.json` (none);
- `release-desktop-test/`: the test builds for Windows and macOS.

Restore: `git clone lapin-all.bundle lapin`.

## The licence

- **Lapin is MIT, MIT only** (decided 23.10.2026). ifc-lite's licence is no longer a concern of the project.
- **The practical consequence:** no code from the fork goes into lapin-pro. The logic is rewritten in C#, as engine modules, or in new TypeScript, for the interface. The old files remain the specification and the source of the tests.
- **Dependencies** are permissive only, checked by `pnpm check:licenses`. manifold is Apache-2.0.
- **The BubbleBIM Pro engine** is a separate project, with its own terms.
- **The licence audit** (24.10.2026, `decisions-pending.md` #105): the only area derived from ifc-lite, the wall joins, was rewritten from scratch. The old version was also removed from the git history (#109, 24.10.2026).

## The architecture (revised 23.10.2026)

**Lapin remains a classic modeller, on the IFC schema.** Its model is the IFC entities and relationships (IFC4X3):
- `IfcWall` with `IfcWallType` and `IfcMaterialLayerSetUsage`;
- `IfcOpeningElement` related by `IfcRelVoidsElement` and `IfcRelFillsElement`;
- `IfcRelConnectsPathElements` for joins;
- the property sets and quantities.

Today's Lapin works this way too. From BubbleBIM **only the geometry engine** is taken, as a library, not its graph model (walls that connect axes).

```
lapin-pro (MIT)
├─ engine/                        Lapin's engine (C#, .NET 10), engine/Lapin.slnx
│  ├─ src/Model/Lapin.Model        the IFC4X3 entities and relationships, transactions (one undo step each),
│  │                               GlobalIds, the layer rules (the core on the axis)
│  ├─ src/Platform/Lapin.Platform  the module contract and loading, the "core" module (spatial structure,
│  │                               materials, the shared commands), the session, the diffs, geometry
│  ├─ src/Platform/Lapin.Package   the .lapin package, format 3 (zip, SHA-256, Arrow tables)
│  ├─ src/Modules/<Group>/Lapin.<X> a module: the IFC classes it writes, its commands, its geometry (module.json)
│  ├─ src/Host/Lapin.Host          WebSocket JSON-RPC, the same protocol as BubbleBIM (doc.execute, diffs with meshes)
│  └─ tests/Lapin.Engine.Tests     xUnit, with the tests ported from the fork
├─ ui/                            Lapin's interface (React + three.js); ui/src/modules/<id> is a module's UI half
├─ products/                      the editions: the lists of modules
├─ desktop/                       the Tauri shell
   (engine/src/Geometry           the geometry kernel: Lapin.Geometry (B-rep, tessellator, skeleton) and
                                  Lapin.Booleans (manifold), taken from BubbleBIM Pro on 25.10.2026; before that,
                                  the submodule vendor/bubblebim-pro)
```

A module has an engine half (`engine/src/Modules/…`, with `module.json`), an interface half (`ui/src/modules/<id>`), or both. Engine modules are built into `engine/artifacts/modules/<id>/`, and the host loads them by product. `pnpm build:product <id>` puts only the product's modules into the application.

- **The project file** is the `.lapin` package, with the model's intent, the views, sheets and standards (`docs/format-lapin.md`). IFC is the exchange format, for import and export.
- **The protocol** with the interface stays the one from M0. Only the engine behind it changes: in M0 it was BubbleBIM's host, from now on it is Lapin's host.

### Compatibility with BubbleBIM: through IFC, with the design intent

**Today,** Bubbles does not send geometry but the intent: a `bubbles-house` JSON with the walls' axes, the openings, the outlines and the roof. Lapin builds native elements from it, with its own tools.

**The proposed improvement:** the exchange goes through an IFC file that also carries the intent.
1. **The file opens anywhere.** The elements have their IFC geometry, so the file works in any IFC program.
2. **The intent is written in IFC too:**
   - the walls' `Axis` representation;
   - `IfcMaterialLayerSetUsage` for the layers;
   - each element's parameters, in a shared property set, for example `Pset_BubbleBIM_Intent`: axes, outlines, slope, the openings' rules.
   Lapin rebuilds native elements from them, as it does today with `bubbles-house`. What has no intent (an IFC from another program) comes in as a reference.
3. **Stable GlobalIds.** A second export of the same house from BubbleBIM **updates** the elements in Lapin, by GlobalId, instead of duplicating them. What you have already changed in Lapin is kept or flagged.
4. **And the other way.** BubbleBIM can link Lapin's IFC as a reference, as it already links IFCs, or later read its intent.

Today's `bubbles-house` format thus becomes a special case of the same IFC, and the two programs stay independent in their internal model.

## The milestones (replanned 23.10.2026, a proposal)

The old numbering had two variants: the one in the M0 commit (M4 phases, M6 reporting, M7 launch) and the one in `transfer.md` (M7 MEP). This one replaces both.

- **M0 – Skeleton** (done, `d09bd60`): Tauri, the engine as a submodule, one wall seen in three.js. The interface mock: `6a5b582`.
- **M1 – Lapin's engine and the format** (done on 24.10.2026: the model, the platform, the Architecture module with layered walls and the flip, `.lapin` format 3, the host, the editions and `build:product`, binary meshes, the recovery journal, the interface connected to the engine):
  - `Lapin.Ifc.Model` (IFC4X3 entities and relationships, commands, undo) and `Lapin.Host`, on the same protocol;
  - the layered wall type: layer set, usage, the axis on the core, flip;
  - `.lapin` format 3: save, open, journal;
  - meshes sent as binary;
  - the mock's interface connected to the engine: browser, inspector, open and save.

  The format comes now, not at the end, because it fixes the shape of the model.
- **M2 – Walls:** joins, mitres, height zones, finish layers, lintels, cut priorities, foundations.
  - Done (24.10.2026): L, T and in-line joins, butt or mitre, with priority (as path elements, in the platform); height zones and finish pieces, with their quantities; lintels and the dropped ring beam; slabs; cut priorities; the Structure module (columns, beams, strip and pad foundations).
  - Also done (24.10.2026, second round): openings in slabs; steel I/H, U, T, L profiles (with the IPE, HEA, HEB, UPN, L catalogue); columns at the grid intersections.
  - Also done (25.10.2026): visual seams — a material that continues in the same plane from one element to another (the façade past the next storey) no longer has a line between them, in 3D, elevations and sections; the elements stay separate.
  - Left: joining layers by function (at M5); corners as a merge of the geometry in 3D (#108).
  - Also done: lintels joined at corners (the longest one passes, #35).
- **M3 – The door and window configurator.**
  - Done (24.10.2026): openings in walls, doors and windows with their types, the configurator (ported with its tests), the basic symbols in plan.
  - Also done: the type's drawings from the spec (the plan cut through the frame and sashes, the elevation in both conventions, with the hardware, the section), ported with their tests; the full lintel in the configurator and the prefabricated products from CSV.
  - Left: mapping onto imported models.
- **Roofs** (approved with M1–M7, done 24.10.2026): `IfcRoof` ported from BubbleBIM Pro with its tests (shapes, styles, rules per edge, rafters, gutters, downpipes, snow guards, quantities), walls that rise to the gable and stop under the eaves or under the slopes, the tool and the inspector. Also done: roof windows and dormers (ported from BubbleBIM Pro).
- **M4 – Materials and drawing standards.**
  - Done (24.10.2026): the `Lapin.Drawing` library with hatches (`.pat`, the library, the fill, the region, ported with their tests), materials per view type (on `IfcMaterial`, the appearance for IFC), the drawing standards (text styles, dimensions, grids, tags, layers, groups, read tolerantly), their commands in `core`, and the Materials and Drafting standards dialogs.
  - Also done: the materials' 3D textures (drawn by the application), Object Styles with the project's pen table.
- **M5 – 2D drawing:** the generator, the view range, pens, symbols, dimensions.
  - Done (24.10.2026): the generator on BubbleBIM's exact hidden lines (cut, seen, hidden, above, beyond), the view range, the pen table and line styles, drawing by material (hatches, joins in the same material, membranes), views as entities (plans, foundation plans, sections, elevations), section marks and levels, grids, associative aligned dimensions and texts, with their tools.
  - Also done (second round): joining collinear lines; linear dimensions, chains and driving dimensions, attached to grids and to the walls' axes too, robust to rotation; tags (TAG); line edits (Linework); Visibility/Graphics with sets and 3D; View Range; the base point and the survey point; full CAD drafting (command line, snaps, ortho, guides, layers, offset, trim, extend, fillet, hatches), on plans, sections and elevations.
  - Also done: removing stray lines (projections longer than 1.5 × the drawing's extent).
- **M6 – Phases, sheets, DXF and PDF, reading `.lapin` format 2.**
  - Done (24.10.2026): sheets (A0–A4, title block, views at their scale), PDF and DXF R2000, the joinery and lintel schedules (cards with the type's drawing, rows, cells, tables, checking, CSV, paging, placing on a sheet).
  - Also done: phases (the model ported with its tests, views and 3D by phase and filter).
  - Also done: reading format 2 (old projects are rebuilt natively).
  - Also done: XLSX, continuing on a new sheet, the schedule's settings (rows, tables, checking), old projects' sections and sheets, sheets with guides, handles and warnings.
- **M7 – The desktop launch:**
  - Done (24.10.2026): the macOS build of the Lapin edition checked (the application, the engine packaged with its modules); the ? menu with "Report an issue", the community on Skool and the website; the start page with the links.
  - Left (your decisions, in `decisions-pending.md` #79–80): signing and notarization, the DMG, Windows (manifold for Windows).
  - macOS and Windows (manifold for Windows), with signing and notarization;
  - the welcome page, "Report an issue", the Skool link;
  - then the old web application and the fork are retired.
- **New modules** (24.10.2026, second round, asked for after M7):
  - `civil`: terrain from survey points and LandXML, contour lines, breaklines, platforms with cut and fill;
  - `studio`: solids from sketches (extrude, sweep, revolve) and live booleans between elements;
  - the interface in Romanian, the MCP bridge for AI agents, move/copy/rotate/mirror for any element, Create Similar and Match Type.
- **M8 – The cloud hub:**
  - accounts;
  - the update channel per edition (Tauri's updater);
  - the `.lapinlib` libraries;
  - later, synchronisation and versions.
- **M9 – Residential MEP,** as the `mep` module, with its own edition: the network and the route, heating, water, drainage, electrical, ventilation.
- **Later:** timber and CLT, infrastructure.

At each milestone, the specification tests from `transfer.md` are ported before the code.

## Risks

- **The Pro engine is very new** (two days of commits). Lapin will ask for changes to it, so who changes it and how it is versioned must be settled.
- **manifold is compiled for macOS arm64 only.** Windows, and later Linux for the web, must be added.
- **Meshes travel as JSON.** On large models, a binary format becomes necessary.
- **Feature parity** with today's Lapin will take time. Until then, the old web application stays available as a preview.
