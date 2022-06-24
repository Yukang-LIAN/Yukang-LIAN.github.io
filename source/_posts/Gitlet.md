---
title: Gitlet
date: 2022-06-22 10:18:53
tags:
---
## 前言

最近在学习UCB的CS61b课程，我学习的版本是2018Spring，用了大约一个月完成了所有Lab、Homework和Project，之前就听群友说61b的gitlet项目非常值得做，但是2018Spring版本没有，于是找来了21Spring的gitlet写了一下，前前后后写了6天总共花了37个小时（仅编代码与debug时间），代码大约1200行，其中有大概10小时花在给`merge`函数debug，只是我当时脑子不清醒逻辑有点混乱所以才导致花了这么长时间，其实只是几行代码顺序写反了，找bug找了很久:sob:。附上时间、分数以及[Github链接](https://github.com/Yukang-LIAN/005-UCB-CS61B-GITLET/tree/main/proj2)。

<div align="center">
    <img src="https://raw.githubusercontent.com/Yukang-LIAN/PictureRepo/main/Gitlet_TimeSpend.png"  style="zoom: 40%;" />
    </br>
    <img src="https://raw.githubusercontent.com/Yukang-LIAN/PictureRepo/main/GItlet_Score.png"  style="zoom: 40%;" />
</div>

## Gitlet总体介绍

Gitlet是UCB CS61b课程中的一个项目，实现了简化版的git，支持`add`，`commit`，`log`，`checkout`，`merge`等操作。git是一个分布式的版本控制工具，有一定编程经验的小伙伴肯定对这个软件比较熟悉，我只会`add`，`commit`，`push`，`pull`，这对我来说就已经够用了:satisfied:，如果想要了解更多的git知识，请点击[git官网](https://git-scm.com)查看。

## Gitlet原理

原理上gitlet与git并无二致，都是将一个版本的所有文件内容保存起来，可以在不同版本之间进行切换，保证所有存在过的版本都不会丢失，那如何区分一个版本呢？答案是用commit命令创建一个版本快照（snapshot），就相当于用相机拍下这个时刻的照片，并且按顺序将照片串起来，当想回到某个版本时，只需要按照顺序往前翻就可以了。下图展示了Gitlet的简单原理。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/GitletOverviewFig1.png"  style="zoom: 60%;" />
</div>

当Gitlet初始化时，系统自动生成一个`init commit`，这个`commit`不记录任何文件，之后，每有一个新的版本，会有一个新的`commit`，`commit`会记录版本文件信息，也就是图中的指针指向的两个文件。

当前最新版本的`commit`会由一个`Head`进行标记，此处是`commit2`，也就是说当前文件夹内只包含`3.txt`和`4.txt`，当需要切换到`commit1`时，我们可以看到`commit1`记录了`1.txt`和`2.txt`，那如何进行版本转换呢？只需要将`Head`移动到`commit1`上，读取`commit1`记录的文件`1.txt`和`2.txt`，将`commit1`记录的文件重新写入到当前文件夹，并且删除`3.txt`和`4.txt`即可完成版本转换，现在的文件夹就只包含`1.txt`和`2.txt`了。

上图中`commit1`和`commit2`内容完全不同，如果`commit1`和`commit2`存在相同的文件，只需要保存一个就可以，这样能够在很大程度上节省空间。如下图所示。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/GitletOverviewFig2.png"  style="zoom: 60%;" />
</div>

这样转换版本时只需要将`4.txt`删除，添加新的`1.txt`就可以了。

除此之外，gitlet还可以实现不同的分支功能，当多人进行同时开发时，为了互不影响，可以采取多个路线同时进行的方式，在gitlet中叫做`branch`，

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/GitletOverviewFig3.png"  style="zoom: 50%;" />
</div>

## Gitlet内部结构实现

## 各功能编写思路

### init

### add

### commit

### rm

### log

### global-log

### find

### status

### checkout

### branch

### rm-branch

### reset

### merge

## 调试指南

## 踩过的坑

## 总结