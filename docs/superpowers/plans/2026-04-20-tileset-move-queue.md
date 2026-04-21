# Tileset Move Queue Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a staged move workflow for RAW and Items that lets the user queue item category moves, inspect them in a docked palette-like panel, and apply them to `data/1310/tilesets.xml` in one batch.

**Architecture:** Introduce an in-memory move-queue service that overlays queued state on top of the existing palette model instead of mutating XML on every click. Extend palette selection and context menus to enqueue or retarget moves, add a docked staging panel that reuses the existing palette UI model, then add a single XML apply path that updates only touched sections and refreshes only affected panels.

**Tech Stack:** C++, wxWidgets, wxAUI dock panes, existing palette UI classes, pugixml, current `Materials` / `Tileset` structures.

---

## File Structure

**New files**
- `source/tileset_move_queue/tileset_move_queue.h`
- `source/tileset_move_queue/tileset_move_queue.cpp`
- `source/tileset_move_queue/tileset_xml_rewriter.h`
- `source/tileset_move_queue/tileset_xml_rewriter.cpp`
- `source/palette/panels/tileset_move_queue_panel.h`
- `source/palette/panels/tileset_move_queue_panel.cpp`

**Modified files**
- `source/palette/controls/virtual_brush_grid.h`
- `source/palette/controls/virtual_brush_grid.cpp`
- `source/palette/panels/brush_palette_panel.h`
- `source/palette/panels/brush_palette_panel.cpp`
- `source/palette/palette_window.h`
- `source/palette/palette_window.cpp`
- `source/ui/gui.h`
- `source/ui/gui.cpp`
- `source/ui/main_frame.cpp`
- `source/game/materials.h`
- `source/game/materials.cpp`

**Likely verification touch points**
- `D:\redux\data\1310\tilesets.xml`
- `docs/superpowers/specs/2026-04-20-tileset-move-queue-design.md`

### Responsibility split

- `tileset_move_queue.*`: session-only queue state, recent targets, source/target lookup, queued/unqueued queries, batch apply orchestration.
- `tileset_xml_rewriter.*`: parse and rewrite `tilesets.xml`, duplicate detection, range coverage checks, range splitting, sorted insertion, per-section updates.
- `tileset_move_queue_panel.*`: docked staging UI in palette style with `Palette`, `Tileset`, search, view mode, `Apply`, `Discard`.
- `brush_palette_panel.*`: source palette integration, multi-select coordination, `PPM -> Move to`, queued styling, local refresh.
- `virtual_brush_grid.*`: multi-select support, queued visual state, right-click target selection hooks.
- `palette_window.*`, `gui.*`, `main_frame.cpp`: pane creation, lifecycle wiring, lightweight refresh entry points.

### Task 1: Add Session Queue Foundations

**Files:**
- Create: `source/tileset_move_queue/tileset_move_queue.h`
- Create: `source/tileset_move_queue/tileset_move_queue.cpp`
- Modify: `source/ui/gui.h`
- Modify: `source/ui/gui.cpp`
- Test: manual compile verification

- [ ] **Step 1: Define the queue data model**

Create `TilesetMoveQueue` with:

- queued move entry keyed by item id,
- source palette / tileset,
- target palette / tileset,
- `Last used` target history capped at 3,
- queries:
  - `isQueued(itemId)`
  - `queuedTarget(itemId)`
  - `queuedSource(itemId)`
  - `entriesForTarget(palette, tileset)`
  - `clear()`

- [ ] **Step 2: Wire a global queue owner into GUI**

Expose the queue through `GUI` so source palettes and the future staging panel can access one shared session state without rebuilding `Materials`.

- [ ] **Step 3: Verify compile-only foundation**

Run:

```powershell
cmd /c build_windows.bat
```

Expected:

- build succeeds,
- no UI behavior change yet.

### Task 2: Add Multi-Select and Queued Visual State to Source Palettes

**Files:**
- Modify: `source/palette/controls/virtual_brush_grid.h`
- Modify: `source/palette/controls/virtual_brush_grid.cpp`
- Modify: `source/palette/panels/brush_palette_panel.h`
- Modify: `source/palette/panels/brush_palette_panel.cpp`
- Test: manual RAW/Items palette interaction

- [ ] **Step 1: Extend grid selection model**

Add support for multi-selection in `VirtualBrushGrid` with:

- anchor selection,
- ctrl-toggle behavior,
- shift range selection if feasible with current layout,
- accessors for selected brushes / selected item ids.

If shift-range is too invasive for the first pass, implement:

- single click selects one,
- ctrl-click toggles additional entries,
- clicking empty space clears selection.

- [ ] **Step 2: Add queued-state rendering**

Overlay queued state in source palettes by:

- keeping items visible,
- dimming them or drawing a queued badge/border,
- preserving current hover and selected styling.

- [ ] **Step 3: Connect source palettes to queue queries**

Teach `BrushPalettePanel` to ask the queue whether a brush/item id is queued and what its current target is, so RAW and Items palettes can visually reflect in-session move state.

- [ ] **Step 4: Verify UI behavior manually**

Verify:

- source palettes still open and render normally,
- selection still works in list and grid views,
- no freeze/regression in RAW and Items.

Run:

```powershell
cmd /c build_windows.bat
```

Expected:

- build succeeds.

### Task 3: Add `PPM -> Move to` with Last Used Targets

**Files:**
- Modify: `source/palette/controls/virtual_brush_grid.h`
- Modify: `source/palette/controls/virtual_brush_grid.cpp`
- Modify: `source/palette/panels/brush_palette_panel.h`
- Modify: `source/palette/panels/brush_palette_panel.cpp`
- Modify: `source/game/materials.h`
- Modify: `source/game/materials.cpp`
- Test: manual context menu flow

- [ ] **Step 1: Build a target catalog for RAW and Items**

Expose a lightweight way to enumerate valid `Palette -> Tileset` targets from existing tilesets so the context menu can build:

- `Last used`
- `RAW -> ...`
- `Items -> ...`

- [ ] **Step 2: Add right-click move menu**

On selected entries in RAW and Items, open a context menu:

- `Move to`
- top section: up to 3 `Last used` targets,
- full nested target list under RAW and Items.

- [ ] **Step 3: Queue or retarget selected items**

When the user picks a target:

- enqueue all currently selected item ids,
- if already queued, just replace target,
- update `Last used`,
- refresh only affected palettes.

- [ ] **Step 4: Verify repeated move workflow**

Manual verification:

- move several RAW items to one target,
- move another selection with `Last used`,
- retarget already queued items without confirmation,
- ensure source items remain visible but marked queued.

### Task 4: Add Docked Staging Panel

**Files:**
- Create: `source/palette/panels/tileset_move_queue_panel.h`
- Create: `source/palette/panels/tileset_move_queue_panel.cpp`
- Modify: `source/ui/main_frame.cpp`
- Modify: `source/ui/gui.h`
- Modify: `source/ui/gui.cpp`
- Test: manual dock/panel behavior

- [ ] **Step 1: Create palette-style staging panel**

Implement a docked panel that mirrors palette UX:

- `Palette` selector,
- `Tileset` selector,
- search,
- list/grid toggle,
- existing grid rendering reuse where practical,
- `Apply` and `Discard` buttons.

- [ ] **Step 2: Feed panel contents from queue state**

The panel should show queued items grouped by destination:

- `RAW -> Walls`
- `Items -> Armor`

No XML write occurs here.

- [ ] **Step 3: Dock the panel in AUI**

Add the panel like minimap/tool panels:

- normal docked pane,
- not modal,
- hidden by default unless we decide to auto-show when queue becomes non-empty.

- [ ] **Step 4: Verify docked workflow**

Manual verification:

- source palette visible,
- staging panel visible,
- queued items appear under the correct destination view,
- search and view toggle still work inside staging panel.

### Task 5: Add `Discard`

**Files:**
- Modify: `source/tileset_move_queue/tileset_move_queue.h`
- Modify: `source/tileset_move_queue/tileset_move_queue.cpp`
- Modify: `source/palette/panels/tileset_move_queue_panel.cpp`
- Modify: `source/palette/panels/brush_palette_panel.cpp`
- Test: manual queue reset flow

- [ ] **Step 1: Implement queue discard**

`Discard` should:

- clear all queued entries,
- preserve no pending targets,
- clear staging panel contents,
- restore source palettes to unqueued visual state.

- [ ] **Step 2: Verify non-persistent session semantics**

Manual verification:

- queue some items,
- hit `Discard`,
- confirm source palettes fully restore,
- restart app without `Apply`,
- confirm nothing was persisted.

### Task 6: Add XML Rewriter and Batch Apply

**Files:**
- Create: `source/tileset_move_queue/tileset_xml_rewriter.h`
- Create: `source/tileset_move_queue/tileset_xml_rewriter.cpp`
- Modify: `source/tileset_move_queue/tileset_move_queue.cpp`
- Test: manual apply against `D:\redux\data\1310\tilesets.xml`

- [ ] **Step 1: Parse and index target XML**

Implement a rewriter that:

- loads `D:\redux\data\1310\tilesets.xml`,
- locates palette sections and tileset nodes,
- normalizes single item entries and ranges,
- supports lookup:
  - exact item membership,
  - range coverage,
  - current source section.

- [ ] **Step 2: Implement move operations**

For each queued item:

- remove from old section,
- split source ranges if needed,
- skip target duplication if already covered,
- insert into target in sorted numeric position.

- [ ] **Step 3: Save with minimal formatting damage**

Write back using pugixml with the least invasive output practical for touched sections only. Preserve unaffected structure as much as current tooling allows.

- [ ] **Step 4: Run one focused manual verification**

Use a small known batch:

- move single items between sections,
- move an item already present in wrong section,
- move an item out of a source range,
- confirm result ordering is correct.

### Task 7: Refresh Only Affected Panels After Apply

**Files:**
- Modify: `source/tileset_move_queue/tileset_move_queue.cpp`
- Modify: `source/game/materials.cpp`
- Modify: `source/palette/panels/brush_palette_panel.cpp`
- Modify: `source/palette/palette_window.cpp`
- Modify: `source/ui/gui.cpp`
- Test: manual post-apply refresh

- [ ] **Step 1: Limit refresh scope**

After apply:

- refresh RAW and/or Items only if touched,
- refresh staging panel,
- avoid full shell rebuild,
- avoid destroying all palettes.

- [ ] **Step 2: Reload touched material views safely**

Update in-memory palette data to match rewritten XML without reintroducing the current RAW/Walls freeze pattern.

- [ ] **Step 3: Verify post-apply UX**

Manual verification:

- queued items disappear from staging panel,
- items appear only in destination category,
- items no longer appear as queued in source,
- no heavy freeze on apply.

### Task 8: Final Verification Pass

**Files:**
- No new code expected unless fixes are needed
- Test: full manual workflow

- [ ] **Step 1: Full workflow test**

Run this manual sequence:

1. Open RAW palette
2. Select multiple items
3. `PPM -> Move to -> Items -> target`
4. Use `Last used` on a second batch
5. Open staging panel and inspect grouped contents
6. Retarget one queued selection
7. Discard once
8. Rebuild queue
9. Apply
10. Confirm `tilesets.xml` and palettes match expected move-only semantics

- [ ] **Step 2: Build verification**

Run:

```powershell
cmd /c build_windows.bat
```

Expected:

- build succeeds,
- resulting `rme.exe` launches,
- RAW/Items/staging workflows all function.

## Self-Review

### Spec coverage

- Source palettes and multi-select: covered in Tasks 2 and 3
- `Last used`: covered in Task 3
- docked staging panel: covered in Task 4
- `Discard`: covered in Task 5
- `Apply` and XML rules: covered in Task 6
- limited refresh and freeze avoidance: covered in Task 7
- full workflow validation: covered in Task 8

### Placeholder scan

Known intentionally flexible item:

- exact queued visual treatment is still to be finalized during Task 2, but the task goal is explicit and constrained to dimming/badge behavior already agreed in the spec.

### Type consistency

- Queue service is centralized in `TilesetMoveQueue`
- XML write path is isolated in `TilesetXmlRewriter`
- UI integration stays in palette-specific files and a dedicated staging panel
