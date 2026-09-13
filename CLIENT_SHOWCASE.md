# MogaiVerity — Engineering Showcase

> Minecraft Java Edition 1.20.1 Forge · Autonomous NPC Architecture

MogaiVerity 不是一个“多写几个 Goal 就能完成”的跟宠 Mod。

它的核心目标是让一个球形 NPC 在 Minecraft 世界里表现出**连续、稳定、有资源约束的自主生活行为**：知道自己当前处于什么生活状态、有哪些事情允许做、哪些事情值得先做、当前目标应该坚持多久、什么时候必须被紧急事件打断、卡住以后如何恢复，以及一个长期目标该如何被拆成连续的小步骤。

当前开发重点已经从“继续发明规则”进入到：

```text
World Observation
→ Candidate Generation
→ Decision / Lease
→ Action Executor
→ World State Change
→ Re-scan
```

也就是把已经成型的决策系统继续接到 Minecraft 世界观察与动作执行层。

---

## 已经形成的核心工程层

| 工程层 | 当前实现 | 解决的问题 |
|---|---|---|
| Forge Entity Foundation | 球实体、owner、NBT、孵化、耐火、原版 PathNavigation、基础 FollowGoal | 让 NPC 真正存在于世界里，而不是文档概念 |
| Life State Orchestration | 三档玩家模式、六种内部生活状态、Capability 权限矩阵、Action Category 权限 | 防止不同功能同时争抢控制权 |
| Instant Task Decision | Candidate + Scorer + Task Lease + Forced Interrupt | 决定“现在做什么”，同时避免每 tick 横跳 |
| Long-term Progression | ProgressGoalScorer + Goal Lease + Recovery | 让“获得钻石”等长期目标不会做一步就换人生 |
| Failure Recovery | 无进展检测、连续失败统计、retry cooldown | 避免坏目标无限重试和原地抽搐 |
| Repetition Memory | 指数衰减的重复行为记忆 | 避免 NPC 长时间机械重复同一件事 |
| Resource Demand | shortage 驱动的资源批次 | 一棵树只算 progress，不会误判整个资源目标完成 |
| Home System | HomeAnchor、生活圈、回家策略、安全屋与家内行为规则 | 给长期建设与日常生活一个稳定空间中心 |
| Follow Activity | 真实 FollowGoal、局部停留判断、附近微任务、召回与送货策略 | 区分“玩家在赶路”和“玩家在这里活动” |
| Domain Scorers | Farm / Mining / Tree / Building / Inventory / Trade / Rescue 等评分器 | 各功能只负责自己的专业决策，不互相硬编码 |

---

## 为什么不是“给宠物加几十个 Goal”

普通的功能堆叠很容易变成：

```text
砍树 Goal
挖矿 Goal
种田 Goal
跟随 Goal
交易 Goal
建造 Goal
……
↓
每个 Goal 都觉得自己现在该运行
↓
互相抢方向盘
```

MogaiVerity 在功能 Goal 之上增加了一层统一编排：

```text
当前生活状态
↓
允许的 Capability / Action Category
↓
生成当前可行 Candidate
↓
奖励 / 惩罚评分
↓
Task / Goal Lease
↓
执行
↓
进度 / 失败反馈
↓
继续、抢占、恢复或重新决策
```

因此新增一个功能时，不需要把其他十几个功能重新改一遍。

---

## 真实实现：Task Lease

即时决策不是“每一 tick 重新抽最高分”。

当前任务会被 Lease 保持最小承诺阶段；普通任务只有在当前承诺完成后才允许自然切换，而真正的紧急事件拥有明确的强制抢占入口。

```java
boolean forcedInterrupt = best.type().isForcedPriority()
        && best.type().canInterrupt(current.type());
boolean currentCommitmentDone = current.commitmentSatisfied(gameTick);

if (forcedInterrupt || currentCommitmentDone || currentIsSoftIdle) {
    return startLease(best, gameTick);
}

return current;
```

这解决的是自主 NPC 中非常常见的体验问题：

```text
想砍树
→ 下一 tick 挖矿分高了 0.2
→ 转身去挖矿
→ 再下一 tick 又想砍树
→ 来回横跳
```

---

## 真实实现：卡死恢复与 retry cooldown

任务被选中以后并不意味着永远坚持。

系统会记录：

- 最后一次真实进展时间；
- 连续失败次数；
- 当前具体目标 key；
- 被放弃目标的临时冷却。

达到恢复条件后，当前租约会被释放，同一个失败目标暂时进入 retry cooldown。

```java
if (currentNeedsRecovery) {
    block(current.key(), gameTick);
    current = null;
    consecutiveFailures = 0;
}
```

这样不会出现：

```text
路径失败
→ 下一 tick 再选同一块矿
→ 路径失败
→ 再选同一块矿
→ 无限复读
```

同时，紧急救援等强制优先事件不会被普通 retry cooldown 阻塞。

---

## 真实实现：两层目标系统

项目将“长期发展方向”和“下一步动作”分开。

例如：

```text
长期目标：获得钻石

木头
→ 工作台
→ 木镐
→ 石头
→ 石镐
→ 铁
→ 铁镐
→ 钻石
```

制作出铁镐只是当前长期目标的一次 progress，不会让 NPC 立刻重新抽一个完全无关的人生目标。

长期目标拥有自己的：

- Candidate
- Scorer
- Lease
- Recovery Policy

这使发展行为可以跨越多个前置步骤保持连续性。

---

## 真实实现：资源缺口而不是单次采集

资源任务锁定的是一个“需求批次”，不是一棵树或一块矿。

例如建造目标还缺 40 木头：

```text
砍一棵树
→ 重新扫描库存
→ shortage 仍然 > 0
→ 继续当前资源批次
```

只有真实资源缺口归零后，这个采集批次才 complete。

这让自主行为不会出现：

```text
“我要盖房子”
→ 砍一棵树
→ “木头任务完成”
→ 跑去干别的
```

---

## 真实实现：六种内部生活状态

玩家模式和内部生活状态不是同一层。

当前已有：

```text
FOLLOW_TRAVEL
FOLLOW_CLOSE_LOCAL
FOLLOW_LOCAL
FOLLOW_HOME_LOCAL
INDEPENDENT_HOME
INDEPENDENT_BOOTSTRAP
```

例如：

**FOLLOW_TRAVEL** 允许跟随、拾取、补灯、送货、聊天等轻量行为，不允许突然开始长期建设。

**FOLLOW_HOME_LOCAL** 在玩家长期停留于家附近时开放 Farm / Build / Store / Craft 等家园行为。

**INDEPENDENT_HOME** 则允许以 HomeAnchor 为中心运行完整自主生活模块。

状态层只回答：

> 这件事现在允不允许做？

真正回答：

> 允许做的这些事情里，现在最值得做哪个？

仍然由评分器负责。

---

## 已有真实 Forge 行为

基础 Follow 已经不是设计稿，而是真实 Minecraft `Goal`：

- owner 超过约 4 格开始追；
- 进入约 2 格停止；
- 每 5 tick 重新寻路；
- 直接使用 Minecraft 原版 `PathNavigation`；
- 普通跟随不依赖自动 TP。

球实体本身也已经具备 owner / NBT 持久化、孵化、耐火、基础导航等真实 Forge 层实现。

---

## Farm 为什么单独拆三层决策

农田并不是一个“看见成熟作物就收”的单 Goal。

目前已经分成：

| 决策器 | 负责的问题 |
|---|---|
| `FarmSiteScorer` | 农田应该建在哪里 |
| `IrrigationPlanScorer` | 用现成水、搬水，还是建本地无限水 |
| `WaterPlacementScorer` | 水源具体应该放在哪一格 |

评分会考虑 Home 距离、已有农田、地形平整度、清场成本、天然水、spill risk、downhill escape risk、protected area conflict 等条件。

执行器以后只需要负责“实际锄地 / 放水 / 播种”，不用重新决定“为什么这里值得建田”。

---

## Rescue 的难点不是一个 TP

救援系统的判断需要区分：

- 普通小摔落，不抢占；
- 预计高伤害坠落，允许抢占；
- 轨迹进入岩浆，即使低空也可能救；
- 玩家潜行主动进入岩浆，视为明确意图，不擅自救；
- 同一次事故只触发一次，避免连续重复救援。

因此 Rescue 的重点不是“会不会传送过去”，而是**什么时候应该认为这真的构成一次需要抢占当前工作的事故**。

---

## 当前开发边界

为了避免把“已经设计”写成“已经完成”，当前边界明确区分为两部分。

### 已经形成主体的部分

- Forge 球实体基础；
- owner / NBT / 孵化 / Follow；
- Life State / Capability / Action Category；
- 即时任务评分；
- Task Lease；
- 长期 Progress Goal + Goal Lease；
- 卡死恢复与 retry cooldown；
- 重复行为衰减；
- 资源 shortage 批次；
- Home / Follow / Farm / Mining / Tree / Trade / Rescue 等领域决策层。

### 仍在继续接入的部分

- 统一 World Observation；
- Minecraft 方块 / 容器 / 村民 / 流体 → Candidate 的扫描层；
- 真正挖掘、放置、种植、开箱、交易、建造等 Action Executor；
- 动态 Chunk Ticket Manager；
- 最终 Renderer / 正式视觉动画；
- 配置 UI 与完整 Guidebook。

这也是当前下一阶段的主要工作，而不是继续无限追加纸面功能。

---

## 实现证据

更完整的经过筛选的实现片段，包括状态权限、Lease、恢复、资源批次、真实 FollowGoal，以及最后的自主评分算式：

**[查看 Implementation Evidence](IMPLEMENTATION.md)**

完整开发源码不在公开展示仓库中。
