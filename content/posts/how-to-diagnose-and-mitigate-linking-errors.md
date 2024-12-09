---
title: "6x'ing performance by fixing a linking error in the Python interpreter"
date: "2024-12-14"
description: "How to diagnose and mitigate linking errors"
summary: "The context and methodology for how to address linking errors."
tags: ["compilation", "linking", "symbols"]
# categories: ["python"]
# series: ["AI"]
draft: true
ShowToc: true
TocOpen: true
---

My case study on [fixing a symbol error in the Python interpreter](/posts/python-interpreter-performance-symbol-error.md) shows how hard but impactful these problems can be: a subtle bug in how we build the C `decimal` library led to a key journey being 6x slower. The fix was just one line in the build file:

```plz
deps = ["@//cc/clang:compiler-rt_builtins"]
```

The limitations of that case study is that it's very specific — it doesn't teach you the key underlying concepts to know how to think about these concepts from first principles.

In this post I want to attempt building that context and sharing a methodology for how you can approach it. I'll explain the concepts using `C` because it's arguably the most influential programming language, but the general principles can be applied to any.

## What is `C`?

C is a programming language. As such, C is used in C code like:

```c
#include <stdio.h>
int main() {
  printf("Hello, world!")
}
```

Yet like any language it is defined by a specification — in C's case, ISO C. For example, `printf` is defined by ISO C like this:

## The `C` compilation process

### Preprocessing

### Assembling

### Linking

#### Static linking

#### Dynamic linking

## Programs as symbols

## Compiler runtime libraries

## A methodology for linking errors
