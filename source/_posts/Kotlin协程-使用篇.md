---
title: Kotlin协程 - 使用篇
date: 2020-05-25 22:57:26
tags:
 - kotlin
 - 协程
categories:
 - 编程语言
---

使用协程已经有较长的时间了，但一直停留在launch、async启动协程，suspend方法挂起的阶段。这段时间系统梳理Kotlin知识时才发现，对于协程（仅对Kotlin）还有很多概念不甚了解。例如CoroutineScope对协程生命周期的重要性、协程父子结构的作用、结构化并发、一些Kotlin协程中约定俗称的规定等。

# 概述

## 什么是协程

说到协程，不同的人，有不同的认识。去网上搜索，也有各种各样的答案。

有人说，协程是轻量级线程，从API使用方式的相似性上，的确很像：新启动的协程

### 协程解决了什么问题，如果没有协程，我们要如何解决



## Kotlin的协程



# 启动协程



# 核心组件

## 作用域



## 协程上下文

是什么？

### 调度器



### Job



## 自问自答

- CoroutineScope、CoroutineContext、Job的关系
- 为什么不要使用GlobalScope
- 



# Kotlin协程的额外功能

- 异步流
- 通道
- 监督
- 使用协程+通道实现actor





