# Game State Machine

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 原版规则，零改动

## Summary

Game State Machine 拥有空当接龙的当前局号与生命周期（NoGame / Playing / Won / Stuck），协调 Deal Generator、Card & Deck Model、Rules、Undo、Statistics 与 Save/Resume。它保证新局、重开、下一局/上一局、撤销后重判与读档的状态一致，并在进入胜利/卡死时只发一次事件；不修改模型、不判断移动、不写文件。

> **Quick reference** — Layer: `Feature` · Priority: `MVP` · Key deps: `Deal Generator`, `Rules & Move Validator`

## Overview

Game State Machine 拥有空当接龙的当前局号与生命周期：没有牌局、发牌中、进行中、已胜利、卡死。它把“新局 / 重开 / 下一局 / 上一局 / 撤销后重判 / 读档”翻译成对 Deal Generator、Card & Deck Model、Rules、Undo、Statistics 与 Save/Resume 的协调调用，并在每次移动后重新判断胜利或卡死。它不持有牌堆、不判断移动合法性、不实现撤销或统计记录。

## Detailed Design

### Core Rules

**1. 所有权**

- 拥有：当前局号 `N`、生命周期状态、命令协调、胜利/卡死判定时机。
- 不拥有：牌堆状态（Card & Deck Model）、移动合法性（Rules）、撤销历史（Undo）、统计记录（Statistics）、存档格式（Save/Resume）、输入（Drag & Drop）、UI。
- 是“当前是哪一局、处于哪个阶段”的唯一来源。

**2. 状态**

- `NoGame`：应用启动，尚未载入牌局。
- `Dealing`：正在生成/发牌；瞬态。
- `Playing`：进行中，未胜利，且至少有一条合法移动。
- `Won`：52 张全部在回收单元。
- `Stuck`：未胜利且没有任何合法移动。
- `Restoring`：正在恢复存档；瞬态。
- `Error` 是命令结果，不是持久状态；失败时保持原状态。

**3. 命令**

- `new_game(N)`：校验 N → 调用 Deal Generator → 调用 Card & Deck Model 发牌 → 清空 Undo → 状态 `Playing` → 当前 N = N → 发出 `GameStarted`。
- `restart_deal()`：用当前 N 重新发牌；清空 Undo；`Playing`；发出 `GameRestarted`。
- `next_deal()` / `previous_deal()`：按边界钳制；目标与当前 N 不同则调用 `new_game`，相同则返回 `AT_LIMIT`（不重新发牌）。
- `select_deal(N)`：若 N = 当前 N → `restart_deal()`；否则 `new_game(N)`。
- `on_move_applied(result)`：一次合法移动执行后，重新判定状态。
- `on_undo_applied()`：一次撤销恢复模型后，重新判定状态。
- `load_game(snapshot)`：恢复模型与 N，重新判定，发出 `GameLoaded`。
- `snapshot()`：返回 N + 模型 + 状态；序列化由 Save/Resume 负责。`NoGame` / `Dealing` / `Restoring` 时返回 `GAME_NOT_READY`。
- `can_accept_moves()`：仅 `Playing` 为 true。
- `get_state()` / `get_deal_number()`。

**4. 状态转移**

| 当前 | 命令 | 结果 | 守卫 |
|------|------|------|------|
| `NoGame` | `new_game(N)` | `Dealing` → `Playing` | N 合法 |
| `Playing` | `new_game` / `restart` / `next` / `prev` | `Dealing` → `Playing` | N 合法/存在 |
| `Playing` | `on_move_applied` | `Playing` / `Won` / `Stuck` | 移动已合法执行 |
| `Playing` | `on_undo_applied` | `Playing` / `Won` / `Stuck` | 撤销已执行 |
| `Won` | `new_game` / `restart` / `next` / `prev` | `Dealing` → `Playing` | 允许 |
| `Won` | `on_move_applied` | `Won` | 拒绝 `MOVES_NOT_ACCEPTED` |
| `Won` | `on_undo_applied` | `Won` | 拒绝 `GAME_ALREADY_WON` |
| `Stuck` | `new_game` / `restart` / `next` / `prev` | `Dealing` → `Playing` | 允许 |
| `Stuck` | `on_move_applied` | `Stuck` | 拒绝 `MOVES_NOT_ACCEPTED` |
| `Stuck` | `on_undo_applied` | `Playing` / `Won` / `Stuck` | 允许 |
| `NoGame` / `Playing` / `Won` / `Stuck` | `load_game(snapshot)` | `Playing` / `Won` / `Stuck` | 快照合法；`Dealing` / `Restoring` 返回 `COMMAND_IN_PROGRESS` |
| `NoGame` | `restart` / `next` / `prev` / `undo` | `NoGame` | `NO_CURRENT_GAME` |

**5. 胜利与卡死判定**

- 每次移动、撤销或读档后：先 `Rules.is_won(model)`；为 true → `Won`。
- 否则 `Rules.is_stuck(model)`；为 true → `Stuck`。
- 否则 → `Playing`。
- 胜利优先于卡死：已胜利即使 `legal_move_count = 0` 也不是 `Stuck`。
- `Stuck` 是提示性状态：允许撤销/重开/新局；不阻塞 Undo。
- 空列数量不用于胜负判断；只使用 `is_won` / `is_stuck`。

**6. 局号导航**

- `next_deal` 与 `previous_deal` 用 `clamp(N ± 1, deal_number_min, deal_number_max)`；**`AT_LIMIT` 仅在 clamp 结果等于当前 N 时返回**（next 在 N=max，prev 在 N=min），此时不重新发牌，状态与 Undo 不变。N=999,999 的 next 目标为 1,000,000、N=2 的 prev 目标为 1，仍会正常重新发牌。
- 无当前牌局时调用导航 → `NO_CURRENT_GAME`。
- State 不生成随机局号；首次启动的 N 由 UI 决定（默认 1 或上次存档）。
- `new_game(N)` 对同一 N 也有效：它是同一局号的重新发牌。直接调用 `new_game(N)` 仍视为新局；UI 的「选局」在 N = 当前 N 时路由为 `restart_deal()`，避免统计口径不一致。

**7. 事件**

- `GameStarted { N }`、`GameRestarted { N }`、`GameWon { N }`、`GameStuck { N }`、`GameLoaded { N }`、`StateChanged { from, to }`。
- 事件在状态更新之后同步发出；不使用异步队列。
- `GameWon` / `GameStuck` 每次进入终态只发一次；重复的 `on_move_applied` 不重复发。
- Statistics 消费 `GameStarted` 与 `GameWon`；Animation/Audio 消费终态事件；Toolbar 读 `StateChanged`。

**8. 协调**

- 新局、重开、成功读档：通知 Undo 清空历史；失败读档不清空。
- 撤销后：State 重新判定，不修改模型。
- 保存：State 提供快照；Save/Resume 写文件。
- 读档：先校验算法版本与模型不变量；失败则报错并保持原状态。

**9. 错误**

- 错误码：`INVALID_DEAL_NUMBER`、`DEAL_NUMBER_OUT_OF_RANGE`、`NO_CURRENT_GAME`、`MOVES_NOT_ACCEPTED`、`GAME_ALREADY_WON`、`INVALID_SNAPSHOT`、`MODEL_INVARIANT_FAILED`、`COMMAND_IN_PROGRESS`、`GAME_NOT_READY`。
- `AT_LIMIT` 是导航结果（不是错误）：边界处 no-op，状态与 Undo 不变。
- 错误不改变状态；`Dealing` / `Restoring` 失败时回到命令前的状态。
- `new_game` 失败不得清空 Undo。

### States and Transitions

```text
NoGame --new_game(N)--> Dealing --> Playing
Playing --move--> Playing | Won | Stuck
Playing --undo--> Playing | Won | Stuck
Won --new_game/restart/next/prev--> Dealing --> Playing
Stuck --undo--> Playing | Won | Stuck
Stuck --new_game/restart/next/prev--> Dealing --> Playing
NoGame/Playing/Won/Stuck --load_game(valid)--> Playing | Won | Stuck
```

`Dealing` 与 `Restoring` 是瞬态；命令是同步的，不重入。

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|------|------|------|------|------------|
| Deal Generator | 上游 | `generate(N)` | 52 张序列/错误 | Deal 拥有算法；State 无状态 |
| Card & Deck Model | 上游 | `deal(sequence)` / `snapshot` / `restore` | 模型状态 | Model 拥有牌堆 |
| Rules & Move Validator | 上游 | `is_won` / `is_stuck` / `legal_moves` | 判定结果 | Rules 只读 |
| Drag & Drop Interaction | 下游 | 移动已执行通知 | `can_accept_moves` | Interaction 拥有输入 |
| Undo History | 下游 | 撤销已执行通知 | `clear` / 移动记录 | Undo 拥有历史 |
| Statistics & Records | 下游 | `GameStarted` / `GameWon` | 记录 | Stats 拥有记录 |
| Save/Resume Game | 下游 | `snapshot` / `load_game` | 存档数据 | Save 拥有格式 |
| Toolbar & Menus | 下游 | 命令 | 状态/错误 | Toolbar 拥有命令 |
| Table Rendering / Animation / Audio | 下游 | 状态/事件 | 反馈 | 各自拥有表现 |
| Settings / Accessibility | 无 | — | — | 不直接交互 |

**接口契约**：

- `new_game(N) -> Result`
- `restart_deal() -> Result`
- `next_deal() -> Result`
- `previous_deal() -> Result`
- `on_move_applied(MoveResult) -> Result`
- `on_undo_applied() -> Result`
- `load_game(snapshot) -> Result`
- `snapshot() -> GameSnapshot | error`（`NoGame` / `Dealing` / `Restoring` 返回 `GAME_NOT_READY`）
- `get_state() -> State`
- `get_deal_number() -> int`
- `can_accept_moves() -> bool`
- State 不修改模型、不判断移动、不写文件；只协调。

## Formulas

本节公式只做边界钳制与状态映射；不产生随机局号，不修改模型。

### The next_deal formula

The next_deal formula is defined as:

`next_deal(N) = clamp(N + 1, deal_number_min, deal_number_max)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| N | int | 1–1,000,000 | `get_deal_number()` | 当前局号 |
| deal_number_min | const int | 1 | registry（`deal_number_min`） | 下界 |
| deal_number_max | const int | 1,000,000 | registry（`deal_number_max`） | 上界 |
| next_deal | int | 1–1,000,000 | 本公式 | 目标局号 |

**Output Range:** 1–1,000,000。N=1 时为 2；N=999,999 时为 1,000,000（正常重新发牌）；N=1,000,000 时仍为 1,000,000，但目标等于当前 N，返回 `AT_LIMIT`，不重新发牌。不回绕。

**Example:** N=42 → 43；N=1,000,000 → 1,000,000（`AT_LIMIT`，状态与 Undo 不变）。

### The previous_deal formula

The previous_deal formula is defined as:

`previous_deal(N) = clamp(N − 1, deal_number_min, deal_number_max)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| N | int | 1–1,000,000 | `get_deal_number()` | 当前局号 |
| deal_number_min | const int | 1 | registry（`deal_number_min`） | 下界 |
| deal_number_max | const int | 1,000,000 | registry（`deal_number_max`） | 上界 |
| previous_deal | int | 1–1,000,000 | 本公式 | 目标局号 |

**Output Range:** 1–1,000,000。N=2 时为 1（正常重新发牌）；N=1 时仍为 1，但目标等于当前 N，返回 `AT_LIMIT`，不重新发牌；N=1,000,000 时为 999,999。不回绕。

**Example:** N=42 → 41；N=1 → 1（`AT_LIMIT`，状态与 Undo 不变）。

### The state_after_evaluation formula

The state_after_evaluation formula is defined as:

`state_after_evaluation(won, stuck) = Won if won = 1, else (Stuck if stuck = 1, else Playing)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| won | int | {0,1} | `Rules.is_won(model)` | 52 张全部在回收单元 |
| stuck | int | {0,1} | `Rules.is_stuck(model)` | 无合法移动；可能与 won 同时为 1 |
| state_after_evaluation | enum | {Won, Stuck, Playing} | 本公式 | 移动/撤销/读档后的状态 |

**Output Range:** {Won, Stuck, Playing}。`(won=1, stuck=1)` 可达（全部归位后无合法移动），此时返回 `Won`——胜利优先。`(0,0)` → `Playing`。永不返回 `NoGame` / `Dealing` / `Restoring`。

**Example:** `(1,1)` → Won；`(0,1)` → Stuck；`(0,0)` → Playing；`(1,0)` → Won。

### The can_accept_moves formula

The can_accept_moves formula is defined as:

`can_accept_moves(state) = 1 if state = Playing, else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| state | enum | {NoGame, Dealing, Playing, Won, Stuck, Restoring} | `get_state()` | 生命周期状态；`Error` 不是状态 |
| can_accept_moves | int | {0,1} | 本公式 | 是否允许应用移动 |

**Output Range:** {0,1}；只有 `Playing` 为 1。`Stuck` → 0（允许撤销/重开，不允许移动）；`Won` → 0（`GAME_ALREADY_WON`）；`NoGame` → 0（`NO_CURRENT_GAME`）；`Dealing` / `Restoring` → 0（同步命令，不重入）。

**Example:** `Playing` → 1；`Stuck` → 0；`Won` → 0；`NoGame` → 0。

## Edge Cases

- **If `new_game(N)` 的 N 越界或非法**: 返回 `DEAL_NUMBER_OUT_OF_RANGE` / `INVALID_DEAL_NUMBER`，状态与 Undo 都不变。
- **If 在 `Dealing` / `Restoring` 期间重入命令**: 返回 `COMMAND_IN_PROGRESS`，状态不变；命令同步、不可重入。
- **If `load_game` 在 `Dealing` / `Restoring` 调用**: 返回 `COMMAND_IN_PROGRESS`，状态不变。
- **If `restart` / `next` / `prev` / `undo` 在 `NoGame` 调用**: 返回 `NO_CURRENT_GAME`，状态不变。
- **If `next_deal` 在 N = 1,000,000**: 返回 `AT_LIMIT`，不重新发牌，状态与 Undo 不变。
- **If `previous_deal` 在 N = 1**: 返回 `AT_LIMIT`，不重新发牌，状态与 Undo 不变。
- **If `on_move_applied` 在 `Won` 或 `Stuck`**: 返回 `MOVES_NOT_ACCEPTED`，状态不变。
- **If `on_move_applied` 在 `NoGame`**: 返回 `NO_CURRENT_GAME`，状态不变。
- **If `on_undo_applied` 在 `Won`**: 返回 `GAME_ALREADY_WON`，状态不变；胜利后不允许撤销。
- **If `on_undo_applied` 在 `Stuck`**: 允许；重新判定后进入 `Playing` / `Won` / `Stuck`。
- **If `won = 1` 且 `stuck = 1`**: 返回 `Won`；胜利优先。
- **If 读档快照无效或算法版本不匹配**: 返回 `INVALID_SNAPSHOT`，状态不变。
- **If 读档模型不变量校验失败**: 返回 `MODEL_INVARIANT_FAILED`，状态不变。
- **If 读档失败（`INVALID_SNAPSHOT` / `MODEL_INVARIANT_FAILED`）**: 状态、当前 N 与 Undo 都保持不变；只有成功读档才清空 Undo。
- **If 读档的是已胜利快照**: 进入 `Won`，只发 `GameLoaded`，不发 `GameWon`（避免统计重复计数）。
- **If `snapshot()` 在 `NoGame` / `Dealing` / `Restoring` 调用**: 返回 `GAME_NOT_READY`，不产生快照。
- **If Deal Generator 在 `new_game` 中失败**: 状态不变，Undo 不清空。
- **If Card & Deck Model.deal 失败**: 返回 `MODEL_INVARIANT_FAILED`，状态不变，Undo 不清空。
- **If 事件订阅者抛异常**: 状态已在发事件前更新；记录错误，命令仍返回成功；订阅者不得依赖异常回滚。
- **If 重复进入终态**: `GameWon` / `GameStuck` 每次进入只发一次；重复 `on_move_applied` 不重复发。
- **If 从 `Won` / `Stuck` 开新局**: 发 `GameStarted`，不重复发旧的终态事件。
- **If 空列数量为 8**: 不用于胜负判断；只使用 `is_won`。
- **If 同一 N 再次 `new_game`**: 合法；视为同一局号的重新发牌，清空 Undo。
- **If 首次启动没有 N**: 状态为 `NoGame`；由 UI 调用 `new_game(N)`，State 不生成随机 N。
- **If `on_move_applied` 收到未经 Rules 批准的移动**: State 不重新验证，假定调用方已执行合法移动；调用方负责只在 `legal = true` 时通知。
- **If 多个命令并发**: 必须串行化；State 不保证线程安全。
- **If 存档快照包含当前 N 以外的局号**: 以快照内的 N 为准；State 不交叉验证 UI 的显示值。

## Dependencies

**上游依赖（Hard）**：

- **Deal Generator**：`generate(N) -> DealSequence | error`；局号范围与算法版本。
- **Card & Deck Model**：`deal(sequence)`、`snapshot()`、`restore(snapshot)`；模型不变量。
- **Rules & Move Validator**：`is_won(model)`、`is_stuck(model)`、`legal_moves(model)`；只读。

**下游依赖（依赖本系统的系统）**：

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Drag & Drop Interaction | Hard | `can_accept_moves` / `on_move_applied` | 只在 `Playing` 接受移动 |
| Toolbar & Menus | Hard | `new_game` / `restart` / `next` / `prev` / `select_deal` | 命令与错误展示 |
| Undo History | Hard | `on_undo_applied` / `clear` | 新局/重开/读档清空历史 |
| Statistics & Records | Soft | `GameStarted` / `GameWon` | 只消费事件，不写模型 |
| Save/Resume Game | Hard | `snapshot` / `load_game` | Save 拥有格式，State 拥有 N/状态 |
| Table Rendering / Animation / Audio | Soft | 状态/事件 | 终态反馈 |
| Settings / Accessibility | None | — | 不直接交互 |

**接口契约**：

- `new_game(N) -> Result`
- `restart_deal() -> Result`
- `next_deal() -> Result`
- `previous_deal() -> Result`
- `select_deal(N) -> Result`
- `on_move_applied(MoveResult) -> Result`
- `on_undo_applied() -> Result`
- `load_game(snapshot) -> Result`
- `snapshot() -> GameSnapshot | error`
- `get_state() -> State`
- `get_deal_number() -> int`
- `can_accept_moves() -> bool`
- State 不修改模型、不判断移动、不写文件；只协调并发出事件。

**Provisional assumptions（下游未设计，先假定）**：

- Drag & Drop 只在 Rules 返回 `legal = true` 并执行模型移动后调用 `on_move_applied`。
- Undo History 在收到 `clear` 后清空；撤销执行后调用 `on_undo_applied`。
- Statistics 提供 `on_game_started(N)` 与 `on_game_won(N)`。`GameWon` 对同一活动局实例幂等（重复投递不重复计数）；`GameStarted` 对每个成功的 `new_game` 命令恰好发出一次，Statistics 不按 N 去重——同一 N 的 `new_game(N)` 是合法新局，必须计为一局。
- Save/Resume 保存 `GameSnapshot`（N + 模型 + 状态 + 算法版本），恢复前校验版本。
- Toolbar 负责首次启动的 N（默认 1 或上次存档），State 不生成随机局号。

**Registry references**：`deal_number_min`、`deal_number_max`、`game_completion`、`is_stuck`、`legal_move_count` 的 `referenced_by` 应追加 `design/gdd/game-state-machine.md`（Phase 5b 更新）。

## Visual/Audio Requirements

无直接视觉/音频；`GameWon` / `GameStuck` 事件由 Animation & Feedback 与 Audio & SFX 消费。

## Game Feel

状态转移是瞬时的；它不引入动画延迟，胜负/卡死反馈的节奏由 Animation & Feedback 决定。

## UI Requirements

Toolbar & Menus 调用命令并显示错误；`Won` / `Stuck` 由 UI 显示覆盖层；State 不直接绘制。

## Cross-References

| 系统 | 引用内容 | 状态 |
|------|----------|------|
| Deal Generator | `generate(N)`、局号范围、算法版本 | GDD 已完成 |
| Card & Deck Model | `deal` / `snapshot` / `restore` | GDD 已完成 |
| Rules & Move Validator | `is_won` / `is_stuck` / `legal_moves` | GDD 已完成 |
| Undo History | `clear` / `on_undo_applied` | GDD 已完成（design/gdd/undo-history.md） |
| Statistics & Records | `GameStarted` / `GameWon` | GDD 已完成（design/gdd/statistics-records.md） |
| Save/Resume Game | `snapshot` / `load_game` | GDD 未编写 |
| Toolbar & Menus | 命令与状态 | GDD 已完成（design/gdd/toolbar-menus.md） |

## Acceptance Criteria

**测试约定**：状态 `NoGame` / `Dealing` / `Playing` / `Won` / `Stuck` / `Restoring`；当前局号 N ∈ 1–1,000,000；事件 `GameStarted` / `GameRestarted` / `GameWon` / `GameStuck` / `GameLoaded` / `StateChanged`；“状态不变”指 `get_state()` 与 `get_deal_number()` 均不变，Undo 历史按条目保留。

### 状态转移与命令

- **GIVEN** `NoGame`，**WHEN** `new_game(N)` 且 N 合法，**THEN** 经 `Dealing` 进入 `Playing`、`get_deal_number() = N`、Undo 清空、发出 `GameStarted{N}`。
- **GIVEN** 当前 N，**WHEN** `select_deal(N)`（同 N），**THEN** 路由为 `restart_deal()`：同 N 重发、Undo 清空、发出 `GameRestarted`，不发 `GameStarted`；**GIVEN** `select_deal(M)` 且 `M ≠ N`，**THEN** 发出 `GameStarted`。
- **GIVEN** `NoGame`，**WHEN** `new_game(N)` 且 N 越界或非法，**THEN** 返回 `DEAL_NUMBER_OUT_OF_RANGE` / `INVALID_DEAL_NUMBER`，状态与 Undo 均不变。
- **GIVEN** `NoGame`，**WHEN** 调用 `restart_deal` / `next_deal` / `previous_deal` / `on_undo_applied`，**THEN** 返回 `NO_CURRENT_GAME`，状态不变。
- **GIVEN** `Playing` 且 N=1,000,000，**WHEN** `next_deal()`，**THEN** 返回 `AT_LIMIT`，不重新发牌，状态与 Undo 不变；**GIVEN** `Playing` 且 N=1，**WHEN** `previous_deal()`，**THEN** 同样返回 `AT_LIMIT`。
- **GIVEN** 当前 N，**WHEN** 计算 `next_deal`，**THEN** `clamp(N+1, 1, 1,000,000)`：N=1→2；N=999,999→1,000,000（正常重新发牌）；N=1,000,000→1,000,000 且返回 `AT_LIMIT`；不回绕。
- **GIVEN** 当前 N，**WHEN** 计算 `previous_deal`，**THEN** `clamp(N−1, 1, 1,000,000)`：N=42→41；N=2→1（正常重新发牌）；N=1→1 且返回 `AT_LIMIT`；不回绕。
- **GIVEN** Rules 的 `won` / `stuck` 标志，**WHEN** 计算 `state_after_evaluation`，**THEN** `(1,1)`→Won、`(0,1)`→Stuck、`(0,0)`→Playing、`(1,0)`→Won；永不返回 `NoGame` / `Dealing` / `Restoring`。
- **GIVEN** 任意状态，**WHEN** 计算 `can_accept_moves`，**THEN** 仅 `Playing` 返回 1；`NoGame`、`Dealing`、`Won`、`Stuck`、`Restoring` 返回 0。

### 移动、撤销与终态

- **GIVEN** `Playing` 且一次移动已由 Rules 执行，**WHEN** `on_move_applied(result)`，**THEN** 按 `Rules.is_won` / `is_stuck` 进入 `Playing`、`Won` 或 `Stuck`；`won=1` 且 `stuck=1` 时进入 `Won`（胜利优先）。
- **GIVEN** `Won` 或 `Stuck`，**WHEN** `on_move_applied`，**THEN** 返回 `MOVES_NOT_ACCEPTED`，状态不变。**GIVEN** `Stuck`，**WHEN** `on_undo_applied`，**THEN** 允许撤销并重判为 `Playing` / `Won` / `Stuck`。**GIVEN** `Won`，**WHEN** `on_undo_applied`，**THEN** 返回 `GAME_ALREADY_WON`，状态不变。
- **GIVEN** 进入 `Won` 或 `Stuck`，**THEN** `GameWon` / `GameStuck` 每次进入只发一次；重复 `on_move_applied` 不重复发；Statistics 只记录一次 `GameWon`。

### 读档与保存

- **GIVEN** 合法快照，**WHEN** `load_game(snapshot)`，**THEN** 进入 `Playing` / `Won` / `Stuck`，发出 `GameLoaded{N}`；**GIVEN** 无效快照或算法版本不匹配，**THEN** `INVALID_SNAPSHOT`，状态不变；**GIVEN** 模型不变量失败，**THEN** `MODEL_INVARIANT_FAILED`，状态不变；**GIVEN** 已胜利快照，**THEN** 进入 `Won` 且只发 `GameLoaded`，不发 `GameWon`。
- **GIVEN** `NoGame` / `Dealing` / `Restoring`，**WHEN** `snapshot()`，**THEN** 返回 `GAME_NOT_READY`，不产生快照。
- **GIVEN** 读档失败，**THEN** 状态、当前 N 与 Undo 均不变；只有成功读档才清空 Undo。**GIVEN** `Dealing` / `Restoring`，**WHEN** `load_game`，**THEN** `COMMAND_IN_PROGRESS`，状态不变。

### 错误原子性与协调

- **GIVEN** 任意返回错误的命令，**THEN** 状态与当前 N 不变；State 只协调，不修改模型或 Rules；并发命令串行化，`Dealing` / `Restoring` 期间重入返回 `COMMAND_IN_PROGRESS`。
- **GIVEN** `new_game` 期间 Deal Generator 失败，**THEN** 状态不变、Undo 不清空；**GIVEN** `Card & Deck Model.deal` 失败，**THEN** `MODEL_INVARIANT_FAILED`、状态不变、Undo 不清空。
- **GIVEN** 同一 N 再次 `new_game`，**THEN** 视为重新发牌，清空 Undo，发出 `GameStarted`。
- **GIVEN** 事件订阅者抛异常，**THEN** 状态已在发事件前更新，命令仍返回成功；订阅者异常不回滚状态。
- **GIVEN** `snapshot()` 成功，**THEN** 快照包含 N、模型与状态；State 不写文件，序列化归 Save/Resume。
- **GIVEN** 首次启动未指定 N，**THEN** 状态为 `NoGame`；由 UI 调用 `new_game(N)`，State 不生成随机局号。

## Open Questions

| 问题 | Owner | 目标解决时间 |
|------|-------|--------------|
| 首次启动的 N（1 / 上次存档 / 随机） | Toolbar & Menus + user | 首个可玩版本前 |
| 是否提供“下一局/上一局”按钮（原版以选局号为主） | Toolbar & Menus | Toolbar GDD |
| 胜利/卡死覆盖层的时机与可跳过性 | ux-designer | UX 阶段 |
| 存档频率与失败读档的提示 | Save/Resume Game | Save/Resume GDD |
| 算法版本迁移策略 | Save/Resume + architecture | 架构阶段 |
