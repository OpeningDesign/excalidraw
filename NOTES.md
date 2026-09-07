# Notes — OpeningDesign fork

Lessons from working *inside* the Excalidraw editor, and where we intend to take
this fork. Upstream's own docs cover how Excalidraw works; this is what we have
learned changing it, and what the AEC use case still needs from it.

The consuming application is
[SketchSpace](https://github.com/OpeningDesign/SketchSpace) — multi-page
collaborative redlining of construction documents. Its `NOTES.md` covers the
integration side; this file covers the editor.

---

## Lessons learned

### Adding a preference touches six places

`wheelBehavior` is the worked example. Miss any of these and it fails quietly:

1. **`packages/excalidraw/appState.ts`** — the default in `getDefaultAppState()`.
   This is the single source of truth; `restore.ts` falls back to it.
2. **`APP_STATE_STORAGE_CONF`**, same file — every `AppState` key needs a
   `{ browser, export, server }` entry saying where it is allowed to persist.
   Omit it and the preference silently fails to survive a reload.
3. **`packages/excalidraw/data/restore.ts`** — persisted and imported appState is
   **untrusted**. Validate the value explicitly and fall through to the default
   on anything unknown, rather than letting it reach the handler.
4. **`components/main-menu/DefaultItems.tsx`** — the item itself, and adding it to
   the default children of `Preferences`.
5. **`locales/en.json`** — labels.
6. **Snapshots** — several suites serialise the whole appState, so a new key or a
   changed default touches a hundred-plus snapshot values.

### A preference can be invisible to embedders

`DefaultMainMenu` in `components/LayerUI.tsx` — the fallback used when a host app
passes no `<MainMenu>` — contains **no `<MainMenu.DefaultItems.Preferences />`
at all**. Everything in the Preferences submenu (box-selection mode, snap mode,
grid mode, object snapping, and anything you add) therefore renders nowhere for
an embedder relying on the fallback.

excalidraw.com does not hit this because `excalidraw-app` supplies its own menu.
We lost an hour to it, suspecting the build.

**Arguably an upstream bug**, and a small, well-motivated PR: add `Preferences` to
`DefaultMainMenu`.

### Update snapshots one file at a time

Running `vitest --update` across several files gave `history.test.tsx` a
`number of renders` value of 3 where running it alone produces 6. The batch write
baked in the wrong number, and the test then failed by itself — looking exactly
like a regression from the change under test. Stashing and re-running on a clean
tree is what separated the two.

Render counts depend on how tests are batched. Snapshot them the way they
normally run.

### The package build externalises its siblings

`scripts/buildPackage.js` marks `@excalidraw/common`, `element`, `math` and
`fractional-indexing` as esbuild externals, and runs with `packages: "external"`.
So a build from master is **not** self-contained: a consumer needs all five
packages plus ~31 third-party runtime imports.

The 0.18.1 npm release inlined the siblings, which is why depending on the
published package needs none of this. Anyone vendoring a master build has to
reproduce it — see SketchSpace's `scripts/sync-editor.mjs`.

### Semantics worth knowing before building on the editor

- **Reconciliation** (`data/reconcile.ts`): higher `version` wins, ties broken by
  the **lower** `versionNonce`. A server that holds authoritative scene state has
  to mirror this exactly or clients disagree with it.
- **Element order** is a fractional index string, not an array position
  (`element/fractionalIndex.ts`). Concurrent reorders converge without a
  coordinator. The same trick works for ordering anything else.
- **Images load via `image.src = dataURL`** on a plain `new Image()`
  (`element/image.ts`), so a same-origin URL works wherever a data URL does — and
  stays untainted for canvas export.
- **`image/svg+xml` is a supported image type**, so SVG renders as vector rather
  than being rasterised on load.
- **`customData`** survives round-trips and is the right place to hang external
  identity — SketchSpace stores IFC GlobalIds there.

### Fork hygiene

`master` is an untouched mirror of `excalidraw/excalidraw`. Work lives on
`SketchSpace`, so `git log master..SketchSpace` is exactly what we have changed.
`upstream`'s push URL is disabled locally on purpose.

Changes offered upstream get their own branch. Defaulting `wheelBehavior` to
`zoom` is fork-only and must not reach `feat/wheel-zoom-preference`, because the
basis on which
[excalidraw#12051](https://github.com/excalidraw/excalidraw/pull/12051) is offered
is that existing behaviour is unchanged.

---

## Roadmap

Only things that have to live *inside* the editor. Anything that can be done in
the host app belongs in SketchSpace instead — pages, collaboration, persistence
and the Bonsai integration all live there deliberately, and the editor stays a
single-scene drawing surface.

Items below are intentions, not commitments; the ones marked *unproven* have not
been prototyped.

### Drafting ergonomics

Excalidraw is built for sketching; construction documents are drafted. Done so
far: an opt-in mouse-wheel zoom preference, defaulting to zoom here.

Candidates: snapping to geometry inside a placed drawing, scale-aware stroke
widths so a redline reads correctly at plot scale, and true-to-scale drawing and
dimensioning. *Unproven.*

### Vector fidelity for placed drawings

Sheets arrive as SVG and are placed as image elements — they render as vectors,
but their geometry is opaque. You cannot snap to a wall line, click a door, or
let a redline attach to something *in* the drawing.

Importing an SVG as real elements is the obvious answer and the wrong one: a
sheet is tens of thousands of paths and would swamp both the scene and the
collaboration layer. Something in between — queryable geometry without making
every path an element — is the interesting problem, and the one that would most
change what the tool can do. *Unproven.*

### First-class annotation anchoring

`customData` already carries an IFC GlobalId per placement, and that is enough
for the host app. What is missing is any editor notion that an annotation
*belongs to* something: no visual affordance, no reflow when the target moves, no
way to ask "what is attached to this?"

Worth attempting in the host first, and only moving into the editor if it clearly
cannot work there.

### Plot-accurate output

Construction documents are printed and measured. Export currently targets screens.
Sheet-size output at a true scale, with correct line weights, is a prerequisite
for anything issued for construction. *Unproven.*

### Layer discipline

A sheet has a locked titleblock, movable drawing placements, and free annotation
above them. Today that is `locked` per element plus z-order convention. Real
layers — lockable, hideable, orderable as a group — would express it directly.

### Possibly upstreamable

Kept separate because these are generally useful, not AEC-specific:

- `Preferences` missing from `DefaultMainMenu` (see above).
- The wheel-zoom preference itself — already open as
  [#12051](https://github.com/excalidraw/excalidraw/pull/12051).
