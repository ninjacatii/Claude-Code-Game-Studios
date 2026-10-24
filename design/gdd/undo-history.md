# Undo History

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 失败可无损重来

## Summary

Undo History 保存当前牌局每一步合法移动前的模型快照，并在撤销时恢复上一步、通知 State 重判。它不判断移动、不修改规则、不实现 redo；新局/重开/成功读档清空；深度上限 4096。它是“失败可无损重来”的执行层。

> **Quick reference** — Layer: `Feature` · Priority: `MVP` · Key deps: `Rules & Move Validator`, `Game State Machine`, `Card & Deck Model`

## Overview

Undo History 保存当前牌局每一步合法移动前的模型快照，并在玩家撤销时恢复上一步：弹出最近快照、调用 Card & Deck Model.restore、通知 Game State Machine 重新判断胜负。它不判断移动、不修改规则、不实现重做；新局、重开与成功读档会清空历史。它是“失败可无损重来”的执行层，让玩家可以放心尝试。

## Detailed Design

### Core Rules

**1. 范围与所有权**

- 拥有：撤销栈、移动前快照、恢复协调、清空。
- 不拥有：规则、模型、游戏状态、工具栏、渲染。
- 读取：`Card & Deck Model.snapshot()`；写入：仅在撤销时调用 `restore()`。

**2. 数据结构**

- 每条已提交记录：`{ snapshot, move_result }`。
- `snapshot` 是 Card & Deck Model 的不可变完整状态。
- `move_result` 是 Rules 的 `MoveResult`（源/目标/牌/原因），用于日志与提示；恢复只需要快照。
- LIFO 栈；深度 0–4096。

**3. 记录一次移动（顺序固定）**

1. Rules 批准移动。
2. 调用方调用 `record_before_move(model, move_result)`；Undo 捕获 `model.snapshot()` 并放入 **pending** 槽。
3. 调用方执行 `model.move`。
4. 成功 → `commit()`；失败 → `discard_pending()`。
- 只有已提交记录可撤销；未提交的 pending 不进入栈。
- 同一时间只允许一个 pending；在 commit/discard 前再次 record → `RECORD_IN_PROGRESS`。调用方在移动失败或拖拽取消时必须调用 `discard_pending()`；该操作幂等，防止 pending 泄漏。

**4. 撤销**

- `undo(state)`：
  - 深度为 0 → `NOTHING_TO_UNDO`。
  - 存在 pending（`Recording`）→ `RECORD_IN_PROGRESS`。
  - `state ∈ {NoGame, Dealing, Restoring, Won}` → `UNDO_NOT_ALLOWED`；`Playing` / `Stuck` 允许。
  - 弹出最近的已提交记录。
  - `model.restore(entry.snapshot)`（必须原子）。
  - 成功 → `State.on_undo_applied()`；发出 `UndoPerformed(entry.move_result)`；返回成功。
  - 失败 → **丢弃**该损坏记录（弹出后不推回，避免永久重试）；深度减 1；模型不变；发出 `UndoFailed`；返回 `RESTORE_FAILED`。
- `can_undo(state) = depth > 0 ∧ pending = none ∧ state ∈ {Playing, Stuck}`。
- 撤销 0ms、无动画；Table Rendering 同帧重绘。
- 撤销不重新验证；恢复的是曾经有效的状态。

**5. 清空**

- `clear()` 清空栈并丢弃 pending；由 State 在新局、重开、**成功**读档时调用。
- 失败读档不清空（与 State 的原子性一致）。
- 清空在**存在记录或 pending** 时发出 `UndoCleared`；已为空时幂等且不发事件。

**6. 深度与内存**

- 最大深度 4096；提交时若深度已是 4096：先 `evict stack[0]`，再 push；深度保持 4096，发出 `UndoEvicted`（带被丢弃的 `MoveResult`），不进入 4097 的瞬态。
- 快照不可变，不共享、不原地修改。
- 内存 O(深度 × 快照大小)，受 4096 限制。

**7. 事件**

- `UndoRecorded`（捕获 pending 时）、`UndoCommitted`（提交成功时）、`UndoEvicted`（4096 淘汰时）、`UndoPerformed`（撤销成功时）、`UndoCleared`（非空清空时）、`UndoFailed`（恢复失败时）。
- Animation/Audio 可选消费 `UndoPerformed`；本系统不播放动画。
- Table Rendering 在恢复后重绘；Statistics 不受影响。

**8. 线程与并发**

- 单线程；不允许并发 record/undo/clear。
- 拖拽期间撤销命令应先取消拖拽（Toolbar/Drag & Drop 负责）。

### States and Transitions

| 状态 | 进入 | 离开 | 说明 |
|------|------|------|------|
| Empty | 初始/清空 | Recording | 无可撤销记录 |
| Recording | `record_before_move` | Ready / Empty | 持有 pending |
| Ready | `commit` | Ready | pending 入栈 |
| Recording | `discard_pending` | Empty / Ready | 移动失败 |
| Ready | `undo` | Restoring → Ready/Empty | 恢复快照 |
| Any | `clear` | Empty | 新局/重开/成功读档 |

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|------|------|------|------|------------|
| Rules & Move Validator | 上游 | `MoveResult` | 无 | Rules 拥有判定 |
| Card & Deck Model | 上游 | `snapshot` / `restore` | 恢复后的模型 | Model 拥有状态 |
| Game State Machine | 接口 | 当前状态 / `on_undo_applied` / `clear` | 状态重判 | State 拥有生命周期 |
| Drag & Drop Interaction | 下游 | record/commit/discard | 撤销记录 | Undo 拥有历史 |
| Toolbar & Menus | 下游 | 撤销命令 | 结果 | Toolbar 拥有命令 |
| Table Rendering & Layout | 下游 | `UndoPerformed` | 重绘 | Rendering 拥有画面 |
| Animation & Audio | 下游 | `UndoPerformed` | 可选反馈 | 各自拥有表现 |

**接口契约**：

- `record_before_move(model, move_result) -> Result`
- `commit() -> Result`
- `discard_pending() -> Result`（幂等；移动失败/取消时必须调用）
- `undo(state) -> Result`
- `clear()`
- `can_undo(state) -> bool`（depth > 0、无 pending、state ∈ {Playing, Stuck}）
- `depth() -> int`
- `peek() -> MoveResult | null`
- 事件：`UndoRecorded`、`UndoCommitted`、`UndoEvicted`、`UndoPerformed`、`UndoCleared`、`UndoFailed`
- 错误：`RECORD_IN_PROGRESS`、`NOTHING_TO_UNDO`、`UNDO_NOT_ALLOWED`、`RESTORE_FAILED`、`INVALID_SNAPSHOT`
- Undo 不判断移动、不修改规则、不实现 redo。

## Formulas

本节公式只描述已提交栈深度与撤销可用性；pending 不计入深度。`can_undo(state)` 的接口判定还要求 `state ∈ {Playing, Stuck}` 且无 pending。

### The can_undo formula

The can_undo formula is defined as:

`can_undo(depth) = 1 if depth > 0, else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| depth | int | 0–4096 | `depth()`（已提交栈） | 可撤销记录数；不含 pending |
| can_undo | int | {0,1} | 本公式 | 1 = 有可撤销记录 |

**Output Range:** {0,1}。depth=0 → 0；depth≥1 → 1，包括 4096。接口 `can_undo(state)` 还要求无 pending 且状态为 Playing/Stuck。

**Example:** 新局 depth=0 → 0；12 步后 depth=12 → 1；撤销到底 depth=0 → 0。

### The depth_after_commit formula

The depth_after_commit formula is defined as:

`depth_after_commit(depth) = min(depth + 1, 4096)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| depth | int | 0–4096 | 提交前栈深度 | 已提交记录数 |
| 4096 | const int | 4096 | 本 GDD §6 | 硬上限 |
| depth_after_commit | int | 1–4096 | 本公式 | 提交后深度 |

**Output Range:** 1–4096。depth=4095 与 4096 都映射到 4096；达到上限时提交必须先淘汰 `stack[0]` 再 push，深度不进入 4097。淘汰发出 `UndoEvicted`。

**Example:** depth=0 → 1；depth=40 → 41；depth=4095 → 4096；depth=4096 → 4096（淘汰最旧一条）。

### The depth_after_undo formula

The depth_after_undo formula is defined as:

`depth_after_undo(depth) = max(depth − 1, 0)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| depth | int | 0–4096 | 撤销前栈深度 | 已提交记录数 |
| depth_after_undo | int | 0–4095 | 本公式 | 成功撤销后的深度 |

**Output Range:** 0–4095。`undo()` 由 `can_undo` 门控，depth≥1；`max(...,0)` 是防御式下限。成功撤销深度减 1；恢复失败时坏记录被丢弃，深度同样减 1，但模型不变。

**Example:** depth=3 → 2；depth=1 → 0；depth=0 → 0（`NOTHING_TO_UNDO` 时不会走到这里）。

### The max_undo_depth formula

The max_undo_depth formula is defined as:

`max_undo_depth = 4096`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| max_undo_depth | const int | 4096 | 本 GDD §6 | 已提交记录硬上限 |

**Output Range:** 恒为 4096；无输入、无 clamp。内存上限 O(4096 × 快照大小)；按 52 张牌的快照估算约 2–2.5MB。

**Example:** 第 4097 次提交淘汰最旧记录，保留最近 4096 条。

## Edge Cases

- **If `record_before_move` 后移动失败或拖拽取消**: 调用方必须 `discard_pending()`；该操作幂等，防止 pending 泄漏。
- **If 在 pending 未处理时再次 `record_before_move`**: 返回 `RECORD_IN_PROGRESS`，不覆盖 pending。
- **If 在 `Recording` 时调用 `undo`**: 返回 `RECORD_IN_PROGRESS`；`can_undo(state)` 在有 pending 时为 false。
- **If `commit` 时没有 pending**: 返回 `NO_PENDING`，栈不变。
- **If `discard_pending` 时没有 pending**: 幂等成功，栈不变。
- **If depth = 4096 时提交**: 先淘汰 `stack[0]`，再 push；深度保持 4096；发出 `UndoEvicted`。
- **If `model.restore` 失败**: 丢弃该坏记录（不推回），深度减 1，模型不变，发出 `UndoFailed`，返回 `RESTORE_FAILED`。
- **If `model.restore` 不是原子操作**: 必须先恢复到临时模型再交换；不得留下半恢复状态。
- **If 快照在记录时已损坏**: `record_before_move` 返回 `INVALID_SNAPSHOT`，不创建 pending，调用方不得执行移动。
- **If 撤销时 `state` 为 `NoGame` / `Dealing` / `Restoring` / `Won`**: 返回 `UNDO_NOT_ALLOWED`，栈不变。
- **If 撤销时 `state` 为 `Stuck`**: 允许；恢复后由 State 重新判定为 Playing/Won/Stuck。
- **If 新局 / 重开 / 成功读档**: `clear()` 清空栈与 pending，发出 `UndoCleared`。
- **If 读档失败**: 不清空栈（与 State 的原子性一致）。
- **If 拖拽期间收到撤销命令**: 先取消拖拽，再执行撤销（Toolbar/Drag & Drop 负责）。
- **If `move_result` 与快照不一致**: 以快照为准；`move_result` 只用于日志/提示，不参与恢复。
- **If 事件订阅者抛异常**: 恢复已完成；记录错误，不回滚。
- **If 需要 redo**: 不支持；本系统没有 redo 栈。
- **If 存档/读档**: 撤销历史不持久化；成功读档后从空栈开始。
- **If `clear` 在已空历史调用**: 幂等成功，不发 `UndoCleared`。
- **If 拖拽进行中调用 undo 且未先取消**: Toolbar 必须先调用 Drag & Drop 的取消；若存在 pending，undo 返回 `RECORD_IN_PROGRESS`；本系统不检测拖拽状态。
- **If `record_before_move` 返回 `INVALID_SNAPSHOT`**: 调用方不得执行移动；集成测试覆盖。
- **If 并发 record/undo/clear**: 单线程；调用方必须串行化。
- **If 快照被下游修改**: 快照不可变；记录时做防御性复制或只读封装。
- **If `UndoPerformed` 触发重绘**: Table Rendering 在同一帧重绘；无动画。
- **If 4096 条记录仍不足以覆盖整局**: 单局 FreeCell 实际步数远低于 4096；淘汰最旧记录是可接受的降级。

## Dependencies

**上游依赖**：

- **Rules & Move Validator**（Hard）：`MoveResult`；只读。
- **Card & Deck Model**（Hard）：`snapshot()` / `restore()`；`restore` 必须原子。
- **Game State Machine**（接口）：当前状态、`on_undo_applied()`、新局/重开/成功读档时的 `clear`。

**下游依赖（依赖本系统的系统）**：

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Drag & Drop Interaction | Hard | `record_before_move` / `commit` / `discard_pending` | 移动成功后提交 |
| Toolbar & Menus | Hard | `undo(state)` / `can_undo(state)` | 撤销按钮 |
| Table Rendering & Layout | Soft | `UndoPerformed` | 同帧重绘 |
| Animation & Feedback | Soft | `UndoPerformed` | 可选反馈；0ms 无动画 |
| Audio & SFX | Soft | `UndoPerformed` | 可选轻音效 |
| Statistics & Records | 无 | — | 不直接交互 |

**接口契约**：

- `record_before_move(model, move_result) -> Result`
- `commit() -> Result`
- `discard_pending() -> Result`（幂等）
- `undo(state) -> Result`
- `clear()`
- `can_undo(state) -> bool`
- `depth() -> int`
- `peek() -> MoveResult | null`
- 事件：`UndoRecorded`、`UndoCommitted`、`UndoEvicted`、`UndoPerformed`、`UndoCleared`、`UndoFailed`
- 错误：`RECORD_IN_PROGRESS`、`NO_PENDING`、`NOTHING_TO_UNDO`、`UNDO_NOT_ALLOWED`、`RESTORE_FAILED`、`INVALID_SNAPSHOT`
- Undo 不判断移动、不修改规则、不实现 redo、不持久化历史。

**Provisional assumptions（下游未设计，先假定）**：

- Drag & Drop 在 `model.move` 前 record、成功后 commit、失败/取消后 discard。
- Toolbar 用 `can_undo(state)` 决定按钮可用性；Won/NoGame 时禁用。
- State 在 `on_undo_applied` 后重新判定胜负；新局/重开/成功读档调用 `clear`。
- Card & Deck Model 的 `restore` 原子且接受自身产出的快照。
- Animation/Audio 对 `UndoPerformed` 做去重，不重复播放。

**Registry references**：`max_undo_depth=4096` 是本系统局部常量，不与其它系统共享；暂不登记。

## Visual/Audio Requirements

- 无直接视觉/音频；撤销是 0ms 瞬移，Table Rendering 在同一帧重绘。
- 无撤销动画、无闪光、无确认弹窗、无“已撤销”文字。
- 可选：Audio & SFX 在 `UndoPerformed` 上播放极轻的“回位”音效，默认关闭；本系统不拥有音频资源。
- 可选：Animation & Feedback 不插值撤销；如未来需要反向动画，另行设计，不在 MVP。

## Game Feel

- **即时**：点击撤销后同一帧恢复；无动画、无延迟、无确认。
- **一次一步**：每次撤销弹出一条记录；连续点击可连续回退，无冷却。
- **无惩罚**：撤销不扣分、不弹窗、不中断操作；`Stuck` 状态仍可撤销。
- **可预期**：按钮/快捷键仅在 `can_undo(state)` 为 true 时可用；不可用时明确禁用，不产生“点了没反应”。
- **无记忆负担**：不显示撤销次数上限；4096 的上限在单局中实际不可达。
- **静默**：默认无音效、无动画；反馈由牌面位置本身提供。

## UI Requirements

- Toolbar & Menus 提供一个“撤销”按钮/菜单项；可用性由 `can_undo(state)` 决定。
- 无 redo 按钮、无撤销次数显示、无确认对话框。
- 在 `Won`、`NoGame`、栈为空或有 pending 时禁用；`Stuck` 时可用。
- 快捷键（如 Ctrl+Z）由 Toolbar 映射到 `undo(state)`；本系统不直接处理输入。
- 撤销后 Table Rendering 同帧重绘；不需要本系统刷新 UI。

## Cross-References

| 系统 | 引用内容 | 状态 |
|------|----------|------|
| Rules & Move Validator | `MoveResult` 载荷 | GDD 已完成 |
| Card & Deck Model | `snapshot` / `restore`（原子） | GDD 已完成 |
| Game State Machine | 状态门控、`on_undo_applied`、`clear` | GDD 已完成 |
| Drag & Drop Interaction | record/commit/discard 时序 | GDD 已完成 |
| Toolbar & Menus | 撤销按钮/快捷键 | GDD 已完成（design/gdd/toolbar-menus.md） |
| Table Rendering & Layout | 恢复后重绘 | GDD 已完成 |
| Animation & Feedback | 0ms 无动画；未来可选 | GDD 未编写 |
| Audio & SFX | 可选轻音效，默认关闭 | GDD 未编写 |

## Acceptance Criteria

**测试约定**：`depth` 指已提交栈深度，不含 pending；`state` 为 Game State Machine 当前状态；“模型不变”指状态指纹逐字节相同。

### 记录与提交

- **GIVEN** 一步已获 Rules 批准的移动，**WHEN** 调用 `record_before_move → model.move（成功）→ commit`，**THEN** 快照只在 commit 时入栈、depth 恰好 +1、记录可撤销。
- **GIVEN** `record_before_move` 成功但移动失败或拖拽取消，**WHEN** 调用 `discard_pending()` 两次，**THEN** pending 清空、栈/depth 不变、第二次幂等成功。
- **GIVEN** 存在未提交的 pending，**WHEN** 再次 `record_before_move`，**THEN** 返回 `RECORD_IN_PROGRESS` 且不覆盖 pending；**GIVEN** 无 pending，**WHEN** `commit`，**THEN** 返回 `NO_PENDING`、栈不变。
- **GIVEN** `record_before_move` 捕获 pending，**WHEN** 事件检查，**THEN** `UndoRecorded` 在捕获时发出一次；`UndoCommitted` 在 commit 成功时发出一次。

### 撤销

- **GIVEN** depth ≥ 1、无 pending、state 为 Playing 或 Stuck，**WHEN** `undo(state)`，**THEN** 原子恢复顶部快照、`State.on_undo_applied()` 执行、`UndoPerformed` 恰好一次、depth 减 1、返回成功。
- **GIVEN** state 为 Won / NoGame / Dealing / Restoring，**WHEN** `undo(state)`，**THEN** 返回 `UNDO_NOT_ALLOWED`，栈/depth/模型不变，不发 `UndoPerformed` 或 `UndoFailed`。
- **GIVEN** 存在 pending，**WHEN** `undo(state)` 或 `can_undo(state)`，**THEN** undo 返回 `RECORD_IN_PROGRESS`、can_undo 为 false、栈不变。
- **GIVEN** 顶部记录的 `model.restore` 失败，**WHEN** `undo(state)`，**THEN** 丢弃该坏记录、depth 减 1、模型不变、发出 `UndoFailed`、返回 `RESTORE_FAILED`。
- **GIVEN** `Stuck` 状态，**WHEN** `undo(state)`，**THEN** 允许撤销；恢复后由 State 重判为 Playing/Won/Stuck。

### 清空、上限与不变量

- **GIVEN** 非空栈或有 pending，**WHEN** 新局/重开/成功读档调用 `clear()`，**THEN** depth=0、pending 丢弃、`UndoCleared` 恰好一次；**GIVEN** 读档失败，**THEN** 不清空，栈/depth 保留。
- **GIVEN** 已空历史，**WHEN** `clear()`，**THEN** 幂等成功且不发 `UndoCleared`。
- **GIVEN** depth=4096，**WHEN** commit 成功，**THEN** 先淘汰 `stack[0]` 再 push、`UndoEvicted` 携带被淘汰的 `MoveResult`、depth 保持 4096、无 4097 瞬态、保留最近 4096 条。
- **GIVEN** 已提交快照，**WHEN** 之后修改模型、存档/读档或请求 redo，**THEN** 快照仍等于记录时状态（防御性复制/只读）、存档不含撤销历史（读档后 depth=0）、不存在 redo 操作。
- **GIVEN** 任意成功撤销，**WHEN** 执行，**THEN** 0ms、无动画、`UndoPerformed` 一次；相同输入序列产生相同 depth、栈顺序与事件序列（确定性）。

### 集成

- **GIVEN** 拖拽进行中，**WHEN** 工具栏撤销，**THEN** 先取消拖拽再执行撤销；**GIVEN** depth=0、存在 pending 或状态不允许，**THEN** 工具栏按钮禁用（`can_undo(state)=false`），成功 commit 后启用。
- **GIVEN** `record_before_move` 返回 `INVALID_SNAPSHOT`，**WHEN** 调用方处理，**THEN** 不执行移动、不创建 pending；集成测试覆盖。

## Open Questions

| 问题 | Owner | 目标解决时间 |
|------|-------|--------------|
| 4096 深度上限是否符合混合参考体验（已声明为工程约束，不声称原版） | systems-designer + user | 首个可玩版本前 |
| `Won` 状态拒绝撤销是否符合混合参考（Windows 7/XP 功能基线） | ux-designer + user | 首个可玩版本前 |
| 撤销音效默认关闭 | Audio & SFX + Settings | 对应 GDD |
| 快照内存目标 | architecture | 架构阶段 |
| 是否需要 redo | user | 不做；如未来需要另行设计 |
| Ctrl+Z 快捷键映射 | Toolbar & Menus | Toolbar GDD |
