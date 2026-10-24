# Toolbar & Menus

> **Status**: In Design
> **Author**: 用户 + Claude Code
> **Last Updated**: 2026-10-24
> **Last Verified**: 2026-10-24
> **Implements Pillar**: 原版规则，零改动

## Summary

Toolbar & Menus 是牌桌之外的唯一命令入口：菜单栏与键盘快捷键暴露新局、选局、重开、撤销、提示、统计与退出，并把各系统的结果转成玩家可见的状态、文案与 Won/Stuck 覆盖层。MVP 采用菜单栏 + 快捷键，不添加工具栏或状态栏，保持 Table Rendering 的 1280×720 几何不变；它不改变规则、不执行移动、不拥有牌桌绘制。

> **Quick reference** — Layer: `Presentation` · Priority: `MVP` · Key deps: `Game State Machine`, `Undo History`, `Hint System`, `Statistics & Records`, `Table Rendering & Layout`, `Drag & Drop Interaction`

## Overview

Toolbar & Menus 是牌桌之外的唯一命令入口：它把新局、选局、重开、撤销、提示、统计与退出暴露为菜单项和键盘快捷键，并把 Game State Machine、Undo、Hint、Statistics 的结果转成玩家可见的状态、文案与终态覆盖层。它不改变规则、不执行移动、不拥有牌桌绘制；它服务于「原版规则，零改动」与「一秒读懂牌面」，让玩家在不离开牌桌的情况下完成所有辅助操作。

## Detailed Design

### Core Rules

**1. 范围与所有权**

- 拥有：菜单栏、命令路由、快捷键、选局对话框、统计对话框、Won 对话框、Stuck/消息横幅、窗口标题、命令启用判定、错误到文案的映射。
- 不拥有：牌桌几何/绘制（Table Rendering）、牌局生命周期（State）、撤销栈（Undo）、提示选择（Hint）、统计数据（Statistics）、拖拽输入（Drag & Drop）、模型/规则。
- 是**唯一命令入口**；不直接调用 Card & Deck Model 或 Rules。
- 所有玩家可见字符串来自字符串表（默认中文），使用 `tr()`；不硬编码。

**2. UI 表面**

- **菜单栏**：高 28 窗口坐标 px（视口单位），水平铺满窗口宽度，锚定窗口顶部；不随牌桌 letterbox 缩放。它落在 Table Rendering 顶部 38px 边距内：0.8 牌桌缩放下边距为 30.4 窗口坐标 px，菜单栏仍不覆盖顶排卡牌（见 Formulas 的 `menu_bar_fits`）。
- **无工具栏、无状态栏**；不使用牌桌以外的常驻元素。局号显示在窗口标题，临时消息使用横幅。
- **窗口标题**：有牌局时 `空当接龙 — 局号 #N`；`NoGame` 时 `空当接龙`。N 为不带千分位的十进制数字（如 `#1000000`）。在 `GameStarted` / `GameRestarted` / `GameLoaded` / `StateChanged` 后更新。
- **层级**：牌桌 < Hint 高亮 < 拖拽层 < 菜单栏/横幅 < 对话框/菜单弹层。菜单栏与横幅在拖拽层之上；对话框在最上层。

**3. 菜单与快捷键**

| 菜单 | 项 | 快捷键 | 说明 |
|---|---|---|---|
| Game | 新局 | F2 | `NoGame` 时 `new_game(1)`；否则 `next_deal()`；到上限显示消息 |
| Game | 选局… | F3 | 打开选局对话框 |
| Game | 重开本局 | F4 | `restart_deal()` |
| Game | — | — | 分隔线 |
| Game | 撤销 | Ctrl+Z | `Undo.undo(state)` |
| Game | 提示 | F10 | `Hint.request_hint(model)` |
| Game | — | — | 分隔线 |
| Game | 统计… | 无 | 打开统计对话框 |
| Game | — | — | 分隔线 |
| Game | 退出 | 无 | 立即退出；MVP 无存档、无确认 |
| Help | 关于 | 无 | 打开最小关于对话框 |

- 菜单助记：Alt+G / Alt+H；Esc 关闭菜单/对话框；Enter 确认对话框（破坏性操作除外，见各对话框）。
- **牌桌有焦点** = 没有菜单/对话框/文本输入持有焦点；此时快捷键全局生效。对话框打开时只保留 Esc/Enter 与文本框输入。
- 快捷键为**混合参考式暂定值**（Windows 7/XP 功能基线），需参考机截图确认（见 Open Questions）。

**4. 命令启用**

| 命令 | 启用条件 | 备注 |
|---|---|---|
| 新局 | 总是 | `NoGame` 也可 |
| 选局… | 总是 | 打开对话框 |
| 重开本局 | `state ∈ {Playing, Won, Stuck}` | `NoGame` / `Dealing` / `Restoring` 禁用 |
| 撤销 | `Undo.can_undo(state)` 且无拖拽 | Won/NoGame/Dealing/Restoring 禁用 |
| 提示 | `state = Playing` 且无拖拽 | 其他状态禁用 |
| 统计… | 总是 | 只读 |
| 退出 / 关于 | 总是 | — |

- 拖拽进行中：菜单栏禁用（无法打开），避免拖拽期间改变牌局；拖拽结束后恢复。
- 启用状态在 `StateChanged`、`GameStarted`、`GameRestarted`、`GameLoaded`、`UndoCommitted`、`UndoPerformed`、`UndoCleared`、`DragStarted`、`DragDropped`、`DragCancelled` 后刷新。
- **拖拽开始时**：清除 Hint 高亮；**拖拽结束**后刷新启用状态。

**5. 命令路由**

- 新局：`NoGame` → `State.new_game(1)`；否则 `State.next_deal()`。`AT_LIMIT` → 消息「已到最大局号」，不重新发牌。
- 选局：对话框确认 → `State.select_deal(N)`；成功关闭对话框；错误在对话框内显示。
- 重开：`State.restart_deal()`。
- 撤销：`Undo.undo(State.get_state())`；成功由 Undo 发出 `UndoPerformed`，State 重判；失败显示消息。
- 提示：`model = State.snapshot().model` → `Hint.request_hint(model)`。
  - 有 `move`：`TableRendering.set_hint_highlight(source, destination)`。
  - `NO_USEFUL_MOVES`：清除高亮，消息「没有可用的移动」。
  - `GAME_WON`：清除高亮，不显示提示（Won 对话框接管）。
  - 新局/重开/读档/撤销后调用 `Hint.reset()`。
  - `DragStarted`、成功移动、撤销、新局/重开/读档、状态变为 Won/Stuck/NoGame 时清除高亮。
- 统计：打开对话框时 `Statistics.get_stats()`；重置确认后 `Statistics.reset_stats()`。
- 退出：请求应用退出；MVP 不保存、不确认。
- 关于：最小模态对话框，显示游戏名与「经典复刻」；无外链、无玩法说明。

**6. 对话框**

- **选局对话框**：模态；数字输入，初始值 = 当前 N（`NoGame` 为 1）；范围 `deal_number_min..deal_number_max`（1–1,000,000）；OK/Cancel；非法输入显示行内错误并禁用 OK；Enter = OK（仅当 OK 可用），Esc = Cancel。确认调用 `State.select_deal(N)`；错误码映射为行内文案。
- **统计对话框**：模态；只读显示局数、胜局、胜率（`win_percentage_exact`）、当前连胜、最长连胜；底部「重置统计…」；重置需二次确认（「清空所有统计？不可撤销；当前进行中的局不会计入。」）；确认后 `reset_stats()` 并刷新为零。默认按钮是「关闭」，Enter 不触发重置；重置确认的默认按钮是「取消」，Enter 不重置。
- **Won 对话框**：模态；`StateChanged → Won` 时自动打开一次；标题「你赢了」，显示局号；按钮：新局（关闭 Won → 打开选局对话框）、下一局（`next_deal()`）、统计（关闭 Won → 打开统计对话框）、关闭。Enter = 关闭（默认，安全）。单模态交接：先关闭 Won 再打开目标对话框。下一局在 N=1,000,000 时显示「已到最大局号」并**保持 Won 对话框打开**。关闭后不改变状态；重复进入 Won 不重复弹窗（State 只发一次 `GameWon`）。
- **关于对话框**：模态；游戏名 + 「Windows 经典空当接龙复刻」；关闭；Enter/Esc = 关闭。
- 所有模态对话框使用**透明全窗口输入阻挡层**：牌桌不可点击，但不做背景变暗/遮罩染色。同一时间只允许一个模态；Won 事件在统计/选局对话框打开时排队，待其关闭后再打开。

**7. 横幅**

- **Stuck 横幅**：非模态；`StateChanged → Stuck` 时显示；文本「没有可用的移动」；按钮：撤销（`can_undo` 时启用）、重开本局、新局（打开选局对话框）、关闭。横幅不阻断牌桌与菜单；关闭后不再自动出现，直到下一次进入 Stuck。离开 Stuck 时自动隐藏。
- **消息横幅**：非模态；显示命令失败/提示无可用移动等短消息；无自动超时；在下一次命令、状态变化或手动关闭时消失。Stuck 横幅优先于消息横幅；两者互斥。
- 横幅位于牌桌底部居中，宽度 ≤ 480 窗口坐标 px，高度 ≤ 64 窗口坐标 px；可覆盖部分牌面，但可关闭，不阻挡牌桌输入（仅横幅自身按钮可点击）。

**8. 错误与文案**

| 来源 | 错误/原因 | 显示 |
|---|---|---|
| State | `INVALID_DEAL_NUMBER` / `DEAL_NUMBER_OUT_OF_RANGE` | 选局对话框行内：「局号必须是 1–1,000,000 的整数」 |
| State | `NO_CURRENT_GAME` | 消息横幅：「当前没有牌局」 |
| State | `GAME_ALREADY_WON` | 消息横幅：「本局已胜利」 |
| State | `MOVES_NOT_ACCEPTED` | 消息横幅：「当前状态不接受移动」 |
| State | `COMMAND_IN_PROGRESS` | 消息横幅：「操作进行中，请稍候」 |
| State | `GAME_NOT_READY` | 消息横幅：「牌局尚未准备好」 |
| State | `AT_LIMIT` | 消息横幅：「已到最大局号」 |
| Undo | `NOTHING_TO_UNDO` / `UNDO_NOT_ALLOWED` | 消息横幅：「当前不能撤销」 |
| Undo | `RESTORE_FAILED` / `INVALID_SNAPSHOT` | 消息横幅：「撤销失败，牌局未变」 |
| Hint | `NO_USEFUL_MOVES` | 消息横幅：「没有可用的移动」 |
| Hint | `GAME_WON` | 不显示；Won 对话框接管 |
| 统计 | 重置成功 | 统计对话框刷新为零；可选消息「统计已清零」 |

- 文案是默认中文，字符串键归字符串表；错误码到文案的映射集中在一处，不散落。

### States and Transitions

Toolbar 没有牌局状态；只有 UI 呈现状态。

| 状态 | 进入 | 离开 | 说明 |
|---|---|---|---|
| Table | 启动；关闭所有对话框 | 打开菜单/对话框/横幅 | 牌桌可交互 |
| MenuOpen | 点击菜单/Alt+助记 | 选择项、Esc、点击外部 | 菜单弹层 |
| DialogOpen | 选局/统计/Won/关于 | 确认、取消、Esc | 模态；透明输入阻挡 |
| BannerVisible | Stuck/消息 | 关闭、状态变化、下一次命令 | 非模态；Stuck 优先 |

- 同一时间最多一个 `DialogOpen`；菜单打开时打开对话框 → 先关闭菜单。
- `Won` 事件到达时若已有对话框 → 排队；对话框关闭后打开 Won。
- `StateChanged` 离开 Stuck → 隐藏 Stuck 横幅；离开 Won → 关闭 Won 对话框（若打开）。
- `GameStarted` / `GameRestarted` / `GameLoaded` → 关闭所有对话框/横幅，回到 Table；更新标题；`Hint.reset()`。
- 菜单/对话框状态不写入存档；MVP 无存档。

### Interactions with Other Systems

| 系统 | 方向 | 流入 | 流出 | 接口所有权 |
|---|---|---|---|---|
| Game State Machine | 上游 | 命令结果/错误、`StateChanged`、`GameStarted`、`GameRestarted`、`GameWon`、`GameStuck`、`GameLoaded`、`snapshot()` | `new_game` / `select_deal` / `restart_deal` / `next_deal` | State 拥有生命周期；Toolbar 拥有命令 |
| Undo History | 上游 | `can_undo(state)`、`UndoCommitted` / `UndoPerformed` / `UndoCleared` / `UndoFailed` | `undo(state)` | Undo 拥有历史；Toolbar 拥有入口 |
| Hint System | 上游 | `HintResult` | `request_hint(model)` / `reset()` | Hint 拥有选择；Toolbar 拥有触发与文案 |
| Statistics & Records | 上游 | `StatsSnapshot` | `get_stats()` / `reset_stats()` | Stats 拥有数据；Toolbar 拥有视图与确认 |
| Table Rendering & Layout | 上游 | `set_hint_highlight` / `clear_hint`、层级/窗口坐标 | 覆盖层/菜单/对话框 | Table 拥有几何；Toolbar 拥有 UI |
| Drag & Drop Interaction | 上游（事件） | `DragStarted` / `DragDropped` / `DragCancelled` | 无 | Drag 拥有输入；Toolbar 只读事件 |
| Animation & Feedback | 无 | — | — | MVP 无菜单/对话框动画 |
| Audio & SFX | 无 | — | — | 菜单/对话框无音效 |
| Readability & Accessibility | 下游（Full Vision） | 键盘路径/焦点 | 可访问表示 | Accessibility 拥有替代 |

**接口契约：**

- `open_select_deal()` / `open_statistics()` / `open_about()`：UI 内部命令。
- `refresh_commands()`：重新计算启用状态。
- `show_message(text_key)` / `show_stuck_banner()` / `hide_banners()`。
- Toolbar 不直接调用 Card & Deck Model 或 Rules；模型只通过 `State.snapshot()` 读取并交给 Hint。
- Toolbar 不修改 State、Undo、Hint、Statistics 的内部状态；只调用其命令。
- `set_hint_highlight(source, destination)` / `clear_hint()` 由 Table Rendering 实现（见 Hint GDD 的 Visual/Audio 接口）；Toolbar 负责转发与清除。
- 菜单/对话框使用窗口坐标；牌桌使用逻辑坐标。`to_logical` / `to_window` 由 Table Rendering 提供。

**Provisional assumptions（依赖未显式提供，先假定）：**

1. State 的命令/错误/事件契约如 State GDD；`snapshot()` 在 Playing/Won/Stuck 可用。
2. Undo 的 `can_undo(state)` 与事件如 Undo GDD；Toolbar 在 `UndoCommitted` / `UndoPerformed` / `UndoCleared` 后刷新启用。
3. Hint 的 `request_hint(model)` / `reset()` 如 Hint GDD；Table Rendering 提供 `set_hint_highlight` / `clear_hint`（已于 2026-10-24 补进 Table Rendering GDD）。
4. Statistics 的 `get_stats()` / `reset_stats()` 如 Statistics GDD；`StatsSnapshot.win_rate` 是唯一胜率来源。
5. Drag & Drop 提供 `DragStarted` / `DragDropped` / `DragCancelled` 事件；否则 Toolbar 无法可靠禁用提示/清除高亮。
6. 菜单栏使用窗口坐标、28 窗口坐标 px；在 0.8 牌桌缩放下不覆盖顶排（见 Formulas）。
7. `project.godot` 的 stretch 模式未钉死；UI 坐标策略需 ADR 固定。
8. 玩家可见字符串来自字符串表，默认中文；不硬编码。
9. MVP 无 Save/Resume UI；读档对话框留给 VS。
10. 模态对话框的透明输入阻挡层不改变牌桌绘制；不做背景变暗。

## Formulas

> **坐标单位（先读）。** 本节所有 `menu_bar_*` 公式与 `top_y × table_scale`、`offset_y` 使用**同一单位**：Table Rendering `scale` 公式中 `window_w` / `window_h` 的单位（`get_viewport().get_visible_rect().size`，OS 无关的窗口坐标 px）。菜单栏必须是根视口内的 Control，不得放在额外缩放的 `CanvasLayer` / `SubViewport` 中。OS 显示缩放 125/150/200% 在三者中约掉，0.8 下的 2.4 px 间隙不变。若 `project.godot` 的 stretch 模式未钉死（当前没有 `[display]` 段），`table_scale` 可能不是 0.8；这是阻断性架构前提，见 Dependencies / Open Questions。

**常量：** `MENU_BAR_HEIGHT = 28`（本 GDD，窗口坐标 px）；`top_y = 38`、`min_scale = 0.8`、`base_canvas_width = 1280`、`base_canvas_height = 720`（Table Rendering / registry）；`deal_number_min = 1`、`deal_number_max = 1,000,000`（registry）。最小窗口 = `(1280 × 0.8, 720 × 0.8) = (1024, 576)`，此时 `offset_y = 0`。

### The menu_bar_fits formula

The menu_bar_fits formula is defined as:

`menu_bar_fits(menu_bar_height, top_y, table_scale) = 1 if menu_bar_height ≤ top_y × table_scale, else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| menu_bar_height | int（窗口坐标 px） | 28（常量） | 本 GDD | 菜单栏总占用高度，含底部分隔线/边框 |
| top_y | int（逻辑画布 px） | 38 | Table Rendering `top_y`（registry） | 顶排卡牌上边缘的逻辑 y |
| table_scale | float | 0.8–无界 | Table Rendering `scale` 公式 | 逻辑画布 → 窗口坐标的均匀缩放 |
| menu_bar_fits | int | {0,1} | 本公式 | 1 = 菜单栏不覆盖顶排卡牌 |

**Output Range:** {0,1}。当前常量下，可达域 `table_scale ∈ [0.8, ∞)` 内**恒为 1**；最小间隙在 `table_scale = 0.8`：`38 × 0.8 − 28 = 2.4` 窗口坐标 px。仅当 `top_y × table_scale < 28` 时为 0；不 clamp。等价调参护栏：`menu_bar_height_max = floor(top_y × min_scale) = 30`；当前 28 留 2 px 余量，改成 31 就失败。这是启动断言/CI 参数化测试，不是运行时布局分支。

**Example:** `(28, 38, 0.8) → 1`（间隙 2.4，1024×576）；`(28, 38, 1.0) → 1`（间隙 10）；`(28, 38, 1.5) → 1`（间隙 29，1920×1080）；`(31, 38, 0.8) → 0`（回归用例）；`(28, 34, 0.8) → 0`（回归用例）。

> **前提：** 本式按最坏情况 `offset_y = 0` 评估。对 ≥1024×576 的窗口 `offset_y ≥ 0`，故本式保守且充分。运行期用 `menu_bar_clearance` 记录真实间隙；若返回 0，启动时记录 error 并继续绘制菜单栏，**不得**在运行期移动/压缩菜单栏。卡牌 1px 黑边烘焙在贴图内侧、Hint 高亮内缩，无外扩覆盖层，所以 `top_y × table_scale` 就是顶排视觉最上缘。

### The menu_bar_clearance formula

The menu_bar_clearance formula is defined as:

`menu_bar_clearance(menu_bar_height, top_y, table_scale, offset_y) = offset_y + top_y × table_scale − menu_bar_height`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| menu_bar_height | int（窗口坐标 px） | 28 | 本 GDD | 菜单栏总高度 |
| top_y | int（逻辑画布 px） | 38 | Table Rendering | 顶排卡牌上边缘 |
| table_scale | float | 0.8–无界 | Table Rendering | 均匀缩放 |
| offset_y | float（窗口坐标 px） | 受支持窗口 ≥ 0；裁剪时 < 0 | Table Rendering `letterbox_offset` | 画布上边缘到窗口上边缘 |
| menu_bar_clearance | float（窗口坐标 px） | 无界 | 本公式 | 菜单栏下缘到顶排上缘的净间隙 |

**Output Range:** 无界浮点；`menu_bar_fits = 1` 当且仅当 `menu_bar_clearance ≥ 0`。受支持窗口下界为 2.4（`scale = 0.8, offset_y = 0`）；窗口被裁剪时可降到负数，不 clamp，记录 warning。

**Example:** `(28, 38, 0.8, 0) → 2.4`；`(28, 38, 1.25, 50) → 69.5`（1600×1000，菜单栏整条落在黑色 letterbox 上）；`(28, 38, 0.8, −18) → −15.6`（裁剪场景，重叠）。

### The deal_number_valid formula

The deal_number_valid formula is defined as:

`deal_number_valid(N) = 0 if N = ⊥; 1 if deal_number_min ≤ N ≤ deal_number_max; 0 otherwise`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| N | int64 或 ⊥ | ⊥ 或任意 int64 | 选局对话框文本解析结果 | 候选局号；⊥ = 非法文本/溢出 |
| deal_number_min | int | 1 | registry | 下界 |
| deal_number_max | int | 1,000,000 | registry | 上界 |
| deal_number_valid | int | {0,1} | 本公式 | 1 = OK 按钮可用 |

**Output Range:** {0,1}。闭区间 `[1, 1,000,000]` 恰好为 1，两个端点为 1；`0`、`1,000,001`、`⊥` 为 0。整数要求由解析阶段承担：非整数文本在解析时变成 ⊥。

**文本解析契约：**

| 输入 | 结果 | 说明 |
|------|------|------|
| `"42"` / `" 42 "` / `"000042"` | 42 | Trim 空白；接受前导零 |
| `"0"` / `"1000001"` | 越界 | OK 禁用 |
| `"-1"` / `"+1"` / `"1.5"` / `"1e6"` / `"1,000,000"` | ⊥ | 拒绝符号、小数点、指数、千分位 |
| `"１２３"`（全角） | ⊥ | 拒绝；显示同一行内错误 |
| `""` / `"   "` | ⊥ | 空输入 |
| `"99999999999999999999"` | ⊥ | int64 溢出也解析为 ⊥，不抛异常 |

建议实现：`long.TryParse(text.Trim(), NumberStyles.None, CultureInfo.InvariantCulture, out var n)`。**State 在 `select_deal(N)` 时仍以 registry 常量重新校验**；对话框的 0/1 只是 OK 按钮的启用条件，不是权威。

**Example:** `N=1 → 1`；`N=42 → 1`；`N=1,000,000 → 1`；`N=0 → 0`；`N=1,000,001 → 0`；`N=⊥（"12.5"）→ 0`。

### The win_percentage_exact formula

The win_percentage_exact formula is defined as:

`win_percentage_exact(games_won, games_played) = 0 if games_played = 0, else min(100, floor((200 × games_won + games_played) / (2 × games_played)))`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| games_won | int32 | 0–games_played | `StatsSnapshot` | 累计胜局数 |
| games_played | int32 | 0–2,147,483,647 | `StatsSnapshot` | 累计局数 |
| win_percentage_exact | int | 0–100 | 本公式 | 精确整数半进位百分比 |

**Output Range:** 0–100。`games_played = 0` 由分支定义为 0，排除 0/0。全整数运算，无浮点、无 NaN；C# 实现须先转 `long` 再乘（`200 × games_won` 在 int32 下会溢出，int64 安全）。数学等价：`floor(100 × won / played + 0.5) = floor((200 × won + played) / (2 × played))`，所有操作数非负，整数除法即 floor。`min(100, …)` 是显示防御：不变量成立时不会触发；若计数器被外部破坏，记录 error 但不显示 >100。

**Example:** `(0,0) → 0`；`(6,8) → 75`；`(1,8) → 13`（12.5% 半进位，不是银行家舍入的 12）；`(57,200) → 29`（浮点式会得 28，这是回归见证）；`(199,200) → 100`；`(1,3) → 33`；`(1,200) → 1`。

> **与 Statistics GDD 的关系：** 本公式读取 `StatsSnapshot` 的 `games_won` / `games_played` 做整数格式化，不重算/不展示浮点 `win_rate`，也不写回统计。Statistics GDD 的「UI 必须使用 `StatsSnapshot.win_rate`」需收窄为「不得用浮点除法重算 `win_rate`；整数百分比可用整数算术从计数器格式化」（本次跨文档澄清）。

### The command_enabled formula

The command_enabled formula is defined as:

`command_enabled(command, state, can_undo, dragging) = available(command, state, can_undo) ∧ drag_permitted(command, dragging)`

`available(New, s, u) = 1`
`available(SelectDeal, s, u) = 1`
`available(Restart, s, u) = 1 if s ∈ {Playing, Won, Stuck}, else 0`
`available(Undo, s, u) = u ∧ (s ∈ {Playing, Stuck})`
`available(Hint, s, u) = 1 if s = Playing, else 0`
`available(OpenStatistics, s, u) = 1`
`available(Exit, s, u) = 1`
`available(About, s, u) = 1`
`drag_permitted(c, d) = 0 if d = 1 ∧ c ∈ {New, SelectDeal, Restart, Undo, Hint}; else 1`

**Variables:**

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| command | enum `Command` | {New, SelectDeal, Restart, Undo, Hint, OpenStatistics, Exit, About} | 菜单/快捷键 | 被查询命令 |
| state | enum `GameState` | {NoGame, Dealing, Playing, Won, Stuck, Restoring} | Game State Machine | 当前状态 |
| can_undo | int | {0,1} | `Undo.can_undo(state)` 接口版 | 1 = 有可撤销记录且状态允许（含 pending 判定） |
| dragging | int | {0,1} | Drag & Drop `DragStarted` / `DragDropped` / `DragCancelled` | 1 = 拖拽进行中 |
| command_enabled | int | {0,1} | 本公式 | 1 = 菜单项可用 / 快捷键生效 |

**Output Range:** {0,1}。定义域为 8 × 6 × 2 × 2 = 192 个组合；公式对全集有定义，可做穷举金表测试。`Restart` 在 `Dealing` / `Restoring` 为 0 是防御性的（这两个状态只存在于同步命令内部）。**拖拽门控（已决定）：** New / SelectDeal / Restart / Undo / Hint 全部由 `dragging=0` 门控；OpenStatistics / Exit / About 不门控。菜单栏本身在拖拽中不可打开（`menu_bar_openable(dragging) = ¬dragging`），这是输入层规则；`command_enabled` 仍对 Statistics/Exit/About 返回 1。

**Example:** `command_enabled(Restart, NoGame, 0, 0) = 0`；`(Restart, Won, 0, 0) = 1`；`(Undo, Playing, 1, 0) = 1`；`(Undo, Playing, 1, 1) = 0`；`(Hint, Stuck, 0, 0) = 0`；`(New, NoGame, 0, 0) = 1`；`(New, Playing, 0, 1) = 0`；`(OpenStatistics, Playing, 0, 1) = 1`。

> **`can_undo` 必须使用 Undo GDD 的接口谓词 `can_undo(state)`**（depth > 0、无 pending、state ∈ {Playing, Stuck}），不是仅深度的 `can_undo(depth)` 公式。`available(Undo)` 中的状态项是冗余保险，用于抓出误接深度版的实现：`(Undo, Won, 1, 0)` 必须为 0。

### 常量与可调性

本系统零可调参数：菜单栏高度是布局常量，命令启用与输入解析由规则固定。`menu_bar_height = 28` 是调参护栏，最大允许值 `floor(top_y × min_scale) = 30`；不得运行期改变。`top_y`、`min_scale`、`deal_number_min/max` 来自 registry，Toolbar 不重定义。`win_percentage_exact` 是显示公式，不写回统计。`menu_bar_fits`、`menu_bar_clearance`、`deal_number_valid`、`win_percentage_exact`、`command_enabled` 都是 Toolbar 内部公式，不注册为跨系统公式。

## Edge Cases

**几何与菜单栏**

- **If 窗口恰为 1024×576**：`table_scale = 0.8`、`offset_y = 0`；菜单栏下缘 28，顶排上缘 30.4，间隙 2.4 窗口坐标 px；不覆盖。保留 1024×576 截图作为证据。
- **If 16:9 大窗口（1920×1080，scale 1.5）**：`offset_y = 0`，间隙 29；不覆盖。
- **If 高于 16:9 的窗口（1600×1000，scale 1.25，offset_y = 50）**：菜单栏整条落在黑色 letterbox 上，到卡牌间隙 69.5；菜单栏必须自带不透明背景，不能假设背后是绿色桌面。
- **If 有效视口小于 1024×576（如 1920×1080 @200% OS 缩放）**：`offset_y < 0`，顶排可能压到菜单栏下或被裁掉；记录 warning，保持菜单栏位置不变，沿用 Table Rendering 的裁剪规则；不得运行期移动菜单栏。
- **If OS 显示缩放为 125/150/200%**：菜单栏、`top_y × table_scale`、`offset_y` 使用同一视口单位时，0.8 下的间隙仍为 2.4；若实测菜单栏高度 ≠ 28，用实测值重算，间隙 < 0 时记录 error。
- **If 系统字号/主题把菜单栏撑高于 28**：固定高度容器 + `custom_minimum_size.y = 28` + 固定主题字号；超出部分裁剪，或把实测高度作为 `menu_bar_height`；**不得静默重叠**。
- **If `top_y` 或 `min_scale` 被调参（如 top_y 34、min_scale 0.75）**：`menu_bar_fits` 可能为 0；由 CI 参数化测试捕获；运行期不改变布局，只记录 error。
- **If `project.godot` 的 stretch 模式未钉死**：`table_scale` 可能不是 0.8（4.7 新项目默认 `canvas_items` / `expand`）；此时 30.4 不成立，菜单栏适配公式进入 `NOT ASSESSED`，必须先按 ADR 固定 stretch/坐标策略。

**局号输入**

- **If 输入为空或全空白**：`⊥` → OK 禁用，不调用 `State.select_deal`。
- **If 输入含小数点、符号、千分位、指数或全角数字**：`⊥` → OK 禁用，显示与越界相同的行内文案。
- **If 输入超出 int64**：`⊥`，不抛异常。
- **If 输入恰为 1 或 1,000,000**：合法，OK 可用。
- **If 输入为前导零（`"000042"`）**：按解析契约接受为 42。
- **If 对话框判定合法但 State 拒绝（registry 常量漂移）**：以 State 的错误码为准，行内显示；OK 可用性不是权威。
- **If 选择当前局号 N**：`State.select_deal(N)` 路由为 `restart_deal()`（`GameRestarted`，不记 loss、不增局数）；Toolbar 不做统计特判。
- **If 局号已到 1,000,000 时按 F2**：`next_deal` 返回 `AT_LIMIT`，不重新发牌、不产生 `GameStarted`、统计不变；显示「已到最大局号」；New 仍保持启用。

**胜率与统计对话框**

- **If `games_played = 0`**：`win_rate = 0.0` → 显示 0%，绝不 NaN。
- **If `games_won = 1, games_played = 8`**：显示 **13%**（精确半进位，不是银行家舍入的 12）。
- **If `games_won = 57, games_played = 200`**：必须显示 **29%**；浮点式会显示 28%——这是回归用例。
- **If `games_won = 199, games_played = 200`**：显示 100%，尽管 `won < played`；这是整数半进位的既定显示事实。
- **If 计数器违反 Statistics 不变量（如 `won > played`）**：`min(100, …)` 保证不显示 >100；记录 error，不写回 Statistics。
- **If 统计对话框打开期间发生 `reset_stats()`**：公式是纯函数；下次读取快照显示 0%，不逐帧刷新。
- **If 统计对话框打开时游戏被外部改变**：对话框不自动刷新；只有关闭再打开时重读快照。

**命令与状态**

- **If `NoGame`**：New/Select/Statistics/Exit/About 可用；Restart/Undo/Hint 禁用。
- **If `Won`**：Restart 可用；Undo（即使误传深度版 `can_undo = 1`）与 Hint 禁用；Won 对话框在输入层拦截其余命令。
- **If `Stuck`**：Restart 可用；Undo 依 `can_undo(state)`；Hint 禁用；Stuck 横幅显示。
- **If `Playing` 且拖拽中**：New/Select/Restart/Undo/Hint 全部禁用；Statistics/Exit/About 的 `command_enabled` 仍为 1，但菜单栏在拖拽中不可打开，快捷键也不存在这些命令。
- **If Undo 有 pending 记录**：`can_undo(state) = false` → 禁用，即使 depth > 0。
- **If `Playing` 但 `hint_candidate_count = 0`**：Hint 仍启用，返回 `NO_USEFUL_MOVES`，消息横幅显示「没有可用的移动」；不按 `hint_available` 变灰。
- **If 命令在菜单打开期间变为禁用（如 `DragStarted`）**：关闭菜单并刷新启用状态，不留下可点击的陈旧项。

**对话框与横幅**

- **If 已有模态对话框打开时收到 Won**：Won 对话框排队；当前对话框关闭后再打开，不叠加。
- **If Stuck 横幅与消息横幅同时存在**：Stuck 优先；消息横幅在 Stuck 消失后不自动恢复（已被覆盖）。
- **If 消息横幅显示期间发生状态变化**：消息横幅立即隐藏；Stuck 横幅按新状态显示/隐藏。
- **If 关闭模态对话框**：焦点返回牌桌；快捷键恢复；不改变牌局状态。
- **If 在牌局进行中退出**：MVP 立即退出，不确认、不保存；会话统计丢失。
- **If 拖拽中打开菜单**：菜单栏不可打开；拖拽结束后恢复。
- **If 菜单栏在 letterbox 上**：菜单栏自带不透明背景，不依赖牌桌颜色。
- **If 4.6 双焦点下用鼠标与键盘混合操作**：菜单选择、对话框按钮、快捷键都必须可用；鼠标悬停与键盘焦点可同时存在，不互相清除。

## Dependencies

**上游依赖：**

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Game State Machine | Hard | `new_game` / `select_deal` / `restart_deal` / `next_deal`、`get_state`、`snapshot()`、错误码、`StateChanged` / `GameStarted` / `GameRestarted` / `GameWon` / `GameStuck` / `GameLoaded` | State 拥有生命周期；Toolbar 拥有命令 |
| Undo History | Hard | `undo(state)`、`can_undo(state)`、`UndoCommitted` / `UndoPerformed` / `UndoCleared` / `UndoFailed` | Undo 拥有历史；Toolbar 拥有入口 |
| Hint System | Hard | `request_hint(model)`、`reset()`、`HintResult` | Hint 拥有选择；Toolbar 拥有触发与文案 |
| Statistics & Records | Hard | `get_stats()`、`reset_stats()`、`StatsSnapshot` | Stats 拥有数据；Toolbar 拥有视图与确认 |
| Table Rendering & Layout | Hard | `set_hint_highlight` / `clear_hint`、`top_y` / `min_scale` / `scale` / `letterbox_offset`、`to_logical` / `to_window`、层级 | Table 拥有几何；Toolbar 拥有 UI |
| Drag & Drop Interaction | Soft（事件） | `DragStarted` / `DragDropped` / `DragCancelled` | Drag 拥有输入；Toolbar 只读事件 |

**下游依赖（依赖本系统的系统）：**

| 系统 | 类型 | 接口 | 备注 |
|------|------|------|------|
| Readability & Accessibility | Soft（Full Vision） | 键盘路径、焦点、菜单/对话框可访问表示 | Accessibility 拥有替代；MVP 只保证键盘可达 |
| Animation & Feedback | None | — | MVP 菜单/对话框无动画 |
| Audio & SFX | None | — | 菜单/对话框无音效 |
| Settings | None | — | 无设置入口 |
| Save/Resume Game | None（MVP） | — | VS 的读档 UI 另议 |

**接口契约：**

- Toolbar 是唯一命令入口；不直接调用 Card & Deck Model 或 Rules。
- 提示所需模型通过 `State.snapshot().model` 读取，再传给 `Hint.request_hint(model)`；Toolbar 不缓存模型。
- `can_undo` 必须使用 Undo GDD 的接口谓词 `can_undo(state)`（depth > 0、无 pending、state ∈ {Playing, Stuck}），不是仅深度的 `can_undo(depth)`。
- `StatsSnapshot` 是唯一统计数据来源；`win_percentage_exact` 只读取 `games_won` / `games_played` 做整数格式化，不写回。
- `set_hint_highlight(source, destination)` / `clear_hint()` 由 Table Rendering 实现（接口定义见 Hint GDD 的 Visual/Audio）；Toolbar 负责转发与清除。
- 菜单/对话框使用窗口坐标；牌桌使用逻辑坐标；`to_logical` / `to_window` 由 Table Rendering 提供。
- Toolbar 不修改 State / Undo / Hint / Statistics 的内部状态；只调用其命令。
- `previous_deal` 不暴露给玩家（无菜单入口）；`next_deal` 只由「新局」使用。

**双向一致性：**

- Game State Machine 将 Toolbar 列为 Hard 下游；Undo、Hint、Statistics 同样列为 Hard 下游；本 GDD 将四者列为 Hard 上游。方向不同，强度不同，二者一致。
- Table Rendering 将 Toolbar 列为 UI 下游（覆盖层/层级）；本 GDD 将 Table Rendering 列为 Hard 上游（几何/高亮/坐标）。方向不同，强度不同，二者一致。
- **系统索引：** 已在 2026-10-24 将 Toolbar 行依赖补为 `4, 5, 6, 7, 8, 10`（Table Rendering 为 Hard，Drag & Drop 为 Soft 事件）。
- **Table Rendering 接口：** `set_hint_highlight` / `clear_hint` 已于 2026-10-24 补进 Table Rendering GDD 的 Interactions/接口表与 Cross-References。
- Drag & Drop GDD 的 Interactions 表未列 Toolbar；本 GDD 只依赖其 `DragStarted` / `DragDropped` / `DragCancelled` 事件。若 Drag 不提供这两个事件，需在 Drag GDD 补齐或改为 Toolbar 轮询拖拽状态（不推荐）。
- `project.godot` 无 `[display]` 段；stretch 模式未钉死。Table Rendering 的 `scale` 公式与 `menu_bar_fits` 依赖它，必须先由 ADR 固定（见 Open Questions）。

**Provisional assumptions（依赖未显式提供，先假定）：**

1. State 的命令/错误/事件契约如 State GDD；`snapshot()` 在 Playing/Won/Stuck 可用。
2. Undo 的 `can_undo(state)` 与事件如 Undo GDD；Toolbar 在 `UndoCommitted` / `UndoPerformed` / `UndoCleared` 后刷新启用。
3. Hint 的 `request_hint(model)` / `reset()` 如 Hint GDD；Table Rendering 提供 `set_hint_highlight` / `clear_hint`（已于 2026-10-24 补进 Table Rendering GDD）。
4. Statistics 的 `get_stats()` / `reset_stats()` 如 Statistics GDD；`StatsSnapshot` 计数器满足不变量。
5. Drag & Drop 提供 `DragStarted` / `DragDropped` / `DragCancelled` 事件。
6. 菜单栏与 Table Rendering 的 `top_y` / `table_scale` / `offset_y` 使用同一视口单位；OS 显示缩放在比较中约掉。
7. `project.godot` 的 stretch 模式在实现前由 ADR 固定（推荐 `stretch/mode = "disabled"` 或 Table 改读 `DisplayServer.window_get_size()`）；在此之前 `menu_bar_fits` 标 `NOT ASSESSED`。
8. 玩家可见字符串来自字符串表，默认中文；不硬编码。
9. MVP 无 Save/Resume UI；读档对话框留给 VS。
10. 模态对话框使用透明全窗口输入阻挡层，不做背景变暗；菜单栏高度固定 28，主题字号不改变它。

**Registry references：** `top_y`、`min_scale`、`deal_number_min`、`deal_number_max`、`win_rate` 的 `referenced_by` 已追加 `design/gdd/toolbar-menus.md`。可选新增常量 `menu_bar_height = 28`（unit: `window_px`，source: toolbar-menus.md）；若 Table Rendering 的顶部边距理由引用它，其 `referenced_by` 也含 `table-rendering-layout.md`。`menu_bar_fits`、`menu_bar_clearance`、`deal_number_valid`、`win_percentage_exact`、`command_enabled` 均为 Toolbar 内部公式，不注册。

## Visual/Audio Requirements

**视觉判据**：任何 UI 元素都必须能在 Windows 经典空当接龙里找到出处；找不到就不加。菜单栏、对话框与横幅使用经典 Windows 灰 `#C0C0C0`、黑字、1px 凹凸边框；选中项 navy `#000080` + 白字；**无**渐变、阴影、圆角、发光、半透明、悬停位移。

- **菜单栏**：不透明 `#C0C0C0`，高 28 窗口坐标 px，水平铺满；系统 UI 字体 14 窗口 px，黑字；选中项 navy/白字；禁用项 50% 灰字；分隔线 1px 暗/亮双线。在 letterbox 上也必须不透明，不能透出黑色或绿桌。
- **对话框**：不透明 `#C0C0C0` 面板 + 1px 凹凸边框；黑字；居中；**不做背景变暗/遮罩染色**；按钮为经典灰底黑字；键盘焦点用 1px 点线矩形（黑或 navy）标出；无圆角。
- **Stuck/消息横幅**：不透明 `#C0C0C0` + 1px 边框，黑字，底部居中，≤480×64 窗口坐标 px；按钮经典样式；不做变暗。
- **Won 对话框**：同灰色面板；标题「你赢了」；无彩带、无庆祝动画、无音效。
- **字体与缩放**：菜单/对话框使用窗口坐标，不随牌桌 0.8 缩放；系统 UI 字号 14 px，随 OS 显示缩放；文本自动换行；中文默认，字符串来自字符串表。
- **无新增资产**：使用 Godot 主题/StyleBox（纯色 + 边框），不新增贴图、不新增音频事件。菜单点击/对话框开关/横幅出现均无声音。
- **视觉验收**：1024×576、1920×1080、1600×1000 三档截图；菜单栏不透明、不覆盖顶排；对话框外像素不变；键盘焦点可见。截图保留在 `production/qa/evidence/toolbar/`。

> 本系统无新增视觉资产，不需要 `/asset-spec`。

## Game Feel

- **即时。** 菜单、对话框、横幅、快捷键都在输入后的同帧响应；无淡入、无滑出、无加载动画、无音效。
- **不打断。** Stuck/消息横幅是非模态的，牌桌仍可操作；只有真正的模态（选局/统计/Won/关于）才阻挡牌桌，且用透明输入层而非变暗。
- **安全默认。** Enter 在统计、重置确认、Won、关于中都是「关闭/取消」，不会误触破坏性操作；New/Restart 不弹确认，保持原版一两步重开。
- **焦点可预期。** 关闭对话框后焦点回到牌桌，快捷键立即恢复；4.6 双焦点下鼠标悬停与键盘焦点互不清除；焦点始终可见。
- **安静。** 菜单/对话框不发声、不闪动、不脉冲；Won 不庆祝，Stuck 不惩罚。
- **可读。** 0.8 牌桌缩放下菜单/对话框文字仍按窗口字号显示，不随牌桌缩小；1024×576 下不覆盖顶排。
- **无重复触发问题。** 快捷键不做防抖；模态打开时快捷键被抑制；菜单打开时注入 `DragStarted` 会关闭菜单并刷新，不留下陈旧可点击项。
- **混合参考节奏。** 菜单/快捷键的行为与混合参考（Windows 7/XP 功能基线）一致：F2 下一局、F3 选局、F4 重开、Ctrl+Z 撤销、F10 提示；不添加现代化快捷面板。

## UI Requirements

- **表面**：只提供菜单栏 + 键盘快捷键；**无工具栏、无状态栏**。窗口标题显示 `空当接龙 — 局号 #N`（无千分位；`NoGame` 为 `空当接龙`）。
- **菜单**：Game（新局、选局…、分隔线、撤销、提示、分隔线、统计…、分隔线、退出）；Help（关于）。无上/下一局、无玩法说明、无教程。Alt+G / Alt+H 打开菜单。
- **快捷键**：F2 新局、F3 选局、F4 重开、Ctrl+Z 撤销、F10 提示；统计仅菜单；Esc 关闭菜单/对话框。牌桌有焦点时全局生效；模态打开时只保留 Esc/Enter 与文本框输入。
- **默认按钮**：选局 Enter=OK（仅 OK 可用）、Esc=Cancel；统计 Enter=关闭、重置确认默认 Cancel；Won Enter=关闭；关于 Enter/Esc=关闭。破坏性操作不得是默认按钮。
- **对话框**：选局、统计、Won、关于均为模态；透明全窗口输入阻挡层（不变暗）；同时只允许一个模态；Won 在已有模态打开时排队；单模态交接（Won 点新局/统计先关闭自身再开目标）。
- **横幅**：Stuck 横幅底部居中、非模态、≤480×64 窗口坐标 px；按钮撤销/重开本局/新局/关闭；关闭后直到下次 Stuck 不再出现。消息横幅同区域、无自动超时、下一条命令/状态变化/手动关闭消失；Stuck 优先。
- **焦点**：牌桌有焦点 = 无菜单/对话框/文本输入持焦；关闭对话框后焦点回牌桌；4.6 双焦点下鼠标悬停与键盘焦点分离且都可见；所有菜单项/按钮键盘可达。
- **本地化**：所有玩家可见字符串走字符串表 + `tr()`，默认中文；自动换行；不在源码硬编码。错误码到文案的映射集中一处。
- **无障碍**：键盘可达；焦点可见；0.8 牌桌缩放下 UI 按窗口字号显示；胜率以数字显示，不依赖颜色；高对比/色盲模式在 Full Vision 由 Readability & Accessibility 提供。
- **不做**：常驻 HUD、工具栏、状态栏、设置入口、教程、排行榜、统计导出、提示气泡、菜单动画/音效。
- **窗口与缩放**：菜单栏/对话框使用窗口坐标，不随牌桌 letterbox 缩放；在 1024×576、1920×1080、1600×1000 与 125/150/200% OS 缩放下保持可读、不覆盖顶排。

## Cross-References

| 文档 | 章节 | 关系 |
|------|------|------|
| `design/gdd/game-state-machine.md` | Core Rules §3/§7、Dependencies | 命令、错误码、`StateChanged` / `GameStarted` / `GameRestarted` / `GameWon` / `GameStuck` / `GameLoaded`、`snapshot()` |
| `design/gdd/undo-history.md` | Interactions / Formulas / Dependencies | `undo(state)`、`can_undo(state)`、Undo 事件；必须用接口谓词而非仅深度 |
| `design/gdd/hint-system.md` | Detailed Design / Visual-Audio / UI | `request_hint(model)`、`reset()`、`HintResult`、高亮接口与清除时机 |
| `design/gdd/statistics-records.md` | Detailed Design / Formulas / UI | `get_stats` / `reset_stats`、`StatsSnapshot`；本次已收窄胜率显示措辞，允许 `win_percentage_exact` 整数格式化 |
| `design/gdd/table-rendering-layout.md` | Formulas / UI Requirements / 禁止添加 | `top_y` / `min_scale` / `scale` / `letterbox_offset`、层级；已补 `set_hint_highlight` / `clear_hint` 接口 |
| `design/gdd/drag-drop-interaction.md` | Interactions / Edge Cases | `DragStarted` / `DragDropped` / `DragCancelled`；Toolbar 依赖这些事件做拖拽门控与高亮清除 |
| `design/gdd/game-concept.md` | Pillars / Anti-Pillars / Visual Identity | 菜单栏 + 快捷键的原版 UI；无工具栏/状态栏/教程/成就；1280×720、0.8 最小缩放 |
| `design/gdd/systems-index.md` | Toolbar 行 / Dependency Map / Progress | 依赖已补 5（Drag & Drop）与 10（Table Rendering）；MVP 10/10 |
| `design/registry/entities.yaml` | formulas / constants | `top_y`、`min_scale`、`deal_number_min/max`、`win_rate` 的 `referenced_by`；可选 `menu_bar_height=28` |
| `project.godot` | `[display]`（缺失） | stretch 模式未钉死；`menu_bar_fits` / 几何 AC 在 ADR 前 `NOT ASSESSED` |

**已知跨文档事项：**

- Statistics GDD 的「UI 必须使用 `StatsSnapshot.win_rate`」已收窄为「不得用浮点除法重算；整数百分比可用计数器格式化」，以支持 `win_percentage_exact`。
- Table Rendering GDD 已补 `set_hint_highlight` / `clear_hint`（2026-10-24）；接口定义见 Hint GDD。
- 系统索引 Toolbar 行已补 5 与 10（2026-10-24）；本 GDD 依赖 Drag & Drop（Soft 事件）与 Table Rendering（Hard 几何/层级）。
- Drag & Drop 的事件是 `DragStarted` / `DragDropped` / `DragCancelled`，不是 `DragEnded`；本 GDD 已改用前者。
- `project.godot` 无 `[display]` 段；stretch 模式未钉死，是 `menu_bar_fits` 与 30.4 间隙数字的阻断性前提。
- 混合参考快捷键（F2/F3/F4/Ctrl+Z/F10）是暂定值，需参考机截图确认。

## Acceptance Criteria

> **测试约定。** 状态：NoGame / Dealing / Playing / Won / Stuck / Restoring（Dealing/Restoring 为同步瞬态，用夹具注入）。命令：New / SelectDeal / Restart / Undo / Hint / OpenStatistics / Exit / About。快捷键：F2 / F3 / F4 / Ctrl+Z / F10、Alt+G / Alt+H、Esc。`can_undo` = depth > 0 ∧ 无 pending ∧ state ∈ {Playing, Stuck}（接口版，不是仅深度）。`State.snapshot()` = `{N, model, state}`；`HintResult` = `{has_hint, move, reason}`。高亮接口：`set_hint_highlight(source, destination)` / `clear_hint()`；destination 索引：空桌面列 −1、空顶行槽 0、非空牌堆 height−1。常量：菜单栏 28 窗口 px、`top_y=38`、`min_scale=0.8`、画布 1280×720、局号 1–1,000,000、最小窗口 1024×576。
>
> **测试层级。** 公式单测 `tests/unit/toolbar/`（BLOCKING）；行为集成 `tests/integration/toolbar/`（BLOCKING）；每次 UI 变更截图保留在 `production/qa/evidence/toolbar/` 并由负责人签收（UI/Visual BLOCKING，`qa.level: minimal` 也不豁免）；架构检查为阻塞项；性能在参考机钉死前 CI 为 ADVISORY。

### Core Rules

- **AC-C01（字符串表）** GIVEN 任意界面，WHEN 检查所有菜单/对话框/横幅/标题文本，THEN 均来自字符串表键值；源码无未经 `tr()` 的玩家可见字面量；替换某键为哨兵串后 UI 出现哨兵。
- **AC-C02（UI 表面）** GIVEN 1280×720 与 1920×1080，WHEN 测量菜单栏，THEN 高 28 窗口 px、宽=视口宽、上缘 y=0；1920×1080 下不得为 42（不随 1.5 缩放）；牌桌外常驻元素只有菜单栏，无工具栏/状态栏。
- **AC-C03（窗口标题）** GIVEN `NoGame`，THEN 标题为「空当接龙」；GIVEN Playing/Won/Stuck，THEN 为「空当接龙 — 局号 #N」且数字=N、无千分位；在 `GameStarted` / `GameRestarted` / `GameLoaded` / `StateChanged` 后立即反映新 N。
- **AC-C04（层级与不透明）** GIVEN 菜单弹层/模态/横幅，THEN 菜单弹层不透明覆盖牌面；模态打开时对话框矩形外像素与关闭时逐像素相同（透明阻挡、无变暗），对话框在最上层；横幅在牌面之上、仅自身按钮可点击；1600×1000 时菜单栏 0–28 整条为自身不透明背景、无 letterbox/绿桌透出。
- **AC-C05（菜单结构）** GIVEN 菜单栏，THEN Game 顺序恰为 新局、选局…、分隔线、撤销、提示、分隔线、统计…、分隔线、退出；Help 仅「关于」；无上/下一局；分隔线不可点。
- **AC-C06（快捷键）** GIVEN 牌桌有焦点、Playing，WHEN 按 F2，THEN 开 N+1 局；F3 → 选局对话框；F4 → 同 N 重发且 Undo 清空；Ctrl+Z（可用时）→ 撤销一步；F10 → 高亮或「没有可用的移动」；Alt+G/Alt+H 开菜单；Esc 关闭；结果与菜单项一致。
- **AC-C07（对话框快捷键抑制）** GIVEN 模态打开，WHEN 按 F2/F3/F4/Ctrl+Z/F10，THEN 无命令执行（局号/状态/Undo 深度/统计不变），不打开第二个对话框；Esc 关闭且不确认；文本输入正常。
- **AC-C08（状态×命令启用）** GIVEN `dragging=0`，THEN：NoGame/Dealing/Restoring 下 New/Select/Stats/Exit/About 可用、Restart/Undo/Hint 禁用；Playing（can_undo=0）除 Undo 外均可用；Playing（can_undo=1）Undo 可用；Won（即使注入 can_undo=1）Undo/Hint 禁用、其余可用；Stuck（can_undo=0）Undo/Hint 禁用、其余可用；Stuck（can_undo=1）Undo 可用、Hint 禁用。逐项点击+快捷键验证。
- **AC-C09（拖拽门控）** GIVEN 拖拽中，THEN 菜单不可打开、New/Select/Restart/Undo/Hint 五个快捷键无效果；`DragStarted` 同帧清除高亮；`DragDropped` / `DragCancelled` 后恢复；Statistics/Exit/About 的 `command_enabled=1` 但 UI 不可达（刻意不对称，测试不得据此断言可达）。
- **AC-C10（启用刷新）** GIVEN 任意下列事件，WHEN 下一次输入前，THEN 启用已刷新：StateChanged、GameStarted、GameRestarted、GameLoaded、UndoCommitted、UndoPerformed、UndoCleared、DragStarted、DragDropped、DragCancelled。见证：commit 后 Undo 可用、撤销回 0 后禁用、DragStarted 五项禁用、进入 Won 后 Hint 禁用/Restart 可用；菜单打开时注入 DragStarted 则菜单关闭且无陈旧可点击项。
- **AC-C11（can_undo 谓词）** GIVEN Playing depth=1 无 pending → Undo 可用；有 pending → 禁用；Won depth=1 → 禁用；Stuck depth=1 无 pending → 可用；NoGame/Dealing/Restoring → 禁用。
- **AC-C12（Hint 不按候选数变灰）** GIVEN 注入 Playing 且 `hint_candidate_count=0`，THEN Hint 仍可用；F10 显示「没有可用的移动」且无高亮。
- **AC-C13（新局）** GIVEN NoGame，THEN `new_game(1)`；GIVEN Playing/Won/Stuck 且 N<1,000,000，THEN `next_deal()`；GIVEN N=1,000,000，THEN 横幅「已到最大局号」、不重发、无 `GameStarted`、统计不变、New 仍可用。
- **AC-C14（重开）** GIVEN Playing/Won/Stuck，THEN `restart_deal()`：同 N、`UndoCleared`、`GameRestarted`、标题不变、`Hint.reset()`、关闭所有对话框/横幅、回到 Playing（除非立即获胜）。
- **AC-C15（撤销）** GIVEN Undo 可用，WHEN 触发，THEN 恰调用 `undo(get_state())`；成功 → `UndoPerformed`、牌面恢复、State 重判、高亮清除、`Hint.reset()`、刷新；注入 `NOTHING_TO_UNDO` / `UNDO_NOT_ALLOWED` → 「当前不能撤销」；`RESTORE_FAILED` / `INVALID_SNAPSHOT` → 「撤销失败，牌局未变」。
- **AC-C16（提示路由）** GIVEN 触发 Hint，THEN `State.snapshot()` 恰一次，其 `.model` 同一引用传给 `request_hint`；有 `has_hint` → `set_hint_highlight` 恰一次（source=cards 源堆完整顶部索引段，destination 按 −1/0/height−1）、无横幅；`NO_USEFUL_MOVES` → `clear_hint()` + 「没有可用的移动」；`GAME_WON` → `clear_hint()`、无横幅；拖拽中不调用、不显示。
- **AC-C17（高亮生命周期）** GIVEN 已显示高亮，THEN 下一次提示/拖拽开始/成功移动/撤销/新局/重开/读档/进入 Won/Stuck/NoGame → 同帧替换或清除；缩放/焦点变化/暂停 → 保留并重绘；`Hint.reset()` 后下一次 `cycle_index=0`。
- **AC-C18（统计对话框）** GIVEN 打开，THEN `get_stats()` 恰一次，只读显示局数/胜局/胜率/当前连胜/最长连胜；胜率 = `win_percentage_exact(won, played)` 的 0–100 整数加 `%`，不得浮点重算 `win_rate`；重置确认文案恰为「清空所有统计？不可撤销；当前进行中的局不会计入。」；取消 → 不调用；确认 → `reset_stats()` 恰一次并刷为 `0/0/0%/0/0`；打开期间外部变化不自动刷新，关闭再开才重读。
- **AC-C19（退出）** GIVEN 任意状态，WHEN 退出，THEN 立即退出、无确认、不写文件；重启后统计全零、状态 NoGame。
- **AC-C20（关于）** GIVEN 打开关于，THEN 模态显示「空当接龙：经典复刻」与「Windows 经典空当接龙复刻」；无外链、无玩法说明；Enter/Esc 关闭；牌桌被阻挡。
- **AC-C21（选局对话框）** GIVEN 打开，THEN 预填当前 N（NoGame 为 1），OK/Cancel；Esc/Cancel → 不调用；合法+OK → `select_deal(N)` 恰一次、成功关闭；State 错误 → 保持打开、行内「局号必须是 1–1,000,000 的整数」、状态不变；对话框判合法但 State 拒绝 → 以 State 为准；非法输入 → OK 禁用且不调用。
- **AC-C22（Won 对话框）** GIVEN `GameWon`，THEN 恰打开一次；标题「你赢了」、显示 N、按钮 新局/下一局/统计/关闭；已有模态时排队，关闭后再开；重复 `GameWon` 不弹第二次；关闭后仍 Won；离开 Won 关闭；下一局 N<max → `next_deal()`，N=max → 横幅「已到最大局号」且对话框保持打开；新局先关闭 Won 再开选局；统计先关闭 Won 再开统计。
- **AC-C23（模态与焦点）** GIVEN 模态打开，THEN 点击外部牌桌无交互（不选牌/拖拽/移动），外部像素与关闭时逐像素相同；同一时间至多一个模态；关闭后焦点回牌桌、快捷键恢复；开关对话框不改变牌局状态。
- **AC-C24（Stuck 横幅）** GIVEN 转为 Stuck，THEN 底部居中非模态显示「没有可用的移动」，按钮 撤销（can_undo 时可用）/重开本局/新局（打开选局）/关闭；不阻断牌桌与菜单；关闭后直到下次 Stuck 不再出现；离开 Stuck 自动隐藏。
- **AC-C25（消息横幅）** GIVEN 命令失败/无提示，THEN 横幅出现，宽≤480、高≤64 窗口 px，无自动超时；下一次命令/状态变化/手动关闭消失；Stuck 优先且消息被丢弃不恢复；横幅可见时牌桌其余输入可用。
- **AC-C26（错误文案映射）** GIVEN 各错误码，THEN 显示：`INVALID_DEAL_NUMBER`/`DEAL_NUMBER_OUT_OF_RANGE` → 选局行内「局号必须是 1–1,000,000 的整数」；`NO_CURRENT_GAME` → 「当前没有牌局」；`GAME_ALREADY_WON` → 「本局已胜利」；`MOVES_NOT_ACCEPTED` → 「当前状态不接受移动」；`COMMAND_IN_PROGRESS` → 「操作进行中，请稍候」；`GAME_NOT_READY` → 「牌局尚未准备好」；`AT_LIMIT` → 「已到最大局号」；`NOTHING_TO_UNDO`/`UNDO_NOT_ALLOWED` → 「当前不能撤销」；`RESTORE_FAILED`/`INVALID_SNAPSHOT` → 「撤销失败，牌局未变」；`NO_USEFUL_MOVES` → 「没有可用的移动」；`GAME_WON` → 不显示。映射集中一处（架构检查）。

### Formulas

- **AC-F01（menu_bar_fits）** GIVEN `(28,38,0.8)` / `(28,38,1.0)` / `(28,38,1.5)`，THEN 均为 1；GIVEN `(31,38,0.8)` / `(28,34,0.8)`，THEN 均为 0；GIVEN `(28,35,0.8)`（乘积恰 28），THEN 1；输出 {0,1}、不 clamp。
- **AC-F02（menu_bar_clearance）** GIVEN `(28,38,0.8,0)` → 2.4；`(28,38,1.5,0)` → 29；`(28,38,1.25,50)` → 69.5；`(28,38,0.8,−18)` → −15.6；容差 1e−9；`clearance ≥ 0` 当且仅当 `fits = 1`。
- **AC-F03（deal_number_valid）** GIVEN `N=1/42/1,000,000` → 1；`N=0/1,000,001/⊥` → 0；不 clamp。
- **AC-F04（解析契约）** GIVEN `"42"` / `" 42 "` / `"000042"` → 42 合法；`"1"` / `"1000000"` 合法；`"0"` / `"1000001"` 越界；`"-1"` / `"+1"` / `"1.5"` / `"1e6"` / `"1,000,000"` / `"１２３"` / `""` / `"   "` / `"99999999999999999999"` → ⊥、OK 禁用、不抛异常；OK 可用当且仅当结果为 1。
- **AC-F05（win_percentage_exact）** GIVEN `(0,0)`→0；`(6,8)`→75；`(1,8)`→13（半进位，不是银行家 12）；`(57,200)`→29（浮点式会得 28）；`(199,200)`→100；`(1,3)`→33；`(1,200)`→1；`(0,5)`→0；`(5,5)`→100；`(201,200)`→100（`min` 防御）；`(2,000,000,000, 2,147,483,647)`→93（int64 防溢出）；全整数、无 NaN、0–100。
- **AC-F06（command_enabled）** GIVEN 8×6×2×2=192 组合，WHEN 求值，THEN 与金表逐项一致；见证：`(Restart,NoGame,0,0)=0`、`(Restart,Won,0,0)=1`、`(Undo,Playing,1,0)=1`、`(Undo,Playing,1,1)=0`、`(Undo,Won,1,0)=0`、`(Hint,Stuck,0,0)=0`、`(Hint,Playing,0,0)=1`、`(New,NoGame,0,0)=1`、`(New,Playing,0,1)=0`、`(OpenStatistics,Playing,0,1)=1`、`(Exit,NoGame,0,1)=1`、`(About,Restoring,0,1)=1`；`drag_permitted=0` 仅当 `dragging=1` 且命令 ∈ {New,SelectDeal,Restart,Undo,Hint}。

### Geometry（截图证据）

- **AC-G01（1024×576）** GIVEN 最小窗口，THEN `scale=0.8`、`offset=(0,0)`、菜单下缘 28、顶排上缘 30.4、间隙 2.4、不重叠；截图 `production/qa/evidence/toolbar/1024x576.png`。
- **AC-G02（1920×1080）** GIVEN 16:9 大窗口，THEN `scale=1.5`、`offset=(0,0)`、顶排上缘 57、间隙 29；菜单高仍 28 而非 42；截图。
- **AC-G03（1600×1000）** GIVEN 高于 16:9，THEN `scale=1.25`、`offset=(0,50)`、菜单整条落在黑色 letterbox（0–28 在 0–50 内）、间隙 69.5、菜单不透明；截图。
- **AC-G04（裁剪视口 960×540）** GIVEN 如 1920×1080@200%，THEN `scale` 钳 0.8、`offset_y=−18`、间隙 −15.6；记录 warning；菜单保持 y=0 高 28，不运行期移动/压缩；牌桌按 Table 裁剪；截图。
- **AC-G05（OS 缩放 125/150/200%）** GIVEN 视口 1024×576，THEN 菜单高仍 28、间隙 2.4；实测 ≠28 则按实测重算，间隙 <0 记 error，不得静默重叠；每档截图；环境无法设置则 `NOT ASSESSED`。

### Cross-system

- **AC-X01（不直接调用 Model/Rules）** GIVEN 全部命令，WHEN 用依赖扫描与 spy 检查，THEN Toolbar 对 Card & Deck Model / Rules 的直接调用为 0。
- **AC-X02（提示模型只经 snapshot）** GIVEN 触发 Hint，THEN `snapshot()` 恰一次，传给 `request_hint` 的是其 `.model` 同一引用；不缓存模型，第二次重新取快照。
- **AC-X03（不直接改状态）** GIVEN 写检测替身/只读接口，WHEN 执行全部命令，THEN Toolbar 对 State/Undo/Hint/Statistics 的内部写入为 0，只调用公开命令。
- **AC-X04（State 事件驱动）** GIVEN `GameStarted`/`GameRestarted`/`GameLoaded`，THEN 标题更新、关闭所有对话框/横幅、`Hint.reset()`；GIVEN `GameWon` 或 `StateChanged→Won`，THEN Won 对话框恰一次；GIVEN `StateChanged→Stuck`，THEN Stuck 横幅；离开 Stuck 隐藏、离开 Won 关闭；每次 `StateChanged` 刷新启用。
- **AC-X05（Undo 事件）** GIVEN `UndoCommitted`/`UndoPerformed`/`UndoCleared`，THEN 刷新；`UndoFailed` → 「撤销失败，牌局未变」；使用 `can_undo(state)` 而非 `depth()`。
- **AC-X06（Statistics 集成）** GIVEN 统计对话框，THEN 百分比 = `win_percentage_exact(快照计数器)`；无浮点重算；确认时 `reset_stats()` 恰一次、无其他写。
- **AC-X07（高亮实参）** GIVEN 有提示，THEN source = `move.cards` 源堆完整顶部索引段，destination 按 −1/0/height−1；无结果 `clear_hint()` 一次；Toolbar 只转发不算几何。
- **AC-X08（Drag 事件契约）** GIVEN Drag & Drop，THEN `DragStarted`、`DragDropped`、`DragCancelled` 均存在且被订阅；缺失则集成编译失败。
- **AC-X09（4.6 双焦点）** GIVEN 仅键盘，THEN 所有菜单项/按钮可达、焦点可见、Enter/Space 激活；GIVEN 鼠标悬停+键盘焦点共存，THEN 互不清除；鼠标转键盘焦点落在最后控件；关闭对话框焦点回牌桌；人工验证+截图。

### 性能

- **AC-P01（菜单打开）** GIVEN Release、预热、近 5 年主流桌面 x86-64，WHEN 100 次（丢前 10），THEN 输入到首帧可见 P95 ≤ 16.7ms、max ≤ 33.4ms、无 >50ms 卡顿。
- **AC-P02（对话框打开）** GIVEN 选局/统计/关于各 100 次，THEN P95 ≤ 16.7ms、max ≤ 33.4ms；`get_stats` O(1)。
- **AC-P03（命令分发）** GIVEN Playing 下 New/Restart/Undo/Hint 各 100 次，THEN 输入到命令入口 P95 ≤ 16.7ms、max ≤ 33.4ms（不含下游执行）。
- **AC-P04（启用刷新）** GIVEN 1000 次事件驱动 `refresh_commands()`，THEN P95 ≤ 1ms、max ≤ 5ms，不随深度/计数增长。
- 参考机未在 `project.yaml` 钉死前，P01–P04 在 CI 为 ADVISORY、参考机为 BLOCKING；报告记录硬件与构建配置。

### 不在本 GDD 验证范围（NOT ASSESSED）

- 原版快捷键保真度（F2/F3/F4/Ctrl+Z/F10 为暂定值，无参考机截图）。
- `project.godot` 无 `[display]` 段、stretch 未钉死；`table_scale` 可能不是 0.8，G01–G05 与 `menu_bar_fits` 实测换算在 ADR 前 `NOT ASSESSED`。
- 原版视觉保真度（art bible 不存在；本 GDD 只验收结构/尺寸/不透明/层级，不验收具体像素风格）。
- 高对比/色盲 Full Vision（Accessibility GDD 未写；MVP 仅保证键盘可达）。
- Save/Resume UI（MVP 无；`GameLoaded` 只能用注入事件验证）。
- Hint 高亮与拖拽层的相对层级归 Hint / Table Rendering GDD。
- Statistics 计数语义、Undo 原子性/深度、Drag 阈值/取消归各自 GDD。

## Open Questions

| # | 问题 | 负责人 | 目标 |
|---|------|--------|------|
| 1 | 混合参考（Windows 7/XP 功能基线）快捷键是否为 F2/F3/F4/Ctrl+Z/F10？统计是否有快捷键？ | 用户 / UX | 参考机截图后 |
| 2 | `project.godot` 无 `[display]` 段；stretch 模式未钉死，Table 的 `scale` 可能不是 0.8。需 ADR 固定 UI 坐标策略后，`menu_bar_fits` 与几何 AC 才能判定。 | technical-director / godot-specialist | 实现前（ADR） |
| 3 | （已解决 2026-10-24）系统索引 Toolbar 行已补 Drag & Drop（5）与 Table Rendering（10）。 | systems-index / producer | 已完成 |
| 4 | （已解决 2026-10-24）Table Rendering GDD 已补 `set_hint_highlight` / `clear_hint`。 | Table Rendering GDD | 已完成 |
| 5 | 菜单栏高度由主题/系统字号撑高时，固定 28 + 裁剪，还是用实测高度重算？当前：固定 28，实测 ≠28 时记录 error。 | UX / technical-artist | 实现前 |
| 6 | 退出时若牌局进行中，是否要确认？当前：不确认、立即退出（MVP 无存档）。 | 用户 / UX | MVP 验收 |
| 7 | 「新局」在 `Playing/Won/Stuck` 用 `next_deal()`（N+1）；混合参考（Windows 7/XP 功能基线）是否如此，还是随机/当前 N？ | 用户 | 参考机截图后 |
| 8 | Won 对话框「新局」打开选局对话框还是直接下一局？当前：打开选局；「下一局」直接 `next_deal()`。 | UX | MVP UI |
| 9 | 统计胜率显示已定为精确整数半进位；标签/小数位仍归 UX。 | Toolbar & Menus / UX | MVP UI |
| 10 | 字符串表/本地化的所有者与格式；MVP 仅中文默认。 | localization-lead / UX | 实现前 |
| 11 | Vertical Slice 的 Save/Resume UI（读档对话框、未保存提示）如何接入菜单。 | Save/Resume / Toolbar | VS |
| 12 | 高对比/色盲模式的菜单/对话框样式（Full Vision）。 | Readability & Accessibility | Full Vision |
| 13 | 拖拽中 Statistics/Exit/About 的 `command_enabled=1` 但菜单不可打开，是刻意不对称；测试不得据此断言 UI 可达。 | UX / QA | 实现前 |
