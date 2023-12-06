---
title: C2编译match阶段
date: 2023-4-25 17:25:48
tags:
- Hotspot
- Java
- C2 match
- BRUS
---

本文主要分析C2编译器的指令选择原理，如有偏差，请指正。
<!--more-->

#

ad文件是个 DSL（domain-specific language），描述了一个BURS匹配系统的匹配规则。

