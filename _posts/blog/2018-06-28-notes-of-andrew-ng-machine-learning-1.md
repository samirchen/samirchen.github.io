---
layout: post
title: Andrew Ng 机器学习课程笔记
description: 机器学习入门学习。
category: blog
tag: Machine Learning
---

Week 1: Introduction


## 什么是机器学习

### 机器学习的定义

 一个程序被认为能从经验 E 中学习，解决任务 T，达到性能度量值 P，当且仅当，有了经验 E 后，经过 P 评判， 程序在处理 T 时的性能有所提升。

以下西洋棋为例，经验 e 就是程序上万次的自我练习的经验，任务 t 就是下棋，性能度量值 p 就是它在与一些新的对手比赛时赢得比赛的概率。



### 机器学习算法

最常用的两种机器学习算法：

- 监督学习(Supervised Learning)
- 无监督学习(Unsupervised Learning)

此外，还有强化学习(Reinforcement Learning)和推荐系统(Recommender Systems)。


## 监督学习算法

在监督学习中，我们已经有一个给定的数据集，并且对于给定的输入已经知道正确的输出是什么，而且也确定输入和输出之间存在关系。

监督学习问题分为「回归(Regression)」和「分类(Classification)」问题。在回归问题中，我们试图预测连续输出中的结果，这意味着我们试图将输入变量映射到某个连续函数。在分类问题中，我们试图预测离散输出中的结果。换句话说，我们试图将输入变量映射到离散类别。

比如，给定一张人的照片，我们根据给定的图片预测他们的年龄，这是一个「回归(Regression)」问题。对于一个肿瘤患者，我们来预测肿瘤是恶性的还是良性的，这是一个「分类(Classification)」问题。




[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/notes-of-andrew-ng-machine-learning-1