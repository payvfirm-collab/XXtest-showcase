# Implementation Evidence

这份页面展示 MogaiVerity **已经写进开发仓库的真实实现思路**。

它不是完整源码镜像，也不是伪代码清单；下面选取的是最能说明工程结构的实现片段，用来回答一个问题：这个项目目前到底已经做了什么，而不是“计划以后做什么”。

> 完整开发源码仍保留在私有仓库中。这里公开的是经过挑选的实现证据，不包含完整可交付源码。

---

## 1. 状态编排不是一个“跟随开关”

球内部有多个生活状态。状态层首先决定**哪些能力可以运行**，然后任务评分器才在允许的能力中选择具体行为。

当前已有状态包括：

```text
FOLLOW_TRAVEL
FOLLOW_CLOSE_LOCAL
FOLLOW_LOCAL
FOLLOW_HOME_LOCAL
INDEPENDENT_HOME
INDEPENDENT_BOOTSTRAP
```

能力矩阵是显式代码，而不是散落在十几个 Goal 里的 if：

```java
case FOLLOW_TRAVEL -> EnumSet.of(
        BallCapability.FOLLOW_PLAYER,
        BallCapability.PICKUP_ITEMS,
        BallCapability.PLACE_LIGHT,
        BallCapability.DELIVER_TO_PLAYER,
        BallCapability.CHATTER,
        BallCapability.PLAYER_LOCATION_REPLY
);

case INDEPENDENT_HOME -> EnumSet.of(
        BallCapability.WANDER_AROUND_HOME,
        BallCapability.MINE,
        BallCapability.CHOP_TREE,
        BallCapability.FARM,
        BallCapability.BUILD,
        BallCapability.TRADE,
        BallCapability.PICKUP_ITEMS,
        BallCapability.PLACE_LIGHT,
        BallCapability.STORE_ITEMS,
        BallCapability.CRAFT,
        BallCapability.RETURN_HOME,
        BallCapability.CHATTER,
        BallCapability.PLAYER_LOCATION_REPLY
);
```

这让“玩家正在赶路时别突然扩农田”“在家附近才允许长期建设”这样的约束有统一入口，不需要每加一个功能就重写其他模块。

---

## 2. Task Lease：选了就先认真做，不每 tick 变心

如果 NPC 每 tick 都重新挑最高分任务，很容易出现来回横跳。因此即时任务通过 `TaskLease` 保持最小承诺时间，同时允许真正的紧急任务强制抢占。

```java
boolean forcedInterrupt = best.type().isForcedPriority()
        && best.type().canInterrupt(current.type());
boolean currentCommitmentDone = current.commitmentSatisfied(gameTick);
boolean sameTask = current.type() == best.type()
        && current.key().equals(best.key());

if (sameTask) {
    return current;
}

boolean currentIsSoftIdle = current.type() == BallTaskType.IDLE
        || current.type() == BallTaskType.WANDER;

if (forcedInterrupt || currentCommitmentDone || currentIsSoftIdle) {
    return startLease(best, gameTick);
}

return current;
```

正常任务不会因为另一个候选只高了零点几分就立刻改主意；但玩家坠落救援等硬优先级任务仍可以打断当前工作。

---

## 3. 卡死恢复：失败目标会被临时封锁

自主 NPC 最容易出现的体验问题之一不是“不会选任务”，而是**选中了一个永远无法完成的目标以后无限复读**。

当前决策器会记录无进展时间和连续失败次数；达到恢复条件后，释放当前租约，并把同一个具体目标暂时放进 retry cooldown。

```java
boolean currentNeedsRecovery = current != null
        && current.type() != BallTaskType.IDLE
        && current.type() != BallTaskType.WANDER
        && recoveryPolicy.stalled(
                gameTick,
                lastProgressTick,
                consecutiveFailures
        );

if (currentNeedsRecovery) {
    block(current.key(), gameTick);
    current = null;
    consecutiveFailures = 0;
}
```

然后下一轮评分会跳过仍在 cooldown 的普通目标，而强制优先事件不受这个普通冷却限制。

这部分解决的是典型的：

```text
目标 A 寻路失败
→ 下一 tick 又选 A
→ 再次失败
→ 无限原地抽搐
```

---

## 4. 长期目标和即时任务是两层脑

项目没有把“我要获得钻石”直接当成一个动作。

长期目标层负责维持发展方向；即时任务层负责完成当前阶段。比如“获得钻石”过程中，制作工作台、木镐、石镐、铁镐都只是中间步骤，不会每完成一步就重新抽一个人生目标。

目前长期目标层已经有独立的：

- `ProgressGoalCandidate`
- `ProgressGoalScorer`
- `ProgressGoalLease`
- `ProgressGoalDecisionEngine`
- `GoalRecoveryPolicy`

长期评分会综合需求、解锁价值、随身库存覆盖、自有仓储覆盖、环境机会、工作量、风险、预计步骤和距离。

因此它能区分“眼前便宜的小事”和“虽然麻烦但会解锁后续能力的关键升级”。

---

## 5. 资源采集锁的是“缺口批次”，不是一棵树

一棵树、一块铁矿只代表 progress，不代表整个采集任务完成。

```java
public static double collectionNeed(ResourceDemand demand) {
    if (demand.satisfied()) return 0.0;
    return 0.35 + demand.shortageRatio() * 0.65;
}

public static boolean shouldContinueGathering(ResourceDemand demand) {
    return !demand.satisfied();
}

public static String batchKey(String goalKey, ResourceDemand demand) {
    return "gather:" + goalKey + ":" + demand.resourceKey();
}
```

例如当前建造目标还缺 40 木头，砍完第一棵树以后会重新扫描库存；只要真实缺口仍然大于 0，就继续当前资源批次。

目标是避免这种行为：

```text
我要盖房
→ 砍一棵树
→ “木头任务完成”
→ 去做别的了
```

---

## 6. Forge 跟随行为已经是真实 Goal，不是文档

基础 Follow 已经接到真实 Minecraft `Goal` 与 `PathNavigation`。

```java
@Override
public boolean canUse() {
    owner = resolveOwner();
    return owner != null
            && owner.isAlive()
            && ball.distanceToSqr(owner) > START_DISTANCE_SQR;
}

@Override
public void tick() {
    if (owner == null) return;

    ball.getLookControl().setLookAt(owner, 30.0F, 30.0F);
    if (--repathCooldown <= 0) {
        repathCooldown = REPATH_INTERVAL_TICKS;
        ball.getNavigation().moveTo(owner, speed);
    }
}
```

当前基础策略是：

- 超过约 4 格开始追；
- 进入约 2 格停止；
- 每 5 tick 重算路径；
- 使用原版 `PathNavigation`；
- 普通跟随不自动 TP。

更复杂的 `FOLLOW_LOCAL` / 附近微任务逻辑建立在这层真实行为之上。

---

## 7. 农田不是一个“找成熟作物”的 Goal

农田目前已经拆成三个独立决策问题：

1. **FarmSiteScorer**：在哪里建田；
2. **IrrigationPlanScorer**：用现成水、搬水还是建本地无限水；
3. **WaterPlacementScorer**：具体水源应该放在哪一格。

评分会考虑靠家程度、现有农田、地形平整、清场成本、天然水、spill risk、downhill escape risk、protected area conflict 等条件。

也就是说，后续真正接执行器时，执行层负责“锄这一格 / 放这一桶水”，而不是把“为什么要在这里建田”重新硬编码一遍。

---

## 8. 当前真正的工程边界

现阶段不是所有功能都已经拥有 Minecraft 世界执行器。

已经比较完整的是：

- Forge 球实体基础；
- owner / NBT / 孵化 / 普通 Follow；
- 自主任务评分；
- Task Lease / Goal Lease；
- 失败恢复和 retry cooldown；
- 长期发展目标；
- 重复行为衰减；
- 资源缺口驱动；
- 三档玩家模式 / 六种内部生活状态；
- Capability / Action Category 权限层；
- Home / Follow / Farm / Mining / Tree / Trade / Rescue 等决策策略。

仍需要继续接入的是统一的：

```text
World Observation
→ Candidate Generation
→ Decision / Lease
→ Action Executor
→ World State Change
→ Re-scan
```

也就是把已经存在的“脑子”彻底接到 Minecraft 世界里的“眼睛和手”。

---

## 9. 为什么这样拆

如果把所有东西都写成互相独立的 Minecraft Goal，项目很快会遇到：

- 多个 Goal 抢控制权；
- 任务每 tick 横跳；
- 一个坏目标无限重试；
- 跟随状态突然开始长期建造；
- 资源只收一份就误判“任务完成”；
- 后续新增功能要同时修改十几个旧 Goal。

当前结构把这些问题分别交给：

```text
Life State
→ Capability / Action Category
→ Candidate
→ Scorer
→ Task / Goal Lease
→ Recovery
→ Executor
```

目标不是把行为数量堆大，而是让行为之间真正能够共存。

---

# 最后放算式：这块确实最费力

前面的状态、Lease、恢复、资源缺口都不是装饰，它们最后会汇到“当前到底该做什么”的评分上。

自主层会把候选任务统一转成评分问题。普通任务考虑当前需求、环境机会、准备程度、距离和重复行为惩罚；超过硬工作距离的普通任务直接失去候选资格。

```java
public double score(TaskCandidate c) {
    if (c.distanceBlocks() > hardWorkRange && !c.type().isForcedPriority()) {
        return Double.NEGATIVE_INFINITY;
    }

    double distance01 = Math.min(1.0, c.distanceBlocks() / normalWorkRange);
    if (c.distanceBlocks() > normalWorkRange) {
        distance01 += (c.distanceBlocks() - normalWorkRange)
                / Math.max(1.0, hardWorkRange - normalWorkRange);
    }

    double effectiveRepetitionPenalty =
            c.repetitionPenalty() * (1.0 - c.need() * 0.90);

    return c.need() * weights.need()
            + c.opportunity() * weights.opportunity()
            + c.readiness() * weights.readiness()
            - distance01 * weights.distancePenalty()
            - effectiveRepetitionPenalty * weights.repetitionPenalty();
}
```

当前默认权重大致是：

```text
need              = 4.0
opportunity       = 3.0
readiness         = 2.0
distance penalty  = 2.2
repetition penalty= 3.0
```

其中有一个刻意设计：**需求越高，重复惩罚越弱。**

这意味着“最近已经砍过树”只会在资源不紧缺时鼓励它换换事情做；如果当前建筑真的还缺很多木头，它不会为了追求表面上的“行为多样性”突然停工。

换句话说，这个算式真正做的不是“随机挑一个高分动作”，而是在需求、机会、准备度、距离和行为多样性之间持续找平衡。

> 算式太费力了，放最后看。😶
