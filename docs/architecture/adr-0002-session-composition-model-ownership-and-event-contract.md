# ADR-0002: Session composition, model ownership, and event contract

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
- `godot-specialist` (engine validation; first draft rejected, then revised)
- `technical-director` review skipped — lean mode

## Summary

The game needs one owner for the card model, one place that constructs the
object graph, and a communication contract that does not make State, Undo,
Statistics, Drag and Presentation depend on each other. We decided that
`GameBootstrap : Node` is the only constructor, `GameSession` is a pure C#
container that wires the constructed services, one `CardDeckModel` instance
lives for the whole session, and services communicate through direct typed C#
events with a shared command gate and isolated subscriber failures.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 / .NET 8 |
| **Domain** | Core |
| **Layer** | Foundation |
| **Knowledge Risk** | LOW for this decision — C# events, constructor injection and a command gate are BCL-level; no Godot 4.4–4.7 API is used. Project-level UI/Input/Rendering risk stays with ADR-0004/0006/0007. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `current-best-practices.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | `GD.PushError` C# binding; `_Ready` / `_ExitTree` lifecycle order; nullable build; event-order and reentrancy tests; architecture check including `GameSession.cs` |

> **Evidence note.** The installed editor is
> `4.7.2.stable.mono.official.ed1daf0bf`; its runtime config targets `net8.0`;
> .NET SDK `8.0.426` is installed. A temporary `Godot.NET.Sdk/4.7.2` project
> built with 0 warnings/0 errors and ran headless on 2026-10-24. C# toolchain
> behavior and `GD.PushError` are **NOT SOURCEABLE** in the current
> engine-reference library (no `modules/csharp.md`); treat these as local
> evidence, not engine-reference conclusions.

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (must be Accepted before this can be implemented) |
| **Enables** | ADR-0003, ADR-0005, ADR-0006, ADR-0007 |
| **Blocks** | State, Undo, Statistics, Drag and Toolbar implementation epics |
| **Ordering Note** | ADR-0001 must be written and Accepted first; ADR-0002 assumes its layer and project-layout rules |

## Context

### Problem Statement

ADR-0001 establishes the layers and the pure C# boundary, but not who owns the
model, who constructs the services, or how modules notify each other. Without
this decision, State will grow direct references to Statistics/Toolbar, Undo
will call State directly, Presentation will bind against a composition object,
and event loops or stale views will be discovered only during integration.

### Current State

- ADR-0001 is written and `Proposed`; its registry has 7 stances, including
  `card_pile_state → card-deck-model` and the ban on Godot types in pure layers.
- `docs/architecture/architecture.md` v1 defines the intended modules and data
  flows but contains stale text: it says Statistics consumes `GameRestarted`
  and `GameLoaded`, and it does not define the `GameSession` container or the
  event consumption matrix.
- No C# source exists yet; this ADR is the implementation contract.
- The first ADR-0002 draft was rejected by `godot-specialist` for eight
  blocking issues: State was placed in Core, `GameSession` became a second
  composition root, TableLayout/Drag were missing, logger placement was
  undefined, Undo/GDD ordering conflicted, load-to-Won semantics were unclear,
  reentrancy was incomplete, and `Bind()` risked a Presentation → App
  dependency. This ADR is the revised design.

### Constraints

- Foundation/Core/Feature must stay free of Godot types (ADR-0001).
- `GameBootstrap` is the only place that constructs concrete services;
  `GameSession` must not become a second composition root.
- Presentation must not reference `FreeCell.App`; `Bind()` takes only
  Foundation/Core/Feature interfaces.
- One `CardDeckModel` instance per session; State does not own pile data.
- Single-threaded, synchronous, deterministic; no event bus, no Autoload.
- Test code must not compile into the game assembly; unit tests must be able to
  construct the session without a scene tree.
- `Nullable` is enabled; event handlers must not be `async void`.

### Requirements

- One owner for model lifetime and one owner for each service.
- Direct, typed, synchronous events with a deterministic order.
- Exactly-once terminal events; load-to-Won must not emit `GameWon`.
- Subscriber exceptions must not roll back a completed command.
- Reentrant mutating commands must return `COMMAND_IN_PROGRESS`.
- Presentation binds through pure interfaces, never through `GameSession`.
- Lifecycle subscriptions must be explicit and leak-free.
- GDDs must be synchronized where this ADR changes the interface contract.

## Decision

### 1. App composition: one constructor, one pure container

`GameBootstrap : Node` is the **only** place that constructs concrete services.
It creates, in order:

1. `GodotGameLogger` (Presentation/Logging, plain class, `IGameLogger`)
2. `CardDeckModel`
3. `DealGenerator`
4. `RulesMoveValidator`
5. `TableLayout` (with the initial window size; updated on `Window.SizeChanged` per ADR-0004)
6. `UndoHistory(model, commandGate, logger)`
7. `GameStateMachine(model, deal, rules, undo, commandGate, logger)`
8. `HintSystem(rules)`
9. `Statistics()`
10. `DragDropController(state, rules, model, undo, layout, commandGate, logger)`
11. `GameSession` (receives the constructed services; does not `new` them)

`GameSession` is a pure C# container/facade in `FreeCell.App`:

- It holds one model instance and the constructed services.
- It wires internal subscriptions: State → Statistics, Undo → State.
- It exposes read-only service references for `GameBootstrap` to pass into
  Presentation `Bind()` calls.
- It implements `IDisposable` and unsubscribes its internal handlers.
- It does **not** construct concrete services, hold Godot objects, or raise
  events in its constructor.

`GameBootstrap` lifecycle:

- Constructs the object graph in `_Ready` with the initial window size. The
  size is not final: `TableLayout.UpdateViewport` and the TableView transform
  refresh on `Window.SizeChanged` (ADR-0004).
- Calls `Bind()` on `TableView`, `ToolbarController` and `DragInputAdapter`.
- Optionally calls `State.NewGame(1)` **after** `Bind()` if auto-start is chosen
  (see architecture QQ-01). No event may be raised before views are bound.
- `_ExitTree` calls `_session?.Dispose()`; Presentation owns its own event
  subscriptions and unsubscribes in its own `_ExitTree`.
- `GameBootstrap` is single-entry: it is the main scene root and is not removed
  and re-added. Restarting the game uses `ReloadCurrentScene`, which recreates
  the session; `_Ready` is not expected to run twice on the same instance.

This preserves the registry stance “GameBootstrap is the only composition
root”: `GameSession` is a wiring container, not a constructor.

### 2. Model ownership

- `GameBootstrap` constructs exactly one `CardDeckModel`; `GameSession` owns
  its session lifetime by holding the reference.
- State, Rules, Undo, Hint, Drag and TableView receive the same instance.
- State holds a reference but does not own pile data. New game/restart/load call
  `Model.Deal` / `Model.Restore`; they never create a second model.
- `UndoHistory` stores immutable snapshots; it does not own the model.
- `TableView` binds `ICardDeckReadModel` (read-only seam), not the mutable
  `ICardDeckModel`.
- `DragDropController` receives `ICardDeckModel` because it executes the
  approved move.

### 3. Direct typed C# events

- Services expose `event Action<T>?` with no `sender` parameter. The publisher
  is known from the service that exposes the event; `EventHandler<T>` adds no
  information here.
- Event names match the GDD exactly: `GameStarted`, `GameRestarted`,
  `GameWon`, `GameStuck`, `GameLoaded`, `StateChanged`, `UndoRecorded`,
  `UndoCommitted`, `UndoPerformed`, `UndoEvicted`, `UndoCleared`, `UndoFailed`,
  `DragStarted`, `DragMoved`, `DragDropped`, `DragCancelled`, `MoveExecuted`,
  `MoveRejected`.
- Handler methods use `On<EventName>`. The `project.yaml` `signals:
  on_event_name` rule applies to Godot signals in Presentation; pure C# events
  use PascalCase names and `On...` handlers.
- Payloads are immutable `readonly record struct` or `readonly record` types in
  the owning module or Foundation. `System.Numerics.Vector2` is used for pure
  pointer positions.
- No global event bus. No Godot `[Signal]` in Foundation/Core/Feature.
- Events are synchronous, ordered, non-reentrant and on the main thread.

### 4. Event order and exactly-once rules

State command:

```text
1. enter CommandGate
2. update internal state
3. if settled state changed: raise StateChanged(from, to)
4. raise the specific event (GameStarted / GameRestarted / GameLoaded /
   GameWon / GameStuck)
5. leave CommandGate
6. return Result
```

- `Dealing` and `Restoring` are transient internal states; they are not
  published as `StateChanged`. Only the settled state is published.
- `StateChanged` is raised only when `from != to`.
- `GameStarted` is raised exactly once per successful `new_game`.
- `GameRestarted` is raised exactly once per successful `restart_deal`.
- `GameLoaded` is raised exactly once per successful `load_game`.
- `GameWon` / `GameStuck` are raised exactly once per terminal entry caused by
  a move or undo. A load into `Won`/`Stuck` raises only `StateChanged` (if the
  state changed) and `GameLoaded`; it never raises `GameWon`/`GameStuck`.
- Repeated `OnMoveApplied` in a terminal state is rejected and raises nothing.
- A move/undo that leaves the state unchanged raises only `StateChanged` if the
  state actually changed; terminal events are not repeated.

Undo:

```text
1. enter CommandGate
2. Model.Restore(snapshot)
3. raise UndoPerformed(entry.move_result)
   -> GameStateMachine.OnUndoPerformed(e) [subscriber]
      -> re-evaluate Rules.IsWon / IsStuck
      -> raise StateChanged / GameWon / GameStuck
4. leave CommandGate
5. return Result
```

- If restore fails: raise `UndoFailed`, do **not** raise `UndoPerformed`, do
  not call `State.OnUndoPerformed`.
- `GameSession` subscribes State to `UndoPerformed` before any Presentation
  binding. This ordering invariant guarantees State re-evaluates before a view
  redraws.
- `GameStateMachine.OnUndoPerformed(UndoPerformed e)` returns `void`. It is a
  contractually non-failing internal transition: it only re-evaluates pure
  predicates on a valid restored model. If it throws, the error is logged as
  critical and the session is considered faulted; the game must be restarted.
- State still depends on `IUndoHistory.Clear()` for new/restart/load; Undo does
  not depend on State.

### 5. Statistics wiring

`GameSession` wires only:

- `State.GameStarted → Statistics.OnGameStarted`
- `State.GameWon → Statistics.OnGameWon`

It does **not** wire `GameRestarted`, `GameStuck` or `GameLoaded` to Statistics.
Statistics keeps its own no-op methods for the first two for defensive
compatibility; MVP does not subscribe `GameLoaded`. This matches
`statistics-records.md`; the architecture document's stale text is corrected in
the same changeset.

### 6. Subscriber exceptions and logging

- `SafeRaise` in Foundation invokes each subscriber from
  `GetInvocationList()`, casts to `Action<T>` and invokes directly (never
  `DynamicInvoke`).
- Each subscriber is wrapped in `try/catch`; an exception is logged via
  `IGameLogger` and the remaining subscribers still run.
- A raised command is not rolled back; the owner has already updated its state.
- Event handlers must not be `async void`; asynchronous work is not permitted in
  the event path.
- High-frequency `DragMoved` may use direct single-subscriber invocation, but
  it must still isolate exceptions and must not use reflection.
- `IGameLogger` lives in Foundation. `GodotGameLogger` lives in
  `src/Presentation/Logging/`, is a plain class (not a `Node`), and implements
  `GD.PushError`. `GameBootstrap` creates it and injects `IGameLogger`.
- `NullGameLogger` is the test implementation.

### 7. Shared command gate

A single `CommandGate` in Foundation is shared by State and Undo:

```csharp
public sealed class CommandGate
{
    public bool IsBusy { get; }
    public IDisposable? TryEnter();   // null when busy
}
```

- `NewGame`, `RestartDeal`, `NextDeal`, `PreviousDeal`, `SelectDeal`,
  `OnMoveApplied`, `OnUndoApplied` and `Undo.Undo` enter the gate.
- The gate is held through event dispatch, so a subscriber that calls a
  mutating command receives `COMMAND_IN_PROGRESS`; the original command
  completes.
- `Model.Deal` and `Model.Restore` are only called inside a held gate.
- `State.OnUndoPerformed` is an internal transition invoked while Undo holds the
  gate; it must not try to enter the gate again.
- `SafeRaise` does not detect reentrancy; the gate is the only guard.

### 8. Bind contracts

Presentation binds only to Foundation/Core/Feature interfaces; it never binds
to `FreeCell.App.GameSession`.

```csharp
public void TableView.Bind(
    ICardDeckReadModel model,
    TableLayout layout,
    IGameStateMachine state,
    IUndoHistory undo,
    IDragDropController drag,
    IGameLogger logger);

public void Toolbar.Bind(
    IGameStateMachine state,
    IUndoHistory undo,
    IHintSystem hint,
    IStatistics statistics,
    TableView tableView,
    IDragDropController drag);

public void DragInputAdapter.Bind(
    IDragDropController drag,
    TableLayout layout);
```

- `TableView` uses `ICardDeckReadModel`; only Drag receives the mutable
  `ICardDeckModel`.
- `Bind()` is idempotent: it unsubscribes previous handlers before subscribing
  new ones, or asserts it is called once.
- `DragInputAdapter` calls `IDragDropController.NotifyLayoutChanged(logicalPointer)`
  after a window resize; it is a command, not an event, and is not in the
  event matrix.
- Presentation subscribes in `Bind()` and unsubscribes in `_ExitTree()`.
- `GameBootstrap` does not unsubscribe Presentation handlers; each owner does.

### 9. Event consumption matrix

| Event | Producer | Consumers | Guarantee |
|---|---|---|---|
| `StateChanged` | State | Toolbar (refresh/hide), TableView (mode), Drag (cancel on state change) | Only `from != to`; no transient `Dealing`/`Restoring` |
| `GameStarted` | State | Statistics, Toolbar, TableView | Exactly once per successful `new_game` |
| `GameRestarted` | State | Toolbar, TableView | Exactly once per successful restart |
| `GameWon` | State | Statistics, Toolbar (Won dialog), Audio (VS) | Exactly once per terminal entry from move/undo; never from load |
| `GameStuck` | State | Toolbar (Stuck banner), Audio (VS) | Exactly once per terminal entry from move/undo; never from load |
| `GameLoaded` | State | Toolbar, TableView | Exactly once per successful load; never raises `GameWon`/`GameStuck` |
| `UndoRecorded` / `UndoCommitted` / `UndoCleared` / `UndoEvicted` | Undo | Toolbar (refresh), debug | Per operation; no State re-evaluation |
| `UndoPerformed` | Undo | State (re-evaluate), Toolbar, TableView, Audio (VS) | Exactly once per successful undo |
| `UndoFailed` | Undo | Toolbar, debug | Per failed undo; State is not re-evaluated |
| `DragStarted` / `DragDropped` / `DragCancelled` | Drag | TableView, Toolbar | Once per gesture |
| `DragMoved` | Drag | TableView | Many per gesture |
| `MoveExecuted` | Drag | Audio (VS), Animation (VS) | Once per successful move; **not** consumed by State |
| `MoveRejected` | Drag | Audio (VS), debug | Once per rejected move |

Move path: Drag calls `State.OnMoveApplied(result)` directly; `MoveExecuted` is
a post-state notification for Audio/Animation, not a State trigger.

### Architecture Diagram

```text
GameBootstrap : Node (only constructor)
  ├── GodotGameLogger : IGameLogger        [Presentation/Logging]
  ├── CardDeckModel                         [Foundation]
  ├── DealGenerator, RulesMoveValidator     [Core]
  ├── TableLayout                           [Core]
  ├── UndoHistory, GameStateMachine         [Feature]
  ├── HintSystem, Statistics                [Feature]
  ├── DragDropController                    [Feature]
  └── GameSession (pure container/wiring)  [App]
        ├── holds the one model
        ├── wires State -> Statistics
        ├── wires Undo -> State
        └── exposes services to Bind()

GameBootstrap._Ready()
  ├── construct services + GameSession
  ├── TableView.Bind(ICardDeckReadModel, TableLayout, State, Undo, Drag, logger)
  ├── Toolbar.Bind(State, Undo, Hint, Statistics, TableView, Drag)
  ├── DragInputAdapter.Bind(Drag, TableLayout)
  └── optional State.NewGame(1) after Bind

GameBootstrap._ExitTree()
  └── GameSession.Dispose()  [internal subscriptions only]
```

### Key Interfaces

```csharp
namespace FreeCell.Foundation;

public interface IGameLogger
{
    void Error(string message, Exception? exception = null);
}

public interface ICardDeckReadModel
{
    int Revision { get; }
    bool IsDealt { get; }
    int Count(PileId pile);
    Card? TopCard(PileId pile);
    ImmutableArray<Card> Cards(PileId pile);
    int FoundationHeight(PileId foundation);
    int EmptyFreeCellCount { get; }
    int EmptyTableauColumnCount { get; }
    bool GameComplete { get; }
}

public interface ICardDeckModel : ICardDeckReadModel
{
    ModelMoveResult Move(PileId source, PileId destination, int count);
    ModelSnapshot Snapshot();
    SnapshotRestoreResult Restore(ModelSnapshot snapshot);
    Result Deal(DealSequence sequence);
}

public sealed class CommandGate
{
    public bool IsBusy { get; }
    public IDisposable? TryEnter();
}


namespace FreeCell.Core;

public interface IDealGenerator { /* Generate(N) */ }
public interface IRulesMoveValidator { /* CanMove/LegalMoves/... */ }
public sealed class TableLayout { /* pure geometry */ }


namespace FreeCell.Feature;

public interface IGameStateMachine
{
    event Action<GameStarted>? GameStarted;
    event Action<GameRestarted>? GameRestarted;
    event Action<GameWon>? GameWon;
    event Action<GameStuck>? GameStuck;
    event Action<GameLoaded>? GameLoaded;
    event Action<StateChanged>? StateChanged;

    void OnUndoPerformed(UndoPerformed e);
    // commands: NewGame/RestartDeal/NextDeal/PreviousDeal/SelectDeal/
    // OnMoveApplied/OnUndoApplied/Snapshot/LoadGame
}

public interface IUndoHistory
{
    event Action<UndoPerformed>? UndoPerformed;
    event Action<UndoFailed>? UndoFailed;
    // RecordBeforeMove/Commit/DiscardPending/Undo/Clear/CanUndo/Depth
}

public interface IHintSystem { /* RequestHint/Reset */ }
public interface IStatistics { /* OnGameStarted/OnGameWon/GetStats/ResetStats */ }

public interface IDragDropController
{
    void NotifyLayoutChanged(Vector2 logicalPointer);   // command, not event
    // OnPointer* + drag events
}


namespace FreeCell.Presentation;

public partial class TableView : Node2D
{
    public void Bind(ICardDeckReadModel model, TableLayout layout,
        IGameStateMachine state, IUndoHistory undo,
        IDragDropController drag, IGameLogger logger);
}

public partial class ToolbarController : Control
{
    public void Bind(IGameStateMachine state, IUndoHistory undo,
        IHintSystem hint, IStatistics statistics,
        TableView tableView, IDragDropController drag);
}

public partial class DragInputAdapter : Node
{
    public void Bind(IDragDropController drag, TableLayout layout);
}

public sealed class GodotGameLogger : IGameLogger
{
    public void Error(string message, Exception? exception = null)
        => GD.PushError(exception is null ? message : $"{message}\n{exception}");
}


namespace FreeCell.App;

public sealed class GameSession : IDisposable
{
    public ICardDeckModel Model { get; }
    public IDealGenerator Deal { get; }
    public IRulesMoveValidator Rules { get; }
    public TableLayout Layout { get; }
    public IUndoHistory Undo { get; }
    public IGameStateMachine State { get; }
    public IHintSystem Hint { get; }
    public IStatistics Statistics { get; }
    public IDragDropController Drag { get; }
}

public partial class GameBootstrap : Node { /* constructs, binds, disposes */ }
```

### Implementation Guidelines

**Must:**

- Keep `GameBootstrap` the only place that constructs concrete services.
- Pass constructed services into `GameSession`; `GameSession` must not `new`
  a service.
- Construct in `_Ready` after the viewport size is known; call `Bind()` before
  any `State.NewGame`.
- Bind Presentation with pure interfaces only; never pass `GameSession`.
- Use `ICardDeckReadModel` in `TableView`; only Drag gets `ICardDeckModel`.
- Use direct `Action<T>` events; invoke subscribers through `SafeRaise`.
- Hold `CommandGate` across state update **and** event dispatch.
- Subscribe Presentation in `Bind()` and unsubscribe in `_ExitTree()`.
- Keep `GameSession` free of `Godot`/`GodotSharp` references.
- Enable `<Nullable>enable</Nullable>` in `FreeCell.csproj`.
- Use `public partial class GameBootstrap`.
- Use `NullGameLogger` in unit tests.
- Extend the ADR-0001 architecture check to include
  `src/App/GameSession.cs` as pure C#; only `src/App/GameBootstrap.cs` may
  reference Godot in App.

**Must never:**

- Let `GameSession` construct services or become a second composition root.
- Put `IGameStateMachine` in `FreeCell.Core`; it belongs to `FreeCell.Feature`.
- Let Presentation bind `FreeCell.App.GameSession` or any App type.
- Use a global event bus, Godot signals in pure layers, or Autoload.
- Allow `async void` event handlers or `Delegate.DynamicInvoke`.
- Re-enter a mutating command from an event handler.
- Raise `GameWon`/`GameStuck` from `LoadGame`.
- Wire `GameRestarted`, `GameStuck` or `GameLoaded` to Statistics in MVP.
- Let `GameBootstrap` unsubscribe Presentation handlers owned by adapters.
- Let `GameSession` hold a `Node` or `GodotObject`.
- Raise events from the `GameSession` constructor.

## Alternatives Considered

### Alternative 1: Injected typed event bus

- **Description**: Services depend on `IEventBus.Publish<T>/Subscribe<T>`; no
  direct event fields.
- **Pros**: Strong decoupling; easy to add cross-cutting subscribers.
- **Cons**: Hidden subscription graph; harder to trace event order; extra fake
  in every test; a global bus is one step from a service locator.
- **Estimated Effort**: Medium.
- **Rejection Reason**: A small game has a known, finite event set; direct
  events plus `GameSession` wiring are easier to audit and test.

### Alternative 2: Godot signals for cross-layer events

- **Description**: Services inherit `Node` and declare `[Signal]` delegates.
- **Pros**: Godot-native; editor-inspectable.
- **Cons**: Forces Foundation/Core/Feature to be Godot objects, violating
  ADR-0001; signals cannot be declared in pure C# classes; scene-tree lifecycle
  becomes part of gameplay.
- **Estimated Effort**: High and architecturally invalid.
- **Rejection Reason**: Direct C# events are the only option consistent with the
  pure-core boundary.

### Alternative 3: Direct method calls with no events

- **Description**: State calls Statistics and Toolbar directly; Undo calls
  State; Presentation polls or is poked by State.
- **Pros**: Fewest abstractions; easy to follow a single call stack.
- **Cons**: State knows every consumer; new consumers require editing State;
  circular dependencies; views cannot be tested independently.
- **Estimated Effort**: Low initially, high maintenance.
- **Rejection Reason**: Violates one-way dependencies and makes State a god
  object.

### Alternative 4: `GameSession` constructs the services

- **Description**: `GameBootstrap` creates `GameSession`, and `GameSession`
  constructs all gameplay services.
- **Pros**: Single call site for object construction.
- **Cons**: Creates a second composition root, contradicting ADR-0001 and the
  registry stance that `GameBootstrap` is the only constructor.
- **Estimated Effort**: Low.
- **Rejection Reason**: Rejected by the engine specialist as a registry
  conflict. `GameBootstrap` constructs; `GameSession` only wires and holds.

## Consequences

### Positive

- One model instance and one object-graph constructor; no hidden second root.
- State, Undo, Statistics and Presentation do not know each other's concrete
  types; subscriptions are visible in `GameSession` and `Bind()`.
- Event order and exactly-once semantics are testable without a scene tree.
- `CommandGate` prevents nested commands from event handlers.
- `ICardDeckReadModel` gives Presentation a read-only seam.
- `GameSession` can be unit-tested with `NullGameLogger`.

### Negative

- Manual subscription wiring; adding an event requires updating the matrix and
  tests.
- `SafeRaise` allocates an invocation-list array per event; acceptable because
  events are not per-frame, but `DragMoved` should avoid unnecessary work.
- Two App types (`GameBootstrap` and `GameSession`) require a documented split.
- GDDs must be kept in sync when event names or ordering change.

### Neutral

- `Action<T>` has no `sender`; the publisher is identified by the service.
- One model instance per session; a new game resets the same instance.
- `GameSession` is a container, not a service locator; it is passed only to
  `GameBootstrap`, never to Presentation.

## Risks

| Risk | Mitigation |
|---|---|
| `GameSession` becomes a second constructor | Constructor takes already-built services; architecture check + review |
| State placed in Core | `IGameStateMachine` lives in Feature; Feature may reference Core, not vice versa |
| TableLayout/Drag missing from session | Session includes both; `GameBootstrap._Ready` constructs them with the initial window size and updates on resize |
| `GodotGameLogger` in App uses Godot | Place in `src/Presentation/Logging/`; App `GameSession` stays pure |
| Undo/GDD ordering conflict | Sync `undo-history.md` and `game-state-machine.md`; `OnUndoPerformed` returns void |
| Load-to-Won emits `GameWon` | `LoadGame` only raises `StateChanged`/`GameLoaded`; explicit test |
| Nested Undo/State command from handler | Shared `CommandGate` held through event dispatch |
| `Bind()` receives `GameSession` | Bind signatures use pure interfaces; architecture check bans App types in Presentation |
| `_Ready` runs once; scene re-add leaves no bind | `GameBootstrap` is single-entry; restart uses `ReloadCurrentScene` |
| Presentation unsubscribes twice | Presentation owns its subscriptions; GameBootstrap only disposes session |
| `State.OnUndoPerformed` throws | Contractually non-failing; log critical and mark session faulted |
| `async void` escapes `SafeRaise` | Ban in implementation guidelines; code review/analyzer |
| `GetInvocationList` allocations on `DragMoved` | Single-subscriber direct invocation; no reflection |
| `GD.PushError` binding unknown | Verify against local editor; local build evidence only |
| Nullable warnings break 0-warning gate | `<Nullable>enable</Nullable>` and treat warnings as errors in CI |
| Architecture check misses `GameSession.cs` | Extend ADR-0001 check to scan `src/App/GameSession.cs` as pure |

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `game-state-machine.md` | Synchronous lifecycle events; `GameStarted` once; win/stuck once; load-to-Won only `GameLoaded` (TR-game-state-machine-003/004) | Event order and exactly-once rules |
| `undo-history.md` | Undo restores model and triggers State re-evaluation (TR-undo-history-003/006) | `UndoPerformed` event; `OnUndoPerformed`; `UndoFailed` |
| `statistics-records.md` | Consumes `GameStarted`/`GameWon`; no `GameLoaded` (TR-statistics-records-002/004/007) | GameSession wiring only those two events |
| `drag-drop-interaction.md` | Legal path calls State after model move; events once per gesture (TR-drag-drop-interaction-006/010) | Direct `State.OnMoveApplied`; `MoveExecuted` for Audio/Animation |
| `table-rendering-layout.md` | Redraw on State/Undo/Drag events (TR-table-rendering-layout-005/008) | Bind subscribes pure interfaces; `ICardDeckReadModel` |
| `toolbar-menus.md` | Command gating and Won/Stuck UI (TR-toolbar-menus-004/009) | `GameWon`/`GameStuck` one-shot; StateChanged refresh/close |
| `hint-system.md` | Hint reset on new/restart/load/undo, owned by Toolbar (TR-hint-system-004) | Toolbar calls `Reset()`; GameSession does not subscribe Hint |
| `card-deck-model.md` | One authoritative model; write only after Rules (TR-card-deck-model-004/007) | One model instance; read-only seam for TableView |

All 76 provisional TRs are covered indirectly by the composition and event
rules; `/architecture-review` builds the formal traceability matrix.

GDD sync applied on 2026-10-24:

- `undo-history.md`: Undo raises `UndoPerformed`; State subscribes.
- `game-state-machine.md`: `OnUndoPerformed` contract; load-to-Won semantics.
- `toolbar-menus.md`: Won/Stuck UI is triggered by `GameWon`/`GameStuck`;
  `StateChanged` refreshes/closes.
- `drag-drop-interaction.md`: State is called directly; `MoveExecuted` is for
  Audio/Animation.

## Performance Implications

- **CPU**: Direct delegate invocation; no reflection, no `DynamicInvoke`, no
  per-frame LINQ. `SafeRaise` allocates one `Delegate[]` per event; negligible
  for move/new-game/terminal events. `DragMoved` may invoke one subscriber
  directly.
- **Memory**: One model instance and one service graph; no event bus or
  per-event history. `GameSession` holds references, not copies.
- **Load Time**: Negligible; no extra assemblies or reflection scanning.
- **Network**: Not applicable.

## Migration Plan

No code exists yet. The first implementation story must:

1. Add `CommandGate`, `IGameLogger`, `ICardDeckReadModel` to Foundation.
2. Add `GameSession` and `GameBootstrap` to App.
3. Add the typed events and `SafeRaise`.
4. Wire State/Statistics/Undo in `GameSession`.
5. Add `Bind()` signatures using pure interfaces.
6. Extend the ADR-0001 architecture check to include `GameSession.cs`.
7. Sync the four GDDs listed above.
8. Add tests for event order, exactly-once, load-to-Won, reentrancy and
   subscriber exceptions.

**Rollback plan**: If direct events prove unworkable, write a superseding ADR
that introduces an injected event bus; `GameSession` is the single place that
would need to change wiring. No gameplay rules change.

## Validation Criteria

- `dotnet build FreeCell.csproj` completes with 0 errors and 0 warnings
  (`Nullable` enabled).
- `GameSession` contains no `Godot`/`GodotSharp` reference.
- `TableLayout` and `DragDropController` are constructed and reachable from
  `GameSession`.
- `GameBootstrap._Ready` constructs, binds, and only then optionally calls
  `NewGame(1)`.
- Unit tests prove:
  - `StateChanged` fires before `GameWon`.
  - `GameWon`/`GameStuck` fire once per terminal entry, never on load.
  - `GameStarted`/`GameRestarted`/`GameLoaded` fire once per successful command.
  - `UndoPerformed` causes exactly one State re-evaluation; `UndoFailed` causes
    none.
  - A subscriber exception does not roll back state or stop later subscribers.
  - A mutating command called from an event handler returns
    `COMMAND_IN_PROGRESS`.
  - `Bind()` accepts no `FreeCell.App` type.
- Architecture check fails if `GameSession.cs` references Godot, or if any
  Presentation `Bind()` signature references `FreeCell.App`.
- `registry.sh validate` reports no new duplicate/status/layout issues.
- The four GDD sync edits are present.

## Related

- ADR-0001 — layered boundaries and pure C# core
- ADR-0003 — model snapshot and serialization schema
- ADR-0005 — C# test execution strategy
- ADR-0006 — input routing and 4.6 dual focus
- ADR-0007 — Table drawing, atlas and HiDPI
- `docs/architecture/architecture.md` §3.6, §4.1, §4.5, §4.8, §5.5–§5.9
- `design/gdd/game-state-machine.md`, `undo-history.md`, `statistics-records.md`,
  `drag-drop-interaction.md`, `toolbar-menus.md`, `table-rendering-layout.md`,
  `hint-system.md`
- `docs/registry/architecture.yaml`
