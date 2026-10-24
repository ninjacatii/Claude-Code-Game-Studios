# Table Rendering & Layout

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 一秒读懂牌面

## Summary

Table Rendering & Layout 把 Card & Deck Model 的牌堆画成 1280×720 逻辑画布上的牌桌：8 列、4 自由单元、4 回收单元、自适应级联与均匀缩放。它提供命中测试、卡牌矩形、拖拽层与源牌隐藏给 Drag & Drop，并在窗口缩放与 HiDPI 下保持几何一致；不判断规则、不修改模型、不播放动画。

> **Quick reference** — Layer: `Core` · Priority: `MVP` · Key deps: `Card & Deck Model`

## Overview

Table Rendering & Layout 把 Card & Deck Model 的牌堆状态画成可见的牌桌：8 个桌面列、4 个自由单元、4 个回收单元，以及 52 张牌的位置、层级与缩放。它提供命中测试与卡牌矩形给 Drag & Drop，隐藏正在拖拽的源牌，并把拖拽层放在牌堆之上、工具栏之下。它不判断规则、不修改模型、不处理输入；它负责“一眼读懂”的布局、卡面可读性与窗口缩放下的稳定几何。

## Detailed Design

### Core Rules

**1. 范围与所有权**

- 拥有：逻辑布局、卡牌矩形、层级、缩放/letterbox、命中测试、拖拽层绘制、源牌隐藏、模型变化时的重绘。
- 不拥有：规则、模型、输入、游戏状态、动画插值、音效、工具栏/菜单。
- 读取：Card & Deck Model 的牌堆与牌；Drag & Drop 的拖拽状态；State 的 `NoGame` 用于空桌显示。
- 不修改任何游戏状态；只提供几何与绘制。

**2. 逻辑画布与缩放**

- 基准逻辑画布：1280×720。
- 均匀缩放：`scale = min(window_w / 1280, window_h / 720)`；最小 0.8（窗口最小 1024×576）。窗口最小尺寸在启动时设置；`window_w/h` 使用视口可见矩形尺寸（Godot `get_viewport().get_visible_rect().size`），不使用屏幕像素。
- 水平/垂直偏移：`offset_x = (window_w − 1280 × scale) / 2`，`offset_y = (window_h − 720 × scale) / 2`；letterbox 不拉伸。
- `to_logical(window_pos) = ((x − offset_x) / scale, (y − offset_y) / scale)`；`to_window` 为其逆。
- 禁止非均匀拉伸；卡牌比例在任何窗口尺寸下保持不变。

**3. 卡牌几何**

- 卡牌尺寸：140×190 逻辑像素（比例 0.7368，接近原版 71:96）。
- 水平间距 12；8 列 `x_i = 38 + i × 152`（i=0…7）；左右边距 38。
- 顶行 y=38，高 190：自由单元在列 0–3 上方；回收单元在列 4–7 上方。顶行槽位 x 与桌面列共用 `column_x(0..7)`，不维护两套横向间距。
- 桌面列顶部 y=256；级联偏移随列高 K 自适应：`cascade(K) = min(30, floor((720 − 256 − 190) / (K − 1)))`（K ≥ 2；K ≤ 1 时取 30）。K=10 → 30，K=11 → 27，K=12 → 24，K=13 → 22；最后一牌底部 ≤ 716，不越出 720 画布。第 k 张牌 `y = 256 + k × cascade(K)`。
- 列的命中区域：x 范围内，y 从 256 到最后一牌底部 +12；空列使用首牌槽位。
- `card_rect(pile, index)` 接受 `index = -1`：空列返回首槽矩形 `tableau_y(0, 0)`；`index ≥ 0` 返回牌矩形。
- 回收单元槽位从左到右绑定花色 C、D、H、S；自由单元从左到右索引 0–3。

**4. 绘制**

- 卡面：等级 + 花色，红/黑；使用纹理图集；不 tint、不投影；1px 黑色边框与 8px 圆角**烘焙进贴图**，不是运行时描边。
- 空槽：使用与卡牌同尺寸的空槽轮廓贴图；**不使用运行时线条绘制**（4.7 的 `CanvasItem` 线条无抗锯齿羽化，线宽会变细）。
- 背景：原版绿色 `#008000` 平面填充；无渐变、无图案、无暗角。
- 源牌隐藏：拖拽单元对应的源牌在桌面上不绘制；模型仍保留它们。
- 拖拽层：在牌堆之上、工具栏/菜单/覆盖层之下；全彩、1.0 缩放、0° 旋转、无阴影。
- 重绘：模型变化、拖拽状态变化、窗口缩放、State 变化时重绘；空闲时不逐帧重绘。使用脏标记与纹理图集，避免每帧分配。
- 绘制使用边一致取整（`round(x·s+offset)` 与 `round((x+w)·s+offset)` 的差为宽度）；命中测试使用逻辑浮点矩形，二者误差被 12px 列间隙吸收。

**5. 命中测试**

- `hit_test(logical_pos) -> HitResult | null`，`HitResult = { pile, card_index }`。
- 优先级：顶行槽位（自由单元/回收单元）→ 桌面列 → null。
- 桌面列：从顶牌向下检查卡牌矩形；命中哪张返回哪张的 `card_index`；若在列命中区域但不在任何卡牌上，返回顶牌索引；空列返回 `card_index = -1`。
- 顶行：命中槽位矩形；`card_index = 0`（空槽也为 0，便于判断）。
- `card_rect(pile, index) -> Rect`：桌面列按 `y = tableau_y(index, K)`（`K` 为列高；`30` 只是 K≤10 的常见值，不是固定级联）；顶行 `y = 38`；尺寸 140×190。
- 命中测试与绘制使用同一逻辑矩形与同一 `scale/offset`；拖拽层对命中测试透明。
- `hit_test` 对画布外（含 letterbox 黑边）返回 null，避免黑边点击落到牌桌。

**6. 窗口缩放**

- 窗口尺寸变化：重算 `scale`、`offset`、全部矩形；保持均匀缩放与 letterbox。
- 最小窗口 1024×576（scale 0.8）；更小时由窗口管理器限制；若平台允许更小，按 0.8 缩放并裁剪，记录警告。
- 拖拽期间缩放：Drag & Drop 负责重新缩放 `grab_offset`；Table Rendering 更新矩形，二者使用同一逻辑坐标。

**7. 状态与覆盖层**

- `NoGame`：显示空桌（背景 + 16 个空槽：8 个顶行槽 + 8 个桌面列首槽），不绘制牌。
- `Playing` / `Won` / `Stuck`：按模型绘制牌；胜利/卡死覆盖层由 Toolbar/UI 绘制在拖拽层之上。
- 拖拽层与覆盖层同时存在时，状态变化先取消拖拽（Drag & Drop 职责）。

**8. 性能**

- 52 张牌 + 16 个空槽；纹理图集；绘制调用稳定且低。
- 空闲时不重绘；拖拽时只更新拖拽层位置。
- 命中测试 O(牌堆数 + 列内牌数)，可忽略。
- 目标 60fps；窗口缩放只重排一次。

### States and Transitions

Table Rendering 无游戏状态机；以下是布局/绘制状态：

| 状态 | 进入 | 离开 | 说明 |
|------|------|------|------|
| `NoGame` | 无模型 | `Dealt` | 空桌 + 16 空槽 |
| `Dealt` | 新局/读档 | `Dragging` / `Resized` | 按模型绘制 |
| `Dragging` | 拖拽开始 | `Dealt` | 源牌隐藏 + 拖拽层 |
| `Resized` | 窗口尺寸变化 | 之前状态 | 重算几何后恢复 |

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|------|------|------|------|------------|
| Card & Deck Model | 上游 | 牌堆/牌读取 | 无 | Model 拥有状态 |
| Drag & Drop Interaction | 下游 | 拖拽事件（`DragStarted` / `DragMoved` / `DragDropped` / `DragCancelled`） | `hit_test` / `card_rect`；TableView 消费事件并绘制拖拽层/隐藏源牌 | Rendering 拥有几何 |
| Animation & Feedback | 下游 | 起止矩形 | 插值动画 | Animation 拥有表现 |
| Toolbar & Menus | 下游 | 覆盖层 | 层级 | Toolbar 拥有 UI |
| Hint System | 下游 | 高亮请求 | 无 | Hint 拥有选择；Toolbar 转发 |
| Readability & Accessibility | 下游 | 几何/卡牌矩形 | 可访问表示 | Accessibility 拥有替代 |
| Game State Machine | 上游 | `NoGame` / 终态 | 无 | State 拥有生命周期 |

**接口契约**：

- `hit_test(logical_pos) -> HitResult | null`（`HitResult = { pile, card_index }`；空列 `card_index = -1`）
- `card_rect(pile, index) -> Rect`（0-based 自底向顶；顶行 index 0）
- `show_drag_layer(unit, logical_position)` / `move_drag_layer(logical_position)` / `hide_drag_layer()` / `set_source_hidden(pile, count, hidden)`：TableView 内部绘制接口，由 `DragStarted` / `DragMoved` / `DragDropped` / `DragCancelled` 驱动；Drag 不直接调用 TableView。
- `set_hint_highlight(source, destination)` / `clear_hint()`：Hint GDD 定义的提示高亮；由 Toolbar 转发。`source` 为 `move.cards` 的源堆与索引；`destination` 索引：空桌面列 −1、空顶行槽 0、非空牌堆 height−1。
- `to_logical(window_pos)` / `to_window(logical_pos)` / `get_scale()` / `get_offset()`
- Table Rendering 不修改模型、不处理输入、不播放动画。

## Formulas

**常量**：`BASE_W=1280`、`BASE_H=720`、`CARD_W=140`、`CARD_H=190`、`COL_PITCH=152`、`MARGIN_X=38`、`TOP_Y=38`、`TABLEAU_Y0=256`、`CASCADE_MAX=30`、`MIN_SCALE=0.8`。全部为逻辑像素。

### The column_x formula

The column_x formula is defined as:

`column_x(i) = 38 + i × 152`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| i | int | 0–7 | 桌面列/顶行槽位序号 | 从左到右 |
| column_x | int | 38–1102 | 本公式 | 左边缘逻辑坐标 |

**Output Range:** 38–1102。i=7 → 1102，右边缘 1242，左右边距对称 38。i 越界为调用方错误，不 clamp。

**Example:** i=7 → 38 + 1064 = 1102；i=4 → 646（第一个回收单元）。

### The cascade_height formula

The cascade_height formula is defined as:

`cascade_height(K) = 30 if K ≤ 2 else min(30, floor(274 / (K − 1)))`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| K | int | 0–13 | Card & Deck Model 列高 | 列内牌数 |
| cascade_height | int | 22–30 | 本公式 | 相邻牌的垂直偏移 |

**Output Range:** 22–30。K≤10 → 30；K=11 → 27；K=12 → 24；K=13 → 22。最后一牌底部 ≤ 716，不越出 720 画布。

**Example:** K=7 → 30；K=11 → 27；K=13 → 22。

### The tableau_y formula

The tableau_y formula is defined as:

`tableau_y(k, K) = 256 + k × cascade_height(K)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| k | int | 0–(K−1) | 列内索引（0-based 自底向顶） | 第 k 张牌 |
| K | int | 0–13 | Card & Deck Model 列高 | 列内牌数 |
| tableau_y | int | 256–526 | 本公式 | 牌顶部 y |

**Output Range:** 顶部 y 256–526；牌底部 446–716。不越出画布。

**Example:** K=7、k=3 → 256 + 90 = 346；K=13、k=12 → 256 + 264 = 520，底部 710。

### The card_rect formula

The card_rect formula is defined as:

`card_rect(pile, index) = Rect2(column_x(col(pile)), row_y(pile, index), 140, 190)`

`col(F_f) = f`、`col(S_s) = 4 + s`、`col(T_i) = i`
`row_y(T_i, k) = tableau_y(k, K(T_i))`、`row_y(F_f, 0) = row_y(S_s, 0) = 38`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| pile | enum | T0–T7、F0–F3、S0–S3 | 命中结果/模型 | 牌堆 |
| index | int | 0–(K−1)（桌面列）；0（顶行） | 命中结果/模型 | 0-based 自底向顶 |
| col(pile) | int | 0–7 | 本公式 | 列/槽位序号 |
| row_y | int | 38 或 256–526 | 本公式 | 顶部 y |
| card_rect | Rect | x 38–1102，y 38–526，140×190 | 本公式 | 逻辑矩形 |

**Output Range:** 顶行 y=38；桌面列 y 由 `tableau_y` 决定。`index=-1` 不参与本公式；空列槽位取 `tableau_y(0, 0)=256`。

**Example:** `card_rect(T2, 3)`（K=7）→ (342, 346, 140, 190)；`card_rect(F1, 0)` → (190, 38, 140, 190)；`card_rect(S3, 0)` → col=7 → (1102, 38, 140, 190)。

### The scale formula

The scale formula is defined as:

`scale = max(0.8, min(window_w / 1280, window_h / 720))`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| window_w / window_h | float | ≥1 | 视口可见矩形尺寸（`get_viewport().get_visible_rect().size`） | 客户端逻辑像素 |
| scale | float | 0.8–无界 | 本公式 | 均匀缩放 |

**Output Range:** 0.8–无界；1920×1080 → 1.5，2560×1440 → 2.0，3840×2160 → 3.0。最小 0.8；更小的窗口由最小窗口尺寸限制，若平台允许则裁剪并记录警告。

**Example:** 1600×1000 → min(1.25, 1.3889) = 1.25；800×600 → min(0.625, 0.8333) = 0.625 → 0.8。

### The letterbox_offset formula

The letterbox_offset formula is defined as:

`offset_x = (window_w − 1280 × scale) / 2`
`offset_y = (window_h − 720 × scale) / 2`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| window_w / window_h | float | ≥1 | 视口尺寸 | 客户端尺寸 |
| scale | float | 0.8–无界 | 本公式 | 均匀缩放 |
| offset_x / offset_y | float | 可负（裁剪） | 本公式 | 窗口边缘到画布边缘 |

**Output Range:** clamp 未生效时，一个偏移为 0，另一个 ≥0；clamp 生效时（窗口小于 1024×576），短边偏移为负（居中裁剪）。非均匀拉伸不可能发生。

**Example:** 1600×1000、scale 1.25 → (0, 50)；1024×576 → (0, 0)；800×600、scale 0.8 → (−112, 12)。

### The coordinate_transform formula

The coordinate_transform formula is defined as:

`to_logical(p_w) = ((p_w.x − offset_x) / scale, (p_w.y − offset_y) / scale)`
`to_window(p_l) = (p_l.x × scale + offset_x, p_l.y × scale + offset_y)`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| p_w | Vector2 | 任意窗口像素 | 输入事件 | 窗口坐标 |
| p_l | Vector2 | 任意逻辑像素 | Drag & Drop/模型 | 逻辑坐标 |
| scale / offset | — | 见上 | 本 GDD | 缩放与 letterbox |
| to_logical / to_window | Vector2 | 无界 | 本公式 | 互逆变换 |

**Output Range:** 无界。窗口角点映射到画布外；`hit_test` 对画布外（x∉[0,1280] 或 y∉[0,720]）返回 null。`to_window ∘ to_logical` 为恒等（≤1 ulp）。

**Example:** offset (0,50)、scale 1.25：`to_logical((800,500)) = (640,360)`；`to_window((640,360)) = (800,500)`。

## Edge Cases

- **If 列高 K=13**: 级联 22，最后一牌底部 710；所有牌仍在画布内、可命中。
- **If 列高 K=11 / K=12**: 级联 27 / 24，底部 ≤716；不越界。
- **If 列高 K=14（模型允许但正常玩法不可达）**: 按防御性公式计算或记录 error；正常合法序列上限 13，不将其视为可玩状态。
- **If 列高 K≤10**: 级联 30；常规布局。
- **If 列高 K=0 / 1**: 级联取 30；没有级联效果。
- **If 窗口小于 1024×576**: `scale` 钳制到 0.8，偏移变负，画布居中裁剪；记录警告，并尽量由窗口最小尺寸阻止。
- **If 点击落在 letterbox 黑边**: `hit_test` 返回 null；不得映射到牌桌。
- **If HiDPI/系统缩放**: `window_w/h` 使用视口可见矩形尺寸，不使用屏幕像素；否则缩放与命中测试会错位。
- **If 绘制取整**: 使用边一致取整；避免图集边缘 1px 缝隙；命中测试用逻辑浮点矩形。
- **If 级联牌重叠**: 命中测试从顶牌向下检查，最高索引优先；返回被点中的那张。
- **If 空列**: `hit_test` 返回 `card_index = -1`；`card_rect(-1)` 取空槽矩形 `tableau_y(0)`；空列不可拖拽。
- **If 空顶行槽位**: `hit_test` 返回 `card_index = 0`；可作为落点。
- **If 回收单元花色绑定**: 槽位从左到右 C、D、H、S；渲染按花色查询模型绑定，不按索引猜测。
- **If 源牌隐藏**: 模型仍保留牌；拖拽期间命中测试被 Drag & Drop 禁用，不会命中隐藏牌。
- **If 拖拽层与牌堆重叠**: 拖拽层绘制在牌堆之上；命中测试对拖拽层透明。
- **If 拖拽层与工具栏/覆盖层重叠**: 工具栏/覆盖层在拖拽层之上。
- **If NoGame**: 绘制空桌与 16 个空槽（8 顶行 + 8 桌面列首槽），不绘制牌。
- **If Won/Stuck**: 牌桌保持可见；覆盖层由 UI 绘制在拖拽层之上。
- **If 拖拽期间窗口缩放**: 重算几何；Drag & Drop 重新缩放 `grab_offset`；两者使用同一逻辑坐标。
- **If 4K/高倍缩放**: scale 可到 3.0；卡面使用高分辨率资源与 mipmap，避免模糊。
- **If 使用运行时线条画空槽**: 4.7 的 `CanvasItem` 线条无抗锯齿羽化，线宽会变细；必须使用空槽贴图。
- **If 列命中区域延伸**: 从 256 到最后一牌底部 +12；空列使用首牌槽位；不要越过相邻列的 x 范围。
- **If 模型原子变化**: 等模型操作完成后再重绘；不绘制半副牌。
- **If 拖拽开始**: 源牌隐藏与拖拽层出现在同一帧；无闪烁。
- **If 合法落位 0ms**: 同一帧重绘到目标位置；无过渡帧。
- **If 非法回弹**: Table Rendering 保持静态；Animation 使用起止矩形插值，不逐帧重排牌桌。
- **If 索引约定 top-down**: 取错牌；契约固定 0-based 自底向顶，集成测试覆盖。
- **If 未来高对比/无障碍模式**: Full Vision 由 Accessibility 提供；MVP 保持原版红/黑与花色符号冗余。
- **If 窗口最大化/全屏**: 均匀放大，不设最大 scale；letterbox 保持比例。
- **If 多显示器 DPI 变化**: 重新读取视口尺寸并重算 scale/offset。

## Dependencies

**上游依赖**：

- **Card & Deck Model**（Hard）：牌堆、牌、列高、回收单元花色绑定；只读。
- **Game State Machine**（接口）：`NoGame` / 终态用于空桌与覆盖层；只读。

**下游依赖（依赖本系统的系统）**：

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Drag & Drop Interaction | Hard | `hit_test` / `card_rect`；Drag 事件由 TableView 消费 | 几何与拖拽层 |
| Animation & Feedback | Soft | 起止矩形 | 回弹/落位插值 |
| Toolbar & Menus | Soft | 覆盖层层级 | 胜利/卡死覆盖层 |
| Readability & Accessibility | Soft | 卡牌矩形/几何 | 高对比/键盘替代在 Full Vision |
| Statistics & Records | 无 | — | 不直接交互 |

**接口契约**：

- `hit_test(logical_pos) -> HitResult | null`（`HitResult = { pile, card_index }`；空列 `card_index = -1`；画布外 null）
- `card_rect(pile, index) -> Rect`（0-based 自底向顶；顶行 index 0；空列用 `tableau_y(0)`）
- `show_drag_layer(unit, logical_position)` / `move_drag_layer(logical_position)` / `hide_drag_layer()` / `set_source_hidden(pile, count, hidden)`：TableView 内部绘制接口，由 `DragStarted` / `DragMoved` / `DragDropped` / `DragCancelled` 驱动；Drag 不直接调用 TableView。
- `set_hint_highlight(source, destination)` / `clear_hint()`：Hint GDD 定义的提示高亮；由 Toolbar 转发。`source` 为 `move.cards` 的源堆与索引；`destination` 索引：空桌面列 −1、空顶行槽 0、非空牌堆 height−1。
- `to_logical(window_pos)` / `to_window(logical_pos)` / `get_scale()` / `get_offset()`
- Table Rendering 不修改模型、不处理输入、不播放动画。

**Provisional assumptions（下游未设计，先假定）**：

- Drag & Drop 传入逻辑坐标并遵守 0-based 自底向顶索引；拖拽层对命中测试透明。
- Animation & Feedback 使用 `card_rect` 的起止位置插值；Table Rendering 本身不做动画。
- Toolbar/覆盖层绘制在拖拽层之上；Table Rendering 不绘制胜利/卡死 UI。
- 卡面资源至少按 3× 目标尺寸提供，或使用可无损放大的矢量资源。

**Registry references**：`card_width`、`card_height`、`column_pitch`、`margin_x`、`tableau_y0`、`cascade_max`、`min_scale`、`base_canvas_width`、`base_canvas_height` 应登记（Phase 5b 更新）。

## Visual/Audio Requirements

**视觉判据**：任何可见元素都必须能在 1995 年 Windows FreeCell 找到出处。本系统只有四个表面：卡面、空槽轮廓、平面桌面、黑色 letterbox；其余一律不加。

**卡面**

- 白底 `#FFFFFF`；黑色 `#000000`；红桃/方块 `#FF0000`。卡面只用这三种颜色。
- 1px 黑色边框、8px 圆角，**烘焙进图集贴图**，不是运行时描边。
- **顶牌/完全可见牌**：左上与右下角标距边 8px，块 ≤34×46；等级字高 26px，花色 pip 16×20，间隔 2px；右下角标旋转 180°。
- **被覆盖牌（级联条带 ≤22px）**：左上角紧凑索引距边 5px，块高 ≤15px；等级字高 13px，花色 pip 8×8，等级与花色并排；只覆盖左上角，不改变中央 pip；紧凑索引不得与 Hint 高亮环（1–4px）重叠。右下角标只在完全可见时绘制。
- “10”显示为两个数字，不用 `T`。
- 中央 pip 限制在 72×130；仅用标准 English-pattern 排布；A 为一张居中大 pip（约 48px）。
- 宫廷牌：English-pattern 内框，内缩 10px，为角标留 L 形净空；从**公有领域** English-pattern 牌组取材，不描摹 `cards.dll`。
- 字体：高对比粗衬线（Source Serif 4 Bold 或 Noto Serif Bold，OFL）；在 0.8 缩放下验证可读。
- 无 tint、无阴影；拖拽中的牌全彩、1.0 缩放、0°，无投影。
- 只有一副牌、一种样式；无牌背选择。

**卡背**

- 不需要，不制作。空当接龙所有牌正面朝上；`NoGame` 显示空桌，不显示牌堆。

**桌面**

- 平面 `#008000`，单色填充 1280×720；无纹理、渐变、暗角、图案、噪点。
- 平面色在 4K 下天然清晰，无需额外资源。

**空槽**

- 一张 140×190 空槽贴图复用于全部 16 个槽位（8 个顶行槽 + 8 个空桌面列首槽）。
- 1px 轮廓、无填充、无内阴影；默认白色约 55% alpha 叠在 `#008000` 上（备选 `#C0C0C0`，以参考截图为准确认）。
- **烘焙贴图，绝不使用运行时线条**；渲染路径中不得出现 `draw_line` 或未填充的 `draw_rect`（4.7 线条无抗锯齿羽化，线宽会变细）。
- 自由单元与回收单元默认**无视觉区分**；位置与花色绑定（C、D、H、S 从左到右）是唯一线索。若无障碍需要区分，作为可选项放在 Accessibility，不改变默认牌桌。
- 建议把空槽作为卡面图集的第 53 个 tile，保证圆角与卡面一致。

**Letterbox**

- 画布未填满窗口时使用 `#000000` 黑边；`hit_test` 对画布外返回 null，视觉死区与逻辑死区一致。

**分辨率与图集**

| 档位 | 单 tile | 图集 | 说明 |
|------|---------|------|------|
| 1x | 140×190 | 2048×1024 | scale ≤1.0 |
| 2x | 280×380 | 4096×2048 | 1.0 < scale ≤2.0 |
| 3x | 420×570 | 4096×4096（9×6） | scale >2.0 |

- 矢量母版（SVG/PDF）保存；按 `scale × DisplayServer.screen_get_scale()` 选档，避免 4K + 200% OS 缩放时选错档导致模糊。
- 开启 mipmap、Linear Mipmap 过滤、Fix Alpha Border；每个 tile 外扩 4 texel（3x）/2（2x）/1（1x），并限制 mip 链不跨 tile。
- 不裁剪、不旋转打包（旋转会让等级侧躺）。
- 超过 4K 时允许放大；矢量母版可再导出 4x，不需重画。

**禁止添加**

- 阴影、投影、内阴影、高光、光泽、斜面、渐变、桌面纹理/噪点/图案、暗角、悬停抬升/染色、拖拽时的合法落点高亮、选中描边/发光、光标变化、本系统发起的任何动画、粒子/VFX、装饰性牌背、胜利/卡死覆盖层美术、窗口边框/工具栏美术、运行时线条图元。
- **唯一例外 — Hint System 提示高亮**：允许 `HintResult` 指定的源牌整段轮廓与目标槽/顶牌高亮；仅使用 9-slice 抗锯齿填充环贴图（`ui_hint_ring_9slice_*`，由 Hint System 规格定义，本系统只消费），不使用 `draw_line` / 未填充 `draw_rect`；颜色 `#000080`、3 逻辑 px 内缩、无动画、无发光/脉冲。其余禁令不变。

**音频**

- 本系统不产生音频、不拥有音频资源；发牌/落牌/胜利音效归 Audio & SFX。
- 不得从重绘路径触发音效（缩放与模型变化也会重绘）；Audio & SFX 应订阅模型/状态事件。

**资产清单（供 `/asset-spec`）**

| 资产 | 命名 | 备注 |
|------|------|------|
| 卡面图集 1x | `ui_card_faces_atlas_1x.png` | 52 张 + 空槽 tile |
| 卡面图集 2x | `ui_card_faces_atlas_2x.png` | 同布局 |
| 卡面图集 3x | `ui_card_faces_atlas_3x.png` | 同布局 |
| 紧凑索引图集 | `ui_card_compact_index_1x.png` / `_2x` / `_3x` | 被覆盖牌的 13px 等级 + 8×8 花色索引（或作为卡面图集额外 tile） |
| 牌面字体 | `ui_font_rank_bold_serif.ttf` | OFL |
| 宫廷牌源 | `ui_card_court_english-pattern.svg` | 公有领域矢量母版 |
| 桌面 / letterbox | 无资产 | 颜色常量 `#008000` / `#000000` |
| 卡背 | 无 | 不需要 |

**视觉验收**

1. 3840×2160 截图使用 3x 图集；等级笔画无灰色光晕、无重采样模糊（1:1 采样）。
2. 1024×576 截图：顶牌等级字高 ≥20 设备像素；被覆盖牌的紧凑索引等级 ≥10 设备像素、花色符号可见；红/黑一眼可辨。
3. 卡面颜色直方图只含 `#FFFFFF`、`#000000`、`#FF0000`，以及边缘 1px 内的抗锯齿混色；出现第四种颜色即缺陷。
4. 渲染路径零运行时线条。
5. 无输入时相隔 1 秒的两张截图逐像素相同（本系统无待机动画）。

## Game Feel

- **几何稳定**：卡牌位置只由模型与布局公式决定；不抖动、不随机偏移。
- **级联自适应**：13 张列压缩到 22px，所有牌仍在画布内、可点击；被覆盖牌用紧凑索引保持等级/花色可读；常见列高保持 30px。
- **无本系统动画**：模型变化即重绘；合法落位 0ms，非法回弹由 Animation & Feedback 插值；待机时画面静止。
- **缩放**：均匀缩放，无拉伸；0.8–3.0 范围内卡面清晰；窗口缩放即时重排。
- **命中一致**：视觉矩形与命中矩形使用同一几何；12px 列间隙吸收取整误差；无“看得见点不到”。
- **安静**：无悬停、无光标变化；除 Hint System 的提示环例外外无高亮；牌桌本身不制造注意力。
- **可读性**：0.8 缩放下等级与花色仍可辨；红/黑 + 花色符号双通道，不依赖颜色单一信息。
- **层级**：拖拽层在牌堆之上、工具栏/覆盖层之下；无闪烁、无穿透。

## UI Requirements

- 无直接 UI；本系统是几何/绘制服务。
- 工具栏、菜单、胜利/卡死覆盖层由 Toolbar & Menus 绘制，层级在拖拽层之上。
- Letterbox 使用黑色；不使用绿色边条冒充牌桌。
- 光标保持系统默认；无悬停、无选中轮廓；高亮仅限 Hint System 的提示环例外。
- 无障碍高对比/色盲模式在 Full Vision 由 Readability & Accessibility 提供；MVP 保持原版红/黑 + 花色符号。

## Cross-References

| 系统 / 来源 | 引用内容 | 状态 |
|-------------|----------|------|
| Card & Deck Model | 牌堆、牌、列高、回收单元花色绑定 | GDD 已完成 |
| Drag & Drop Interaction | `hit_test` / `card_rect` / 拖拽层 / 源牌隐藏 | GDD 已完成 |
| Animation & Feedback | 起止矩形、回弹插值 | GDD 未编写 |
| Toolbar & Menus | 覆盖层层级、菜单栏/对话框/横幅 | GDD 已完成（design/gdd/toolbar-menus.md） |
| Hint System | `set_hint_highlight` / `clear_hint` 提示高亮接口 | GDD 已完成（design/gdd/hint-system.md） |
| Readability & Accessibility | 高对比/色盲模式 | GDD 未编写 |
| Art Bible | 卡面/桌面/空槽视觉锚点 | 未编写 |
| Godot 4.7 rendering | `CanvasItem` 线条无抗锯齿羽化；使用贴图空槽 | `docs/engine-reference/godot/modules/rendering.md` |
| 卡面素材来源 | 公有领域 English-pattern 牌组；OFL 字体 | 需在资产阶段确认授权 |

## Acceptance Criteria

**测试约定**：逻辑画布 1280×720；卡牌 140×190；索引 0-based 自底向顶；`hit_test` 返回 `{pile, card_index}`，画布外/null；`card_rect(pile, -1)` 返回空列首槽。

### 几何与缩放

- **GIVEN** 基准画布 1280×720，**WHEN** 计算 `column_x(i)`（i=0…7），**THEN** x = 38,190,…,1102，右边距对称 38；**WHEN** 映射顶行，**THEN** 自由单元在列 0–3 上方、回收单元在 4–7 上方，`card_rect(F1,0)=(190,38,140,190)`、`card_rect(S3,0)=(1102,38,140,190)`；**GIVEN** K=7 的 T2，**WHEN** `card_rect(T2,3)`，**THEN** `(342,346,140,190)`；**GIVEN** 空列命中，**THEN** `card_index=-1`，`card_rect(T,-1)` 返回首槽 `(column_x,256,140,190)`。
- **GIVEN** 列高 K=10/11/12/13，**WHEN** 计算 `cascade_height`，**THEN** 返回 30/27/24/22，最后一牌底部 `256+(K−1)×cascade+190 ≤716`（K=13 → 710）；**WHEN** 逐张命中 13 张列，**THEN** 每张返回自己的索引，全部可点击。
- **GIVEN** 1920×1080，**WHEN** 计算 scale/offset，**THEN** scale=1.5、offset=(0,0)；**GIVEN** 1024×576，**THEN** scale=0.8、offset=(0,0)；卡牌比例始终 140:190，无非均匀拉伸。
- **GIVEN** 800×600（低于最小窗口），**WHEN** 计算，**THEN** scale 钳制 0.8、offset=(−112,12)、画布居中裁剪并记录警告；**GIVEN** 点击 letterbox（1600×1000、scale 1.25、offset (0,50)，窗口点 (800,10) → 逻辑 (640,−32)），**WHEN** `hit_test`，**THEN** 返回 null。
- **GIVEN** HiDPI/系统缩放，**WHEN** 读取 `window_w/h`，**THEN** 使用 `get_viewport().get_visible_rect().size`，不使用屏幕像素，绘制与命中测试保持对齐。
- **GIVEN** 任意逻辑点 p 与 scale/offset，**WHEN** 执行 `to_window(to_logical(p))`，**THEN** 结果与 p 相差 ≤1 ulp；例：scale 1.25、offset (0,50)，`to_logical((800,500))=(640,360)`、`to_window((640,360))=(800,500)`。

### 命中、层级与状态

- **GIVEN** 级联牌重叠，**WHEN** 点击落在多张矩形内，**THEN** 返回最高索引（最上层）；优先级为顶行槽位 → 桌面列 → null；**GIVEN** 点击在列命中区域但不在牌上（含最后一张下方 12px），**THEN** 返回顶牌索引 K−1。
- **GIVEN** 空顶行槽位，**WHEN** 命中，**THEN** `card_index=0`，可作为落点。
- **GIVEN** 拖拽开始，**WHEN** 该帧渲染，**THEN** 源牌隐藏（模型仍保留）且拖拽层同帧出现，无闪烁；拖拽层在牌堆之上、工具栏/菜单/覆盖层之下，且对命中测试透明；**GIVEN** 合法落位 0ms，**THEN** 同一帧重绘到目标位置，无过渡帧。
- **GIVEN** `NoGame`，**WHEN** 渲染，**THEN** 平面桌面 + 16 个空槽（8 顶行 + 8 桌面列首槽），无牌；**GIVEN** `Won`/`Stuck`，**THEN** 牌桌保持可见，覆盖层由 UI 绘制在拖拽层之上，状态变化前先取消拖拽。
- **GIVEN** 拖拽期间窗口缩放，**WHEN** 重算几何，**THEN** scale/offset/全部矩形只更新一次，Drag & Drop 在同一逻辑坐标下重新缩放 `grab_offset`，无错位、无闪烁。

### 视觉、资源与性能

- **GIVEN** 任意渲染帧，**WHEN** 检查，**THEN** 桌面为纯 `#008000`（无渐变/纹理/暗角）；卡面只用 `#FFFFFF`/`#000000`/`#FF0000`（边缘 1px 内允许抗锯齿混色）；1px 黑边框与 8px 圆角烘焙在图集里；渲染路径无 `draw_line` 或未填充 `draw_rect`；letterbox 为 `#000000`；无阴影/高光/tint；无卡背。
- **GIVEN** 窗口 scale 与 OS display scale，**WHEN** 选择图集，**THEN** 1x ≤1.0、2x 1.0–2.0、3x >2.0，按 `scale × DisplayServer.screen_get_scale()` 选档并开启 mipmap；3840×2160 用 3x 时等级笔画 1:1 无灰晕；1024×576 时等级字高 ≥20 设备像素、红/黑可辨。
- **GIVEN** 无输入且模型不变，**WHEN** 相隔 1 秒截两张图，**THEN** 逐像素相同（无待机动画、无逐帧重绘）；只在模型/拖拽/缩放/状态变化时重绘；拖拽只更新拖拽层位置；缩放只重排一次；绘制调用与每帧分配维持 60fps 目标。
- **GIVEN** 相同状态与输入，**WHEN** 重复计算几何与命中，**THEN** 结果完全一致（确定性纯几何）；绘制与命中共享同一逻辑浮点矩形与 scale/offset；Drag & Drop 集成测试锁定 0-based 自底向顶的 `card_index` 契约。
- **GIVEN** 空列，**WHEN** 调用 `card_rect(pile, -1)`，**THEN** 返回所在列首槽矩形；实现不得让 -1 参与 `tableau_y` 的算术。
- **GIVEN** 13 张列，**WHEN** 渲染与命中，**THEN** 使用自适应级联（22px），不是固定 30px；所有牌在画布内；被覆盖牌紧凑索引等级 ≥10 设备像素、花色可见；顶牌等级 ≥20 设备像素。

## Open Questions

| 问题 | Owner | 目标解决时间 |
|------|-------|--------------|
| 4K 以上（5K/6K）是否需要 4x 图集 | art-director + technical-artist | 资产阶段 |
| 卡面素材授权（公有领域 English-pattern 牌组） | art-director + producer | 资产阶段 |
| 空槽轮廓颜色（白 55% vs #C0C0C0） | art-director | 参考截图确认 |
| 高对比/色盲模式 | Readability & Accessibility | 该 GDD 编写时 |
| 窗口最小尺寸的平台行为 | technical-director | 架构阶段 |
| 卡面字体最终选择 | art-director | 资产阶段 |
