# 空当接龙：经典复刻 — Master Architecture

## Document Status

- **Version**: 1
- **Last Updated**: 2026-10-24
- **Engine**: Godot 4.7.2 / C# / .NET 8 / Forward+
- **GDDs Covered**: `card-deck-model`, `deal-generator`, `rules-move-validator`, `game-state-machine`, `drag-drop-interaction`, `undo-history`, `hint-system`, `statistics-records`, `table-rendering-layout`, `toolbar-menus`
- **ADRs Referenced**: none yet — 0 ADRs exist; all decisions below are proposals until their ADRs are written
- **TD-ARCHITECTURE**: 2026-10-24 — skipped — lean mode
- **LP-FEASIBILITY**: 2026-10-24 — skipped — lean mode

> **Scope.** This document covers the 10 MVP systems that have GDDs. The five
> non-MVP systems (`Save/Resume Game`, `Animation & Feedback`, `Audio & SFX`,
> `Settings`, `Readability & Accessibility`) are mapped as placeholders only;
> no behavior is specified for them here.

---

## 1. Engine Knowledge Gap Summary

### 1.1 Baseline

| Field | Value |
|---|---|
| Engine | Godot 4.7.2 |
| Language | C# / .NET 8 |
| Rendering | Forward+ (Windows D3D12 default since 4.6) |
| LLM training covers | approximately Godot 4.3 (cutoff 2025-05) |
| Post-cutoff versions | 4.4, 4.5, 4.6, 4.7 — all HIGH risk |
| Installed at pin time | **NOT DETERMINED** — no Godot found on PATH or usual install locations |
| gdUnit4 | 6.2.x lists 4.5–4.7.1; 6.2.2 observed running on 4.7.2 |

The installed-version gap is **unknown**, not absent. This document must not be
read as proof that any Godot 4.7 API compiles on the target machine.

### 1.2 HIGH RISK Domains

| Domain | Post-cutoff changes that matter here | Systems affected |
|---|---|---|
| **UI** | 4.6 dual-focus (mouse focus separate from keyboard/gamepad focus); 4.7 new-project defaults `stretch/mode=canvas_items`, `aspect=expand`; `PopupMenu` search bar; `Control.accessibility_live` type moved to `AccessibilityServer`; `Control.offset_transform_*` | Toolbar & Menus, Table View, Drag & Drop, Hint highlight |
| **Input** | 4.7 `InputEvent.device` for mouse/keyboard is `DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD`, not `0`; 4.6 dual-focus | Drag & Drop, Toolbar & Menus |
| **Rendering** | 4.7 `CanvasItem` lines are drawn without an antialiasing feather; 4.6 D3D12 default on Windows; glow processes before tonemapping | Table View, Hint highlight |
| **Core / file** | Since 4.4, `FileAccess.store_*` returns `bool` | Save/Resume (Vertical Slice) |
| **C# API** | 4.7 `RichTextLabel.add_image/update_image` parameter changes; `OptimizedTranslation.generate()` returns `bool`; `Animation.length` is `double`; `Control.accessibility_live` enum moved | Toolbar if rich text/accessibility is used; Animation (VS) |

### 1.3 MEDIUM RISK Domains

- **Project stretch defaults.** `project.godot` has no `[display]` section. A new
  4.7 project defaults to `canvas_items` / `expand`; the GDDs require manual
  `1280×720` letterboxing. This must be pinned by ADR-0004 before Table/Toolbar
  UI work.
- **Font hinting.** 4.7 changed the dynamic-font hinting default. Card rank/suit
  textures are pre-baked, so this is only a risk if dynamic fonts are used for
  card faces.
- **HiDPI / OS display scale.** Atlas selection and `DisplayServer.screen_get_scale()`
  need measurement on the target machine.
- **C# test execution.** The framework's CI command uses gdUnit4's GDScript
  runner. Whether C# tests run through gdUnit4 on 4.7.2, or pure C# unit tests
  need a separate `dotnet test` project, is unresolved (ADR-0005).

### 1.4 LOW RISK Domains

- Pure C# data structures, collections, arithmetic, JSON serialization.
- Basic `Node`, `Node2D`, `Sprite2D`, `Control`, `MenuBar`, `AcceptDialog` usage
  that predates 4.3.
- Input event **types** (`InputEventMouseButton`, `InputEventMouseMotion`,
  `InputEventKey`); the risk is in device IDs and focus semantics, not the types.

### 1.5 Systems Touching HIGH/MEDIUM Risk

| GDD system | Domain | Risk |
|---|---|---|
| Table Rendering & Layout | Rendering / UI / Input | HIGH |
| Toolbar & Menus | UI / Input | HIGH |
| Drag & Drop Interaction | Input / UI | HIGH |
| Hint System | Rendering (highlight) | HIGH |
| Save/Resume Game | Core / `FileAccess` | HIGH (VS) |
| Settings / Readability & Accessibility | UI | HIGH (not designed) |

### 1.6 Verification Rule

Any API marked ⚠️ in this document must be checked against
`docs/engine-reference/godot/` before coding. If the reference does not cover
it, record `NOT SOURCEABLE — <API> is not covered by <path>` rather than filling
the gap from training data. A post-cutoff API that cannot be verified must not
be used as the basis of a blocking ADR.

---

## 2. System Layer Map

### 2.1 Layers

| Layer | Module / system | Status | Responsibility |
|---|---|---|---|
| **App** (composition root, not a GDD system) | `GameBootstrap` | Architecture addition | Constructs services, wires events, owns startup/shutdown; the only place that knows every concrete type |
| **Presentation** | Table View (drawing half of Table Rendering & Layout) | MVP | Card/table drawing, drag layer, hint highlight, source hiding |
| | Toolbar & Menus | MVP | Menu bar, dialogs, banners, title, command gating |
| | Animation & Feedback | VS, no GDD | Placeholder only |
| | Audio & SFX | VS, no GDD | Placeholder only |
| | Settings | Full Vision, no GDD | Placeholder only |
| | Readability & Accessibility | Full Vision, no GDD | Placeholder only |
| **Feature** | Game State Machine | MVP | Deal number, lifecycle, command coordination, terminal-state evaluation |
| | Undo History | MVP | Snapshot stack, undo |
| | Hint System | MVP | Candidate ranking and cycling |
| | Statistics & Records | MVP | Session counters |
| | Drag & Drop Interaction | MVP | Pointer lifecycle, drop routing |
| | Save/Resume Game | VS, no GDD | Placeholder only |
| **Core** | Deal Generator | MVP | Deal number → 52-card sequence |
| | Rules & Move Validator | MVP | Legality, enumeration, win/stuck predicates |
| | Table Layout (geometry half of Table Rendering & Layout) | MVP | Pure scale/offset/rect/hit-test geometry |
| **Foundation** | Card & Deck Model | MVP | Sole card/pile state, invariants, snapshots, serialization |
| | Shared Primitives | Architecture addition | `Card`, `PileId`, `Move`, `DealSequence`, `GameState`, `Result`, `LogicalRect` |
| **Platform** | Godot 4.7.2 / .NET 8 / Windows | — | Engine and OS; no project code |

### 2.2 Dependency Direction

```text
App / GameBootstrap
        |
        v
Presentation: TableView, Toolbar, Animation, Audio, Settings, Accessibility
        |
        v
Feature: State, Undo, Hint, Statistics, Drag, Save/Resume
        |
        v
Core: Deal, Rules, TableLayout
        |
        v
Foundation: CardDeckModel, Shared Primitives
        |
        v
Platform: Godot 4.7.2 / .NET 8 / Windows
```

Arrows mean **depends on**. The following are forbidden:

- Foundation/Core referencing `Godot` namespaces or node types.
- Feature referencing Presentation.
- Presentation directly calling `CardDeckModel` or `Rules` mutators. Toolbar
  goes through State; Drag goes through Rules/Model only on the legal path;
  Table View is read-only.
- Any module raising events back into a lower layer.

### 2.3 Table Rendering Split

`Table Rendering & Layout` is one GDD system but two modules:

| Module | Layer | Owns | Why separate |
|---|---|---|---|
| **Table Layout** | Core | `scale`, `offset`, card rects, cascade, hit test, coordinate transforms | Pure geometry; fully unit-testable; Drag must not depend on Godot drawing nodes |
| **Table View** | Presentation | Sprites, atlas, drag layer visuals, hint highlight, source hiding | Godot rendering and scene-tree lifecycle |

This split resolves an otherwise inverted dependency: the GDD lets Table
Rendering read `State.NoGame`, but State is Feature and Table Layout is Core.
After the split, Table View subscribes to State events; Table Layout never
depends on State. Observable behavior is unchanged.

### 2.4 Engine Awareness — Foundation/Core

| Module | Engine APIs | Risk | Note |
|---|---|---|---|
| Card & Deck Model | none — pure C# | LOW | Serialization uses `System.Text.Json`, not Godot |
| Shared Primitives | none — pure C# | LOW | `System.Numerics.Vector2` for points |
| Deal Generator | none — pure C# | LOW (business risk HIGH) | Algorithm correctness is the risk, not the engine |
| Rules & Move Validator | none — pure C# | LOW | Read-only logic |
| Table Layout | none — pure C# | LOW | Receives viewport size from the Presentation adapter |

The HIGH-risk engine work is therefore concentrated in Presentation and the
Drag adapter, not in the pure core.

### 2.5 Non-MVP Placeholders

`Save/Resume Game`, `Animation & Feedback`, `Audio & SFX`, `Settings`, and
`Readability & Accessibility` appear in the layer map because the systems index
lists them, but they have no GDD. This architecture makes **no behavioral
commitment** for them. Their module boundaries may change when their GDDs are
written.

---

## 3. Module Ownership

### 3.1 Foundation

| Module | Owns | Exposes | Consumes | Engine APIs / risk |
|---|---|---|---|---|
| **Card & Deck Model** | Card/pile state, invariants I1–I5, `Revision`, `ModelSnapshot`, schema version | `ICardDeckModel`: reads, `Move`, `Snapshot/Restore`, `Deal`, `Serialize/Deserialize` | `DealSequence`, Shared Primitives | none; pure C# — LOW |
| **Shared Primitives** | Value types and vocabulary | `Card`, `Suit`, `Rank`, `PileId`, `Move`, `DealSequence`, `GameState`, `Result<T>`, `ErrorCode`, `LogicalRect` | none | none; pure C# — LOW |

### 3.2 Core

| Module | Owns | Exposes | Consumes | Engine APIs / risk |
|---|---|---|---|---|
| **Deal Generator** | Microsoft LCG algorithm, `AlgorithmVersion` | `Generate(N) -> DealSequence` | Shared Primitives | none; pure C# — LOW; algorithm HIGH |
| **Rules & Move Validator** | Rule predicates, enumeration order, reason codes | `CanMove`, `LegalMoves`, `AutoMoveDestination`, `IsWon`, `IsStuck`, `ValidSequence`, `MaxSequenceLength` | `ICardDeckModel`, Shared Primitives | none; pure C# — LOW |
| **Table Layout** | Scale/offset, card rects, cascade, hit test | `CardRect`, `HitTest`, `ToLogical/ToWindow`, `LayoutChanged` | Viewport size, tableau heights | none; pure C# + `System.Numerics` — LOW |

### 3.3 Feature

| Module | Owns | Exposes | Consumes | Engine APIs / risk |
|---|---|---|---|---|
| **Game State Machine** | Current N, lifecycle, event contract, terminal-state timing | Commands, `Snapshot/LoadGame`, `CanAcceptMoves`, state events | Deal, Model, Rules, `IUndoHistory` | none; pure C# — LOW |
| **Undo History** | LIFO snapshot stack, pending slot, depth cap | `RecordBeforeMove/Commit/Discard`, `Undo`, `Clear`, `CanUndo`, events | Model, `GameState` vocabulary | none; pure C# — LOW |
| **Hint System** | Candidate cache `H = (ranked_moves, cursor)` | `RequestHint`, `Reset` | Rules, Model read-only | none; pure C# — LOW |
| **Statistics & Records** | Session counters, active game | `OnGameStarted/OnGameWon/...`, `GetStats`, `ResetStats` | State events | none; pure C# — LOW |
| **Drag & Drop Interaction** | Pointer lifecycle, drag unit, grab offset, cancellation | `OnPointerDown/Move/Up/Cancel`, `IsDragging`, `DragUnit`, drag events | State, Rules, Model, Undo, Table Layout | ⚠️ `Node._UnhandledInput`, `InputEventMouseButton/Motion/Key`, `DoubleClick`, device type checks — HIGH (4.7 device IDs, 4.6 dual focus); C# binding names **NEEDS VERIFICATION** |
| **Save/Resume Game** (VS) | Save envelope, version, file location | `Save/Load` (to be designed) | State snapshot, Deal version | ⚠️ `FileAccess` or `System.IO` — HIGH (4.4 `store_*` returns `bool`); format **NOT DESIGNED** |

### 3.4 Presentation

| Module | Owns | Exposes | Consumes | Engine APIs / risk |
|---|---|---|---|---|
| **Table View** | Card/empty-slot drawing, atlas, drag layer visuals, hint highlight, source hiding | `Bind`, `Redraw`, `SetHintHighlight/ClearHint`, `IDragView` | Model read-only, Table Layout, State/Undo/Drag events | ⚠️ `Node2D`, `Sprite2D`, `AtlasTexture`, `Viewport.SizeChanged`, `DisplayServer.ScreenGetScale()` — HIGH (4.7 line AA, stretch defaults, HiDPI) |
| **Toolbar & Menus** | Menu bar, dialogs, banners, title, command gating, string table | `Bind`, `RefreshCommands`, dialog commands | State/Undo/Hint/Stats/Table View/Drag events | ⚠️ `Control`, `MenuBar`, `PopupMenu`, `AcceptDialog`, `ConfirmationDialog`, `LineEdit`, `CanvasLayer`, `Window.Title`, focus — HIGH (4.6 dual focus, 4.7 stretch/accessibility type) |
| **Animation / Audio / Settings / Accessibility** | Not designed | Placeholder interfaces only | — | Not assessed; must not bind APIs before their GDDs and ADRs |

### 3.5 App

| Module | Owns | Exposes | Consumes | Engine APIs / risk |
|---|---|---|---|---|
| **GameBootstrap** | Construction order, service instances, event wiring, startup/shutdown | Entry point only | Every module | `Node`, `SceneTree`, `PackedScene` — LOW |

### 3.6 Key Ownership Decisions

1. **One `CardDeckModel` instance per session.** `Deal` resets it; snapshots are
   immutable copies. State holds a reference but does not own pile data. Undo
   stores only snapshots.
2. **State and Undo do not hold each other as concrete objects.** State depends
   on `IUndoHistory.Clear()`; Undo depends on the `GameState` vocabulary and
   raises `UndoPerformed`. State subscribes to that event and runs
   `OnUndoApplied`. This preserves the GDD's observable behavior while removing
   a compile-time cycle.
3. **Drag does not depend on Table View.** Drag depends on Table Layout and its
   own events. Table View subscribes to `DragStarted/Moved/Dropped/Cancelled` and
   draws the drag layer/hides the source. The GDD's
   `set_drag_layer`/`set_source_hidden` responsibilities are fulfilled by the
   view consuming events; behavior is equivalent.
4. **Toolbar is the only command entry point.** It does not call Model or Rules
   directly. Hint receives the model via `State.Snapshot().Model`.
5. **Statistics only consumes State events.** It does not read Model, Rules, or
   Undo.
6. **`GameBootstrap` is the only composition root.** No service locator, no
   global singleton, no static mutable state.

---

## 4. Data Flow

### 4.1 Startup and Initialization

```text
Main.tscn
  -> GameBootstrap._Ready()
    -> new CardDeckModel()
    -> new DealGenerator(), new RulesMoveValidator(), new TableLayout(viewport size)
    -> new UndoHistory(model)
    -> new GameStateMachine(model, deal, rules, undo)
    -> new HintSystem(rules), new Statistics()
    -> new TableView(model, layout)
    -> new DragDropController(state, rules, model, undo, layout)
    -> new ToolbarController(...)
    -> wire events (State/Undo/Drag -> views)
    -> set minimum window size 1024x576
    -> initial state NoGame; optionally State.NewGame(1) per QQ-01
```

- Construction order: Foundation → Core → Feature → Presentation → App.
- All services are constructed on the main thread; there is no async
  initialization.
- `GameBootstrap` is the only place that knows every concrete type.

### 4.2 Drag and Drop (the frame-critical path)

```text
InputEvent (mouse)
  -> DragDropController._UnhandledInput
    -> TableLayout.ToLogical / HitTest
    -> State.CanAcceptMoves()
    -> Rules.ValidSequence / CanMove
      -> Undo.RecordBeforeMove(model, MoveResult)
      -> CardDeckModel.Move(...)              [synchronous, same frame]
      -> Undo.Commit()
      -> State.OnMoveApplied(ModelMoveResult)
        -> Rules.IsWon / IsStuck
        -> StateChanged / GameWon / GameStuck [synchronous events]
          -> TableView.Redraw()
          -> Statistics (GameStarted/GameWon)
          -> Toolbar.RefreshCommands()
          -> Audio (VS)
```

- **Data transferred:** `PileId`, `Move`, `ModelMoveResult`.
- **Producer → consumer:** Drag → Rules/Model/Undo/State; State → views/Stats.
- **Synchronous calls + synchronous events.** No cross-thread flow.
- **Failure paths:**
  - Rules rejects → 120 ms snapback; no model/state change.
  - Model `Move` fails unexpectedly → snapback and log.
  - `RecordBeforeMove` fails → abort the move, snapback, no State notification.
  - `Commit` fails → move stands, do not roll back, `DiscardPending`, log.

### 4.3 Hint

```text
Toolbar F10
  -> State.Snapshot().Model            [exactly one snapshot]
  -> HintSystem.RequestHint(model)
  -> HintResult
    -> TableView.SetHintHighlight(source, destination)
    -> or TableView.ClearHint + message
```

- **Data:** `HintResult`, `Move`, destination index (empty tableau = −1, empty
  top-row slot = 0, non-empty pile = height−1).
- **Synchronous.** Hint never writes the model and never executes a move.
- **Clear paths:** next hint, drag start, successful move, undo, new game,
  restart, load, entering Won/Stuck/NoGame.

### 4.4 Undo

```text
Toolbar Ctrl+Z
  -> UndoHistory.Undo(State.State)
    -> CardDeckModel.Restore(snapshot)   [atomic]
    -> UndoPerformed
      -> GameStateMachine.OnUndoApplied()
        -> StateChanged / Won / Stuck
          -> TableView.Redraw()
          -> Toolbar.RefreshCommands()
          -> HintSystem.Reset()
```

- **Data:** `ModelSnapshot`, `MoveResult`.
- **Synchronous, 0 ms, no animation.**
- Restore failure: discard the bad record, leave the model unchanged, emit
  `UndoFailed`.

### 4.5 New Game, Restart, Load

```text
Toolbar
  -> State.NewGame(N) | RestartDeal() | SelectDeal(N)
    -> DealGenerator.Generate(N)
    -> CardDeckModel.Deal(sequence)
    -> Undo.Clear()
    -> State = Playing
    -> GameStarted / GameRestarted / GameLoaded
      -> TableView.Redraw()
      -> Statistics.OnGameStarted()
      -> Toolbar: title, close dialogs/banners, Hint.Reset()
```

- Only a successful load clears Undo. A failed load leaves history intact and
  does not emit `GameLoaded`.
- A won snapshot loaded from disk emits `GameLoaded` only, not `GameWon`.

### 4.6 Window Resize

```text
Viewport.SizeChanged
  -> TableLayout.UpdateViewport(w, h)
    -> recompute scale/offset/card rects once
    -> LayoutChanged
      -> TableView.Reposition()
      -> DragDropController.OnLayoutChanged()   [recompute grab offset]
```

- All main-thread.
- The menu bar uses window coordinates and does not scale with Table Layout.
- Minimum window 1024×576; below that, scale is clamped to 0.8, the canvas is
  cropped, and a warning is logged.

### 4.7 Save/Load (Vertical Slice, not implemented)

```text
State.Snapshot()
  -> SaveService.Serialize(envelope: algorithmVersion + gameSnapshot + stats?)
  -> file

file
  -> SaveService.Deserialize + validate schema/algorithm/model invariants
  -> State.LoadGame(snapshot)
  -> GameLoaded -> views/Stats reconcile (VS design required)
```

- Save format, statistics persistence, and active-game reconciliation are
  Vertical Slice concerns. This architecture only reserves the interface.
- File I/O must not block the main thread. If asynchronous, the result must be
  marshalled back to the main thread before calling State.

### 4.8 Event Architecture

- **Direct typed C# events**, no global event bus.
- Owners publish: State, Undo, Drag.
- `GameBootstrap` wires subscriptions. Subscribers depend on event contracts,
  not on the publisher's concrete type.
- Events are synchronous, ordered, and non-reentrant. State emits after its own
  state is updated. A subscriber exception must not roll back state; use a
  `SafeRaise` helper that catches and logs per subscriber.
- There is no cross-thread event flow.

### 4.9 Threading

- All gameplay logic, UI, and event handling run on the Godot main thread.
- No locks, no background workers, no async in Core/Foundation/Feature.
- The only possible asynchronous work is Vertical Slice file I/O.

---

## 5. API Boundaries

The interfaces below are the contracts programmers implement against. Exact
error codes and edge conditions remain owned by the GDDs.

### 5.1 Shared Primitives (Foundation)

```csharp
public enum Suit { Clubs, Diamonds, Hearts, Spades }
public enum Rank : byte { Ace = 1, Two = 2, /* ... */ King = 13 }
public readonly record struct Card(Suit Suit, Rank Rank);

public enum PileType { Tableau, FreeCell, Foundation }
public readonly record struct PileId(PileType Type, int Index);

public readonly record struct Move(
    PileId Source,
    PileId Destination,
    ImmutableArray<Card> Cards);

public readonly record struct DealSequence(ImmutableArray<Card> Cards);

public enum GameState { NoGame, Dealing, Playing, Won, Stuck, Restoring }

public readonly record struct Result(bool Success, ErrorCode Error);
public readonly record struct Result<T>(bool Success, T Value, ErrorCode Error);

public readonly record struct LogicalRect(double X, double Y, double Width, double Height);
```

**Naming note.** `Move` is the intent used by Hint and `LegalMoves`.
`MoveResult` belongs to Rules (`Legal`, `Reason`). `ModelMoveResult` belongs to
the model (`Success`, `FailureReason`). Do not collapse these into one type.

### 5.2 Card & Deck Model

```csharp
public interface ICardDeckModel
{
    int Revision { get; }                 // increments on every successful mutation
    bool IsDealt { get; }

    int Count(PileId pile);
    Card? TopCard(PileId pile);
    ImmutableArray<Card> Cards(PileId pile);   // immutable defensive copy
    int FoundationHeight(PileId foundation);
    int EmptyFreeCellCount { get; }
    int EmptyTableauColumnCount { get; }
    bool GameComplete { get; }

    ModelMoveResult Move(PileId source, PileId destination, int count);
    ModelSnapshot Snapshot();
    Result Restore(ModelSnapshot snapshot);
    Result Deal(DealSequence sequence);

    string Serialize();                       // schema version in the payload
}
```

**Caller invariants:**

- `Move` is called only after `Rules.CanMove` returns `Legal = true`; `Deal` is
  called only by State during new/restart/load.
- Reads return immutable collections; callers cannot mutate the model through
  them.
- A failed operation leaves the state unchanged.
- `ModelMoveResult` contains `Success`, `Source`, `Destination`, `Cards`, and
  `FailureReason`.
- `Revision` is an architectural addition to support redraw/debugging; it does
  not change any GDD rule.

### 5.3 Deal Generator

```csharp
public interface IDealGenerator
{
    string AlgorithmVersion { get; }          // "Microsoft FreeCell LCG v1"
    Result<DealSequence> Generate(int dealNumber);
}
```

**Caller invariants:** N must be an integer in [1, 1,000,000]. The generator is
pure and stateless; it never calls the model.

### 5.4 Rules & Move Validator

```csharp
public interface IRulesMoveValidator
{
    MoveResult CanMove(ICardDeckModel model, PileId source, PileId destination, int count);
    ImmutableArray<Move> LegalMoves(ICardDeckModel model);
    MoveResult AutoMoveDestination(ICardDeckModel model, PileId source, Card clickedCard);
    bool IsWon(ICardDeckModel model);
    bool IsStuck(ICardDeckModel model);
    bool ValidSequence(ICardDeckModel model, PileId column, int count);
    int MaxSequenceLength(ICardDeckModel model, PileId column);
}
```

**Caller invariants:**

- Read-only model; no writes.
- `LegalMoves` returns a fixed order, deduplicated, free cells enumerated by
  slot, upper bound 768.
- `AutoMoveDestination` handles only top cards; non-top returns `NOT_TOP_CARD`.
- `CanMove` always starts the move unit at the source top card.

### 5.5 Game State Machine

```csharp
public interface IGameStateMachine
{
    GameState State { get; }
    int DealNumber { get; }
    bool CanAcceptMoves();

    Result NewGame(int dealNumber);
    Result RestartDeal();
    Result NextDeal();
    Result PreviousDeal();
    Result SelectDeal(int dealNumber);
    Result OnMoveApplied(ModelMoveResult result);
    Result OnUndoApplied();

    Result<GameSnapshot> Snapshot();          // NoGame/Dealing/Restoring -> GAME_NOT_READY
    Result LoadGame(GameSnapshot snapshot);

    event Action<GameStarted> GameStarted;
    event Action<GameRestarted> GameRestarted;
    event Action<GameWon> GameWon;
    event Action<GameStuck> GameStuck;
    event Action<GameLoaded> GameLoaded;
    event Action<StateChanged> StateChanged;
}
```

**Caller invariants:**

- `NewGame` clears Undo only on success.
- `GameStarted` is emitted exactly once per successful `new_game`.
- `GameWon`/`GameStuck` is emitted once per terminal entry.
- Events are synchronous; subscriber exceptions do not roll back state.
- `Snapshot` does not write files; serialization belongs to Save/Resume.

### 5.6 Undo History

```csharp
public interface IUndoHistory
{
    Result RecordBeforeMove(ICardDeckModel model, MoveResult move);
    Result Commit();
    Result DiscardPending();                  // idempotent
    Result Undo(GameState state);
    void Clear();
    bool CanUndo(GameState state);
    int Depth { get; }
    MoveResult? Peek();

    event Action<UndoPerformed> UndoPerformed;
    event Action<UndoFailed> UndoFailed;
    // other events per GDD: UndoRecorded, UndoCommitted, UndoEvicted, UndoCleared
}
```

**Caller invariants:**

- `RecordBeforeMove` precedes `model.Move`; `Commit` follows success; failure or
  cancellation calls `DiscardPending`.
- Only one pending record at a time.
- `Undo` is allowed only in `Playing`/`Stuck` with no pending; restore is atomic.
- Undo does not call State directly; State subscribes to `UndoPerformed`.

### 5.7 Hint System

```csharp
public interface IHintSystem
{
    HintResult RequestHint(ICardDeckModel model);
    void Reset();
}
```

**Caller invariants:**

- Read-only model; never calls `CanMove`/`AutoMoveDestination`; never executes a
  move.
- Candidates come only from `Rules.LegalMoves`.
- Toolbar must not request or display a hint during a drag.
- Highlight lifetime is owned by Toolbar/Table View.

### 5.8 Statistics & Records

```csharp
public interface IStatistics
{
    void OnGameStarted(int dealNumber);
    void OnGameWon(int dealNumber);
    void OnGameRestarted(int dealNumber);     // no-op
    void OnGameStuck(int dealNumber);         // no-op
    StatsSnapshot GetStats();
    void ResetStats();
}
```

**Caller invariants:**

- Does not read Model, Rules, or State.
- `GetStats` returns an immutable copy; `win_rate` is derived on every read.
- Legal events must not throw; wrong-game/duplicate/unknown events are ignored.

### 5.9 Drag & Drop

```csharp
public interface IDragDropController
{
    void OnPointerDown(Vector2 logicalPosition);
    void OnPointerMove(Vector2 logicalPosition);
    void OnPointerUp(Vector2 logicalPosition);
    void OnCancel();
    void OnLayoutChanged();

    bool IsDragging { get; }
    Move? DragUnit { get; }

    event Action<DragStarted> DragStarted;
    event Action<DragMoved> DragMoved;
    event Action<DragDropped> DragDropped;
    event Action<DragCancelled> DragCancelled;
    event Action<MoveExecuted> MoveExecuted;
    event Action<MoveRejected> MoveRejected;
}
```

**Caller invariants:**

- `OnPointer*` receives logical coordinates; indices are 0-based bottom-to-top.
- A drag starts only when `State.CanAcceptMoves()` is true.
- The legal path is exactly `record -> move -> commit -> State.OnMoveApplied`.
- Cancel/reject paths call `DiscardPending`; model and State remain unchanged.

### 5.10 Table Layout (Core)

```csharp
public sealed class TableLayout
{
    public double Scale { get; }
    public Vector2 Offset { get; }

    public void UpdateViewport(double windowWidth, double windowHeight);
    public Vector2 ToLogical(Vector2 windowPoint);
    public Vector2 ToWindow(Vector2 logicalPoint);

    public LogicalRect CardRect(PileId pile, int index, int tableauHeight);
    public HitResult? HitTest(Vector2 logicalPoint, IReadOnlyList<int> tableauHeights);

    public event Action LayoutChanged;
}
```

**Caller invariants:**

- All inputs are logical coordinates; outside the canvas returns null.
- `CardRect` index is 0-based bottom-to-top; `-1` means the first empty slot.
- Pure geometry and deterministic; does not read the model or draw.

### 5.11 Table View (Presentation)

```csharp
public partial class TableView : Node2D, IDragView
{
    public void Bind(ICardDeckModel model, TableLayout layout);
    public void Redraw();
    public void SetHintHighlight(Move move);
    public void ClearHint();

    // IDragView
    public void ShowDragLayer(Move unit, Vector2 logicalPosition);
    public void MoveDragLayer(Vector2 logicalPosition);
    public void HideDragLayer();
    public void SetSourceHidden(PileId pile, int count, bool hidden);
}
```

**Caller invariants:**

- Read-only model; never mutates State or Undo.
- Redraws in the same frame as State/Undo/Drag events.
- Source hiding and drag-layer display are same-frame; legal drops are 0 ms.
  Illegal snapback is 120 ms with a ≤150 ms watchdog.

### 5.12 Toolbar (Presentation)

```csharp
public partial class ToolbarController : Control
{
    public void Bind(
        IGameStateMachine state,
        IUndoHistory undo,
        IHintSystem hint,
        IStatistics statistics,
        TableView tableView,
        IDragDropController drag);

    public void RefreshCommands();
    public void ShowMessage(string stringKey);
    public void ShowStuckBanner();
    public void HideBanners();
}
```

**Caller invariants:**

- The only command entry point; never calls Model or Rules directly.
- All visible strings come from the string table through `tr()`; error-code to
  text mapping lives in one place.
- Modal dialogs suppress global shortcuts; the specified commands are disabled
  during a drag; focus returns to the table after closing.
- Menu bar height is 28 window pixels; it uses window coordinates and does not
  scale with Table Layout.

### 5.13 Cross-Boundary Constraints

- Gameplay code in Foundation/Core/Feature must not reference `Godot`
  namespaces.
- Presentation adapters own window↔logical conversion, Godot input events↔pure
  C# pointer events, and node lifecycle.
- No cross-thread calls. Vertical Slice file I/O may be asynchronous but must
  marshal back to the main thread before calling State.
- Card/snapshot collections use `ImmutableArray<T>` or defensive copies; never
  expose a mutable internal collection.
- Engine APIs appear only in Presentation and the Drag adapter; Core/Foundation
  stay portable and unit-testable.

---

## 6. ADR Audit and Traceability

### 6.1 ADR Audit

`docs/architecture/adr-*.md`: **0 ADRs**. There is nothing to audit for engine
compatibility, version recording, GDD linkage, or cross-ADR conflicts.

| ADR | Engine Compat | Version | GDD Linkage | Conflicts | Valid |
|---|---|---|---|---|---|
| — | — | — | — | — | — |

### 6.2 Technical Requirements Baseline

- **Extracted from:** 10 GDDs
- **Total:** 76 requirements
- **Registered:** 0
- **Provisional:** 76
- **TR registry:** `docs/architecture/tr-registry.yaml` exists with
  `requirements: []`

All IDs are provisional until `/architecture-review` registers them. The full
list is in Appendix A.

### 6.3 Coverage

| Scope | Count | Covered | Gaps |
|---|---:|---:|---:|
| Technical Requirements Baseline | 76 | 0 | **76** |

| System | Provisional IDs | Covered |
|---|---|---:|
| Card & Deck Model | 001–008 | 0/8 |
| Deal Generator | 001–008 | 0/8 |
| Rules & Move Validator | 001–008 | 0/8 |
| Game State Machine | 001–008 | 0/8 |
| Drag & Drop Interaction | 001–010 | 0/10 |
| Undo History | 001–006 | 0/6 |
| Hint System | 001–008 | 0/8 |
| Statistics & Records | 001–007 | 0/7 |
| Table Rendering & Layout | 001–008 | 0/8 |
| Toolbar & Menus | 001–009 | 0/9 |

**Interpretation.** 76/76 gaps is the expected first-pass state: no ADRs exist
yet. The required ADRs below group the decisions; `/architecture-review` will
build the formal traceability matrix after they are written.

---

## 7. Required ADRs

### 7.1 Must Have Before Coding Starts

| Priority | ADR | Covers / unblocks |
|---|---|---|
| **1** | **ADR-0001 — Layered boundaries and pure C# core.** Presentation → Feature → Core → Foundation; Core/Foundation do not reference Godot; Table splits into Layout + View; DI instead of global singletons. | Dependency rules for all 76 requirements; unblocks unit testing and parallel implementation of every module. |
| **2** | **ADR-0002 — Session composition, model ownership, and event contract.** One `CardDeckModel` instance; `GameBootstrap` wiring; direct typed C# events; State↔Undo decoupled via `IUndoHistory` + `UndoPerformed`; single-threaded synchronous flow. | State 001/003/008, Undo 003/006, Statistics 002/007, Drag 005/006/008, Table 005/008, Toolbar 009. |
| **3** | **ADR-0003 — Model snapshot and serialization schema.** Immutable `ModelSnapshot`, `Revision`, schema version, atomic restore, `ImmutableArray`, invariant validation on load. | Card 001–006/008, Undo 001/005/006, State 005, Deal 004. |
| **4** | **ADR-0004 — Window stretch and coordinate model.** `stretch/mode="disabled"`; manual 1280×720 letterbox; menu bar in window coordinates; Table Layout in logical coordinates; minimum window 1024×576. | Table 001/006/007/008, Toolbar 001/003/004/009, Drag 002/008. **HIGH RISK — verify against 4.7 stretch defaults.** |
| **5** | **ADR-0005 — C# test execution strategy.** Pure C# core tests (xUnit or gdUnit4 C#); Godot integration tests (gdUnit4 headless); screenshot evidence paths; CI command. | Verification method for every logic requirement; unblocks `/test-setup` and the first story. |

### 7.2 Should Have Before the Relevant System Is Built

| ADR | Needed before | Covers |
|---|---|---|
| **ADR-0006 — Input routing and 4.6 dual focus.** `_UnhandledInput`, GUI priority, modal suppression, device-type checks, double-click threshold. | Drag/Toolbar implementation | Drag 001/002/008/009, Toolbar 002/004/009/010, Table 003 |
| **ADR-0007 — Table drawing, atlas, and HiDPI.** `Sprite2D`/`AtlasTexture`, baked borders, 1x/2x/3x, mipmaps, no runtime lines. | Table View implementation | Table 002–007, Hint 006, Toolbar 009 |
| **ADR-0008 — Localization and string table.** Godot `tr()`, string keys, default Chinese, centralized error mapping. | Toolbar implementation | Toolbar 006/008 |
| **ADR-0009 — Save/Resume format and file storage.** JSON schema, algorithm version, `FileAccess` or `System.IO`, failure recovery. | Vertical Slice | Card 006, Deal 007, State 005/007, Save (VS) |

### 7.3 Can Defer to Implementation / Later Tiers

- Statistics persistence and load reconciliation (Vertical Slice).
- Animation & Feedback curves and completion signals (Vertical Slice).
- Audio & SFX event/de-duplication (Vertical Slice).
- Settings storage and audio toggle (Full Vision).
- Readability & Accessibility high-contrast/keyboard alternatives (Full Vision).

---

## 8. Architecture Principles

1. **Rules are pure functions; the model is the only state.** Rules is read-only,
   deterministic, and unit-testable. Every write is gated by Rules; the model
   only enforces structural invariants.
2. **Dependencies point down.** Presentation → Feature → Core → Foundation. The
   composition root is the only place that knows all concrete types.
   Core/Foundation never reference Godot.
3. **Single-threaded, synchronous, deterministic.** Game logic has no random
   numbers, no time dependency, and no async. The only randomness is the deal
   number LCG. Events are synchronous; subscriber exceptions do not roll back
   state.
4. **Every player-visible behavior has a verifiable contract.** Pure logic is
   unit-tested; cross-system paths are integration-tested; UI and visual changes
   retain screenshots; performance becomes BLOCKING once the reference machine
   is pinned.
5. **Fidelity over modernization.** Original rules unchanged; 1995 visuals;
   Windows 7/XP feature baseline. MVP adds no animation/audio/settings/
   accessibility extras. Every deviation is recorded.

### 8.1 MVP Scope Guardrails

- No network, accounts, leaderboards, cloud saves, achievements, skins, or
  tutorial.
- No animation or audio polish in MVP; Table View draws final positions
  directly.
- No statistics persistence in MVP; Statistics is session-only.
- No mouse-only exception for keyboard reachability: Toolbar must be keyboard
  reachable even in MVP, because 4.6 dual-focus makes this a real risk.

---

## 9. Open Questions

| ID | Summary | Priority | Resolution path |
|---|---|---|---|
| QQ-01 | First launch `NoGame` vs concept “direct deal” (W5) | Medium | Design decision; `GameBootstrap` can call `State.NewGame(1)` or stay `NoGame` |
| QQ-02 | Statistics loss vs “no punishment” tension (W1) | Medium | Playtest / Statistics design review |
| QQ-03 | Hint Tier 1 may recommend unsafe foundation moves (W2) | Medium | Playtest / Hint rule review |
| QQ-04 | Menu “新局” vs Won dialog “新局” naming (W6) | Low | Copy/UX decision |
| QQ-05 | `project.godot` has no `[display]` section; stretch not pinned | **High (blocks UI)** | ADR-0004 |
| QQ-06 | Godot 4.7.2 not installed/located; cannot compile-verify locally | **High** | Install Godot .NET; mark all engine APIs `NEEDS VERIFICATION` until then |
| QQ-07 | C# test execution method (xUnit vs gdUnit4 C#) undecided | **High** | ADR-0005 / `/test-setup` |
| QQ-08 | Table GDD said Table reads State; this architecture uses Table View subscriptions | Medium | ADR-0001/0007 confirmation; behavior unchanged |
| QQ-09 | Model needs `Revision`/change notification for redraw; not explicit in GDD | Low | ADR-0003 confirmation; additive interface |
| QQ-10 | Non-MVP systems (Save/Animation/Audio/Settings/Accessibility) have no GDD | Medium | Design in their phases; current placeholders only |
| QQ-11 | Performance reference machine not pinned; `project.yaml` has no `performance` section | Medium | Use `.claude/docs/technical-preferences.md` targets: 60 fps, 16.6 ms, 200 draw calls, 512 MB |
| QQ-12 | String table path and default Chinese not pinned | Low | ADR-0008; proposed `assets/data/strings.csv` |

---

## Appendix A — Technical Requirements Baseline

> All IDs are **provisional**. The TR registry is empty; `/architecture-review`
> will assign permanent IDs.

### A.1 Card & Deck Model

| Req ID | Requirement |
|---|---|
|---|---|
| TR-card-deck-model-001 | 52 unique cards; `suit` + `rank`; all face-up; no `face_up` state |
| TR-card-deck-model-002 | 8 ordered tableau columns; 4 free cells (capacity 1); 4 suit-bound foundations (A→K) |
| TR-card-deck-model-003 | Invariants I1–I5 hold; failed operations are atomic (state fingerprint unchanged) |
| TR-card-deck-model-004 | Atomic move + immutable reads; internal remove/append not public write operations |
| TR-card-deck-model-005 | Full immutable snapshot/restore; schema-versioned serialize/deserialize; reject invalid states |
| TR-card-deck-model-006 | Read-only derived queries: top, count, empty counts, foundation height, `game_completion`; move result includes failure reason |
| TR-card-deck-model-007 | No Rules dependency, no RNG; caller gates legality before writing |
| TR-card-deck-model-008 | Same sequence + operation script → identical serialized state; O(1) reads, O(k) moves |

### A.2 Deal Generator

| Req ID | Requirement |
|---|---|
|---|---|
| TR-deal-generator-001 | Accept only integer N in [1, 1,000,000]; out-of-range returns `DEAL_NUMBER_OUT_OF_RANGE`, no partial output |
| TR-deal-generator-002 | Pure function; no time, process, engine RNG, or global state |
| TR-deal-generator-003 | 31-bit Microsoft LCG constants 214013/2531011; 64-bit intermediate; advance then read |
| TR-deal-generator-004 | Rank-major deck order C→D→H→S and `deck_index`; row-major deal then column-major transpose |
| TR-deal-generator-005 | Preserve `rand % (52−i)` modulo bias; 52 draws; forced last card; output matches `Model.deal` |
| TR-deal-generator-006 | Golden vectors and SHA-256 for N = 1, 617, 32,000, 32,001, 1,000,000, 11,982, 10,021 |
| TR-deal-generator-007 | O(52), ≤1 ms/call, ≤100 ms/1000; pure/thread-safe; algorithm-version string for saves |
| TR-deal-generator-008 | No solvability check; #11982 and other unsolvable deals generate normally |

### A.3 Rules & Move Validator

| Req ID | Requirement |
|---|---|
|---|---|
| TR-rules-move-validator-001 | Read-only model, no persistent state; same input → same result |
| TR-rules-move-validator-002 | Legal sequence = contiguous suffix, rank−1, alternating color; tableau target empty or rank+1 opposite color |
| TR-rules-move-validator-003 | Free cell top card only/capacity 1; foundation same suit next rank A→K, no move out; free-cell→free-cell illegal |
| TR-rules-move-validator-004 | Supermove capacity `(F+1)×2^E`; empty target `(F+1)×2^(E−1)`; cap 13; single card exempt |
| TR-rules-move-validator-005 | `legal_moves` complete, deduplicated, fixed order (target outer, length inner); free cells by slot; upper bound 768 |
| TR-rules-move-validator-006 | `auto_move_destination`: foundation first, then lowest-index empty free cell; top card only; no chain |
| TR-rules-move-validator-007 | `is_won`/`is_stuck`; win guard; empty-column count does not decide win |
| TR-rules-move-validator-008 | `MoveResult` reason codes and fixed validation order; `INVALID_MODEL` defensive, no clamp/negative shift |

### A.4 Game State Machine

| Req ID | Requirement |
|---|---|
| TR-game-state-machine-001 | Owns current N and lifecycle `NoGame/Dealing/Playing/Won/Stuck/Restoring`; `Error` is a command result |
| TR-game-state-machine-002 | Commands `new_game/restart/next/prev/select` with guards; same-N `select_deal` routes to restart |
| TR-game-state-machine-003 | Coordinates Deal→Model→Undo; synchronous events after state update; `GameStarted` exactly once per successful `new_game` |
| TR-game-state-machine-004 | Re-evaluate win/stuck after move/undo/load; win has priority |
| TR-game-state-machine-005 | New/restart/successful load clear Undo; failed load does not; `snapshot` contains N, model, state |
| TR-game-state-machine-006 | Load validates algorithm version and model invariants; failure is atomic; success emits `GameLoaded` |
| TR-game-state-machine-007 | Deal-number navigation clamps; `AT_LIMIT` is a no-op; `NO_CURRENT_GAME` when no game |
| TR-game-state-machine-008 | Errors do not change state; commands are non-reentrant; subscriber exceptions do not roll back; no random N, no file writes |

### A.5 Drag & Drop Interaction

| Req ID | Requirement |
|---|---|
|---|---|
| TR-drag-drop-interaction-001 | Pointer capture and `Idle/Pressed/Dragging/Dropping/Resolving/Cancelled`; only one drag at a time |
| TR-drag-drop-interaction-002 | 4 logical-pixel Euclidean threshold, strict `>`; grab offset cached before source hiding |
| TR-drag-drop-interaction-003 | Drag unit = contiguous suffix from clicked card to top; must pass `valid_sequence`; fixed during drag |
| TR-drag-drop-interaction-004 | 1:1 follow, no clamp, no scale/rotation; no legal-target highlight, no cursor change, no shadow |
| TR-drag-drop-interaction-005 | Release hit test → `can_move`; legal 0 ms; illegal 120 ms snapback + `Resolving ≤150 ms` watchdog |
| TR-drag-drop-interaction-006 | Legal path `record_before_move → move → commit → State.on_move_applied`; failure/cancel `discard_pending` |
| TR-drag-drop-interaction-007 | Double-click OS interval and ≤4 px movement; `auto_move_destination`; no drag layer, no chain |
| TR-drag-drop-interaction-008 | Cancel on Esc/right-click/focus loss/state change; model and State unchanged |
| TR-drag-drop-interaction-009 | Logical coordinates, 0-based bottom-to-top index; topmost-card hit priority; `_unhandled_input` not swallowed by toolbar |
| TR-drag-drop-interaction-010 | Events at most once per gesture; 60 fps; no per-frame allocation; toolbar command cancels drag first |

### A.6 Undo History

| Req ID | Requirement |
|---|---|
|---|---|
| TR-undo-history-001 | LIFO stack of immutable pre-move snapshots + `MoveResult`; depth 0–4096 |
| TR-undo-history-002 | Single pending slot; `record_before_move → commit/discard`; idempotent discard |
| TR-undo-history-003 | Undo only when `Playing/Stuck` and no pending; atomic restore; State re-evaluation; 0 ms |
| TR-undo-history-004 | Clear on new/restart/successful load; failed load preserves; empty-clear idempotent |
| TR-undo-history-005 | At depth 4096 evict `stack[0]` before push, emit `UndoEvicted`, no 4097 transient |
| TR-undo-history-006 | Restore failure discards bad record, model unchanged, emits `UndoFailed`; no redo, no persistence; ~2–2.5 MB cap |

### A.7 Hint System

| Req ID | Requirement |
|---|---|
| TR-hint-system-001 | `request_hint(model)` read-only; `HintResult` has `has_hint/move/reason/candidate_count/cycle_index`; `reset()` clears cycle |
| TR-hint-system-002 | Candidates only from `Rules.legal_moves`; filter whole-column permutation (tableau→empty tableau, count = height) |
| TR-hint-system-003 | Five tiers + exclusion; tie by `enumeration_index`; fully deterministic |
| TR-hint-system-004 | Cycle candidates with wrap; reset on list change/win/no candidates; undo to same board continues cycle |
| TR-hint-system-005 | No-hint reasons `GAME_WON`/`NO_USEFUL_MOVES`; `NO_USEFUL_MOVES` coincides with `is_stuck` |
| TR-hint-system-006 | No model writes, no `can_move`/`auto_move_destination`; no text generation |
| TR-hint-system-007 | Highlight contract `set_hint_highlight(source, destination)` / `clear_hint`; clear lifecycle owned by Toolbar/Table |
| TR-hint-system-008 | Worst-case P95 ≤1 ms, max ≤3 ms including one `legal_moves`; `NOT ASSESSED` until reference hardware is pinned |

### A.8 Statistics & Records

| Req ID | Requirement |
|---|---|
| TR-statistics-records-001 | Session-only in-memory counters; no file I/O; cleared on app close |
| TR-statistics-records-002 | Consumes `GameStarted`/`GameWon`; `GameRestarted`/`GameStuck` no-op; MVP does not subscribe `GameLoaded` |
| TR-statistics-records-003 | Unfinished active game + new `GameStarted` records loss; `games_played` includes active game; current streak resets |
| TR-statistics-records-004 | `GameWon` idempotent per active game; `games_won/current/longest` update together |
| TR-statistics-records-005 | `StatsSnapshot` five fields, immutable; `win_rate` derived on every read |
| TR-statistics-records-006 | `reset_stats` clears all and is idempotent; mid-game reset discards active game; int32 checked |
| TR-statistics-records-007 | O(1) events/queries; no locks/threads/event log; deterministic and synchronously visible |

### A.9 Table Rendering & Layout

| Req ID | Requirement |
|---|---|
| TR-table-rendering-layout-001 | Logical 1280×720 canvas; uniform scale `max(0.8, min(w/1280,h/720))`; letterbox; minimum window 1024×576; invertible transforms |
| TR-table-rendering-layout-002 | Cards 140×190; column pitch 152, margin 38; top y=38; tableau y0=256; cascade 22–30; all cards inside canvas |
| TR-table-rendering-layout-003 | Hit priority top row→tableau→null; `{pile, card_index}`; empty column −1; outside canvas null; 0-based bottom-to-top |
| TR-table-rendering-layout-004 | Atlas card faces with baked borders/corners; no runtime lines; pure `#008000` table; no card back/shadow/tint |
| TR-table-rendering-layout-005 | Drag layer above piles, below toolbar/overlays; source hidden same frame; hint highlight; drag layer transparent to hit test |
| TR-table-rendering-layout-006 | Redraw only on model/drag/resize/state changes; 60 fps; atlas; no per-frame allocation; idle screenshots identical |
| TR-table-rendering-layout-007 | HiDPI 1x/2x/3x atlas and mipmaps; top card rank ≥20 device px; covered index ≥10 device px |
| TR-table-rendering-layout-008 | Resize recomputes once; Drag rescales `grab_offset`; `NoGame` shows 16 empty slots; Won/Stuck overlays drawn by Toolbar |

### A.10 Toolbar & Menus

| Req ID | Requirement |
|---|---|
| TR-toolbar-menus-001 | Menu bar + shortcuts only; no toolbar/status bar; menu bar 28 window px, anchored top, not letterbox-scaled |
| TR-toolbar-menus-002 | Game/Help structure and F2/F3/F4/Ctrl+Z/F10, Alt+G/H, Esc/Enter shortcuts |
| TR-toolbar-menus-003 | Title `空当接龙 — 局号 #N`; `NoGame` title `空当接龙`; updated on GameStarted/Restarted/Loaded/StateChanged |
| TR-toolbar-menus-004 | `command_enabled` formula; drag disables New/Select/Restart/Undo/Hint; Stats/Exit/About remain enabled; refresh after events |
| TR-toolbar-menus-005 | Command routing: New NoGame→new_game(1) else next_deal; Select/Restart/Undo/Hint/Stats/Exit/About |
| TR-toolbar-menus-006 | Select, Statistics, Won, About dialogs; transparent full-window input blocker; one modal; Won queue |
| TR-toolbar-menus-007 | Stuck and message banners; non-modal; mutually exclusive with Stuck priority; size caps |
| TR-toolbar-menus-008 | Visible strings from string table + `tr()`; centralized error-code mapping; Hint highlight forwarding/clearing |
| TR-toolbar-menus-009 | 4.6 dual-focus keyboard reachability, visible focus, focus return; performance budget; retained screenshots |

---

## Appendix B — HIGH RISK API Verification Checklist

Before implementing any of the following, verify against
`docs/engine-reference/godot/` and record the result in the relevant ADR or
story.

| # | Item | Why | Source to check |
|---|---|---|---|
| 1 | Project stretch mode and aspect | 4.7 new-project defaults `canvas_items`/`expand`; GDD requires manual letterbox | `VERSION.md`, `breaking-changes.md`, `modules/ui.md` |
| 2 | Mouse/keyboard device IDs | 4.7 no longer uses `0`; some joypads use `0` | `breaking-changes.md`, `modules/input.md` |
| 3 | Dual-focus behavior | 4.6 separates mouse/touch focus from keyboard/gamepad focus | `modules/ui.md`, `modules/input.md` |
| 4 | `CanvasItem` line antialiasing | 4.7 lines have no AA feather; GDD forbids runtime lines | `breaking-changes.md`, `modules/rendering.md` |
| 5 | `FileAccess.store_*` return type | 4.4 returns `bool`; save code must check it | `breaking-changes.md` |
| 6 | `Control.accessibility_live` enum | 4.7 type moved to `AccessibilityServer` | `breaking-changes.md`, `modules/ui.md` |
| 7 | `RichTextLabel.add_image/update_image` | 4.7 parameter changes; only if used | `breaking-changes.md` |
| 8 | `DisplayServer.screen_get_scale()` / `Window` min size | HiDPI and minimum-window behavior need runtime verification | `modules/ui.md`, `modules/rendering.md` |
| 9 | gdUnit4 6.2.2 on 4.7.2 | README lists up to 4.7.1; engine reference observed 6.2.2 | `current-best-practices.md` |
| 10 | Installed Godot version | `NOT DETERMINED`; local compile is not proven | `VERSION.md` |

---

*End of architecture document.*
