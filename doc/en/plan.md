> [Version française](../fr/plan.md)

# Plan: the Kazo Solid app for Kazo OS

Step 0 of the Kazo OS [Craft app port guide](https://github.com/Kazo-ca/Os/blob/main/doc/en/wasm/dev/craft-app-port-guide.md),
written on 2026-11-14. This document sets the goal, what reading and trying SolidCraft showed,
the decisions, the questions asked to Benoit and the work items. Each work item is written to be
handed as it is to an implementation thread. The closest references are Kazo Deck
([plan](https://github.com/Kazo-ca/Os-App-Deck/blob/deck/base/doc/en/plan.md), one window per
document) and Kazo Fig ([plan](https://github.com/Kazo-ca/Os-App-Fig/blob/main/doc/en/plan.md),
upstream build shipped unmodified).

## Goal

Kazo Solid is **the parametric 3D design (CAD) app of Kazo OS**: constrained sketches, extrudes,
revolves, fillets, assemblies, sheet metal, on an editable timeline. It wraps the official web
build of [SolidCraft](https://github.com/tangency-labs/solidcraft) (Tangency Labs, Rust and egui,
version 0.41) and connects it to the files, dialogs and associations of Kazo OS. No Kazo OS app
does CAD today.

**What sets this port apart from the others**:

- **The licence.** The geometry code is `MIT OR Apache-2.0`, but the interface and the brand are
  under a separate licence (`LICENSE-BRAND`). A modified build cannot be distributed without
  written permission. So Kazo Solid ships **the official build, byte for byte**, and nothing goes
  to the Store before the maintainers answer in writing (see "Licence").
- **No Craft hosted mode.** SolidCraft has no `craft-web-host`. It has its own mode,
  `?embed=editor`, where the host page keeps the files and moves them with three control methods.
  A small script in our page bridges it to the shared Kazo OS module.
- **The size.** The `.wasm` is 159.6 MiB: above GitHub's 100 MiB limit and above Kazo OS's limit
  of 128 MiB per file. It is shipped compressed (49 MiB) and decompressed by the page at start.
- **No File menu.** In `embed=editor` mode SolidCraft has no application bar and no start page:
  Open and Save are the Kazo window's job.

The shared Kazo code (iframe, proxy, file picker, shortcuts, `DocumentSession`, exports) comes
from the `Kazo.Os.AppSdk.Craft` module of Kazo OS
([ADR 0259](https://github.com/Kazo-ca/Os/blob/main/doc/en/wasm/dev/adr/0259-shared-craft-app-host-module.md),
[ADR 0264](https://github.com/Kazo-ca/Os/blob/main/doc/en/wasm/dev/adr/0264-craft-app-exports-reach-the-window.md),
[guide](https://github.com/Kazo-ca/Os/blob/main/doc/en/wasm/framework/craft-apps.md)). Minimum
version: **Kazo OS 3.3.4**, already released.

## Licence

Read on 2026-11-14 in upstream's `LICENSE-BRAND`, `NOTICE` and git history.

| Point | Finding |
| --- | --- |
| Code | `MIT OR Apache-2.0` (`LICENSE-MIT`, `LICENSE-APACHE`). Geometry kernel `solidcore` (Apache-2.0), vendored in the repository. |
| Brand and interface | `LICENSE-BRAND`, added on 2026-11-10 (commit `312454f89`, first version: `v0.40.0`). It covers `crates/ui-kit`, `crates/look`, `crates/theme`, and the SolidCraft name, logo and app icon. |
| Allowed without asking | Read, build and modify as part of SolidCraft itself, including forks for development, testing, learning or contributing back (section 2.a). Show SolidCraft in articles and tutorials (2.c). |
| Needs written permission | Using these crates in another program (3.a); distributing them other than as part of SolidCraft (3.b); distributing a fork or modified build under the SolidCraft name, logo or icon, or with a look that could be taken for SolidCraft's (3.c); using the name or logo to name anything else (3.d). |
| Consequence | **Renaming is not enough**: a renamed modified build is "another program" that uses the UI kit. |
| Earlier versions | Section 5 leaves versions published before the licence under `MIT OR Apache-2.0` (last one: `v0.40.0-rc.3`, commit `7001949c4`, 2026-11-07). Path rejected by Benoit: it freezes the interface and goes against what the maintainers want. |
| Self-hosting | Upstream provides for it: a `solidcraft-web-<version>.zip` in every release, the `ghcr.io/tangency-labs/solidcraft-web` image, and the documented `?embed=editor` mode for "a page that keeps the designs itself". |
| The name "Fusion" | Upstream's `NOTICE` recalls that Fusion is an Autodesk trademark for CAD software. The name "Kazo Fusion" was dropped for that reason. |

**Path chosen by Benoit on 2026-11-14**: ask the maintainers for written permission **and**
build meanwhile on the unmodified official build. The request (text in
`solidcraft/upstream/permission-request.md` in the project files, posted by Benoit as an issue
at `tangency-labs/solidcraft`) asks for permission to offer the official build inside Kazo OS and
lets the maintainers choose the name of the Store entry: "Kazo Solid, powered by SolidCraft" with
our icon, or "SolidCraft" with their name and icon.

**Rules that follow for the whole series**:

1. No modified build of SolidCraft is distributed. The shipped `.js` and `.wasm` have the
   SHA-256 hashes of the upstream release's `build.json`; a test checks it.
2. The fork `Kazo-ca/solidcraft` is only for reading, trying and proposing changes upstream.
   Nothing in it goes into the package.
3. An integration need that requires a change in SolidCraft becomes an upstream issue or pull
   request (posted by Benoit), not a shipped patch.
4. **Nothing is published to the Store before the written permission.** Work item 4 waits for
   the answer.

## Reading SolidCraft (step 0, 2026-11-14)

Read on upstream `main` (version 0.42.0 in progress) and at the `v0.41.0` tag (2026-11-12). The
fork [`Kazo-ca/solidcraft`](https://github.com/Kazo-ca/solidcraft) exists.

| Point | Finding |
| --- | --- |
| Web build | `apps/solidcraft-web` (Trunk 0.21.14, `trunk build --release --public-url ./`), eframe on wgpu: WebGPU, or WebGL 2 with `?webgl`. Rust 1.95 at least. Official zip made by `packaging/web/package.sh`. |
| `?embed=editor` mode | Ribbon, design browser, timeline and properties; **no** application bar or start page; **nothing** is kept in the browser. `Ctrl+N`, `Ctrl+O`, `Ctrl+S`, `Ctrl+W` and `Ctrl+,` do nothing there ("those belong to the page"); `Ctrl+Z` and `Ctrl+Y` undo and redo. One design per frame, no tabs. |
| Command API | `window.solidcraft` in every mode (`request`, `execute`, `commands`, `inspect`, `buttons`). A **same-origin** host page reaches it as `frame.contentWindow.solidcraft`. 642 commands. Parameters: at most 16 MiB of JSON; at most 256 pending requests. |
| Moving designs | `document.export` → `{ name, revision, dirty, format: "json", text }` (read only); `document.saved { revision }` → the design counts as saved at that revision, refused when it changed since; `document.import { text, name? }` → replaces the active design, without asking about unsaved changes (that is the host's to decide). |
| Following changes | `document.revision` → `{ document_id, revision }`: only reads, does not wait behind a slow command; `revision` moves on every design change, never on a selection or a view. No message pushed to the page. |
| File format | `.sc` design (`.solidcraft` from earlier versions still opens). Two forms: **container** (a ZIP: the design's JSON, binary payloads `blobs/<sha256>`, caches) or **JSON** alone. SolidCraft recognizes a container by its first bytes (`PK`), whatever the name. `document.export` returns the JSON. |
| Imports | Ribbon › Insert: STEP, IGES, meshes (3MF, STL, OBJ, PLY, glTF), point clouds, images, DXF, SVG, through the browser's file picker (File System Access, or `<input type=file>`), or a drop on the page. The `file.insert_step` and `file.insert_mesh` commands take a **path** in the virtual file system, not bytes. |
| Exports | Commands `file.export` (STL, OBJ, STEP, 3MF, glTF, URDF, JSON), `file.export_step`, `file.export_stl`, `file.export_3mf`, `file.export_obj`, `file.export_view_dxf`, by path. In the web build a path under `/downloads/` becomes a download. |
| Theme | `ui.set { dark: true\|false }`. 27 themes in the preferences (`ui.prefs`, key `theme`). Dark by default. |
| Language | Preference `language` (`system` by default) through `ui.prefs`. |
| Network | Update check (`check_updates`), AI features (`ai_features`, agents), sending to Printables (`file.open_in_easyprint`, with consent). To turn off or to state in the Store entry. |
| Service worker | The official `index.html` registers `sw.js` (offline use and updates). Not to be registered inside Kazo OS, which has its own. |
| Canvas | `solidcraft_canvas` and `#solidcraft_loading`: one instance per HTML document, so one iframe per window. |
| Brand | App icons in `icons/`, loading logo `loading-logo.svg`: brand material. Not shipped until the maintainers have answered. |
| Upstream repository rules | Rust only, no panic outside tests, everything is a command, never break wasm, no agent attribution, an issue before a pull request. |

### Trial (2026-11-14)

Done with the official `solidcraft-web-0.41.0.zip` of the upstream release
(SHA-256 `dba9a9389bc479c0c12412493b1bec61d4d4d8434cd4e4cca67ad2a38183fc04`, 56.8 MB), with no
change at all, in Chromium without a graphics card (WebGL 2 through SwiftShader, `?webgl`), in a
same-origin frame with `?embed=editor`.

| Point | Result |
| --- | --- |
| Content | `index.html`, a 193 KB `.js`, a `.wasm` of **167,302,762 bytes (159.6 MiB)**, fonts, icons, licences, `build.json` (SHA-256 of every file). 176 MB unpacked. `gzip -9` of the `.wasm`: **51,373,954 bytes (49.0 MiB)**. |
| Limits | GitHub: 100 MiB per file, **exceeded** by the raw `.wasm`. Kazo OS: 128 MiB per file and a package of 128 MiB unpacked, **exceeded**; package of 64 MiB compressed, met with the compressed `.wasm` (about 52 MiB in total). |
| Start | `window.solidcraft` present 2.2 s after the page load, from a local server. Empty design "Untitled", dark theme. |
| Our page with the compressed `.wasm` | A page of ours that loads the official `.js` and gives it the `.wasm` through `DecompressionStream('gzip')` (`init({ module_or_path: Response })`): **starts in 2.0 s**, same behaviour, **no service worker** registered. The `.js` and the `.wasm` stay upstream's, byte for byte. |
| Moving a design | Box made with `solid.box`: `document.export` returns `dirty: true`, `format: "json"`, 539 characters starting with `"format": "solidcraft/55"`. `document.saved { revision }`: `dirty: false`. `document.import { text, name: "Bracket" }`: design replaced, new `document_id`, `errors: 0`. |
| Theme | `ui.set { dark: false }`: light interface. |
| Storage | `localStorage`, IndexedDB and OPFS empty after the trial. |
| `Ctrl+S` | Does nothing in the frame, as announced. |
| Exports by command | `file.export { path: "/downloads/part.step" }` and `file.save_as { path: "/downloads/part.sc" }` answer ok (9,745 and 10,199 bytes) but **no download came out** in `embed=editor` mode during the trial. No method returns the bytes of a virtual file. |
| Window | The ribbon is readable at 1280 × 820. |

Screenshots and scripts of the trial: `solidcraft/trials/` in the project files (outside the
repository).

**Not verified at this step**: WebGPU rendering (no graphics card here); memory per window;
ribbon › Insert end to end (browser file picker replaced by the OS one); a way to get an export
(STEP, STL) out in `embed=editor` mode; whether a design JSON named `.sc` opens in desktop
SolidCraft (read in the code: yes, by its first bytes); what `document.export` does with imported
bodies (in the JSON or in separate payloads); a large design close to the 16 MiB limit.

## Decisions

| # | Subject | Decision |
| --- | --- | --- |
| 1 | Name and identity | "**Kazo Solid**" (Benoit, 2026-11-14). Id `ca.kazo.solid`, publisher `pub.kazo`, assembly `Kazo.Solid`, Kazo Vector's category (graphics), several windows, 1280 × 820 window, minimum content 800 × 520. The Store entry's name may become "SolidCraft" if the maintainers ask for it (see "Licence"). |
| 2 | Shared code | `Kazo.Os.AppSdk.Craft` module. Submodule `external/KazoOs` on the `v3.3.4` tag, nuget.org packages in Release, `minimumVersion` 3.3.4. |
| 3 | Screen change | `canChangeScreen: false` in the manifest from work item 2 (ADR 0263), like the other Craft apps. |
| 4 | Where the build comes from | **The official build, unmodified**: `tools/fetch-solidcraft.sh` downloads `solidcraft-web-<version>.zip` from the `tangency-labs/solidcraft` release, checks its SHA-256 (pinned in the script), then the SHA-256 hashes of `build.json`. No Rust compilation, no submodule of the fork. |
| 5 | Size | The `.wasm` is stored **compressed** (`gzip -9`, one file of about 49 MiB) in `wwwroot/solidcraft/`, next to the official `.js` and a `solidcraft.json` (version, URL and SHA-256 of the zip, names, sizes and SHA-256 before and after compression). In the repository: the Store builds the app at the tagged commit and takes no binary from the author. No Kazo OS limit to change, so **no ADR in `Kazo-ca/Os`**. Submitted to Benoit (Q3). |
| 6 | Iframe page | A page of ours, `wwwroot/host/index.html`: canvas `solidcraft_canvas`, `#solidcraft_loading`, and a `boot.js` module that decompresses the `.wasm` and starts the official `.js`. Parameters `?embed=editor` always, `webgl` as a fallback. It does not register `sw.js` and ships neither upstream's web manifest nor its icons. |
| 7 | Bridge to the module | A page script, `wwwroot/host/solid-bridge.js` (minimal, commented JavaScript, no inline script), translates the `window.solidcraft` API into the messages `CraftFrame` expects: `ready`, `open { name, bytes }` → `document.import`, `save { name, saveAs, bytes }` ← `document.export`, `dirty`. The C# module is reused unchanged. If an extension point is missing, that is a request for `Kazo-ca/Os`, in a separate thread. |
| 8 | Saved format | The **design's JSON** (`document.export`), written as UTF-8 in a `.sc` file. Desktop SolidCraft recognizes it by its content. To verify in work item 3 with desktop SolidCraft if possible. |
| 9 | Opening a container | A `.sc` or `.solidcraft` saved by desktop SolidCraft is a ZIP. The window recognizes it (`PK`), reads the design's JSON from it in C# and passes it to `document.import`. If the design needs binary payloads that `document.import` cannot receive, the window says so plainly and opens nothing. Exact behaviour recorded in work item 3. |
| 10 | "Modified" and saved | The bridge reads `document.revision` after each real user gesture (key, click, wheel), never on a free-running timer, and sends `dirty` when the revision moves away from the last saved one. It calls `document.saved { revision }` only **after** the window confirmed the write: a cancelled dialog or a refused write leaves the design modified. |
| 11 | One design per window | SolidCraft has no tabs in `embed=editor` mode. A double-click, Open and `Ctrl+N` open a **new Kazo Solid window**, unless the window is still empty and untouched: it takes the file. Like Kazo Deck. |
| 12 | File commands | Submitted to Benoit (Q4): a small Kazo menu bar above SolidCraft, or shortcuts only. |
| 13 | Shortcuts | `ReservedKeys.Os`. `Ctrl+N`, `Ctrl+O`, `Ctrl+S`, `Ctrl+Shift+S` go to the window (SolidCraft leaves them to the page). Conflicts with the OS chords are recorded in work item 3 and noted in the Store entry. |
| 14 | Close guard | Yes / No / Cancel when the revision is not the saved one. Yes saves and waits for the write to finish. `document.import` asks nothing: the window asks before replacing a modified design. |
| 15 | Theme and language | `ui.set { dark }` at `ready` and on every OS theme change. OS language through `ui.prefs { language }` if SolidCraft has the language; otherwise `system`. |
| 16 | Settings and recovery | Nothing is kept in the browser in `embed=editor` mode: no `WebStorageFile`, **no crash recovery in the first version**, with a `TODO` issue for later (same decision as Kazo Vector, Photo and Fig). |
| 17 | Network | Update check turned off at `ready` (`ui.prefs { check_updates: false }`): the version comes from the package. AI features and sending to Printables stay as SolidCraft has them; their state in `embed=editor` mode is recorded in work item 3 and written in the Store entry. |
| 18 | Imports and exports | Imports go through SolidCraft's ribbon › Insert, with the OS file picker in place of the browser's (`PickRequested`). Exports: recorded in work item 3; if nothing comes out in `embed=editor` mode, that is a limit of the first version, written in the Store entry, and an upstream request (already asked in the permission request). |
| 19 | Associations | Submitted to Benoit (Q5). |
| 20 | Icon and brand | Kazo Solid's own icon, without the SolidCraft logo, and our own loading screen, until the maintainers have chosen. Store entry: "powered by SolidCraft", link to `solidcraft.io` and to the upstream repository. |
| 21 | Upstream track | Drafts in English, with no agent attribution, in `solidcraft/upstream/` in the project files; Benoit posts them. |
| 22 | Kazo OS ADR | None planned. If an answer needs one (Q3 = B), next free number **0272**, to recheck on `main`, the open pull requests and every `*/base` branch of `Kazo-ca/Os` when writing it. |

## Questions asked to Benoit

Already answered on 2026-11-14: the licence path (permission requested **and** unmodified
official build) and the name (Kazo Solid, not Kazo Fusion).

Asked in the thread on 2026-11-14, **waiting for an answer**:

**Q3. How to ship a 160 MiB `.wasm`?** It exceeds Kazo OS's limit of 128 MiB per file and
GitHub's limit of 100 MiB.

- **A (recommended)**: the `.wasm` compressed in the package (49 MiB), decompressed by our page
  at start. Tried: starts in 2 s. Nothing to change in Kazo OS, the app ships with 3.3.4. Cost:
  the package is about 52 MiB for a limit of 64 MiB, so little room if SolidCraft grows; and a
  decompression every time a window opens.
- B: raise the Kazo OS limits (file and unpacked package to 256 MiB) with an ADR in
  `Kazo-ca/Os`. Helps other large apps, but needs a new OS version before Kazo Solid ships, and
  the file stays too large for GitHub (it would have to be downloaded at build time).
- C: A now, and a `TODO` issue for B if the package gets close to 64 MiB.

**Q4. Where do Open and Save go?** In `embed=editor` mode SolidCraft has no File menu.

- **A (recommended)**: a small Kazo menu bar above SolidCraft (File: New, Open, Save, Save as),
  plus the shortcuts `Ctrl+N`, `Ctrl+O`, `Ctrl+S`, `Ctrl+Shift+S`. Everything is visible; a
  little interface of ours, outside SolidCraft's frame, so without touching their look.
- B: shortcuts only, plus the close guard. Nothing to draw, but a new user does not see how to
  save.

**Q5. Which file associations?**

- **A (recommended)**: default app for `.sc` and `.solidcraft` only. STEP, STL, 3MF and the
  others are imported through the ribbon › Insert into an open design; no association in the
  first version, because SolidCraft does not receive their bytes by command. A `TODO` issue to
  open them by double-click when upstream allows it.
- B: like A, plus "Open with" for `.json` (a SolidCraft design can be a `.json`). Useful to few
  people, and the text editor stays the default app for `.json`.
- C: wait until upstream offers opening STEP and meshes by command before shipping the app. A
  fuller Store entry, but an unknown release date.

## Work items

One working repository, `Kazo-ca/Os-App-Solid`: integration branch `solid/base`, one branch and
one pull request per work item, one implementation thread per work item. Benoit merges
`solid/base` into `main`.

| No. | Branch (to `solid/base`) | Content | Depends on |
| --- | --- | --- | --- |
| 1 | `solid/1-plan` | this plan (fr and en) | — |
| 2 | `solid/2-solution` | solution, manifest, official build fetched and verified, iframe page, window that starts | plan merged, Q3 |
| 3 | `solid/3-window` | bridge, open, save, close guard, menus, shortcuts, theme, imports and exports | 2, Q4 and Q5 |
| 4 | `solid/4-store` | Store entry, icon, screenshots, README, licences, measurements, installed journey, catalog, `features.md` | 3 **and the maintainers' written permission** |

### 2. Solution and official build (`solid/2-solution`)

- Renamed solution (`Kazo.Solid`, `Kazo.Solid.Os`, `Kazo.Solid.Tests`, `tests/e2e`),
  `[AppManifest]` of decision 1, `canChangeScreen: false`, Kazo Deck's capabilities,
  associations according to Q5.
- `tools/fetch-solidcraft.sh` (decisions 4 and 5): downloads the zip of the pinned version
  (`0.41.0`, or the latest release when the work item starts), checks the SHA-256 hashes,
  compresses the `.wasm`, writes `wwwroot/solidcraft/` and `solidcraft.json`. Copies neither
  `sw.js`, nor `manifest.webmanifest`, nor `icons/`, nor `loading-logo.svg`; copies the licences
  (`LICENSE-MIT`, `LICENSE-APACHE`, `LICENSE-BRAND` if present, `NOTICE`,
  `THIRD-PARTY-LICENSES.txt`) and the fonts. Workflow `fetch-solidcraft.yml` that reruns the
  script and fails if the result differs from the repository.
- xUnit tests: the SHA-256 hashes of the `.js` and of the decompressed `.wasm` are those of
  upstream's `build.json`; the sizes stay under the Kazo OS limits.
- Iframe page and `boot.js` (decision 6), window on `CraftFrame` and `FrameStartup`: loading,
  app, error, "Retry"; theme at `ready`. Model for `boot.js`: `solidcraft/trials/boot.js` in the
  project files.
- **Done when**: the app starts in WebGL 2 (and in WebGPU on a machine that has it), two windows
  run together, the `.kazoapp` installs in the released Kazo OS 3.3.4; package size, start time
  and memory per window written here.

### 3. Window (`solid/3-window`)

- `solid-bridge.js` (decisions 7 and 10) with its tests (`npm run test:unit`, simulated page).
- Opening: file requested at launch, Open, double-click; JSON and container (decisions 8 and 9);
  same window or new window (decision 11).
- Saving: Save, Save as, first save of a design with no file; a cancelled dialog or a refused
  write leaves the design modified, with a message.
- Title `*name — Kazo Solid`, close guard (decision 14), menus according to Q4, shortcuts
  (decision 13), theme followed live, language (decision 15), updates turned off (decision 17).
- Imports through the ribbon › Insert with the OS dialog; exports: record what comes out in
  `embed=editor` mode (decision 18) and ask Benoit if several forms are equally good.
- Measurements: a design of several MB of JSON (open and save time, memory), and what happens
  close to the 16 MiB limit.
- **Done when**: no case loses changes unless the user chose it, no browser dialog or download
  comes out of the app, and the OS shortcuts work with the cursor in the model.

### 4. Store (`solid/4-store`)

- **Starts only with the maintainers' written permission**, and follows their conditions (name,
  icon, wording, version).
- Icon and Store entry in fr and en ("powered by SolidCraft"; known limits: no crash recovery,
  imports through the ribbon, exports according to work item 3, 16 MiB per design, network
  features, window that stays on its screen), screenshots taken in the released Kazo OS, README
  rebuilt from the entry (`store:readme`, "Try it" with `?autolaunch=ca.kazo.solid`),
  `THIRD-PARTY-NOTICES` in fr and en (SolidCraft, `solidcore`, `LICENSE-BRAND` and the
  permission received, fonts).
- Journey in the released Kazo OS (`tests/installed/`), "Tag a release" workflow, pull request
  to the `Os-AppMarketPlace` catalog (signing happens there, no key here), `features.md` row in
  `Kazo-ca/Os` (fr and en).
- **Done when**: Kazo Solid installs from the Store, opens a `.sc` by double-click, edits it and
  saves it.

## Upstream track

Posted by Benoit, never by a thread:

- the permission request (`solidcraft/upstream/permission-request.md`), which also asks two
  questions: how stable `?embed=editor` is, and a way to pass bytes (STEP, 3MF, STL) both ways
  in `embed=editor` mode;
- to write after work item 3, depending on what it records: opening a `.sc` container through
  `document.import`, a message pushed when the design changes, exports in `embed=editor` mode.

## Progress

- [x] Step 0: reading of the licence and of SolidCraft 0.41, trial of the official web build in `embed=editor` mode.
- [x] Benoit's answers on the licence and the name (2026-11-14).
- [x] Plan in fr and en.
- [ ] Permission request posted by Benoit; maintainers' answer.
- [ ] Benoit's answers to Q3, Q4 and Q5.
- [ ] Work item 2: solution, official build, iframe page, window.
- [ ] Work item 3: window, opening, saving, close guard.
- [ ] Work item 4: Store (after the written permission).
- [ ] Release: `solid/base` into `main` (Benoit), tag `v0.1.0`, catalog, `features.md`.
