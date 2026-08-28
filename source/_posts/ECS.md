---
title: ECS 简介
date: 2022-07-02 20:28:16
categories: ECS
tags:
	- ecs
---

ECS全称为Entity Component System 与传统OOP开发模式差异较大。

## OOP

### 开发方式

 	在游戏后端开发场景中，如技能的实现；
```C++
class SkillBase {
 public:
  virtual int Release() = 0;
  virtual int SelectTarget() = 0;
  ...
};

class SarcasmSkill : public SkillBase {
 public:
  int Release() override;
  int SelectTarget() override;
};

class RecoveSkill : public SkillBase {
 public:
  int Release() override;
  int SelectTarget() override;	
}

```

  随着需求增加，技能类会不断膨胀，可读性会降低，难以维护。

### 组件
	
	OOP中的Component，是对复杂切独立的功能的抽象，如SLG游戏中大地图上建筑有多种能力，如调动、守军、攻打玩家等，
	把每种能力抽象成组件，建筑有徐建就表示拥有这项能力，本质还是OOP的编程思想。


## ECS

### ECS含义
	在ECS架构下，Component不同于OOP，只包含数据成员，Entity由多个Component组成，
	System负责实现业务逻辑。



