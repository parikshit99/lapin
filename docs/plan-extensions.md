# Lapin Extensions: plan

The user's decision (12.11.2026): an extension store, **Lapin Extensions**, at the future address store.lapinbim.com.

- **Stage 0** is done (decision 499).
- **Stage 1** is built and was **published on 12.11.2026** (decision 504, §10):
  - store.lapinbim.com is live, in the container `lapin-store`, with DiscourseConnect switched on on the forum;
  - the catalogue is **signed** with the store's key (`2daeb4d5dc6eae92`, in `STORE_KEYS` from the same day);
  - the steps and the upkeep are in `deploy/store/README.md`.
- **Stage 2** is decided in §11, and **2a** (paid extensions) is built in §12 (decision 660, not published).

Extensions come in four kinds, in this order:

1. **Content packs** (no code): furniture (`.lapinlib`), project templates (`.lapin`), profiles (`.lapinprofiles`), hatches (`.pat`), title blocks (the title blocks' `.json`), price lists (`.lapinprices`), and types of walls, slabs, doors, windows, columns, beams and stairs (`.lapintypes`, the kind `types`, §9.1). **The first.**
2. **Python scripts and tools**, after stage 4 of the Python plan (`docs/plan-python-scripting.md`, decision 489).
3. **Interface plug-ins in JS**, later, on the `ui/src/platform/module.ts` contract and the override points (`docs/overrides.md`).
4. **C# modules** for partners, signed (the engine's contract, `module.json`).

## 1. The decisions (12.11.2026)

- **Stage 0, now, with no new server:** a public GitHub repo `lapin-extensions` with a JSON index built from one manifest per extension (`extensions/<id>/manifest.json`); authors add their extension by pull request; the package file sits in the author's GitHub release, with its SHA-256 in the manifest.
- **In Lapin:** File → Extensions…, with:
  - **Browse**: search, categories, details, licence, author, compatible versions;
  - **Installed**;
  - Install (download, check the SHA-256, put into the right library), Update, Uninstall.

  It works on the desktop and in the browser (in the browser, content packs only).
- **The manifest:** id, name, version (semver), author, licence (**SPDX, mandatory, shown to the user**; any licence is accepted, the permissive ones are marked as such), the range of Lapin versions, kind, the permissions asked for, contents, sha256, page, forum topic.
- **Checking:**
  - content is checked automatically in the catalogue repo's CI (a GitHub Action that validates PRs, pure Node, no dependencies): the manifest's schema and the schema of the package's data;
  - **code** (scripts, later JS and C#) is read by Claude, with the user's final approval.
- **0 % commission** in the first phase; paid extensions later, through a *merchant of record* (stage 2). Lapin's premium modules (mep-pro, urban, rail, planning-pro) would be sold there too.
- **Stage 1, later:**
  - store.lapinbim.com, with accounts through the forum's Discourse SSO (forum.lapinbim.com);
  - the author's dashboard;
  - packages signed by Lapin, with a new key separate from the updater's (§9.7);
  - download statistics (§9.9).
- **The first extensions, ours, as examples:** a furniture pack, a Romanian project template, a Python script (manifest + script; it installs, but runs only when the Python console exists, and is marked so).

## 2. The architecture (stage 0)

```
lapin-extensions (GitHub, public)          lapinbim.com                         Lapin
  extensions/<id>/manifest.json   ──CI──▶  /extensions/index.json      ◀──────  File → Extensions… (index)
  tools/validate.mjs, build-index.mjs      /extensions/files/<id>/<v>/… (copies, for the browser)
the author's release (GitHub)  ◀──────────────────────────────────────────────  the package (desktop: through the engine)
```

- **The index**: `{format: "lapin-extensions-index", version: 1, generated, extensions: [manifest…]}`. An entry with a wrong manifest is left out (and written to the console), not the whole index. The address, in order:
  1. the one written in the dialog (Catalogue…, `lapin.extensions.index`);
  2. else `LAPIN_EXTENSIONS_INDEX` given to the desktop engine (a local index in development);
  3. else `https://lapinbim.com/extensions/index.json` (**not yet published**).
- **Downloading**, on the desktop, goes through the engine (`extensions.fetch {url}` in `EngineHub`, outside the call queue, so a download does not hold up edits):
  - the Tauri webview may only reach `127.0.0.1` (CSP), and GitHub release files do not answer browsers (no CORS);
  - https only, anywhere; http only on this computer; local files by path or `file://`;
  - at most 64 MB;
  - guests of a shared session are refused.

  In the browser it uses `fetch`, and the packages come from the copies on lapinbim.com (`package.mirror`, set by `build-index.mjs --mirror` at publishing).
- **Checking in Lapin**, in order. A wrong package touches nothing (not even the installed version, on an update).
  1. The Lapin version is in the manifest's range, the kind's module is in the edition, and code is accepted on the desktop only.
  2. The size and the SHA-256 are exactly as in the manifest (WebCrypto).
  3. The file is read and checked as a file of its kind (`packs.ts`).
  4. Then the existing importer runs.
- **The installers, per kind** (`addContentInstaller`, registered by the module that owns the library, as with "Save as template"):
  - furniture → My furniture (`furnitureLibrary.import` / IndexedDB in the browser);
  - template → the project templates (`templates.import` / IndexedDB);
  - profiles → the computer's profile libraries;
  - hatches → My hatches;
  - title blocks → the computer's title blocks;
  - price lists → the computer's price libraries;
  - types → kept on the computer and brought into the open project (`types.importFrom`, §9.1);
  - **the library kinds (§3.1)**: each file is checked without the engine (`packs.ts`) and kept on the computer:
    - symbols → the computer's symbols (`keepSymbol`);
    - line types, standards, rule packs, bar shapes, parameters, families → files in IndexedDB `lapin-libraries` (`computerFileRows`);
    - mappings → the computer's matching memory, put back on uninstall.

    Each is brought into a project on request ("Add to this project"), through the library's importer: `symbols.importFrom`, `standards.importLin`, `standards.importFrom`, `rules.import`, `rebar.importShapes`, `params.importDefs`, `studio.importFamily`;
  - scripts → kept as files (IndexedDB `lapin-extensions`, which the console will read), **never run**.

  An item with the same name already in the library is replaced only if the user says so.
- **What is installed** is kept in `lapin.extensions`, with the other `lapin.*` settings, so also in the desktop's copy, `settings.json`. It holds the installed manifest and **the receipt**: what it added (the furniture's ids, the template's id, the library's name).
  - Uninstalling removes only what is on the receipt. What was already in the library (skipped on import) stays, and what was already put into projects stays.
  - Updating = download and check the new version, then uninstall the old one, then install the new one.
- **No new dependency**, no extension code run, and the engine changed only in the host (`extensions.fetch`, `extensions.source`, the `extensionPage` link for https pages).

## 3. The manifest (schema 1)

The JSON schema is in the catalogue repo (`schema/manifest.schema.json`); the rules are written twice, identically: `ui/src/platform/extensions/manifest.ts` (Lapin) and `tools/lib.mjs` (CI).

| Field | Required | What it is |
|---|---|---|
| `schema` | yes | `1` |
| `id` | yes | lower-case letters, digits, `.` and `-`, max. 64; the folder's name; a reversed domain or the GitHub name first (`ro.ciuntucbimstudio.mobilier-living`) |
| `name`, `summary` | yes | max. 80 / 200 characters |
| `description` | no | text |
| `screenshots` | no | **stage 1**: at most 6 screenshots for the store's page: files in the extension's folder (`screenshot.webp`, `screenshot-2.png`…, ≤ 2 MB) or https addresses; Lapin does not show them |
| `version` | yes | semver; it goes up at every change (checked by CI against `main`) |
| `author` | yes | `{name, github?, url?, email?, forum?}`; `github` is the account allowed to change the extension through GitHub; `forum` the name on forum.lapinbim.com (set by the store, stage 1) |
| `license` | yes | an SPDX expression (`MIT`, `CC-BY-4.0`, `Apache-2.0 OR MIT`); shown as "permissive" or "read its terms" |
| `lapin` | yes | the range of versions: `>=0.1.860`, `>=0.1.860 <0.3.0`, `^0.1.900`, `~1.2.0`, `0.1.x`, `*`, `||` |
| `kind` | yes | `furniture`, `template`, `types`, `profiles`, `hatches`, `titleBlock`, `priceList`, `script` |
| `tags`, `languages` | no | max. 12 tags; ISO 639-1 codes |
| `permissions` | no | code only: `model.read`, `model.write`, `files.read`, `files.write`, `network`, `process`; a content pack asks for nothing |
| `requires` | no | modules beyond the kind's own (a script: `scripting`) |
| `contents` | yes | `{file, items?}`: the file's name (the kind's extension) and what it contains, shown before installing |
| `package` | yes | `{url (https: the author's release or the store's address, store.lapinbim.com/packages/…), sha256, size (≤ 64 MB), mirror?, download?, signature?}`; the last three are set at publishing: the copy on lapinbim.com, the address that counts the download (store.lapinbim.com/api/download/<id>/<version>) and the store key's signature (stage 1) |
| `homepage`, `forum` | no | https |
| `indicative` | for prices | `true` is required for a price list (the prices and norms are the author's, indicative) |

The package files:
- `.lapinlib` (`lapin-furniture-library` v1, decision 476);
- `.lapin` (format 3, zip);
- `.lapintypes` (`lapin-type-pack` v1, §9.1);
- `.lapinprofiles` (`lapin-profiles` v1);
- `.pat` (the AutoCAD format);
- the title block (`lapin.titleBlock` v1, Title blocks → Export);
- `.lapinprices` (**new**: `{format: "lapin-price-list", version: 1, name, currency, indicative: true, rows}`, the rows as in the price library);
- `.py` (with the header `# lapin: name=…, version=…`).

### 3.1 The library kinds (decision 263, stage 4b)

The new kinds, one file each, with the same manifest schema:

| Kind | File | Format | Notes |
|---|---|---|---|
| `symbols` | `.lapinsymbols` | `lapin-symbols` v1 | |
| `lineTypes` | `.lin` | the AutoCAD format | |
| `standards` | `.lapinstandards` | `lapin-standards` v1 | |
| `rules` | `.lapinrules` | `lapin-rules` v1 | **always indicative**: the manifest says `"indicative": true`, as for price lists |
| `barShapes` | `.lapinbarshapes` | `lapin-bar-shapes` v1 | needs the `rebar` module |
| `parameters` | `.lapinparams` | `lapin-parameters` v1 | |
| `mapping` | `.lapinmapping` | `{app: "lapin", kind: "table" \| "layers"}` | |
| `families` | `.lapinfamily` | `lapin-family` v1 | needs the `studio` module |

- **The rules** live in three identical places: `ui/src/platform/extensions/manifest.ts` + `packs.ts`, and `lapin-extensions/tools/lib.mjs` + `schema/manifest.schema.json`.
- **Checking:** each file is checked without the engine (format, version, the required fields); the engine checks it once more when it brings it into a project.
- **Installing** does not touch the open project (unlike `types`). The "Add to this project" button in Installed and in the Catalogue ("From libraries") does that.
- **Uninstalling** deletes only what the receipt kept; what a project has already taken stays.

## 4. The stages

| Stage | What it brings | State |
|---|---|---|
| **0. The catalogue on GitHub** | the `lapin-extensions` repo (README, the schema, the manifests, `validate.mjs`, `build-index.mjs`, `make-examples.mjs`, the validation action); File → Extensions… (Browse, Installed, Install, Update, Uninstall, the catalogue's address); the installers per kind; the three examples | **done** (decision 499); to be published (§8) |
| **0.1** | types as a separate kind (`types`, a `.lapintypes` brought into the open project by `types.importFrom`, one undo step; Manage → Type packs…); non-commercial licences marked; exporting a price list as `.lapinprices` | **done** (decision 499, the answers in §9) |
| **0.2** | screenshots in the manifest (done in stage 1: `screenshots`); an example type pack in the catalogue (`make-examples.mjs`) | the example still to do |
| **1. store.lapinbim.com** | the static site generated from the catalogue (`lapin-extensions/site/`); the store's service (`deploy/store/`): accounts through DiscourseConnect (forum.lapinbim.com), the author's dashboard (submit an extension or a new version: a form + a file + screenshots → a pull request in the catalogue; the review's state; the downloads), the anonymous download counter, hosting the catalogue (lapinbim.com/extensions/); **signing** (the store's Ed25519 key, separate from the updater's: the index and each package, checked in Lapin); the legal drafts (`lapin-extensions/legal/`); reviews and ratings are left for later | **built** (decision 504, §10); to be published by the user (`deploy/store/README.md`) |
| **2. Paid extensions** | the authors', sold through their own seller (§11); kinds 3 (JS) and 4 (C#) | **2a built** (decision 660, §12), not published; kinds 3 and 4 still to do |

## 5. Safety and permissions

- **Content never runs.** A content pack is read by Lapin's existing importers, which already check every entry (the engine checks furniture, templates go through `TemplateLibrary`). No HTML, no JS, no formula evaluated.
- **Integrity**: the SHA-256 and the size from the manifest, checked in Lapin and in CI. A change of the release's file after approval makes the install fail (and the CI on `main` fail). In stage 1, Lapin's signature over the index.
- **Scripts** (kind 2):
  - on the desktop a script can do anything the user can (`plan-python-scripting.md` §7);
  - the manifest's permissions are **declared and shown**, not enforced, until the JS plug-ins. There the interface will enforce them, with an isolated `iframe` and only the commands asked for;
  - an installed script does not run on installing, nor when a project opens; running it will ask for trust, like the project's scripts;
  - scripts are not installed in the browser.
- **Guests** of a shared session cannot read anything through the host (`extensions.*` belong to the host).
- **Review**: CI checks the schema, the file and the version. An existing extension is changed only by its author (`author.github` = the PR's author) or by a maintainer. Code is read by Claude and approved by the user before merging.
- **Withdrawal**: an extension taken out of the index no longer appears. Installed ones stay, and Installed shows that it is no longer in the catalogue. Later: a list of revoked packages, read at start, which marks them.
- **The signature** (stage 1, §10): the index (`index.json.sig`) and each package (`package.signature`) are signed by the store's key.
  - Lapin checks them and, from the build that has the key, refuses an unsigned Lapin catalogue.
  - A user's own catalogue may be unsigned, with a warning.

## 6. Legal to-dos

The catalogue's operator: **Ciuntuc BIM Studio SRL** (CUI RO51725860, J2025031471009, Huși, VS).

1. **The authors' terms** (contributing to the catalogue): the author warrants that they hold the rights to the content (furniture, models, textures, images, prices) and that the licence chosen is theirs; grants the catalogue the right to list, copy (mirror) and show it; is responsible for their content; accepts withdrawal on a breach.
2. **The users' terms**: extensions are their authors', under their licences, with no warranty from Ciuntuc BIM Studio; prices and norms are indicative; scripts run at the user's own risk.
3. **Notice and takedown** (DSA): a contact address (ionut@ciuntucbimstudio.ro or a dedicated address), the procedure (receipt, provisional withdrawal, the author's answer), the deadlines; a form on the forum or an issue template.
4. **Privacy / GDPR**:
   - stage 0 collects no data: the index and the packages are static files. How long the nginx server's logs are kept is still to be settled;
   - stage 1 brings accounts through the forum's SSO, download statistics (anonymous) and reviews. It needs a privacy policy for the store: the legal basis (contract/legitimate interest), retention and rights.
5. **Stage 2**: the contract with the merchant of record, the terms of sale, withdrawal from the contract for digital content, invoicing (the MoR is the seller to the buyer), the distribution contract with the authors (the commission).
6. **Trademarks**: "Lapin" and "Lapin Extensions" — trademark registration (OSIM / EUIPO) to be checked before stage 1.

## 7. The examples (in `lapin-extensions`)

| id | kind | licence | what it is |
|---|---|---|---|
| `ro.ciuntucbimstudio.mobilier-living` | furniture | CC-BY-4.0 | six living-room types made with the configurator (chest of drawers, TV unit, bookcase, sofa, armchair, coffee table) |
| `ro.ciuntucbimstudio.sablon-romania` | template | CC0-1.0 | the storeys Foundations / Ground floor / Floor 1 / Attic and the project parameters Romanian building-permit documentation asks for (height regime, site area, POT, CUT, the category and class of importance) |
| `ro.ciuntucbimstudio.pereti-perimetrali` | script | MIT | four walls around a rectangle, one undo step, on the planned `lapinbim` API; it installs, and runs when the console exists |

Their files are made with `node tools/make-examples.mjs --lapin <lapin-pro>` (a Lapin engine started by the script), into `dist/`, with `SHA256SUMS.txt`; the manifests get the size and the SHA-256.

## 8. What is left to publish

1. Creating the public repo `Ciuntuc-BIM-Studio/lapin-extensions` on GitHub and pushing the local folder (without `dist/`).
2. One release per example (`mobilier-living-v1.0.0`, `sablon-romania-v1.0.0`, `pereti-perimetrali-v1.0.0`) with the file from `dist/<id>/`, exactly the one whose SHA-256 is in the manifest (`SHA256SUMS.txt`).
3. Publishing the index on lapinbim.com:
   - run `node tools/build-index.mjs --mirror https://lapinbim.com/extensions/files/`;
   - put `index.json` and the copies of the files (checked by SHA-256) into `/extensions/`. That is a vhost on Hetzner, next to the forum (§9.5), with the `Access-Control-Allow-Origin` header for the web edition. Hosting is set up separately;
   - later, this happens automatically on a push to `main`.
4. A forum topic for the catalogue and one per example (`forum` in the manifests).

## 9. The questions, with the user's answers (12.11.2026: all the recommendations)

1. **Licences: any SPDX expression is accepted**, and proprietary `LicenseRef-…` ones too. The permissive ones are marked "permissive", the others "read its terms" (as until now).
2. **Non-commercial ones are accepted and clearly marked "Non-commercial"**, in Browse's list and in the details.
   - **Detection** (`isNonCommercial`, the same in `manifest.ts` and in the catalogue's `tools/lib.mjs`): any SPDX identifier with `-NC` (CC-BY-NC-*, CC-BY-NC-SA-*, CC-BY-NC-ND-*, with their national variants), plus `PolyForm-Noncommercial-1.0.0`, `NCGL-UK-2.0`, and a `LicenseRef-…` that says non-commercial.
   - **The expression is evaluated:** `MIT OR CC-BY-NC-4.0` can be taken under MIT, so it is not marked; `CC-BY-NC-4.0 AND MIT` is.
   - `validate.mjs` writes "permissive", "read its terms" or "non-commercial" for each extension.
3. **Types are a separate kind, `types`, with the file `.lapintypes`** (done, §9.1). The engine command `types.importFrom` brings them into the open project; Manage → Type packs… exports and imports them.
4. **`.lapinprices` is also exported from Price lists…**: each list's export button in the computer's library writes the file the installer reads (`writePriceList`), always with `indicative: true`.
5. **The index lives on Hetzner**, next to the forum. Hosting is set up separately; nothing goes on the server in this stage.
   - **Serving:** an nginx vhost serves `/extensions/index.json` and `/extensions/files/<id>/<version>/<file>` under lapinbim.com. Cloudflare stays only the DNS and the proxy.
   - **Headers:** `Access-Control-Allow-Origin: *` for the web edition. `Cache-Control` is short on the index (5 min) and long on the files, which are immutable: a new version is another path.
   - **Publishing:** `build-index.mjs --mirror https://lapinbim.com/extensions/files/`, then the index and the copies checked by SHA-256, through rsync, from the catalogue's CI on a push to `main` (a deploy key for that folder only).
6. **Every package is copied to lapinbim.com** (a mirror), for all extensions.
   - The web edition reads the copies (`package.mirror`). The desktop reads the author's release and, if that does not answer, the copy (to do in stage 1).
   - `build-index.mjs --mirror` already exists; a copy is made only after its SHA-256 matches the manifest's.
   - Authors accept the copying through the authors' terms (§6.1).
7. **A new signing key for stage 1, separate from the updater's**: Ed25519, kept like the updater's (a file outside the repo, its password in the keychain, never in chat).
   - It signs the index (`index.json.sig`) and each package in the index.
   - Lapin has the public key built in and can rotate it (a list of valid keys).
   - The updater and the catalogue do not share their key: a leaked catalogue key cannot sign an application update.
8. **The 64 MB limit stays**, for every kind.
9. **Download statistics come in stage 1**: an anonymous counter on the store's API (per extension and version, with no IP kept), shown in the author's dashboard. In stage 0 there is nothing, and the nginx logs are not used for statistics.

### 9.1 Type packs (`.lapintypes`, the kind `types`)

**The chosen format: JSON, not a small `.lapin`.** It can be read and checked without the engine (the catalogue's CI, pure Node), it shows what it contains, and a type brought in carries nothing of the rest of a project (storeys, views, settings).

```
{ format: "lapin-type-pack", version: 1, name?,
  types:     [{ref, class, name, predefinedType?, attrs, material?}],
  resources: [{ref, class, name, predefinedType?, attrs}] }
```

- `ref` is the entity's id in the project it came from; any text in `attrs` equal to a `ref` points at that item (a layer set at its materials, a beam type at its copy of the profile). `material` is the `ref` of what gives the type its IfcRelAssociatesMaterial (its layer set or a material).
- **Export** (`types.export {ids, name?}` → `{text, file}`):
  - only types: the class its module describes as `type` (walls, slabs, doors, windows, columns, beams, stairs, curtain walls, ceilings, skirtings…);
  - with all they need: the layer sets, the materials named by id or by name under a key that says material (a door's frame), the copies of profiles and any other resource in `attrs`;
  - never the spatial structure or placed elements;
  - the attributes the project writes for itself (`added`, `imported`) stay out.
- **Import** (`types.importFrom {text, refs?}`, **one undo step**) brings the chosen types (without `refs`: all) and only what they need. Ids are always new.
  - **A material** is the project's when the project has one with the same name (without changing it); otherwise it is created as the pack has it (the colour, the appearance, the hatches).
  - **A resource** identical to one of the project's (the class, the name, the attributes) is that one (a profile copy from the same library); otherwise it is created.
  - **An identical type** (the class, the name, the attributes, the layers) **is reused**. One with the same name but different is added beside it as "Name (2)": the project's own is never changed. Any other type is created.
  - **A wrong pack is refused whole:** another format, a newer version, a class that is not a type in this edition, or the spatial structure among the resources.
- `types.readPack {text}` says which types it has and which already have a name in the project; `types.list` gives the project's types for export.
- **The installer** (`typesInstaller`, registered by core): the pack is checked (`readTypePack`), kept on the computer (IndexedDB `lapin-type-packs`), and its types brought into the open project; Installed → Add its types to this project brings them into another. Uninstalling forgets the pack; the types already in projects stay.
- **Manage → Type packs…**: the project's types ticked and exported (Export types…), a `.lapintypes` read, its types chosen and added (Import types…).

## 10. Stage 1: what was built (decision 504, 12.11.2026)

Approved by the user on 12.11.2026. The code and the tests are done; nothing is published on the server, the forum's settings are not changed, and no key or token is made (the steps are in `deploy/store/README.md`).

- **The store's site** (`lapin-extensions/site/build.mjs`, Node with no dependencies; `npm run site`):
  - the list: search and filters by kind; it works without JavaScript too;
  - one page per extension at `/extension/<id>/`: description, screenshots, the versions from the manifest's git history, the licence (marked permissive / read its terms / non-commercial), the Lapin versions, the size, the SHA-256, the author, the page, the forum topic, how to install, the downloads counted by the store, "Signed";
  - the authors' pages, Publish, the legal drafts;
  - Lapin's look: the tokens from `theme.css`, paper and graphite, following the system or chosen;
  - the texts in `site/strings.mjs`, English first, ready for other languages (`--lang`). All the text from manifests is escaped.
- **The "Open in Lapin" link**: not done. `lapin://` would need the `tauri-plugin-deep-link` plug-in (and `single-instance` on Windows and Linux), a new Rust dependency, and the scheme registered in each platform's packages: not cheap now. The page says "In Lapin: File → Extensions…, search for "<name>", then Install" and has a button that copies the id. To be done with a desktop build, separately.
- **The store's service** (`deploy/store/`). It follows the `deploy/feedback` model: Node 22 with no dependencies, JSON files with locking and atomic writes (not SQLite), the container `lapin-store` on `127.0.0.1:3465`.
  - **What it serves:** the pages, the catalogue (`lapinbim.com/extensions/*` is routed here by the tunnel) and the files authors upload.
  - **Sign-in through DiscourseConnect:** the shared secret is in `store.env` on the server. It keeps the forum's id, user name and display name, never the email. Sessions are in memory only, for 14 days.
  - **The author's dashboard** (`/dashboard/`): a new extension or a new version, with the manifest's fields, the file checked with `checkPackage` as in CI and in Lapin, at most 6 screenshots, and the authors' terms accepted. It becomes **a pull request** in `Ciuntuc-BIM-Studio/lapin-extensions` (a branch, the manifest, the screenshots; the GitHub token is in `store.env`). The dashboard shows its state (open / the checks / accepted / refused), withdrawal and the downloads.
  - **The files live on lapinbim.com**, not in GitHub releases: `store.lapinbim.com/packages/<id>/<v>/<file>` is `package.url`, served only as a download. The author needs no GitHub account, and the file is exactly the one checked.
  - **Ownership:** an id belongs to the first author whose pull request is accepted; only they (or a maintainer) send its versions.
  - **Downloads:** `/api/download/<id>/<v>` counts (per extension, version, month; nothing about who) and sends to the copy. Lapin tries this address first, then the author's file, then the copy (§9.6, done).
  - **Limits** per visitor and per author, and a daily clean-up: refused or withdrawn submissions are deleted after 90 days.
  - **CLI:** `list`, `stats`, `owner`, `forget` (deleting someone's data, GDPR). Tests: `pnpm test:store` (the forum and GitHub simulated).
- **Signing**: the store's Ed25519 key, separate from the updater's.
  - **The key** is made once by `node scripts/store-key.mjs`: the encrypted file `~/.lapin/store-key.pem`, its password in the keychain item `lapin-store-key` (not yet made).
  - **Publishing:** `lapin-extensions/tools/publish.mjs` runs on the Mac, so the key never reaches the server.
    - It builds the index with `mirror` and `download`, and copies the packages after checking their SHA-256.
    - It signs each package (`package.signature`, over `lapin-extension-package`, `1`, the id, the version, the SHA-256 and the size, one by one) and the index (`index.json.sig`).
    - It generates the site and, with `--upload`, sends it all through rsync.
  - **In Lapin** (`ui/src/platform/extensions/signature.ts`, `keys.ts`):
    - Ed25519 is checked through WebCrypto, or through our own verifier (RFC 8032, BigInt) where WebCrypto has no Ed25519;
    - the public keys are in `STORE_KEYS` (a list, so keys can rotate);
    - **while the list is empty, signing is not active** and Lapin's catalogue is read as in stage 0;
    - once the key is in the list, an unsigned or wrongly signed Lapin catalogue is not read, and its unsigned packages do not install;
    - a catalogue the user chose may be unsigned (Browse says "This catalogue is not signed by Lapin Extensions…"), but never wrongly signed.
  - **The order:** first the signed index is published, then the build with the key.
- **Legal** (`lapin-extensions/legal/`, English as the reference + Romanian copies in `legal/ro/`, **DRAFTS, to be reviewed by a lawyer**):
  - the terms of use;
  - the authors' terms: the licence given to the catalogue (listing, copies, signing), the author's warranties, withdrawal;
  - the privacy notice, under GDPR:
    - the controller: Ciuntuc BIM Studio SRL, CUI RO51725860, J2025031471009, Huși, hello@lapinbim.com;
    - the data: the forum identity, the submissions, anonymous downloads, technical data in memory only;
    - the processors: Cloudflare, Hetzner, GitHub;
    - the rights, and the supervisory authority ANSPDCP;
  - notice and action (DSA): the point of contact, the notice's content, provisional withdrawal, the statement of reasons, appeals.
- **What must be done to publish** (`deploy/store/README.md`):
  1. A fine-grained GitHub token, for `lapin-extensions` only: Contents, Pull requests, Checks.
  2. On the forum: `enable discourse connect provider`, and `discourse connect provider secrets` = `store.lapinbim.com|<secret>`.
  3. On the server: `store.env`, the service in compose, `deploy/store/deploy.sh`.
  4. In Cloudflare: two tunnel routes, `store.lapinbim.com` and `lapinbim.com` with the path `^/extensions/`, to `127.0.0.1:3465`.
  5. `publish.mjs --unsigned --upload`.
  6. Then, when the user decides, the key and the build with it.

## 11. Stage 2: the user's decision (12.11.2026)

- **2a, approved:** other authors' paid extensions are **sold by their authors, through their own seller** (Lemon Squeezy, Gumroad, Paddle…), who handles VAT and invoices.
  - The store only sends to the author's purchase page, and Lapin checks the licence key through the author's provider. **0 % commission.**
  - The proposed manifest field (**implemented in stage 2a, §12**, with `terms` added) is **`purchase`**, not `licence`, so as not to be confused with `license`. That one is the SPDX licence, which stays mandatory and states the terms of use:

  ```
  purchase: { provider: "lemonsqueezy" | "gumroad" | "paddle" | …,
              productId: "<the product's id at the provider>",
              checkoutUrl: "https://…",     // the author's purchase page
              price?: "29 EUR" }            // only shown, indicative; the real price is the seller's
  ```

  - **A paid package has no public copy:** `package.url` is the seller's download address, or a store address that asks for the key.
  - **Lapin asks for the licence key** at install and checks it through the provider's public validation API (Lemon Squeezy `/v1/licenses/validate`, Gumroad `/v2/licenses/verify`), on the desktop through the engine. The key stays on the computer.
  - **To be settled at implementation:** the accepted providers, what happens without internet, revocation.
- **2b, changed:** Lapin's premium modules (mep-pro, urban, rail, planning-pro) are **not** sold as extensions in the store. They become a **premium platform "like a SaaS, but local + server"** (the desktop application plus a server), planned separately in `docs/plan-premium-platform.md` (**to be written**, outside this stage).
- Kinds 3 (JS plug-ins) and 4 (signed C# modules, for partners) stay for later.

## 12. Stage 2a: paid extensions, sold by their authors (decision 660, 17.11.2026)

The user's decisions (17.11.2026, all the recommendations):

1. **What can be sold now:** content and Python scripts (the kinds that exist today). JS plug-ins (kind 3) and signed C# modules (kind 4) stay for a stage with a sandbox and enforced permissions. An explicit list (`PAID_KINDS`) says which kind may be paid, so a new kind does not become sellable without someone deciding.
2. **The providers:** Lemon Squeezy and Gumroad were proposed; **only Gumroad remains** (the user's decision of 17.11.2026, §12.9). It has a public key-verification API, with no API account of ours. Checked on 17.11.2026 in their documentation:
   - **Lemon Squeezy, removed:** it closed sign-ups on 16.11.2026 and stops processing payments from 31.03.2027 (https://www.lemonsqueezy.com/blog/saying-goodbye-to-lemon-squeezy). Its sellers are sent to Stripe Managed Payments, which, as the migration guide itself says (https://docs.lemonsqueezy.com/migration), "does not deliver files, license keys or download links".
   - **Gumroad:**
     - the call: `POST https://api.gumroad.com/v2/licenses/verify`, a form with `product_id`, `license_key`, `increment_uses_count=false`. The default is `true`: each check would count a use;
     - the answer: `{success, uses, purchase: {product_id, refunded, disputed, dispute_won, chargebacked, subscription_ended_at, subscription_cancelled_at, subscription_failed_at, …}}`;
     - a disabled key answers 404 `{success: false, message: "This license key has been disabled."}`, and an unknown one 404 with another message.
3. **Where the paid package comes from:** the store holds it, **with no public copy at all**, and gives it only after a good key, checked with the provider **by the store's server**. Our signature stays on the package; the author needs no hosting.
4. **Without internet:** a successful check is remembered on the computer; Lapin checks again at updates; after installing, nothing is blocked without internet.
5. **Revocation** (a refund, a disabled key): no updates and a note in Lapin; nothing installed is removed.
6. **Review:** the same pull request as for free ones (manifest + screenshots / an example); the authors' terms get a paragraph about selling: the author is the seller and handles refunds; the store only lists.
7. **Legal:** the terms in `lapin-extensions/legal/` (English as the reference, the copies in `legal/ro/`) completed for paid extensions; on the extension's page, "Sold by <author> through <provider>" with the link to the seller's terms; they stay marked "DRAFT, to be reviewed by a lawyer".

### 12.1 The `purchase` field in the manifest

```
purchase: { provider:    "gumroad",                       // the only one accepted today (§12.9)
            productId:   "<the product's id at the provider>", // Gumroad: product_id (the product's Content tab)
            checkoutUrl: "https://…",                         // the author's purchase page (Buy)
            terms:       "https://…",                         // the seller's terms of sale and refund
            price?:      "29 EUR" }                           // only shown, "indicative"; the real price is the seller's
```

The rules (the same in `lapin-extensions/tools/lib.mjs` + `schema/manifest.schema.json`, in the store's copy `deploy/store/server/catalogue-lib.mjs` and in `ui/src/platform/extensions/manifest.ts`):

- `provider` from the list of accepted providers (`PROVIDERS`); `productId` 1–100 characters `[A-Za-z0-9_=+/.-]`; `checkoutUrl` and `terms` https; `price` text of at most 40 characters.
- `kind` from `PAID_KINDS` (today: all the content kinds and `script`).
- **`package.url` is the store's address that asks for the key**: `https://<store>/api/licensed/<id>/<version>`, with the manifest's id and version. `package.mirror` and `package.download` are absent: there is no public copy, and no counter that would send to a copy. `sha256`, `size` and `signature` stay: Lapin checks the package it receives exactly as it checks a free one.
- `license` stays SPDX and mandatory (usually a `LicenseRef-<Author>-EULA`, with the text at `purchase.terms` or `homepage`).

### 12.2 Where the paid package lives and how the author gives it

- **On the server:** **`/data/paid/<id>/<version>/<file>`**, outside any served path (`/packages/…` is for free ones under review, `/extensions/files/…` for copies). Only `POST /api/licensed/…` reads it, after the key, and only if its size and SHA-256 are the published index's.
- **The simple path (the store):** in the author's dashboard, "Sold through your own shop", with the provider, the product's id, the purchase address, the terms and the price shown. The uploaded file is checked (`checkPackage`) and put directly in `/data/paid/…`, not in `/data/packages/…`. The pull request holds only the manifest and the screenshots.
- **The GitHub path:**
  1. The author opens the pull request with the manifest and sends the file **privately** to the maintainer (an email to hello@lapinbim.com or a private download link).
  2. The maintainer puts it in `lapin-extensions/private/<id>/<version>/<file>`. The folder is **ignored by git**, so it is never in the public repo.
  3. `node tools/validate.mjs --only <id>` checks it from there.
  4. `node tools/publish.mjs --upload` sends it to `store-data/paid/` (rsync without deletion), after checking its size and SHA-256.
- **The public CI** cannot download a paid package: it checks the manifest and writes "paid: the package is checked privately". A paid pull request is accepted only after the package was checked by the store at upload, or by the maintainer from `private/`.
- `publish.mjs`: a paid package is not copied into `extensions/files/` and gets no `mirror` / `download`; it is signed like any other (the signature is over the id, version, SHA-256 and size, so it does not need the file).

### 12.3 The store's service (`deploy/store/server`)

| Address | What it does |
|---|---|
| `POST /api/licence/check` `{id, key}` | checks the key with the extension's provider → `{status: active | revoked | invalid | unavailable, message?}` |
| `POST /api/licensed/<id>/<version>` `{key}` | the key is checked; `active` → the file (`application/octet-stream`, `no-store`); otherwise 403 (`invalid`, `revoked`), 503 (`unavailable`), 404 (no paid extension / the file not put yet), 429 |
| `OPTIONS` on both | CORS for the browser edition (`POST`, `content-type`) |

- **The key goes in the request's body (POST)**, never in the address: addresses end up in the nginx and Cloudflare logs.
- **The providers** (`providers.mjs`): one function per provider, a `fetch` with `AbortSignal.timeout(10 s)`. The result is one of:
  - `active`: a good key, for this product;
  - `revoked`: a key of the product, but refunded, disabled, expired, disputed, or the subscription ended;
  - `invalid`: not a key of this product;
  - `unavailable`: the provider does not answer, 429, 5xx.

  For Gumroad:
  - `success`, plus `product_id`, plus none of `refunded`, `chargebacked`, `disputed` without `dispute_won`, a past `subscription_ended_at`, or `subscription_failed_at` → `active`;
  - otherwise → `revoked`;
  - a 404 "disabled" → `revoked`;
  - another 404 → `invalid`.
- **What the store keeps: nothing about keys.** No key, no hash of a key, no IP address is written to disk or to the log. The log has one line per check: the time, the extension's id, the provider and the result (`licence <id> gumroad: active`). Rate limits use, as until now, a salted hash of the address Cloudflare gives, **in memory only**, for at most an hour.
- **Limits:** 20 checks an hour per visitor, 60 a minute in all (a limit of ours, so the provider is not hammered); beyond them, 429 / `unavailable`. The request's body ≤ 4 KB; the key 1–200 printable characters.
- **The author's dashboard:** the `purchase` fields (optional), the paid file put in `/data/paid/`; `cli.mjs paid` lists the paid files held.

### 12.4 In Lapin

- **Browse:**
  - the list and the details show the label "Paid" and the price "(indicative)";
  - the details also show "Sold by <author> through <provider>", **Buy** and **Seller's terms**. Buy opens `checkoutUrl` in the system's browser: `support.open` → `extensionPage` on the desktop, a new tab in the browser;
  - Install shows the **Licence key** field, for the key received from the seller.

  Installing, in order:
  1. the key goes to `POST /api/licensed/…`: on the desktop through the engine (`extensions.post`), because the webview cannot reach other sites; in the browser, `fetch` with CORS;
  2. the size, the SHA-256 and the signature are checked, as for any package;
  3. the kind's installer runs.
- **The key stays on the computer:** `lapin.extensions.licences` (`{<id>: {key, provider, status, checkedAt}}`), with the other `lapin.*` settings: in the browser in the page's storage, on the desktop also in `settings.json` in the user's Lapin folder. It goes into no project, report or export. Uninstalling keeps it (a reinstall does not ask again); **Forget licence key** deletes it.
- **Without internet:** installed means installed; nothing checks the key at use or at start.
- **Checking again:** when the catalogue is read, `POST /api/licence/check` runs for each installed paid extension that has a new version in the catalogue, or was checked more than 7 days ago. `unavailable` / no network: the earlier state stays. `revoked` / `invalid`: the state becomes `revoked`.
- **Revocation:** a `revoked` paid extension gets no more Update; Installed and the details say "Its licence is no longer valid (refunded or disabled by its seller): it gets no more updates. What is installed stays." and offer **Enter another key**.
- The new texts are in all the languages of `ui/src/i18n`.

### 12.5 Legal (`lapin-extensions/legal/`, DRAFTS)

- **The authors' terms** get a paragraph "Selling an extension":
  - the author is the seller, through their provider, who is the merchant of record;
  - the author sets the price, invoices, handles VAT and refunds, and answers to the buyer;
  - the store only lists, holds and signs the package, and gives it to buyers with a good key; 0 % commission;
  - the licence given to the store for paid packages covers only keeping, checking, signing and delivering them to buyers, not a public copy;
  - the author keeps `purchase` true (a key revoked on a refund).
- **The terms of use** get "Paid extensions": the purchase contract is between the buyer and the author (through the provider), and Ciuntuc BIM Studio is not a party; refunds go to the seller; the key stays on the computer; what is installed stays after revocation, with no updates.
- **Privacy:** on a check, the key passes through the store to the provider and is not kept; what is written in the log; the provider (today Gumroad) as a recipient, each with its own policy.
- **The extension's page:** "Paid", the price "indicative", **Buy** (to `checkoutUrl`), "Sold by <author> through <provider>" with the link to `purchase.terms`; no download button.

### 12.6 The test example

`lapin-extensions/examples/paid/<id>/manifest.json` is a paid manifest marked "EXAMPLE — not published", **outside `extensions/`**, so neither the index, the site nor publishing sees it. The tests read it directly. For an end-to-end try, the maintainer copies it into a local catalogue (the `extensions/` of a working copy, never on `main`), with a test product of the provider (`deploy/store/README.md` → "Paid extensions").

### 12.7 The tests

- `lapin-extensions`: `tools/lib.test.mjs` (the `purchase` rules), `site/build.test.mjs` (Paid, Buy, Sold by, no download), `tools/catalogue.test.mjs` (no `mirror` / `download` for paid ones).
- The store: `deploy/store/server/server.test.mjs` with the providers simulated (a good key, another product's, refunded, disabled, a provider down, a timeout), the file given only after a good key and only if it is the index's, no key and no address in the log, the limits, CORS.
- The engine: `ExtensionFetchTests` (`extensions.post`: only https or this computer, the state and body returned, guests refused).
- The interface (vitest): `manifest.test.ts`, `install.test.ts`, `licence.test.ts` (the key kept, checking again, revocation, no updates).

### 12.8 What is left

- **Activations / seats:** we do not count computers per key (Gumroad `uses`); a key passed on works until the author disables it. To do if authors ask.
- **Paddle, Stripe Managed Payments** and other sellers without a licence API: keys issued by the store, §12.10.
- Kinds 3 and 4, with the sandbox and enforced permissions.
- A public list of the store's revocations (an extension taken out) — §5.
- The publishing steps: `deploy/store/README.md` → "Paid extensions".

### 12.9 The question, with the user's answer (17.11.2026)

1. **Lemon Squeezy closes on 31.03.2027:** do we keep it or remove it? **Answer: removed now.** It closed sign-ups on 16.11.2026 and stops processing payments from 31.03.2027; Stripe Managed Payments, where it sends its sellers, gives no licence keys. **Gumroad is the only provider** for now: `PROVIDERS` / `PROVIDER_URLS` have only `gumroad` (the catalogue, the store, Lapin), a manifest with `lemonsqueezy` is refused, and the legal texts and the guide name only Gumroad.

### 12.10 A later stage (not done): licence keys issued by the store

For sellers with no licence API (Stripe Managed Payments, Paddle). Lapin's checking stays the same (`/api/licence/check`, `/api/licensed/…`); only the store answers instead of the provider.
1. **The author's seller tells the store, through a webhook, at every sale.** The webhook is signed with the webhook's secret and checked by the store.
2. **The store generates the key** and sends it to the buyer, or the seller shows it on the receipt.
3. On a refund, another webhook revokes it.

- **The store becomes the keeper of the keys**, and so of some personal data: the key tied to a purchase (the order's id, probably the buyer's email, so it can be sent again). That would need:
  - keys kept only as a hash (Argon2/scrypt), plus a lookup by a prefix, never in clear;
  - the legal basis: the contract, art. 6(1)(b);
  - the retention: as long as the extension is listed, plus the limitation period;
  - the right to erasure, with the key revoked;
  - a processing agreement with each author (the store would be the seller's processor);
  - the privacy notice and the authors' terms changed;
  - each author's webhook secrets in `store.env` or encrypted on disk;
  - backups.
- **It is not built now**; to be decided when an author asks for it.
