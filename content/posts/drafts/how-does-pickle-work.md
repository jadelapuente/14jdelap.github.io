---
title: "How does Python's pickle serialize data for inter-process communication?"
date: "2024-12-29"
description: "A primer to understand data serialization in Python's widespread pickle library."
summary: "A primer to understand data serialization in Python's widespread pickle library."
tags: ["unix", "python"]
categories: ["software engineering"]
ShowToc: true
TocOpen: false
draft: true
---

- Processes have their own virtual address space
- There are different ways of having

Functions are pickable in Python when they are:

Defined at module level (not nested inside other functions/classes)
Importable from their module
Have no references to global variables or external state

In your code, sender, receiver, and print_process_id meet these criteria since they:

Are module-level functions
Only use their passed arguments and built-in/imported modules
Don't reference any closure variables

Things that are NOT pickable include:

Lambda functions
Methods of class instances
Functions defined inside other functions
Objects with references to unpicklable things

The multiprocessing module requires pickling to pass objects between processes, which is why these constraints matter for Process targets.
