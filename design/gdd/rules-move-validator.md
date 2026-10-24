# Rules & Move Validator

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 原版规则，零改动

## Summary

Rules & Move Validator 只读地判断空当接龙的每个动作是否合法：桌面列降序异色、回收单元单向 A→K、自由单元容量 1、标准超级移动容量，以及多张牌的序列整体移动。它还提供胜利、卡死判定与双击目标选择；不修改模型，不实现撤销或提示选择。

> **Quick reference** — Layer: `Core` · Priority: `MVP` · Key deps: `Card & Deck Model`

## Overview

Rules & Move Validator 判断空当接龙中的每一个动作是否合法：牌可以移到哪一列、哪个自由单元或哪个回收单元，以及多张牌能否作为一个序列整体移动。它只读 Card & Deck Model，不修改任何牌堆；Drag & Drop Interaction 在落牌前询问它，Game State Machine 用它判断胜利与卡死，Hint System 用它枚举可行动作。它实现原版规则：桌面列降序异色、回收单元单向、标准超级移动容量。

## Detailed Design

### Core Rules

**1. 范围与只读契约**

- Rules & Move Validator 只读 Card & Deck Model，不修改任何牌堆。
- 输入：模型状态、源牌堆、目标牌堆、数量 `count`。
- 输出：`MoveResult`（合法/非法、原因、移动单元）。
- 不负责发牌、渲染、动画、输入、撤销或提示选择。

**2. 定义**

- 桌面列有序，自底向顶；顶牌是最后一个元素。
- **合法序列**：桌面列的连续后缀，自底向顶每张比上一张等级小 1，且颜色交替。形式化：对序列 `c[0..k−1]`，`rank(c[i+1]) = rank(c[i]) − 1` 且 `color(c[i+1]) ≠ color(c[i])`。
- 空列：0 张牌的桌面列。
- 自由单元：0 或 1 张牌。
- 回收单元：绑定花色，A→K 单向锁定；标准规则不允许把牌移回。
- 移动单元：单张牌或合法序列。

**3. 合法移动（完整集合）**

- **桌面列 → 桌面列**：
  - 移动单元必须是合法序列（长度 1–13）。
  - 目标顶牌等级 = 移动单元底牌等级 + 1，且颜色相反；或目标为空列。
  - 若长度 > 1：长度 ≤ 超级移动容量（见 §4）。
  - 单张移动不受容量限制。
  - 源与目标必须不同。
- **桌面列 → 自由单元**：只能移动顶牌，长度 1；目标自由单元为空。
- **桌面列 → 回收单元**：只能移动顶牌，长度 1；目标回收单元同花色，等级 = 当前高度 + 1（空为 A）。
- **自由单元 → 桌面列**：单张；目标顶牌等级 = 牌等级 + 1 且异色；或空列。
- **自由单元 → 回收单元**：单张；同花色、下一等级。
- **回收单元 → 任何牌堆**：非法 `FOUNDATION_LOCKED`；标准规则不允许移回。
- **任何牌堆 → 自由单元**：只有顶牌；目标自由单元为空。自由单元中的牌不能再移到另一个自由单元（无意义移动，不纳入合法移动）。
- **其他**：源空 → `SOURCE_EMPTY`；`count < 1` 或 `count > 源高度` → `COUNT_INVALID`；源 = 目标 → `SAME_SOURCE_AND_TARGET`；非顶牌点击（仅 `auto_move_destination`） → `NOT_TOP_CARD`。

**4. 超级移动容量**

- `F` = 移动前空自由单元数（0–4）；`E` = 移动前空桌面列数（0–8），**包含目标空列**；因此空目标必有 `E ≥ 1`。
- 目标为**非空**桌面列：`capacity = (F + 1) × 2^E`。
- 目标为**空**桌面列：`capacity = (F + 1) × 2^(E − 1)`；要求 `E ≥ 1`。目标列不能同时作为中转。
- 多张移动合法当且仅当 `sequence_length ≤ capacity`。
- 单张移动忽略容量。
- 合法序列最大长度 13；容量超过 13 时以 13 为实际上限。
- 超级移动是原版“一次移动整段”的便利；底层仍等价于逐张移动。游戏按整段执行，不做逐张动画。

**5. 胜利与卡死**

- `is_won(model) = game_complete`（52 张全在回收单元）。
- `legal_moves(model)` 枚举所有合法移动单元与目标。
- `is_stuck(model) = not is_won and legal_moves is empty`。
- 卡死是提示性状态：玩家仍可撤销或重开；Rules 不宣布失败、不阻塞输入。
- 枚举必须包含：顶牌 → 回收单元、顶牌 → 空自由单元、合法序列 → 桌面列（含空列、容量内）、自由单元牌 → 桌面列/回收单元。回收单元移出不在枚举内。
- `legal_moves` 必须去重：同一移动单元到同一目标只出现一次。
- 枚举顺序（确定性）：源按桌面列 1–8、自由单元 1–4；每个源依次尝试回收单元、自由单元目标（索引升序）、桌面列目标（索引升序）；**目标外层、长度内层**：每个目标按 `count = 1 .. max_movable_length` 递增。Hint System 可在此顺序上选择。

**6. 双击 / 自动移动**

- `auto_move_destination(model, source, card)`：`source` 是牌堆，`card` 是玩家点击的牌。
  - 若 `card` 不是 `source` 的顶牌 → `NOT_TOP_CARD`。
  - 优先级：回收单元（若合法）→ 空自由单元（最低索引）→ `NO_AUTO_MOVE`。
  - 回收单元按花色唯一；自由单元按索引 1–4 从左到右。
  - 双击只移动一张牌，不移动序列。
  - 若 `source` 是回收单元，返回 `FOUNDATION_LOCKED`。
- 双击不产生连锁；每次只处理一张。

**7. MoveResult 契约**

- `{ legal, source, destination, cards, reason }`。
- 合法：`cards` 为移动单元按源顺序（自底向顶）；`reason = NONE`。
- 非法：`legal = false`、`cards = []`、`reason` 为定义代码；模型不变。
- 原因代码：`SOURCE_EMPTY`、`COUNT_INVALID`、`SAME_SOURCE_AND_TARGET`、`NOT_TOP_CARD`、`SEQUENCE_NOT_MOVABLE`、`ILLEGAL_TABLEAU_DESTINATION`、`ILLEGAL_FREE_CELL_MOVE`、`SUPERMOVE_LIMIT_EXCEEDED`、`FREE_CELL_OCCUPIED`、`FOUNDATION_MISMATCH`、`FOUNDATION_LOCKED`、`NO_AUTO_MOVE`、`INVALID_MODEL`。
- 校验顺序（确定性）：源存在/非空 → count 合法 → 源 ≠ 目标 → 源不是回收单元 → 移动单元合法 → 目标占用/结构 → 回收单元规则 → 桌面列规则 → 容量。同一非法请求总是得到同一原因。
- `NOT_TOP_CARD` 只能由 `auto_move_destination` 返回；`can_move` 的移动单元总是从顶牌开始。`INVALID_MODEL` 在模型不变量校验失败，或调用方传入的 `E` / `destination_empty` 与模型不一致时返回。

**8. 所有权与无状态**

- Rules 无持久状态；同一模型 + 同一请求 → 同一结果。
- Rules 不调用模型 mutator；调用方拿到合法结果后调用 `model.move`。
- 模型结构不变量由 Card & Deck Model 保证；若模型无效，返回 `INVALID_MODEL`，不尝试修复。
- 撤销由 Undo History 恢复快照；Rules 不处理撤销。
- 提示由 Hint System 从 `legal_moves` 中选择；Rules 不选。

### States and Transitions

Rules 无持久状态机；以下是请求级状态：

| 状态 | 进入 | 离开 | 说明 |
|------|------|------|------|
| Idle | — | 收到请求 | 无持久状态 |
| Validating | 收到请求 | 完成全部检查 | 只读模型 |
| Legal | 全部检查通过 | 返回 MoveResult | 调用方执行 |
| Illegal | 任一检查失败 | 返回 MoveResult | 模型不变 |

模型本身不被 Rules 修改。

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|------|------|------|------|------------|
| Card & Deck Model | 上游 | 模型读取 | 无写入 | Model 拥有状态；Rules 只读 |
| Drag & Drop Interaction | 下游 | 移动请求 | `MoveResult` | Interaction 拥有输入；Rules 拥有合法性 |
| Game State Machine | 下游 | 模型状态 | `is_won` / `is_stuck` / `legal_moves` | State 拥有生命周期 |
| Undo History | 下游 | `MoveResult` | 移动记录 | Undo 拥有历史 |
| Hint System | 下游 | `legal_moves` | 建议移动 | Hint 拥有选择 |
| Statistics & Records | 下游 | `is_won` | 记录 | Stats 拥有记录 |
| Save/Resume Game | 下游 | 模型状态 | 无 | Save 拥有格式 |
| Toolbar & Menus | 下游 | 命令 | 合法性结果 | Toolbar 拥有命令 |
| Table Rendering / Animation / Audio | 下游 | `MoveResult` | 反馈 | 各自拥有表现 |
| Settings / Accessibility | 无 | — | — | 不直接交互 |

**接口契约**：

- `can_move(model, source, destination, count) -> MoveResult`
- `legal_moves(model) -> Move[]`
- `auto_move_destination(model, source, card) -> MoveResult`（`card` 是点击的牌；非顶牌返回 `NOT_TOP_CARD`）
- `is_won(model) -> bool`
- `is_stuck(model) -> bool`
- Rules 只读模型，不修改；同一输入产生同一结果。
- 调用方只有在 `legal = true` 时才调用 `Card & Deck Model.move`。

## Formulas

本节公式是纯判定函数：只读模型状态，不修改任何牌堆；相同输入必得相同输出。

### The valid_sequence formula

The valid_sequence formula is defined as:

`valid_sequence(column, count) = 1 if 1 ≤ count ≤ height(column) and ∀ i ∈ 0..count−2: rank(c[i+1]) = rank(c[i]) − 1 ∧ color(c[i+1]) ≠ color(c[i]), else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| column | tableau | 1–8 | 模型状态 | 源桌面列 |
| count | int | 0–52 | 请求 | 从顶部取出的连续张数 |
| height(column) | int | 0–52 | `Card & Deck Model` | 列高 |
| c[i] | card | 1–13 rank | 模型状态 | 自底向顶的第 i 张 |
| valid_sequence | int | {0, 1} | 本公式 | 1 = 合法序列 |

**Output Range:** {0,1}。count=1 时条件为空真（单张总是合法序列）；count=0 或 > height 时为 0。

**Example:** 自底向顶 `K♠ Q♥ J♣ T♦`，count=4 → 1；`K♠ Q♠` → 0（同色）；`K♠ J♥` → 0（等级不连续）；height=4、count=5 → 0。

### The max_sequence_length formula

The max_sequence_length formula is defined as:

`max_sequence_length(column) = 0 if height(column) = 0, else 1 + trailing_links(column)`

`trailing_links` 从顶牌向下数相邻且满足“等级减 1、颜色交替”的对数。

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| column | tableau | 1–8 | 模型状态 | 源桌面列 |
| height(column) | int | 0–52 | `Card & Deck Model` | 列高 |
| trailing_links | int | 0–12 | 模型状态 | 从顶部向下连续的合法链接数 |
| max_sequence_length | int | 0–13 | 本公式 | 该列可整体移动的最长后缀 |

**Output Range:** 0–13。13 是结构上限（等级严格递减且只有 13 个等级），不是可调参数。

**Example:** `[K♠, Q♥, J♣, T♦]` → 4；`[K♠, Q♠]` → 1；空列 → 0。一致性测试：`valid_sequence(column, count) ⟺ 1 ≤ count ≤ max_sequence_length(column)`。

### The tableau_accepts formula

The tableau_accepts formula is defined as:

`tableau_accepts(destination, card) = 1 if destination is empty or (rank(top(destination)) = rank(card) + 1 ∧ color(top(destination)) ≠ color(card)), else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| destination | tableau | 1–8 | 模型状态 | 目标桌面列 |
| card | card | rank 1–13 | 模型状态 | 待放置的底牌 |
| top(destination) | card or null | — | 模型状态 | 目标顶牌；空列为 null |
| tableau_accepts | int | {0, 1} | 本公式 | 1 = 接受 |

**Output Range:** {0,1}。空列接受任意牌；非空列只接受等级小 1 且异色的牌。

**Example:** 空列接受任意牌；顶牌 9♥ 接受 8♠、8♣，拒绝 8♥、8♦、7♠。

### The foundation_accepts formula

The foundation_accepts formula is defined as:

`foundation_accepts(foundation, card) = 1 if suit(card) = foundation_suit(foundation) ∧ rank(card) = foundation_height(foundation) + 1, else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| foundation | foundation | 1–4 | 模型状态 | 目标回收单元 |
| card | card | rank 1–13 | 模型状态 | 待放置的牌 |
| foundation_suit | enum | {C,D,H,S} | 模型状态 | 回收单元绑定花色 |
| foundation_height | int | 0–13 | `Card & Deck Model` | 当前高度 |
| foundation_accepts | int | {0, 1} | 本公式 | 1 = 接受 |

**Output Range:** {0,1}。空回收单元只接受 A；高度 13 时不接受任何牌。

**Example:** 空 ♥ 回收单元接受 A♥；高度 9（顶牌 9♥）接受 10♥，拒绝 J♥ 或 10♠。

### The foundation_next_rank formula

The foundation_next_rank formula is defined as:

`foundation_next_rank(suit) = foundation_height(suit) + 1`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| suit | enum | {C,D,H,S} | 模型状态 | 回收单元花色 |
| foundation_height | int | 0–13 | `Card & Deck Model` | 当前高度 |
| foundation_next_rank | int | 1–14 | 本公式 | 下一张应收等级；14 = 已完成 |

**Output Range:** 1–14。14 是越界哨兵，表示已完成、拒绝一切；**不得 clamp 到 13**，否则会接受第二张 K。

**Example:** 高度 0 → 1（A）；高度 12 → 13（K）；高度 13 → 14（完成）。

### The supermove_capacity formula

The supermove_capacity formula is defined as:

`supermove_capacity(F, E, destination_empty) = (F + 1) × 2^E if destination_empty = 0`
`supermove_capacity(F, E, destination_empty) = (F + 1) × 2^(E − 1) if destination_empty = 1 and E ≥ 1`
`destination_empty = 1 ⇒ E ≥ 1`（E 包含目标空列）；若调用方传入 `E = 0`，防御性返回 `INVALID_MODEL`（正常模型不可达）

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| F | int | 0–4 | `empty_free_cell_count` | 移动前空自由单元数 |
| E | int | 0–8 | `empty_tableau_column_count` | 移动前空桌面列数，**不含源列**（即使移动后源列变空） |
| destination_empty | int | {0, 1} | 请求 | 目标列是否为空 |
| supermove_capacity | int | 1–1280 | 本公式 | 多牌移动的最大张数 |

**Output Range:** 非空目标 1–1280；空目标 1–640；实际生效上限为 13（合法序列最长 13）。空目标时 `E ≥ 1`（E 包含目标空列）；`E=0` 属调用方/模型不一致，防御性返回 `INVALID_MODEL`，不做负移位或 clamp。

**Example:** F=3、E=1、非空目标 → `(3+1)×2 = 8`；空目标 → `(3+1)×2^0 = 4`；F=4、E=0 → 5；F=4、E=8 → 1280（被 13 截断）。

### The max_movable_length formula

The max_movable_length formula is defined as:

`max_movable_length(source, destination) = 0 if source is empty`
`max_movable_length(source, destination) = 1 if max_sequence_length(source) = 1`
`max_movable_length(source, destination) = min(max_sequence_length(source), rank_count_per_suit, supermove_capacity(F, E, destination_empty)) otherwise`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| source | pile | tableau/free cell | 模型状态 | 源牌堆 |
| destination | pile | tableau | 模型状态 | 目标桌面列 |
| max_sequence_length | int | 0–13 | 本 GDD | 源列最长合法后缀；自由单元源视为 1 |
| rank_count_per_suit | int | 13 | registry | 等级上限 |
| supermove_capacity | int | 1–1280 | 本 GDD | 容量 |
| max_movable_length | int | 0–13 | 本公式 | 本次可移动的最大张数 |

**Output Range:** 0–13。单张移动在容量 ≥ 1 时总是允许；容量公式已保证这一点，公式中的单张分支是显式不变量，不是额外代码分支。

**Example:** F=1、E=1、源列有 4 张合法序列：非空目标容量 `(1+1)×2=4` → 可移 4；空目标容量 `(1+1)×1=2` → 只能移 2。

### The legal_move_count formula

The legal_move_count formula is defined as:

`legal_move_count(model) = |legal_moves(model)|`

`legal_moves` 是 Core Rules §5 的完整枚举；自由单元按**格**枚举（每个空自由单元是一个独立目标），不按“自由单元类别”合并。

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| model | model | — | 模型状态 | 当前牌局 |
| legal_moves | list | 0–768 | 本 GDD | 去重后的合法移动列表 |
| legal_move_count | int | 0–768 | 本公式 | 合法移动数量 |

**Output Range:** 0–768（结构上限）。`legal_move_count = 0` 是卡死的必要条件，不是充分条件（还要排除已胜利）。

**Example:** 48 张牌接近终局、存在 4 个空自由单元时，按格枚举得到的计数大于按类别合并；两者都只用于 Hint/统计，`is_stuck` 只关心是否为零。

### The is_stuck formula

The is_stuck formula is defined as:

`is_stuck(model) = 1 if game_completion(model) = 0 and legal_move_count(model) = 0, else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| game_completion | int | {0,1} | `Card & Deck Model` | 是否已胜利 |
| legal_move_count | int | 0–768 | 本公式 | 合法移动数量 |
| is_stuck | int | {0,1} | 本公式 | 1 = 无合法移动且未胜利 |

**Output Range:** {0,1}。已胜利时即使计数为 0 也返回 0——胜利守卫是必需的，不是可选。

**Example:** 未胜利且 `legal_move_count = 0` → 1；已胜利且计数为 0 → 0；未胜利但仍有合法移动 → 0。

## Edge Cases

- **If 源牌堆为空**: 返回 `SOURCE_EMPTY`，模型不变。
- **If count < 1 或 count > 源高度**: 返回 `COUNT_INVALID`，模型不变。
- **If 源 = 目标**: 返回 `SAME_SOURCE_AND_TARGET`，不产生状态变化。
- **If 源是回收单元**: 返回 `FOUNDATION_LOCKED`；标准规则不允许移回，撤销是唯一回退方式。
- **If 移动非顶牌的单张**: 返回 `NOT_TOP_CARD`。
- **If 序列不是连续降序异色**: 返回 `SEQUENCE_NOT_MOVABLE`。
- **If 桌面列目标同色或等级不连续**: 返回 `ILLEGAL_TABLEAU_DESTINATION`。
- **If 目标为空列且 E = 0**: 正常模型不可达（E 包含目标空列，空目标 ⇒ E ≥ 1）；防御性返回 `INVALID_MODEL`，不做负移位或 clamp。
- **If 多牌移动超过容量**: 返回 `SUPERMOVE_LIMIT_EXCEEDED`；玩家仍可逐张移动，容量只限制整段移动。
- **If 单张移动且容量为 1**: 合法；单张不受容量限制。
- **If 目标自由单元已被占用**: 返回 `FREE_CELL_OCCUPIED`。
- **If 自由单元中的牌移到另一个自由单元**: 返回 `ILLEGAL_FREE_CELL_MOVE`；无意义移动，不纳入合法移动。
- **If 回收单元花色或等级不匹配**: 返回 `FOUNDATION_MISMATCH`。
- **If 回收单元已完成（高度 13）**: `foundation_next_rank = 14`，拒绝一切；不 clamp 到 13。
- **If 双击非顶牌**: 返回 `NOT_TOP_CARD`。
- **If 双击时回收单元与自由单元都可放**: 回收单元优先。
- **If 双击时两者都不可放**: 返回 `NO_AUTO_MOVE`，不产生移动。
- **If 双击自由单元中的牌**: 只尝试回收单元；不会再送回另一个自由单元。
- **If 双击回收单元中的牌**: 返回 `FOUNDATION_LOCKED`，不移动。
- **If 序列长度超过 13**: 不可能通过 `valid_sequence`；实现应断言而非截断。
- **If 容量超过 13**: 以 13 为实际上限；不改变公式，只在 `max_movable_length` 取 min。
- **If E 在移动前为 1、移动后源列变空**: E 仍按移动前的值计算，源列不计入中转；这是原版约定。
- **If `legal_move_count = 0` 且未胜利**: `is_stuck = 1`；玩家仍可撤销/重开，Rules 不宣布失败。
- **If `legal_move_count = 0` 且已胜利**: `is_stuck = 0`；胜利守卫优先。
- **If 枚举漏掉某一类移动**: 会误报卡死；`legal_moves` 必须覆盖桌面列→回收单元、自由单元→回收单元、桌面列→自由单元、桌面列→桌面列、自由单元→桌面列五类。
- **If 枚举顺序不确定**: Hint 结果会随实现变化；使用 §5 的固定顺序。
- **If 模型在验证期间被其他系统修改**: 结果可能过期；调用方必须在验证与执行之间保持模型不变。
- **If 模型本身无效**: 返回 `INVALID_MODEL`，不尝试修复；先由 Card & Deck Model 校验。
- **If 同一移动单元有多个等价自由单元目标**: 按格枚举，每个空自由单元都是一个独立目标；Hint 使用最低索引。

## Dependencies

**上游依赖**：Card & Deck Model（GDD 已完成，Hard）。锁定事实：只读模型、结构不变量、`foundation_height`、`game_complete`、`empty_free_cell_count`、`empty_tableau_column_count`、顶牌/张数读取。

**下游依赖（依赖本系统的系统）**：

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Drag & Drop Interaction | Hard | `can_move` / `auto_move_destination` | 落牌前必须得到 `legal = true` |
| Game State Machine | Hard | `is_won` / `is_stuck` / `legal_moves` | 生命周期与胜负显示 |
| Undo History | Hard | `MoveResult` | 记录移动，供撤销 |
| Hint System | Hard | `legal_moves` | 从固定顺序中选择建议 |
| Statistics & Records | Soft | `is_won` | 记录胜负 |
| Save/Resume Game | Soft | 模型状态 | 保存/恢复后由 Rules 重新判定 |
| Toolbar & Menus | Soft | 命令合法性 | 重开/新局不经过 Rules |
| Table Rendering / Animation / Audio | Soft | `MoveResult` | 反馈表现 |
| Settings / Accessibility | None | — | 不直接交互 |

**接口契约**：

- `can_move(model, source, destination, count) -> MoveResult`
- `legal_moves(model) -> Move[]`（固定顺序、去重、按格枚举自由单元）
- `auto_move_destination(model, source, card) -> MoveResult`（`card` 是点击的牌；非顶牌返回 `NOT_TOP_CARD`）
- `is_won(model) -> bool`
- `is_stuck(model) -> bool`
- Rules 只读模型，不修改；同一输入产生同一结果。
- 调用方只有在 `legal = true` 时才调用 `Card & Deck Model.move`；Rules 不执行移动。

**Provisional assumptions（下游未设计，先假定）**：

- Card & Deck Model 提供 `top_card`、`count`、`foundation_height`、`game_complete`、`empty_free_cell_count`、`empty_tableau_column_count` 的只读查询。
- Drag & Drop Interaction 在 `drag` 开始时读取一次模型，落牌时再次调用 `can_move`，不缓存过期结果。
- Undo History 保存 `MoveResult` 与执行前的模型快照；Rules 不参与撤销。
- Hint System 从 `legal_moves` 的固定顺序中选择，不在 Rules 内实现启发式。

**Registry references**：`empty_free_cell_count`、`empty_tableau_column_count`、`foundation_height`、`game_completion`、`rank_count_per_suit`、`tableau_count`、`suit_count`、`card_color` 的 `referenced_by` 应追加 `design/gdd/rules-move-validator.md`（Phase 5b 更新）。

## Visual/Audio Requirements

无。纯规则判定，不产生视觉或音频需求；合法/非法反馈由 Drag & Drop Interaction 与 Animation & Feedback 表现。

## Game Feel

规则本身是瞬时的；它保证每次落牌判定一致、可预期，使 Drag & Drop Interaction 的吸附/回弹与 Animation & Feedback 的反馈不被规则歧义干扰。

## UI Requirements

无直接 UI。Drag & Drop Interaction 用 `can_move` 高亮合法目标；Toolbar & Menus 用 `is_won` / `is_stuck` 决定提示；Hint System 读取 `legal_moves`。

## Cross-References

| 系统 / 来源 | 引用内容 | 状态 |
|-------------|----------|------|
| Card & Deck Model | 模型读取、结构不变量、高度/完成/空位计数 | GDD 已完成 |
| Drag & Drop Interaction | `can_move` / `auto_move_destination` | GDD 已完成（design/gdd/drag-drop-interaction.md） |
| Game State Machine | `is_won` / `is_stuck` | GDD 已完成（design/gdd/game-state-machine.md） |
| Undo History | `MoveResult` | GDD 已完成（design/gdd/undo-history.md） |
| Hint System | `legal_moves` | GDD 已完成（design/gdd/hint-system.md） |
| Supermove 规则 | `(F+1)×2^E` 与空目标 `(F+1)×2^(E−1)` | 已查证（Wikipedia FreeCell） |

规则来源：Wikipedia — *FreeCell*（`https://en.wikipedia.org/wiki/FreeCell#Supermoves`），2026-10-24 查证；回收单元单向与标准移动规则同页。

## Acceptance Criteria

**测试约定**：牌记法 `A,2–9,T,J,Q,K` + `C,D,H,S`；牌堆 `T1–T8`（桌面列）、`FC1–FC4`（自由单元）、`FD-C/D/H/S`（绑定花色的回收单元）；“状态指纹”指全部牌堆的有序牌列表。`can_move(source, destination, count)` 的移动单元总是从源顶牌开始；`auto_move_destination(source, card)` 额外接收点击的牌，用于检测非顶牌。

### 合法与非法移动

- **GIVEN** T1 自底向顶 `[Q♥, J♣]`、T2 顶牌 `K♠`、1 个空自由单元、没有空列，**WHEN** `can_move(T1, T2, 2)`，**THEN** `legal = true`、`reason = NONE`、`cards = [Q♥, J♣]`（自底向顶），模型不变。
- **GIVEN** T1 `[Q♥, J♣]`、1 个空自由单元、T2 为空（E=1，包含 T2），**WHEN** `can_move(T1, T2, 2)`，**THEN** 合法；容量 `(1+1)×2^0 = 2`。
- **GIVEN** 桌面列顶牌 `7♦`、FC2 为空，**WHEN** `can_move(T, FC2, 1)`，**THEN** 合法。**GIVEN** 桌面列顶牌 `7♥`、♥ 回收单元高度 6，**WHEN** `can_move(T, FD-H, 1)`，**THEN** 合法。
- **GIVEN** FC1 持有 `5♣`、桌面列顶牌 `6♦` 或该列为空，**WHEN** `can_move(FC1, T, 1)`，**THEN** 合法。**GIVEN** FC1 持有 `A♠`、♠ 回收单元为空，**WHEN** `can_move(FC1, FD-S, 1)`，**THEN** 合法。
- **GIVEN** 源牌堆为空，**WHEN** `can_move(..., 1)`，**THEN** `SOURCE_EMPTY`、`cards = []`、模型不变。**GIVEN** 源高 3，**WHEN** `count = 0` 或 `4`，**THEN** `COUNT_INVALID`，位置不变。
- **GIVEN** 源与目标为同一牌堆，**WHEN** `can_move(P, P, 1)`，**THEN** `SAME_SOURCE_AND_TARGET`，模型不变。**GIVEN** T `[9♠, 8♥]`，**WHEN** `auto_move_destination(T, 9♠)`，**THEN** `NOT_TOP_CARD`，不移动；`can_move` 从不产生该原因。
- **GIVEN** 列 `[K♠, Q♠]`（同色）或 `[K♠, J♥]`（等级缺口），**WHEN** 请求 2 张移动，**THEN** `SEQUENCE_NOT_MOVABLE`。**GIVEN** 合法序列底牌 `8♥`、目标顶牌 `9♥` 或等级缺口，**WHEN** `count` 等于序列长度，**THEN** `ILLEGAL_TABLEAU_DESTINATION`。
- **GIVEN** FC3 已被占用、源为桌面列顶牌，**WHEN** `can_move(桌面列, FC3, 1)`，**THEN** `FREE_CELL_OCCUPIED`。**GIVEN** `10♠` 与 ♥ 回收单元（空或高度 9），**WHEN** `can_move(..., FD-H, 1)`，**THEN** `FOUNDATION_MISMATCH`。
- **GIVEN** 3 张合法序列、F=0、E=0、非空目标（容量 1），**WHEN** `can_move(..., 3)`，**THEN** `SUPERMOVE_LIMIT_EXCEEDED`；**WHEN** `count = 1` 且目标接受顶牌，**THEN** 合法（单张忽略容量）。
- **GIVEN** 回收单元持有 ≥1 张牌，**WHEN** `can_move(foundation, anyPile, 1)`，**THEN** 对每个目标都返回 `FOUNDATION_LOCKED`。**GIVEN** 回收单元高度 13，**WHEN** 提供任意牌（包括第二张同花色 K），**THEN** `FOUNDATION_MISMATCH`；`foundation_next_rank = 14`，不得 clamp 到 13。
- **GIVEN** FC1 持有 `5♣`、FC2 为空，**WHEN** `can_move(FC1, FC2, 1)`，**THEN** `ILLEGAL_FREE_CELL_MOVE`；自由单元→自由单元不纳入合法移动。
- **GIVEN** 任意合法 `MoveResult`，**WHEN** 调用方执行 `model.move`，**THEN** 模型接受并产生该移动；**GIVEN** 任意非法 `MoveResult`，**WHEN** 不执行，**THEN** 状态指纹与调用前逐字节相同。

### 公式

- **GIVEN** 列 `[K♠, Q♥, J♣, T♦]`，**WHEN** 计算 `valid_sequence(count=4)` 与 `count=1`，**THEN** 均为 1；**GIVEN** `[K♠, Q♠]` 或 `[K♠, J♥]`，**THEN** 为 0；**GIVEN** 高 4、`count=5` 或 `count=0`，**THEN** 为 0。`max_sequence_length` 分别为 4、1、0；一致性：`valid_sequence(c,n)=1 ⟺ 1 ≤ n ≤ max_sequence_length(c)`。
- **GIVEN** 空桌面列或顶牌 `9♥`，**WHEN** 计算 `tableau_accepts`，**THEN** 空列接受任意牌；`9♥` 接受 `8♠`、`8♣`，拒绝 `8♥`、`7♠`。**GIVEN** 空 ♥ 回收单元或高度 9，**WHEN** 计算 `foundation_accepts`，**THEN** 空单元只接受 `A♥`；高度 9 接受 `10♥`，拒绝 `J♥`、`10♠`。
- **GIVEN** ♥ 回收单元高度 0、12、13，**WHEN** 计算 `foundation_next_rank`，**THEN** 分别为 1、13、14；高度 14 时拒绝所有牌，包括第二张 K♥。
- **GIVEN** F=3、E=1，**WHEN** 计算 `supermove_capacity`，**THEN** 非空目标为 8、空目标为 4；**GIVEN** F=4、E=0，**THEN** 非空目标为 5；**GIVEN** F=4、E=8，**THEN** 为 1280，但 `max_movable_length` 以 13 截断。E 按移动前计算，且不含源列（即使移动后源列变空）。
- **GIVEN** F=1、E=1、源列有 4 张合法序列，**WHEN** 计算 `max_movable_length`，**THEN** 非空目标为 4、空目标为 2。
- **GIVEN** 4 个空自由单元，**WHEN** 枚举一张牌到自由单元的移动，**THEN** `legal_moves` 产生 4 条（每格一条，不合并）；不存在重复的 `(移动单元, 目标)`；五类移动都出现在合适位置；自由单元→自由单元不出现。
- **GIVEN** 已胜利（52 张在回收单元、`legal_move_count=0`），**WHEN** 计算 `is_stuck`，**THEN** 为 0；**GIVEN** 未胜利且计数为 0，**THEN** 为 1；**GIVEN** 未胜利且有 ≥1 条合法移动，**THEN** 为 0。

### 确定性、只读与双击

- **GIVEN** 任意模型与请求，**WHEN** 重复调用 `can_move` / `legal_moves` / `is_won` / `is_stuck`，**THEN** 调用前后牌位置逐字节相同，且相同输入得到相同 `MoveResult`。
- **GIVEN** 顶牌同时满足回收单元与空自由单元，**WHEN** `auto_move_destination`，**THEN** 选择回收单元；**GIVEN** 只有空自由单元可放，**THEN** 选择最低索引的空自由单元；**GIVEN** 两者都不可放，**THEN** `NO_AUTO_MOVE`，模型不变。
- **GIVEN** 自由单元中的牌，**WHEN** 双击，**THEN** 只尝试回收单元；**GIVEN** 回收单元中的牌，**WHEN** 双击，**THEN** `FOUNDATION_LOCKED`。每次双击只移动一张牌，不产生连锁。
- **GIVEN** 任意模型，**WHEN** 枚举 `legal_moves`，**THEN** 顺序固定：源按桌面列 1–8、自由单元 1–4；每个源依次回收单元、自由单元（索引升序）、桌面列（索引升序）；桌面列源的移动长度从 1 到 `max_movable_length` 递增。
- **GIVEN** 调用方传入与模型不一致的 `E` / `destination_empty`，或模型不变量校验失败，**WHEN** 调用，**THEN** 返回 `INVALID_MODEL`，不做负移位或 clamp；正常模型不可达该分支（防御性测试）。
- **GIVEN** 同一非法请求，**WHEN** 多次调用，**THEN** 按 Core Rules §7 的校验顺序返回同一 `reason`；原因码与顺序一一对应。

## Open Questions

| 问题 | Owner | 目标解决时间 |
|------|-------|--------------|
| 双击优先级（回收单元优先）是否符合混合参考（Windows 7/XP 功能基线） | ux-designer + user | 首个可玩版本前 |
| 超级移动空目标 `E−1` 约定是否与混合参考一致 | systems-designer | 首个可玩版本前 |
| Hint 从 `legal_moves` 选哪一条 | Hint System GDD | Hint GDD 编写时 |
| `INVALID_MODEL` 的模型校验接口 | Card & Deck Model + architecture | 架构阶段 |
| `legal_moves` 全量枚举的性能上限 | lead-programmer | 架构阶段 |
