# Systems Index: 空当接龙：经典复刻

> **Status**: Draft
> **Created**: 2026-10-24
> **Last Updated**: 2026-10-24
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

本作是 Windows 经典《空当接龙》的忠实复刻，机械范围由四条支柱约束：原版规则零改动、手感优先于画面、一秒读懂牌面、失败可无损重来。系统共 15 个、6 个类别；10 个进入 MVP，3 个进入 Vertical Slice，2 个进入 Full Vision。没有叙事、经济、成长或联网系统。

**1. Card & Deck Model** — 定义 52 张牌、8 列牌堆、4 个自由单元与 4 个回收单元的数据结构与基本操作；是所有规则与界面的共同基础。

**2. Deal Generator** — 把局号转换为确定性的初始牌面；与混合参考（Windows 7/XP 1–1,000,000）局号的洗牌一致性是最大的技术不确定点。

**3. Rules & Move Validator** — 判定拖拽或双击是否合法、序列是否可整体移动，并识别胜利与卡死。

**4. Game State Machine (inferred)** — 管理发牌、进行中、胜利、卡死四个状态，以及新局、重开本局的转换。

**5. Drag & Drop Interaction** — 把鼠标输入转成选择、拖拽、落位与双击自动归位；手感的核心所在。

**6. Undo History** — 记录每一步可逆操作，按原版行为提供撤销。

**7. Hint System** — 在玩家请求时给出一个合法且有用的下一步；提示质量直接影响“是否像原作”。

**8. Statistics & Records** — 记录局数、胜率与连胜；MVP 只在会话内保存，Vertical Slice 再落盘。

**9. Save/Resume Game (inferred)** — 退出或重开后恢复进行中的牌局；原版行为，但不影响核心循环验证。

**10. Table Rendering & Layout** — 绘制牌桌、卡牌与层级，处理窗口缩放与 1280x720 下的可读性。

**11. Toolbar & Menus (inferred)** — 暴露新局、重开本局、选局号、撤销与提示；是辅助功能的入口。

**12. Animation & Feedback (inferred)** — 发牌、移动与落位的短促动效，单次不超过 150ms；服务手感而非装饰。

**13. Settings (inferred)** — 音效开关等最小设置；不改变规则与布局。

**14. Audio & SFX (inferred)** — 原版风格的卡牌音效；可选，缺失不影响可玩性。

**15. Readability & Accessibility (inferred)** — 高对比、键盘操作与缩放可读性；在 Full Vision 阶段补齐。

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | Card & Deck Model | Core | MVP | Designed | design/gdd/card-deck-model.md | — |
| 2 | Deal Generator | Core | MVP | Designed | design/gdd/deal-generator.md | 1 |
| 3 | Rules & Move Validator | Core | MVP | Designed | design/gdd/rules-move-validator.md | 1 |
| 4 | Game State Machine (inferred) | Core | MVP | Designed | design/gdd/game-state-machine.md | 2, 3 |
| 5 | Drag & Drop Interaction | Gameplay | MVP | Designed | design/gdd/drag-drop-interaction.md | 1, 3, 4, 6, 10 |
| 6 | Undo History | Gameplay | MVP | Designed | design/gdd/undo-history.md | 1, 3, 4 |
| 7 | Hint System | Gameplay | MVP | Designed | design/gdd/hint-system.md | 1, 3 |
| 8 | Statistics & Records | Persistence | MVP | Designed | design/gdd/statistics-records.md | 4 |
| 9 | Save/Resume Game (inferred) | Persistence | Vertical Slice | Not Started | — | 1, 4 |
| 10 | Table Rendering & Layout | UI | MVP | Designed | design/gdd/table-rendering-layout.md | 1 |
| 11 | Toolbar & Menus (inferred) | UI | MVP | Designed | design/gdd/toolbar-menus.md | 4, 5, 6, 7, 8, 10 |
| 12 | Animation & Feedback (inferred) | UI | Vertical Slice | Not Started | — | 5, 10 |
| 13 | Settings (inferred) | UI | Full Vision | Not Started | — | 10, 14 |
| 14 | Audio & SFX (inferred) | Audio | Vertical Slice | Not Started | — | 4, 5 |
| 15 | Readability & Accessibility (inferred) | Meta | Full Vision | Not Started | — | 5, 10, 11 |

---

## Categories

| Category | Description | Systems |
|----------|-------------|---------|
| **Core** | 牌局的基础数据、规则与状态 | Card & Deck Model, Deal Generator, Rules & Move Validator, Game State Machine |
| **Gameplay** | 玩家实际操作的玩法系统 | Drag & Drop Interaction, Undo History, Hint System |
| **Persistence** | 记录与跨会话连续性 | Statistics & Records, Save/Resume Game |
| **UI** | 玩家可见的界面与反馈 | Table Rendering & Layout, Toolbar & Menus, Animation & Feedback, Settings |
| **Audio** | 声音与音效 | Audio & SFX |
| **Meta** | 核心循环之外的系统 | Readability & Accessibility |

---

## Priority Tiers

| Tier | Definition | Target Milestone | Design Urgency |
|------|------------|------------------|----------------|
| **MVP** | Required for the core loop to function. Without these, you can't test "is this fun?" | First playable prototype | Design FIRST |
| **Vertical Slice** | Required for one complete, polished area. Demonstrates the full experience. | Vertical slice / demo | Design SECOND |
| **Alpha** | All features present in rough form. Complete mechanical scope, placeholder content OK. | Alpha milestone | Design THIRD |
| **Full Vision** | Polish, edge cases, nice-to-haves, and content-complete features. | Beta / Release | Design as needed |

---

## Dependency Map

### Foundation Layer (no dependencies)

1. **Card & Deck Model** — 所有牌局数据与操作的唯一来源。

### Core Layer (depends on foundation)

1. **Deal Generator** — depends on: Card & Deck Model
2. **Rules & Move Validator** — depends on: Card & Deck Model
3. **Table Rendering & Layout** — depends on: Card & Deck Model

### Feature Layer (depends on core)

1. **Game State Machine** — depends on: Deal Generator, Rules & Move Validator
2. **Drag & Drop Interaction** — depends on: Card & Deck Model, Rules & Move Validator, Undo History, Table Rendering & Layout
3. **Undo History** — depends on: Card & Deck Model, Rules & Move Validator, Game State Machine
4. **Hint System** — depends on: Card & Deck Model, Rules & Move Validator
5. **Statistics & Records** — depends on: Game State Machine
6. **Save/Resume Game** — depends on: Card & Deck Model, Game State Machine

### Presentation Layer (depends on features)

1. **Toolbar & Menus** — depends on: Game State Machine, Drag & Drop Interaction, Undo History, Hint System, Statistics & Records, Table Rendering & Layout
2. **Animation & Feedback** — depends on: Drag & Drop Interaction, Table Rendering & Layout
3. **Audio & SFX** — depends on: Game State Machine, Drag & Drop Interaction

### Polish Layer (depends on everything)

1. **Settings** — depends on: Table Rendering & Layout, Audio & SFX
2. **Readability & Accessibility** — depends on: Drag & Drop Interaction, Table Rendering & Layout, Toolbar & Menus

**Bottleneck systems**: Card & Deck Model (1), Rules & Move Validator (3), Game State Machine (4), Table Rendering & Layout (10).
**Leaf systems**: Save/Resume Game (9), Animation & Feedback (12), Settings (13), Readability & Accessibility (15).

---

## Recommended Design Order

| Order | System | Priority | Layer | Agent(s) | Est. Effort |
|-------|--------|----------|-------|----------|-------------|
| 1 | Card & Deck Model | MVP | Foundation | game-designer, systems-designer | S |
| 2 | Deal Generator | MVP | Core | systems-designer | M |
| 3 | Rules & Move Validator | MVP | Core | game-designer, systems-designer | M |
| 4 | Table Rendering & Layout | MVP | Core | ux-designer, technical-artist | M |
| 5 | Game State Machine | MVP | Feature | game-designer | S |
| 6 | Drag & Drop Interaction | MVP | Feature | ux-designer, ui-programmer | M |
| 7 | Undo History | MVP | Feature | game-designer | S |
| 8 | Hint System | MVP | Feature | systems-designer | M |
| 9 | Statistics & Records | MVP | Feature | game-designer | S |
| 10 | Toolbar & Menus | MVP | Presentation | ux-designer | S |
| 11 | Save/Resume Game | Vertical Slice | Feature | game-designer | S |
| 12 | Animation & Feedback | Vertical Slice | Presentation | technical-artist, ux-designer | M |
| 13 | Audio & SFX | Vertical Slice | Presentation | sound-designer | S |
| 14 | Settings | Full Vision | Polish | ux-designer | S |
| 15 | Readability & Accessibility | Full Vision | Polish | accessibility-specialist, ux-designer | S |

Effort: S = 1 session, M = 2–3 sessions, L = 4+ sessions.

---

## Circular Dependencies

- None found.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| Deal Generator | Technical | 局号洗牌与混合参考（Windows 7/XP 1–1,000,000）一致需要移植并逐局校验算法 | 在首个可运行版本中对照混合参考局号；不可行则使用自有确定性算法并在文档中注明差异 |
| Drag & Drop Interaction | Design / Technical | 拖拽、落位与双击手感不达标会被直接与原作比较 | 用一次性原型验证手感，再写 GDD |
| Hint System | Design | 提示若不够好用，会被认为不如原作 | 先定义可接受的提示策略，用若干牌局测试 |
| Table Rendering & Layout | Technical | 窗口缩放与卡牌层级影响可读性 | 先在 1280x720 与缩放窗口下验证布局 |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 15 |
| Design docs started | 10 |
| Design docs reviewed | 0 |
| Design docs approved | 0 |
| MVP systems designed | 10/10 |
| Vertical Slice systems designed | 0/3 |

---

## Next Steps

- [ ] Review and approve this systems enumeration
- [ ] Design MVP-tier systems first (use `/design-system [system-name]`)
- [ ] Run `/design-review` on each completed GDD
- [ ] Run `/gate-check technical-setup` when MVP systems are designed
- [ ] Validate the highest-risk systems with `/vertical-slice` before committing to Production
