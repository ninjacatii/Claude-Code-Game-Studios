# Deal Generator

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 原版规则，零改动

## Summary

Deal Generator 把局号 1–1,000,000 转换为确定的 52 张牌序列，并转置为 Card & Deck Model 的列优先发牌顺序。它实现 Microsoft FreeCell 的 LCG 与抽取算法，保证同一局号在任何机器上产生同一副牌；不读取时间或引擎 RNG，不判断合法性，也不持有牌局状态。

> **Quick reference** — Layer: `Core` · Priority: `MVP` · Key deps: `Card & Deck Model`

## Overview

Deal Generator 把玩家选择的局号转换为一副确定的 52 张牌序列，并按 Card & Deck Model 的列优先顺序交给模型发牌。同一个局号在任何机器、任何次运行上产生同一副牌；不同局号预期产生不同牌局（该性质由 1–1,000,000 的全量验收测试验证，不由公式单独保证）。它是唯一允许产生随机性的系统，但随机性完全由局号决定，不依赖时间、进程或引擎版本。它不判断合法性，也不持有牌局状态——只回答“这副牌是什么”。

## Detailed Design

### Core Rules

**1. 输入与输出**

- 输入：局号 N，整数 1–1,000,000（含端点）。
- 输出：52 张牌的序列，按 Card & Deck Model 的**列优先**顺序排列：第 1 列自底向顶，然后第 2 列……第 8 列。
- 输出不可变；调用方把它交给 Card & Deck Model 的发牌操作。内部可用 0–51 的 `card_id` 表示牌，对外 `DealSequence` 仍是 52 张 `(suit, rank)`，两者由 `deck_index` 一一对应。
- 生成器是纯函数：相同 N 必得相同输出；不读写时间、进程、引擎 RNG 或全局状态。

**2. 牌序与索引**

- 标准牌组按“等级优先、花色 C→D→H→S”排列：AC, AD, AH, AS, 2C, 2D, 2H, 2S, …, KS。
- `deck_index(rank, suit) = (rank − 1) × 4 + suit_index`，suit_index C=0、D=1、H=2、S=3。
- 例：AC=0，AS=3，2C=4，KS=51。

**3. Microsoft LCG**

- 状态 `state` 为 31 位无符号整数（0 ≤ state < 2^31）；初始 `state = N`。
- 每次抽取：`state = (214013 × state + 2531011) mod 2^31`；`rand = floor(state / 2^16)`，范围 0–32767。
- 必须使用 64 位中间值或等价的无溢出实现；不得使用浮点、`System.Random` 或引擎 RNG。
- **不要修正模偏差**：`rand % (52 − i)` 的偏差是原版算法的一部分；使用拒绝采样会改变局号结果。

**4. 抽取与发牌**

- 初始化 `deck = [AC, AD, AH, AS, 2C, …, KS]`（52 张）。
- 对 i = 0…51：
  - `j = rand % (52 − i)`；
  - 交换 `deck[j]` 与 `deck[51 − i]`；
  - `dealt[i] = deck[51 − i]`（取出的牌）。
- 行优先分发到 8 列：`dealt[0..7]` 为第 0 行（每列一张，列 1–8），`dealt[8..15]` 为第 1 行，……，`dealt[48..51]` 为最后一行（列 1–4）。
- 每列自底向顶：列 c（1-based）的第 r 张（r=0 为底）是 `dealt[r × 8 + (c − 1)]`。
- 列高：c=1…4 为 7，c=5…8 为 6（来自 Card & Deck Model 的 `deal_column_height`）。

**5. 列优先输出**

- 按列 1…8、每列自底向顶，把上述牌拼接为 52 张序列。
- 第 c 列的起始下标使用 Card & Deck Model 的 `deal_start_index(c)`。
- 输出必须与 Card & Deck Model 的 `deal(sequence)` 契约完全一致：第 c 列取连续切片，切片第一张为列底、最后一张为列顶。
- 这是行优先发牌与列优先模型之间的唯一转置点；转置错误会让牌面看起来“洗错了”。

**6. 局号范围与拒绝**

- 有效范围 1–1,000,000（含端点）；1–32,000 为 Windows 95 原版子集；功能基线见 `game-concept.md` §Reference Version。
- 生成器只接受整数 N。N < 1 或 N > 1,000,000：拒绝，错误 `DEAL_NUMBER_OUT_OF_RANGE`；不产生部分输出。非整数输入的解析属于 Toolbar & Menus / Game State Machine，格式错误在那里映射为 `INVALID_DEAL_NUMBER`，不会传入生成器。
- 不检查可解性：#11982 等不可解局号照常生成；跳过它们会破坏局号与 Windows 的一致性。
- 不支持“随机新局”；随机性只能来自局号。UI 的“下一局/上一局”由 Toolbar & Menus / Game State Machine 决定。

**7. 性能与并发**

- O(52) 时间、O(52) 空间；单次生成可忽略不计。
- 纯函数，天然线程安全；若缓存结果，缓存必须不可变，且键为 N。

### States and Transitions

| 状态 | 进入 | 离开 | 说明 |
|------|------|------|------|
| Idle | — | generate(N) | 无持久状态 |
| Generating | N 合法 | 52 次抽取 + 转置完成 | 不读外部状态 |
| Done | 输出完整 | 返回调用方 | 输出不可变 |
| Error | N 非法 | 返回错误 | 无部分输出 |

同一 N 的重复调用不改变状态，输出逐字节相同。生成过程中任何实现断言失败都不得返回半副牌；整体失败。

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|------|------|------|------|------------|
| Card & Deck Model | 下游 | 52 张列优先序列 | 发牌后的模型 | Generator 拥有序列；Model 拥有存储 |
| Game State Machine | 下游 | 新局/重开请求 + 局号 | 发牌结果 | State 拥有生命周期；Generator 无状态 |
| Toolbar & Menus | 下游 | 玩家选择的局号 | 生成请求 | Toolbar 拥有输入；经 State 调用 |
| Statistics & Records | 间接 | 无直接接口 | 通过 State 的 GameStarted 记录 | Stats 不记录局号；Generator 不直接交互 |
| Save/Resume Game | 下游 | 局号 | 恢复时重新生成 | Save 拥有格式；Generator 无状态 |
| Hint System | 无 | — | — | 不直接交互 |
| Table Rendering / Animation / Audio | 无 | — | — | 通过模型间接消费 |
| Settings / Accessibility | 无 | — | — | 不直接交互 |

**接口契约**：`generate(deal_number) -> DealSequence` 或错误。`DealSequence` 是 52 张、无重复、列优先、不可变；调用方不得修改。Generator 不调用 Card & Deck Model；由 Game State Machine 或调用方把序列交给模型。

## Formulas

本节公式全部是纯整数函数：不读取时间、进程状态、`System.Random` 或引擎 RNG；唯一输入是 N 及其派生值。不得使用浮点。

### The deck_index formula

The deck_index formula is defined as:

`deck_index(rank, suit) = (rank − 1) × 4 + suit_index(suit)`

`suit_index(C) = 0, suit_index(D) = 1, suit_index(H) = 2, suit_index(S) = 3`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| rank | int | 1–13 | 牌标识 | 1=A, 2–10, 11=J, 12=Q, 13=K |
| suit | enum | {C, D, H, S} | 牌标识 | 等级优先排序中的花色 |
| suit_index | int | 0–3 | 本 GDD 固定映射 | C=0, D=1, H=2, S=3 |
| deck_index | int | 0–51 | 本公式 | 有序牌组 `[AC, AD, AH, AS, 2C, …, KS]` 中的 0-based 位置 |

**Output Range:** 0–51（含端点）。最小 0 在 `(rank=1, suit=C)`；最大 51 在 `(rank=13, suit=S)`。公式不校验越界输入；调用方必须保证 rank/suit 合法。

**Example:** AC → 0；AS → 3；2C → 4；KS → `(13−1)×4+3 = 51`。

### The lcg_next formula

The lcg_next formula is defined as:

`lcg_next(state) = (214013 × state + 2531011) mod 2^31`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| state | uint31 | 0–2,147,483,647 | 初始为 N；之后为上一步输出 | 31 位 LCG 状态 |
| 214013 | const int | — | 本 GDD（Microsoft FreeCell） | 乘数，不得调整 |
| 2531011 | const int | — | 本 GDD（Microsoft FreeCell） | 增量，不得调整 |
| 2^31 | const int | 2,147,483,648 | 本 GDD | 模数 |
| lcg_next | uint31 | 0–2,147,483,647 | 本公式 | 推进后的状态 |

**Output Range:** 0–2,147,483,647。该映射在完整周期上是 2^31 个状态的置换（数学性质；运行时只验证向量，不穷举 2^31 状态）。原始乘积最大约 4.59×10^14，超过 32 位；必须使用 64 位中间值，或用 `uint` 回绕后与 `0x7FFFFFFF` 掩码。带符号 `int` 不回绕/不掩码会产生负状态，从而改变整个局号集合。

**Example:** `lcg_next(1) = 2,745,024`；`lcg_next(2,745,024) = 1,210,316,419`。

### The lcg_rand formula

The lcg_rand formula is defined as:

`lcg_rand(state) = floor(state / 2^16)`，等价于 `state >> 16`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| state | uint31 | 0–2,147,483,647 | `lcg_next` 的输出 | 已推进的状态 |
| 2^16 | const int | 65,536 | 本 GDD | 右移位数；丢弃低 16 位 |
| lcg_rand | int | 0–32,767 | 本公式 | 一次抽取消费的随机值 |

**Output Range:** 0–32,767（含端点）。这是纯投影，不推进状态。丢弃低 16 位是原版行为，不得“修正”。

**Example:** `lcg_rand(2,745,024) = 41`；`lcg_rand(1,210,316,419) = 18,467`。

**顺序约束（关键）**：抽取循环必须**先推进、后读取**。对第 i 次抽取：`state(i+1) = lcg_next(state(i))`，然后 `rand(i) = lcg_rand(state(i+1))`，且 `state(0) = N`。反过来“先读后推进”会让 `rand(0) = floor(N / 65536)`，产生完全不同的局号集合，且表面上同样合理。

### The draw_index formula

The draw_index formula is defined as:

`draw_index(i, rand) = rand mod (52 − i)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| i | int | 0–51 | 抽取循环 | 0-based 抽取序号 |
| rand | int | 0–32,767 | `lcg_rand` | 本次抽取的随机值 |
| j | int | 0–(51−i) | 本公式 | 要换到收缩牌组尾部的牌的下标 |

**Output Range:** 0 到 `51−i`。i=0 时 0–51；i=51 时除数 1，输出恒为 0（最后一张被强制）。i=52 会除以零，循环上界是唯一防线。**模偏差是原版的一部分**：32,768 = 52×630 + 8，余数 0–7 出现 631 次、8–51 出现 630 次；不得用拒绝采样修正。

**Example:** `draw_index(0, 41) = 41`；`draw_index(1, 18467) = 5`；`draw_index(51, 18467) = 0`。

### The column_major_sequence formula

The column_major_sequence formula is defined as:

`sequence[deal_start_index(c) + r] = dealt[8 × r + (c − 1)]`
for all `c ∈ 1..8`, `r ∈ 0..(deal_column_height(c) − 1)`

配套使用 Card & Deck Model 的已注册公式：
`deal_column_height(c)` → `7,7,7,7,6,6,6,6`
`deal_start_index(c)` → `0,7,14,21,28,34,40,46`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| c | int | 1–8 | 本公式 | 列号，1-based，从左到右 |
| r | int | 0–6 | 本公式 | 列内行号，0 = 列底 |
| dealt | int[52] | 每项 0–51 | 抽取循环 | 行优先抽取顺序的牌 |
| deal_column_height(c) | int | 6 或 7 | Card & Deck Model（registry） | 第 c 列张数 |
| deal_start_index(c) | int | 0–46 | Card & Deck Model（registry） | 第 c 列底牌在输出序列中的下标 |
| sequence | int[52] | 每项 0–51 | 本公式 | 列优先输出，可直接交给 `deal(sequence)` |

**Output Range:** 恰好 52 个槽位，索引 0–51，每个恰好写入一次。`c=1, r=0` 写 0；`c=8, r=5` 写 51。合计 `4×7 + 4×6 = 52`。若任一侧未覆盖 52 个位置或出现重叠，序列就有重复或空洞，发牌损坏。

**逆映射（用于测试/调试转置错误）**：找到满足 `deal_start_index(c) ≤ p < deal_start_index(c) + deal_column_height(c)` 的唯一 c；则 `r = p − deal_start_index(c)`，`dealt_position = 8 × r + (c − 1)`。

**Example（N = 1）**：抽取循环得到 `dealt[0] = 41`（JD）、`dealt[1] = 5`（2D）。转置后 `dealt[0] → c=1, r=0 → sequence[0] = 41`；`dealt[1] → c=2, r=0 → sequence[7] = 5`。两张连续抽出的牌在输出中相隔 7 个槽位——这是真正的转置，不是复制。第 6 行：`dealt[48]→sequence[6]`、`dealt[49]→sequence[13]`、`dealt[50]→sequence[20]`、`dealt[51]→sequence[27]`。

## Edge Cases

- **If N < 1 或 N > 1,000,000**: 生成器返回 `DEAL_NUMBER_OUT_OF_RANGE`，不产生输出。**If 输入不是整数**: 由 UI 解析层拒绝为 `INVALID_DEAL_NUMBER`，生成器不会被调用。
- **If N = 1、32,000、32,001 或 1,000,000**: 均为合法边界；分别生成并纳入验收。
- **If LCG 使用带符号 32 位且未掩码**: 状态可能为负，后续模运算结果改变；实现必须使用 64 位中间值，或 `uint` 回绕后与 `0x7FFFFFFF` 掩码。
- **If 抽取顺序写成“先读后推进”**: `rand(0)` 会变成 `floor(N / 65536)`，所有局号结果改变；必须“先推进后读取”。
- **If 交换顺序写成“先取尾牌再交换”**: `dealt[i]` 会取到错误的牌；必须先交换 `deck[j]` 与 `deck[51−i]`，再取 `deck[51−i]`。
- **If 修正模偏差（拒绝采样）**: 局号结果改变；模偏差是原版行为，必须保留。
- **If 牌序写成花色优先或花色顺序不同**: 所有局号结果改变；必须等级优先、C→D→H→S。
- **If 行优先牌面直接当列优先输出**: 列内容错位；必须使用 `column_major_sequence` 转置，并用 #1 / #617 测试向量校验。
- **If 输出出现重复或缺失牌**: 内部断言失败，整体失败，不返回半副牌。
- **If N = 11982（已知不可解）**: 照常生成；不得跳过或替换。
- **If 不同 N 产生同一牌局**: 公式不保证唯一性；验收阶段穷举 1–1,000,000 检查。若发现碰撞，记录为已知事实，不修改算法（修改会破坏与 Windows 的一致性）。
- **If 缓存结果**: MVP 不要求缓存；若实现，缓存必须按 N 键控且不可变，不得跨算法版本复用。
- **If 算法版本变更**: 所有局号结果变化，属破坏性变更；必须提升存档/序列化版本并在变更日志中注明。
- **If 并发调用**: 纯函数，允许并发；若共享缓存，缓存需线程安全或不可变。
- **If UI 请求超出范围的“上一局/下一局”**: Generator 拒绝为 `DEAL_NUMBER_OUT_OF_RANGE`；Toolbar & Menus / Game State Machine 负责在边界停住或回绕，不把越界值传给 Generator。
- **If 存档只保存局号**: 恢复时重新生成；若算法版本不同，必须拒绝或迁移，不得静默生成不同牌局。
- **If 字符串输入带前导零或本地化数字**: 解析为整数由 UI 负责；`000123` 与 `123` 视为同一局号，格式错误由 UI 返回 `INVALID_DEAL_NUMBER`。

## Dependencies

**上游依赖**：Card & Deck Model（GDD 已完成）。锁定事实：`deck_size=52`、`tableau_count=8`、`deal_column_height` 为 7/7/7/7/6/6/6/6、`deal_start_index` 为 0/7/14/21/28/34/40/46、切片第一张为列底、所有牌正面朝上。

**下游依赖（依赖本系统的系统）**：

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Game State Machine | Hard | 新局/重开时调用 `generate(N)`，把序列交给 `deal()` | State 拥有生命周期；Generator 无状态 |
| Toolbar & Menus | Soft | 玩家输入局号，经 State 调用 | 边界处理归 Toolbar/State |
| Statistics & Records | 间接 | 无直接接口 | 通过 State 的 GameStarted 记录；不记录局号 |
| Save/Resume Game | Soft | 保存局号，恢复时重新生成 | 必须校验算法版本 |
| Hint System | None | 不直接交互 | — |
| Table Rendering / Animation / Audio | None | 通过模型间接消费 | — |
| Settings / Accessibility | None | 不直接交互 | — |

**接口契约**：

- `generate(deal_number) -> DealSequence | error`。
- `DealSequence`：52 张、无重复、列优先、不可变；每张牌标识为 `(suit, rank)`。
- Generator 不调用 Card & Deck Model；不持有状态；不读取时间、进程状态或 RNG。
- 生成器错误原因：`DEAL_NUMBER_OUT_OF_RANGE`（整数越界）、`INTERNAL_INVARIANT_VIOLATION`（内部断言失败）。`INVALID_DEAL_NUMBER` 属于 UI 解析层，不会由生成器返回。
- 算法版本固定为 “Microsoft FreeCell LCG v1”；变更算法必须提升版本号。

**Provisional assumptions（下游未设计，先假定）**：

- Game State Machine 在 `generate(N)` 成功后立即调用 `Card & Deck Model.deal(sequence)`。
- Toolbar & Menus 负责把用户输入解析为 1–1,000,000 的整数，并在边界处停止或回绕。
- Save/Resume Game 保存局号与算法版本；恢复时校验版本后重新生成。
- Statistics & Records 通过 State 的 GameStarted 记录局数/胜负，不记录局号；不做可解性判断。

**Registry references**：`deal_column_height` 与 `deal_start_index` 的 `referenced_by` 应追加 `design/gdd/deal-generator.md`（Phase 5b 更新）。

## Visual/Audio Requirements

无。纯逻辑系统，不产生视觉或音频需求；发牌后的画面由 Table Rendering & Layout 负责。

## Game Feel

无直接手感。生成是瞬时的；它保证每次发牌正确且可复现，使 Drag & Drop Interaction 与 Animation & Feedback 的手感不受数据错误干扰。

## UI Requirements

无直接 UI。Toolbar & Menus / Game State Machine 负责局号输入、解析、错误展示与上一局/下一局边界；Generator 只接受整数。

## Cross-References

| 系统 / 来源 | 引用内容 | 状态 |
|-------------|----------|------|
| Card & Deck Model | `deal_start_index`、`deal_column_height`、列优先收牌契约 | GDD 已完成 |
| Microsoft FreeCell 算法 | LCG、牌序、行优先发牌；#1 / #617 官方向量 | 已查证（freeCodeCamp / Rosetta Code） |
| Save/Resume Game | 保存局号与算法版本；恢复时重新生成 | GDD 未编写 |
| Game State Machine | 调用 `generate(N)` 并把序列交给模型 | GDD 已完成（design/gdd/game-state-machine.md） |

算法与官方向量来源：freeCodeCamp Rosetta Code — *Deal cards for FreeCell*（`https://raw.githubusercontent.com/freeCodeCamp/freeCodeCamp/57dca1e495aa6312864cc0a5038cc61e4fea8086/curriculum/challenges/english/22-rosetta-code/rosetta-code-challenges/deal-cards-for-freecell.md`），2026-10-24 查证。

## Acceptance Criteria

**测试约定**：牌记法 `A,2–9,T,J,Q,K` + `C,D,H,S`，`T=10`；`card_id = (rank−1)×4 + suit_index`，`C=0,D=1,H=2,S=3`（AC=0、AS=3、JD=41、KS=51）。列优先输出：列 1–4 各 7 张、列 5–8 各 6 张，起始下标 `0,7,14,21,28,34,40,46`。SHA-256 对 52 个列优先 card id 的字节（0–51）计算。#1 / #617 的向量来自公开算法；32,000 / 32,001 / 1,000,000 / 11,982 / 10,021 的 SHA-256 是复现 #1 / #617 的独立参考实现生成的派生固定值，采用前需用第二实现复核。

### 核心规则

- **GIVEN** N = 1，**WHEN** 调用 `generate(1)`，**THEN** 返回恰好 52 个 card id、全部互异、按列优先排列；若返回类型可变，修改返回集合后再次调用仍得到相同结果；失败调用不返回任何 52 张序列。
- **GIVEN** N = 1 的初始牌组，**WHEN** 第一次抽取前检查索引，**THEN** id 41 = JD、id 51 = KS；花色优先的牌组会把 3S 放在 41，因此“第一张是 JD 而不是 3S”证明牌序为等级优先 C→D→H→S。
- **GIVEN** N = 1、state(0) = 1，**WHEN** 执行前两次抽取，**THEN** state(1) = 2,745,024、rand(0) = 41；state(2) = 1,210,316,419、rand(1) = 18,467；dealt[0] = JD、dealt[1] = 2D；先读后推进的实现会得到 rand(0) = 0 并先发 AC，必须断言不是 AC。
- **GIVEN** N = 1 与初始牌组，**WHEN** 抽取循环执行 i = 0…51，**THEN** 恰好 52 次抽取；i=0 时 j=41，交换 deck[41]（JD）与 deck[51]（KS）后 dealt[0] = JD；i=1 时 j=5，交换 deck[5]（2D）与 deck[50]（KH）后 dealt[1] = 2D；i=51 时 j=0，dealt[51] = 6H；52 次后每张牌恰好出现一次。先取后换会得到 KS/KH，必须断言不是。
- **GIVEN** N = 1 的 dealt[0..51]，**WHEN** 按行优先铺入 8 列，**THEN** 行 0 = `JD 2D 9H JC 5D 7H 7C 5H`；行 1 = `KD KC 9S 5S AD QC KH 3H`；行 2 = `2S KS 9D QD JS AS AH 3C`；行 3 = `4C 5C TS QH 4H AC 4D 7S`；行 4 = `3S TD 4S TH 8H 2C JH 7D`；行 5 = `6D 8S 8D QS 6C 3D 8C TC`；行 6 = `6S 9C 2H 6H`。
- **GIVEN** N = 1 与起始下标/列高，**WHEN** 生成列优先输出，**THEN** `sequence[0..6] = JD KD 2S 4C 3S 6D 6S`；`[7..13] = 2D KC KS 5C TD 8S 9C`；`[14..20] = 9H 9S 9D TS 4S 8D 2H`；`[21..27] = JC 5S QD QH TH QS 6H`；`[28..33] = 5D AD JS 4H 8H 6C`；`[34..39] = 7H QC AS AC 2C 3D`；`[40..45] = 7C KH AH 4D JH 8C`；`[46..51] = 5H 3H 3C 7S 7D TC`；排序后恰好是 0–51。判别点：`sequence[1] = KD`（不是 2D）、`sequence[7] = 2D`（不是 5H）；行优先复制会得到另一组值。
- **GIVEN** N ∈ {1, 32,000, 32,001, 1,000,000}，**WHEN** 生成，**THEN** 全部成功、各 52 张互异、列高固定；`generate(32,000) ≠ generate(32,001)`；SHA-256 固定值：32,000 = `19f3bf37fbaa13999d30c4f46565891638b1cbfa1fe6193f63ca47d3dbf6277c`；32,001 = `eeb0f83fed7b101cccfe59c609a60341c34c48ba8ff1571df8c68749fdba22cc`；1,000,000 = `ee76389c0e442990d4d26db66c422e38a57cb443999f273957b4a379f63bf5c2`。
- **GIVEN** N ∈ {0, −1, 1,000,001, 2,147,483,647}，**WHEN** 调用 `generate`，**THEN** 返回 `DEAL_NUMBER_OUT_OF_RANGE`，无可观察的 52 张输出；随后 `generate(1)` 仍与金标一致（无残留状态）。
- **GIVEN** 命名参考机上的 Release 构建，**WHEN** 对 1–1,000 共 1,000 次调用计时，**THEN** 每次恰好 52 次 LCG 推进与 52 次抽取；总耗时 ≤ 100 ms，单次最坏 ≤ 1 ms；分配量不随 N 增长。
- **GIVEN** 生成器源码/IL，**WHEN** 扫描，**THEN** 不包含 `System.Random`、加密 RNG、Godot `RandomNumberGenerator`/`GD.Rand*`、`DateTime`/`Time`、`Stopwatch`、`Environment.TickCount`、`Guid.NewGuid` 或浮点运算；不存在“随机新局”入口；不依赖 Card & Deck Model。
- **GIVEN** 8 个线程各 100 次并发调用 `generate(N)`，**WHEN** 与串行结果比较，**THEN** 每个 N 的结果完全一致，无异常、无数据损坏。

### 公式

- **GIVEN** rank ∈ 1–13、suit ∈ {C,D,H,S}，**WHEN** 计算 `deck_index`，**THEN** AC=0、AD=1、AH=2、AS=3、2C=4、2S=7、TC=36、TS=39、JC=40、JD=41、QS=47、KC=48、KD=49、KH=50、KS=51；52 个组合恰好覆盖 0–51，每个一次。
- **GIVEN** state ∈ 0–2,147,483,647，**WHEN** 计算 `lcg_next`，**THEN** `lcg_next(0)=2,531,011`、`lcg_next(1)=2,745,024`、`lcg_next(2,745,024)=1,210,316,419`、`lcg_next(10,023)=99,662`、`lcg_next(2^31−1)=2,316,998`；输出永不为负。未掩码的带符号 32 位实现会得到 −937,167,229 与 −2,147,383,986，不得出现。
- **GIVEN** state ∈ 0–2,147,483,647，**WHEN** 计算 `lcg_rand`，**THEN** `lcg_rand(0)=0`、`lcg_rand(65,535)=0`、`lcg_rand(65,536)=1`、`lcg_rand(2,745,024)=41`、`lcg_rand(1,210,316,419)=18,467`、`lcg_rand(2^31−1)=32,767`；输出范围 0–32,767；同一 state 重复调用结果相同（不推进状态）。
- **GIVEN** i ∈ 0–51、rand ∈ 0–32,767，**WHEN** 计算 `draw_index`，**THEN** `draw_index(0,41)=41`、`draw_index(1,18,467)=5`、`draw_index(51,18,467)=0`、`draw_index(0,32,760)=0`、`draw_index(0,32,765)=5`、`draw_index(0,32,767)=7`；i=0 时对 32,768 个 rand 值穷举，输出均等于 `rand mod 52`，无拒绝、无重抽；余数 0–7 出现 631 次、8–51 出现 630 次（保留模偏差）；i=51 时除数 1、输出恒 0；循环不调用 i=52。
- **GIVEN** 合成 `dealt[k] = k`（k=0…51），**WHEN** 应用 `sequence[deal_start_index(c)+r] = dealt[8r+(c−1)]`，**THEN** `sequence = [0,8,16,24,32,40,48, 1,9,17,25,33,41,49, 2,10,18,26,34,42,50, 3,11,19,27,35,43,51, 4,12,20,28,36,44, 5,13,21,29,37,45, 6,14,22,30,38,46, 7,15,23,31,39,47]`；排序后为 0–51；逆映射对全部 52 个位置往返一致。

### 金标向量

- **GIVEN** N = 1，**WHEN** 生成，**THEN** 52 张列优先输出为：
  `JD KD 2S 4C 3S 6D 6S | 2D KC KS 5C TD 8S 9C | 9H 9S 9D TS 4S 8D 2H | JC 5S QD QH TH QS 6H | 5D AD JS 4H 8H 6C | 7H QC AS AC 2C 3D | 7C KH AH 4D JH 8C | 5H 3H 3C 7S 7D TC`
  且 SHA-256 = `e7fbac543fa086d3fd627d5e5bff3f70d0168b19fae5dc96495982a1634f6637`。
- **GIVEN** N = 617，**WHEN** 生成，**THEN** 52 张列优先输出为：
  `7D TD TH KD 4C 4S JD | AD 7S QC 5H QS TS KS | 5C QD 3H 9S 9C 2H KC | 3S AC 9D 3C 9H 5D 4H | 5S 6D 6S 8S 7C JC | 8C 8H 8D 7H 6H 6C | 2D AS 3D 4D 2C JH | AH KH TC JS 2S QH`
  且 SHA-256 = `82be423e6fb9866e52b1358cd1d4b61e697823267fdd4d246cb517951db2f880`。

### 陷阱（变异检测）

- **GIVEN** 先读后推进的实现，**WHEN** 生成 N = 1，**THEN** 第一张会是 AC（id 0）；正确结果必须是 JD（id 41）。
- **GIVEN** 先取后换的实现，**WHEN** 生成 N = 1，**THEN** 前两张会是 KS、KH；正确结果必须是 JD、2D。
- **GIVEN** 未掩码的带符号 32 位 LCG，**WHEN** 计算 state 2,745,024 与 10,023，**THEN** 会得到负数；正确结果必须是 1,210,316,419 与 99,662，且任何负 rand 都不得进入 `draw_index`。
- **GIVEN** 拒绝采样修正模偏差的实现，**WHEN** 生成 N = 10,021（首个 rand = 32,762，落在 32,760–32,767 偏差带；j=2，deck[2]=AH），**THEN** 正确输出 `sequence[0] = AH`，SHA-256 = `6de3cdef1d1c5562b3015e83f5edc08b8fcf57595c9f12957dd940e9e9bc95dd`；拒绝采样会消耗下一次 rand 并产生不同牌局。
- **GIVEN** 花色优先的牌组，**WHEN** 生成 N = 1，**THEN** 第一张会是 3S（id 11）；正确结果必须是 JD（id 41）。
- **GIVEN** 行优先复制而非转置，**WHEN** 生成 N = 1，**THEN** `sequence[1]` 会是 2D（id 5）、`sequence[7]` 会是 5H（id 18）；正确结果必须是 `sequence[1] = KD`（id 49）、`sequence[7] = 2D`（id 5）。

### 确定性与集成

- **GIVEN** N ∈ {1, 617, 32,000, 32,001, 1,000,000}，**WHEN** 在同一进程、两个独立进程、正序与倒序调用中分别生成，**THEN** 同一 N 的 52 个 card id 逐字节相同，并与金标/边界固定值一致；结果不受调用顺序或其他局号影响。
- **GIVEN** N = 1 与 N = 617 的生成序列，**WHEN** 传给 `Card & Deck Model.deal(sequence)`，**THEN** 模型接受、产生 8 列、列高 7/7/7/7/6/6/6/6、列内容与金标一致、52 张互异；32,000、32,001、1,000,000 同样被接受且无重复/缺失。
- **GIVEN** N = 11,982（已知不可解），**WHEN** 生成，**THEN** 正常接受、不跳过、不替换、不做可解性检查；SHA-256 = `9d503301074e9c28584e1c8c6198d686fc713da9b668e1d98ae878366548d025`。
- **GIVEN** N 遍历 1–1,000,000，**WHEN** 把每个牌局编码为 52 字节并精确比较，**THEN** 得到 1,000,000 个互异牌局、0 碰撞（重量级/夜间测试，CI 预算 ≤ 10 分钟；参考实现 54.3 秒完成）。
- **GIVEN** 实现了缓存（可选），**WHEN** 两次调用 `generate(N)`，**THEN** 缓存键仅为 N、缓存不可变或返回防御性副本、不跨算法版本复用；若 MVP 未实现缓存，标记 NOT APPLICABLE（不是通过）。

**跨系统说明**：局号输入的解析、前导零、`INVALID_DEAL_NUMBER` 展示、上一局/下一局边界，属于 Toolbar & Menus / Game State Machine 的故事，不在本 GDD 的验收条件内。

## Open Questions

| 问题 | Owner | 目标解决时间 |
|------|-------|--------------|
| 生成器的公开签名与返回类型（不可变序列 / 错误表示） | lead-programmer / `/create-architecture` | 架构阶段 |
| 是否暴露内部纯函数（`InternalsVisibleTo`）以直接测试公式 | lead-programmer | 架构阶段 |
| 是否实现局号结果缓存 | lead-programmer | 实现前 |
| 存档 / 算法版本迁移 | Save/Resume Game + architecture | Save/Resume GDD |
| 性能阈值（1,000 次 ≤ 100ms、单次 ≤ 1ms）的参考机 | technical-director | 架构阶段 |
