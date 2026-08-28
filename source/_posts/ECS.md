---
title: ECS 简介
date: 2022-07-02 20:28:16
categories: ECS
tags:
	- ecs
	- cpp
---

ECS 全称 Entity Component System。它和传统 OOP 的继承树不同：对象不再靠“我是什么”来获得能力，而是靠“我带了哪些数据”，逻辑则集中在 System 里跑。

游戏后端里技能、Buff、建筑、单位经常会组出大量组合。继承很快写不动，ECS 就是为这种组合爆炸准备的。

## OOP 的困境

### 用继承实现技能

技能常见写法是基类加一堆派生类：

```cpp
class SkillBase {
 public:
  virtual int Release() = 0;
  virtual int SelectTarget() = 0;
};

class SarcasmSkill : public SkillBase {
 public:
  int Release() override;
  int SelectTarget() override;
};

class RecoverSkill : public SkillBase {
 public:
  int Release() override;
  int SelectTarget() override;
};
```

嘲讽、治疗还能看懂。后面再加上“嘲讽且治疗”“治疗且位移且禁锢”，要么继续派生，要么在基类里堆开关。类会不断膨胀，复用靠复制粘贴，改一处逻辑要翻好几个子类。

继承表达的是 **is-a**。技能更像 **has-a**：有没有选敌、有没有伤害、有没有位移。用继承硬套，层次会越来越深。

### OOP 里的组件

OOP 也可以把独立能力抽成组件。例如 SLG 大地图上的建筑，可能具备调动、守军、攻打玩家等能力：每种能力做成一个组件，建筑挂上组件就表示拥有这项能力。

这已经是组合，不是继承。但组件对象里通常仍带着方法和状态，本质还是 OOP：调用时走的是 `building->garrison->Do()` 这种对象方法。它能缓解继承膨胀，数据还是和逻辑长在一起，遍历所有“有血量的单位”时，缓存也不友好。

## ECS 是什么

ECS 把一件事拆成三块，职责分得很死：

| 概念 | 是什么 | 不该放什么 |
| --- | --- | --- |
| Entity | 一个 ID，用来把组件拼成“一个东西” | 业务逻辑、成员函数 |
| Component | 纯数据，只描述状态 | 释放技能、寻路、伤害结算 |
| System | 按组件集合跑逻辑 | 长期持有某个 Entity 的私有流程 |

一句话：Entity 是编号，Component 是数据，System 是函数。

### Entity

Entity 不是类实例，通常就是 `uint32_t` / `uint64_t`。创建实体等于发一个 ID，销毁等于回收 ID 并摘掉组件。它本身没有 `Release()` 这种方法。

### Component

和 OOP 组件相反，这里的 Component **只含数据成员**：

```cpp
struct Position {
  float x{0};
  float y{0};
};

struct Health {
  int hp{100};
  int max_hp{100};
};

struct SkillTag {
  int skill_id{0};
  int cooldown_ms{0};
};
```

有 `Health` 就能掉血，有 `SkillTag` 就能进技能系统，没有对应组件就不会被那个系统扫到。组合发生在数据层，不必再派生 `RecoverSarcasmBuilding`。

### System

System 负责业务。它查询“同时具备某些组件的实体”，成批处理：

```cpp
void HealthSystem(World& world) {
  world.View<Health>().Each([](Entity e, Health& hp) {
    if (hp.hp <= 0) {
      world.Destroy(e);
    }
  });
}

void SkillSystem(World& world, int dt_ms) {
  world.View<SkillTag, Position>().Each([&](Entity e, SkillTag& skill, Position& pos) {
    if (skill.cooldown_ms > 0) {
      skill.cooldown_ms -= dt_ms;
      return;
    }
    // 按 skill_id 结算，读写的是组件数据，而不是某个技能子类
    ReleaseSkill(world, e, skill, pos);
  });
}
```

技能差异变成数据（`skill_id`、冷却、范围），而不是类层次。新技能优先加配置和数据，而不是加一个派生类。

## 和 OOP 的差别

同一套“有血量、能放技能的单位”：

```text
OOP:
  Unit : Actor
    + hp
    + virtual ReleaseSkill()
    Warrior : Unit
    Mage : Unit

ECS:
  Entity 42
    Position { x, y }
    Health   { hp, max_hp }
    SkillTag { skill_id, cooldown_ms }
  HealthSystem / SkillSystem / MoveSystem
```

OOP 问的是“这个对象是谁，它会什么”；ECS 问的是“哪些 ID 带了这几块数据，用同一段逻辑扫一遍”。

因此：

- 组合是默认能力，不用提前设计很深的继承树。
- 同类数据可以连续存放，适合每帧扫大量单位。
- 逻辑集中，调伤害公式通常只改 System，不必打开二十个技能子类。

## 什么时候用

适合：单位数量大、能力组合多、每帧/每 tick 都要批量更新的系统，例如移动、AOI、Buff、伤害、冷却。

不适合硬套的：强流程、强交互、生命周期不规则的模块，例如账号登录、一次性 GM 指令、复杂 UI 状态机。这些继续用普通对象模型往往更直接。

后端里常见做法是局部采用：战斗、场景对象走 ECS，网关、存储、活动配置仍走常规 OOP / 数据表。不必把整个进程都改成 Entity。

## 小结

OOP 组件解决的是“别再用继承堆能力”，但数据和逻辑还在对象上。ECS 再走一步：组件只留数据，System 统一算逻辑，Entity 只是把它们串起来的 ID。

技能类膨胀、建筑能力排列组合，这类问题用 ECS 会顺很多。先把数据和逻辑拆开，再考虑用 EnTT、Flecs 还是自研 World，顺序不要反。
