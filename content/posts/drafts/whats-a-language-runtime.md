---
title: "What's a language runtime? Understanding runtimes through C and Go."
date: "2024-12-27"
description: "Understand what is a language runtime by examining the binaries of simple C and Go programs."
summary: "Understand what is a language runtime by examining the binaries of simple C and Go programs."
tags: ["go", "c", "compilation", "symbols"]
categories: ["go", "c", "software engineering"]
# series: ["AI"]
ShowToc: true
TocOpen: false
draft: true
---

A runtime is a language's the execution environment: it provides the functions and variables so that a program can execute the code in its language. For example, you need the C runtime to execute C code.

My intention is to compare the runtimes of C and Go by analyzing the symbol tables of simple programs. What motivates me is that there isn't a singular way of implementing a runtime because it depends on each language's design decisions. Thus, this will show how:

- The basic responsibilities of runtimes
- How runtime implementations can differ
- How runtimes reflect a language's design philosophy

> I'm comparing C and Go because:
>
> - C has a much smaller runtime than Go's
> - Go was heavily influenced by C despite making many different design decisions
> - Go's runtime shows how modern languages can build higher-level abstractions while maintaining good performance
