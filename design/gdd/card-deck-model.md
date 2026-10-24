# Card & Deck Model

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 原版规则，零改动

## Summary

Card & Deck Model 定义空当接龙的全部牌局数据：52 张标准牌、8 个桌面列、4 个自由单元与 4 个回收单元，以及原子移动、快照与序列化操作。它不判断移动是否合法——合法性属于 Rules & Move Validator——但它是规则、交互、渲染、提示、撤销与存档共同读写的唯一数据源。所有牌始终正面朝上。

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `None`

## Overview

Card & Deck Model 是空当接龙的唯一牌局数据源。它定义 52 张标准牌（A–K、四种花色）、8 个桌面列、4 个自由单元与 4 个回收单元，以及在这些结构上取牌、放牌、读取顶牌、提取连续后缀等无规则操作。所有牌在发牌后始终正面朝上——经典空当接龙没有背面朝下的牌，也没有翻牌动作；模型不判断移动是否合法，只保证结构不变量：每张牌恰好属于一个牌堆、自由单元最多一张、回收单元最多 13 张、整副牌不重不漏。它是规则、交互、渲染、提示、撤销与存档共同读写的底座。

## Detailed Design

### Core Rules

**1. 牌（Card）**

- 牌由 `suit` 与 `rank` 唯一标识；`suit` ∈ {♣ clubs, ♦ diamonds, ♥ hearts, ♠ spades}，`rank` ∈ {1=A, 2–10, 11=J, 12=Q, 13=K}。
- 颜色由花色派生：♦/♥ 为红色，♣/♠ 为黑色。
- 标准牌组恰好 52 张：每种花色 13 张，每种 `(suit, rank)` 恰好一张。
- **所有牌在发牌后始终正面朝上**。经典空当接龙没有背面朝下的牌、没有翻牌动作、没有隐藏信息；`face_up` 不是模型状态。

**2. 牌堆（Pile）**

- **桌面列（Tableau）**：8 列，有序栈，自底向顶存储；顶牌是最后一个元素；容量 0–52。
- **自由单元（Free Cell）**：4 个，每个容量 0 或 1。
- **回收单元（Foundation）**：4 个，每个绑定一种花色；只接受该花色的牌，且必须从 A 开始连续递增到 K；容量 0–13。
- 发牌完成后，52 张牌中的每一张恰好属于一个牌堆。

**3. 模型不变量（始终成立）**

- I1：发牌完成后以及每次成功操作后，总牌数 = 52，且没有重复牌。未发牌的模型允许为空（0 张），不适用 I1–I5。
- I2：每张牌恰好在一个牌堆中。
- I3：自由单元 ≤ 1 张。
- I4：回收单元 ≤ 13 张，只含绑定花色的牌，且等级从 A 起连续无缺口。
- I5：桌面列没有容量上限；所有牌正面朝上。
- 违反不变量的操作必须失败，并使模型保持操作前的状态（原子性）。

**4. 操作（行为契约）**

- 创建标准牌组。
- 按给定顺序把 52 张牌发到 8 列：前 4 列各 7 张，后 4 列各 6 张；全部正面朝上。Deal Generator 提供的序列按发牌顺序排列；第 i 列取 `deal_start_index(i)` 到 `deal_start_index(i) + deal_column_height(i) − 1` 的切片，切片第一张为列底，最后一张为列顶。**Deal 是原子替换**：即使模型已发牌，也以新序列整体替换；输入非法时保留替换前的完整状态。
- 读取任意牌堆的顶牌、张数、是否为空；读取返回不可变副本，调用方不能通过返回值修改模型。
- 移除顶牌、移除顶部连续 `k` 张（`1 ≤ k ≤ 列高`）、追加一张牌或一段连续序列，都是**原子移动的内部步骤**，不作为独立公开操作暴露；对外暴露原子移动、快照/恢复，序列化/反序列化由 `ModelSnapshotSerializer` 基于快照完成。
- 移除只按位置处理顶部连续后缀，不验证这 `k` 张是否构成合法序列。
- 追加只检查结构不变量（容量、花色、回收单元连续性、牌是否已存在），不检查桌面列的降序/异色规则。
- 原子移动：从源移除并追加到目标；任一步失败则整体不生效。
- 快照与恢复：为撤销提供完整状态副本。
- 序列化与反序列化：为存档提供完整状态；反序列化必须先验证全部不变量，失败则拒绝加载。

**5. 边界与所有权**

- 合法移动（桌面列降序且异色、可整体移动的序列、超级移动容量）、胜利/卡死判定、双击自动归位是否可用，全部由 Rules & Move Validator 决定。
- 模型不调用 Rules，避免循环依赖；调用方必须先获得 Rules 的许可，再调用模型。
- 模型是唯一数据源；其他系统只能通过原子移动、快照/恢复与反序列化修改状态，不得直接改写内部牌堆数组。读取返回不可变副本，不使用可写视图。
- 模型不产生随机数；所有随机性属于 Deal Generator。
- 回收单元与花色的绑定由初始化提供；模型提供按花色查询回收单元的接口，索引本身不承载花色语义。
- 失败结果必须包含 `success = false` 与 `failure_reason`；失败时 `cards` 为空，`source`/`destination` 为请求值，模型不发生任何变更。

### States and Transitions

模型本身没有“发牌/进行中/胜利/卡死”的游戏状态——那是 Game State Machine 的职责。模型只暴露牌堆级状态与整体一致性。

| 实体 | 状态 | 转移 | 触发 |
|------|------|------|------|
| 自由单元 | 空 | 占用 | 追加 1 张牌 |
| 自由单元 | 占用 | 空 | 移除该牌 |
| 回收单元 | 等级 n（0–13） | 等级 n+1 | 追加该花色下一等级 |
| 回收单元 | 等级 13 | 完成 | 追加 K |
| 回收单元 | 等级 n+1 | 等级 n | 撤销移除顶牌 |
| 桌面列 | 空 | 非空 | 追加牌或序列 |
| 桌面列 | 非空 | 空 | 移除全部剩余牌 |
| 整个模型 | 未发牌（空） | 已发牌（一致） | 发牌成功；发牌前不适用 I1–I5 |
| 整个模型 | 一致 | 一致 | 每个原子操作成功完成后 |
| 整个模型 | 一致 | 无效 | 反序列化校验失败；拒绝加载，不进入无效状态 |

没有“背面朝下”状态。

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|------|------|------|------|------------|
| Deal Generator | 上游 | 洗牌后的 52 张牌顺序 | 初始化后的模型 | Deal 拥有洗牌；模型拥有存储 |
| Rules & Move Validator | 下游 | 模型读取 | 合法/非法判定 | Rules 拥有合法性；模型拥有状态 |
| Game State Machine | 下游 | 模型状态 | 新局/重开/胜负状态 | State 拥有生命周期；模型拥有牌堆 |
| Drag & Drop Interaction | 下游 | 命中测试、模型读取 | 经 Rules 批准的移动请求 | Interaction 拥有输入；模型拥有变更 |
| Undo History | 下游 | 快照 | 恢复请求 | Undo 拥有历史；模型提供快照/恢复 |
| Hint System | 下游 | 模型读取 | 建议移动 | Hint 拥有启发式；模型只读 |
| Statistics & Records | 下游 | 张数/完成状态 | 局数、胜率、连胜 | Stats 拥有记录；模型只读 |
| Table Rendering & Layout | 下游 | 牌堆与牌 | 画面 | Rendering 拥有画面；模型只读 |
| Save/Resume Game | 下游 | 序列化状态 | 恢复后的模型 | Save 拥有格式；模型拥有状态 |
| Toolbar & Menus | 下游 | 状态 | 命令 | Toolbar 拥有命令；经 State 访问模型 |
| Animation & Feedback | 下游 | 移动结果 | 动效 | Animation 拥有表现 |
| Audio & SFX | 下游 | 移动结果 | 音效 | Audio 拥有表现 |
| Settings | 无 | — | — | 无直接交互 |
| Readability & Accessibility | 下游 | 模型读取 | 可访问表示 | Accessibility 拥有表现 |

**接口契约**：模型只提供读取与原子移动；所有写入必须先经过 Rules 的合法性判断。移动结果需包含源牌堆、目标牌堆、移动的牌（有序）、操作是否成功与失败原因，供撤销、动画、音效与统计消费。

## Formulas

本系统的公式全部是结构恒等式或派生状态，不含平衡数值。移动合法性（桌面列降序异色、可移动序列、超级移动容量）属于 Rules & Move Validator，不在本节。

### The deck_size formula

The deck_size formula is defined as:

`deck_size = suit_count × rank_count_per_suit`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| suit_count | int | 4（constant） | Core Rules §1 — {clubs, diamonds, hearts, spades} | 花色数量 |
| rank_count_per_suit | int | 13（constant） | Core Rules §1 — A=1 … K=13 | 每种花色的等级数量 |
| deck_size | int | 52 | Derived | 牌组总张数，也是 I1 的期望值 |

**Output Range:** 52–52（单一取值）。两个输入都是原版规则固定的常量；`deck_size` 同时是 I1 的期望值、4 个回收单元容量之和（4 × 13）与发牌分配的总量。

**Example:** suit_count = 4，rank_count_per_suit = 13 → deck_size = 4 × 13 = 52。

### The deal_column_height formula

The deal_column_height formula is defined as:

`deal_column_height(i) = base_height + 1 if i ≤ remainder, else base_height`
`base_height = floor(deck_size / tableau_count)`
`remainder = deck_size mod tableau_count`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| i | int | 1–8（1-based） | Core Rules §2 — 8 列 | 桌面列序号，1 为最左列 |
| deck_size | int | 52 | Derived（deck_size） | 待发牌总张数 |
| tableau_count | int | 8（constant） | Core Rules §2 | 桌面列数 |
| base_height | int | 6 | Derived | 每列至少张数 |
| remainder | int | 4 | Derived | 需要多分一张的列数 |
| deal_column_height(i) | int | 6 或 7 | Derived | 第 i 列在发牌后应有的张数 |

**Output Range:** 每列 6–7 张；8 列合计恒等于 `deck_size` = 52。i ≤ 4 得 7，i ≥ 5 得 6。

**Example:** deck_size = 52，tableau_count = 8 → base_height = 6，remainder = 4。i = 1–4 → 7；i = 5–8 → 6；合计 4 × 7 + 4 × 6 = 52。

### The deal_start_index formula

The deal_start_index formula is defined as:

`deal_start_index(i) = Σ_{j=1}^{i−1} deal_column_height(j)`，且 `deal_start_index(1) = 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| i | int | 1–8 | 同 deal_column_height | 桌面列序号 |
| deal_start_index(i) | int | 0–46 | Derived | 第 i 列取用洗牌序列时的起始下标（0-based） |

**Output Range:** 0–46。i = 8 时占用下标 46–51，结束下标恒为 `deck_size − 1`，可作为“发牌恰好用尽整副牌”的断言点。切片的第一张是列底，最后一张是列顶。

**Example:** 高度序列 7,7,7,7,6,6,6,6 → start = 0, 7, 14, 21, 28, 34, 40, 46。第 8 列占用 46–51，顶牌为下标 51 = deck_size − 1。

### The empty_free_cell_count formula

The empty_free_cell_count formula is defined as:

`empty_free_cells = free_cell_count − Σ_{f=1}^{free_cell_count} occupied(f)`

> **Canonical name:** registry 与下游 GDD 使用 `empty_free_cell_count` 指代此输出；本节的 `empty_free_cells` 是同一值的别名，范围 0–4，语义不变。

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| free_cell_count | int | 4（constant） | Core Rules §2 | 自由单元数量 |
| occupied(f) | int | 0 或 1 | 模型状态 | 第 f 个自由单元是否被占用 |
| empty_free_cells | int | 0–4 | Derived | 当前空自由单元数量，只读 |

**Output Range:** 0–4。发牌后为 4；4 个全占用时为 0。**不得 clamp 到 0**：若结果为负，说明 I3 已被破坏，应拒绝该状态而不是伪装成“全占用”。

**Example:** F1、F3 各有一张牌，F2、F4 为空 → empty_free_cells = 4 − 2 = 2。

### The empty_tableau_column_count formula

The empty_tableau_column_count formula is defined as:

`empty_tableau_columns = tableau_count − Σ_{i=1}^{tableau_count} non_empty(i)`
`non_empty(i) = 1 if column_height(i) > 0, else 0`

> **Canonical name:** registry 与下游 GDD 使用 `empty_tableau_column_count` 指代此输出；本节的 `empty_tableau_columns` 是同一值的别名，范围 0–8，语义不变。

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| tableau_count | int | 8（constant） | Core Rules §2 | 桌面列数 |
| non_empty(i) | int | 0 或 1 | 模型状态 | 第 i 列是否非空 |
| column_height(i) | int | 0–52 | 模型状态 | 第 i 列当前张数（无上限，I5） |
| empty_tableau_columns | int | 0–8 | Derived | 当前空列数量，只读 |

**Output Range:** 0–8。发牌刚完成时为 0。**`empty_tableau_columns = 8` 不等于胜利**：48 张在回收单元、4 张在自由单元、桌面全空时，空列为 8 但 `game_complete = false`。

**Example:** 8 列中有 3 列为空 → empty_tableau_columns = 8 − 5 = 3。

### The foundation_height formula

The foundation_height formula is defined as:

`foundation_height(f) = top_rank(f) if foundation_empty(f) = false, else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| f | int | 1–4 | Core Rules §2 | 回收单元序号，每个绑定一种花色 |
| foundation_empty(f) | bool | true/false | 模型状态 | 该回收单元是否为空 |
| top_rank(f) | int | 1–13 | 模型状态 | 顶牌等级；由 I4 保证等于张数 |
| foundation_height(f) | int | 0–13 | Derived | 已接收牌数，等于下一张应收等级 |

**Output Range:** 0–13。0 = 空；13 = 完成（顶牌为 K）。因 I4 要求从 A 起连续无缺口，张数恒等于顶牌等级，可 O(1) 读出而不必数牌。

**Example:** ♥ 回收单元顶牌为 9♥ → foundation_height = 9，下一张应收 10♥。

### The game_completion formula

The game_completion formula is defined as:

`foundation_card_total = Σ_{f=1}^{4} foundation_height(f)`
`game_complete = (foundation_card_total = deck_size)`

> **Canonical name:** registry 与下游 GDD 使用 `game_completion` 指代此布尔输出；本节的 `game_complete` 是同一值的别名，范围 {0,1}，语义不变。

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| foundation_height(f) | int | 0–13 | Derived | 第 f 个回收单元的高度 |
| foundation_card_total | int | 0–52 | Derived | 已归位牌数 |
| deck_size | int | 52 | Derived（deck_size） | 完成阈值 |
| game_complete | bool | true/false | Derived | 整局是否完成 |

**Output Range:** `foundation_card_total` 0–52；`game_complete` 仅在总数为 52 时为 true。等价写法 `∀f : foundation_height(f) = 13` 只在 I1–I4 全部成立时同值；完成状态必须在不变式校验通过后计算。

**Example:** 13 + 13 + 13 + 9 = 48 ≠ 52 → game_complete = false。

**配套派生量（只读，供 UI/统计）：** `completion_fraction = foundation_card_total / deck_size`，使用 `double`，范围 0.0–1.0，步长 1/52 ≈ 0.019231；比较容差 1e-9。仅用于显示，胜负判定必须比较整数。

### The card_color formula

The card_color formula is defined as:

`card_color(suit) = Red if suit ∈ {diamonds, hearts}; Black if suit ∈ {clubs, spades}`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| suit | enum | {clubs, diamonds, hearts, spades} | Core Rules §1 | 花色，牌的标识之一 |
| card_color(suit) | enum | {Red, Black} | Derived | 牌面颜色，由花色唯一决定 |

**Output Range:** 恰好 2 个取值，覆盖 4 种花色；每种颜色对应 2 种花色 = 26 张牌。定义域之外的令牌必须校验失败，不得落入默认分支。

**Example:** hearts → Red；spades → Black；clubs → Black。

### 常量与可调性

本系统零可调参数：4、8、13、52 全部由“原版规则，零改动”支柱固定。实现中 `suit_count`、`rank_count_per_suit`、`tableau_count`、`free_cell_count` 应各定义一次并由上述公式引用，不得在各处重复字面量。

## Edge Cases

- **If 反序列化校验失败（I1–I4 任一不成立）**: 拒绝加载，保持调用前的模型状态，并报告具体违规项；不得部分加载或自动修复。
- **If 花色令牌不在四种之内**: 操作失败；`card_color` 不得落入默认分支返回 Black。未知花色会让 Rules 的异色判定在错误数据上给出“合法”结论。
- **If 发牌时模型非空**: 发牌整体失败，不写入任何列。
- **If 发牌序列不足 52 张、超过 52 张或含重复牌**: 在写入任何列之前拒绝；不留半发牌状态。
- **If 发牌后高度合计 ≠ 52，或第 8 列结束下标 ≠ 51**: 视为实现错误；原子回滚并报告断言失败。
- **If 向已占用的自由单元追加**: 失败，原因“容量已满”；模型不变。
- **If 向顶牌为 K 的回收单元追加**: 失败；若待追加的牌已在模型中，原因优先为“牌已存在”（`DUPLICATE_CARD`），否则才为“容量已满”（`CAPACITY_FULL`）。不是花色或连续性错误。
- **If 向空回收单元追加非 A，或向非空回收单元追加非下一等级**: 失败，原因“等级不连续”。
- **If 向回收单元追加花色不匹配的牌**: 失败，原因“花色不匹配”。
- **If 在未发牌的模型上执行非发牌操作**: 失败，原因“模型未发牌”；先完成发牌。
- **If 从空牌堆移除顶牌**: 失败，原因“空堆”。
- **If 从桌面列请求移除 k 张但 k > 列高**: 失败，模型不变。
- **If 调用方请求按任意牌列表移除**: 接口不存在，调用失败；模型只接受顶部连续后缀 `(column, k)`，不判断这 k 张是否构成合法序列。
- **If 把已在模型中的牌再次追加**: 失败，原因“牌已存在”；第二次追加不改变任何牌堆。
- **If 源与目标相同**: 失败，不产生状态变化。
- **If 快照或存档的 schema 版本不匹配**: 拒绝恢复，保留当前状态；不得尝试迁移。
- **If `empty_free_cells` 计算结果为负**: 说明 I3 已被破坏；报告错误，不 clamp 为 0。
- **If `empty_tableau_columns = 8`**: 合法状态，不等于胜利；胜负只能由 `game_complete` 判定。
- **If `completion_fraction` 的浮点值等于 1.0**: 仅用于显示；胜负判定必须比较整数。
- **If 拖动被取消或释放到非法位置**: 模型不发生任何变更；反馈由 Drag & Drop Interaction 与 Rules 处理。
- **If 存档包含背面朝下标志**: 视为不兼容并拒绝；模型没有 `face_up` 状态。
- **If 多个系统并发写模型**: 写入必须串行化；模型不保证并发安全，由 Game State Machine 或 Drag & Drop Interaction 负责排序。

## Dependencies

**上游依赖**：无。Card & Deck Model 是依赖图的根。

**下游依赖（依赖本系统的系统）**：

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Deal Generator | Hard | 传入洗牌后的 52 张牌顺序，调用发牌操作 | 洗牌与局号属于 Deal；模型只存储 |
| Rules & Move Validator | Hard | 读取模型；判定合法后调用原子操作 | Rules 拥有合法性；模型不回调 Rules |
| Game State Machine | Hard | 新局/重开/胜负状态；读取模型 | State 拥有生命周期 |
| Drag & Drop Interaction | Hard | 命中测试读取；经 Rules 批准后写入 | 输入与反馈归 Interaction |
| Undo History | Hard | 快照/恢复 | 快照必须完整且不可变 |
| Hint System | Soft | 只读模型 | 无提示仍可玩，但不满足原版辅助 |
| Statistics & Records | Soft | 只读计数与完成状态 | 统计缺失不影响牌局 |
| Table Rendering & Layout | Hard | 只读牌堆与牌 | 无画面则不可玩 |
| Save/Resume Game | Soft | 序列化/反序列化 | 无存档仍可玩，但不符合原版行为 |
| Toolbar & Menus | Soft | 通过 State 调用命令 | 无工具栏仍可用快捷键；MVP 仍需要 |
| Animation & Feedback | Soft | 读取移动结果 | 缺省不影响规则 |
| Audio & SFX | Soft | 读取移动结果 | 可选 |
| Settings | None | 无直接交互 | — |
| Readability & Accessibility | Soft | 只读模型 | 缺省不影响核心循环 |

**接口契约**：

- **读**：任何下游系统都可以读取；读操作不得修改状态。
- **写**：只有经 Rules & Move Validator 批准后的调用方（Drag & Drop Interaction、Game State Machine）可以写入；Deal Generator 仅在初始化时写入；Undo History 与 Save/Resume Game 通过恢复/反序列化写入。
- **移动结果**：`{source, destination, cards, success, failure_reason}`。成功时 `cards` 为按源顺序（自底向顶）移动的牌；失败时 `cards` 为空、`success = false`，`source`/`destination` 为请求值，模型不变。供撤销、动画、音效与统计消费。
- **读取**：返回不可变副本；调用方不能通过返回值修改模型。
- **禁止**：下游直接改写内部牌堆数组；模型调用 Rules（避免循环依赖）；模型产生随机数。

**双向一致性**：下游 GDD 尚未编写；每个下游 GDD 的 Dependencies 节必须列出本系统，并引用本节接口契约。发现单向依赖时在此处或下游 GDD 中补齐。

**Provisional assumptions（下游未设计，先假定）**：

- Rules & Move Validator 将提供 `can_move(source, destination, count)` 谓词与 `legal_destinations(card)` 查询；模型假定调用方在变更前使用它们。
- Save/Resume Game 将拥有序列化格式与版本号；模型提供带 schema 版本的快照。
- Undo History 将保存不可变快照；模型本身不实现撤销。

## Visual/Audio Requirements

无。本系统是数据层，不产生视觉或音频需求。视觉由 Table Rendering & Layout 读取模型后负责；音效由 Audio & SFX 读取移动结果后负责。

## Game Feel

无。数据层不直接参与手感；手感由 Drag & Drop Interaction（输入与落位）与 Animation & Feedback（动效时长 ≤150ms）负责。本系统只保证操作结果的确定性，使手感层可以稳定复现。

## UI Requirements

无直接 UI。模型为 Toolbar & Menus、Table Rendering & Layout、Statistics & Records 提供只读数据；工具栏命令通过 Game State Machine 调用模型。

## Cross-References

| 系统 | 引用内容 | 状态 |
|------|----------|------|
| Deal Generator | 提供 52 张发牌顺序；本文件的 `deal_start_index` 定义切片方向 | GDD 已完成（design/gdd/deal-generator.md） |
| Rules & Move Validator | 拥有合法性、序列与超级移动；本模型不回调 | GDD 已完成（design/gdd/rules-move-validator.md） |
| Save/Resume Game | 拥有序列化格式与版本；模型提供带 schema 版本的快照 | GDD 未编写 |
| Table Rendering & Layout | 只读牌堆与牌；回收单元的视觉顺序由该 GDD 决定 | GDD 已完成（design/gdd/table-rendering-layout.md） |

## Acceptance Criteria

### 核心规则与不变量

- **GIVEN** 创建标准牌组，**WHEN** 枚举全部牌，**THEN** 恰好 52 张、每种花色 13 张、每个 `(suit, rank)` 恰好一次、无重复；牌对象不包含 `face_up` 或任何可见性字段。
- **GIVEN** 52 张互异的发牌序列，**WHEN** 从空模型发牌，**THEN** 第 1–4 列各 7 张、第 5–8 列各 6 张；每列切片第一张为列底、最后一张为列顶；合计 52；所有牌可读。
- **GIVEN** 已发牌模型，**WHEN** 读取全部 16 个牌堆，**THEN** 恰好 8 个桌面列、4 个自由单元、4 个回收单元；顶牌等于最后元素、count 等于长度、isEmpty 当且仅当 count = 0；4 个回收单元绑定 4 种不同花色；发牌后回收单元高度均为 0。
- **GIVEN** 已发牌模型，**WHEN** 任意成功操作完成后枚举全部牌，**THEN** 恰好 52 张、无重复、每张恰好属于一个牌堆（I1/I2）。
- **GIVEN** 自由单元已有 1 张牌，**WHEN** 尝试再放入一张，**THEN** 失败 `CAPACITY_FULL`，状态指纹不变（I3）。
- **GIVEN** 回收单元，**WHEN** 追加正确花色的下一等级，**THEN** 成功且高度加 1；**WHEN** 追加错误等级、错误花色或第 14 张，**THEN** 分别失败 `RANK_DISCONTINUITY`、`SUIT_MISMATCH`、`CAPACITY_FULL`，状态不变（I4）。
- **GIVEN** 桌面列，**WHEN** 追加任意数量（包括使列高达到 14），**THEN** 不发生容量失败（I5）。
- **GIVEN** 有效状态，**WHEN** 任一操作失败，**THEN** 完整状态指纹与操作前逐字节相同（原子性）。
- **GIVEN** 任意读取接口，**WHEN** 调用方修改返回的集合或牌对象，**THEN** 模型状态指纹不变（返回不可变副本）。

### 操作

- **GIVEN** 未发牌的空模型，**WHEN** 执行一次合法发牌，**THEN** 状态匹配发牌分布与方向，且通过 I1–I5。
- **GIVEN** 未发牌模型，**WHEN** 请求任何非发牌操作，**THEN** 失败“模型未发牌”，状态不变。
- **GIVEN** T1 高 7、顶牌为 C7，**WHEN** 原子移动顶部 2 张（C6、C7）到空桌面列，**THEN** 结果为 `[C6, C7]`（按源顺序），T1 高 5、顶牌 C5，目标列高 2、顶牌 C7。
- **GIVEN** 目标自由单元已被占用，**WHEN** 原子移动一张牌进去，**THEN** 失败 `CAPACITY_FULL`，源与目标均不变。
- **GIVEN** 结构性合法但 Rules 非法的移动（C7 移到 DA 上），**WHEN** 直接调用模型原子移动，**THEN** 移动成功；证明模型不做合法性判断。
- **GIVEN** 有效状态 S，**WHEN** 快照 → 成功移动 → 恢复，**THEN** 状态指纹等于 S。
- **GIVEN** 有效状态，**WHEN** 序列化后反序列化到新模型，**THEN** 状态指纹相同，且顶牌/count/isEmpty 读取一致。
- **GIVEN** 有效当前状态，**WHEN** 反序列化一个违反 I1–I4 的状态，**THEN** 拒绝并报告具体违规项，当前状态不变，无部分加载。
- **GIVEN** 相同 52 张顺序与相同操作脚本，**WHEN** 两个模型独立运行，**THEN** 每一步序列化状态相同（模型不消费随机数）。
- **GIVEN** 一次成功移动，**WHEN** 检查结果，**THEN** `success = true`、source/target 正确、`cards` 按源顺序；失败时 `success = false`、`cards` 为空、模型不变。
- **GIVEN** 源与目标相同，**WHEN** 请求原子移动，**THEN** 失败 `SAME_SOURCE_AND_TARGET`，状态不变。
- **GIVEN** 源列高 6，**WHEN** 原子移动请求 k = 7，**THEN** 失败，状态不变。
- **GIVEN** 把已在模型中的牌再次追加（通过内部步骤或故障注入），**WHEN** 执行，**THEN** 失败 `DUPLICATE_CARD`，状态不变。

### 公式

- **GIVEN** `suit_count = 4`、`rank_count_per_suit = 13`，**WHEN** 计算 `deck_size`，**THEN** 结果为 52，等于创建的牌组张数与 `game_complete` 的阈值。
- **GIVEN** `deck_size = 52`、`tableau_count = 8`，**WHEN** 计算 `deal_column_height(i)`，**THEN** i = 1–4 返回 7、i = 5–8 返回 6，合计 52；发牌后实际列高为 `7,7,7,7,6,6,6,6`。
- **GIVEN** 上述列高，**WHEN** 计算 `deal_start_index(i)`，**THEN** 结果为 `0,7,14,21,28,34,40,46`；第 8 列占用下标 46–51；发牌后每列底牌与起始下标一致。
- **GIVEN** 发牌后状态，**WHEN** 计算 `empty_free_cells`，**THEN** 为 4；FC1、FC3 被占用时为 2；四个全占用时为 0。**GIVEN** 故障注入使占用数超过 4，**WHEN** 计算，**THEN** 报告错误而不是返回 clamp 后的 0。
- **GIVEN** 发牌后状态，**WHEN** 计算 `empty_tableau_columns`，**THEN** 为 0；3 列为空时为 3；48 张在回收单元、4 张在自由单元、8 列全空时为 8，且 `game_complete = false`。
- **GIVEN** 回收单元，**WHEN** 计算 `foundation_height`，**THEN** 空时为 0；顶牌为 9♥ 且共 9 张时为 9；顶牌为 K 且共 13 张时为 13。
- **GIVEN** 回收单元高度 `13,13,13,9`，**WHEN** 计算 `game_completion`，**THEN** `foundation_card_total = 48`、`game_complete = false`；高度全为 13 时总数为 52、`game_complete = true`；判定只比较整数。
- **GIVEN** `foundation_card_total = 22`，**WHEN** 计算 `completion_fraction`，**THEN** 结果为 22/52 ≈ 0.423076923（`double`，容差 1e-9）；总数为 52 时为 1.0，且仅用于显示。
- **GIVEN** 花色 D、H，**WHEN** 计算 `card_color`，**THEN** 为 Red；花色 C、S 为 Black；标准牌组中红黑各 26 张；未知花色令牌必须失败，不得默认返回 Black。
- **GIVEN** 实现源码，**WHEN** 检查常量，**THEN** `suit_count=4`、`rank_count_per_suit=13`、`tableau_count=8`、`free_cell_count=4` 各定义一次并由公式引用，无重复字面量（静态检查）。

### 失败路径

- **GIVEN** 有效当前状态，**WHEN** 反序列化以下任一状态：51 张、重复牌、同一张牌出现在两个牌堆、自由单元含 2 张、回收单元等级缺口、回收单元花色错误，**THEN** 拒绝为 `INVALID_INVARIANT` 并报告具体违规项，当前状态不变。
- **GIVEN** 花色令牌不在四种之内，**WHEN** 创建牌或反序列化，**THEN** 校验失败，`card_color` 不被成功调用，不返回默认 Black。
- **GIVEN** 已发牌模型，**WHEN** 再次发牌且序列合法，**THEN** 以新序列整体替换旧牌面，全部列一次性更新，成功后模型仍满足 I1–I5；**WHEN** 序列非法，**THEN** 整体失败，旧牌面逐字节不变，任何列都不写入。
- **GIVEN** 空模型，**WHEN** 发牌序列为 51 张、53 张或含重复牌，**THEN** 在写入任何列之前拒绝，模型保持发牌前状态。
- **GIVEN** 回收单元顶牌为 K，**WHEN** 追加另一张同花色牌，**THEN** 失败；若该牌已在模型中，原因为 `DUPLICATE_CARD`，否则为 `CAPACITY_FULL`。
- **GIVEN** 快照或存档的 schema 版本不匹配，**WHEN** 请求恢复，**THEN** 拒绝为 `INVALID_SCHEMA`，当前状态保留，不尝试迁移。
- **GIVEN** 存档包含 `face_up` 或等价的可见性字段，**WHEN** 反序列化，**THEN** 拒绝为 `INVALID_SCHEMA`，不映射到任何状态。
- **GIVEN** 8 个桌面列全空、48 张在回收单元、4 张在自由单元，**WHEN** 计算完成状态，**THEN** 状态合法、`empty_tableau_columns = 8`、`game_complete = false`。
- **GIVEN** 拖拽被取消或释放到非法目标，**WHEN** 交互结束且未调用模型操作，**THEN** 模型指纹不变（调用方/集成测试）。

### 跨系统

- **GIVEN** 完整路径 Drag & Drop → Rules → Model，**WHEN** 玩家尝试 C7 移到 DA 上（结构性合法、Rules 非法），**THEN** Rules 拒绝、模型操作未被调用、模型不变；与“直接调用模型会成功”的单元事实成对，证明门在调用方。
- **GIVEN** Rendering、Statistics、Hint 或 Accessibility 消费者获得模型数据，**WHEN** 尝试修改返回集合或牌对象，**THEN** 只修改私有副本或抛出，模型指纹不变。
- **GIVEN** 一次成功移动，**WHEN** 检查结果载荷，**THEN** 包含 `success`、`source`、`destination`、按源顺序的 `cards` 与 `failure_reason`；失败载荷不报告已应用的变更。
- **GIVEN** 模型程序集或命名空间，**WHEN** 运行依赖检查，**THEN** 不存在对 Rules & Move Validator 的编译期或运行期引用（静态检查）。
- **GIVEN** 模型的公开接口，**WHEN** 检查可写入口，**THEN** 只有原子移动、快照/恢复与反序列化能改变状态，不存在直接数组/索引写入或公开 setter（接口检查配合不可变副本测试）。

## Open Questions

| 问题 | Owner | 目标解决时间 |
|------|-------|--------------|
| 局号洗牌是否与混合参考（Windows 7/XP 1–1,000,000）完全一致（属于 Deal Generator） | Deal Generator GDD + technical-director | 首个可运行版本前 |
| 回收单元的视觉顺序（索引→花色映射） | Table Rendering & Layout | 该 GDD 编写时 |
| 序列化格式与版本号 | Save/Resume Game + architecture ADR | Save/Resume GDD 编写时 |
| 模型 API 的 C# 命名与类型映射 | lead-programmer / `/create-architecture` | 架构阶段 |
| 是否需要模型暴露移动事件，还是由调用方广播 | `/create-architecture` | 架构阶段 |
