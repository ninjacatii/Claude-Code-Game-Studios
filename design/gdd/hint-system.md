# Hint System

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 原版规则，零改动

## Summary

Hint System 在玩家请求时，从 Rules & Move Validator 已枚举的合法移动中按五级阶梯选出一个有用建议，并用蓝色高亮指出源牌与目标；重复请求会循环候选，胜利或没有可用移动时不返回建议。它让卡关的玩家有一个可信的下一步，同时保持原版规则、操作与牌桌不变。

> **Quick reference** — Layer: `Feature` · Priority: `MVP` · Key deps: `Card & Deck Model`, `Rules & Move Validator`

## Overview

Hint System 在玩家请求提示时，从 Rules & Move Validator 已枚举的合法移动中挑选一个对当前局面有帮助的下一步，并指出源牌与目标位置供界面高亮。它只读模型、不执行移动、不改变规则；没有它游戏仍可正常游玩，但会失去原版辅助中缓解卡关的一项。玩家通过工具栏或快捷键主动触发，提示以高亮而非长文本呈现，主要服务于“原版规则，零改动”与“一秒读懂牌面”两条支柱。

## Detailed Design

### Core Rules

**1. 范围与所有权**

- 拥有：提示选择、候选排序、循环状态、无提示原因。
- 不拥有：移动合法性（Rules & Move Validator）、模型状态（Card & Deck Model）、输入绑定与状态文案（Toolbar & Menus）、高亮绘制（Table Rendering & Layout）。
- 只读模型；从不调用 `model.move`、`can_move` 或任何写入接口；从不执行移动。
- 提示只是建议；玩家仍通过拖拽或双击执行移动，执行时重新经过 Rules 判定。

**2. 输入与输出**

- 唯一入口：`request_hint(model) -> HintResult`；生命周期清理：`reset()`。
- `HintResult` 字段：
  - `has_hint: bool`
  - `move: Move | null`，其中 `Move = {source, destination, cards}`；`cards` 按源顺序自底向顶（与 Rules 的 MoveResult 契约一致）
  - `reason: NONE | GAME_WON | NO_USEFUL_MOVES`
  - `candidate_count: int`（过滤后的候选数）
  - `cycle_index: int`（`0..candidate_count−1`；无提示时为 `−1`）
- 无提示时：`has_hint = false`、`move = null`、`candidate_count = 0`、`cycle_index = −1`。
- 前置条件：调用方必须传入满足 Card & Deck Model 不变量的模型；Hint 不校验模型，也不尝试修复。

**3. 候选来源与过滤**

- 候选只来自 `Rules.legal_moves(model)`；Hint 不自造移动、不改写移动。
- 过滤掉「整列搬运」：源是桌面列、目标是空桌面列、`count = height(source)`。该移动只是把整列换到另一个空列，牌面等价、无净收益；Rules 会枚举它，但 Hint 不推荐。
- 其余合法移动全部保留，包括单张与序列。
- 过滤后的数量记为 `hint_candidate_count`；它可以小于 `legal_move_count`。

**4. 候选排序（阶梯优先级）**

排序键为 `(tier, enumeration_index)` 升序；`enumeration_index` 是该移动在 `legal_moves(model)` 返回列表中的 0 基下标。

| Tier | 名称 | 谓词 | 净效果 |
|---|---|---|---|
| 1 | 回收移动 | `destination.type = foundation` | 回收单元 +1；不可逆进度 |
| 2 | 净增空列 | 源桌面列、目标非空桌面列、`count = height(source)` | 空桌面列 +1 |
| 3 | 净增空自由单元 | 源自由单元、目标非空桌面列 | 空自由单元 +1 |
| 4 | 合并序列 | 源桌面列、目标非空桌面列、`count < height(source)` | 牌叠到已有牌上，并露出源列下一张 |
| 5 | 其他 | 其余未被过滤的候选（桌面列→自由单元、自由单元→空列、桌面列→空列的部分展开） | 净收益 ≤ 0 或混合 |
| — | 排除 | 源桌面列、目标空桌面列、`count = height(source)` | 纯列置换，无净收益 |

- Tier 1 优先于一切；同层严格按 `enumeration_index` 升序，因此等价自由单元目标自动取最低索引（与 Rules GDD 的「Hint 使用最低索引」一致）。
- Tier 5 是兜底：当没有更高层候选时，提示仍返回一个合法移动，而不是沉默。
- 排序必须确定：同一模型 + 同一候选列表 → 同一顺序。

**5. 循环候选**

- 内部状态：`H = (ranked_moves, cursor)`；`cursor` 是上一次返回的候选下标，初始为 `−1`。
- 两条移动相同的定义：`source` 相同、`destination` 相同、`cards` 的长度、顺序与内容都相同；不比较对象身份。
- `request_hint(model)` 流程（顺序固定）：
  1. 若 `Rules.is_won(model) = true`：`reset()`；返回 `NO_HINT(GAME_WON)`。
  2. 计算 `current = ranked_moves(model)`（过滤 + 排序）。
  3. 若 `current` 为空：`reset()`；返回 `NO_HINT(NO_USEFUL_MOVES)`。
  4. 若 `current` 与 `H.ranked_moves` 完全相同：`cursor = (H.cursor + 1) mod |current|`；否则 `cursor = 0`。
  5. `H = (current, cursor)`；返回 `current[cursor]`，`reason = NONE`。
- 首次请求：`H` 为空、候选列表不同 → `cursor = 0`，返回最佳候选。
- 只有一个候选时：每次请求都返回它（模 1 循环）。
- 候选列表变化（任何改变合法移动的牌局变化）：下一次请求从最佳候选重新开始。
- 候选列表不变但模型变化：循环继续。这种情况只可能出现在撤销/读档恢复到同一牌面时；**任何合法移动都会改变候选列表**——包括整列搬运，它会改变其他候选的 source/destination。Hint 不需要模型版本号。
- `reset()`：清空 `H`；下一次请求从最佳候选开始。调用方（Toolbar & Menus）在新局、重开、读档、撤销后调用 `reset()`；不调用也不会产生非法提示，只会从循环中间继续。

**6. 无提示情形**

- `GAME_WON`：已胜利；不返回移动。
- `NO_USEFUL_MOVES`：未胜利且过滤后候选为空；不返回移动。
- 在有效模型下，`NO_USEFUL_MOVES` 与 `is_stuck = 1` 重合：若存在整列搬运，则要么有空自由单元（任意桌面列顶牌 → 空自由单元是未被过滤的候选），要么自由单元全满（任意自由单元牌 → 空桌面列是未被过滤的候选），两种情况都保证 `hint_candidate_count > 0`。因此「未胜利 + 候选为空」当且仅当 `legal_move_count = 0`。保留独立 reason 是为了与 `GAME_WON` 区分，不是因为存在二者分离的局面。
- 调用方负责把 `reason` 转成玩家可见文案（例如「没有可用的移动」）；Hint 不生成文案。

**7. 确定性**

- 无随机数、无时间、无历史依赖；同一模型 + 同一 `H` → 同一 `HintResult`。
- 不读取 Undo History、不读取统计、不读取设置。
- 单线程；与其他系统一样串行调用。

**8. 只读与不执行**

- Hint 只读查询：牌堆类型、高度、顶牌、牌列表、`legal_moves`、`is_won`。
- Hint 不调用任何写入接口；除循环缓存 `ranked_moves` 外不保存牌局副本。
- 返回的移动可能在玩家执行前过期；玩家执行时由 Rules 重新判定，Hint 不保证过期移动仍合法。

### States and Transitions

Hint 没有牌局状态；只有循环缓存状态。

| 状态 | 进入 | 离开 | 说明 |
|---|---|---|---|
| Idle | 初始化；`reset()`；任何返回 `NO_HINT` 的请求 | 请求且候选非空 → Cycling | 无缓存候选，`cursor = −1` |
| Cycling | 请求且候选非空 | 请求且候选为空或已胜利 → Idle；`reset()` → Idle | 缓存 `ranked_moves` 与 `cursor` |
| Cycling（续） | 请求且候选列表与缓存完全相同 | — | `cursor` 前进一位，模 N |
| Cycling（重置） | 请求且候选列表与缓存不同 | — | `cursor = 0`，返回最佳候选 |

- 不存在「已完成」状态：循环无限回绕。
- 不存在跨牌局的持久状态：`reset()` 或候选列表变化都会回到最佳候选。

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|---|---|---|---|---|
| Card & Deck Model | 上游 | 只读牌堆/牌查询 | 无 | 模型拥有状态；Hint 只读 |
| Rules & Move Validator | 上游 | `legal_moves(model)`、`is_won(model)` | 选中的 `Move` | Rules 拥有合法性；Hint 拥有选择 |
| Toolbar & Menus | 下游 | `request_hint` / `reset` 调用 | `HintResult` | Toolbar 拥有触发与文案；Hint 拥有选择 |
| Table Rendering & Layout | 下游（经 Toolbar） | 高亮请求（源牌 + 目标） | 无 | Rendering 拥有绘制；Hint 拥有「标什么」 |
| Drag & Drop Interaction | 下游（玩家执行） | 无 | 无 | Interaction 拥有执行；开始拖拽时调用方清除高亮 |
| Undo History | 无 | 无 | 无 | Hint 不读取历史 |
| Game State Machine | 无直接接口 | 无 | 无 | 生命周期清理由 Toolbar 转达 |
| Statistics & Records | 无 | 无 | 无 | — |
| Settings / Accessibility | 无 | 无 | 无 | 提示无开关；若未来加开关，由 Settings 拥有 |

**接口契约：**

- `request_hint(model) -> HintResult`
- `reset() -> void`
- Hint 不调用模型写入，不调用 Rules 的 `can_move` / `auto_move_destination`。
- Toolbar 在以下时机调用 `reset()`：新局、重开本局、成功读档、撤销之后。
- Toolbar 在拖拽进行中不得请求或显示提示；Hint 不感知拖拽。
- Toolbar 把 `HintResult.move` 转成高亮请求：源牌为 `move.cards` 的全部卡牌，目标为 `move.destination` 的落位槽或顶牌；具体绘制归 Table Rendering。
- 高亮必须在以下任一情况清除：下一次提示、牌局变化、开始拖拽、撤销、新局/重开/读档。

**与 Rules 枚举顺序的关系：**

- `legal_moves` 的固定顺序是 Hint 的唯一平局裁决；Hint 不重新枚举、不改写 Rules 的排序。
- Rules GDD 的「Hint 使用最低索引」在本 GDD 中体现为 Tier 内按 `enumeration_index` 升序。

**Provisional assumptions（下游未设计，先假定）：**

- Toolbar & Menus 提供触发入口（按钮/快捷键）与无提示文案；具体键位与文案由 Toolbar GDD 决定。
- Table Rendering & Layout 接受「源牌列表 + 目标牌堆」的高亮请求，并能绘制蓝色描边。
- 撤销、新局、重开与读档由 Toolbar 统一协调，因此 `reset()` 的调用不新增 Hint 的上游依赖。

## Formulas

> **定义域。** 以下公式只接受 `Rules.legal_moves(model)` 返回的移动与满足不变量的模型。`src_type` / `dst_type` 是牌堆句柄上的类型标签（`tableau | free_cell | foundation`）；若 Card & Deck Model 不提供类型标签，实现必须先补该只读查询（见 Dependencies 的 provisional assumption）。

### The hint_tier formula

The hint_tier formula is defined as:

`hint_tier(move, model) = 0 if EXCLUDE(move, model) else 1 if DST_FOUNDATION(move) else 2 if GAIN_COLUMN(move, model) else 3 if GAIN_FREE_CELL(move, model) else 4 if CONSOLIDATE(move, model) else 5`

子谓词按从左到右求值，**先命中先返回**：

- `EXCLUDE(m, x) = src_type = tableau ∧ dst_type = tableau ∧ height(destination) = 0 ∧ count = height(source)`
- `DST_FOUNDATION(m) = dst_type = foundation`
- `GAIN_COLUMN(m, x) = src_type = tableau ∧ dst_type = tableau ∧ height(destination) > 0 ∧ count = height(source)`
- `GAIN_FREE_CELL(m, x) = src_type = free_cell ∧ dst_type = tableau ∧ height(destination) > 0 ∧ count = 1`
- `CONSOLIDATE(m, x) = src_type = tableau ∧ dst_type = tableau ∧ height(destination) > 0 ∧ count < height(source)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| move | Move | `legal_moves(model)` 的元素 | `Rules.legal_moves(model)` | 待分层候选 `{source, destination, cards}`；不含 `legal` / `reason` |
| model | model | 有效模型（不变量成立） | Card & Deck Model | 只读牌局；公式不修改它 |
| src_type | enum | {tableau, free_cell} | 模型牌堆类型 | 源牌堆类型；`foundation` 源不在合法枚举内 |
| dst_type | enum | {tableau, free_cell, foundation} | 模型牌堆类型 | 目标牌堆类型 |
| count | int | 1–13 | `move.cards` | 移动单元张数 `= \|cards\|` |
| height(source) | int | 1–52 | 模型只读查询 | 源桌面列张数；仅当 `src_type = tableau` 时有意义 |
| height(destination) | int | 0–52 | 模型只读查询 | 目标桌面列张数；仅当 `dst_type = tableau` 时有意义 |
| hint_tier | int | 0–5 | 本公式 | 层级；0 = 被排除，1–5 = 五个优先级层 |

**Output Range:** {0, 1, 2, 3, 4, 5}。六个分支互斥且穷尽：桌面列→桌面列按「目标空 + `count = height`（0）／目标空 + `count < height`（5）／目标非空 + `count = height`（2）／目标非空 + `count < height`（4）」划分；自由单元源只可能是回收单元（1）、非空桌面列（3）或空桌面列（5）；桌面列→自由单元落入（5）。`count > height(source)` 不可达（`COUNT_INVALID` 不会被枚举）。定义域限于 `legal_moves(model)` 的输出；`src_type = foundation` 或非法移动不在定义域内，实现应断言，**不得静默返回 5**——否则枚举回归会被掩盖成一条可用提示。

**Example:** 部分 fixture（只列出用于说明分层的移动，不是完整 `legal_moves`）：局面 T1 = `[8♥, 7♠]`（自底向顶，高 2）、T2 顶牌 `9♣`、T3 顶牌 `8♦`、T6/T7 为空列、FC1 空、FC4 持有 `A♠`、♠ 回收单元为空：
- `T1→T2`：count 2 = height → **2**；
- `T1→T3`：count 1 < height → **4**；
- `T1→FC1` → **5**；`T1→T6`（空列，count 1 < height）→ **5**；
- `T1→T7`（空列，count 2 = height）→ **0**（排除）；
- `FC4→♠回收单元` → **1**（目标类型优先于源类型）。

### The hint_ranked_moves formula

The hint_ranked_moves formula is defined as:

`hint_ranked_moves(model) = sort_ascending([m for m ∈ legal_moves(model) if hint_tier(m, model) > 0], key = (hint_tier(m, model), enumeration_index(m)))`

`enumeration_index(m)` = `m` 在**未过滤**的 `legal_moves(model)` 返回列表中的 0 基下标。Rules §5 已明确枚举为「目标外层、长度内层」——即对每个目标，按 `count = 1 .. max_movable_length` 递增尝试。

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| model | model | 有效模型 | Card & Deck Model | 只读牌局 |
| legal_moves(model) | Move[] | 长度 0–768 | Rules §5 | 固定顺序、去重、按格枚举自由单元的合法移动 |
| hint_tier(m, model) | int | 0–5 | 本 GDD | 层级；0 被过滤 |
| enumeration_index(m) | int | 0–767 | Rules 枚举顺序 | **未过滤**列表中的 0 基下标；只在单次 `legal_moves` 调用内有效 |
| hint_ranked_moves | Move[] | 长度 0–768 | 本公式 | 过滤并按 `(tier, index)` 升序的候选列表 |

**Output Range:** 长度 0–768。排序键二元组不会重复（枚举下标唯一），因此顺序完全确定；同一模型 → 同一列表。实现可先按 tier 做稳定划分，再保持枚举顺序拼接——这与按 `(tier, index)` 排序等价。**不得对过滤后的列表重新编号**：重编号是单调变换、顺序不变，但会让下标失去与 Rules 的对应关系。

**Example:** 承上（**部分 fixture**：只列出用于说明排序的 6 条；完整枚举还会包含 T2/T3 顶牌 → FC1 等移动；示例的 idx 顺序只用于说明分层，不按 Rules §5 的枚举顺序，实际 index 以 Rules 为准），`legal_moves` 按枚举顺序返回 6 条：idx0 `T1→T2`（tier 2）、idx1 `T1→T3`（4）、idx2 `T1→FC1`（5）、idx3 `T1→T6`（5）、idx4 `T1→T7`（0）、idx5 `FC4→♠FD`（1）。排序结果：`[idx5 (1), idx0 (2), idx1 (4), idx2 (5), idx3 (5)]`——剔除 idx4，Tier 5 内按枚举下标保持 `FC1` 在 `T6` 前。

### The hint_candidate_count formula

The hint_candidate_count formula is defined as:

`hint_candidate_count(model) = |hint_ranked_moves(model)| = Σ over m ∈ legal_moves(model) of [hint_tier(m, model) > 0]`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| model | model | 有效模型 | Card & Deck Model | 只读牌局 |
| legal_moves(model) | Move[] | 长度 0–768 | Rules §5 | 合法移动全集 |
| hint_tier(m, model) | int | 0–5 | 本 GDD | 仅用于计数指示函数 |
| hint_candidate_count | int | 0–768 | 本公式 | 过滤后候选数（Hint 自有值，不复用 `legal_move_count`） |

**Output Range:** 0–768。上界是**继承**的：`hint_candidate_count ≤ legal_move_count(model) ≤ 768`（结构上限由 Rules 拥有，并已在 registry 登记为 `legal_move_count` 的 `output_range: [0, 768]`；Hint 不重新推导它）。它可以严格小于 `legal_move_count`，因为每条被排除的纯列置换都是合法移动但不是候选。排除条数的结构上界：一次纯列置换由 `(源列, 目标空列)` 唯一确定，设非空桌面列 `s`、空桌面列 `e`，则 `s + e ≤ tableau_count = 8`，故 `exclusions ≤ s·e ≤ floor(8²/4) = 16`（还须通过超级移动容量，实际更少），于是 `hint_candidate_count ≥ legal_move_count − 16`。**0–768 内可达的精确最大值无法从依赖契约推导**；768 只是继承的安全上界。

**Example:** 上例（部分 fixture）`legal_move_count = 6`、排除 1 条（idx4）→ `hint_candidate_count = 5`。在完整牌局中，`hint_candidate_count = 0` 只在 `legal_move_count = 0` 时出现（证明见 Detailed Design §6）。

### The hint_cycle_index formula

The hint_cycle_index formula is defined as:

`hint_cycle_index(previous_cursor, candidate_count, same_candidate_list) = −1 if candidate_count = 0; 0 if same_candidate_list = 0; (previous_cursor + 1) mod candidate_count otherwise`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| previous_cursor | int | −1–767 | Hint 内部状态 `H.cursor` | 上一次返回的候选下标；−1 = 无缓存 |
| candidate_count | int | 0–768 | `hint_candidate_count(model)` | 本次候选数 |
| same_candidate_list | int | {0, 1} | 本 GDD §5 的逐元素比较 | 1 = `current` 与 `H.ranked_moves` 长度相同且逐条满足同源、同目标、`cards` 长度/顺序/内容相同 |
| hint_cycle_index | int | −1–767 | 本公式 | 本次返回的候选下标 |

**Output Range:** −1–767（`candidate_count − 1` 的最大值）。三支按书写顺序求值：`candidate_count = 0` 优先（此时 `H` 已被 `reset()`，−1 表示无提示）；否则列表变化或首次请求（`H` 为空 ⇒ `same_candidate_list = 0`）返回 0；否则前进一位并回绕。`same_candidate_list = 1 ⇒ candidate_count ≥ 1` 是缓存不变量（空列表只能与空列表相等，已被第一支拦下），因此取模除数恒为正、不存在除零。

**Example:** `candidate_count = 5`：(a) 首次请求 → `same = 0` → 0；(b) `cursor = 0, same = 1` → 1；(c) `cursor = 4, same = 1` → `5 mod 5 = 0`（回绕）；(d) 玩家执行了一次纯列置换（该移动本就不在候选内，但它改变其他候选的 source/destination）→ 候选列表变化 → `same = 0`，下一次从 0 开始；(e) 任何改变候选内容的移动 → `same = 0` → 0；(f) 未胜利且 `legal_move_count = 0`（真卡死）→ `candidate_count = 0` → −1。

### The hint_available formula

The hint_available formula is defined as:

`hint_available(model) = 1 if game_completion(model) = 0 and hint_candidate_count(model) > 0, else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| model | model | 有效模型 | Card & Deck Model | 只读牌局 |
| game_completion | int | {0, 1} | registry：`design/gdd/card-deck-model.md` | 1 = 52 张全在回收单元；`Rules.is_won(model)` 是它的包装 |
| hint_candidate_count | int | 0–768 | 本 GDD | 过滤后候选数 |
| hint_available | int | {0, 1} | 本公式 | 1 = 本次请求能给出提示 |

**Output Range:** {0, 1}。胜利守卫**必需**且优先：已胜利时 `hint_candidate_count` 通常已为 0，但仍显式排除，与 `HintResult.reason` 的判定顺序（§5 第 1 步 `GAME_WON` 先于第 3 步 `NO_USEFUL_MOVES`）保持一致。等价关系：`hint_available = 1 ⟺ HintResult.has_hint = true`。实现应让 `HintResult.has_hint` 由本公式派生，而不是维护第二份判定。

**Example:** 未胜利、`hint_candidate_count = 5` → 1；未胜利、`legal_move_count = 0` → `hint_candidate_count = 0` → 0（此时 `is_stuck = 1`，Hint 报 `NO_USEFUL_MOVES`）；已胜利 → 0。

### 常量与可调性

本系统零可调参数：Tier 顺序由用户批准的阶梯固定，没有权重或阈值需要平衡。`hint_candidate_count` 的上界继承 `legal_move_count` 的 768；`tableau_count = 8` 仅用于排除条数的上界论证，不新建常量。若未来要调整提示质量，应新增独立的调参入口，而不是修改 Tier 顺序——Tier 顺序是“原版规则，零改动”的一部分。

## Edge Cases

- **If 已胜利（`game_completion = 1`）**：返回 `NO_HINT(GAME_WON)`，`reset()`，`candidate_count = 0`，`cycle_index = −1`。胜利判定必须先于候选计算，即使某实现下列表非空。
- **If 未胜利且 `legal_move_count = 0`**：返回 `NO_HINT(NO_USEFUL_MOVES)`，`reset()`。这是真正的卡死局面；在有效模型下 `NO_USEFUL_MOVES` 与 `is_stuck = 1` 重合。
- **If 未胜利、`legal_move_count > 0`，但所有合法移动都是整列搬运**：该情形不可达。只要存在整列搬运，就必然存在至少一个未被过滤的自由单元/桌面列候选（证明见 Detailed Design §6）；实现遇到它应视为 Rules 枚举错误，而不是返回 `NO_USEFUL_MOVES`。
- **If 候选数 = 1**：每次请求都返回同一条，`cycle_index` 恒为 0；模 1 回绕不是特例。
- **If 候选数 = 2**：返回顺序为 0 → 1 → 0 → 1；候选列表不变时持续循环。
- **If 某个候选在牌局变化后不再合法**：候选列表与缓存不同 → `cursor = 0`，返回新的最佳候选；不得 clamp 到旧下标。
- **If 玩家执行了一次整列搬运（该移动从不在候选内）**：候选列表仍会变化（其他候选的 source/destination 改变）→ `cursor = 0`，下一次从最佳候选开始；不得误以为「不在候选内」就等于「列表不变」。
- **If 玩家执行了候选中的移动**：该移动不再合法 → 候选列表变化 → 下一次提示从最佳候选开始。
- **If 候选列表变为空而 `H` 非空**：必须先 `reset()` 再返回，`cursor` 不得残留；否则下一次请求可能误入「列表相同」分支，从错误的起点继续。
- **If 撤销/读档后调用方没有调用 `reset()`**：下一次请求仍按候选列表比较自动校正；若列表不同则从 0 开始，若恰好相同则继续循环。这是容忍，不是错误；Toolbar 仍应调用 `reset()`。
- **If 模型变化但候选列表完全相同**：循环继续。这在有效模型下只可能出现在撤销/读档恢复到同一牌面时；任何合法移动都会改变候选列表（包括整列搬运）。
- **If 提示返回后、玩家执行前牌局发生变化**：该移动可能已过期；调用方必须清除高亮，玩家执行时由 Rules 重新判定。Hint 不保证过期移动仍合法。
- **If 拖拽进行中请求提示**：调用方不得请求或显示；Hint 不感知拖拽。若仍被调用，Hint 正常返回，但 Toolbar 必须丢弃或在拖拽结束后再显示。
- **If 牌局不在 Playing 状态（发牌中、结算中）**：调用方不得请求；Hint 只检查胜利，不读取 Game State Machine。对发牌中的空模型调用属于违反前置条件。
- **If 模型无效**：不在定义域内；实现应断言，**不得返回 `INVALID_MODEL`，不得修复**。调用方必须先由 Card & Deck Model 校验。
- **If 同一移动单元有多个等价空自由单元目标**：自由单元目标恒为 `count = 1`，Rules 按索引升序枚举，Hint 按 `enumeration_index` 升序 → 自动取最低索引格。Rules GDD 的「Hint 使用最低索引」由此满足；目标/长度的嵌套歧义不影响这一项。
- **If 目标为空桌面列且有多个 `count < height` 的合法长度**：返回哪一条取决于 Rules 的枚举嵌套。当前假定「目标外层、长度内层」；若 Rules 改为长度外层，提示可能返回不同的 Tier 5 候选，必须重新核对（见 Dependencies / Open Questions）。
- **If 存在多个 Tier 1 候选**：按 `enumeration_index` 升序取第一条，不按等级或花色再做次级排序。
- **If 存在 Tier 1 候选但不安全（例如过早把 2 送上回收单元）**：仍返回它。本系统不实现「安全自动归位」判断；这是原版保真与提示质量之间的已知取舍（见 Open Questions）。
- **If 只剩 Tier 5 候选**：返回枚举顺序中的第一条；提示仍然有效，只是价值较低。
- **If 桌面列高度为 1 的牌被移到自由单元**：归入 Tier 5（空列 +1、自由单元 −1，净收益 0），不归入 Tier 2。
- **If 自由单元中的牌被移到空桌面列**：归入 Tier 5（自由单元 +1、空列 −1，净收益 0），不归入 Tier 3。
- **If 提示的移动是多张序列**：`move.cards` 必须包含全部卡牌且按源顺序自底向顶；高亮必须覆盖整段序列，不能只标顶牌，否则玩家会误读移动范围。
- **If 同一帧内连续请求两次**：按顺序处理，`cursor` 前进两次；Hint 不做防抖，调用方若需要节流应自行处理。
- **If 候选被过滤后下标不连续**：`cycle_index` 指向**过滤后**的 `hint_ranked_moves`，`enumeration_index` 指向**未过滤**的 `legal_moves`；不得混用或对过滤后列表重新编号。
- **If 存档/读档**：Hint 的循环状态不写入存档，读档后从 Idle 开始；牌局状态由 Save/Resume 负责。
- **If 玩家在提示后立即获胜**：下一次请求返回 `GAME_WON` 并清空循环；不需要等待 `reset()`。
- **If 玩家撤销了获胜的一步**：下一次请求重新计算；若候选列表变化则从最佳候选开始，若相同则继续循环；Toolbar 应调用 `reset()`。

## Dependencies

**上游依赖：**

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Card & Deck Model | Hard | 只读牌堆查询：类型、高度、顶牌、牌列表 | 模型拥有状态；Hint 从不写入 |
| Rules & Move Validator | Hard | `legal_moves(model)`、`is_won(model)` | Rules 拥有合法性；Hint 拥有选择 |

**下游依赖（依赖本系统的系统）：**

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Toolbar & Menus | Hard（玩家可达性） | `request_hint` / `reset` | Toolbar 拥有触发、文案与高亮转发；无 Toolbar 则提示无法被玩家触发 |
| Table Rendering & Layout | Soft（经 Toolbar） | 高亮请求 | Rendering 拥有绘制；Hint 只决定「标什么」 |
| Drag & Drop Interaction | None | — | 玩家执行提示的移动；Hint 不调用 Interaction |
| Undo History | None | — | Hint 不读取历史；撤销后由 Toolbar 调用 `reset()` |
| Game State Machine | None（经 Toolbar 转达） | — | 生命周期清理由 Toolbar 协调 |
| Save/Resume Game | None | — | Hint 的循环状态不写入存档 |
| Settings | None | — | 提示当前无开关 |
| Statistics & Records | None | — | 不记录提示使用 |
| Readability & Accessibility | Soft | 高亮可见性、未来键盘触发 | Accessibility 拥有可访问表示 |

**接口契约：**

- **读**：`legal_moves(model)` 返回的 `Move[]` 元素形如 `{source, destination, cards}`，不含 `legal` / `reason`；`cards` 按源顺序自底向顶。
- **读**：`is_won(model)` 是 `game_completion(model) = 1` 的包装；Hint 在公式中使用 registry 已登记的 `game_completion`。
- **写**：Hint 不写模型、不写 Rules、不写任何下游系统。
- **调用**：`request_hint(model) -> HintResult`；`reset() -> void`。
- **调用方义务**：Toolbar 在新局、重开、成功读档、撤销后调用 `reset()`；在拖拽进行中不请求或显示提示；在牌局变化后清除旧高亮。
- **不保证**：提示返回后到玩家执行之间牌局不变；过期移动由 Rules 重新判定。

**双向一致性：**

- Card & Deck Model GDD 将 Hint 列为 Soft 下游（「无提示仍可玩」），本 GDD 将模型列为 Hard 上游（「无模型无法工作」）；方向不同，强度不同，二者一致。
- Rules GDD 将 Hint 列为 Hard 下游，并将「Hint 使用最低索引」写入其 Edge Cases；本 GDD 的 Tier 内 `enumeration_index` 升序满足该约束。
- Toolbar & Menus 尚无 GDD；其 GDD 编写时必须把 Hint 列为上游依赖，并引用本节接口。发现单向依赖时在 Toolbar GDD 或此处补齐。

**Provisional assumptions（依赖未显式提供，先假定）：**

1. **牌堆句柄带类型标签。** `move.source` / `move.destination` 能回答 `type ∈ {tableau, free_cell, foundation}` 与稳定索引。若 Card & Deck Model 只给索引，必须补一条只读查询 `pile_type(pile)`；否则 `hint_tier` 无法实现。
2. **Rules §5 已明确「目标外层、长度内层」。** 即对每个目标，按 `count = 1 .. max_movable_length` 递增尝试。Hint 不再保留该歧义。
3. **`enumeration_index` 只在单次 `legal_moves` 调用内有效。** 任何改变合法移动集合的牌局变化都会重排下标；Hint 的循环比较使用移动内容，不跨调用缓存下标。
4. **`legal_moves` 不保证五类移动都出现。** 具体局面可能只有其中几类；Tier 5 兜底保证稀疏局面仍有提示。
5. **`Move` 与 `MoveResult` 形状不同。** `legal_moves` 的元素没有 `legal` / `reason`；Hint 不读取不存在的字段。
6. **Toolbar 是唯一调用方。** 它负责触发、文案、高亮转发与 `reset()`；其他系统不直接调用 Hint。
7. **Hint 状态是瞬态的。** 循环缓存不序列化、不跨会话保存。

**Registry references：** `game_completion`、`legal_move_count`、`tableau_count` 的 `referenced_by` 应追加 `design/gdd/hint-system.md`；新增 `hint_tier`、`hint_ranked_moves`、`hint_candidate_count`、`hint_cycle_index`、`hint_available` 五条 formula 记录（Phase 5 更新）。

## Visual/Audio Requirements

> **前提：`design/art/art-bible.md` 不存在。** 以下色值、厚度与圆角是从 `design/gdd/game-concept.md` 的 Visual Identity Anchor（绿桌 `#008000`、白卡面、红/黑花色、蓝色高亮；动效 <150ms）推导的**暂定值**，art bible 建立后必须逐项复核。

**视觉判据**：提示高亮是临时标注，不是牌桌装饰。它只回答“搬哪几张、放到哪”，不改变牌面、不引入第二种高亮色、不在无提示时出现。

**高亮对象**

- **源（moving unit）**：`HintResult.move.cards` 的全部卡牌，覆盖整段序列而不是只标顶牌。绘制为**一个整体轮廓**（所有卡牌矩形的并集），不逐张画框——逐张画框会在 22–30px 的同列级联中把横线画到相邻牌面上。
- **目标（destination）**：
  - 空自由单元：该格空槽矩形；
  - 空桌面列：该列首槽 `card_rect(pile, −1)`（`tableau_y(0)`），不是整列命中区；
  - 空回收单元：该回收单元空槽；
  - 非空回收单元：该回收单元顶牌；
  - 非空桌面列：该列顶牌。
- 不标源列其他牌、目标列其余牌、工具栏按钮。

**视觉处理**

- **颜色**：`#000080` 深蓝。对白卡面 16.0:1，对绿桌 `#008000` 3.12:1（满足非文本 UI 的 3:1）。**不采用纯 `#0000FF` 作单色**：对绿桌仅 1.67:1，在空槽目标上几乎消失；若要鲜蓝必须配 1px 白色外键线。
- **厚度**：3 逻辑像素。外缘距高亮矩形边 1px，即色带占 1–4px（保留卡面 1px 黑边 / 空槽白色槽线）。0.8 缩放 → 2.4 设备像素（下限 2px）；3.0 缩放 → 9 设备像素。8px 圆角与卡面一致。
- **内缩而非外扩**：源列与目标列的牌下方总有一张牌，级联仅 22–30px；顶牌角标从 8px 开始，被覆盖牌紧凑索引从 5px 开始；外扩 3px 的顶边会正压在下方牌的等级字形上。内缩到 1–4px 落在紧凑索引之前（5px），rank/suit 不受影响，也不会画到任何相邻牌上。这是与原版“外框观感”的已知取舍（原版行为未查到公开资料）。
- **不做**：tint、背景变暗、抬升、阴影、发光、脉冲、描边动画、光标变化、半透明填充块。不改变其他牌与卡面的绘制。
- 空槽目标上，原有白色槽线保留在蓝带外侧，形成“白槽线 + 深蓝内环”的双边缘——这是免费的非颜色线索，也是 0.8 下的第二道边缘。
- **非颜色线索**：蓝色在红绿色盲下仍可区分；卡面上是 16:1 明度边缘，空槽上是 3.12:1 明度边缘加原有白线。默认不加额外键线；无障碍模式若要求，改为 4px + 1px 白键线（由 Readability & Accessibility 定义）。

**绘制顺序与重绘**

- 提示层在牌堆层与空槽层之上、拖拽层之下、工具栏/菜单/覆盖层之下。拖动开始即清除，此顺序是防御性的，保证残影不盖住拖拽中的牌或 UI。
- 提示层对命中测试透明，不参与 `hit_test`。
- 只在“设置/清除高亮、模型变化、窗口缩放、状态变化”时重绘；无动画，故待机不重绘。
- 与卡面共用同一 `card_rect` 与 `scale/offset`；缩放后按新矩形重算，不缓存旧矩形。
- **不得使用运行时线条图元**（`draw_line` / 未填充 `draw_rect`；4.7 下线宽会变细且无羽化），使用带抗锯齿的填充环：9-slice 环贴图（1x/2x/3x 或矢量源）或等效的带 AA 填充样式。这是 Table Rendering GDD 现有禁令的延续。

**持续与清除**

- **无自动超时**：保留到下列任一事件。静态标注，玩家读多张序列需要时间；自动消失等于新增计时行为。
- 下一次提示：有结果则同帧替换（无交叉淡入）；`has_hint = false`（`GAME_WON` / `NO_USEFUL_MOVES`）则清除。
- 牌局变化：任何成功移动（拖拽落位、双击）、撤销、新局、重开本局、读档 → 立即清除，由 Toolbar 在模型变化事件上统一清除。
- 开始拖拽：指针越过 4px 阈值、进入 `Dragging` 的同一帧清除；只按下不拖拽不清除。
- 状态变化：`Won` / `Stuck` / `NoGame` → 清除。
- 窗口缩放、焦点变化、暂停：不清除；缩放后按新几何重绘。
- 拖拽进行中请求提示：调用方不得显示；若已收到结果则丢弃。

**动效 — 不做任何动画**

- 提示在请求的同帧出现，清除时同帧消失。
- 理由：Visual Identity Anchor 只保留发牌/落位两类动效，提示不在其中；加淡入即第三类，与“手感和延迟优先”冲突，连续按提示时动画还会互相打断。

**音频 — 无声音**

- 提示不播放任何音效。音频是可选/Minimal 范围；原版以视觉为主；提示可连续触发，叠音会变成噪音。音频所有权在 Audio & SFX，本系统不新增音频事件。
- 不得由重绘路径触发音效（缩放/模型变化也会重绘）。

**给 Table Rendering & Layout 的接口**

- `set_hint_highlight(source: {pile, index}[] | null, destination: {pile, index} | null)`；`clear_hint()`。
- `source`：`move.cards` 对应的源牌堆与索引。移动单元恒为源列顶部连续段，索引范围 = `height(source) − |cards| .. height(source) − 1`，由 Rendering 从模型推导或由 Toolbar 传入。
- `destination`：空桌面列传 `index = −1`（对应 `card_rect(pile, −1)`）；空顶行槽传 `0`；非空牌堆传 `height − 1`。
- Rendering 只把引用映射到 `card_rect` 并绘制；不调 Rules、不判断提示是否仍合法。若源列顶部段与传入 `cards` 已不一致，跳过绘制（防御性，不改变 Hint 行为）。
- 高亮不修改模型、不改变卡牌绘制、不影响命中测试。

**禁止添加**

- 第二种高亮色、渐变描边、外发光、脉冲/呼吸、循环闪烁、粒子、屏幕变暗、其他牌降饱和、卡牌抬升/缩放、光标变化、提示文字气泡（文案归 Toolbar）、音效。
- 不为提示新增 UI 屏幕、面板或按钮；触发入口归 Toolbar。

**资产清单**

- `ui_hint_ring_9slice_1x.png` / `_2x` / `_3x`（或矢量源 `ui_hint_ring_9slice.svg`）：3px 环带 + 8px 圆角，白色，运行时调制为 `#000080`；随卡面图集按 `scale × display_scale` 选档。
- 无音频资产。

**视觉验收**

1. 1024×576（scale 0.8）：环带 ≥2 设备像素；与无提示截图对比，所有卡牌 8px 角标区逐像素不变（高亮只改 1–4px 边带）。
2. 1280×720：环带中心取样为 `#000080`（±抗锯齿）；环带外无蓝色像素。
3. 源列与目标列相邻时，两高亮框之间的 12px 绿桌间隙完整可见、不接触。
4. 高亮后静置 1 秒，两张截图逐像素相同（无脉冲/闪烁/呼吸）。
5. 提示 → 开始拖拽：拖拽层出现的同一帧高亮消失，不重叠。
6. 连续两次提示：旧框与新框同帧切换，无交叉淡入。
7. 灰度化截图：环带在绿桌与白卡上仍可辨（明度差，不只靠色相）。
8. 提示不触发任何音频事件；待机无逐帧重绘。

**暂定值（art bible 建立后复核）**

| 项 | 暂定值 | 依据 |
|---|---|---|
| 高亮色 | `#000080` | 原版 Windows 选择色；对白 16.0:1、对绿 3.12:1 |
| 环带厚度 | 3 逻辑 px | 0.8 → 2.4 设备 px；3.0 → 9 设备 px |
| 环带位置 | 内缩 1–4px | 避开 8px 角标；不压相邻牌 |
| 圆角 | 8px | 与卡面一致 |
| 动画 | 无 | Visual Identity Anchor 仅三类动效 |
| 音频 | 无 | 音频可选 / Minimal |
| 持续时间 | 直到清除，无超时 | 静态标注 |
| 无障碍增强 | 4px + 1px 白键线（仅模式内） | 备用，不进默认牌桌 |

**跨文档备注**：Table Rendering GDD 的「禁止添加」目前明确禁止「合法落点高亮、选中描边/发光」；本系统需要一条**窄例外**（仅提示环、9-slice 填充环、无运行时线条）。该例外已获用户批准，具体修订在 Table Rendering GDD 中单独进行。

## Game Feel

- **安静地指一下，不是系统通知。** 提示不抢焦点、不改变光标、不弹窗、不发声；玩家可以完全忽略它继续思考。
- **零延迟。** 提示在请求的同帧出现，清除时同帧消失；不新增输入延迟，不做淡入淡出。连续按提示时只切换高亮，不打断玩家的读牌节奏。
- **静态。** 静置 1 秒画面逐像素不变；待机不逐帧重绘。没有脉冲、呼吸、闪烁或循环动画。
- **一眼可读。** 0.8 缩放下环带仍有 ≥2 设备像素；序列整体高亮而不是只标顶牌，玩家能立刻看出“搬哪几张、放到哪”。
- **不遮挡。** 内缩环带落在卡面 8px 角标边距内，不压等级/花色；源列与目标列相邻时，两道高亮之间的 12px 绿桌间隙保持可见。
- **计算不可感知。** 最坏情形（候选数接近 768）下 `request_hint` 仍在 1ms 预算内；不掉帧、不卡输入。
- **拖拽优先。** 开始拖拽的同一帧清除高亮，提示层绝不与拖拽层重叠。
- **无提示时保持安静。** `NO_USEFUL_MOVES` / `GAME_WON` 不震动、不闪红、不发声；只由 Toolbar 显示一行文案或不显示。
- **与混合参考对比。** 不闪烁、不脉冲、不自动消失。若未来参考机截图证明混合参考（Windows 7/XP 功能基线）提示有闪烁或定时消失，再单独评估（见 Open Questions）。

## UI Requirements

- **触发入口**：Toolbar & Menus 提供「提示」按钮与键盘快捷键；具体键位由 Toolbar GDD 决定，本 GDD 不钉死。Hint 不创建自己的按钮、面板或屏幕。
- **无提示文案**：Toolbar 根据 `reason` 映射。建议：`NO_USEFUL_MOVES` → 「没有可用的移动」；`GAME_WON` → 不显示提示（胜利界面已接管）。Hint 不生成文案、不硬编码字符串。
- **高亮层**：由 Table Rendering & Layout 绘制，位于牌堆层与空槽层之上、拖拽层与工具栏/覆盖层之下；对命中测试透明，不参与 `hit_test`。
- **清除时机**：下一次提示、牌局变化（成功移动/撤销/新局/重开/读档）、开始拖拽、`Won` / `Stuck` / `NoGame`。
- **焦点**：快捷键在牌桌有焦点时可用；提示不抢焦点、不改变光标。
- **无障碍**：键盘可达；高对比/色盲模式在 Full Vision 由 Readability & Accessibility 提供；默认提示不依赖单一色相（明度差 + 空槽白色槽线双通道）。
- **本地化**：文案归 Toolbar 的字符串表；Hint 只传 `reason` 枚举。
- **不做**：提示文字气泡、指向箭头、屏幕变暗、其他牌降饱和、提示音、提示历史面板。

## Cross-References

| 文档 | 章节 | 关系 |
|------|------|------|
| `design/gdd/card-deck-model.md` | Interactions / Dependencies | 只读牌堆查询与 `Move` 形状；Hint 不写模型 |
| `design/gdd/rules-move-validator.md` | Core Rules §3/§5/§7、Formulas、Edge Cases | `legal_moves` 固定顺序、去重、按格枚举；`is_won`；「Hint 使用最低索引」 |
| `design/gdd/game-state-machine.md` | States and Transitions | 生命周期（新局/重开/读档）由 Toolbar 转达 `reset()`；无直接接口 |
| `design/gdd/toolbar-menus.md`（已完成） | UI | 触发、文案、`reset()` 时机、高亮转发 |
| `design/gdd/table-rendering-layout.md` | Visual/Audio、UI Requirements、禁止添加 | 高亮绘制与层级；需要窄例外（已批准） |
| `design/gdd/drag-drop-interaction.md` | Game Feel / UI | 玩家执行提示的移动；开始拖拽清除高亮；无直接依赖 |
| `design/gdd/undo-history.md` | Dependencies | 撤销后 Toolbar 调用 `reset()`；Hint 不读取历史 |
| `design/gdd/save-resume-game.md`（未编写） | Persistence | Hint 循环状态不写入存档 |
| `design/gdd/game-concept.md` | Pillars / Visual Identity Anchor | 支柱 1、3、4；蓝色高亮与 <150ms 动效约束 |
| `design/gdd/systems-index.md` | 系统行 / 高风险表 | Layer Feature、Priority MVP、依赖 1/3、提示质量风险 |
| `design/registry/entities.yaml` | formulas / constants | 新增 5 条 `hint_*` 公式；引用 `game_completion`、`legal_move_count`、`tableau_count` |

**已知文档间措辞歧义**：`game-concept.md` 的 Visual Identity Anchor 写「唯一的高亮色只用于可落位提示」，而 `drag-drop-interaction.md` 写「不做合法落点高亮」。本 GDD 的解释是：蓝色高亮只用于**提示建议的落点**，不用于拖拽时的合法落点预览。建议在 art bible 或概念文档中改一句措辞，避免后续误加拖拽高亮。

## Acceptance Criteria

> **测试前提。** 公式级测试使用 `legal_moves` / `is_won` 桩，端到端测试使用真实 Card & Deck Model + Rules。五个公式若不在公开接口上，实现须提供 `internal` + `InternalsVisibleTo` 或纯函数门面，否则公式级 AC 无法执行。

### Core Rules

- **AC-01（只读）** GIVEN 任一有效模型，WHEN 调用 `request_hint(model)`，THEN 调用前后的模型快照逐字段相同，且 Hint 未调用任何写入接口。
- **AC-02（结果形状·有提示）** GIVEN 存在至少一个候选，WHEN 调用 `request_hint`，THEN `has_hint = true`、`move ≠ null`、`reason = NONE`、`candidate_count ≥ 1`、`cycle_index ∈ [0, candidate_count − 1]`。
- **AC-03（结果形状·无提示）** GIVEN 已胜利，WHEN 调用 `request_hint`，THEN `has_hint = false`、`move = null`、`reason = GAME_WON`、`candidate_count = 0`、`cycle_index = −1`。
- **AC-04（候选来源）** GIVEN 任一有效模型，WHEN `request_hint` 返回移动，THEN 该移动的 `source`、`destination`、`cards` 与 `legal_moves(model)` 中某一条逐一相同。
- **AC-05（过滤整列搬运）** GIVEN `legal_moves(model)` 包含一条源桌面列、目标空桌面列、`count = height(source)` 的移动，WHEN 调用 `request_hint`，THEN 返回的移动不是该条，且 `has_hint = true`（该局面必然还存在至少一个未被过滤的候选，见 AC-17）。
- **AC-06（Tier 1 优先）** GIVEN 同时存在 Tier 1 与更低层的候选，WHEN 调用 `request_hint`，THEN 返回移动的 `hint_tier = 1`。
- **AC-07（层间顺序）** GIVEN 只有 Tier 3 与 Tier 5 的候选，WHEN 调用 `request_hint`，THEN 返回移动的 `hint_tier = 3`。
- **AC-08（同层平局）** GIVEN 两条候选同层，WHEN 调用 `request_hint`，THEN 返回 `enumeration_index` 较小的那条。
- **AC-09（最低自由单元索引）** GIVEN 同一张牌可移到多个空自由单元，WHEN 调用 `request_hint`，THEN 目标为索引最低的空自由单元。
- **AC-10（首次请求）** GIVEN 全新的 Hint 实例与 N ≥ 2 个候选，WHEN 调用 `request_hint`，THEN 返回排序第一的候选，`cycle_index = 0`。
- **AC-11（循环与回绕）** GIVEN 同一模型与 N ≥ 2 个候选，WHEN 连续调用 `request_hint` N 次，THEN 返回的 `cycle_index` 依次为 `0, 1, …, N−1`；第 N+1 次回到 `0`。
- **AC-12（列表变化重置）** GIVEN 已返回过一次提示，WHEN 牌局变化导致候选列表不同，THEN 下一次请求返回新的最佳候选，`cycle_index = 0`。
- **AC-13（显式 reset）** GIVEN `cursor ≠ 0`，WHEN 调用 `reset()` 且模型未变，THEN 下一次请求返回 `cycle_index = 0`。
- **AC-14（同一牌面继续）** GIVEN 撤销/读档恢复到与上次完全相同的牌面，WHEN 调用 `request_hint`，THEN 候选列表相同，`cursor` 前进一位而不是重置。
- **AC-15（胜利优先·桩）** GIVEN 桩使 `is_won = true` 且 `legal_moves` 非空，WHEN 调用 `request_hint`，THEN `reason = GAME_WON`、`move = null`。真实胜利局面下 `legal_moves` 为空，因此本 AC 只能对桩执行。
- **AC-16（卡死）** GIVEN 有效模型、未胜利且 `legal_move_count = 0`，WHEN 调用 `request_hint`，THEN `reason = NO_USEFUL_MOVES`、`has_hint = false`、`cycle_index = −1`。
- **AC-17（无有用移动与卡死重合）** GIVEN 任一有效模型，WHEN `request_hint` 返回 `NO_USEFUL_MOVES`，THEN `legal_move_count = 0`（等价于 `is_stuck = 1`）。「只剩整列搬运」的局面不可构造，属性测试应验证该不可达性。
- **AC-18（确定性）** GIVEN 同一模型与同一循环状态，WHEN 两次独立调用 `request_hint`（中间 `reset()`），THEN 两次结果完全相同。实现不得读取时间、随机数、Undo History、统计或设置。

### Formulas

- **AC-F01（hint_tier = 1）** GIVEN 一条回收移动，WHEN 计算 `hint_tier`，THEN 结果为 1。
- **AC-F02（hint_tier = 2）** GIVEN 源桌面列、目标非空桌面列、`count = height(source)`，WHEN 计算，THEN 结果为 2。
- **AC-F03（hint_tier = 3）** GIVEN 源自由单元、目标非空桌面列，WHEN 计算，THEN 结果为 3。
- **AC-F04（hint_tier = 4）** GIVEN 源桌面列、目标非空桌面列、`count < height(source)`，WHEN 计算，THEN 结果为 4。
- **AC-F05（hint_tier = 5）** GIVEN 桌面列 → 自由单元、自由单元 → 空桌面列、或桌面列 → 空桌面列且 `count < height(source)`，WHEN 计算，THEN 结果为 5。
- **AC-F06（hint_tier = 0）** GIVEN 源桌面列、目标空桌面列、`count = height(source)`，WHEN 计算，THEN 结果为 0。
- **AC-F07（hint_ranked_moves）** GIVEN 一组带已知 tier 与枚举下标的桩移动，WHEN 计算 `hint_ranked_moves`，THEN 结果按 `(tier, enumeration_index)` 升序、剔除 tier 0，且长度等于 `hint_candidate_count`。
- **AC-F08（hint_candidate_count）** GIVEN 桩列表中有 k 条 tier > 0、m 条 tier = 0，WHEN 计算，THEN 结果为 k。
- **AC-F09（hint_cycle_index·首次）** GIVEN `candidate_count = 5`、`same_candidate_list = 0`，WHEN 计算，THEN 结果为 0。
- **AC-F10（hint_cycle_index·前进）** GIVEN `previous_cursor = 0`、`candidate_count = 5`、`same = 1`，WHEN 计算，THEN 结果为 1。
- **AC-F11（hint_cycle_index·回绕）** GIVEN `previous_cursor = 4`、`candidate_count = 5`、`same = 1`，WHEN 计算，THEN 结果为 0。
- **AC-F12（hint_cycle_index·空）** GIVEN `candidate_count = 0`，WHEN 计算，THEN 结果为 −1。
- **AC-F13（hint_available·胜利守卫）** GIVEN 桩使 `game_completion = 1` 且候选数 > 0，WHEN 计算，THEN 结果为 0。
- **AC-F14（hint_available·可提示）** GIVEN `game_completion = 0`、`hint_candidate_count = 5`，WHEN 计算，THEN 结果为 1。
- **AC-F15（hint_available·无候选）** GIVEN `game_completion = 0`、`hint_candidate_count = 0`，WHEN 计算，THEN 结果为 0。

### 跨系统与性能

- **AC-X01（与 Rules 一致）** GIVEN 任一有效模型，WHEN `request_hint` 返回移动，THEN 该移动属于 `legal_moves(model)`；Hint 不调用 `can_move` 或 `auto_move_destination` 来“验证”它。
- **AC-X02（与 Model 一致）** GIVEN 任一有效模型，WHEN 调用 `request_hint`，THEN 模型快照不变；Hint 只使用只读查询。
- **AC-X03（过滤一致性）** GIVEN `legal_moves` 包含整列搬运，WHEN 计算候选，THEN 该条从不进入 `hint_ranked_moves`。
- **AC-X04（空合法移动）** GIVEN `legal_moves` 为空且未胜利，WHEN 调用 `request_hint`，THEN `NO_USEFUL_MOVES`。
- **AC-X05（胜利优先）** 同 AC-15；桩验证 `GAME_WON` 先于候选计算。
- **AC-X06（可用性等价）** GIVEN 任一有效模型，WHEN 调用 `request_hint`，THEN `hint_available(model) = 1` 当且仅当 `has_hint = true`。
- **AC-X07（候选数一致）** GIVEN 任一有效模型，WHEN 计算，THEN `candidate_count = |hint_ranked_moves(model)| = hint_candidate_count(model)`。
- **AC-X08（确定性跨实例）** GIVEN 同一模型与同一初始状态，WHEN 两个独立 Hint 实例各请求一次，THEN 结果相同。
- **AC-X09（性能）** GIVEN Release 构建、单线程、无调试器，WHEN 对最坏情形（候选数接近 768 的桩）连续调用 1,000 次并丢弃前 100 次预热，THEN P95 ≤ 1.0 ms、max ≤ 3.0 ms（含一次 `legal_moves`）；Hint 自身（不含 Rules）P95 ≤ 0.25 ms。报告必须记录实际候选数与硬件型号；**目标硬件未在 `project.yaml` 钉死前，本项标 `NOT ASSESSED`**。
- **AC-X10（测试可达性）** GIVEN 五个公式，WHEN 运行公式级测试，THEN 能通过 `internal` + `InternalsVisibleTo` 或纯函数门面直接调用；若只能通过公开 `request_hint` 间接观察，公式级 AC 不成立。
- **AC-X11（无效模型）** GIVEN 违反不变量的模型，WHEN 调用 `request_hint`，THEN 实现按前置条件断言或拒绝，**不得返回 `INVALID_MODEL`、不得修复模型**。本项是防御性契约检查，不属于正常玩法路径。

### 不在本 GDD 验证范围（NOT ASSESSED）

以下行为属于下游系统，必须在其 GDD 中验收，本 GDD 不声称已验证：

- Toolbar & Menus：提示按钮/快捷键、无提示文案、`reset()` 调用时机。
- Table Rendering & Layout：高亮的绘制、缩放重绘、清除时机（拖拽开始、牌局变化、撤销、新局/重开/读档、`Won/Stuck/NoGame`）。
- 目标硬件性能：`project.yaml` 未设置 `platform` / `performance` 前，AC-X09 的数值不可判定。
- 白盒约束：不调用 `can_move`、不读取 Undo/统计/设置、不保存牌局副本、单线程——通过实现审查或依赖扫描验证，不作为黑盒 AC。
- 提示质量的“好用程度”：属于主观体验，由 playtest 评估，不由单元测试断言。

## Open Questions

| # | 问题 | 负责人 | 目标 |
|---|------|--------|------|
| 1 | （已解决 2026-10-24）Rules §5 已明确「目标外层、长度内层」；Hint 示例 index 以 Rules 为准。 | Rules GDD | 已完成 |
| 2 | Card & Deck Model 是否提供牌堆类型标签 `pile_type(pile)`？没有它 `hint_tier` 端到端不可测。 | Card & Deck Model / architecture | 2026-10-31 |
| 3 | Art bible 缺失；`#000080`、3px 环带、8px 圆角都是暂定值。 | art-director / 用户 | 2026-11-07（`/art-bible` 后） |
| 4 | Table Rendering GDD 的窄例外如何措辞（仅提示环、9-slice 填充环、无运行时线条）？ | Table Rendering GDD | 实现前 |
| 5 | Tier 1 回收移动是否应排除「不安全」的低牌（例如过早把 2 送上回收单元）？当前决定：不排除。 | game-designer / 用户 | 首个可玩版本（playtest） |
| 6 | Toolbar 的触发键位、无提示文案、`reset()` 调用时机。 | Toolbar & Menus GDD | MVP UI 前 |
| 7 | 目标硬件与性能预算未钉死；AC-X09 在此之前标 `NOT ASSESSED`。 | producer / technical-director | Technical Setup |
| 8 | 提示质量的 playtest：用已知局号验证「提示是否像原作」。 | qa-lead / 用户 | 首个可玩版本 |
| 9 | `Stuck` 状态下 Toolbar 应显示什么（提示、重开、撤销）？ | Toolbar & Menus / Game State Machine | MVP UI 前 |
| 10 | 混合参考（Windows 7/XP 功能基线）提示是否有闪烁、定时消失或外框？公开资料未记录；当前采用「内缩、无动画、无超时」。 | 用户 / art-director | 参考机截图后 |
| 11 | 概念文档「唯一高亮色只用于可落位提示」与 Drag & Drop「不做合法落点高亮」的措辞歧义。 | creative-director / art-director | art bible |

**注意**：问题 1 若最终改为「长度外层、目标内层」，空桌面列目标的 Tier 5 候选与 `hint_ranked_moves` 示例会改变，必须同步更新本 GDD 与 Rules 的验收标准。问题 2 未解决前，`hint_tier` 只能做桩测试，不能做端到端测试。
