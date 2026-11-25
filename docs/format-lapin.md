# The `.lapin` format in lapin-pro: Arrow / Parquet or a graph database (analysis, 23.10.2026)

**Decided:** the `.lapin` format is kept.
**Chosen (24.10.2026):** the recommendation below. What is already done is at the end, under "State".

## What the format must be able to do

1. **Hold the intent, not the result:** parameters, types, IFC relationships, stable GlobalIds. Geometry is derived. The principles of `Lapin/docs/format-lapin-nativ.md` (14.10) still hold.
2. **Open and save fast on large models.** A save rewrites only what changed, and recovery after a crash comes from the journal.
3. **Survive editions.** A project opened in an edition without a module keeps that module's elements, untouched, like rule 4 in BubbleBIM Pro.
4. **Read old projects:** `.lapin` v2 (IFC + `overlay.json` + `native/*.arrow`).
5. **Fit the Unity / Unreal model:** a desktop application, with a cloud for accounts, updates and libraries. Later also synchronisation, versions and teamwork.

## What Unity and Unreal do

| | The source (saved) | The derived data (regenerated) | Cloud |
|---|---|---|---|
| Unity | text assets (YAML) + `.meta` with a stable GUID | `Library/` (a cache, can be deleted) | Asset Store: asset packages; Unity Version Control |
| Unreal | `.uasset` / `.umap`; *One File Per Actor* (one file per object, for merging and sync) | Derived Data Cache, local or shared | Fab / Marketplace: packages; plug-ins as separate modules |

The lessons for Lapin:
- **a stable identity** per object (for us, the GlobalId);
- **the source separate from the cache;**
- **small parts** that can be synchronised and compared;
- **libraries are packages in the same format,** not another system.

## The options

### A. A graph database (Neo4j, Memgraph, Kùzu…) as the project file

- **For:**
  - IFC is a graph (entities + objectified relationships), and propagation (joins, openings, hosts) is a graph traversal.
- **Against:**
  - **There is no embeddable, MIT, maintained one for .NET.**
    - Neo4j is a server on the JVM, GPLv3 / commercial.
    - Memgraph is a server.
    - Kùzu (embedded, MIT) was archived on 10.10.2025, after its acquisition by Apple, and no longer gets fixes.
  - A database file is opaque and tied to the engine's version. It cannot be emailed with confidence, compared, synchronised in parts or repaired by hand.
  - We have the graph **in memory** anyway: `Lapin.Ifc.Model` in C#, like `GraphStore` in BubbleBIM. Dependency queries happen there, in microseconds, without a query language.
- **Where it would make sense:** in the cloud, later, for queries across many projects (portfolio, libraries). That is a server decision, not a format one.

### B. Arrow IPC / Parquet everywhere

- **For:**
  - Arrow IPC is read without decoding, and it is already the decision of 14.10 for the working tables.
  - The .NET library (`Apache.Arrow`) is Apache-2.0.
  - It is the same good format for the meshes sent to the interface (the "JSON meshes" risk in `plan.md`).
- **Against:**
  - Tables are immutable, so a small edit rewrites the table.
  - Nested parameters ended up in JSON columns.
  - Parquet is compact but must be decoded on opening, so it is good for analysis and transfer, not for working.

### C. SQLite as an application file

- **For:**
  - transactional, incremental saving;
  - crash resistance (WAL);
  - partial loading;
  - a single file.
- **Against:**
  - it cannot be content-addressed, so cloud sync has to be built separately;
  - it does not keep the v2 package's shape (zip + manifest);
  - libraries and modules would have a different format from the project.
- **It stays the fallback**, if saving in chunks proves too slow.

## The recommendation: a graph in memory, a package of content-addressed parts, Arrow for tables, Parquet for analysis

```
.lapin  (zip, format 3; also reads format 2)
manifest.json                      the version of the format, the schema, the kernel; each part with its SHA-256 and size;
                                   the modules that wrote each part
project/…                          document, views, sheets, drawings, standards, catalogues: JSON, as in format 2
model/<module>/<class>/<storey>.arrow the intent: one row per element (GlobalId, type, storey, parameters typed as Arrow struct/list),
                                   split by module × class × storey
model/types.arrow, materials.arrow, relations.arrow, properties.arrow
model/opaque/<GlobalId>.ifc        IFC geometry kept exactly (elements imported without intent), written back identically
journal/ops.jsonl                  the operations since the last compaction: recovery now, teamwork later
derived/<hash kernel+parameters>/  mesh cache (binary, Arrow), can be deleted
versions/index.json                the project's versions: each one a manifest of its own, with number, name, note, moment and revisions (docs/versions.md)
versions/blobs/<sha256>            a part named by a version that the live project no longer has, stored once
issues/index.json                  the issues and their files, readable without Lapin (the source is the LapinIssue record in the model)
issues/blobs/<sha256>              a file made by an issue (PDF, transmittal, CSV, manifest, DXF, IFC on request), stored once
analytics/*.parquet                tables for BI, only for versions or on export
scripts/<name>.py                  the project's scripts as text (the scripting module, decision 489; Lapin reads the LapinScript entity), the module's parts in the manifest
<anything else>                    other tools' entries (dag/, source/…, a file of theirs in scripts/), kept byte for byte ("Foreign parts", below)
```

- **The source of truth is the graph in the engine.**
  - The file is only its shape on disk.
  - A save rewrites only the dirty chunks (module × class × storey). Between saves, the journal holds the edits.
- **A part's owner is a module.**
  - A part of a module missing from the edition is copied back untouched.
  - The module's elements are shown from `derived/`.
- **Content addressing** (SHA-256, already in the v2 manifest) gives for free:
  - **sync:** only the parts with a new hash are uploaded;
  - **versions:** a version is a manifest, and unchanged parts are shared;
  - **the diff** per element, by GlobalId.
- **The same format for the cloud:**
  - **the library** (`.lapinlib`) is a package with only `types`, `materials`, the joinery catalogue, hatches and standards;
  - **the edition** is a package with only its modules (engine + UI); a project opened in an edition without a module keeps that module's part untouched.
- **What changes from 14.10:**
  - the store is no longer in TypeScript but in C#, in the engine;
  - parameters as typed Arrow structs replace the JSON columns;
  - the journal holds operations, not only the last state, because teamwork needs them.

## Format 2 (old projects)

On opening, the model is read from the compacted IFC, and `overlay.json` is applied over it. Where they exist, the `native/*.arrow` tables give the parameters directly. The project is then saved in format 3.
Specification tests: `package-io.test.ts`, `project-file*.test.ts`, `versions.test.ts`, `native-tables.test.ts`.

## State (24.10.2026)

**Done**, in `engine/src/Platform/Lapin.Package`, with tests in `PackageTests`:
- the format 3 zip, with `manifest.json` (format, version, product, the modules with their schema version, each part with its SHA-256 and size, class → the module that wrote it);
- `project/document.json`;
- `model/<module>/<class>/<storey>.arrow`: one row per entity, with the attributes in typed columns (Float64, Utf8, Boolean, list of Float64) and the rest in JSON, marked in the field's metadata;
- `model/relations.arrow`;
- the creation order, kept on each row;
- saving through a temporary file, then replacing the target;
- refusing a damaged part (wrong hash) and a project from a newer Lapin;
- an edition without a module keeps its elements and writes them back untouched.

**The recovery journal is done too** (24.10.2026), but outside the package: `<app data>/recovery/*.jsonl`, with every transaction, undo and redo, written to disk at once (`Lapin.Host/Recovery.cs`, tests in `RecoveryTests`).

**Versions are done too** (26.10.2026, `docs/versions.md`): `versions/index.json` holds each version as a manifest; a save writes only the blobs of the parts the live project no longer has (content addressing, as the analysis foresaw); restoring in one undo step, comparing element by element and part by part, exporting as a self-contained package (`PackageVersions`, `ModelCompare`, `VersionTests`).

**Issues in the file are done too** (31.10.2026, decision 185, `docs/plan-cde.md`): `issues/blobs/<sha256>` holds the issues' files, content-addressed like the versions' blobs (the same bytes once, however many issues name them), and `issues/index.json` lists them for whoever opens the zip without Lapin. What a save writes: the hashes that the `LapinIssue` records in the model hold (`stored: true`); a file dropped (Delete stored files) or an issue deleted goes at the next save. The format stays 3: an older file, without `issues/`, opens as before, and an older Lapin ignores the folder. Blobs are read on demand from the package the project was opened from (`PackageIssues`); new ones stay in memory until saved (tests in `CdeStoreTests`).

**Foreign parts are done too** (06.11.2026, `PackageForeign`, tests in `PackageForeignTests`): the entries other tools put in the zip (`dag/manifest.json`, `source/model.py`, any folder or file at the root) are read on opening and written back at every save, the same bytes and the same zip date, after Lapin's parts.
- **Lapin's own** are the root names `manifest.json`, `project/`, `model/`, `versions/`, `issues/` and those reserved by the analysis above (`journal/`, `derived/`, `analytics/`), compared case-insensitively (a `Model/x` cannot shadow `model/` on a file system that does not tell them apart). An entry under them, or a path with `\`, an empty segment, `.` or `..`, is not kept; what is Lapin's is always rewritten.
- **In the manifest**, apart from `parts`: `"foreign": [{path, sha256, size}]`, in path order. The field is absent when there are no foreign parts, so a file without them is written exactly as before. An older Lapin reads the manifest as before (the reader ignores unknown fields, and the hashes checked are only those of `parts`) and loses the foreign parts on saving, as it did before.
- **The foreign hash is not checked on opening:** the other tool may rewrite its entries between two saves. The hash says what Lapin last wrote, so that tool can see whether the entry changed since.
- **They do not go into versions.** A version holds the project, not what other tools left beside it: the version's manifest has no `foreign`, and restoring does not touch them. Exporting a version as a separate package does not take them either.
- **They are held in memory** as they were on opening (`ProjectFiles`, on desktop and in the browser). If a tool writes to the file while it is open in Lapin, its change is lost at the next save, as with any two programs editing the same file. "Save as" carries them into the new file; a new project starts without them, and an old project (format 2) has none.

**Left:**
- the `derived/` cache (the geometry shown for missing modules);
- incremental saving, which rewrites only the dirty parts;
- `analytics/*.parquet`;
- ~~reading format 2~~ done at M6 (24.10.2026): `LegacyPackage` reads the native tables, `LegacyImport` rebuilds the project through commands;
- the `.lapinlib` libraries, at M8.
