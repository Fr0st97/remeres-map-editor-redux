# Tileset Move Queue Design

Date: 2026-04-20
Project: RME tileset category reassignment workflow
Scope: RAW and Items palette move queue, staging panel, deferred apply to `data/1310/tilesets.xml`

## Goal

Replace the current manual workflow of moving items between tileset categories with an in-editor tool that lets the user:

- select one or more items in `RAW` or `Items`,
- assign them to a target `Palette -> Tileset` using `PPM -> Move to`,
- review all pending moves in a docked staging panel,
- apply all moves in one batch to `D:\redux\data\1310\tilesets.xml`.

This is a true move workflow, not copy:

- after `Apply`, an item should exist only in its new target category,
- it should no longer remain in its previous category.

## Source of Truth

Permanent storage:

- `D:\redux\data\1310\tilesets.xml`

Not used for this workflow:

- `items.xml`

Session-only state:

- in-memory staging queue

Rule:

- if the user closes the program or discards changes before `Apply`, nothing is written to `tilesets.xml`.

## Main UX

### 1. Source palettes

The workflow starts from the existing `RAW` and `Items` palettes.

User action:

- multi-select one or more tiles,
- right-click,
- use `Move to`.

### 2. Context menu

`PPM -> Move to` should contain:

- `Last used`
- up to 3 recently used targets
- full target navigation by `Palette -> Tileset`

Example:

- `Last used -> RAW -> Walls`
- `Last used -> Items -> Armor`
- `Move to -> RAW -> Others`
- `Move to -> Items -> Armor`

Rules:

- selecting a target immediately places the selected items into the staging queue,
- if an item is already queued, choosing a new target changes the target directly,
- no confirmation is required for target reassignment,
- `Last used` is global for the tool and stores the latest 3 targets.

### 3. Source palette presentation

Queued items should remain visible in their source palette, but be visually marked.

Preferred behavior:

- item remains in place,
- item is greyed out and/or marked as queued,
- item is still inspectable,
- item should not feel like it disappeared from the current context.

This avoids loss of orientation during bulk work.

### 4. Staging panel

The queue must be shown in a normal docked side panel, visually matching existing panels like `Palette` or `Minimap`.

It must not be a separate modal or a large floating workflow window.

The staging panel should behave like a regular palette panel:

- `Palette` selector
- `Tileset` selector
- search
- list/grid toggle
- item grid/list view
- action buttons: `Apply`, `Discard`

Content model:

- the panel shows queued items grouped by destination,
- the user can browse them the same way as in a normal palette,
- for example:
  - `RAW -> Walls` shows all queued items targeting RAW/Walls,
  - `Items -> Armor` shows all queued items targeting Items/Armor.

## Queue Semantics

The staging queue is a deferred batch of move operations.

Each queued entry should conceptually contain:

- item id / brush reference
- source palette
- source tileset
- target palette
- target tileset

Behavior:

- queued items are not yet written to XML,
- queued items affect UI presentation immediately,
- queued items can be retargeted freely,
- `Discard` clears the queue and restores the visible state,
- `Apply` writes all queued moves to `tilesets.xml`.

## XML Write Rules

All writes must follow the existing rewrite rules from `instrukcje_rewrite.txt`.

### Required behavior

1. Edit only `D:\redux\data\1310\tilesets.xml`
2. Work inside the correct target section
3. Preserve numeric order inside each section
4. Avoid duplicates
5. Respect existing ranges `fromid/toid`
6. If item exists in the wrong section, remove it there and add it to the new one
7. Do not append blindly at the end
8. Avoid unnecessary formatting damage outside touched XML fragments

### Move behavior

For each applied queue entry:

1. locate current source ownership,
2. remove the item from the old location,
3. insert it into the new location in sorted order,
4. skip duplicate insertion if already correctly covered by target rules,
5. preserve section ordering after modification.

## Range Handling

The tool should support items covered by ranges.

Example:

- source contains `<item fromid="5000" toid="5100" />`
- user moves `5055` elsewhere

Expected long-term behavior:

- source range is split as needed,
- result becomes:
  - `5000-5054`
  - `5056-5100`

This is important because many categories may use ranges and the tool should not become unreliable on common real-world data.

## Performance Requirements

The tool must avoid heavy synchronous rebuild behavior during normal staging work.

### During queue operations

Allowed:

- in-memory queue updates
- lightweight palette refresh
- marking queued items in visible panels

Avoid:

- reparsing `tilesets.xml` on every click
- rebuilding the entire shell after every move
- refreshing all palettes when only one or two are affected

### Recommended approach

- parse/index `tilesets.xml` once for the staging session,
- keep cached lookup structures:
  - `id -> current section`
  - range ownership info
  - palette/tileset metadata
- update only in-memory structures during staging,
- write once on `Apply`,
- refresh only affected palette panels after save.

## Refresh Rules

After queue edits:

- update source palette appearance,
- update staging panel contents,
- avoid full application rebuild.

After `Apply`:

- write `tilesets.xml`,
- refresh only affected palette panels,
- avoid rebuilding the full shell unless strictly necessary.

Special caution:

- Walls and RAW are already known to be sensitive to heavy refresh logic and freezes,
- workflow should isolate refresh to touched panels/categories only.

## Non-Goals

The first version does not need:

- drag and drop between panels,
- third-level categories,
- persistent queue across restarts,
- automatic background save,
- a large separate management window.

## Proposed UI Summary

### Source palettes

- multi-select support
- `PPM -> Move to`
- `Last used` with 3 entries
- queued item visual state

### Staging panel

- docked side panel
- same visual language as normal palette panels
- browse by `Palette -> Tileset`
- search and view mode support
- `Apply`
- `Discard`

## Open Implementation Questions

These are not blockers for the concept, but should be finalized before coding:

1. exact multi-select interaction model in existing palette controls
2. exact queued item styling in source panels
3. whether staging panel also allows removing items from queue via context menu
4. whether `Apply` should show a short summary of:
   - moved items
   - skipped items
   - split ranges

## Recommended Direction

Build this as a staged move system with:

- session queue in RAM,
- docked staging palette panel,
- cached XML index,
- single batch apply,
- local panel refresh only.

This gives the best balance of:

- speed,
- safety,
- low freeze risk,
- usability for repeated category cleanup work.
