# Palette: Global Search, View Toggle, RAW "Unused" Mode (Design)

Date: 2026-04-20

## Summary

Add a fixed (non-scrolling) search bar to brush palettes that supports:

- Global search across all tilesets in the current palette page set.
- Search by exact ID or by name (case-insensitive phrase contains).
- Runtime toggle between List and Grid views.
- RAW-only "Only unused" mode that hides items already used by other brush systems.

The feature targets the wxWidgets palette UI and integrates with existing brush selection flow (`g_gui.SelectBrush`, `BrushManager`, `PaletteWindow`).

## Goals

- Make it fast to find items/brushes without knowing which tileset they belong to.
- Keep the search UI anchored below the tileset selector and above the first result; it must not move when scrolling results.
- Provide a compact grid view that can show an ID label for RAW-like entries.
- Provide a RAW-only filter to show items not yet used by wall/ground/border/doodad/etc brushes to support a "brush creator" workflow.

## Non-goals

- Full-text fuzzy search, ranking, stemming, or language-aware tokenization.
- Cross-palette search (e.g., one search box spanning Terrain + RAW simultaneously).
- Persisting the new List/Grid toggle into `config.toml` (initially runtime-only).
- Reworking non-brush palettes (House/Waypoint/Creature) beyond keeping existing behavior.

## UX / Behavior Spec

### Placement

Within each `BrushPalettePanel` (Terrain/Doodad/Collection/Item/RAW):

- The tileset selector remains at the top of the panel (currently a `wxChoicebook`-like UX).
- Immediately below it, a fixed "palette toolbar row" is added containing:
  - Search input control with a right-side dropdown arrow to choose search mode.
  - List/Grid toggle.
  - RAW-only "Only unused" checkbox (only shown for RAW palette).
- Below this row, the scrollable results view (list/grid) is shown.

Scrolling occurs only inside the results control, so the toolbar row remains stationary.

### Search Modes

The search control supports two modes selectable via dropdown (arrow on the right):

1. **ID**
   - Input is numeric (`0-9`), optionally allowing whitespace trimming.
   - Matching is **exact**: input `106` matches only the entry with ID 106.
   - If input is not a valid integer, results are empty (or show "invalid id" prompt).

2. **Name**
   - Matching is case-insensitive.
   - Matching is **phrase contains** (no token split):
     - Query `wall` matches any entry whose display name contains `wall`.
     - Query `brick wall` matches entries whose display name contains the exact substring `brick wall` (spaces included).

Debounce: apply the filter with ~150ms delay after last keystroke to avoid rebuilding results on every keypress.

### Global Search Across Tilesets

When the query is empty:

- The view shows the currently selected tileset content.

When the query is non-empty:

- The view switches to a "Search results" mode that aggregates matches across **all tilesets** in that palette category.
- Results include items from every tileset page that would normally be reachable via the tileset selector.
- Clearing the query restores the previously selected tileset.

No attempt is made to auto-navigate to a specific tileset unless the user clears the query (see "Selection" below).

### List/Grid Toggle

- List mode: show icon + `brush->getName()` text (existing behavior).
- Grid mode:
  - Show icon card tiles.
  - For entries with a reliable numeric ID (primarily `RAWBrush`), show a compact ID label near the bottom edge of the card to avoid consuming vertical space.

The toggle is runtime-only:

- It updates the current view immediately (including search results view).
- It does not write to `g_settings` / `config.toml` in the initial iteration.

### RAW "Only unused" Mode

Only visible for the RAW palette.

When enabled:

- Hide RAW items that appear to already be "used" by other brush systems.
- "Used" is determined from existing editor metadata stored in `ItemEditorData` for the item id.

Definition of "used" (v1):

- `ItemEditorData.brush != nullptr` (assigned by ground/wall/border/table/carpet/etc loaders)
- OR `ItemEditorData.doodad_brush != nullptr` (assigned by doodad loader and by some tileset loading flows)
- OR `ItemEditorData.collection_brush != nullptr` (assigned when items are placed into collection workflows)

Notes:

- `raw_brush` is always present for RAW items and must not be treated as "used" by itself.
- The filter applies both to normal tileset view and to the aggregated search results view for RAW.

### Selection / Activation

- Clicking a result selects it and calls `g_gui.SelectBrush(brush, palette_type)` as today.
- If a selection occurs in "Search results" mode:
  - The selection remains visible in results.
  - The panel records the originating tileset name/index for that entry.
  - When the query is cleared, the tileset selector switches to the recorded tileset, and the same brush remains selected.

## Architecture / Implementation Design

### Problem in Current Structure

`BrushPalettePanel` currently uses a `wxChoicebook` as the tileset selector + page container. This makes it difficult to insert a fixed toolbar row between the selector control and the scrolling results area, because the choicebook manages both.

### Proposed UI Restructure (Recommended)

Refactor `BrushPalettePanel` to explicitly own:

- `wxChoice* tileset_choice` (dropdown for tilesets)
- A fixed toolbar row (`wxPanel` + `wxBoxSizer`):
  - `wxTextCtrl` (or `wxSearchCtrl`) with a mode dropdown.
  - Toggle control for List/Grid.
  - `wxCheckBox` for RAW-only "Only unused".
- A content container:
  - Normal mode: a `BrushPanel` for the selected tileset.
  - Search mode: a dedicated `BrushPanel`-like view that uses a dynamic brush list model.

Use `wxSimplebook` (or a sizer with `Show()/Hide()`) for switching between:

- Tileset view
- Search results view

This keeps the toolbar fixed and avoids duplicating search controls per tileset page.

### Brush List Model

Generalize the underlying list source for the results view so it can represent:

- Tileset-backed lists (`TilesetCategory::brushlist`)
- Search results (a filtered, aggregated vector)

Introduce a small model interface (names illustrative):

- `struct BrushListEntry { Brush* brush; std::string display_name; std::optional<uint16_t> id; std::string tileset_name; };`
- `using BrushList = std::vector<BrushListEntry>;`

The results view (`VirtualBrushGrid`) should be extended to accept either:

- a `TilesetCategory*` (existing path), or
- a pointer/ref to `BrushList` with stable lifetime during viewing.

This change is intentionally scoped to palette UI only.

### Rendering ID in Grid Mode

`VirtualBrushGrid` currently draws list text only in List mode. Add in Grid mode:

- If `entry.id` exists, draw a small text label near the bottom of the card.
- The label is single-line, compact, and does not change card height dramatically.

For tileset-backed mode, `id` is populated only for `RAWBrush` (via `RAWBrush::getItemID()`), and otherwise omitted.

### Computing RAW "Used" Set

When RAW palette is active:

- Build a `std::vector<bool>` or `std::unordered_set<uint16_t>` marking used item ids.
- Source: iterate `g_item_definitions.allIds()` and check `editorData()` fields:
  - `brush`, `doodad_brush`, `collection_brush`
- Cache the computed set and only rebuild when materials/brush definitions change (initially: rebuild on `g_gui.RebuildPalettes()` or on palette refresh).

This keeps filtering cheap while typing.

## Performance Considerations

- Debounce search input (150ms).
- Cache lowercased `display_name` for each entry (for Name mode).
- For ID mode: direct lookup map `id -> entry index` for search results.
- Avoid rebuilding tile textures; the rendering path already caches sprite textures.

## Edge Cases

- Query whitespace: trimmed.
- ID mode with leading zeros: parse as integer; exact id comparison.
- Empty results: show an empty state message ("No matches").
- Brushes without names or sprites: show placeholder name and skip sprite safely (existing behavior).

## Testing / Validation

Manual validation checklist:

- In each brush palette (Terrain/Doodad/Collection/Item/RAW), the toolbar row stays fixed while scrolling results.
- Global search finds items across tilesets without changing tileset selection.
- Clearing query restores previous tileset selection.
- ID mode: `106` only shows ID 106 (not 1106, etc).
- Name mode: phrase-contains works; `brick wall` only matches entries containing that exact substring.
- List/Grid toggle works without recreating palettes.
- RAW "Only unused" hides items used by wall/ground/border/doodad/collection; shows items with all of those pointers null.

## Rollout Plan

Deliver in a single feature branch, but implement in steps:

1. UI restructure + fixed toolbar row.
2. Search results aggregation and matching rules.
3. List/Grid toggle including grid ID label.
4. RAW "Only unused" filter with cached used-set.

