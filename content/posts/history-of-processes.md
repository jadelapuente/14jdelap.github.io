---
title: "From proc to task: how Linux changed processes"
date: "2024-04-03"
summary: "How Linux redefined the internal implementation of processes and threads around memory mappings."
tags: ["unix", "linux", "operating systems"]
categories: ["technology"]
series: ["AI"]
ShowToc: false
TocOpen: false
draft: true
---

At a machine level, programs execute by reading instructions from their program counter and using them to transform data.

Every program is a series of instructions. A process is an *instance* of a program: a running program.
