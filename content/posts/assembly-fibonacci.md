---
title: "Assembly Fibonacci"
date: "2024-02-19"
description: "Writing recursion in assembly forces you to understand how to organize your control and data transfers across function calls."
summary: "Do you really know assembly if you haven't done it in assembly?"
tags: ["assembly", "fibonacci", "computer systems"]
categories: ["software engineering"]
series: ["software engineering"]
ShowToc: false
TocOpen: false
---

```asm
section .text
global fib

fib:
    push rdi              ; push n
    mov eax, edi          ; set return value to n
    cmp edi, 1            ; check base case
    jle base_case

    dec edi               ; if base case isn't met, dec n to n-1 to do fib(n-1) + fib(n-1)
    call fib              ; call fib(n - 1)

    mov r12d, eax         ; eax is fib(n-1), save in r12d
    dec edi               ; dec n-1 to n-2 to call fib(n-1)

    push r12              ; push fib(n-1) to add at the end
    call fib              ; call fib(n-2)

    pop r12               ; pop fib(n-1)
    lea eax, [r12d + eax] ; add and return fib(n-1) + fib(n-2)

base_case:
    pop rdi               ; pop n for outer function execution
    ret
```

After a very painful process I programmed a function to calculate the nth Fibonacci number with assembly. For this solution, I used the Intel syntax on an x86 architecture.

I found that doing this recursive problem required understanding how to organize your control and data transfers across function calls. This included understanding the stack as a data structure and the rsp and rip registers.

The key details to getting the solution right included:

- Handling the base case
- Having both fibonacci calls in the general case to calculate fib(n-1) + fib(n-2)
- Using the stack to store n and n-1 in the stack frames of each procedure
