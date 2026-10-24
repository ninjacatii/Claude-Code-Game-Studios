# ADR-0003: Model snapshot and serialization schema

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

Undo, Save/Resume and the model's invariant tests need one immutable snapshot
format and one versioned JSON payload, not separate deep copies and ad-hoc
serializers. We decided on an immutable `ModelSnapshot` owned by
`CardDeckModel`, shared structurally between snapshots, validated through a
single private-constructor factory, and serialized through explicit
`System.Text.Json` DTOs with schema version 1.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 / .NET 8 |
| **Domain** | Core |
| **Layer** | Foundation |
| **Knowledge Risk** | LOW for this decision — it uses BCL types only (`System.Collections.Immutable`, `System.Text.Json`) and no Godot API. The C#/.NET toolchain and export behavior are **NOT SOURCEABLE** in the current engine reference. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `current-best-practices.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | `dotnet build`; `JsonUnmappedMemberHandling` behavior; `required` DTO members; `ImmutableArray` default/empty handling; `"faceUp"` rejection; JSON round-trip; export-package save/load; content equality/hash tests |

> **Evidence note.** Godot 4.7.2 mono and .NET SDK 8.0.426 are installed; a
> temporary Godot C# project built and ran headless on 2026-10-24. That proves
> the toolchain, not `System.Text.Json`, `ImmutableArray` semantics, trimming or
> export behavior. Those remain local verification items.

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001, ADR-0002 (both must be Accepted before this can be implemented) |
| **Enables** | ADR-0009 (Save/Resume file format), ADR-0005 (unit-test fixtures) |
| **Blocks** | Card & Deck Model, Undo History and Save/Resume implementation epics |
| **Ordering Note** | ADR-0001 and ADR-0002 must be written and Accepted first; this ADR owns the snapshot and payload shape, not file location or atomic file replacement |

## Context

### Problem Statement

Undo needs up to 4096 immutable states, Save/Resume needs a versioned payload,
and tests need a deterministic state fingerprint. Without one snapshot contract,
the model will deep-copy on every move, Undo will exceed its memory target, and
Save/Resume will invent a second serialization path.

### Current State

- ADR-0001 and ADR-0002 are written and `Proposed`.
- `docs/registry/architecture.yaml` has 14 stances, including
  `card_pile_state → card-deck-model` and the ban on Godot types in pure layers.
- The GDDs contain four conflicts this ADR must resolve:
  1. `undo-history.md` says snapshots are "not shared"; structural sharing
     contradicts that wording.
  2. `card-deck-model.md` says a dealt model rejects a second `Deal`; State's
     `new_game`/`restart_deal` need to replace the deal on the same model.
  3. `deal-generator.md` says Save stores only the deal number and regenerates;
     `game-state-machine.md` says Save stores the full `GameSnapshot`.
  4. `architecture.md` still lists `ICardDeckModel.Serialize()` as a second
     serialization path and uses `int Revision`, while the draft snapshot used
     `long`; `Suit` is int-backed in architecture and was proposed as byte.
- The first ADR-0003 draft received a `CONCERNS` verdict: public snapshot
  construction, incomplete enum validation, `ImmutableArray` default handling,
  undefined equality/fingerprint, unresolved GDD conflicts, missing
  `GameSnapshotDto`, error-code mapping gaps, undefined payload grammar, and
  unverified `JsonUnmappedMemberHandling` behavior.

### Constraints

- Foundation/Core/Feature must not reference Godot; JSON must be
  `System.Text.Json`, never `Godot.Json`.
- `ModelSnapshot` must be immutable; callers must not mutate it.
- Undo memory target: roughly 2–3 MB for 4096 snapshots; the GDD's 2–2.5 MB is
  an estimate, not a guarantee.
- Snapshots must support a deterministic state fingerprint.
- Serialization must reject unknown fields, invalid enums, malformed card
  tokens and invariant violations; no partial load.
- `GameState` and `Suit`/`Rank` values must be validated with `Enum.IsDefined`.
- `SchemaVersion` is separate from the deal algorithm version.

### Requirements

- One `ModelSnapshot` type used by Undo, Save/Resume and tests.
- O(1) `Snapshot()`, structural sharing of unchanged piles.
- Private construction; only a validating factory can create a snapshot.
- Deterministic `ContentEquals`, `ContentHash` and canonical JSON.
- Explicit DTOs; no domain-type serialization.
- Atomic `Restore`; failure leaves the current model unchanged.
- Full `GameSnapshot` payload (model + N + state + versions), not deal-number-only.
- `LoadGame` re-evaluates the state and emits only `StateChanged`/`GameLoaded`.

## Decision

### 1. Snapshot model

- `Suit` stays int-backed (matching `architecture.md`); `Rank` is byte-backed.
  Do not claim `sizeof(Card) == 2`; `Card` size is an implementation detail.
- `CardDeckModel` stores an immutable `ModelSnapshot` plus an `int Revision`.
- `ModelSnapshot` is a sealed class with a **private constructor** and a single
  public factory:
  ```csharp
  public sealed class ModelSnapshot : IEquatable<ModelSnapshot>
  {
      public bool IsDealt { get; }
      public ImmutableArray<ImmutableArray<Card>> Piles { get; }  // 16 piles
      public ImmutableArray<Suit> FoundationSuits { get; }        // 4, fixed C,D,H,S

      public static SnapshotCreateResult Create(
          bool isDealt,
          ImmutableArray<ImmutableArray<Card>> piles,
          ImmutableArray<Suit> foundationSuits);

      public SnapshotValidation Validate();
      public bool ContentEquals(ModelSnapshot? other);
      public ulong ContentHash { get; }
      public string CanonicalJson();   // delegates to ModelSnapshotSerializer
  }
  ```
- Pile order is fixed: tableau 0–7, free cells 8–11, foundations 12–15.
  `FoundationSuits` is exactly `[Clubs, Diamonds, Hearts, Spades]` for schema 1.
- `Snapshot()` returns the current `ModelSnapshot` in O(1). A move replaces only
  the source and destination inner arrays and the outer 16-element array; all
  other inner arrays are shared. Snapshots never mutate an array in place.
- `Deal` is an **atomic replace** (user decision): it builds a new
  `ModelSnapshot` from the 52-card column-major sequence, validates it, and
  swaps it in. A dealt model may be re-dealt; invalid input leaves the previous
  state unchanged. This supersedes `card-deck-model.md`'s "re-deal fails" AC.
- `Move` checks structural preconditions first (non-empty source, `1 ≤ count ≤
  height`, source ≠ destination, free-cell capacity, foundation rules), builds a
  new snapshot for the changed piles, validates it, then swaps.
- `Restore(snapshot)` rejects `null`, validates the entire snapshot, then swaps.
  Failure leaves the current state unchanged. `Restore` increments `Revision`.
- `Revision` is `int`, increments on successful `Deal`, `Move` or `Restore`, and
  is not stored in `ModelSnapshot`. Snapshot equality/hash never depends on
  `Revision`.
- `SnapshotCreateResult`, `SnapshotValidation`, `SnapshotViolation` and
  `SnapshotRestoreResult` live in Foundation. `Restore` returns
  `SnapshotRestoreResult` with `ErrorCode`, `Violations` and `Diagnostic`.

### 2. Equality and fingerprint

- `ImmutableArray<T>` equality is reference-based; never use it directly for
  snapshot content equality. `ContentEquals` compares:
  1. `IsDealt`
  2. outer pile count (must be 16)
  3. each inner pile length and each card in order
  4. `FoundationSuits` element by element
- `ContentHash` is deterministic FNV-1a 64-bit over `IsDealt`, each pile length,
  each card's `(byte)Rank` and `(byte)Suit`, and each foundation suit. Do not use
  `HashCode.Combine`, `object.GetHashCode()` or `string.GetHashCode()`; those are
  process-randomized.
- `ModelSnapshot` overrides `Equals`/`GetHashCode` and `==`/`!=` to content
  semantics so tests and dictionaries cannot accidentally use reference equality.
- `CanonicalJson()` is the byte-level fingerprint for GDD acceptance:
  `ModelSnapshotSerializer.Serialize(snapshot)` with deterministic options.
  `ContentHash` is a fast bucket/fingerprint only; it cannot replace
  `ContentEquals`.

### 3. Snapshot validation

`Validate()` checks, in order:

1. `Piles` and `FoundationSuits` are not `default` and have exact lengths
   (16 and 4).
2. Every `Suit`/`Rank` value is defined (`Enum.IsDefined`), not a casted integer.
3. `FoundationSuits` is exactly C, D, H, S in schema 1.
4. If `IsDealt == false`: every pile is empty.
5. If `IsDealt == true`: total cards = 52; every card is unique. Use a 52-bit
   `ulong` bitmap keyed by `(rank−1)×4 + suitIndex`; no LINQ.
6. Free cells contain 0 or 1 card.
7. Each foundation has 0–13 cards, all matching its bound suit, contiguous
   A..K with no gap.
8. No `face_up`/visibility state is represented at all.
9. Tableau columns have no upper capacity limit.

Validation is O(52), called on `Deal`, `Move`, `Restore` and deserialization.
Move is input-driven, not per frame; the cost is negligible. Validation must not
run in `_Process`.

### 4. Payload format

- `SchemaVersion = 1`. It describes the JSON shape and card codec only.
  `AlgorithmVersion` describes the deal algorithm; both are checked on load.
  A change to either is a breaking save change; there is no migration in MVP.
- Card grammar: exactly two uppercase characters, rank then suit:
  - rank: `A`, `2`–`9`, `T` (10), `J`, `Q`, `K`
  - suit: `C`, `D`, `H`, `S`
  - examples: `AS`, `TD`, `7H`; reject lowercase, `10D`, separators and
    unknown tokens.
- `ModelSnapshotDto` uses `List<T>`/arrays, not `ImmutableArray<T>`, so missing
  fields cannot become `default(ImmutableArray)`:
  ```csharp
  public sealed class ModelSnapshotDto
  {
      public required int SchemaVersion { get; init; }
      public required bool IsDealt { get; init; }
      public required List<List<string>> Tableau { get; init; }       // exactly 8
      public required List<string?> FreeCells { get; init; }          // exactly 4, null = empty
      public required List<List<string>> Foundations { get; init; }   // exactly 4
      public required List<string> FoundationSuits { get; init; }     // exactly 4
      [JsonExtensionData] public Dictionary<string, JsonElement>? Extra { get; init; }
  }
  ```
  Inner tableau/foundation arrays are bottom-to-top. `IsDealt == false` still
  uses eight empty tableau arrays, four `null` free cells and four empty
  foundation arrays.
- `GameSnapshotDto`:
  ```csharp
  public sealed class GameSnapshotDto
  {
      public required int SchemaVersion { get; init; }
      public required string AlgorithmVersion { get; init; }
      public required int DealNumber { get; init; }
      public required GameState State { get; init; }
      public required ModelSnapshotDto Model { get; init; }
      [JsonExtensionData] public Dictionary<string, JsonElement>? Extra { get; init; }
  }
  ```
- `GameSnapshot` is the domain record:
  ```csharp
  public sealed record GameSnapshot(
      int SchemaVersion,
      string AlgorithmVersion,
      int DealNumber,
      GameState State,
      ModelSnapshot Model);
  ```
  `State` is advisory. `LoadGame` restores the model and N, re-evaluates
  `Rules.IsWon/IsStuck`, and emits only `StateChanged`/`GameLoaded`; it never
  emits `GameWon`/`GameStuck`.
- `ModelSnapshotSerializer` lives in Foundation; `GameSnapshotSerializer` lives
  in `Feature/State/Serialization/`. Both use one shared static
  `JsonSerializerOptions`:
  - camelCase
  - `PropertyNameCaseInsensitive = false`
  - `AllowTrailingCommas = false`
  - `ReadCommentHandling = Disallow`
  - `NumberHandling = Strict`
  - `WriteIndented = false`
  - `UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow`
  - `JsonStringEnumConverter(allowIntegerValues: false)` for `GameState`
  - no `DefaultIgnoreCondition = WhenWritingNull` (free-cell nulls must survive)
- `JsonUnmappedMemberHandling.Disallow` is a .NET 8 BCL feature but is
  **NOT SOURCEABLE** in the engine reference. It must be tested locally. If it
  is unavailable or does not apply to nested DTOs, reject non-empty
  `[JsonExtensionData]` explicitly and add a `"faceUp": true → INVALID_SCHEMA`
  test. Duplicate JSON properties are not rejected by .NET 8 default; do not
  rely on them for correctness.
- `GameSnapshot.SchemaVersion` is authoritative; `ModelSnapshotDto.SchemaVersion`
  must equal it, otherwise `INVALID_SCHEMA`. `AlgorithmVersion` must equal
  `IDealGenerator.AlgorithmVersion` (ordinal), otherwise `INVALID_SNAPSHOT`.
  `DealNumber` must be in 1–1,000,000.
- Remove `ICardDeckModel.Serialize()`; the only JSON path is
  `ModelSnapshotSerializer`. The model exposes `Snapshot()`/`Restore()`; Save
  composes `GameSnapshot` from State and serializes the DTO.

### 5. Error mapping

| Failure | Model/serializer code | State `LoadGame` code |
|---|---|---|
| Malformed JSON | `INVALID_SCHEMA` | `INVALID_SNAPSHOT` |
| Unknown field (`faceUp`) | `INVALID_SCHEMA` | `INVALID_SNAPSHOT` |
| DTO schema mismatch | `INVALID_SCHEMA` | `INVALID_SNAPSHOT` |
| Card token invalid | `INVALID_SCHEMA` | `INVALID_SNAPSHOT` |
| Model invariant violation | `INVALID_INVARIANT` | `MODEL_INVARIANT_FAILED` |
| Algorithm version mismatch | — | `INVALID_SNAPSHOT` (diagnostic `ALGORITHM_VERSION_MISMATCH`) |
| Deal number out of range | — | `INVALID_SNAPSHOT` |
| Success | `None` | `None` |

All failure results carry `Diagnostic` and, where applicable,
`ImmutableArray<SnapshotViolation>` with `{ Code, Pile, Index, Message }`.
No partial state is applied on failure.

### 6. Atomicity and lifetime

- `Deal`, `Move`, `Restore` and deserialization build complete immutable
  candidates before swapping.
- `Restore` and `LoadGame` are atomic; a failure leaves the current model,
  current N, state and Undo history unchanged.
- `UndoHistory` stores `ModelSnapshot` handles; it does not deep-copy cards.
- `Snapshot()` returns the same immutable object until the next successful
  mutation; callers must not mutate the returned arrays.
- `Revision` changes on every successful swap, including Restore; views use
  events, not Revision polling.

### 7. GDD and architecture sync

This ADR explicitly resolves four conflicts:

- `undo-history.md` §6: change "快照不可变，不共享" to "快照不可变；实现可以
  结构化共享未变化的内层数组，观察上等价于完整副本；不得原地修改".
- `card-deck-model.md`: `Deal` is an atomic replace, not a failure on a dealt
  model; update the AC and operation description.
- `deal-generator.md`: Save/Resume stores the full `GameSnapshot`; the deal
  number and algorithm version are retained for validation/display, not for
  regeneration.
- `architecture.md`: remove `ICardDeckModel.Serialize()`, keep `Revision` as
  `int`, keep `Suit` int-backed, and document `ModelSnapshotSerializer` /
  `GameSnapshotSerializer`.

### Architecture Diagram

```text
CardDeckModel
  ├── ModelSnapshot _state   (immutable, shared inner arrays)
  ├── int _revision
  ├── Snapshot() -> ModelSnapshot                  O(1)
  ├── Deal(sequence) -> new snapshot -> validate -> swap
  ├── Move(...) -> new snapshot -> validate -> swap
  └── Restore(snapshot) -> validate -> swap

ModelSnapshotSerializer (Foundation)
  ModelSnapshot <-> ModelSnapshotDto <-> JSON

GameSnapshotSerializer (Feature/State/Serialization)
  GameSnapshot <-> GameSnapshotDto <-> JSON
```

### Key Interfaces

```csharp
namespace FreeCell.Foundation;

public enum Suit { Clubs, Diamonds, Hearts, Spades }
public enum Rank : byte { Ace = 1, Two = 2, /* ... */ King = 13 }
public readonly record struct Card(Suit Suit, Rank Rank);

public readonly record struct SnapshotViolation(
    string Code, PileId? Pile, int? Index, string Message);

public readonly record struct SnapshotValidation(
    bool Success, ImmutableArray<SnapshotViolation> Violations);

public sealed record SnapshotCreateResult(
    bool Success, ModelSnapshot? Snapshot,
    ImmutableArray<SnapshotViolation> Violations, string? Diagnostic);

public readonly record struct SnapshotRestoreResult(
    bool Success, ErrorCode Error,
    ImmutableArray<SnapshotViolation> Violations, string? Diagnostic);

public sealed record SnapshotDeserializeResult(
    bool Success, ModelSnapshot? Snapshot, ErrorCode Error,
    ImmutableArray<SnapshotViolation> Violations, string? Diagnostic);

public interface ICardDeckModel : ICardDeckReadModel
{
    ModelMoveResult Move(PileId source, PileId destination, int count);
    ModelSnapshot Snapshot();
    SnapshotRestoreResult Restore(ModelSnapshot snapshot);
    Result Deal(DealSequence sequence);
    // no Serialize(): ModelSnapshotSerializer is the only JSON path
}

namespace FreeCell.Feature;

public sealed record GameSnapshot(
    int SchemaVersion, string AlgorithmVersion, int DealNumber,
    GameState State, ModelSnapshot Model);

public sealed record GameSnapshotDeserializeResult(
    bool Success, GameSnapshot? Snapshot, ErrorCode Error,
    ImmutableArray<SnapshotViolation> Violations, string? Diagnostic);
```

### Implementation Guidelines

**Must:**

- Use `ModelSnapshot.Create`; never expose a public snapshot constructor.
- Validate every card and foundation suit with `Enum.IsDefined`.
- Check `ImmutableArray.IsDefault` before `Length`; DTOs use `List<T>`.
- Use `ContentEquals`/FNV `ContentHash`; no default `ImmutableArray` equality.
- Use explicit DTOs; never serialize `ModelSnapshot` directly.
- Use the shared `JsonSerializerOptions`; keep free-cell nulls.
- Reject unknown fields and invalid card tokens; no partial load.
- Validate before swapping; failed operations leave the previous state intact.
- Update the four GDDs and `architecture.md` in the same changeset.

**Must never:**

- Add a public `ModelSnapshot` constructor or `with`-style mutation path.
- Serialize `Card`, `ModelSnapshot` or `GameSnapshot` directly.
- Use `HashCode.Combine`/process-randomized hashes for a fingerprint.
- Use `Godot.Json`, `Godot.FileAccess` or any Godot type in the pure layers.
- Trust a saved `State` over re-evaluating `Rules`.
- Emit `GameWon`/`GameStuck` during load.
- Migrate schema 1 silently; reject and report.

## Alternatives Considered

### Alternative 1: Deep-copy snapshots

- **Description**: `Snapshot()` copies all 52 cards into new arrays for every
  undo entry.
- **Pros**: Simple mental model; no sharing questions.
- **Cons**: Higher memory and allocation; the GDD's "not shared" wording would
  be literally true but costly.
- **Estimated Effort**: Low.
- **Rejection Reason**: Structural sharing gives the same observable immutability
  with lower memory; the Undo GDD wording is updated to match.

### Alternative 2: Binary/card-id snapshots for Undo

- **Description**: Undo stores 52-byte card-id arrays; Save uses JSON.
- **Pros**: Lowest Undo memory.
- **Cons**: Two representations; harder debugging and migration; tests need
  conversion.
- **Estimated Effort**: Medium.
- **Rejection Reason**: The sharing approach meets the memory target without a
  second representation.

### Alternative 3: Save only the deal number

- **Description**: Save `N` and regenerate the initial deal on load.
- **Pros**: Tiny save file; no model serialization.
- **Cons**: Loses all moves; cannot resume a game in progress; contradicts the
  State GDD's `GameSnapshot`.
- **Estimated Effort**: Low.
- **Rejection Reason**: The product requirement is resume, not restart.

## Consequences

### Positive

- One snapshot type for Undo, Save/Resume and tests.
- O(1) snapshots and structural sharing keep Undo memory near target.
- Private construction and full validation prevent invalid states.
- Explicit DTOs and schema version make the payload auditable and testable.
- Deterministic equality/hash/canonical JSON satisfy the GDD fingerprint tests.
- Error mapping is explicit across model, serializer and State.

### Negative

- Structural sharing requires careful content equality; default `ImmutableArray`
  equality is a trap.
- `System.Text.Json` options and `JsonUnmappedMemberHandling` need local
  verification.
- DTO boilerplate and mapping code.
- A full-model save is larger than a deal-number-only save.
- Schema 1 has no migration; future changes need a new ADR.

### Neutral

- `Deal` is now an atomic replace, not a one-shot operation.
- `Revision` is int and not serialized.
- `GameState` is serialized as a string, not a numeric enum.
- Save/Resume file location and atomic file replacement remain ADR-0009.

## Risks

| Risk | Mitigation |
|---|---|
| Public snapshot constructor bypasses validation | Private constructor + `Create`; architecture check/code review |
| Enum cast with undefined value | `Enum.IsDefined` on every card and foundation suit |
| `ImmutableArray` default causes exceptions | DTO uses `List<T>`; `IsDefault` checks before use |
| Wrong content equality | `ContentEquals` compares outer and inner sequences; tests with equal-content distinct arrays |
| Non-deterministic hash | FNV-1a over bytes; no `HashCode.Combine`/`GetHashCode` |
| Unknown `faceUp` field accepted | `UnmappedMemberHandling.Disallow` test; `[JsonExtensionData]` fallback; explicit failing test |
| Missing DTO field silently defaults | `required` properties; explicit validation |
| `GameState` numeric encoding breaks on reorder | String enum converter with integer values disallowed |
| Full-model save considered overkill | State GDD already requires `GameSnapshot`; document the decision |
| Export trimming/AOT breaks STJ | Real export save/load test; use source generation if trimming is enabled |
| Undo memory exceeds target | Measure 4096 snapshots; sharing keeps it near estimate; eviction already in Undo GDD |
| Two serialization paths | Remove `ICardDeckModel.Serialize()`; only serializers |
| `Revision` type mismatch | Keep public `int`; update architecture/ADR-0002 to match |

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `card-deck-model.md` | Snapshot/restore, serialization, invariant validation, atomic failure (TR-card-deck-model-003/004/005/006) | Immutable `ModelSnapshot`, private factory, atomic restore, explicit DTOs |
| `undo-history.md` | 4096 immutable snapshots, memory ceiling (TR-undo-history-001/005/006) | Structural sharing, handle storage, FNV fingerprint, memory target |
| `game-state-machine.md` | `GameSnapshot`, load validation, load-to-Won (TR-game-state-machine-005/006) | Full `GameSnapshot`, schema/algorithm validation, re-evaluate, `GameLoaded` only |
| `deal-generator.md` | Algorithm version for save/restore (TR-deal-generator-007) | `AlgorithmVersion` in envelope; mismatch rejected |
| `statistics-records.md` | No persistence in MVP (TR-statistics-records-001) | Snapshot/format is model/State only; Statistics untouched |

## Performance Implications

- **CPU**: O(52) validation on move/deal/restore; input-driven, not per frame.
  Serialization/deserialization only on save/load. No reflection in normal
  gameplay; JSON metadata is cached by the shared options.
- **Memory**: Snapshots share unchanged inner arrays; target 2–3 MB for 4096
  entries; measure and record the actual value.
- **Load Time**: Save/load is Vertical Slice; JSON is small (52 cards).
- **Network**: Not applicable.

## Migration Plan

No code exists. The implementation order is:

1. Add `ModelSnapshot`, `SnapshotViolation`, `SnapshotCreateResult`,
   `SnapshotValidation` and `SnapshotRestoreResult` to Foundation.
2. Refactor `CardDeckModel` to hold a `ModelSnapshot` and `int Revision`; make
   `Deal` an atomic replace.
3. Add `CardCodec`, `ModelSnapshotSerializer` and `GameSnapshotSerializer`.
4. Update `ICardDeckModel`/architecture/ADR-0002 to remove `Serialize()` and use
   `SnapshotRestoreResult`.
5. Sync the four GDDs and registry.
6. Add tests: enum validation, default/empty arrays, content equality/hash,
   `faceUp` rejection, schema/algorithm mismatch, load-to-Won, round-trip,
   export save/load.

**Rollback plan**: If structural sharing proves unworkable, switch to deep-copy
snapshots behind the same `ModelSnapshot` API; the serialized schema and State
contract do not change. This ADR would be superseded.

## Validation Criteria

- `dotnet build FreeCell.csproj` completes with 0 errors and 0 warnings.
- `ModelSnapshot` has no public constructor; every creation path goes through
  `Create` and validation.
- `Enum.IsDefined` rejects `(Suit)7`, `(Rank)0` and `(Rank)99`.
- `ContentEquals` returns true for two snapshots with equal content but
  different inner array instances; `ContentHash` is stable across processes.
- `CanonicalJson()` is byte-identical for the same snapshot across runs.
- `"faceUp": true` and unknown fields are rejected with `INVALID_SCHEMA`.
- Missing DTO fields are rejected (required members).
- `GameState` is encoded/decoded as a string; unknown names are rejected.
- Malformed JSON, bad card tokens, schema mismatch, algorithm mismatch and model
  invariants map to the table in §5.
- `Restore`/`LoadGame` failure leaves model, N, state and Undo history unchanged.
- Load into a won snapshot emits `StateChanged` (if changed) and `GameLoaded`,
  never `GameWon`/`GameStuck`.
- A 4096-snapshot Undo test records actual memory; the target is 2–3 MB.
- A real exported build completes a save/load round-trip.

## Related

- ADR-0001 — layered boundaries and pure C# core
- ADR-0002 — session composition, model ownership, and event contract
- ADR-0005 — C# test execution strategy
- ADR-0009 — Save/Resume format and file storage
- `docs/architecture/architecture.md` §5.1–5.2, §5.5, §4.7
- `design/gdd/card-deck-model.md`, `undo-history.md`, `game-state-machine.md`,
  `deal-generator.md`
- `docs/registry/architecture.yaml`
