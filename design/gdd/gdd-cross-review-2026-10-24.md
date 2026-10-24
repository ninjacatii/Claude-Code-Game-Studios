# Cross-GDD Review Report

> **Date**: 2026-10-24
> **GDDs Reviewed**: 10 of 10 system GDDs
> **Not read**: none（5 个非 MVP 系统没有 GDD，不在评审集）
> **Focus**: full（Phase 3 完整；Phase 2 为定向人工核查，见 Coverage Note）
> **Systems Covered**: Card & Deck Model、Deal Generator、Rules & Move Validator、Game State Machine、Drag & Drop Interaction、Table Rendering & Layout、Undo History、Hint System、Statistics & Records、Toolbar & Menus

## Coverage Note

原计划并行运行 Phase 2（跨 GDD 一致性）与 Phase 3（游戏设计整体性）。Phase 3 已完成并交付完整报告；Phase 2 专用代理在输出 32k token 上限处失败，未交付完整报告。主会话随后做了一次**定向人工一致性核查**：依赖双向性（2a）、陈旧引用（2c）、AC 交叉（2f）以及已知跨系统接口。**完整的 2b/2d/2e 覆盖标记为 `NOT ASSESSED`**；本报告中的 Blocking 均直接来自 GDD 原文，并与 Phase 3 的交叉阅读一致。

---

## Consistency Issues

### Blocking（must resolve before architecture begins）

#### 🔴 B2. Undo 的 pre-move 记录链断裂

- `undo-history.md` Core Rules §3：记录一次移动的顺序是「Rules 批准 → **调用方**调用 `record_before_move(model, move_result)` → 调用方执行 `model.move` → 成功 `commit()` / 失败 `discard_pending()`」；Dependencies 把 Drag & Drop 列为 Hard 下游，接口为 `record_before_move` / `commit` / `discard_pending`。
- `drag-drop-interaction.md` §9 与 Interactions/Dependencies：只描述 `MoveExecuted`，并写「State 与 Undo 消费 `MoveExecuted`」「Undo History | Hard | `MoveExecuted` | 记录移动」；没有 `record_before_move` / `commit` / `discard_pending`。
- `MoveExecuted` 是移动**之后**的事件，无法补出移动前快照。按当前文字实现，撤销要么永远为空，要么记录移动后状态——MVP 的「失败可无损重来」失效。

**可选修复方向（需用户决定，不在本报告代选）：**
1. 在 Drag & Drop §5/§6 与接口契约补 `record_before_move → model.move → commit/discard_pending`；
2. 引入携带移动前快照的 `BeforeMoveApplied` 事件，由 Undo 订阅；
3. 由 State 统一代录（会扩大 State 当前「不修改模型、不判断移动」的职责）。

#### 🔴 B4. 概念文档写了 FreeCell 不存在的「翻开被压住的牌 / 翻牌」

- `game-concept.md` Core Mechanics #2：「合法落位判定与**被压牌的自动翻开**」；Core Loop：「落位吸附、**翻开被压住的牌**」；Visual Identity Anchor #3：「只保留落位吸附、**翻牌**与发牌三类动效」。
- `card-deck-model.md` Core Rules §1：「**所有牌在发牌后始终正面朝上**。经典空当接龙没有背面朝下的牌、没有翻牌动作、没有隐藏信息；`face_up` 不是模型状态。」Edge Cases：「存档包含背面朝下标志：视为不兼容并拒绝。」
- `table-rendering-layout.md` Visual/Audio：「**卡背：不需要，不制作。** 空当接龙所有牌正面朝上。」
- 下游三个 GDD 正确；概念层需改，否则 Animation & Feedback 会按不存在的动作立项。

#### 🔴 B5. 22px 级联与 26px 等级字 / 16×20 花色冲突

- `table-rendering-layout.md` Core Rules §3 / `cascade_height`：K=13 → 22px；最后一牌底部 ≤716。
- 同文件 Visual/Audio：左上与右下角标距边 8px、等级字高 **26px**、花色 pip **16×20**、间隔 2px；Game Feel：「13 张列压缩到 22px」；Acceptance：「1024×576 截图：等级字高 ≥20 设备像素」。
- 算术：被覆盖牌只露出从牌顶开始的 22px；等级字形从 y=8、高 26px，到 y=34 才结束；只露出 y=8–22（14px），花色 pip 从 y≈36 开始完全不可见。0.8 缩放下可见等级高度约 **11.2 设备像素**，达不到 ≥20 的 AC；K≥11 时「红/黑 + 花色符号」双通道失效。
- 这直接决定卡面图集规格，且违反 Pillar 3「一秒读懂牌面」。

**可选修复方向（需用户决定）：**
1. 缩小角标（等级 ≤14px、pip ≤8px）以塞进 22px 条带；
2. 提高级联下限（缩小卡牌高度、上移桌面列或扩大逻辑画布高度）；
3. 为被覆盖牌额外绘制紧凑 rank/suit 条（偏离原版卡面）；
4. 接受可读性损失并修改 AC（与 Pillar 3 冲突，需记录为偏离）。

### Warnings

- ⚠️ **W3 蓝色高亮措辞冲突**：`game-concept.md` 写「唯一的高亮色只用于**可落位提示**」；`drag-drop-interaction.md` 明确禁止「合法落点高亮/发光/描边」。`hint-system.md` 与 Table Rendering 的窄例外已解释为「提示建议的落点」。建议把概念措辞改为「只用于**提示**指出的落点」。
- ⚠️ **W4 Hint 枚举顺序假设与 Rules §5 不一致**：`hint-system.md` 假定「目标外层、长度内层」，示例把 `T1→FC1` 排在 `T1→T2` 之后；`rules-move-validator.md` §5 规定每个源先试回收、再自由单元目标、再桌面列目标，且长度从 1 到 max 递增。Tier 5 同层平局与 AC-08/AC-F07 的期望值可能受影响。
- ⚠️ **W6 「新局」同名异义**：`toolbar-menus.md` 菜单「新局」= `NoGame` 时 `new_game(1)`，否则 `next_deal()`；Won 对话框「新局」= 打开选局对话框；「下一局」= `next_deal()`。`deal-generator.md` 明确不支持随机新局。若参考版本的 New Game 是随机局，当前没有系统拥有「随机 N」职责。
- ⚠️ **Toolbar ↔ Drag 依赖不对称**：`toolbar-menus.md` 依赖 `DragStarted` / `DragDropped` / `DragCancelled`；`drag-drop-interaction.md` 的 Interactions 表把 Toolbar 写成「无 — 输入不冲突即可」，未列 Toolbar 为事件消费者。系统索引已把 5 加入 Toolbar 依赖，但 Drag GDD 仍未承接。
- ⚠️ **Deal Generator 声称 Statistics 记录局号**：`deal-generator.md` Interactions/Dependencies 写「Statistics & Records | 下游 | 局号 | 记录」；`statistics-records.md` 只消费 `GameStarted` / `GameWon`，不记录局号（只保留内部 `active_game.N`）。属陈旧引用。
- ⚠️ **既有状态注记漂移（10 处）**：`deal-generator.md`（Game State Machine）、`card-deck-model.md`（Deal Generator / Rules / Table Rendering）、`rules-move-validator.md`（Drag & Drop / GSDM / Undo / Hint）、`drag-drop-interaction.md`（Table Rendering / Undo）仍写「GDD 未编写」。不影响数值，但会误导读者。

### Info

- ℹ️ **Table Rendering 空槽数算术错误**：`table-rendering-layout.md` 多处写「24 个空槽（16 个顶行槽 + 8 个桌面列首槽）」；顶行实际是 4 自由 + 4 回收 = **8** 个槽，正确总数是 **16**（8 + 8）。影响 NoGame 截图验收与资产/QA 计数。
- ℹ️ **K=14 模型/渲染范围缺口**：`card-deck-model.md` I5 允许桌面列追加任意数量（可达 14）；`table-rendering-layout.md` 的 `cascade_height` 变量范围写 K 0–13。正常玩法最长 13；建议 Table 注明「K>13 仅防御性公式，正常玩法不可达」。
- ℹ️ **既有 Open Questions 被提升为跨系统阻塞**：Stats Open Q5/Q7、Hint Open Q5/Q10、Undo Open Q1/Q2、Toolbar Open Q1/Q7/Q8 不再是单 GDD 待办；B1/B3 使其成为跨系统问题。

---

## Game Design Issues

### Blocking

#### 🔴 B1. 参考版本未钉死，Pillar 1 不可验证

- `game-concept.md` Unique Hook：「像 1995 年的 Windows 空当接龙」；Visual Identity Anchor：「任何视觉元素都必须能在 1995 年的 Windows 空当接龙里找到出处」。
- `deal-generator.md` §6：有效局号 **1–1,000,000**，并称「1–32,000 为原版子集」。
- `hint-system.md` 把「提示」列为原版辅助；`toolbar-menus.md` 把 **F10 提示**列为「原版式暂定值，需参考机截图确认」。
- `undo-history.md` Open Q1/Q2、`rules-move-validator.md` Open Q1/Q2、`toolbar-menus.md` Open Q7 都把关键保真问题推给「参考机截图后」，但没有定义参考机是哪个 Windows 版本。
- Pillar 1 是第一条支柱，也是 Unique Hook 的全部内容；当前无法验收。

**可选方向（用户决定）：**
1. 钉死 Windows 95 FreeCell（需确认 32,000 局号、菜单/快捷键、是否有提示/统计）；
2. 钉死 Windows 7 FreeCell（1,000,000 局号、提示、统计口径更自然）；
3. 明确声明「混合参考 + 有意偏离清单」，逐条记录偏离。

#### 🔴 B3. Statistics 的 restart / select 同一 N 口径不对称

- `game-state-machine.md` §3：`select_deal(N)` 等同 `new_game(N)`；`restart_deal()` 用当前 N 重新发牌；§6：`new_game(N)` 对同一 N 也有效。
- `statistics-records.md` §3：`on_game_started(N)` 在活动局未胜时记 LOSS、清空 streak；`on_game_restarted(N)` no-op；Edge Cases 明确「同一 N 再开新局按新局处理并记 loss，不按 N 去重」。
- `toolbar-menus.md` Edge Cases：「选择当前局号 N 照常开新局；Statistics 记一次 loss；Toolbar 不做特判」。
- 后果：F4 重开不计 loss；F3 选回同一 N 记 loss。胜利后先 F4 再放弃可免疫掉胜率；F3 同 N 可重复刷胜率/连胜。
- 这与 `statistics-records.md` Overview 的「不把任何失败变成惩罚」和 Pillar 4 直接冲突；Stuck 横幅的「新局」是最自然的换局动作，却会被记 loss。

**可选方向（用户决定）：**
1. `select_deal(current N)` 在 Playing/Stuck 下路由为 `restart_deal()` 语义（不发 `GameStarted`、不记 loss）；
2. restart 也计新局/记 loss（与 Pillar 4 冲突，需明确接受）；
3. 保留原版语义但在 UI 明示「换局号会记一次放弃」，并接受统计可被刷；
4. 新增独立「重玩本局」动作与统计定义（增加 UI 表面，需对照反支柱范围）。

#### 🔴 B5. Pillar 3 可读性（见 Consistency B5）

22px 级联与 26px 等级字 / 16×20 花色冲突；0.8 下可见等级约 11.2 设备像素，达不到自身 AC 的 ≥20；K≥11 花色不可见。属于 Pillar 3「一秒读懂牌面」的直接违背。

### Warnings

- ⚠️ **W1 统计 loss 与「无惩罚」叙事张力**：`statistics-records.md` Overview 说「不把任何失败变成惩罚」，但未胜时开新局记 loss、清空 streak，且开局即计入胜率分母；`toolbar-menus.md` Stuck 横幅的「新局」会触发 loss。建议 playtest 验证玩家是否把胜率下降感知为惩罚；不要在未验证前加防刷/成就化系统（会碰反支柱）。
- ⚠️ **W2 Hint Tier 1 无条件推荐回收**：`hint-system.md` §4 把回收移动列为 Tier 1 且优先于一切；Edge Cases 明确「不安全仍返回」。`rules-move-validator.md` 双击也回收优先。两条辅助路径都会把玩家推向不可逆的回收移动；Undo 可恢复，不破坏 Pillar 4，但会命中概念已列风险「提示若不够好用，会被认为不如原作」。建议用已知局号 playtest 不安全 Tier 1 的出现频率。
- ⚠️ **W5 首次启动不自动发牌**：概念 Flow State 说「首次进入直接发牌」；`game-state-machine.md` 启动状态是 `NoGame`，`toolbar-menus.md` 只有「新局」才调用 `new_game(1)`；没有系统声明启动自动发牌。首次启动可能停在空桌，与 onboarding 承诺不符。

### Holism Summary（3a–3g）

| 检查 | 结论 | 说明 |
|---|---|---|
| 3a 进度循环竞争 | **N/A** | 概念刻意没有跨局成长；唯一元循环是被动统计。 |
| 3b 注意力预算 | **PASS** | 同时主动约 3 个：Drag & Drop（主）、Undo/Hint（按需）、Toolbar（按需）；低于 3–4 上限。 |
| 3c 主导策略 | **CONCERNS** | 玩法层无严格支配选项；统计口径存在退化策略（B3）。 |
| 3d 经济循环 | **N/A** | 无货币/faucet/sink；数值平衡 **NOT ASSESSED — NO DATA**。 |
| 3e 难度曲线 | **N/A** | 无 scaling；难度由局号决定。 |
| 3f 支柱对齐 | **CONCERNS / 部分 FAIL** | 反支柱全部遵守；Pillar 1 因 B1 不可验证；Pillar 3 因 B5 有算术风险；Pillar 4 因 B3/W1 被削弱。 |
| 3g 玩家幻想一致性 | **PASS（两处裂缝）** | 整体身份「经典、安静、无损重来」一致；裂缝是 W1（统计压力）与 W2（提示可信度）。 |

- **反支柱**：10 份 GDD 均未引入剧情/战役、每日任务/成就/皮肤、联网/排行榜/云存档、移动触控优先。**无违规。**
- **视觉保真**：`design/art/art-bible.md` 不存在；Hint 的 `#000080`、3px 环带、Table 的空槽颜色/字体、Drag & Drop 的「全彩不透明替代 1995 XOR」都是暂定或有意偏离。**视觉保真 NOT ASSESSED。**
- **数值平衡**：无 `assets/data/`、无 `design/balance/`；registry 只有常量/公式，没有目标曲线。**NOT ASSESSED — NO DATA**；不得报 HEALTHY 或 CRITICAL。

---

## Cross-System Scenario Issues

Scenarios walked: 5

1. **移动 → Undo → State → Hint** — Drag & Drop / Undo / Rules / State / Hint / Table Rendering
   - 🔴 **B2**：Drag & Drop 只发 `MoveExecuted`（移动后），Undo 无法在 `model.move` 前捕获快照；撤销将为空或状态不变。
2. **胜利 → Statistics → Won 对话框 → Undo/Hint** — State / Statistics / Toolbar / Undo / Hint
   - ⚠️ B3/W1：胜利本身正确计数；但胜利后 F4 重开再放弃可免疫下一次 loss；统计压力与 Pillar 4 张力。
   - Undo 在 Won 禁用、Hint 在 Won 禁用；Won 对话框只弹一次；无重复计数。
3. **卡死 → 横幅 → Undo / 重开 / 新局** — State / Toolbar / Undo / Statistics / Hint
   - 🔴 **B3**：横幅「新局」触发 `GameStarted` → 记 loss；「重开本局」`GameRestarted` → 不记 loss。相同玩家意图，统计结果不同。
   - Undo 依 `can_undo(state)`；Hint 在 Stuck 禁用；横幅非模态，不阻断牌桌。
4. **Hint → 高亮 → 拖拽 → 清除** — Hint / Toolbar / Table Rendering / Drag & Drop / State
   - ⚠️ W2：Tier 1 可能推荐不安全的回收移动；⚠️ W3：蓝色高亮措辞与 Drag 禁令冲突；⚠️ W4：Hint 枚举顺序示例与 Rules §5 不一致。
   - 流程本身可运行：`DragStarted` 清高亮、拖拽期间 Hint 禁用、移动后 State 重判、生命周期 `Hint.reset()`。
5. **选局 → 发牌 → Statistics → Undo 清空 → 标题** — Toolbar / State / Deal Generator / Model / Statistics / Undo / Hint
   - 🔴 **B3**：选回同一 N 记 loss，与 F4 重开口径不同。
   - ⚠️ W5：首次启动 `NoGame`，不会自动发牌；⚠️ W6：菜单「新局」= `next_deal`，Won「新局」= 选局，同名异义。
   - `AT_LIMIT` 路径正确：不重发、无 `GameStarted`、统计不变、显示「已到最大局号」。

---

## GDDs Flagged for Revision

| GDD | Reason | Type | Priority |
|-----|--------|------|----------|
| `design/gdd/game-concept.md` | 「翻开被压住的牌/翻牌」与 FreeCell 全明牌矛盾（B4）；蓝色高亮措辞与 Drag 禁令冲突（W3）；1995 参考与 1M 局号/提示/统计来源未统一（B1）；首次启动「直接发牌」与 NoGame 不符（W5） | Consistency / Design Theory | Blocking |
| `design/gdd/drag-drop-interaction.md` | 未调用 Undo 的 `record_before_move` / `commit` / `discard_pending`，仅发 `MoveExecuted`（B2）；Interactions 未列 Toolbar 为事件消费者 | Consistency | Blocking |
| `design/gdd/undo-history.md` | 依赖表要求 Drag & Drop 调用 record/commit/discard，但下游 GDD 未承接（B2）；4096 深度 / Won 撤销保真未定 | Consistency | Blocking |
| `design/gdd/statistics-records.md` | `GameRestarted` no-op 与 `select_deal(同一 N)` 的 LOSS 口径不对称（B3）；开局即计分母与「无惩罚」张力（W1） | Design Theory / Consistency | Blocking |
| `design/gdd/toolbar-menus.md` | 「新局」= `next_deal` 与 Won「新局」= 选局同名异义（W6）；F3 同 N 触发 loss（B3）；快捷键/无状态栏/标题栏局号均为暂定 | Consistency / Pillar 1 | Blocking |
| `design/gdd/table-rendering-layout.md` | 22px 级联与 26px 等级字 + 16×20 花色冲突，K≥11 花色不可见（B5）；空槽 24 vs 16 算术错误；K=14 契约缺口 | Design Theory (Pillar 3) / Consistency | Blocking |
| `design/gdd/hint-system.md` | Tier 1 无条件推荐回收，可能不安全（W2）；Rules 枚举顺序假设/示例与 Rules §5 不一致（W4）；蓝色高亮歧义（W3） | Design Theory / Consistency | Warning |
| `design/gdd/game-state-machine.md` | `select_deal(N)` = `new_game(N)` 与 restart 的统计后果不同（B3）；首次启动 NoGame 与「直接发牌」不符（W5） | Consistency | Warning |
| `design/gdd/deal-generator.md` | 1–1,000,000 与原版子集/1995 锚点关系需明确（B1）；「不支持随机新局」与 UI New 语义的所有权缺口；声称 Statistics 记录局号（陈旧引用） | Consistency / Pillar 1 | Warning |
| `design/gdd/rules-move-validator.md` | 双击回收优先、supermove 空目标 `E−1` 的保真未验证（B1）；`legal_moves` 枚举顺序应成为 Hint 的正式契约（W4） | Consistency / Pillar 1 | Warning |
| `design/gdd/card-deck-model.md` | 内容正确（无 face-up 状态），但需在概念层澄清「翻牌」不存在（B4）；K=14 与 Table 的 K≤13 范围需注明正常玩法不可达 | Consistency | Info |

---

## Verdict: **FAIL**

5 个 Blocking 必须先解决，才能进入 `/create-architecture` 或 `/gate-check technical-setup`。

### If FAIL — required actions before re-running

1. **B1**：钉死参考版本（Windows 95 / Windows 7 / 混合参考 + 偏离清单），并把它写进 `game-concept.md` 与相关 GDD 的 Open Questions。
2. **B2**：在 Drag & Drop 与 Undo 之间补 pre-move 记录链（或等价事件/State 代录），并更新两侧 Dependencies/Interactions。
3. **B3**：统一 restart 与 select 同一 N 的统计口径，明确 Pillar 4 解释；更新 Statistics/Toolbar/State GDD。
4. **B4**：删除概念中的「翻开被压住的牌/翻牌」，改为「发牌 + 落位吸附」两类动效。
5. **B5**：解决 22px 级联与卡面角标的算术冲突，重定卡面图集规格与可读性 AC；同时修正空槽数 24 → 16。
6. **Warnings**：W1–W6 逐条关闭或记录为有意偏离；清理 10 处「GDD 未编写」状态注记。
7. 修复后重新运行 `/review-all-gdds`；随后逐份 `/design-review`，再 `/gate-check technical-setup`。

### NOT ASSESSED — what could not be reviewed

- **Phase 2 完整覆盖**：2b 规则矛盾、2d 数据/调参所有权、2e 公式兼容性的穷举扫描未由专用代理完成；主会话只做了定向核查。已在报告中列出确认的 Blocking/Warning，但不能声称 Phase 2 全覆盖。
- **原版保真**：没有钉死参考版本、没有参考机截图；B1、W6、Undo 深度、双击优先级、supermove 约定、提示行为、快捷键、标题栏/状态栏均 `NOT ASSESSED`，不能当作已确认与原版一致。
- **视觉保真**：`design/art/art-bible.md` 不存在；色值、字体、环带、空槽样式均为暂定或有意偏离。
- **数值平衡**：无 `assets/data/`、无 `design/balance/`、无目标曲线；`NOT ASSESSED — NO DATA`。
- **Toolbar 几何**：`project.godot` 无 `[display]` 段，stretch 模式未钉死；`menu_bar_fits` 与 G01–G05 在 ADR 前 `NOT ASSESSED`。
- **非 MVP 系统**：Save/Resume、Animation & Feedback、Settings、Audio & SFX、Readability & Accessibility 没有 GDD，不在本次 10 份评审集内。

---

## Post-Review Resolution Log

**2026-10-24**
- **B1 resolved**：参考版本钉死为混合参考（1995 视觉/手感 + Windows 7/XP 功能基线），写入 `game-concept.md` §Reference Version；相关 GDD 的 Open Questions 已改为引用该基线。
- **B4 resolved**：删除 `game-concept.md` 中的「被压牌的自动翻开 / 翻开被压住的牌 / 翻牌」描述；Target Aesthetics、Core Mechanics、Core Loop、Visual Identity Anchor 已改。
- **W3 resolved**：概念的高亮措辞改为「唯一的高亮色只用于提示指出的落点，不用于装饰或拖拽合法落点」。
- **B2 resolved**：`drag-drop-interaction.md` 的合法落位与双击路径现在按 `record_before_move → model.move → commit` 执行，失败/取消调用 `discard_pending`；Interactions/Dependencies/Edge Cases/AC 已同步；`systems-index.md` 的 Drag & Drop 依赖补上 Undo（6）。
- **B3 resolved**：`game-state-machine.md` 的 `select_deal(N)` 在 N = 当前 N 时路由为 `restart_deal()`（`GameRestarted`，不发 `GameStarted`）；`statistics-records.md` 与 `toolbar-menus.md` 的 Edge Cases 同步；同 N 选局不再记 loss、不增局数、不重复计胜。换到不同 N 仍按新局/放弃处理。
- **B5 resolved**：`table-rendering-layout.md` 增加「被覆盖牌紧凑索引」规格（距边 5px、块高 ≤15px、等级 13px、花色 8×8），顶牌保留完整角标；AC 已改为「顶牌 ≥20 设备像素；被覆盖牌紧凑索引 ≥10 设备像素、花色可见」。同时修正空槽数 24 → 16、增加 K=14 防御性注记；`hint-system.md` 的高亮内缩理由已同步。
- **Remaining Blocking**：无（B1–B5 均已解决）。评审 Verdict 仍为 FAIL，因为这是评审时点的快照；需重新运行 `/review-all-gdds` 才能把状态提升。
- **Systems index**：Drag & Drop、Undo History、Statistics & Records、Toolbar & Menus、Table Rendering 均已回到 `Designed`。

---

## Targeted Re-review (2026-10-24)

**Scope note:** `review-scope.sh` returned `CHANGED: none` because `design/gdd/` is untracked in git; this is a false negative, not evidence of no changes. A targeted manual re-review was performed over the five Blocking fixes and the directly affected cross-references. A full fresh Phase 2/3 agent re-run was **not** performed.

**Verified fixed:**
- **B1**：`game-concept.md` §Reference Version（混合参考 + 偏离清单）；相关 GDD Open Questions 已引用。
- **B2**：`drag-drop-interaction.md` 合法落位/双击调用 `record_before_move → model.move → commit`；失败/取消调用 `discard_pending`；Interactions/Dependencies/Edge Cases/AC 已同步；systems index 的 Drag 依赖包含 Undo（6）。
- **B3**：`game-state-machine.md` 的 `select_deal(N)` 在 N = 当前 N 时路由为 `restart_deal()`；Statistics/Toolbar Edge Cases 已同步；同 N 选局不再记 loss、不增局数、不重复计胜。
- **B4**：`game-concept.md` 中所有「翻牌/翻开被压住/背面朝下」引用已删除；Card & Deck Model 与 Table Rendering 保持全明牌正确来源。
- **B5**：`table-rendering-layout.md` 增加被覆盖牌紧凑索引（距边 5px、等级 13px、花色 8×8）+ AC 修订（顶牌 ≥20 设备像素；被覆盖牌索引 ≥10 设备像素、花色可见）；空槽数 24→16；K=14 防御性注记；Hint 内缩理由同步。

**Consistency warnings fixed:** W3（概念高亮措辞）、W4（Rules §5 明确「目标外层、长度内层」+ Hint 示例注记）、Toolbar↔Drag 事件依赖、Deal→Statistics「记录局号」陈旧引用、8 处「GDD 未编写」状态注记。

**Remaining warnings（设计/意图，非阻塞）：**
- **W1**：未完成局被放弃时 Statistics 记 loss / 拉低胜率，与「不把任何失败变成惩罚」存在张力（Open Question #7）。
- **W2**：Hint Tier 1 无条件推荐回收移动，包括不安全的过早低牌（Hint Open Question #5）。
- **W5**：首次启动为 `NoGame`；概念写「首次进入直接发牌」，但没有系统自动调用 `new_game(1)`。
- **W6**：菜单「新局」= `next_deal()`；Won 对话框「新局」= 打开选局；同名异义。

**Verdict: CONCERNS** — 0 Blocking；4 个设计/意图 Warning 保留。若需要最高置信度，在 `/gate-check technical-setup` 前再跑一次完整 `/review-all-gdds`。

**Next:** `/create-architecture`（接受 CONCERNS）——剩余 Warning 是 playtest/意图问题，不是跨文档矛盾。
