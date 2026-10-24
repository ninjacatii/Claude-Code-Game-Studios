# Drag & Drop Interaction

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 手感优先于画面

## Summary

Drag & Drop Interaction 把鼠标按下、移动与释放翻译成空当接龙的落牌动作：抓起合法后缀、1:1 跟手、命中测试路由、Rules 判定、0ms 合法落位与 120ms 非法回弹。它不判断规则、不修改模型、不绘制牌桌；它负责指针捕获、4px 阈值、抓取偏移、双击派发与取消，是“原版手感”的核心。

> **Quick reference** — Layer: `Feature` · Priority: `MVP` · Key deps: `Card & Deck Model`, `Rules & Move Validator`, `Table Rendering & Layout`

## Overview

Drag & Drop Interaction 把鼠标按下、移动与释放翻译成空当接龙的落牌动作。玩家从桌面列或自由单元抓起一张牌或一段合法序列，牌跟随指针移动；释放时向 Rules 询问目标是否合法，合法则执行模型移动并通知 Game State Machine，非法则让牌回到原位。它不判断规则、不修改牌堆、不绘制牌桌；它负责指针捕获、拖拽阈值、抓取偏移、命中测试路由、回弹与双击派发，是“原版手感”的核心。玩家应感到牌是“被手抓住”的：即时跟手、落位干脆、失败无惩罚。

## Detailed Design

### Core Rules

**1. 范围与所有权**

- 拥有：指针捕获、拖拽生命周期、拖拽阈值、抓取偏移、命中测试路由、落点派发、回弹请求、双击派发。
- 不拥有：规则、模型、绘制/布局、动画、音效、游戏状态。
- 读取：Table Rendering & Layout 的命中测试与卡牌矩形；Rules 的 `valid_sequence` / `can_move` / `auto_move_destination`；State 的 `can_accept_moves`。
- 写入：只有在 Rules 返回 `legal = true` 后才调用 `Card & Deck Model.move`；随后调用 `Game State Machine.on_move_applied`；向 Animation & Feedback 请求落位/回弹。

**2. 拖拽生命周期**

- `Idle`：无拖拽。
- `Pressed`：左键按下在牌上，等待阈值或释放。
- `Dragging`：超过阈值，拖拽层激活。
- `Dropping`：释放，命中测试与 Rules 判定。
- `Resolving`：非法回弹期间，输入锁定；以 Animation & Feedback 的完成信号结束，≤150ms 是观测上界。合法落位为 0ms 瞬移，不进入 `Resolving`。若 150ms 内未收到完成信号（异常），看门狗强制结束并记录错误。
- `Cancelled`：Esc/右键/焦点丢失/状态变化，立即回到 `Idle`。
- 同一时间只允许一个拖拽。

**3. 拖拽开始**

- 左键按在牌上：
  - 若 `State.can_accept_moves()` 为 false → 忽略。
  - 若源是回收单元 → 忽略（Rules 会返回 `FOUNDATION_LOCKED`）。
  - 计算从点击牌到顶牌的连续后缀；该后缀必须是合法序列（`Rules.valid_sequence`）。若不是 → 不开始拖拽。
  - 记录源牌堆、点击牌、移动单元长度、抓取偏移（指针 − 点击牌左上角）。
  - 进入 `Pressed`。
- 指针相对按下点的**欧氏距离** > 4 逻辑像素（严格大于；不是累计路径长度）→ 进入 `Dragging`；创建拖拽层；通知 Rendering 隐藏源牌；不做合法落点高亮。
- 移动单元在拖拽开始时固定；拖拽期间模型/状态变化 → 取消。

**4. 拖动**

- 拖拽层位置 = 当前指针 − 抓取偏移。
- 1:1 跟手；不缩放、不旋转；层级高于所有牌堆。
- 源牌在视觉上隐藏，露出空位；模型仍保留它们直到落位成功。
- 不高亮合法目标、不改变光标、不显示悬停轮廓。
- 指针移出窗口仍保持拖拽；在窗口外释放视为非法落点。

**5. 释放与落点**

- 若未进入 `Dragging`：视为点击；检查双击；不产生拖拽。
- 若在 `Dragging` 释放：用 Table Rendering 的命中测试确定目标：
  - 桌面列 → 目标列；`count` = 移动单元长度。
  - 自由单元 → 目标格；目标只接受 `count = 1`，但 Drag & Drop 仍以 `unit_length` 调用 Rules，由 Rules 拒绝多张。
  - 回收单元 → 目标回收单元；同上，只接受 `count = 1`，由 Rules 拒绝。
  - 任何牌堆之外 → 非法。
- 调用 `Rules.can_move(source, destination, count)`：
  - 合法：
    1. 调用 `Undo.record_before_move(model, result)`。**失败则中止本次移动**：回弹、不修改模型、不通知 State、不提交记录，并记录错误。
    2. 调用 `Card & Deck Model.move`。
    3. 调用 `Undo.commit()`；失败时移动已成立、不回滚，调用 `Undo.discard_pending()` 清理并记录错误。
    4. `State.on_move_applied(result)`；0ms 瞬移落位（原版即时，无过渡）；Audio 播放；同帧回到 `Idle`，不进入 `Resolving`。
  - 非法 → 调用 `Undo.discard_pending()`（幂等）；请求 120ms 回弹；不修改模型、不通知 State；进入 `Resolving`，在回弹完成信号时结束；零距离回弹直接结束。
  - `model.move` 意外失败 → 调用 `Undo.discard_pending()`；按非法处理，回弹并记录错误。

**6. 双击**

- 两次按下在 OS 双击间隔内，且两次按下点之间与每次按下期间的移动都不超过 4 逻辑像素（与 `is_drag_start` 的严格 `>` 阈值一致）。
- 调用 `Rules.auto_move_destination(source, card)`：
  - 合法：
    1. 调用 `Undo.record_before_move(model, result)`；失败则中止，不移动、不通知 State，记录错误。
    2. 调用 `model.move`。
    3. 调用 `Undo.commit()`；失败时移动已成立、不回滚，调用 `Undo.discard_pending()` 清理并记录错误。
    4. `State.on_move_applied`；0ms 瞬移；不创建拖拽层。
  - 非法 → 调用 `Undo.discard_pending()`（幂等）；无操作（可选轻微拒绝音效）。
- 若第二次按下后移动超过阈值，则按拖拽处理，不触发双击。
- 双击优先级由 Rules 固定：回收单元优先，其次空自由单元。

**7. 取消**

- `Dragging` 期间 Esc 或右键 → 取消；移除拖拽层；源牌立即恢复；调用 `Undo.discard_pending()`（幂等）；不修改模型、不通知 State。
- 所有取消路径（Esc、右键、焦点丢失、State 变化）都调用 `Undo.discard_pending()`；没有 pending 记录时无副作用。
- 窗口失去焦点 / 应用暂停 → 取消。
- 拖拽期间 State 变化（新局/重开/撤销/胜利）→ 取消。
- 若 `Resolving` 期间收到 State 命令（新局/重开/撤销），命令优先；Drag & Drop 立即隐藏拖拽层/回弹并回到 `Idle`。
- 取消是即时恢复；非法落点是 120ms 回弹；零距离回弹直接结束。

**8. 输入与命中测试**

- MVP 仅鼠标；触摸/手柄不支持。
- 输入在进入拖拽逻辑前统一转换为**逻辑坐标**（`to_logical`）；接口参数命名为 `logical_pos`。逻辑坐标 = 卡牌布局坐标系，已包含 stretch/scale，不含 letterbox 黑边；屏幕/窗口/设备像素不出现在公式内。
- 使用模式匹配 `InputEventMouse`，读取 `InputEventMouse.Position`（视口坐标；`InputEvent` 基类没有 `Position`）。窗口尺寸权威源是 `GetWindow().Size`；`ContentScaleMode=Disabled`、`ContentScaleFactor=1.0` 时必须成立。
- Table Rendering 负责把当前拉伸/缩放换算成卡牌矩形。
- 牌索引统一 **0-based、自底向顶**；`top_index = height − 1`；Table Rendering 必须使用同一约定。
- 命中测试优先最上层卡牌，其次是空牌堆区域；拖拽层对命中测试透明。
- 使用 `_unhandled_input`（或等价路径），避免菜单栏/对话框 UI 吞掉事件；不得用 `_input` 抢在 GUI 之前处理。不要把 `GetLocalMousePosition()`（子节点局部坐标）送入 `to_logical`。
- `State.can_accept_moves()` 为 false 时忽略所有牌面输入。
- 无悬停、无高亮、无光标变化。

**9. 事件与协调**

- `DragStarted`、`DragMoved`、`DragDropped`、`DragCancelled`、`MoveExecuted`、`MoveRejected`。
- Animation & Feedback 消费落位/回弹；Audio & SFX 消费拾取/放下/拒绝；`MoveExecuted` 是移动成功后的通知，供 Animation/Audio 消费，**不触发 State**。State 由合法路径中的 `on_move_applied(result)` 直接调用。Undo 通过 `record_before_move` / `commit` / `discard_pending` 记录，不依赖移动后事件。
- Drag & Drop 不写统计；胜利统计由 State 处理。

**10. 性能与手感**

- 目标 60fps；输入在同一帧内处理。
- 复用拖拽层节点/对象；拖动期间不做每帧分配。
- 命中测试 O(牌堆数)，可忽略。
- 阈值 4 逻辑像素；回弹 120ms；双击间隔用 OS 设置。
- 不引入帧停顿；拖拽即时开始。

### States and Transitions

| 状态 | 进入 | 离开 | 守卫 |
|------|------|------|------|
| `Idle` | — | `Pressed` | 左键按在可拖拽牌上，`can_accept_moves` |
| `Pressed` | 按下 | `Dragging` | 移动 > 4 px |
| `Pressed` | 按下 | `Idle` | 释放：点击/双击处理 |
| `Dragging` | 超过阈值 | `Dropping` | 释放 |
| `Dropping` | 释放 | `Idle` | Rules 合法，0ms 瞬移 |
| `Dropping` | 释放 | `Resolving` | Rules 非法，120ms 回弹 |
| `Resolving` | 非法回弹开始 | `Idle` | 回弹完成信号（≤150ms 看门狗） |
| `Pressed` / `Dragging` | Esc/右键/焦点丢失/状态变化 | `Idle` | 取消，无模型变更 |

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|------|------|------|------|------------|
| Table Rendering & Layout | 上游（几何）/ 下游（事件） | 命中测试、卡牌矩形 | `DragStarted` / `DragMoved` / `DragDropped` / `DragCancelled` | Rendering 拥有布局；TableView 消费事件 |
| Card & Deck Model | 上游 | `move` | 移动结果 | Model 拥有牌堆 |
| Rules & Move Validator | 上游 | `valid_sequence` / `can_move` / `auto_move_destination` | 合法/非法 | Rules 拥有规则 |
| Game State Machine | 接口 | `can_accept_moves` / `on_move_applied` | 移动通知 | State 拥有生命周期 |
| Animation & Feedback | 下游 | 拖拽/落位/回弹请求 | 动画 | Animation 拥有表现 |
| Audio & SFX | 下游 | 拾取/放下/拒绝事件 | 音效 | Audio 拥有声音 |
| Undo History | 上游 | `record_before_move` / `commit` / `discard_pending` | 撤销记录 | Undo 拥有历史；Drag 在 move 前记录 |
| Toolbar & Menus | Soft（事件） | `DragStarted` / `DragDropped` / `DragCancelled` | 拖拽门控、高亮清除 | Toolbar 只读事件 |
| Readability & Accessibility | 下游 | 拖拽事件 | 键盘替代（Full Vision） | Accessibility 拥有替代输入 |
| Statistics & Records | 无 | — | — | 通过 State 间接 |

**接口契约**：

- `on_pointer_down(logical_pos) / on_pointer_move(logical_pos) / on_pointer_up(logical_pos) / on_cancel()`
- `notify_layout_changed(logical_pointer)`：窗口 resize 后由 DragInputAdapter 调用；重算 `grab_offset`，非拖拽时 no-op
- `is_dragging() -> bool`
- `get_drag_unit() -> MoveUnit | null`
- 事件：`DragStarted`、`DragMoved`、`DragDropped`、`DragCancelled`、`MoveExecuted`、`MoveRejected`
- Table Rendering 需提供：
  - `hit_test(logical_pos) -> HitResult | null`
  - `HitResult = { pile, card_index }`：`card_index` 为 0-based 自底向顶；空堆返回 `-1`（不可拖拽）。
  - `card_rect(pile, index) -> Rect`
  - TableView 订阅 `DragStarted` / `DragMoved` / `DragDropped` / `DragCancelled`；拖拽层与源牌隐藏由 TableView 绘制，Drag 不直接调用视图。
- Drag & Drop 不修改模型、不判断规则、不写文件。

## Formulas

**坐标约定**：以下公式全部使用逻辑坐标；原点为逻辑视口左上角，+x 向右，+y 向下，1 单位 = 1 逻辑像素。屏幕/窗口/设备像素不进入公式；输入在 `on_pointer_*` 之前统一经 `to_logical(p)` 转换。牌索引 0-based、自底向顶，`top_index = height − 1`。

### The drag_threshold formula

The drag_threshold formula is defined as:

`drag_threshold = 4`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| drag_threshold | int（逻辑像素） | 4（constant） | 本 GDD §3、§10 | 从按下点出发、离开 `Pressed` 所需的指针位移 |

**Output Range:** 恒为 4（min = max）。换算成物理像素为 `4 × display_scale`；比较也在逻辑空间进行。恰好 4.0 逻辑像素不触发拖拽（严格 `>`）。

**Example:** 1280×720 布局在 1920×1080 窗口 → scale 1.5 → 阈值 6 窗口像素；在 960×540 窗口 → 3 窗口像素。阈值比较始终在逻辑坐标；`atlas_scale`/device factor 不参与。

### The grab_offset formula

The grab_offset formula is defined as:

`grab_offset = pointer_down − card_top_left`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| pointer_down | Vector2（逻辑） | 点击牌矩形内 | `on_pointer_down` | 按下时的指针位置 |
| card_top_left | Vector2（逻辑） | 布局范围内 | Table Rendering `card_rect(pile, clicked_index).TopLeft`，在隐藏源牌**之前**采样 | 点击牌左上角 |
| grab_offset | Vector2（逻辑） | x ∈ [0, card_width)，y ∈ [0, card_height) | 本公式 | 从点击牌原点到指针的偏移；整个拖拽期间缓存 |

**Output Range:** 分量范围 `[0, card_width) × [0, card_height)`（半开矩形）。负分量说明坐标空间不一致，应断言而不是 clamp。按下点必须满足 `card_rect(pile, clicked_index).HasPoint(pointer_down)`。

**Example:** 点击牌左上角 (100, 200)，按下点 (140, 230) → `grab_offset = (40, 30)`；80×110 的牌上分别是 50% 与 27%。

### The drag_layer_position formula

The drag_layer_position formula is defined as:

`drag_layer_position = pointer_current − grab_offset`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| pointer_current | Vector2（逻辑） | 无界，可移出视口 | `on_pointer_move` | 当前指针位置 |
| grab_offset | Vector2（逻辑） | `[0, card_width) × [0, card_height)` | 本公式（按下时缓存） | 指针相对点击牌原点的偏移 |
| drag_layer_position | Vector2（逻辑） | 无界 | 本公式 | 拖拽层中**点击牌左上角**的位置 |

**Output Range:** 无界；不 clamp。clamp 会让牌在窗口边缘停住而指针继续移动，破坏 1:1 跟手。指针移出窗口时继续外推；在窗口外释放视为非法落点。其余移动单元按此原点相对排布。

**Example:** `grab_offset = (40, 30)`、指针 (612, 311) → 拖拽层位置 (572, 281)；指针相对按下点移动 (472, 81)，牌也移动 (472, 81)。

### The is_drag_start formula

The is_drag_start formula is defined as:

`is_drag_start = 1 if (dx² + dy²) > drag_threshold² else 0`
where `dx = pointer_current.x − pointer_down.x`, `dy = pointer_current.y − pointer_down.y`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| pointer_current | Vector2（逻辑） | 无界 | `on_pointer_move` | 当前指针位置 |
| pointer_down | Vector2（逻辑） | 点击牌矩形内 | `on_pointer_down`（缓存） | 按下位置 |
| dx, dy | float（逻辑像素） | 无界 | 派生 | 相对按下点的分量位移 |
| drag_threshold | int | 4 | 本 GDD | 阈值 |
| is_drag_start | int | {0,1} | 本公式 | 1 = 满足 `Pressed → Dragging` 守卫 |

**Output Range:** {0,1}。欧氏距离、严格 `>`；恰好 4.0 → 0。单次移动事件可一次越过阈值并立即开始拖拽；绕圈但离按下点始终不超过阈值不会开始。该公式是**转移守卫**，不是每帧状态；进入 `Dragging` 后由状态机锁存，指针回到 4px 圆内仍保持拖拽。

**Example:** 按下 (100,200)；移到 (103,204)：9+16=25 > 16 → 1（距离 5）；移到 (103,202)：9+4=13 ≤ 16 → 0（约 3.61）；移到 (104,200)：16 > 16 为 false → 0（恰好 4.0）。

### The drag_unit_length formula

The drag_unit_length formula is defined as:

`drag_unit_length = top_index − clicked_index + 1`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| clicked_index | int | 0 ≤ clicked_index ≤ top_index | Table Rendering 命中测试（0-based 自底向顶） | 按下的牌 |
| top_index | int | 0–51；= height − 1；空堆为 −1 | Card & Deck Model | 同堆顶牌 |
| drag_unit_length | int | 1–13 | 本公式 | 移动后缀张数；作为 `Rules.can_move(..., count)` 的 `count` |

**Output Range:** 1–13。后缀必须通过 `valid_sequence` 才会开始拖拽；严格递减异色的序列最多 13 张，与 registry `max_sequence_length` [0,13] 一致。若 `clicked_index > top_index` 或空堆，表达式 ≤ 0，必须按“不拖拽”拒绝，绝不传给 Rules。自由单元/回收单元高度 1，`top_index = clicked_index = 0` → 1，满足 §5 的 `count = 1`。

**Example:** 7 张列的索引 0–6；按在索引 3 → 4 张；按在顶牌 → 1 张。2 个空自由单元 + 1 个空列时容量 `(2+1)×2 = 6`；4 张可移，8 张会被 Rules 拒绝并回弹。

### The snap_back_duration formula

The snap_back_duration formula is defined as:

`snap_back_duration = 120`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| snap_back_duration | int（毫秒） | 120（constant） | 本 GDD §5、§10 | 非法落点回弹动画时长；由 Animation & Feedback 消费 |
| pending_distance | float（逻辑像素） | ≥ 0 | `‖drag_layer_position − card_top_left‖` | 回弹距离；**不是**时长的输入 |

**Output Range:** 恒为 120ms（min = max），与距离无关；60Hz 下约 7.2 帧，实际在第 8 帧结束（≈133.3ms），由 `Resolving` 的 ≤150ms 锁吸收。距离为 0 时直接结束，不播放 120ms。插值曲线、起点与终点由 Animation & Feedback 拥有；Drag & Drop 提供时长与缓存的 home 矩形。

**Example:** 释放点 (572,281)、home (100,200) → 距离 ≈478.9px，120ms 内平均 ≈3.99px/ms；9px 微移同样 120ms，平均 ≈0.075px/ms。

## Edge Cases

- **If 坐标空间混用（屏幕坐标减逻辑矩形）**: 抓取偏移会带上变换误差，拖拽起手跳变；实现必须只使用逻辑坐标，负的 `grab_offset` 断言失败而不是 clamp。
- **If `card_top_left` 在隐藏源牌之后采样**: 会拿到拖拽层/隐藏表示；必须在按下时、隐藏前缓存。
- **If 拖拽期间窗口缩放或布局重排**: TableLayout 更新后，DragInputAdapter 以当前指针重算 `logical_pointer`，调用 `notify_layout_changed`；Drag 用新的 `card_rect(source, clicked_index)` 重算 `grab_offset = logical_pointer − card_top_left`，保持牌在指针下；指针在画布外时保留外推逻辑坐标，不 clamp。非拖拽时 no-op。
- **If Table Rendering 使用 top-down 索引**: `drag_unit_length` 会取错后缀；接口契约固定 0-based、自底向顶，集成测试必须覆盖。
- **If 按下的牌到顶牌不是合法序列**: 不开始拖拽；释放按点击处理。
- **If 按在回收单元上**: 不开始拖拽（Rules `FOUNDATION_LOCKED`）。
- **If `can_accept_moves` 为 false**: 忽略牌面输入；Won/Stuck/NoGame 均不可拖。
- **If 按在空牌堆上**: 不开始拖拽。
- **If 指针位移恰好 4.0 逻辑像素**: 不开始拖拽（严格 `>`）。
- **If 拖拽层在指针下**: 命中测试透明，目标取拖拽层下方的牌堆。
- **If 指针移出窗口**: 继续拖拽；窗口外释放视为非法落点；若 OS 停止投递移动事件，按焦点丢失取消，不冻结。
- **If 在源牌堆上释放**: Rules 返回 `SAME_SOURCE_AND_TARGET`；回弹。
- **If 多张拖到自由单元或回收单元**: Rules 拒绝；回弹；Drag & Drop 不预拒绝，始终以 `unit_length` 调用 Rules。
- **If 拖拽长度超过容量**: 拖拽可以开始，释放时 Rules 拒绝；回弹；不预先按容量限制拖拽。
- **If 双击非顶牌**: Rules 返回 `NOT_TOP_CARD`；无操作。
- **If 双击时回收单元与自由单元都可放**: 回收单元优先。
- **If 双击时两者都不可放**: 无操作（可选轻微拒绝音效）。
- **If 双击期间移动超过阈值**: 转为拖拽，不触发双击。
- **If 双击回收单元上的牌**: 无操作（锁定）。
- **If `Dragging` 期间 State 命令到达**: 先取消拖拽，再处理命令。
- **If `Resolving` 期间 State 命令到达**: 命令优先；立即隐藏拖拽层/回弹并回到 `Idle`。
- **If `model.move` 意外失败**: 调用 `Undo.discard_pending()`；按非法落点回弹，记录错误，不通知 State。
- **If `model.move` 成功但 `State.on_move_applied` 返回错误**: 移动已成立，不回滚；记录错误，继续回到 `Idle`。
- **If `record_before_move` 失败**: 中止本次移动；不调用 `model.move`、不通知 State、不提交记录；回弹并记录错误。不得在无撤销记录的情况下执行移动。
- **If `commit()` 失败**: 移动已成立，不回滚；调用 `discard_pending()` 清理，记录错误；Undo 栈可能缺少该条记录，但不得回滚牌局。
- **If `discard_pending()` 失败**: 幂等；记录错误，模型与 State 不变；下一次 record 前必须处于无 pending 状态。
- **If `Resolving` 期间再次按下**: 忽略，直到动画完成。
- **If 多指针/触摸输入**: MVP 不支持；忽略。
- **If 零距离回弹（取整后距离 0）**: 直接结束，不播放 120ms。
- **If 回弹完成信号 150ms 内未到达**: 看门狗强制结束 `Resolving`，记录错误，进入 `Idle`。
- **If 合法落位**: 0ms 瞬移，不进入 `Resolving`；Animation & Feedback 不播放过渡。
- **If 非法回弹动画**: `Resolving` 以 Animation & Feedback 的完成信号结束，不使用独立计时器。
- **If 拖拽层与胜利/卡死覆盖层同时存在**: 状态变化先取消拖拽；覆盖层在拖拽层之上。
- **If 工具栏按钮在拖拽期间被点击**: 先取消拖拽，再执行工具栏命令。
- **If 音频事件重复触发**: 每次拖拽只发一次拾取、一次放下/拒绝。

## Dependencies

**上游依赖**：

- **Table Rendering & Layout**（Hard，GDD 已完成）：`hit_test(logical_pos)`、`card_rect(pile, index)`（0-based 自底向顶）、拖拽层层级；Drag 发出 `DragStarted` / `DragMoved` / `DragDropped` / `DragCancelled`，由 TableView 消费。
- **Card & Deck Model**（Hard）：`move`；只读查询。
- **Rules & Move Validator**（Hard）：`valid_sequence`、`can_move`、`auto_move_destination`。
- **Game State Machine**（接口）：`can_accept_moves`、`on_move_applied`。
- **Undo History**（Hard）：`record_before_move` / `commit` / `discard_pending`；移动前记录、成功后提交、失败/取消后丢弃。

**下游依赖（依赖本系统的系统）**：

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Animation & Feedback | Hard | `DragStarted` / `DragDropped` / `DragCancelled` / `MoveExecuted` / `MoveRejected` | 落位/回弹动画与完成信号 |
| Audio & SFX | Soft | 拾取/放下/拒绝事件 | 可选音效 |
| Undo History | Hard | `record_before_move` / `commit` / `discard_pending` | 移动前记录、成功后提交 |
| Readability & Accessibility | Soft | 拖拽事件 | 键盘替代在 Full Vision |
| Toolbar & Menus | Soft（事件） | `DragStarted` / `DragDropped` / `DragCancelled` | 拖拽门控、高亮清除；Toolbar 只读事件 |
| Statistics & Records | 无 | — | 通过 State 间接 |

**接口契约**：

- `on_pointer_down(logical_pos) / on_pointer_move(logical_pos) / on_pointer_up(logical_pos) / on_cancel()`
- `notify_layout_changed(logical_pointer)`：窗口 resize 后由 DragInputAdapter 调用；重算 `grab_offset`，非拖拽时 no-op
- `is_dragging() -> bool`
- `get_drag_unit() -> MoveUnit | null`
- 事件：`DragStarted`、`DragMoved`、`DragDropped`、`DragCancelled`、`MoveExecuted`、`MoveRejected`
- Table Rendering 需提供：
  - `hit_test(logical_pos) -> HitResult | null`
  - `HitResult = { pile, card_index }`：`card_index` 为 0-based 自底向顶；空堆返回 `-1`（不可拖拽）。
  - `card_rect(pile, index) -> Rect`（0-based 自底向顶）
  - TableView 订阅 `DragStarted` / `DragMoved` / `DragDropped` / `DragCancelled`；拖拽层与源牌隐藏由 TableView 绘制，Drag 不直接调用视图。
- Drag & Drop 不修改模型、不判断规则、不写文件；只在 Rules 批准后调用模型。

**Provisional assumptions（下游未设计，先假定）**：

- Table Rendering 使用逻辑坐标与 0-based 自底向顶索引；拖拽层可置于所有牌堆之上。
- State 的 `can_accept_moves` 只在 `Playing` 为 true；`on_move_applied` 对合法移动不返回错误。
- Animation & Feedback 对落位与回弹提供“完成”信号，`Resolving` 以此结束。
- Audio & SFX 对拾取/放下/拒绝做去重，不在每次移动事件上重复播放。

**Registry references**：`drag_unit_length`、`snap_back_duration`、`drag_threshold` 应登记；`max_sequence_length` 的 `referenced_by` 应追加 `design/gdd/drag-drop-interaction.md`（Phase 5b 更新）。

## Visual/Audio Requirements

**视觉判据**：任何可见元素都必须能在 1995 年 Windows FreeCell 找到出处；找不到就不加。1995 版可能使用 GDI XOR 反相拖动；本项目选择**全彩不透明 + 隐藏源牌**作为有意偏离，理由是花色可读性优先。该偏离必须记录在 Art Bible 中。

**拖拽层**

- 100% 不透明，无 alpha、无阴影、无柔边、无外发光。
- 1.0 缩放、0° 旋转、无拾取动画；第一帧就出现在正确位置。
- 位置取整到最近逻辑像素，避免子像素模糊。
- 层级高于所有牌堆，低于工具栏、状态栏、菜单与对话框；裁剪在窗口客户区内。
- 多张序列的垂直间距与源列级联完全一致，整个单元刚性移动。
- 使用与桌面相同的牌面资源、同倍率、最近邻；不重采样、不使用单独拖拽贴图。

**源牌堆**

- 源牌同帧隐藏；无幽灵、无虚线占位框、无 gap 高亮。
- 露出的内容由 Table Rendering 绘制：空列为桌面底色，有牌则下一张牌在常规级联位置，空自由单元/回收单元显示桌面空槽轮廓。
- 模型在落位成功前不变；取消时同帧恢复。

**合法落位**

- 0ms 瞬移；无闪光、描边、缩放弹跳、粒子、文字或目标列脉冲。
- 拖拽层同帧销毁，源隐藏同帧解除；“落位感”由位置正确性与可选音效提供。

**非法回弹**

- 120ms 直线回到源矩形原点；线性或缓入均可；无过冲、回弹、弧线；结束位置像素级等于源位置。
- 整个移动单元一起回，内部间距不变。
- 无红闪、红色描边、抖动、“无效移动”文字或屏幕震动。
- 最后一帧精确落在源矩形，同帧销毁并解除隐藏；不要在回弹开始时提前恢复源牌。
- 松手点与源原点取整到最近逻辑像素后相等（距离 0）→ 0ms 结束；否则 120ms。

**双击自动移动**

- 不创建拖拽层；源牌同帧消失、目标槽同帧出现，0ms 瞬移。
- 无目标高亮、落点脉冲或连锁动画。
- 非法双击画面完全无变化；可选极轻拒绝音。

**音频（可选，默认关闭）**

- 1995 原版无音效；任何声音都是新增偏离，默认关闭，由 Settings 控制。
- 事件与相对响度：拾取 20–40ms / −6dB；合法放下 50–80ms / 0dB；非法回落 60–100ms / −4dB；双击自动移动 40–60ms / −3dB。
- 短、干、无混响、居中单声道；纸牌/毛毡质感，不是赌场声。
- 无悬停音、无拖拽循环音、无随距离变化的音高、无按牌数叠加、无 BGM。

**禁止添加**

- 合法落点高亮/发光/描边、悬停轮廓、光标变化、拖拽阴影/半透明、拾取缩放弹跳、旋转/倾斜、惯性/弹簧/磁吸、粒子/拖尾、屏幕震动、非法红闪/红边/抖动、“无效移动”文字、目标列脉冲、张数徽标、拖拽时桌面变暗、拖拽层描边/tint、双击落点闪光/缩放、音效连击/音高递增、任何 BGM。

**视觉验收（可用截图/录像逐条核对）**

1. 拖拽全程 100% 不透明、无阴影、1.0 缩放、0° 旋转，贴图与桌面同一份。
2. 拖拽层覆盖所有牌堆，但不盖住工具栏/菜单。
3. 拖 3 张序列时，牌间垂直偏移与源列一致（像素级）。
4. 源位同帧变空，无幽灵/虚线框；空槽轮廓仍可见。
5. 非法落位 120ms 直线归位，无过冲，结束帧与源位置像素一致。
6. 零距离非法释放 0ms 结束，无可见停顿。
7. 合法落位 0ms；前后两帧唯一差异是牌的位置。
8. 双击不出现拖拽层，无目标高亮。
9. 全程不存在任何光标资源变化。

## Game Feel

- **起手**：超过 4 逻辑像素立即进入拖拽；无延迟、无起手动画。
- **跟手**：牌与指针 1:1，抓取偏移全程保持；不缩放、不旋转、无惯性、无加速度。
- **空位**：源牌在视觉上立即消失，露出空位；模型在落位成功前不变。
- **落点**：没有磁吸；只有指针进入目标命中区域才吸附；合法落位 0ms 瞬移、干脆；非法回弹 120ms、无弹跳。
- **双击**：一帧内响应；回收单元优先，其次最低索引空自由单元；无连锁。
- **节奏**：`Resolving` 锁 ≤150ms；以动画完成信号结束；快速连续拖拽不会重叠。
- **可读性**：不高亮合法目标、不改变光标；牌本身是唯一反馈。
- **缩放**：阈值与拖拽层使用逻辑坐标；窗口缩放不改变手感。
- **性能**：60fps；输入延迟 ≤1 帧；拖动期间无每帧分配。

## UI Requirements

- 无直接 UI；牌桌与拖拽层由 Table Rendering & Layout 绘制。
- 光标保持系统默认；无悬停轮廓、无合法落点高亮。
- 工具栏输入优先：拖拽期间点击工具栏，先取消拖拽再执行命令。
- 胜利/卡死覆盖层出现时 `can_accept_moves` 为 false，牌面输入被忽略。
- Full Vision 的键盘替代输入由 Readability & Accessibility 负责，不在 MVP。

## Cross-References

| 系统 | 引用内容 | 状态 |
|------|----------|------|
| Table Rendering & Layout | 命中测试、卡牌矩形、拖拽层 | GDD 已完成（design/gdd/table-rendering-layout.md） |
| Card & Deck Model | `move`、只读查询 | GDD 已完成 |
| Rules & Move Validator | `valid_sequence` / `can_move` / `auto_move_destination` | GDD 已完成 |
| Game State Machine | `can_accept_moves` / `on_move_applied` | GDD 已完成 |
| Animation & Feedback | 落位/回弹与完成信号 | GDD 未编写 |
| Audio & SFX | 拾取/放下/拒绝 | GDD 未编写 |
| Undo History | `record_before_move` / `commit` / `discard_pending` | GDD 已完成（design/gdd/undo-history.md） |
| Readability & Accessibility | 键盘替代 | GDD 未编写 |
| 原版手感基准 | 拖拽跟手、无高亮、非法回弹 | 概念支柱 2/3（无外部文档） |

## Acceptance Criteria

**测试约定**：坐标全部为逻辑坐标；牌索引 0-based、自底向顶，`top_index = height − 1`；“模型/State 不变”指状态指纹与 `get_state()`/`get_deal_number()` 均不变。

### 拖拽开始

- **GIVEN** 按下后欧氏位移恰好 4.0 逻辑像素，或绕圈但离按下点始终 ≤4，**WHEN** 指针移动，**THEN** 不开始拖拽；**GIVEN** 单次移动事件使位移 >4（如 3,4 → 5.0），**THEN** 进入 `Dragging`、创建拖拽层、隐藏源牌；距离是直线距离，不是累计路径。
- **GIVEN** 按下点 (140,230)、点击牌左上角 (100,200)，**WHEN** 拖拽层显示，**THEN** `grab_offset = (40,30)`（在隐藏源牌前采样）；指针到 (612,311) 时拖拽层左上角为 (572,281)，不 clamp，牌与指针 1:1。
- **GIVEN** 拖拽中窗口从 1280×720 缩放到 1920×1080，**WHEN** `Window.SizeChanged` 完成 reflow，**THEN** `notify_layout_changed(logical_pointer)` 用新 `card_rect` 重算 `grab_offset`，牌保持 under-pointer、不跳变；指针在画布外时保留外推逻辑坐标；非拖拽时 no-op。
- **GIVEN** 7 张列、索引 0–6，按在索引 3，**WHEN** 后缀 3–6 通过 `Rules.valid_sequence`，**THEN** 移动单元为 4 张，长度 `top_index − clicked_index + 1 = 4`，整个拖拽期间固定。
- **GIVEN** 点击后缀不是合法序列、按在回收单元、按在空牌堆，或 `State.can_accept_moves()` 为 false，**WHEN** 指针移动，**THEN** 不开始拖拽；释放按点击处理。

### 落点与反馈

- **GIVEN** 合法释放到桌面列，**WHEN** `Rules.can_move` 返回 true，**THEN** `Card & Deck Model.move` 执行、`State.on_move_applied` 通知、0ms 瞬移落位、同帧回到 `Idle`，不进入 `Resolving`。
- **GIVEN** 非法释放（同源目标、多张到自由单元/回收单元、超过容量、目标为空、`model.move` 意外失败），**WHEN** `Rules.can_move` 返回 false 或模型失败，**THEN** 模型与 State 不变、120ms 直线回弹、`Resolving` 锁；容量只在释放时由 Rules 检查，超容量拖拽仍可开始。
- **GIVEN** 非法释放且拖拽层与源原点取整后距离为 0，**WHEN** 结算，**THEN** 0ms 结束；否则 120ms；结束位置与源位置像素一致。
- **GIVEN** 拖拽层覆盖牌堆，**WHEN** 释放，**THEN** 命中测试对拖拽层透明：桌面列 → 该列且 `count = unit_length`；自由单元/回收单元 → 只接受 `count = 1`，多张由 Rules 拒绝；所有牌堆之外 → 非法。
- **GIVEN** 两次按下在 OS 双击间隔内、每次移动 ≤4 逻辑像素，**WHEN** 处理双击，**THEN** 调用 `Rules.auto_move_destination`，回收单元优先于最低索引空自由单元；成功则 `model.move` + `State.on_move_applied` + 0ms，不创建拖拽层。
- **GIVEN** 双击非顶牌（`NOT_TOP_CARD`）、回收单元上的牌或无处可放，**WHEN** 处理，**THEN** 完整 no-op（可选拒绝音），无拖拽层、无模型/State 变更；第二次按下后移动 >4px 转为拖拽。
- **GIVEN** FC1 持有 `5♣`，**WHEN** 拖到顶牌 `6♦` 的桌面列或空列，**THEN** Rules 合法、0ms 落位；**WHEN** 拖到 ♣ 回收单元高度 4，**THEN** 合法；**WHEN** 拖到 ♥ 回收单元，**THEN** `FOUNDATION_MISMATCH`、回弹。
- **GIVEN** `Dragging`，**WHEN** Esc、右键、焦点丢失或 State 变化，**THEN** 立即取消：移除拖拽层、同帧恢复源牌，模型与 State 不变。
- **GIVEN** `Resolving`，**WHEN** 指针按下/移动，**THEN** 忽略直到回弹完成；**WHEN** State 命令（新局/重开/撤销）到达，**THEN** 命令优先，隐藏拖拽层/回弹并立即回到 `Idle`；若 150ms 内未收到完成信号，看门狗强制结束并记录错误。

### 契约、视觉与性能

- **GIVEN** 任意指针输入，**WHEN** 进入拖拽逻辑，**THEN** 坐标已转换为逻辑坐标（`to_logical`；公式中无屏幕/窗口/设备像素），索引为 0-based 自底向顶且与 Table Rendering 一致；集成测试覆盖 top-down 索引错误。
- **GIVEN** 任意拖拽，**THEN** 拖拽层 100% 不透明、使用桌面同一份牌面资源、1.0 缩放、0° 旋转、无 alpha/阴影/柔边/发光；拖拽层覆盖所有牌堆但不盖工具栏/菜单；源牌同帧隐藏、无幽灵/占位框；无合法落点高亮、无悬停轮廓、无光标变化；合法落位 0ms 无闪光/脉冲；非法回弹无红闪/抖动/文字。
- **GIVEN** 任意手势结束，**THEN** `DragStarted` 每次拖拽只发一次，`DragDropped` / `DragCancelled` / `MoveExecuted` / `MoveRejected` 每次手势至多一次；音频（拾取/放下/拒绝）可选且默认关闭；拖拽层复用、无每帧分配；60fps 下输入同帧处理。
- **GIVEN** 点击工具栏按钮时正在拖拽，**THEN** 先取消拖拽，再执行工具栏命令。
- **GIVEN** `model.move` 成功但 `State.on_move_applied` 返回错误，**THEN** 移动已成立、不回滚；记录错误并回到 `Idle`。
- **GIVEN** 一次合法落位，**THEN** `Undo.record_before_move` 在 `model.move` 之前恰调用一次，`Undo.commit` 在 `model.move` 之后恰调用一次；随后撤销恢复移动前状态。
- **GIVEN** `record_before_move` 失败，**THEN** 不调用 `model.move`、不通知 State、不提交记录；牌回到源位置并记录错误。
- **GIVEN** `commit()` 失败，**THEN** 移动已成立、不回滚；`discard_pending()` 被调用，记录错误。
- **GIVEN** 非法落点或取消，**THEN** `discard_pending()` 恰调用一次（幂等），模型与 State 不变。
- **GIVEN** 拖拽长度超过容量，**WHEN** 释放，**THEN** Rules 返回 `SUPERMOVE_LIMIT_EXCEEDED`，回弹；Drag & Drop 不在拖拽开始时预拒绝。

## Open Questions

| 问题 | Owner | 目标解决时间 |
|------|-------|--------------|
| 窗口外指针捕获：OS 是否继续投递移动事件 | technical-director | 架构阶段 |
| 全彩拖拽偏离 XOR 的 Art Bible 记录 | art-director | Art Bible |
| Table Rendering 的索引约定（0-based 自底向顶） | Table Rendering & Layout | 该 GDD 编写时 |
| Full Vision 的键盘替代输入 | Readability & Accessibility | 该 GDD 编写时 |
| 音频默认关闭与 Settings 开关 | Audio & SFX + Settings | 对应 GDD |
| 合法落位 0ms 在低帧率下的观感 | qa-lead + ux-designer | 首个可玩版本 |
