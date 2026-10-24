# ADR-0004: Window stretch and coordinate model

## Status

Proposed

> **Who may move this to `Accepted`: the user, or `technical-director` on the
> user's explicit confirmation. No other agent, and no skill on its own.**

## Date

2026-10-24

## Last Verified

2026-10-24

## Decision Makers

- User (project owner)
- Claude Code (architecture authoring)
- `godot-specialist` (engine validation; first draft `CONCERNS`, revised)
- `technical-director` review skipped — lean mode

## Summary

The table must stay a fixed 1280×720 logical canvas with a 28-window-pixel menu
bar, while Godot 4.7 now defaults new projects to `canvas_items`/`expand` and
`DisplayServer.ScreenGetScale()` is always 1.0 on Windows. We decided on
`stretch/mode="disabled"` with a manual TableLayout scale/offset, a separate
unscaled CanvasLayer for UI, `Window.Size` as the authoritative window size,
and a measured DPI probe for atlas selection.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 / .NET 8 |
| **Domain** | UI / Rendering / Input |
| **Layer** | Foundation |
| **Knowledge Risk** | **HIGH** — 4.7 changed new-project stretch defaults; 4.6 added dual focus; 4.7 removed the AA feather from `CanvasItem` lines; Windows has a platform-specific `ScreenGetScale` limitation |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `current-best-practices.md`, `modules/ui.md`, `modules/rendering.md`, `modules/input.md` |
| **Post-Cutoff APIs Used** | `display/window/stretch/mode` project setting (4.7 default changed); no new runtime API |
| **Verification Required** | `Window.Size` / `ContentScaleMode` / `ContentScaleFactor` / `GetStretchTransform` probe; `ScreenGetDpi` vs `ScreenGetScale` on Windows; min-size behavior; resize-during-drag; screenshots at five window sizes and OS scales |

> **Evidence boundary.** The engine reference covers the 4.7 stretch default,
> dual focus and line-AA changes, but it has no C# module and no project-settings
> key list. `Window.MinSize`, `InputEventMouse.Position`, `ScreenGetDpi` and the
> exact `project.godot` key names are **NOT SOURCEABLE** there. They must be
> verified against the installed 4.7.2 editor and a real export.
>
> **Platform correction.** `DisplayServer.ScreenGetScale()` is documented as
> returning 1.0 on platforms other than Android, iOS, Web, macOS and Linux
> (Wayland). Windows is an “other platform”; **do not use it as the HiDPI
> factor**.

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001, ADR-0002 (both must be Accepted before implementation) |
| **Enables** | ADR-0006 (input routing), ADR-0007 (Table drawing/atlas/HiDPI) |
| **Blocks** | Table View, Toolbar & Menus and Drag & Drop UI implementation epics |
| **Ordering Note** | Fix the coordinate model before any Table/Toolbar layout code; ADR-0006/0007 consume this ADR |

## Context

### Problem Statement

The GDDs assume a 1280×720 logical table, uniform letterboxing, a 28-window-pixel
menu bar and logical input coordinates. Godot 4.7 defaults new projects to
`canvas_items`/`expand`; if that value is left implicit, the engine scales the
whole canvas and the menu bar becomes 42 px at 1920×1080. The project currently
has no `[display]` section, so the effective value is not pinned.

### Current State

- `project.godot` has only `[application]` and `[rendering]`; there is no
  `[display]` block, `run/main_scene`, or `[dotnet]` section.
- `TableLayout` (Core, pure C#) is designed to compute scale/offset, card rects
  and hit tests from a window size.
- `TableView` is a `Node2D` in Presentation; `CanvasLayer` UI is separate.
- `DragInputAdapter : Node` receives Godot input and calls
  `IDragDropController.OnPointer*`.
- `DisplayServer.ScreenGetScale()` is 1.0 on Windows; `ScreenGetDpi()` is
  implemented there but its relationship to OS scaling is **NOT SOURCEABLE**.
- The first ADR-0004 draft was rejected with `CONCERNS` for five blockers:
  Windows HiDPI, `InputEvent.Position`, `GetVisibleRect`/`ContentScaleFactor`,
  lifecycle wording, and the missing `NotifyLayoutChanged` contract.

### Constraints

- Windows desktop only; no touch/mobile layout.
- 1280×720 logical canvas; uniform scale, never non-uniform.
- Minimum window 1024×576; below that, clamp to 0.8 and crop.
- Menu bar 28 window pixels, never letterbox-scaled.
- Foundation/Core/Feature must not reference Godot (ADR-0001); TableLayout uses
  `System.Numerics.Vector2`.
- `CanvasLayer` UI must remain unscaled and independent of the TableView Node2D
  transform.
- No Camera2D or transformed parent in `Main.tscn`.
- `ContentScaleFactor` must remain 1.0; the root viewport must be in
  `ContentScaleMode.Disabled`.

### Requirements

- Pin the project settings explicitly; do not rely on 4.7 defaults.
- Use `Window.Size` as the authoritative window size.
- Keep menu bar height 28 and compute clearance from the actual height.
- Keep TableLayout pure and testable; convert window→logical only at the
  Presentation boundary.
- Define resize-during-drag behavior for `grab_offset`.
- Define a Windows-valid HiDPI atlas factor; do not use `ScreenGetScale`.
- Capture screenshot evidence at the required sizes.

## Decision

### 1. Project settings

`project.godot` must explicitly set:

```ini
[display]
window/size/viewport_width=1280
window/size/viewport_height=720
window/size/window_width_override=1280   # test-only; actual size asserted
window/size/window_height_override=720
window/size/resizable=true
window/stretch/mode="disabled"
window/stretch/scale=1.0
window/dpi/allow_hidpi=true
```

- Do **not** set `window/stretch/aspect`; it has no effect in `disabled` mode
  and would mislead readers into thinking the engine letterboxes.
- `window_width_override` / `window_height_override` are test-oriented; startup
  asserts the actual `GetWindow().Size` and uses it.
- `window/dpi/allow_hidpi` is present in 4.7.2 but its Windows behavior is
  **NOT SOURCEABLE**; the startup probe records its effect.
- The runtime is the source of truth: after `_Ready`, assert
  `GetWindow().ContentScaleMode == Window.ContentScaleModeEnum.Disabled`,
  `GetWindow().ContentScaleFactor == 1.0`, and
  `GetWindow().GetStretchTransform().Scale == Vector2.One`. If any assertion
  fails, log an error and stop layout initialization rather than silently
  mis-scaling.

### 2. Coordinate spaces

| Space | Unit | Used by |
|---|---|---|
| **Window pixels** | `Window.Size`, `InputEventMouse.Position` | Control/CanvasLayer, menu bar, dialogs, banners |
| **Logical canvas pixels** | 1280×720, `System.Numerics.Vector2`, `LogicalRect` | TableLayout, card rects, hit test, drag math |
| **Device pixels** | physical panel pixels | Atlas selection and screenshot evidence only |

- `GetWindow().Size` is the authoritative window size. `GetVisibleRect()` is a
  cross-check only; if they differ, log and use `Window.Size`.
- `ContentScaleFactor` is not the OS DPI factor. It must stay 1.0 and is never
  used in layout or atlas math.
- `GetFinalTransform()` is the viewport→embedder transform, not a DPI/stretch
  factor; do not use it for atlas selection.

### 3. TableLayout and TableView transform

- `TableLayout.UpdateViewport(windowWidth, windowHeight)` receives
  `GetWindow().Size.X/Y` (not `GetVisibleRect`).
- `scale = max(0.8, min(window_w / 1280, window_h / 720))`; `0.8 = 1024 / 1280`.
- `offset_x = (window_w − 1280×scale) / 2`;
  `offset_y = (window_h − 720×scale) / 2`.
- `TableView` is a `Node2D` at the root of the table; it sets
  `Scale = new Vector2((float)scale, (float)scale)` and
  `Position = new Vector2((float)offset_x, (float)offset_y)`.
- `Main.tscn` must not contain a `Camera2D`; the TableView parent chain must be
  identity. `TableView` children (piles, drag layer, hint highlight) use
  logical coordinates.
- `TableLayout.ToLogical/ToWindow` remain pure and are unit-tested at
  1024×576, 1280×720, 1920×1080, 1600×1000 and 960×540.

### 4. Menu bar and CanvasLayer

- The menu bar is a `Control` inside a `CanvasLayer` with `Layer = 10`.
- The `CanvasLayer` itself must have identity `Transform`, `Scale` and `Offset`;
  `FollowViewportEnabled = false`.
- Target height is 28 window pixels. The actual `MenuBar.Size.Y` is measured and
  used in `menu_bar_clearance`; the theme must guarantee the combined minimum
  size does not exceed 28, otherwise the 28 px contract fails and the gate must
  report it.
- `menu_bar_clearance = offset_y + 38×scale − actual_menu_height`; at
  1024×576 it is 2.4 px, at 1920×1080 it is 29 px. If negative, log an error and
  keep the menu at y=0; never move or compress it at runtime.
- The menu bar is not inside a `SubViewport`, a scaled `Node2D` or a transformed
  `CanvasLayer`.
- Fixed 28 px at 200% OS scaling is a deliberate product tradeoff (physically
  small); it is recorded, not “fixed” by scaling.

### 5. Input and drag

- `DragInputAdapter` uses `_UnhandledInput`, not `_Input`. GUI Controls such as
  the menu bar consume mouse input first; `_UnhandledInput` only sees events the
  UI did not handle.
- Use pattern matching on `InputEventMouse` and read `InputEventMouse.Position`
  (the viewport coordinate space). `InputEvent` itself has no `Position` member.
- Convert with `TableLayout.ToLogical(windowPosition)`; never feed
  `GetLocalMousePosition()` from a TableView child into `ToLogical` (that would
  double-apply the Node2D transform).
- `GetGlobalMousePosition()` is not used: it is CanvasLayer-relative and the
  adapter is not a `CanvasItem`.
- `MenuBar`/dialogs must set `MouseFilter` so they do not pass clicks through to
  the table.

### 6. Resize and drag offset

`Window.SizeChanged` (connected once in `_Ready`, disconnected in `_ExitTree`)
drives one reflow:

```text
1. TableLayout.UpdateViewport(Window.Size.X, Window.Size.Y)
2. TableView.ApplyTransform(scale, offset)
3. DragInputAdapter reads the current pointer window position
4. logical = TableLayout.ToLogical(windowPosition)
5. drag.NotifyLayoutChanged(logical)
```

`NotifyLayoutChanged` is a command, not an event; it replaces the draft's
`OnLayoutChanged` name. It is added to `IDragDropController` and ADR-0002's
contract.

Grab-offset rule during a drag:

- Recompute `grab_offset = logicalPointer − TableLayout.CardRect(source,
  clickedIndex, columnHeight).TopLeft` after the new transform.
- Set `drag_layer_position = logicalPointer − grab_offset`; the card stays under
  the pointer after resize.
- If the pointer is outside the logical canvas, keep the extrapolated logical
  position; do not clamp. Release outside the canvas remains an illegal drop.
- If not dragging, `NotifyLayoutChanged` is a no-op.
- Multiple `SizeChanged` events may arrive; process each with the latest
  `Window.Size`; do not allocate per frame.

### 7. HiDPI and atlas selection

- `DisplayServer.ScreenGetScale()` is **not usable on Windows** (always 1.0).
- Candidate OS factor:
  `os_scale_candidate = DisplayServer.ScreenGetDpi(DisplayServer.ScreenOfMainWindow) / 96.0`.
  The C# binding and its exact Windows semantics are **NOT SOURCEABLE**.
- Startup probe records, at OS 100/125/150/200%:
  `Window.Size`, `GetVisibleRect().Size`, `ScreenGetScale()`, `ScreenGetDpi()`,
  and a captured PNG's actual pixel size.
- Determine `device_pixels_per_window_pixel`:
  1. If `Window.Size` is already physical pixels (PNG/window ratio shows it),
     use `device_factor = 1.0`; `atlas_scale = table_scale`.
  2. If `Window.Size` is logical/DIP and `ScreenGetDpi/96` matches the measured
     OS scale, use `device_factor = max(1.0, ScreenGetDpi/96)`;
     `atlas_scale = table_scale × device_factor`.
  3. If the probe is ambiguous, use `atlas_scale = table_scale`, mark
     TR-table-rendering-layout-007 as **NOT ASSESSED**, and do not claim the
     device-pixel thresholds.
- Atlas selection boundaries are half-open:
  `atlas_scale <= 1.0 → 1x`; `1.0 < atlas_scale <= 2.0 → 2x`;
  `atlas_scale > 2.0 → 3x`. Enable mipmaps and filtered sampling; ADR-0007 owns
  the import presets.
- `ContentScaleFactor` is never part of `atlas_scale`.

### 8. Screenshot and runtime evidence

Required evidence:

- Window sizes: 1024×576, 1280×720, 1920×1080, 1600×1000, 960×540 (cropped).
- OS scales: 100/125/150/200%, recording `Window.Size`, `ScreenGetDpi`,
  `ScreenGetScale`, and the PNG's actual pixel dimensions.
- Measure the menu bar's actual height and the top-row clearance from the PNG.
- Record renderer backend (D3D12 on Windows since 4.6) and HDR state; set HDR
  output off for evidence if the API is available and verified.
- If a condition cannot be established (e.g. OS scale cannot be changed), mark
  that row `NOT ASSESSED` rather than claiming it passed.

### 9. Constants

Define once in TableLayout/Toolbar constants:

- `BaseWidth = 1280`, `BaseHeight = 720`
- `MinScale = 1024 / 1280 = 0.8`
- `TopMargin = 38` (table top row)
- `MenuBarHeight = 28` (target; actual measured height is used for clearance)

No duplicate literals in formulas and tests.

### Architecture Diagram

```text
Window (ContentScaleMode.Disabled, ContentScaleFactor=1.0)
├── Viewport layer 0
│   └── Main (identity)
│       └── TableView : Node2D (Scale=table_scale, Position=offset)
│           ├── piles / card sprites (logical coords)
│           ├── drag layer (logical coords)
│           └── hint highlight (logical coords)
└── CanvasLayer layer 10 (identity, FollowViewportEnabled=false)
    ├── MenuBar (28 window px)
    ├── dialogs / banners
    └── (embedded Window dialogs use engine layer 1024, above this)
```

### Key Interfaces

```csharp
namespace FreeCell.Core;

public sealed class TableLayout
{
    public void UpdateViewport(double windowWidth, double windowHeight);
    public Vector2 ToLogical(Vector2 windowPosition);
    public Vector2 ToWindow(Vector2 logicalPosition);
    // CardRect/HitTest unchanged; coordinates are logical
}

namespace FreeCell.Feature;

public interface IDragDropController
{
    void NotifyLayoutChanged(Vector2 logicalPointer);   // command, not event
    // OnPointer* / IsDragging / events unchanged
}
```

### Implementation Guidelines

**Must:**

- Set the display block explicitly; never rely on 4.7 defaults.
- Use `GetWindow().Size` as the window-size authority.
- Assert `ContentScaleMode.Disabled`, `ContentScaleFactor == 1.0`, and
  `GetStretchTransform().Scale == Vector2.One` at startup.
- Set `GetWindow().MinSize = new Vector2I(1024, 576)` in `_Ready`; keep the
  clamp+crop fallback for platforms/tools that bypass it.
- Call `TableLayout.UpdateViewport` once in `_Ready`/`Bind`; connect
  `Window.SizeChanged` once and disconnect in `_ExitTree`.
- Use `_UnhandledInput` and `InputEventMouse.Position` in `DragInputAdapter`.
- Keep the menu `CanvasLayer` identity and `FollowViewportEnabled=false`.
- Recompute `grab_offset` on resize using the current pointer and the new
  `CardRect`.
- Use the measured DPI probe for `atlas_scale`; use half-open atlas thresholds.
- Record screenshot evidence with actual pixels and OS scale.

**Must never:**

- Use `DisplayServer.ScreenGetScale()` as a Windows HiDPI factor.
- Use `GetVisibleRect()` as the sole window-size authority.
- Set or rely on `ContentScaleFactor` other than 1.0.
- Put the menu inside a scaled Node2D, SubViewport or transformed CanvasLayer.
- Add a Camera2D or a non-identity parent transform in `Main.tscn`.
- Use `_Input` for table drags (it would steal menu clicks).
- Feed `GetLocalMousePosition()` into `ToLogical`.
- Clamp drag positions at the canvas edge.
- Let a negative menu clearance silently overlap the top row.

## Alternatives Considered

### Alternative 1: `canvas_items` + `expand`

- **Description**: Use Godot 4.7 defaults and let the engine scale the canvas.
- **Pros**: Less manual geometry; engine handles stretch.
- **Cons**: The whole UI scales; the menu bar becomes 42/56 px at 1.5/2.0 and
  violates the 28 px contract; TableLayout formulas and GDDs need rewriting.
- **Estimated Effort**: Medium, but breaks a fixed GDD requirement.
- **Rejection Reason**: The menu bar must stay 28 window pixels.

### Alternative 2: `viewport` + `keep` aspect

- **Description**: Render to a fixed 1280×720 viewport; Godot letterboxes it.
- **Pros**: Engine letterboxing; no manual offset.
- **Cons**: UI is inside the viewport and scales with it; input mapping and
  menu bar height change; hidden `ContentScaleFactor` interactions.
- **Estimated Effort**: Medium.
- **Rejection Reason**: Same menu-bar scaling issue and a larger rewrite.

### Alternative 3: SubViewport for the table

- **Description**: Render the table into a SubViewport and letterbox it inside a
  Control.
- **Pros**: Strong separation of table and UI.
- **Cons**: Input mapping, focus, texture filtering and viewport texture overhead;
  more code for a small game.
- **Estimated Effort**: High.
- **Rejection Reason**: The Node2D transform gives the same result with less
  machinery.

## Consequences

### Positive

- Menu bar stays exactly 28 window pixels at every window size.
- TableLayout stays pure C# and unit-testable; TableView is a thin transform.
- `Window.Size` is a single, testable window-size authority.
- Resize-during-drag has a defined, non-jumping rule.
- Windows HiDPI is handled by measurement rather than a broken API.

### Negative

- Manual scale/offset and transform synchronization.
- The 28 px menu bar is physically small at 200% OS scaling.
- The DPI probe is extra startup work and requires screenshot verification.
- Any parent transform or Camera2D breaks the coordinate model silently.

### Neutral

- `stretch/aspect` is omitted because it has no effect in disabled mode.
- `GetVisibleRect` is a cross-check, not the authority.
- `atlas_scale` may equal `table_scale` on Windows if `Window.Size` is already
  physical.
- `NotifyLayoutChanged` is a command, not a domain event.

## Risks

| Risk | Mitigation |
|---|---|
| 4.7 default stretch scales the menu | Explicit `stretch/mode="disabled"` + startup assertions |
| Windows `ScreenGetScale` returns 1.0 | Use `ScreenGetDpi/96` probe; fallback `atlas_scale=table_scale`; mark NOT ASSESSED if ambiguous |
| `ContentScaleFactor` ≠ 1 breaks input | Startup assert; never set it |
| `GetVisibleRect` differs from Window.Size | Window.Size authoritative; log mismatch |
| MenuBar theme minimum > 28 | Pin theme; measure actual height; fail the gate if > 28 |
| Resize during drag jumps the card | Recompute grab_offset from current pointer and new CardRect |
| Multiple resize events per frame | Process each with latest Window.Size; no per-frame allocation |
| CanvasLayer accidentally scaled | Identity transform/scale/offset; FollowViewportEnabled=false; test screenshots |
| Camera2D or parent transform | Main.tscn review; no Camera2D; identity parent chain |
| `_Input` steals menu clicks | `_UnhandledInput` + Control MouseFilter |
| OS scale test is a no-op | Record Window.Size, ScreenGetDpi, PNG pixels; mark NOT ASSESSED if unchanged |
| HDR/D3D12 changes screenshot colors | Record backend; disable HDR for evidence if verified |
| Device-pixel thresholds not met | Probe and measure; if ambiguous, do not claim TR-007 |

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `table-rendering-layout.md` | 1280×720 logical canvas, uniform scale, letterbox, min 1024×576 (TR-table-rendering-layout-001) | `Window.Size` + TableLayout scale/offset + min size |
| `table-rendering-layout.md` | Card geometry and hit test in logical coordinates (TR-002/003) | TableView Node2D transform; TableLayout pure rects |
| `table-rendering-layout.md` | Drawing without runtime lines; HiDPI atlas (TR-004/007) | No lines; measured `atlas_scale` with half-open tiers |
| `table-rendering-layout.md` | Resize once; drag offset rescale (TR-008) | `SizeChanged` reflow + `NotifyLayoutChanged` |
| `toolbar-menus.md` | 28 window px menu bar; no letterbox scaling (TR-toolbar-menus-001) | CanvasLayer identity + measured height/clearance |
| `toolbar-menus.md` | Geometry evidence at 1024×576/1920×1080/1600×1000 (TR-009) | Screenshot matrix and clearance formula |
| `drag-drop-interaction.md` | 1:1 follow and logical coordinates (TR-drag-002/008) | InputEventMouse.Position + TableLayout.ToLogical |
| `drag-drop-interaction.md` | Resize during drag keeps the card under the pointer (TR-008) | `NotifyLayoutChanged` + grab_offset recompute |
| `hint-system.md` | Highlight drawn in table coordinates (TR-hint-006) | Hint highlight child of TableView; logical coords |

## Performance Implications

- **CPU**: One reflow per resize event; no per-frame layout work. Input
  conversion is O(1). Screenshot/probe work is startup-only.
- **Memory**: No per-frame allocations; TableLayout reuses constants.
- **Load Time**: Negligible; one startup assertion/probe.
- **Network**: Not applicable.

## Migration Plan

No code exists. The implementation order:

1. Add the display block and runtime assertions in `GameBootstrap._Ready`.
2. Set `Window.MinSize` and call `TableLayout.UpdateViewport` once.
3. Apply the TableView Node2D transform in one place.
4. Connect `Window.SizeChanged`; implement `NotifyLayoutChanged`.
5. Add the DPI/atlas probe and screenshot evidence.
6. Update Table/Toolbar/Drag GDD formulas and AC to `Window.Size` and
   `atlas_scale`.
7. Run the five-window-size and OS-scale matrix.

**Rollback plan**: If manual letterboxing proves unworkable, switch to a
SubViewport table or `canvas_items` with a compensated UI; either is a new ADR
and requires GDD revisions. The current ADR would be superseded.

## Validation Criteria

- `project.godot` has the display block; startup asserts
  `ContentScaleMode.Disabled`, `ContentScaleFactor == 1.0`,
  `GetStretchTransform().Scale == Vector2.One`.
- At 1024×576: scale 0.8, offset (0,0), menu height 28, top-row clearance 2.4.
- At 1920×1080: scale 1.5, offset (0,0), menu height 28 (not 42), clearance 29.
- At 1600×1000: scale 1.25, offset (0,50), menu over letterbox.
- At 960×540: scale 0.8, negative offset, crop, warning logged.
- `Window.MinSize` is 1024×576; a smaller platform window still clamps/crops.
- Input hit test, drag layer and hint highlight align at every size.
- Resize during drag keeps the card under the pointer; release outside is
  illegal; no jump at the canvas edge.
- `ScreenGetScale` is never used as a Windows factor; `ScreenGetDpi/96` probe
  results are recorded; ambiguous probe marks TR-007 `NOT ASSESSED`.
- Screenshots at the five sizes and four OS scales are retained with actual
  pixel dimensions, menu height and renderer backend.
- No Camera2D; CanvasLayer identity; menu `MouseFilter` consumes clicks;
  drag uses `_UnhandledInput`.

## Related

- ADR-0001 — layered boundaries and pure C# core
- ADR-0002 — session composition, model ownership, and event contract
- ADR-0003 — model snapshot and serialization schema
- ADR-0006 — input routing and 4.6 dual focus
- ADR-0007 — Table drawing, atlas and HiDPI
- `docs/architecture/architecture.md` §3.4, §4.6, §5.9–5.12
- `design/gdd/table-rendering-layout.md`, `toolbar-menus.md`,
  `drag-drop-interaction.md`, `hint-system.md`
- `docs/registry/architecture.yaml`
