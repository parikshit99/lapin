# Transferring Lapin's logic from the ifc-lite fork to lapin-pro (23.10.2026)

What moves from today's Lapin (the ifc-lite fork, `Ciuntuc-BIM-Studio/lapin`, backed up in `~/Documents/GitHub/_backups/lapin-2026-10-23/`) to lapin-pro: the configurators, the layer and type logic, 2D drawing (sections, hidden lines, the view depth, what is seen above the cut plane, line types and weights), materials, standards, phases and the project format.

The paths in this document are read in the fork: `C` = `packages/create/src/in-store`, `V` = `apps/viewer/src`.

## Transfer rules

1. **Our own code (OWN).** The files created in the fork, listed in `Lapin/docs/lapin-own-files.txt`. We port them to lapin-pro and may relicense them MIT. They are rewritten in C#, as engine modules, or in new TypeScript, for the interface; the logic and the numbers are kept.
2. **ifc-lite code, modified or not (MOD/UP), under MPL-2.0.** **It is not copied.** It is rewritten from the behaviour described in this document and from today's tests, used as the specification. Where BubbleBIM Pro already has the equivalent, we start from it.
3. **Today's tests are the specification.** Each area below lists the tests that fix the behaviour. In lapin-pro they become xUnit tests for the engine or vitest tests for the interface, with the same cases and the same numbers.
4. **No ifc-lite in lapin-pro.** No dependency and no file. The licence check (`pnpm check:licenses`) stays mandatory.

## The transfer map

| Area | Source in the fork | Kind | Where it goes in lapin-pro | Specification tests |
|---|---|---|---|---|
| The type catalogue (9 types), fields, defaults, tolerant reading | `V/element-types/*`, `C/catalog-type.ts` | OWN | Engine: `Lapin.Authoring.Types` (writing the type to IFC); UI: each type's module (the Element types dialog) | `catalog-type.test.ts`, `removed-type.test.ts`, `grid-column-type.test.ts` |
| Layers: layer set, usage, core, the axis on the core, flip | `C/foundation-spec.ts` (`coreIndex`, `stemSection`), `C/layer-usage.ts`, `C/element-flip.ts`, `V/lib/wall-core-axis.ts` | OWN | Engine: `Lapin.Authoring.Layers` | `element-flip.e2e.test.ts`, `foundation.e2e.test.ts` |
| Wall joins: L, T, butt, cuts, priority | `C/wall-join*.ts` | **MOD (ifc-lite)** | Engine: rewritten from the specification below; BubbleBIM has `Joins.cs` on topology, but for a different model | `wall-join*.test.ts` (used only as the specification) |
| Mitres, the join preference, linear joins | `C/wall-join-mitre-cut.ts`, `V/lib/wall-join-style.ts`, `linear-join*` | OWN | Engine: `Lapin.Authoring.Walls` | `wall-join-mitre.test.ts`, `linear-join.e2e.test.ts` |
| Height zones, finish layers as parts, quantities | `C/wall-zones*.ts`, `C/wall-parts-frame.ts` | OWN | Engine: `Lapin.Authoring.Walls.Zones` | `wall-zones.test.ts`, `wall-zones.e2e.test.ts` |
| Lintels (rules, products, pieces, reinforcement, the dropped ring beam) | `C/wall-lintels*.ts`, `V/joinery/lintel-*` | OWN | Engine: `Lapin.Authoring.Lintels`; UI: the configurator's fields, the product table | `wall-lintels.test.ts`, `lintel-products.test.ts`, `lintel-schedule.test.ts` |
| Strip and pad foundations | `C/foundation-*.ts`, `C/strip-*.ts`, `C/pad-footing.ts` | OWN | Engine: `Lapin.Authoring.Foundations` (the Structure module) | `foundation.e2e.test.ts`, `strip-foundation-editing.test.ts`, `foundation-place.test.ts` |
| Timber frame, trusses, CLT | `C/timber-*.ts`, `C/truss*.ts`, `V/project/clt.ts` | OWN | Engine: a Timber module, later | `timber-frame*.test.ts`, `truss*.test.ts`, `clt.test.ts` |
| Roof systems (weighted straight skeleton, structure, gable walls, trimming walls) | `C/roof-*.ts`, `C/straight-skeleton.ts`, `C/wall-roof-trim.ts` | OWN | Engine: `Lapin.Authoring.Roofs`; BubbleBIM already has a `WeightedSkeleton` | `roof-system*.test.ts`, `straight-skeleton.test.ts`, `wall-roof-trim*.test.ts` |
| Cut priorities | `C/cut-priority.ts` | OWN | Engine: `Lapin.Authoring.Cuts` | `cut-priority.e2e.test.ts` |
| The door and window configurator (spec, geometry, IFC, symbols, mapping, placing) | `C/joinery-*.ts`, `V/joinery/*` | OWN | Engine: `Lapin.Authoring.Joinery`; UI: the Architecture module (the configurator) | `joinery*.test.ts`, `symbols.test.ts`, `symbol-mapping.test.ts`, `spec-file.test.ts`, `schedule-*.test.ts` |
| Materials, the IFC appearance, `.pat` hatches, textures | `V/project/materials.ts`, `material-*.ts`, `C/material-appearance.ts`, `V/drafting/hatch/*` | OWN | Engine: the materials' IFC appearance; UI: the Materials dialog, the hatch library | `material-appearance.test.ts`, `materials-views.test.ts`, `hatch/*.test.ts` |
| Drawing standards (text, dimensions, grids, tags, layers, groups) | `V/drafting/styles.ts`, `V/project/drafting-standards.ts`, `standards-file.ts` | OWN | UI: the Drafting module; engine: annotations as the view's data | `styles.test.ts`, `standards-file.test.ts` |
| Phases | `V/phases/*` | OWN | Engine: the phase on the element; UI: the Phases module | `phase-model.test.ts` |
| The drawing generator: section, projection, bands, hidden lines | `packages/drawing-2d/*` | **UP/MOD (ifc-lite)** | Engine: starting from BubbleBIM's `HiddenLines` and `Projection.cs`, after the specification below | ifc-lite's tests (only as the specification) |
| The view range (top / bottom / depth / beyond), the depth, hidden lines per view type | `V/project/view-range.ts`, `view-defaults.ts`, `view-hidden-lines.ts`, `view-plane*.ts`, `view-drawings.ts` | OWN | Engine: the view's parameters; UI: the view's properties | `view-range.test.ts`, `view-defaults.test.ts`, `view-hidden-lines.test.ts`, `view-plane.test.ts`, `view-drawings.test.ts` |
| The pen table, object styles, line types, line edits | `V/project/line-styles.ts`, `linework.ts`, `view-linework.ts` | OWN | Engine + UI: the Drafting module | `line-styles.test.ts`, `linework.test.ts` |
| View graphics (categories, hidden classes, layers, graphics sets) | `V/project/view-graphics.ts`, `graphics-sets.ts` | OWN, but resting on ifc-lite's override engine | UI + engine: rewritten without ifc-lite's `graphic-overrides` | `view-graphics.test.ts`, `graphics-sets.test.ts` |
| Drawing by material (bands, joining layers, membranes, seams) | `V/project/material-drawing.ts`, `material-resolver.ts` | OWN | Engine: `Lapin.Drawing.Materials` | `material-drawing.test.ts` |
| Opening symbols, the roof in plan, section marks, grids, associative dimensions | `V/drafting/opening-symbols.ts`, `V/project/roof-plan.ts`, `section-marks.ts`, `view-grids.ts`, `associative-dims.ts`, `drafting/annotation.ts` | OWN | Engine: `Lapin.Drawing.Symbols`; UI: the view's overlays | `opening-symbols.test.ts`, `associative-dims.test.ts` |
| Sheets, viewports, pens on the sheet, DXF R2000, PDF | `V/project/sheets.ts`, `V/components/viewer/sheets/*` | OWN | Engine: DXF and PDF (BubbleBIM has `Dxf.cs`, `Pdf.cs`); UI: the sheet editor | `sheets*.test.ts`, `dxf-r2000.test.ts`, `viewport-regions.test.ts` |
| The `.lapin` package and the project document | `V/project/types.ts`, `V/project/package/*`, `project-file*.ts` | OWN | Engine: opening and saving; the format is kept (it also reads old `.lapin`) | `package-io.test.ts`, `project-file*.test.ts`, `versions.test.ts` |
| **Residential MEP** (M9; branch `feature/residential-mep`, 54 commits, not in `lapin`) | see the MEP section below | OWN (the new files); 33 ifc-lite files touched only for wiring | Engine: `Lapin.Authoring.Mep`; UI: the `mep` module (MEP tab, dialogs, option bars) | the branch's tests: `*-mep*`, `hydronic-*`, `heating-*`, `water-*`, `drainage-*`, `ventilation-*`, `electrical-*`, `route-*` |

## Residential MEP

The branch `feature/residential-mep` (last commit `8b4835045`, 23.10.2026) is on GitHub and in the backup, and was never merged into `lapin`. It has 258 files and about 22,000 lines. The design document is `docs/architecture/residential-mep.md` on the branch, and is ported as the specification.

What it contains:

- **The shared network:** system, equipment, terminal, port, network, route, generated fittings (`apps/viewer/src/mep/*`, `C/mep-equipment-*`, `C/hydronic-connectivity.ts`).
- **The parametric route:** a 2.5D/3D polyline with stations, levels, slope, minimum bend radius, nominal sizes, a mounting plane (floor 0.2 m, ceiling 2.4 m, wall 1.1 m) and generated fittings (elbows, tees, reducers, couplings) (`C/route-*`, `C/hydronic-route-*`).
- **Heating:** boiler, radiator proposals (area × W/m², always shown as an estimate), paired flow/return routes, the suggested minimal main route, the isometric and the route's profile.
- **Water:** runs, in-line fittings (valve, meter), demand and pressure losses, a table.
- **Drainage:** stacks, branches, vents, cleanouts, outlets, with the slope kept.
- **Electrical:** board, circuits, sockets, lights, switches, cables, a table.
- **Ventilation:** fan, circular and rectangular ducts, supply grilles, branches, a table.
- **The plan:** bands per discipline, tags, unconnected ports, export to SVG, DXF, PDF, XLSX.
- **IFC:** IfcBoiler, IfcSpaceHeater, IfcPipeSegment and IfcPipeFitting, IfcValve, IfcFlowMeter, IfcSanitaryTerminal, IfcElectricDistributionBoard, IfcOutlet and the rest, with distribution ports, system relationships and `Pset_LapinMEP*`.

All the new files were created by us (OWN). The 33 ifc-lite files changed on the branch (the ribbon, the plan, snapping, commands, i18n) are only wiring points and are not ported. In lapin-pro the module contract and the engine's protocol replace them.

## The specification of the ifc-lite parts (for rewriting)

### Wall joins (`wall-join*.ts`)

- **The wall:** `{start, end, thickness, alignment center|left|right, offset (+ to the left), startCut?, endCut?}`. An end cut is `{left, right}`: how far each face passes the axis's end.
- **The join:** `{kind L|T|butt, style butt|mitre, point, relating a|b, a, b: {connection ATSTART|ATEND|ATPATH, runsThrough}}`.
- **Options:** the tolerance, by default 2 × the larger thickness; the priority, by default the thicker wall, and `a` on a tie.
- **L:** the priority wall runs to the outer face, the other stops at its inner face.
- **T:** the wall run into stays unchanged, the one that ends stops at the near face.
- **Butt:** collinear walls stop square at the shared point.
- **Joined axis ends** are brought exactly to the intersection point.
- **Refusals:** walls that cross (they must be split first), parallel walls that are not collinear, or walls that do not meet within the tolerance.
- **The wall's body:** a square cut keeps the rectangular profile, an oblique one gives a four-point profile. No booleans are used.
- **The mitre** (L only): the cut runs along the diagonal from the inner corner to the outer one, and neither wall passes through the other. It is written in `IfcRelConnectsPathElements` with Description `Mitre`.

### The drawing generator (`packages/drawing-2d`)

- **The cut:** each triangle cut by the plane gives a segment. The segments are grouped into closed polygons per element and per material (key colour + material), so each layer becomes its own polygon.
- **Projection by bands:** the signed depth is `d = ±(c − the plane's position)`.
  - **Visible**, between `−below` and `0`: a thin continuous line.
  - **Overhead**, between `0` and `above`: a dashed line.
  - An element crossing the plane is drawn continuous.
  - What falls outside both bands is not drawn.
  - Openings (the `IfcOpeningElement` family) are never projected.
- **The default bands:**
  - a plan restricted to its storey: `below` = the cut − the storey's level, `above` = the next storey's level − the cut;
  - sections and elevations: `below` = 0.5 × the model's extent, `above` = 1 mm;
  - an explicit depth: `below` = the depth, with exact clipping of the bands; depth 0 means the cut only.
- **Hidden lines:** a depth raster with 1024 px on the long side, 10 samples per line, bias 0.001. Only visible lines can become hidden.
  - The recommended rewrite: BubbleBIM's exact vector removal (`HiddenLines.Draw`), which is more precise.
- **Stray lines:** projection lines longer than 1.5 × max(the cut area's diagonal, the projected model's diagonal) are removed.
- **Joining collinear lines** at the end.
- **Weights on screen (mm):**

  | Line | Weight | Colour | Dash |
  |---|---|---|---|
  | cut outline | 0.5 | — | — |
  | projection | 0.25 | — | — |
  | silhouette | 0.35 | — | — |
  | edge | 0.18 | — | — |
  | annotation | 0.13 | — | — |
  | hidden | 0.18 | `#666` | [2, 1] |
  | overhead | 0.7 × normal | `#888` | [2, 1] |
  | beyond | 0.13 | `#9ca3af` | — |

- **Graphic overrides:**
  - weights from the ISO 128 series, chosen per category: 0.13 / 0.18 / 0.25 / 0.35 / 0.5 / 0.7 / 1.0 mm (decided 24.10.2026); the chosen weight is the cut lines', and seen lines take the pen two steps lower (0.5 → 0.25, 1.0 → 0.5); the old names heavy, medium, light, hairline read as 0.7 / 0.5 / 0.25 / 0.13;
  - dashes: dashed [2, 1], dotted [0.5, 0.5], dashdot [3, 1, 0.5, 1], center [6, 1, 1, 1];
  - the rules apply in priority order;
  - presets: Default (the materials' colours), Architectural, Fire safety, Structural, MEP, Monochrome.

### Lapin's view rules (OWN, to be ported as they are)

- **The plan:** the default cut is at 1.20 m.
  - The foundation plan: the cut at −0.50 m, the depth 2.50 m.
  - Storeys less than 0.25 m apart are merged.
- **The view range** (as in Revit): Top, Bottom and Depth, each tied to its own storey, the one above, the one below or unlimited (10,000 m), plus an offset.
  - The rule: Top ≥ the cut ≥ Bottom ≥ Depth.
  - The primary band: from the cut to Bottom, a continuous line.
  - Overhead: up to Top, a dashed line (the beams and roof edges above).
  - Beyond: from Bottom to Depth, grey, 0.13 mm.
- **Hidden lines:** shown by default in plan and section, hidden in elevation. The order of decision: the view's setting, then the project's, then the default.
- **Elevations:** the view plane stands 5 cm inside the model's bounds and looks inwards.
- **Pens:** 16 pens on ISO weights, from 0.1 to 2 mm.
  - At the scales 1:20, 1:50, 1:100 and 1:200, the pen index shifts by +2, +1, 0 and −1; for example, pen 5 at 1:100 means 0.35 mm.
  - Dashes on paper (mm): dashed [3, 1.5], hidden [1.5, 1], center [6, 1.5, 1, 1.5], dotted [0.3, 1].
  - The order of decision: the element, the category in the view, the old style per category, the project's object style.
- **Materials in section:**
  - the hatch and the fill come, in order, from the row itself, from the IFC appearance, then from the material's name;
  - a layer under 2 mm is a membrane, drawn as a dashed line;
  - bands of the same material are merged, with a 2 mm tolerance and an overlap of at least 50%.
- **Door and window symbols in plan:**
  - The door: the leaf open at 90°, with its arc, of thickness clamp(depth × 0.4, 0.02, 0.05). Over 1.4 m it has two leaves.
  - The window: the frame, plus the double glazing at ±min(0.012, depth / 3).
- **Sheets:** formats A0–A4, scales 1–1000, a 10 mm margin, a 180 × 50 mm title block.
  - Pens on the sheet: cut 0.5, seen 0.25, hidden 0.18, hatch 0.13 mm.
  - The DXF export is R2000, at 1:1 in mm.

## What BubbleBIM Pro lacks and we write

- The view range with Top, Bottom and Depth and the "beyond" band.
- The pen table, object styles and per-element overrides.
- Joining material layers, and membranes.
- Importing `.pat` hatches.
- Graphics presets and phases.
- Door and window symbols.
- Line edits.

BubbleBIM already has, as a starting point:
- vector hidden-line removal and the section cut (`Projection.cs`);
- line types (cut, visible, thin, hidden, axis, section);
- PDF pens, A4–A0 sheets, DXF and five hatches.

## The order of the transfer

The milestone numbers are those in `plan.md` (replanned 23.10.2026).

1. **M1 – the IFC model and the layered types:** layer set, usage, the axis on the core, flip, writing and rewriting types, plus the `.lapin` format 3 (`format-lapin.md`).
2. **M2 – walls:** joins (rewritten), mitres, height zones, finish layers, lintels, cut priorities, foundations.
3. **M3 – the door and window configurator:** specification, geometry, IFC, symbols and mapping.
4. **M4 – materials and drawing standards:** IFC appearance, `.pat` hatches, text styles, dimensions, grids, tags, layers.
5. **M5 – 2D drawing:**
   - the generator, on BubbleBIM's `HiddenLines`, after the specification above;
   - the view range, pens, drawing by material, symbols, the roof in plan, section marks, grids, dimensions.
6. **M6 – phases, sheets, DXF and PDF export, reading `.lapin` format 2** (old projects).
7. **M7 – the desktop launch; M8 – the cloud hub** (no transfer from the fork).
8. **M9 – residential MEP,** as the `mep` module, with its own edition: the network and the parametric route, then heating (the full vertical slice), water, drainage, electrical and ventilation. It needs walls, openings, plans and sheets, so it comes after M5–M6.
9. **Later:** roofs, timber and CLT.

At each step, the specification tests listed in the map are ported before the code.
