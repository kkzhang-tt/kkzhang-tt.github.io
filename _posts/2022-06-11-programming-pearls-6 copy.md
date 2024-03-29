---

layout: post
title: "编程珠玑: 程序性能分析"
author: "kkzhang"
category: algrithom

---

## 设计层面

为了提高系统性能，可以在多个层面进行设计。

### 问题定义

良好的问题定义可以避免对问题需求的过高估计。问题定义与程序效率之间具有复杂的相互影响。

### 系统结构

将大型系统分解成多个模块

### 算法和数据结构

对于模块性能提升的关键是表示数据的结构和操作这些数据的算法

### 代码调优

与系统无关的代码调优

### 系统软件

改变系统所基于的软件；同时考虑与系统相关的代码调优

### 硬件

更快的硬件可以提高系统的性能

## 原理

当性能问题无法回避时，对于不同的性能目标可以从不同的设计层面进行考虑。

1. ***如果仅需要较小的性能提升，就对效果最佳的层面进行改进***。
    
    首先考虑所有可能的设计层面，然后选择“性价比”最高的那个：较小投入可以获得较大的加速系数。
    
2. ***如果需要较大的性能提升，需要对多个层面进行改进***。
    
    需要从各个不同方向对问题进行深入研究。

