# ADR-0001: Layered boundaries and pure C# core

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
- `godot-specialist` (engine validation)
- `technical-director` review skipped — lean mode

## Summary

The 10 MVP systems need a dependency model that keeps rules, state, undo, hint
and statistics testable without a Godot scene tree, while letting presentation
adapters use Godot 4.7.2 C# without leaking engine types into gameplay code. We
decided on one Godot C# project with strict folder/namespace layers
(`Foundation → Core → Feature → Presentation → App`), a pure C# boundary for the
first three layers, a `TableLayout` (Core) + `TableView` (Presentation) split,
and a single `GameBootstrap : Node` composition root.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 / .NET 8 |
| **Domain** | Core |
| **Layer** | Foundation |
| **Knowledge Risk** | LOW for this decision — it is engine-agnostic and uses no post-cutoff API. Project-level risk remains HIGH for UI/Input/Rendering, handled by ADR-0004/0006/0007. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `current-best-practices.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | `dotnet build` with `Godot.NET.Sdk/4.7.2` + `net8.0`; headless boot; architecture check; export filtering; `project.godot` assembly/main-scene keys |

`Domain: Core` names the engine area this decision touches (C#/project
structure), not a confinement to the Core layer. The decision is cross-layer;
`Layer: Foundation` records how early it must exist.

> **Evidence note.** The installed editor reports
> `4.7.2.stable.mono.official.ed1daf0bf`; its `GodotPlugins.runtimeconfig.json`
> targets `net8.0`; .NET SDK `8.0.426` is installed. A temporary
> `Godot.NET.Sdk/4.7.2` project built with 0 warnings/0 errors and ran headless
> on 2026-10-24. These are local verification results, not engine-reference
> conclusions. The C# toolchain itself is **NOT SOURCEABLE** in the current
> engine-reference library (there is no `modules/csharp.md`).

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | None |
| **Enables** | ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0007, ADR-0008, ADR-0009 |
| **Blocks** | All MVP implementation epics — they need the layer rules and project layout before the first story |
| **Ordering Note** | Write this ADR before ADR-0002; ADR-0002 assumes these boundaries and the composition-root contract |

## Context

### Problem Statement

The GDDs define 10 interacting systems, but the repository has no C# project,
no folder boundaries and no dependency rules. Without this decision, each
implementation story can invent its own references: rules code can grow Godot
node dependencies, Table Rendering can become a Feature dependency, and the
pure logic cannot be tested without launching a scene tree.

### Current State

- `docs/architecture/architecture.md` v1 defines the intended layers, module
  ownership, data flow and API boundaries.
- No ADRs exist yet; this is ADR-0001.
- `project.godot` exists with `config/name="空当接龙：经典复刻"` and Forward+,
  but no `[dotnet]` section and no `run/main_scene`.
- No `FreeCell.csproj`, no `src/` code, no `tests/` test project.
- `.gitignore` currently ignores `*.csproj` and `*.sln` because of unanchored
  Unity-section rules. A future Godot C# project would silently not enter
  version control until this is fixed.
- Godot 4.7.2 mono and .NET SDK 8.0.426 are installed; a temporary smoke test
  built and ran successfully.
- `project.yaml` has the pinned editor path but its `commands.run` / `test`
  still call bare `godot`, which is not on PATH.

### Constraints

- Solo developer; MVP target 4–5 weeks.
- Godot 4.7.2 / .NET 8 / C# / Forward+; Windows desktop only.
- Performance targets from `.claude/docs/technical-preferences.md`: 60 fps,
  16.6 ms frame budget, 200 2D draw calls, 512 MB memory ceiling.
- MVP has no animation, audio, settings or accessibility systems.
- Godot's project root is the repository root, so `docs/`, `design/`,
  `production/`, `tools/`, `studio-tests/` and `tests/` are all under `res://`;
  export filtering must be explicit.
- The GDDs require pure functions and deterministic behavior; unit tests must
  not require a rendered game.

### Requirements

- All 10 MVP systems must have a single, unambiguous home folder and namespace.
- Foundation/Core/Feature must not reference Godot or GodotSharp.
- Presentation may use Godot types; Feature must not reference Presentation.
- Table Rendering must provide geometry to Drag without making Feature depend
  on a Godot view.
- There must be one composition root; no Autoload singleton or static service
  locator.
- Test code must not be compiled into the game assembly.
- A static architecture check must fail on boundary violations before the first
  implementation story.

## Decision

### 1. One Godot C# project, strict folder/namespace layers

Use one project at the repository root:

```text
FreeCell.csproj                 # Godot.NET.Sdk/4.7.2, net8.0, assembly "FreeCell"
project.godot                   # [dotnet] assembly_name = "FreeCell"
src/
  Foundation/                   # namespace FreeCell.Foundation
    Cards/
    Values/
  Core/                         # namespace FreeCell.Core
    Deal/
    Rules/
    Layout/
  Feature/                      # namespace FreeCell.Feature
    State/
    Undo/
    Hint/
    Statistics/
    Drag/
  Presentation/                 # namespace FreeCell.Presentation
    Table/
    Toolbar/
    Input/
  App/                          # namespace FreeCell.App
    GameBootstrap.cs
    Main.tscn
tests/
  unit/
  integration/
```

- `FreeCell.csproj` is the only Godot C# project. It compiles `src/` code.
- Test code lives under `tests/` and is built by a separate test project; it is
  not part of the game assembly.
- File names follow `project.yaml` (`PascalCase`), with file-scoped namespaces.

### 2. Dependency direction

```text
App
  ├──> Presentation
  │       ├──> Feature
  │       │       ├──> Core
  │       │       │       └──> Foundation
  │       │       └──> Foundation
  │       └──> Core
  └──> Foundation
```

The allowed matrix:

| Consumer | May depend on |
|---|---|
| App | Presentation, Feature, Core, Foundation |
| Presentation | Feature, Core, Foundation |
| Feature | Core, Foundation |
| Core | Foundation |
| Foundation | BCL / .NET only |

Forbidden:

- Foundation → Core, Feature, Presentation or App.
- Core → Feature, Presentation or App.
- Feature → Presentation or App.
- Presentation → App.
- Any circular dependency between modules.
- Feature referencing a Godot node type, signal or `NodePath`.

### 3. Pure C# boundary

Foundation, Core and Feature must not reference Godot:

- No `using Godot;`, `Godot.` or `global::Godot` in those folders.
- No `Node`, `Node2D`, `Control`, `Resource`, `PackedScene`, `InputEvent`,
  `Godot.Vector2`, `Godot.Rect2`, `Godot.Color` or other engine types.
- Pure geometry uses `System.Numerics.Vector2` and a project-owned
  `LogicalRect`; collections use `System.Collections.Immutable.ImmutableArray<T>`.
- Conversion between Godot and pure types happens only in Presentation
  adapters.
- Serialization uses explicit DTOs; do not serialize `System.Numerics.Vector2`
  directly (its `X`/`Y` are fields and are skipped by default `System.Text.Json`
  settings). ADR-0003 owns the schema.

### 4. Table Rendering split

`Table Rendering & Layout` is one GDD system, implemented as two modules:

| Module | Layer | Responsibility |
|---|---|---|
| `TableLayout` | Core | Pure `scale`, `offset`, card rects, cascade, `HitTest`, coordinate transforms |
| `TableView : Node2D` | Presentation | Sprites/atlas, drag-layer visuals, hint highlight, source hiding, redraw |

`TableView` subscribes to State/Undo/Drag events. Drag itself is pure C# and
does not reference `TableView`. The Godot input entry point is a separate
`DragInputAdapter : Node` in Presentation, which converts `InputEvent` and
window coordinates into pure pointer calls.

This resolves the otherwise inverted `Table → State` dependency: the pure
layout never reads State; the view reacts to events.

### 5. Composition root

`GameBootstrap : Node` is the **only place that constructs concrete services**:

- It is the single Godot node in the App layer.
- It constructs the model, core services, feature services and the pure C#
  `GameSession` container; the Presentation adapters are scene nodes it binds
  rather than constructs.
- `GameSession` receives already-constructed services. It only holds and wires
  them; it must never `new` a gameplay service. This keeps `GameBootstrap` the
  sole composition root while allowing a pure C# session container.
- It constructs services in `_Ready` using the initial window size. The size is
  not final: `TableLayout.UpdateViewport` and the TableView transform are
  refreshed on `Window.SizeChanged` (ADR-0004). Children must not use services
  until `Bind()` is called.
- Presentation adapters subscribe in `Bind()` and unsubscribe in
  `_ExitTree()`.
- No Autoload, no static service locator, no mutable global state.

`project.godot` must set:

```ini
[application]
run/main_scene="res://src/App/Main.tscn"

[dotnet]
project/assembly_name="FreeCell"
```

The display name (`config/name`) may remain Chinese; the assembly name is ASCII.

### 6. Test and build boundary

The main project must exclude test sources:

```xml
<ItemGroup>
  <Compile Remove="tests/**/*.cs" />
</ItemGroup>
```

- Pure unit tests live in a separate test project under `tests/unit/`.
- Integration tests live under `tests/integration/`.
- The game assembly must never contain test classes or test framework
  references.
- ADR-0005 selects the runner. If it selects a Godot-integrated C# runner that
  requires test code in a Godot assembly, it must create a dedicated test
  assembly and amend or supersede this constraint; it must not silently compile
  tests into `FreeCell.dll`.

### 7. Static architecture check

A static check must exist before the first implementation story. It must fail
(non-zero exit), not warn, when:

- Foundation/Core/Feature contain `using Godot`, `Godot.` or `global::Godot`.
- `src/App/GameSession.cs` contains any Godot reference; it is a pure C#
  container.
- Foundation references Core/Feature/Presentation/App.
- Core references Feature/Presentation/App.
- Feature references Presentation/App.
- A Presentation `Bind()` signature references a `FreeCell.App` type,
  including `GameSession`.
- A new top-level layer folder appears without being added to the check.

It must exempt only `src/App/GameBootstrap.cs` (the single App Godot node) and
`src/Presentation/` for the Godot-reference rule. The check may be a
test-project test or a `.claude/scripts/` script; ADR-0005 decides the runner.
Until it exists, code review enforces the same rules and the first story must
not close without it.

### 8. Project and export hygiene

- `.gitignore` must un-ignore `FreeCell.csproj`, `FreeCell.sln` and
  `tests/**/*.csproj`; the Unity-section `*.csproj` / `*.sln` rules are
  unanchored and otherwise hide all C# project files.
- The export preset must exclude `docs/`, `design/`, `production/`, `tools/`,
  `studio-tests/`, `tests/` and `.claude/`, or use a dependencies-only export.
  The default behavior is **NOT SOURCEABLE** in the engine reference; verify
  with a real export before release.
- The exported build must launch the main scene and run `GameBootstrap`.

### Architecture Diagram

```text
┌─────────────────────────────────────────────────────────────┐
│ App: GameBootstrap : Node (composition root)                │
├─────────────────────────────────────────────────────────────┤
│ Presentation: TableView : Node2D, Toolbar : Control,        │
│               DragInputAdapter : Node                      │
├─────────────────────────────────────────────────────────────┤
│ Feature: State, Undo, Hint, Statistics, Drag (pure C#)      │
├─────────────────────────────────────────────────────────────┤
│ Core: Deal, Rules, TableLayout (pure C#)                    │
├─────────────────────────────────────────────────────────────┤
│ Foundation: CardDeckModel, Shared Primitives (pure C#)      │
├─────────────────────────────────────────────────────────────┤
│ BCL / .NET 8                                                │
└─────────────────────────────────────────────────────────────┘
```

### Key Interfaces

These are layer markers, not the full contracts. Full signatures live in
`docs/architecture/architecture.md` §5 and the GDDs.

```csharp
namespace FreeCell.Foundation;

public interface ICardDeckModel { /* pure C# */ }


namespace FreeCell.Core;

public interface IRulesMoveValidator { /* pure C# */ }

public sealed class TableLayout
{
    // pure scale/offset/card rect/hit-test geometry
}


namespace FreeCell.Feature;

public interface IGameStateMachine { /* pure C# */ }
public interface IUndoHistory { /* pure C# */ }
public interface IHintSystem { /* pure C# */ }
public interface IStatistics { /* pure C# */ }
public interface IDragDropController { /* pure C# input state machine */ }


namespace FreeCell.Presentation;

public partial class TableView : Node2D { /* Godot drawing adapter */ }
public partial class ToolbarController : Control { /* Godot UI adapter */ }
public partial class DragInputAdapter : Node { /* InputEvent -> pure pointer calls */ }


namespace FreeCell.App;

public partial class GameBootstrap : Node { /* composition root */ }
```

### Implementation Guidelines

**Must:**

- Place every module in the folder and namespace of its layer.
- Inject dependencies through constructors; `GameBootstrap` is the only place
  that constructs concrete services.
- Pass constructed services to the pure C# `GameSession`; it holds and wires
  them but never constructs them.
- Keep Godot types at the Presentation boundary.
- Use `System.Numerics.Vector2` and `LogicalRect` in pure layers.
- Exclude `tests/**/*.cs` from the game project.
- Add the static architecture check before the first story closes.
- Set `run/main_scene` and `dotnet/project/assembly_name` in `project.godot`.
- Un-ignore C# project files in `.gitignore`.
- Unsubscribe Presentation handlers in `_ExitTree`.

**Must never:**

- Put `using Godot;` or any `Godot.*` type in Foundation/Core/Feature.
- Let Feature reference Presentation, or Core reference Feature.
- Put gameplay rules or authoritative state in TableView/Toolbar.
- Let Drag call `TableView` directly; use events and the `IDragView` seam.
- Let `GameSession` construct concrete services or reference Godot types.
- Use an Autoload, static service locator, or mutable global state for gameplay.
- Compile test code into the game assembly.
- Let the architecture check exist only as documentation; it must be executable.
- Serialize `System.Numerics.Vector2` directly without a DTO or `IncludeFields`.

## Alternatives Considered

### Alternative 1: Single project without layer rules

- **Description**: One Godot C# project, all files in flat folders, no enforced
  dependency direction.
- **Pros**: Least ceremony; fastest first story.
- **Cons**: Rules/state code accumulates Godot dependencies; unit tests need a
  scene tree; Table/Drag coupling becomes implicit; parallel work conflicts.
- **Estimated Effort**: Lowest initially, highest maintenance.
- **Rejection Reason**: The GDDs require deterministic, testable logic; the
  architecture would erode within the first few stories.

### Alternative 2: Two projects — `FreeCell.Core` class library + `FreeCell.Game`

- **Description**: A separate `FreeCell.Core.csproj` with no GodotSharp
  reference, referenced by the Godot project; tests reference Core directly.
- **Pros**: Compile-time enforcement of the pure boundary; fastest test host;
  no GodotSharp in unit tests.
- **Cons**: Extra csproj and project reference; the root Godot csproj must
  exclude Core sources from its default glob; Godot export must include the
  referenced assembly; more setup for a solo MVP.
- **Estimated Effort**: Medium.
- **Rejection Reason**: For a 4–5 week solo MVP, the single project plus a
  static architecture check gets most of the benefit with less toolchain risk.
  This remains the documented fallback: if ADR-0005 finds that a test project
  referencing the Godot assembly is costly or unreliable, write a superseding
  ADR to split `FreeCell.Core` out. No gameplay code would need rewriting, only
  file moves and project references.

### Alternative 3: Four assemblies, one per layer

- **Description**: Separate projects for Foundation, Core, Feature and
  Presentation.
- **Pros**: Strongest compile-time isolation.
- **Cons**: Four csproj files, more build/export complexity, overkill for a
  small card game.
- **Estimated Effort**: High.
- **Rejection Reason**: The boundary count is small enough that one assembly
  plus a static check is adequate; the extra assemblies add configuration cost
  without a corresponding gameplay benefit.

## Consequences

### Positive

- One build and one Godot project; the shortest path to a runnable game.
- Pure logic can be tested without rendering; `TableLayout` geometry is fully
  unit-testable.
- Table Rendering no longer creates a Core → Feature dependency.
- Drag has a pure state machine and a narrow Presentation input adapter.
- Clear ownership of every GDD system; parallel stories can work in separate
  folders.
- No Autoload/global singleton; lifecycle is explicit.

### Negative

- “No Godot in pure layers” is enforced by a static check, not the compiler;
  the check is mandatory and must be maintained.
- A test project referencing the Godot assembly still loads GodotSharp
  transitively; pure tests are not fully engine-free at the process level.
- `.gitignore` and export filtering need explicit fixes; both are easy to
  forget and both fail silently.
- One assembly means a change in any layer rebuilds the whole game.

### Neutral

- App is a new non-GDD layer containing exactly one Godot node.
- Table Rendering is one GDD system but two code modules.
- Godot `Vector2` ↔ `System.Numerics.Vector2` conversion is an adapter concern,
  not a gameplay concern.

## Risks

| Risk | Mitigation |
|---|---|
| `.gitignore` hides `FreeCell.csproj` / test `.csproj` | Add `!FreeCell.csproj`, `!FreeCell.sln`, `!tests/**/*.csproj`; verify with `git check-ignore` |
| Main project compiles tests by default | `<Compile Remove="tests/**/*.cs" />`; build and inspect the assembly before first story closes |
| Godot export includes `docs/`, `design/`, `tests/` | Export preset exclusions or dependencies-only export; run one real export and inspect the package |
| Pure boundary erodes | Executable architecture check covering `using Godot`, `Godot.`, `global::Godot` and forbidden edges; code review |
| Test project loads GodotSharp and is awkward | ADR-0005 decides runner; fallback is the two-project Core library described in Alternative 2 |
| `GameBootstrap` children use services before construction | Construct in `_EnterTree` or require `Bind()` before use; child `_Ready` runs before parent |
| Event handlers leak after node disposal | Subscribe in `Bind()`, unsubscribe in `_ExitTree()` |
| Assembly name mismatch with Chinese project display name | Set `[dotnet] project/assembly_name="FreeCell"`; keep `config/name` as display-only; verify editor run + export |
| `System.Text.Json` silently drops `Vector2` fields | ADR-0003 uses DTOs or `IncludeFields`; do not serialize pure `Vector2` directly |
| 4.7 UI/Input risks leak into this decision | ADR-0004/0006/0007 own stretch, dual-focus and input device IDs; this ADR is engine-API-free |

## GDD Requirements Addressed

This is a foundational decision. It does not change a GDD rule; it determines
how every GDD system is implemented and tested.

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `card-deck-model.md` | Unique pure C# state with invariants and snapshots (TR-card-deck-model-003/004/005) | Foundation module, no Godot types; unit-testable |
| `deal-generator.md` | Pure deterministic function, no engine RNG (TR-deal-generator-002/003) | Core module, BCL only |
| `rules-move-validator.md` | Read-only, deterministic legality and enumeration (TR-rules-move-validator-001) | Core module, no Godot or Feature references |
| `game-state-machine.md` | Lifecycle coordination without owning pile state (TR-game-state-machine-001/008) | Feature module, pure C#, injected dependencies |
| `undo-history.md` | Snapshot stack independent of rendering (TR-undo-history-001/006) | Feature module, pure C# |
| `hint-system.md` | Read-only selection from `legal_moves` (TR-hint-system-001/006) | Feature module, pure C# |
| `statistics-records.md` | Session-only counters with no I/O (TR-statistics-records-001/007) | Feature module, pure C# |
| `table-rendering-layout.md` | Geometry/hit-test split from Godot drawing (TR-table-rendering-layout-001/003) | `TableLayout` in Core, `TableView` in Presentation |
| `drag-drop-interaction.md` | Pointer state machine separate from Godot input (TR-drag-drop-interaction-001/009) | `Drag` in Feature, `DragInputAdapter` in Presentation |
| `toolbar-menus.md` | Presentation-only command entry point (TR-toolbar-menus-001/005) | Presentation adapter; no Model/Rules writes |

All 76 provisional TRs are covered indirectly by the dependency and testability
rules above; `/architecture-review` builds the formal traceability matrix.

GDD sync applied on 2026-10-24: `table-rendering-layout.md` and
`drag-drop-interaction.md` now describe the TableView event-driven interface
instead of direct `set_drag_layer` / `set_source_hidden` calls from Drag.

## Performance Implications

- **CPU**: Direct C# calls; no reflection, no event bus, no per-frame
  allocation added by the layering. Input conversion is O(1) per event.
- **Memory**: One game assembly; no duplicate layer assemblies. Test code is
  excluded from the game assembly.
- **Load Time**: One assembly to load; no extra assembly resolution.
- **Network**: Not applicable.

## Migration Plan

There is no existing code, so this is a greenfield layout:

1. Create `FreeCell.csproj` at the repository root with
   `Godot.NET.Sdk/4.7.2` and `net8.0`.
2. Add the `src/` layer folders and `tests/` project.
3. Set `project.godot` `run/main_scene` and
   `dotnet/project/assembly_name="FreeCell"`.
4. Add the `.gitignore` exceptions.
5. Add the static architecture check.
6. Add `GameBootstrap` and a minimal main scene; run the headless boot check.

**Rollback plan**: If the single-project layout proves unworkable, write a
superseding ADR that extracts `FreeCell.Core` as an engine-free class library.
No gameplay behavior changes; move files, add the project reference, and update
the architecture check. This ADR would be marked `Superseded by ADR-XXXX`.

## Validation Criteria

- `dotnet build FreeCell.csproj` completes with 0 errors and 0 warnings.
- `.gitignore` no longer hides `FreeCell.csproj`, `FreeCell.sln` or
  `tests/**/*.csproj` (`git check-ignore` returns nothing for them).
- The static architecture check exits non-zero when any pure layer contains
  `using Godot`, `Godot.` or `global::Godot`, or when a forbidden dependency
  edge is introduced.
- `dotnet build tests/unit/...` succeeds; unit tests run without a scene tree
  (runner decided by ADR-0005).
- Godot headless boots `res://src/App/Main.tscn`, `GameBootstrap` constructs
  services, and a C# `_Ready` marker prints.
- A real export excludes `docs/`, `design/`, `production/`, `tools/`,
  `studio-tests/`, `tests/` and `.claude/`, and the exported build launches.
- No `Feature → Presentation`, `Core → Feature`, or pure-layer `Godot` reference
  exists in the final source.
- TableView consumes Drag/State/Undo events; Drag never calls a Godot view.

## Related

- `docs/architecture/architecture.md` §2 (System Layer Map), §3 (Module
  Ownership), §5.10–5.13 (Table Layout/View, Toolbar, cross-boundary
  constraints)
- ADR-0002 (session composition, model ownership, event contract)
- ADR-0003 (model snapshot and serialization schema)
- ADR-0004 (window stretch and coordinate model)
- ADR-0005 (C# test execution strategy)
- ADR-0006 (input routing and 4.6 dual focus)
- ADR-0007 (Table drawing, atlas and HiDPI)
- `design/gdd/table-rendering-layout.md`, `design/gdd/drag-drop-interaction.md`
- `project.yaml`, `project.godot`, `.gitignore`
