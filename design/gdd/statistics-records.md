# Statistics & Records

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 原版规则，零改动

## Summary

Statistics & Records 在当前会话内维护局数、胜局、胜率、当前连胜与最长连胜；它消费 Game State Machine 的 `GameStarted` / `GameWon` 事件，按固定规则记录开局、胜利与放弃，并通过只读快照提供给界面。MVP 不落盘，重启清零；它为玩家提供原版式的最小成绩记录，不改变玩法。

> **Quick reference** — Layer: `Feature` · Priority: `MVP` · Key deps: `Game State Machine`

## Overview

Statistics & Records 在当前会话内维护局数、胜利数、胜率、当前连胜与最长连胜，并通过只读快照把这些成绩提供给界面。它消费 Game State Machine 的 `GameStarted` / `GameWon` 事件，不读取牌局模型、不判断胜负、不改变规则；MVP 不落盘，重启应用即清零。它是原版「最小统计」的忠实实现，让玩家能看到自己的成绩，同时不把任何失败变成惩罚。

## Detailed Design

### Core Rules

**1. 范围与所有权**

- 拥有：会话统计计数（局数、胜局、当前连胜、最长连胜）、活动牌局跟踪、重置。
- 不拥有：局号与生命周期（Game State Machine）、牌局模型（Card & Deck Model）、规则（Rules）、界面（Toolbar & Menus）、存档格式（Save/Resume）。
- 只消费 Game State Machine 的事件；不读模型、不读 Rules、不写状态、不写模型。
- MVP 只保存在内存；关闭应用即清零；不做任何文件 I/O。

**2. 数据模型**

- `games_played: int` — 0..2,147,483,647。
- `games_won: int` — 0..`games_played`。
- `current_streak: int` — 0..`games_won`。
- `longest_streak: int` — 0..`games_won`。
- `active_game: { N: int, won: bool } | none` — 内部状态，不对外暴露。
- 派生值：`games_lost = games_played − games_won`（内部诊断用，不进入公开快照）；`win_rate = 0 if games_played = 0 else games_won / games_played`。
- 不变量：`0 ≤ games_won ≤ games_played`；`0 ≤ current_streak ≤ games_won`；`0 ≤ longest_streak ≤ games_won`；`current_streak ≤ longest_streak`。
- `StatsSnapshot` = 公开只读副本 `{ games_played, games_won, win_rate, current_streak, longest_streak }`；不含 `active_game`、不含 `games_lost`。C# 实现为 `readonly record struct`（或等价的不可变值类型）；赋值即复制，天然满足防御性副本。

**3. 事件处理（顺序固定）**

- `on_game_started(N)`：
  1. 若 `active_game` 存在且 `won = false`：记一次 loss → `current_streak = 0`。旧局已在它开始时就计入 `games_played`，此处不再加 `games_played`。
  2. `games_played = games_played + 1`。
  3. `active_game = { N, won: false }`。
  4. `longest_streak` 不变（loss 不增加最长连胜；新局尚未胜利）。
  - **每个 `GameStarted` 都算一局**，包括同一 N 的新局（`new_game(N)` 对同一局号也合法）；不按 N 去重。
- `on_game_won(N)`：
  1. 若 `active_game = none` → 忽略（防御性；正常流程不会发生）。
  2. 若 `active_game.N ≠ N` → 忽略（过期/异局事件）。
  3. 若 `active_game.won = true` → 忽略（同一局的重复 `GameWon`，幂等）。
  4. 否则：`games_won = games_won + 1`；`current_streak = current_streak + 1`；`longest_streak = max(longest_streak, current_streak)`；`active_game.won = true`。
- `on_game_restarted(N)`：**no-op**。不改变计数、连胜或 `active_game`。若活动局已胜利，重开后仍视为已胜利；重开后的再次胜利按重复 `GameWon` 忽略。若活动局未胜利，重开后仍为未胜利；之后的胜利只计一次。
- `on_game_stuck(N)`：**no-op**。卡死是提示性状态，玩家仍可撤销/重开；loss 只在「未胜利就开新局」时记录。
- `GameLoaded`：**MVP 不订阅**（Save/Resume 是 Vertical Slice）；接口**不提供** `on_game_loaded`，State 不得向 Statistics 投递该事件。若通过未知入口收到，按未知事件忽略（防御性，不属于契约）。VS 必须重新设计活动局的对账（见 Open Questions）。
- 未知/异局 N：忽略；Statistics 信任 State 传来的 N 合法性，不重新校验。
- 重复 `GameStarted`（同一 N）：按新局处理；State 必须保证每个 `new_game` 只发一次 `GameStarted`（已与 State GDD 澄清）。
- Statistics 对合法事件不得抛异常；无效事件一律忽略。

**4. 损失与连胜语义**

- **Loss** = `GameStarted` 到达时，`active_game` 存在且 `won = false`。它把 `current_streak` 清零；旧局仍留在 `games_played` 且不在 `games_won`，因此由 `games_lost = games_played − games_won` 自然体现。
- `GameStuck` **不**当场记 loss：卡死仍可撤销/重开本局，直接记输会违反支柱 4「失败可无损重来」。
- `GameRestarted` **不**记 loss、也**不**记新局：重开是无损重来。
- 胜利：`current_streak++`，并更新 `longest_streak`。
- 失败：`current_streak = 0`，`longest_streak` 不变。
- 胜利后开新局：不重置 `current_streak`；`games_played++`，`active_game` 指向新局。
- `games_played` 包含当前进行中的局；`games_won` 只在 `GameWon` 时增加。

**5. 重置**

- `reset_stats()`：清零 `games_played`、`games_won`、`current_streak`、`longest_streak`；`active_game = none`。
- 幂等：已经为零时再次调用无副作用。
- **局中重置**：当前进行中的局从统计中丢弃；之后该局的 `GameWon` 因 `active_game = none` 被忽略。下一次 `GameStarted` 从零开始。
- 重置确认与警告文案由 Toolbar 负责；Statistics 收到调用就无条件重置。

**6. 读接口**

- `get_stats() -> StatsSnapshot`：返回不可变副本；不暴露内部引用；不包含 `active_game`。
- `win_rate` 每次由计数派生，不单独存储，避免除法结果过期。
- 任何事件之前调用 `get_stats()`：全部为 0，`win_rate = 0`。

**7. 会话与持久化**

- MVP 纯内存；关闭应用丢失全部统计与活动局。
- 不调用任何文件/存档 API；不依赖引擎持久化能力。
- `StatsSnapshot` 是 Vertical Slice 持久化的候选载荷；存储位置、格式、与 Save/Resume 的对账属于 VS 的 Open Question。
- 会话内统计不因新局、重开、撤销、读档（MVP 无读档）而清零，只有 `reset_stats()` 清零。

**8. 确定性与并发**

- 同一事件序列 → 同一 `StatsSnapshot`；无随机数、无时间依赖。
- 单线程、串行调用；不加锁、不做并发安全承诺。
- 每个事件与 `get_stats()` 都是 O(1)；不遍历历史、不保存事件日志。

### States and Transitions

Statistics 没有牌局状态；只有内部跟踪状态。

| 状态 | 进入 | 离开 | 说明 |
|------|------|------|------|
| Empty | 初始化；`reset_stats()` | `GameStarted` → Tracking | 无活动局；计数可能非零（重置后为零） |
| Tracking | `GameStarted`；活动局未胜利 | `GameWon` → Won；`GameStarted` → Tracking（旧局记 loss） | 有活动局，`won = false` |
| Won | `GameWon` 匹配活动局 | `GameStarted` → Tracking（不记 loss） | 活动局已胜利；重复 `GameWon` 忽略 |

转移表：

| 当前 | 事件 | 结果 | 说明 |
|------|------|------|------|
| Empty | `GameStarted(N)` | Tracking | `games_played = 1`；无 loss |
| Tracking | `GameStarted(M)` | Tracking | 旧局记 loss（`current_streak = 0`）；`games_played++`；`active = {M, false}` |
| Tracking | `GameWon(N)` 且 N = active.N | Won | `games_won++`；`current_streak++`；更新 `longest_streak` |
| Tracking | `GameWon(N)` 且 N ≠ active.N | Tracking | 忽略 |
| Tracking | `GameRestarted` / `GameStuck` | Tracking | no-op |
| Won | `GameStarted(M)` | Tracking | 无 loss；`games_played++`；`active = {M, false}` |
| Won | `GameWon` 重复 | Won | 忽略（幂等） |
| Won | `GameRestarted` | Won | no-op；重开后的胜利仍被忽略 |
| Won | `GameStuck` | Won | 不应发生；no-op |
| 任意 | `reset_stats()` | Empty | 全部计数清零；active 清空 |
| 任意 | `GameLoaded`（未知入口） | 不变 | MVP 不订阅；若收到按未知事件忽略；VS 待定 |

- 不存在「Lost」状态：loss 是 `GameStarted` 事件在 Tracking 状态上的副作用。
- `get_stats()` 不改变状态。

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|------|------|------|------|------------|
| Game State Machine | 上游 | `GameStarted`、`GameWon`；忽略 `GameRestarted`、`GameStuck`；不订阅 `GameLoaded` | 无 | State 拥有生命周期；Stats 拥有记录 |
| Toolbar & Menus | 下游 | `get_stats` / `reset_stats` 调用 | `StatsSnapshot` | Toolbar 拥有显示与重置确认；Stats 拥有数据 |
| Save/Resume Game | Vertical Slice 下游 | `StatsSnapshot`（VS） | 恢复 | Stats 拥有数据；Save 拥有格式（VS） |
| Card & Deck Model | 无 | — | — | Stats 不读模型；胜负由 State 事件转达 |
| Rules & Move Validator | 无 | — | — | Stats 不判胜负 |
| Undo History | 无 | — | — | Stats 不读撤销历史 |
| Table Rendering / Animation / Audio | 无 | — | — | 统计不直接出现在牌桌或动效中 |
| Settings / Accessibility | 无直接接口 | — | — | 重置入口经 Toolbar；数字可读性归 Accessibility |

**接口契约：**

- `on_game_started(N: int) -> void`
- `on_game_won(N: int) -> void`
- `on_game_restarted(N: int) -> void`（no-op）
- `on_game_stuck(N: int) -> void`（no-op）
- `get_stats() -> StatsSnapshot`
- `reset_stats() -> void`
- `StatsSnapshot = { games_played, games_won, win_rate, current_streak, longest_streak }`，全部只读。
- Statistics 信任 State 传来的 N；不重新校验局号范围。
- Statistics 不调用 State、模型、Rules 的任何写入或查询接口；只接收事件。
- State 事件在状态更新后同步发出；Statistics 按发出顺序处理，不重排。
- 合法事件不得抛异常；异局/无活动局/重复事件一律静默忽略。
- Toolbar 在用户确认后调用 `reset_stats()`；Statistics 不负责确认对话框。
- `StatsSnapshot` 是不可变副本；调用方修改它不得影响内部状态。

**Provisional assumptions（依赖未显式提供，先假定）：**

1. **`GameStarted` 每个成功的 `new_game` 恰好一次，且同一 N 可再次发。** Statistics 把每个 `GameStarted` 当作新局；不按 N 去重。State GDD 已同步澄清为「`GameWon` 按活动局实例幂等；`GameStarted` 每个成功的 `new_game` 恰好一次，不按 N 去重」。
2. **`GameWon` 每次进入 Won 只发一次。** 重复投递由 Statistics 幂等忽略。
3. **事件同步、有序、不重入。** Statistics 不需要队列或锁。
4. **Save/Resume 在 MVP 不存在。** `GameLoaded` 不订阅；VS 必须重新设计活动局对账。
5. **Toolbar 是唯一显示/重置消费者。** 不做常驻 HUD；统计视图由 Toolbar/Menus 提供。
6. **计数器为 int32。** 溢出是实现错误，不得回绕；会话内实际不可达。

## Formulas

> 本节定义 Statistics 的全部数值规则：`事件 → StatsEventKind → 计数器 → 派生值`。所有公式是纯函数：无随机数、无时间、无 I/O、不调用引擎 API；同一事件序列必然得到同一 `StatsSnapshot`。计数器为 int32；唯一浮点值是每次读取重算的 `win_rate`。只覆盖 MVP；`GameLoaded` 对账与持久化留给 Vertical Slice。约定：`∧` 逻辑与，`∈` 属于，`none` 无活动局，`′` 更新后值，`/` 为实数除法。

### 事件分类枚举 StatsEventKind

| 取值 | 产生条件 | 语义 | 计数作用 |
|---|---|---|---|
| `NEW_GAME` | `GameStarted` 且（`active_game = none` 或 `active_game.won = true`） | 新局，不记 loss | `games_played + 1` |
| `LOSS` | `GameStarted` 且 `active_game ≠ none` 且 `active_game.won = false` | 放弃未完成局并开新局 | `games_played + 1`；`current_streak = 0` |
| `WIN` | `GameWon` 且 `active_game ≠ none` 且 `active_game.N = N` 且 `active_game.won = false` | 活动局获胜 | `games_won + 1`；`current_streak + 1` |
| `NEUTRAL` | 其余全部：`GameRestarted`、`GameStuck`、`GameLoaded`（MVP）、异局/重复/无活动局的 `GameWon`、未订阅事件 | 不改变统计 | 无 |

`LOSS` 不是独立损失事件，而是「未完成活动局上到达的 `GameStarted`」，因此同时满足新局条件。`LOSS` 必须先于 `NEW_GAME` 求值。`GameStarted` 永不分类为 `WIN`/`NEUTRAL`；`GameWon` 永不分类为 `LOSS`/`NEW_GAME`。

### The classify_event formula

The classify_event formula is defined as:

`classify_event(raw_event, active_game, N) = WIN if raw_event = GameWon ∧ active_game ≠ none ∧ active_game.N = N ∧ active_game.won = false; LOSS if raw_event = GameStarted ∧ active_game ≠ none ∧ active_game.won = false; NEW_GAME if raw_event = GameStarted; NEUTRAL otherwise`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| raw_event | enum | {GameStarted, GameWon, GameRestarted, GameStuck, GameLoaded} | State GDD §7 | 输入事件类型 |
| active_game | `{N, won}` 或 `none` | `none` 或 N 1–1,000,000，won {false,true} | 内部状态 | 当前跟踪的牌局 |
| N | int | 1–1,000,000（信任 State，不复验） | 事件载荷 | 事件携带局号 |
| classify_event | enum | {NEW_GAME, LOSS, WIN, NEUTRAL} | 本公式 | 事件效果分类 |

**Output Range:** 枚举四值，无大小序；四分支穷尽且互斥（`WIN`/`LOSS` 分属不同 raw_event 本就互斥；`LOSS` 必须先于 `NEW_GAME`）。

**Example:** `(none, GameStarted(1)) → NEW_GAME`；`({42,false}, GameStarted(43)) → LOSS`；`({42,true}, GameStarted(43)) → NEW_GAME`；`({42,false}, GameWon(42)) → WIN`；`({42,true}, GameWon(42)) → NEUTRAL`；`GameRestarted / GameStuck → NEUTRAL`。

### The games_played_after_event formula

The games_played_after_event formula is defined as:

`games_played_after_event(games_played, event) = games_played + 1 if event ∈ {NEW_GAME, LOSS}, else games_played`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| games_played | int32 | 0–2,147,483,646（+1 输入域） | 前一状态 | 更新前累计局数 |
| event | StatsEventKind | {NEW_GAME, LOSS, WIN, NEUTRAL} | classify_event | 事件效果分类 |
| games_played_after_event | int32 | 0–2,147,483,647 | 本公式 | 更新后累计局数 |

**Output Range:** 0–2,147,483,647。不增量时等于输入（最小 0）；增量时最小 1（输入 0 + NEW_GAME），最大 2,147,483,647（输入 2,147,483,646）。输入为 2,147,483,647 且事件为 NEW_GAME/LOSS 时 `+1` 不可表示——**这是全节唯一可能的整数溢出点**；越界是定义域外的实现错误。C# 实现必须使用 `checked`，抛出 `OverflowException` 且状态不变；不得回绕、不得静默截断。

**Example:** `(0, NEW_GAME) → 1`；`(7, LOSS) → 8`；`(7, WIN) → 7`；`(7, NEUTRAL) → 7`。

### The games_won_after_event formula

The games_won_after_event formula is defined as:

`games_won_after_event(games_won, event) = games_won + 1 if event = WIN, else games_won`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| games_won | int32 | 0–games_played，≤ 2,147,483,647 | 前一状态 | 更新前胜局数 |
| event | StatsEventKind | {NEW_GAME, LOSS, WIN, NEUTRAL} | classify_event | 事件效果分类 |
| games_won_after_event | int32 | 0–2,147,483,647 | 本公式 | 更新后胜局数 |

**Output Range:** 0–2,147,483,647。可达状态下 `WIN` 必要求活动局未胜利，故 `games_won ≤ games_played − 1`，`+1 ≤ games_played ≤ INT32_MAX`，**不会溢出**。公式本身不强制 `games_won ≤ games_played`；该不变量由组合维持。

**Example:** `(0, WIN) → 1`；`(3, WIN) → 4`；`(3, LOSS) → 3`；`(3, NEUTRAL) → 3`。

### The current_streak_after_event formula

The current_streak_after_event formula is defined as:

`current_streak_after_event(current_streak, event) = current_streak + 1 if event = WIN; 0 if event = LOSS; current_streak otherwise`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| current_streak | int32 | 0–games_won | 前一状态 | 更新前当前连胜 |
| event | StatsEventKind | {NEW_GAME, LOSS, WIN, NEUTRAL} | classify_event | 事件效果分类 |
| current_streak_after_event | int32 | 0–games_won′ | 本公式 | 更新后当前连胜 |

**Output Range:** 0–games_won′。最小 0（LOSS，或从 0 起不增量）；最大为 `WIN` 时 `games_won′`。不会溢出：`WIN` 要求活动局未胜利，故 `current_streak ≤ games_won ≤ games_played − 1`，`+1 ≤ INT32_MAX`。`LOSS` 的 0 是赋值，不是减一；连续 LOSS 保持 0。

**Example:** `(0, WIN) → 1`；`(4, WIN) → 5`；`(4, LOSS) → 0`；`(4, NEW_GAME) → 4`；`(4, NEUTRAL) → 4`；`(0, LOSS) → 0`。

### The longest_streak_after_event formula

The longest_streak_after_event formula is defined as:

`longest_streak_after_event(longest_streak, current_streak_after) = max(longest_streak, current_streak_after)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| longest_streak | int32 | 0–games_won | 前一状态 | 更新前最长连胜 |
| current_streak_after | int32 | 0–games_won′ | current_streak_after_event 的输出 | 更新后当前连胜，**必须先算** |
| longest_streak_after_event | int32 | 0–games_won′ | 本公式 | 更新后最长连胜 |

**Output Range:** 0–games_won′。最小 0（两者皆 0）；最大为两输入较大者。除 `reset_stats()` 外非递减；只有 `WIN` 可能使其增加，且只增加 1。**必须消费更新后的 `current_streak`**，否则「在 current = longest 时获胜」会永远漏记一次。

**Example:** `(5, 4) → 5`；`(4, 5) → 5`；`(4, 4)` 且本事件为 WIN（current → 5）→ 5；`(0, 1) → 1`。

### The active_game_after_event formula

The active_game_after_event formula is defined as:

`active_game_after_event(active_game, event, N) = {N, won: false} if event ∈ {NEW_GAME, LOSS}; {active_game.N, won: true} if event = WIN; active_game otherwise`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| active_game | `{N, won}` 或 `none` | `none` 或 N 1–1,000,000 | 内部状态 | 更新前跟踪的牌局 |
| event | StatsEventKind | {NEW_GAME, LOSS, WIN, NEUTRAL} | classify_event | 事件效果分类 |
| N | int | 1–1,000,000（信任 State） | 事件载荷 | 本事件局号 |
| active_game_after_event | `{N, won}` 或 `none` | 同上 | 本公式 | 更新后跟踪的牌局 |

**Output Range:** `none` 仅来自 `none` + NEUTRAL；`{N,false}` 来自 NEW_GAME/LOSS；`{N,true}` 来自 WIN。N 对 Statistics 不透明、不复验。`reset_stats()` 直接把 active_game 置 `none`，不经过本公式。

**Example:** `(none, NEW_GAME, 1) → {1,false}`；`({42,false}, LOSS, 43) → {43,false}`；`({42,false}, WIN, 42) → {42,true}`；`(none, NEUTRAL, 7) → none`。

### The games_lost formula

The games_lost formula is defined as:

`games_lost(games_played, games_won) = games_played − games_won`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| games_played | int32 | 0–2,147,483,647 | 计数器 | 累计局数 |
| games_won | int32 | 0–games_played | 计数器 | 累计胜局数 |
| games_lost | int32 | 0–games_played | 本公式（派生，不存储） | 未获胜局数 |

**Output Range:** 0–games_played。最小 0（won = played，含 0/0）；最大 games_played（won = 0）。不变量成立时不会为负；若被 bug 破坏则为负——**不要 clamp**，按内部不变量失败处理。语义：统计**所有尚未获胜的局，包括进行中的活动局**，因此新局一开始 `games_lost` 就 +1；名称易误读，保持内部，不进入 `StatsSnapshot`。

**Example:** `(10, 7) → 3`（即使第 10 局进行中）；`(0,0) → 0`；`(5,0) → 5`；`(5,5) → 0`。

### The win_rate formula

The win_rate formula is defined as:

`win_rate(games_played, games_won) = 0 if games_played = 0, else games_won / games_played`（实数除法）

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| games_played | int32 | 0–2,147,483,647 | 计数器 | 累计局数 |
| games_won | int32 | 0–games_played | 计数器 | 累计胜局数 |
| win_rate | float64（C# double） | [0, 1] | 本公式（派生，不存储） | 胜率 |

**Output Range:** 0.0–1.0（含端点）。最小 0.0：`games_played = 0`（按定义）或 `won = 0` 且 `played > 0`；最大 1.0：`won = played > 0`。无 NaN（分支排除 0/0），无 ∞（else 分支分母 > 0）。必须是实数除法：int32 直接 `/` 会截断为 {0,1}，实现须先转 double。精度：double 正确舍入有理数；只有 0.0、1.0 与二进可表示分数（1/2、3/4…）精确，1/3、7/10 不精确。不存储、不累加舍入值，每次 `get_stats()` 重算；百分比格式与舍入方式归 Toolbar。

**Example:** `(0,0) → 0.0`；`(8,6) → 0.75`（精确）；`(10,7) → 0.7`（最接近 7/10 的 double）；`(5,0) → 0.0`；`(5,5) → 1.0`；`(3,1) → 0.333…`。

### The stats_after_event formula

The stats_after_event formula is defined as:

`stats_after_event(S, raw_event, N) = (played′, won′, current′, longest′, active′)`，按以下固定顺序计算：

1. `kind = classify_event(raw_event, S.active, N)`
2. `played′ = games_played_after_event(S.played, kind)`
3. `won′ = games_won_after_event(S.won, kind)`
4. `current′ = current_streak_after_event(S.current, kind)`
5. `longest′ = longest_streak_after_event(S.longest, current′)`
6. `active′ = active_game_after_event(S.active, kind, N)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| S | StatsState | 满足四条不变量的状态 | 内部状态 | 更新前 `{played, won, current, longest, active}` |
| raw_event | enum | 与 classify_event 相同 | State 事件 | 输入事件 |
| N | int | 1–1,000,000 | 事件载荷 | 事件局号 |
| stats_after_event | StatsState | 满足四条不变量 | 本公式 | 更新后状态 |

**Output Range:** 合法 `StatsState`；`reset_stats()` 单独定义为 `{0,0,0,0,none}`，不经过本公式。组合顺序不可交换：`longest` 必须消费更新后的 `current`，`active` 必须消费分类结果。

**Example:** `S = {1,0,0,0,{42,false}}`，`GameWon(42)` → `kind = WIN` → `{1,1,1,1,{42,true}}`。`S = {1,0,0,0,{42,false}}`，`GameStarted(43)` → `kind = LOSS` → `{2,0,0,0,{43,false}}`。

### 不变量

- `0 ≤ games_won ≤ games_played`：`WIN` 只可能在 `active_game ≠ none` 时发生，而 `active_game` 只由 `NEW_GAME`/`LOSS` 创建，这两者已在同一事件中执行 `games_played + 1`；`reset_stats()` 同时清零。
- `0 ≤ current_streak ≤ games_won`：`WIN` 使两者同时 +1；`LOSS` 置 0。
- `0 ≤ longest_streak ≤ games_won`：`max(旧值, current′)`，且 `current′ ≤ games_won′`。
- `current_streak ≤ longest_streak`：`WIN` 后 `longest′ ≥ current′`。
- 四条对事件序列归纳成立；任何一条被破坏都是实现错误，不是可恢复状态。

### 常量与可调性

本系统零可调参数：统计口径由 Detailed Design 固定，没有阈值、权重或曲线。计数器为 int32，`win_rate` 为派生 double；显示格式（百分比、小数位、舍入）归 Toolbar，Statistics 不存格式化值。`games_lost` 是内部派生值，不进入公开快照，也不注册为跨系统公式。唯一跨系统数值是 `win_rate`，它通过 `StatsSnapshot` 流向 Toolbar。

## Edge Cases

- **If 初始空状态（无任何事件）**：`get_stats()` 返回 `{0, 0, 0.0, 0, 0}`；`win_rate = 0.0`，不是 NaN。
- **If 首个事件是 `GameStarted(1)`**：`games_played = 1`，不记 loss，`current_streak = 0`，`longest_streak = 0`。
- **If 未胜利时开新局**：旧局记一次 loss（`current_streak = 0`，`longest_streak` 不变），`games_played + 1`；旧局不会再次计入 `games_played`。
- **If 直接调用 `new_game(N)`（不是 UI 选局）且 N = 当前 N**：按新局处理并记 loss，**不按 N 去重**；UI 的「选局」在同一 N 时由 State 路由为 `restart_deal()`（`GameRestarted`），因此不记 loss、不增局数。
- **If 胜利后开新局**：不记 loss；`current_streak` 保留；`games_played + 1`；`active_game` 指向新局。
- **If 收到异局 `GameWon`（N 与活动局不同）**：忽略；活动局仍未胜利；随后的 `GameStarted` 会把它记成 loss。
- **If 同一活动局收到重复 `GameWon`**：忽略，计数不变（`active_game.won` 幂等）。
- **If 无活动局时收到 `GameWon`（重置后或乱序）**：忽略，不增加 `games_won`。
- **If 收到 `GameRestarted` 或 `GameStuck`**：no-op；`Stuck → 重开 → 获胜` 正常计一胜；`Stuck → 开新局` 在新局 `GameStarted` 时记 loss。
- **If 胜利后重开本局（`GameRestarted`）**：`active_game.won` 保持 true；重开后的胜利被忽略；随后开新局也不记 loss——总计仍是一局一胜。
- **If 连胜语义**：两连胜 → `current = 2, longest = 2`；一败 → `current = 0, longest = 2`；再胜 → `current = 1, longest = 2`。`longest_streak` 只在 `reset_stats()` 时下降。
- **If 局中调用 `reset_stats()`**：全部计数清零、`active_game = none`；该局之后的 `GameWon` / `GameRestarted` 均被忽略；只有新的 `GameStarted` 重新武装跟踪。
- **If `reset_stats()` 在已经全零时调用**：幂等；下一次 `GameStarted` 得到 `{1,0,0.0,0,0}`，不记 loss。
- **If 0 胜**：`win_rate = 0.0`；**If 全胜**：`win_rate = 1.0`（精确）；开新局会立即把进行中的局计入分母，胜率当场下降。
- **If `games_played = INT32_MAX` 且收到 `GameStarted`**：`games_played + 1` 不可表示，属于定义域外的实现错误。C# 实现使用 `checked`，抛出 `OverflowException`，状态保持不变；不得回绕、不得静默截断。
- **If 收到未知/无效事件**：分类为 `NEUTRAL`，不改变任何计数，不抛异常。
- **If 调用方修改了 `get_stats()` 返回的对象**：内部状态不受影响；再次 `get_stats()` 返回原值。
- **If 同一事件序列重复执行两次**：两次得到的 `StatsSnapshot` 逐字段相同。
- **If MVP 通过未知入口收到 `GameLoaded`**：按未知事件忽略；MVP 不订阅、不提供 `on_game_loaded`。Vertical Slice 的读档对账必须重新设计，不能直接照搬 NEUTRAL。
- **If `Stuck` 后撤销回到 `Playing` 再获胜**：不记 loss（`GameStuck` 是 no-op），胜利计一次。
- **If 应用关闭时有进行中的局**：MVP 会话统计全部丢弃，不记 loss；持久化后的行为留到 VS 定义。
- **If 胜利后直接调用 `new_game(N)` 重开同一局号**：计新局、不记 loss；再次获胜会再次计数。UI 的「选局」在同一 N 时路由为 `restart_deal()`，不会触发该路径；直接 API 调用仍可复现，但玩家 UI 不暴露。
- **If State 违约传入非法 N（越界/负数）**：Statistics 不校验，公式仍可计算，但行为不在契约内；由 State 的校验保证。
- **If Toolbar 用浮点除法重算胜率**：`0/0` 会得到 NaN；禁止。整数百分比应使用 `win_percentage_exact(games_won, games_played)`（Toolbar GDD），它从快照计数器做整数半进位，不经过浮点。

## Dependencies

**上游依赖：**

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Game State Machine | Hard | `GameStarted { N }`、`GameWon { N }` | State 拥有生命周期；Statistics 只消费事件，不读模型 |

**下游依赖（依赖本系统的系统）：**

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Toolbar & Menus | Hard（玩家可见性） | `get_stats` / `reset_stats` | Toolbar 拥有统计视图与重置确认；无 Toolbar 则统计不可见、不可重置 |
| Save/Resume Game | Soft（Vertical Slice） | `StatsSnapshot` | Stats 拥有数据；Save 拥有格式；MVP 不实现 |
| Card & Deck Model | None | — | Statistics 不直接读取模型 |
| Rules & Move Validator | None | — | Statistics 不判断胜负 |
| Undo History | None | — | Statistics 不读撤销历史 |
| Table Rendering / Animation / Audio | None | — | 统计不直接出现在牌桌或动效中 |
| Settings / Accessibility | None 直接 | — | 重置入口经 Toolbar；数字可读性归 Accessibility |

**接口契约：**

- **读事件**：`on_game_started(N) -> void`、`on_game_won(N) -> void`、`on_game_restarted(N) -> void`（no-op）、`on_game_stuck(N) -> void`（no-op）。
- **不订阅**：MVP 不订阅 `GameLoaded`，不提供 `on_game_loaded`；State 不得向 Statistics 投递该事件。
- **读查询**：`get_stats() -> StatsSnapshot`，返回不可变副本。
- **命令**：`reset_stats() -> void`；幂等；调用方负责确认对话框。
- **写**：Statistics 不写模型、不写状态、不写文件；只维护自己的计数器。
- **事件顺序**：State 在状态更新后同步发出事件；Statistics 按发出顺序处理，不重排。
- **信任 N**：Statistics 不校验 N 的范围；由 State 的 `new_game` 校验保证。
- **不抛异常**：合法事件不得导致异常；异局/重复/无活动局事件静默忽略。

**双向一致性：**

- Game State Machine GDD 将 Statistics 列为 Soft 下游，接口为 `GameStarted` / `GameWon`；本 GDD 将 State 列为 Hard 上游（无事件则统计无法更新）。方向不同，强度不同，二者一致。
- 本次已同步修正 State GDD §7：Statistics 消费 `GameStarted` 与 `GameWon`；provisional assumption 已澄清为「`GameWon` 按活动局实例幂等；`GameStarted` 每个成功的 `new_game` 恰好一次，不按 N 去重」。
- **Toolbar & Menus 尚无 GDD**；系统索引已在 2026-10-24 将 8 补入 Toolbar 的依赖行。其 GDD 编写时仍必须把 Statistics 列为上游并引用本节接口。下游依赖缺口已在索引层关闭，UI 层待 Toolbar GDD。
- Save/Resume Game 尚无 GDD；Vertical Slice 编写时必须决定统计持久化归谁、格式如何、读档如何对账。

**Provisional assumptions（依赖未显式提供，先假定）：**

1. **`GameStarted` 每个成功的 `new_game` 恰好一次，且同一 N 可再次发。** Statistics 把每个 `GameStarted` 当作新局；不按 N 去重。State GDD 已同步澄清。
2. **`GameWon` 每次进入 Won 只发一次；重复投递按活动局实例幂等忽略。**
3. **事件同步、有序、不重入。** Statistics 不需要队列或锁。
4. **Save/Resume 在 MVP 不存在。** `GameLoaded` 不订阅；VS 必须重新设计活动局对账。
5. **Toolbar 是唯一显示/重置消费者。** 不做常驻 HUD；统计视图由 Toolbar/Menus 提供。
6. **计数器为 int32。** `games_played = INT32_MAX` 后再开新局是定义域外错误，不得回绕。
7. **Stats 是会话级。** 关闭应用即清零；MVP 无文件 I/O。

**Registry references：** 已新增 `win_rate`（variables=[games_played, games_won]，output=[0,1]）作为跨系统公式；`games_lost` 与六个转移/组合公式是内部实现，不注册。`StatsSnapshot` 的计数字段是状态而非常量，不注册。Toolbar GDD 编写后，把其路径追加到 `win_rate.referenced_by`。

## Visual/Audio Requirements

- **无直接视觉/音频。** Statistics 只产生数字；统计视图由 Toolbar & Menus 绘制，本系统不拥有屏幕、面板、图表、动画或音效。
- 若 Toolbar 提供统计对话框：数字必须可读，沿用 Table Rendering 的字体与缩放规则；0.8 缩放下仍清晰；不使用颜色作为唯一信息通道（胜率同时以数字显示）。
- 无图表、无进度环、无动画、无音效；不做「成就弹出」「连胜庆祝」等反馈。原版统计是安静的记录，不是奖励循环。
- 无新增资产；不使用牌面图集之外的任何贴图。
- 视觉细节（字号、间距、标签）归 Toolbar/UX spec；本 GDD 只要求「数字正确、可读、不装饰」。

## Game Feel

- **安静、即时、无存在感。** 统计在后台更新，不打断牌局、不弹窗、不发声、不改变光标或焦点。
- 打开统计视图应立即显示当前快照；不做加载动画、数字滚动或淡入。
- 胜利时只更新数据，不播放「成就」「连胜」特效；Reset 立即生效（确认对话框由 Toolbar 负责），不做撤销动画。
- 数字变化不应制造压力：胜率下降只反映记录，不影响玩法、不改变提示或状态。玩家可以忽略统计继续游戏。
- 0.8 缩放与 1280×720 下数字仍可读；不使用颜色/图标作为唯一反馈。
- 无待机重绘：统计视图只在打开或显式刷新时读取；不逐帧更新。

## UI Requirements

- **入口**：Toolbar & Menus 提供一个只读统计视图（菜单项或对话框）；不放在牌桌常驻 HUD 上。具体屏幕/菜单项归 Toolbar GDD 与 UX spec。
- **字段**：局数、胜局、胜率、当前连胜、最长连胜。标签文案归 Toolbar/本地化；Statistics 只提供数值。
- **胜率格式**：UI 不得用浮点除法重算 `win_rate`；显示整数百分比时，可用 `StatsSnapshot` 的 `games_won` / `games_played` 做整数半进位（`win_percentage_exact`，见 Toolbar GDD），避免 0/0 NaN 与银行家舍入。建议显示为整数百分比以便「一秒读懂」；精确的小数位由 UX spec 决定。
- **刷新**：打开视图时读取一次快照；不要求实时刷新。只有关闭再打开时重读；不逐帧更新，也不在重新获得焦点时刷新。
- **重置入口**：Toolbar 提供「重置统计」动作，调用 `reset_stats()`；必须二次确认，文案需说明「不可撤销、当前进行中的局不会计入」。确认文案归 Toolbar/本地化。
- **无障碍**：统计视图键盘可达；0.8 缩放下数字可读；胜率同时以数字显示，不依赖颜色或图标。
- **不做**：常驻 HUD、图表、成就、连胜特效、统计分享、导出、排行榜。

## Cross-References

| 文档 | 章节 | 关系 |
|------|------|------|
| `design/gdd/game-state-machine.md` | Core Rules §3/§7、Dependencies、Provisional assumptions | `GameStarted` / `GameWon` 事件；本次已同步修正 §7 与 Statistics 幂等假设 |
| `design/gdd/toolbar-menus.md`（已完成） | UI | 统计视图、胜率格式（`win_percentage_exact`）、重置确认；系统索引 Toolbar 行已补 8（2026-10-24） |
| `design/gdd/save-resume-game.md`（未编写） | Persistence（VS） | 统计持久化、`GameLoaded` 对账、跨会话活动局 |
| `design/gdd/game-concept.md` | MVP Definition / Pillars | 「局数、胜率、连胜」的最小统计；MVP 会话内保存；支柱 4「失败可无损重来」 |
| `design/gdd/systems-index.md` | 第 8 行 / Leaf systems / Toolbar 行 | 依赖 4；Toolbar 行已补 8（2026-10-24）；Statistics 已移出 Leaf systems |
| `design/registry/entities.yaml` | formulas | 已新增 `win_rate`；内部转移公式不注册 |
| `design/gdd/card-deck-model.md` | — | 无直接依赖；Statistics 不读模型 |
| `design/gdd/rules-move-validator.md` | — | 无直接依赖；Statistics 不判胜负 |
| `design/gdd/undo-history.md` | — | 无直接依赖；Statistics 不读撤销历史 |
| `design/gdd/hint-system.md` | — | 无直接依赖 |

**已知跨文档事项：**

- State GDD 原先写 `on_game_started(N)` 「幂等」，与同一 N 的合法 `new_game` 冲突；本次已改为「`GameWon` 按活动局实例幂等；`GameStarted` 每个成功的 `new_game` 恰好一次，不按 N 去重」。
- 系统索引 Toolbar 行已补上 `8`（2026-10-24）；统计可见性依赖已在索引层建立，Toolbar GDD 仍需引用 Statistics。
- `GameLoaded` 由 State 发出，但 Statistics 在 MVP 不订阅；Vertical Slice 的读档对账必须在 Save/Resume/Statistics 两侧重新设计。
- 概念中的「失败可无损重来」与「放弃未完成局会拉低胜率」存在意图张力：统计只记录、不改变玩法，且提供重置；是否接受由 Open Question 跟踪。

## Acceptance Criteria

> **测试约定。** S0 = 新建 Statistics、未投递任何事件；`get_stats()` 返回 `{games_played:0, games_won:0, win_rate:0.0, current_streak:0, longest_streak:0}`。事件记法：`Started(N)` / `Won(N)` / `Restarted(N)` / `Stuck(N)`；MVP 不提供 `Loaded(N)`。快照按五字段顺序比较；浮点容差 1e-15，标「精确」的除外。`StatsSnapshot` 记法 `{played, won, win_rate, current, longest}`；`StatsState` 记法 `{played, won, current, longest, active}`（含内部 active）。`active_game` 与 `games_lost` 不在快照中，通过测试专用内部视图读取。逻辑测试放 `tests/unit/statistics/`（BLOCKING）；跨系统集成测试放 `tests/integration/statistics-state/`（BLOCKING）；「无 I/O / 不持有模型引用」以架构/依赖检查为证据，不以黑盒行为断言。

### Core Rules

- **AC-C1（接口边界）** GIVEN 新实例，WHEN 检查公开接口，THEN 仅存在 `on_game_started` / `on_game_won` / `on_game_restarted` / `on_game_stuck` / `get_stats` / `reset_stats` 六个契约方法；不存在读取模型、Rules 或 State 的公开方法，也不持有它们的对象引用（架构检查）。
- **AC-C2（会话清零）** GIVEN 任意事件序列，WHEN 应用关闭并重启，THEN `get_stats()` 为 S0，且启动过程未读取任何统计/存档文件。
- **AC-C3（S0 快照）** GIVEN S0，WHEN `get_stats()`，THEN 恰好返回五个字段且值为 `{0,0,0.0,0,0}`；不含 `active_game`、不含 `games_lost`。
- **AC-C4（进行中局计入分母）** GIVEN S0，WHEN `Started(1)`，THEN 快照 `{1,0,0.0,0,0}`；`win_rate` 已以该局为分母。
- **AC-C5（首个新局）** GIVEN S0，WHEN `Started(1)`，THEN 不记 loss，`current_streak = 0`，`longest_streak = 0`，`active_game = {1,false}`。
- **AC-C6（未胜开新局）** GIVEN 快照 `{1,0,0,0}` 且 active `{42,false}`，WHEN `Started(43)`，THEN 快照 `{2,0,0.0,0,0}`、active `{43,false}`；旧局记一次 loss，且不再次计入 `games_played`。
- **AC-C7（胜利后开新局）** GIVEN 快照 `{1,1,1.0,1,1}` 且 active `{42,true}`，WHEN `Started(43)`，THEN 快照 `{2,1,0.5,1,1}`、active `{43,false}`；不记 loss，`current_streak` 保留。
- **AC-C8（胜利计数）** GIVEN active `{42,false}`，WHEN `Won(42)`，THEN 快照 `{1,1,1.0,1,1}`、active `{42,true}`；`games_won`、`current_streak`、`longest_streak` 各 +1。
- **AC-C9（无活动局胜利）** GIVEN active 为 none，WHEN `Won(7)`，THEN 快照与 active 逐字段不变。
- **AC-C10（异局胜利）** GIVEN active `{42,false}`，WHEN `Won(43)`，THEN 快照与 active 逐字段不变；active 仍为 `{42,false}`。
- **AC-C11（重复胜利）** GIVEN active `{42,true}` 且已计胜，WHEN 再次 `Won(42)`，THEN 快照逐字段不变（幂等）。
- **AC-C12（重开/卡死 no-op）** GIVEN 任意状态，WHEN `Restarted(N)` 或 `Stuck(N)`，THEN 快照与 active 逐字段不变。
- **AC-C13（读档不进入 MVP 契约）** GIVEN MVP 接口，WHEN 检查方法表，THEN 不存在 `on_game_loaded`；若通过未知入口投递 `GameLoaded`，按未知事件忽略（防御性）。
- **AC-C14（不抛异常）** GIVEN 任意契约内事件序列，WHEN 逐条投递，THEN 不抛异常；异局、重复、无活动局事件静默忽略。
- **AC-C15（连胜序列）** GIVEN `Started(1), Won(1), Started(2), Won(2)`，THEN `{2,2,1.0,2,2}`；WHEN `Started(3)`（2 已胜），THEN `{3,2,2/3,2,2}` 无 loss；WHEN `Started(4)`（3 未胜），THEN `{4,2,0.5,0,2}`；WHEN `Won(4)`，THEN `{4,3,0.75,1,2}`。
- **AC-C16（卡死不立刻算输）** GIVEN active 未胜，WHEN `Stuck(N)`，THEN 不记 loss；WHEN 随后 `Started(M)`，THEN 才记 loss（`current_streak = 0`，`games_played + 1`）。
- **AC-C17（重开不算新局）** GIVEN active 未胜，WHEN `Restarted(N)`，THEN 不记 loss、不增 `games_played`、active 不变；WHEN 随后 `Won(N)`，THEN 只计一胜。
- **AC-C18（放弃不重复计数）** GIVEN 未胜活动局被放弃，WHEN `Started(M)`，THEN `current_streak = 0`、`longest_streak` 不变，旧局不再次计入 `games_played`。
- **AC-C19（重置）** GIVEN 任意非零状态且存在 active，WHEN `reset_stats()`，THEN 快照为 S0、active 为 none；连续两次调用仍为 S0。
- **AC-C20（局中重置）** GIVEN 有 active 且有计数，WHEN `reset_stats()` 后 `Won(原局号)`，THEN 快照仍为 S0；WHEN `Started(1)`，THEN `{1,0,0.0,0,0}`，不记 loss。
- **AC-C21（重置无条件）** GIVEN 任意状态，WHEN 调用 `reset_stats()`，THEN 无条件生效，不返回错误、不要求外部确认。
- **AC-C22（读接口无副作用）** GIVEN 任意状态，WHEN 连续两次 `get_stats()`，THEN 两次逐字段相等，且不影响后续事件处理。
- **AC-C23（快照值语义）** GIVEN 已取得的一份快照，WHEN 之后发生任意事件，THEN 旧快照逐字段不变；WHEN 调用方修改返回对象，THEN 内部状态与下一次 `get_stats()` 不受影响。
- **AC-C24（胜率派生）** GIVEN 任意状态，WHEN `get_stats()`，THEN `win_rate` 每次由 `games_won / games_played` 现算；内部不存在独立存储的 `win_rate`。
- **AC-C25（单调性）** GIVEN 任意长事件序列且未调用 `reset_stats()`，WHEN 逐步读取，THEN `games_played` 与 `longest_streak` 单调不减；计数只按事件规则变化，没有自动清零。
- **AC-C26（确定性）** GIVEN 同一事件序列，WHEN 在两个全新实例上分别重放，THEN 每个中间快照与最终快照逐字段相同；GIVEN 序列 A，WHEN 执行 A、`reset_stats()`、再执行 A，THEN 两轮快照相同。
- **AC-C27（同步可见）** GIVEN 任意事件，WHEN 处理器返回，THEN `get_stats()` 立即可见新值；不存在队列或延迟。
- **AC-C28（无锁/无日志）** GIVEN 实现，WHEN 检查结构，THEN 不存在锁、后台任务、事件队列、事件日志或随事件数增长的集合（架构检查）。

### Formulas

- **AC-F1（classify_event）** GIVEN 下表输入，WHEN 调用 `classify_event`，THEN 返回对应分类：

| raw_event | active_game | N | 期望 |
|---|---|---|---|
| GameStarted | none | 1 | NEW_GAME |
| GameStarted | `{42,false}` | 43 | LOSS |
| GameStarted | `{42,false}` | 42 | LOSS |
| GameStarted | `{42,true}` | 43 | NEW_GAME |
| GameWon | `{42,false}` | 42 | WIN |
| GameWon | `{42,true}` | 42 | NEUTRAL |
| GameWon | `{42,false}` | 43 | NEUTRAL |
| GameWon | none | 42 | NEUTRAL |
| GameRestarted / GameStuck | 任意 | 任意 | NEUTRAL |

- **AC-F1b（分支穷尽）** GIVEN 全部 raw_event × active ∈ {none, `{42,false}`, `{42,true}`} × N ∈ {42,43} 组合，WHEN 分类，THEN 恰好命中四个分支之一；GameStarted 永不返回 WIN/NEUTRAL；GameWon 永不返回 LOSS/NEW_GAME；LOSS 先于 NEW_GAME。
- **AC-F2（games_played_after_event）** GIVEN `{(0,NEW_GAME),(7,LOSS),(7,WIN),(7,NEUTRAL)}`，WHEN 计算，THEN 依次返回 `{1,8,7,7}`。
- **AC-F3（games_won_after_event）** GIVEN `{(0,WIN),(3,WIN),(3,LOSS),(3,NEUTRAL)}`，WHEN 计算，THEN 依次返回 `{1,4,3,3}`；只有 WIN 增量。
- **AC-F4（current_streak_after_event）** GIVEN `{(0,WIN),(4,WIN),(4,LOSS),(4,NEW_GAME),(4,NEUTRAL),(0,LOSS)}`，WHEN 计算，THEN 依次返回 `{1,5,0,4,4,0}`；LOSS 赋 0，不是减一。
- **AC-F5（longest_streak_after_event）** GIVEN `{(5,4),(4,5),(4,4),(0,1),(0,0)}`，WHEN 计算，THEN 依次返回 `{5,5,4,1,0}`；GIVEN `longest = 4` 且本事件 `current_after = 5`，THEN 返回 5（用旧 current 会错得 4）。
- **AC-F6（active_game_after_event）** GIVEN `{(none,NEW_GAME,1),({42,false},LOSS,43),({42,false},WIN,42),({42,true},NEUTRAL,7),(none,NEUTRAL,7)}`，WHEN 计算，THEN 依次返回 `{{1,false},{43,false},{42,true},{42,true},none}`；LOSS 使用事件的 N，不是旧局的 N。
- **AC-F7（games_lost）** GIVEN `{(10,7),(0,0),(5,0),(5,5)}`，WHEN 计算，THEN 依次返回 `{3,0,5,0}`；GIVEN 第 10 局进行中，THEN 仍返回 3（含活动局）；GIVEN 违反不变量的 `(7,10)`，THEN 返回 −3 且不 clamp。
- **AC-F8（win_rate）** GIVEN `{(0,0),(5,0),(5,5),(8,6),(10,7),(3,1)}`，WHEN 计算，THEN 依次返回 `{0.0, 0.0, 1.0（精确）, 0.75（精确）, ≈0.7, ≈1/3}`；无 NaN、无 ∞；GIVEN `(8,6)` 且实现使用整数除法，THEN 属错误，结果必须是 0.75。
- **AC-F9（stats_after_event）** GIVEN `S={1,0,0,0,{42,false}}`、`Won(42)`，THEN `{1,1,1,1,{42,true}}`；GIVEN `S={1,0,0,0,{42,false}}`、`Started(43)`，THEN `{2,0,0,0,{43,false}}`；GIVEN `S={2,2,2,2,{42,true}}`、`Started(43)`，THEN `{3,2,2,2,{43,false}}`；GIVEN `S={5,4,4,4,{42,false}}`、`Won(42)`，THEN `{5,5,5,5,{42,true}}`（longest 必须消费更新后的 current=5）；GIVEN 任意合法 S 与事件，WHEN 按 classify → played → won → current → longest → active 逐步计算，THEN 每步与组合公式逐字段一致。

### 不变量

- **AC-I1（单事件保持）** GIVEN 任意可达状态 S 与任意契约内事件 e，WHEN 计算 `stats_after_event(S,e)`，THEN 输出满足 `0 ≤ games_won ≤ games_played`、`0 ≤ current_streak ≤ games_won`、`0 ≤ longest_streak ≤ games_won`、`current_streak ≤ longest_streak`。
- **AC-I2（有界穷举）** GIVEN 从 S0 出发、事件字母表 `{Started(42), Won(42), Won(43), Restarted(42), Stuck(42)}` 的全部长度 ≤ 5 的序列（确定性穷举），WHEN 检查每个前缀，THEN 四条不变量成立、计数非负、`win_rate ∈ [0,1]`。
- **AC-I3（长连胜）** GIVEN 10 局全胜，THEN `{10,10,1.0,10,10}`；WHEN `Started(11)`，THEN `{11,10,10/11,10,10}`；WHEN `Started(12)`，THEN `{12,10,5/6,0,10}`；三条后续快照均满足不变量。

### 跨系统

- **AC-X1（重复 GameWon 幂等）** GIVEN `Started(42), Won(42)`，WHEN 再次投递 `Won(42)` 一次或十次，THEN `games_won`、`current_streak`、`longest_streak` 与 active 均不变。
- **AC-X2（迟到异局胜利）** GIVEN `Started(42), Started(43)`（42 记 loss，快照 `{2,0,0,0}`），WHEN 迟到的 `Won(42)`，THEN 快照仍为 `{2,0,0,0}`、active 仍 `{43,false}`；WHEN 随后 `Won(43)`，THEN `{2,1,0.5,1,1}`。
- **AC-X3（同 N 新局计数）** GIVEN `Started(42), Won(42)` 得到 `{1,1,1.0,1,1}`，WHEN `Started(42)`（`new_game(42)`），THEN `{2,1,0.5,1,1}`、active `{42,false}`、无 loss；WHEN 再次 `Won(42)`，THEN `{2,2,1.0,2,2}`（幂等按活动局实例，不按 N）。
- **AC-X4（同 N 不去重）** GIVEN `Started(42)`，WHEN 再次 `Started(42)`，THEN `{2,0,0.0,0,0}`、active `{42,false}`；第二次按 LOSS 处理，证明 Statistics 不按 N 去重。
- **AC-X5（State 恰好一次）** GIVEN Statistics 与真实 State（或遵守契约的测试替身）连接，WHEN 一次成功的 `new_game(N)` 返回，THEN Statistics 恰收到一次 `GameStarted{N}`，`games_played` 恰 +1；WHEN 一次进入 Won，THEN 恰收到一次 `GameWon{N}`，`games_won` 恰 +1，重复 `on_move_applied` 不产生第二次 `GameWon`。
- **AC-X6（单向调用）** GIVEN Statistics 处理任意事件，WHEN 用 State/Rules/Model 的 spy 观察，THEN Statistics 未调用它们的任何方法；只观察到 State → Statistics 的事件流（架构检查）。
- **AC-X7（局中重置）** GIVEN `Started(42), Won(42), Started(43)` 得到 `{2,1,0.5,1,1}`，WHEN `reset_stats()`，THEN S0；WHEN `Won(43)`，THEN 仍 S0；WHEN `Started(44)`，THEN `{1,0,0.0,0,0}`、无 loss。
- **AC-X8（重开/卡死语义）** GIVEN `Started(42)`，WHEN `Restarted(42)` 任意次，THEN 快照 `{1,0,0,0}`、active 不变；WHEN 随后 `Won(42)`，THEN 只计一胜。GIVEN 胜利后 `Restarted(42)`，THEN active 保持 `{42,true}`、重复 `Won(42)` 被忽略；WHEN `Started(43)`，THEN 无 loss。GIVEN `Started(42), Stuck(42)`，THEN 快照不变；WHEN 撤销后 `Won(42)`，THEN 计一胜；WHEN 直接 `Started(43)`，THEN 在 `Started(43)` 时记 loss。
- **AC-X9（事件顺序）** GIVEN 尚未 `Started` 时先收到 `Won(42)`，THEN 忽略、S0；WHEN 随后 `Started(42)`，THEN `{1,0,0.0,0,0}`。GIVEN `Started(42), Won(42), Started(43)` 得到 `{2,1,0.5,1,1}`，WHEN 迟到的 `Won(42)`，THEN 忽略；WHEN `Started(44)`，THEN 43 记 loss，快照 `{3,1,1/3,0,1}`。
- **AC-X10（读档 MVP 外）** GIVEN 任意状态，WHEN 通过未知入口投递 `Loaded(N)`，THEN 快照与 active 不变；MVP 不提供该入口。
- **AC-X11（同步可见）** GIVEN State 在状态更新后同步发事件，WHEN 处理器返回，THEN `get_stats()` 立即反映新计数，事件按发出顺序处理、不重排。

### 性能

- **AC-P1（事件处理）** GIVEN Release 构建、单线程、已预热、近 5 年主流桌面 x86-64（4 核 3.0 GHz 级），WHEN 连续投递 100,000 个混合事件（含 `reset_stats()`），THEN 总耗时 ≤ 1,000 ms（平均 ≤ 10 µs/事件），单次事件最大 ≤ 1 ms。
- **AC-P2（查询开销）** GIVEN 同一环境，WHEN 调用 `get_stats()` 100,000 次，THEN 总耗时 ≤ 500 ms（平均 ≤ 5 µs/次），且耗时与累计事件数无关。
- **AC-P3（O(1) 证据）** GIVEN 连续处理 1,000,000 个事件，WHEN 比较最开始 1,000 次与最后 1,000 次的平均单次耗时，THEN 比值 ≤ 2.0。
- **AC-P4（内存与结构）** GIVEN 预热后处理 1,000,000 个事件，WHEN 统计线程分配量，THEN 总分配 ≤ 1 MB；WHEN 反射检查内部状态，THEN 只有固定大小字段，不存在随事件数增长的集合（架构检查）。

若参考机未在 `project.yaml` 钉死，P1–P4 在 CI 上为 ADVISORY、在参考机上为 BLOCKING；报告必须记录硬件型号与构建配置。

### 不在本 GDD 验证范围（NOT ASSESSED）

- Toolbar & Menus：统计视图、胜率显示格式、重置确认对话框、玩家可见性——需 Toolbar GDD；本 GDD 只保证数据正确。
- Save/Resume Game：统计持久化、读档对账、跨会话活动局——Vertical Slice 另行设计。
- 「无 I/O」「不持有模型/Rules/State 引用」「无锁/无队列/无事件日志」：属于架构/依赖检查证据，不是黑盒行为断言。
- 并发安全：GDD 明确不承诺，不设并发行为 AC。
- 契约外非法 N：不设 AC，由 State 的校验保证。
- `INT32_MAX` 的真实事件序列：会话内不可达，只在 AC-F2 的纯函数层验证。
- 提示质量的「原版感」：主观体验，归 playtest，不由单元测试断言。

## Open Questions

| # | 问题 | 负责人 | 目标 |
|---|------|--------|------|
| 1 | （已解决 2026-10-24）Toolbar 行已补依赖 8；Toolbar GDD 编写时仍需把 Statistics 列为上游。 | systems-index / producer | Toolbar GDD |
| 2 | Vertical Slice 持久化：统计文件归谁、格式、`GameLoaded` 对账、跨会话活动局如何处理？ | Save/Resume / architecture | Vertical Slice |
| 3 | 局中重置后，当前局即使获胜也永远不计；是否符合原版/玩家预期？ | game-designer / 用户 | 首个可玩版本 playtest |
| 4 | （已解决 2026-10-24）Toolbar 采用 `win_percentage_exact` 整数半进位；标签/文案仍归 UX。 | Toolbar & Menus / UX | MVP UI |
| 5 | （已解决 2026-10-24）UI 选局同 N 路由为 `restart_deal()`，不重复计胜；直接 `new_game(N)` 仍计新局，但 UI 不暴露。 | game-designer / 用户 | 已完成 |
| 6 | 应用在未完成局时关闭：MVP 统计清零、不记 loss；VS 是否要记一次放弃？ | game-designer / 用户 | Vertical Slice |
| 7 | 概念「失败可无损重来」与「放弃拉低胜率」的意图张力：统计是否应完全不影响胜负感知？ | creative-director / 用户 | design review |
| 8 | State 的 exactly-once `GameStarted` / `GameWon` 需要实现与集成测试保证；State GDD 已澄清措辞，但尚无测试。 | Game State Machine / QA | 实现阶段 |
| 9 | MVP 会话内统计、重启清零，是否确认不做任何持久化？ | producer / 用户 | MVP 验收 |
| 10 | 若玩家在 `Stuck` 状态停留很久后直接关闭应用，MVP 不记 loss；VS 是否需要超时/关闭时判定？ | game-designer / 用户 | Vertical Slice |
