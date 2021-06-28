---
title: 重构
date: 2020-12-12
tag: 
- 重构
categories:
- 开发
---
之前看了重构的书，最近在公司内部自学网站学习重构的课程，简单记录一些核心要点。
<!--more-->

# 重构

## 代码坏味道

1. 冗余和重复： 重复代码、过多注释、夸夸其谈未来性（过度设计）
2. 局部膨胀： 过长的参数列表
3. 耦合不良： 发散式变化（某一模块因多种原因被修改），霰弹式修改（一个功能修改多处代码）
4. 结构不良： switch多处现身、新增状态码新增分支（违反开闭原则）、分支做多件事（违反单一职责）、临时变量（值域仅用于部分方法间传参）

## 重构手法

**十六字真言**：

- 旧的不变
- 新的创建
- 一步切换
- 旧的再见

小步修改，及时反馈。所谓反馈就是测试。

### 简化语句

1. 合并表达式。确认表达式没有副作用，抽函数表含义；顺序执行可用“或”，嵌套if可用“与”来合并
2. 移动语句。
3. 卫语句代替嵌套，提前返回。一般针对非核心逻辑

### 重组函数

1. 查询和修改函数分离。无副作用的函数需要更少的操心。
2. 已明确参数取代参数。如果参数决定不同执行逻辑，直接代为调用不同的函数。
3. 提炼函数。提取功能，**提升复用性**；函数名描述用途，提升可读性；
4. 内联函数。消除多余的间接性，方便后续重构。

### 重组数据

1. 拆分变量： 变量多次赋值，且赋值的意义不同。
2. 提炼类： 单一职责
3. 内联类： 消除多余间接性，以便按照其他方向进一步拆分。
4. 类继承系列： 函数上移/下移，字段上移/下移，移除子类/子类代替类型码，构造函数本体上移

## 重构遗留系统

1. 分析待拆分模块和db，“同一类型数据只有一处修改”
2. 删除跨模块写库操作，将jion等类型上移到应用层，注意性能下降
3. 待拆分模块提供API给其他模块使用
4. 拷贝新增模块，读老库，原系统中原模块不动
5. 其他模块调用新模块，架空原模块。（方便切回）
6. db数据迁移，原schema不删
7. 调用新schema
8. 删除原schema
