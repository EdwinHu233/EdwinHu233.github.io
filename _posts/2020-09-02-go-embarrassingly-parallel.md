---
layout: post
title: "Golang 笔记：“尴尬的并行”"
---

> 这篇博客是 *The Go Programming Language* 8.5 节的笔记。

有一类简单的并行问题：
所有输入数据两两之间都是独立的，
我们只需要对每个输入数据分别求解，
最后把结果汇总起来。
这种问题被称为 "embarrassingly parallel" 。
