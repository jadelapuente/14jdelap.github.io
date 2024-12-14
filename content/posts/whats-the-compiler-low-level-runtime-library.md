---
title: "Understanding compiler runtime libraries: when hardware needs software's help"
date: "2024-12-14"
description: "Learn how the compiler inserts functions to execute operations that the machine can't perform with native instructions."
summary: "Learn how the compiler inserts functions to execute operations that the machine can't perform with native instructions."
tags: ["c", "compilation", "linking", "symbols", "assembly"]
categories: ["c", "software engineering"]
# series: ["AI"]
ShowToc: true
TocOpen: false
---

> This is the third and final article in an introductory series about compilation, linking, and runtime libraries.
>
> The [first article]({{< relref "/posts/how-to-diagnose-and-mitigate-linking-errors/" >}}) explored how programs are compiled and linked, and how you can use that to solve symbol errors. It's the best starting point if you have little context.
>
> The [second article]({{< relref "posts/python-decimal-module-performance-optimization/">}}) showed a real-world case study where a missing compiler runtime symbol caused a 6x performance regression in a key journey heavily reliant on Python's `decimal` module.

What motivates this final article is the root cause of the missing symbol in the case study: the dynamic linker couldn't find the compiler's low-level runtime library. Since that confused me, I'm hoping to explain these libraries to share the context I wish I had when I was debugging the symbol error in `_decimal`.

## What Are Compiler Runtime Libraries?

Compiler runtime libraries provide low-level support functions that are automatically called when the compiler determines an operation is too complicated to emit inline code for. Most of its functions are for arithmetic operations that the processor can't call directly via a native instruction.

> Compiler runtime libraries also have functions for exception handling and miscellaneous operations, but they won't be the focus of this article.

The 2 most common compiler runtime libraries are

- `libgcc` for GCC-compiled programs
- `compiler-rt` for LLVM compilers like Clang

> These arithmetic operations are in the compiler rather than language runtimes so that they can be re-used across languages. If they were in each language they'd have to be re-implemented on a per-language basis.

## How They Work: A Deep Dive on `__udivti3`

Let's look at how these libraries help with the most common scenarios: extended precision arithmetic. These are added to your binary files when your code uses types larger than the CPU's native word size, the compiler runtime kicks in.

For example, in my machine the native word size is 64 bits:

```bash
$ getconf LONG_BIT
64
```

> Word sizes are important because they're the size of a register. A native machine instruction can't be larger than a register because the computer needs to be able to load the data into a register to execute an instruction.

So if I write a program that uses up to 64 bit types like:

```c
int main()
{
  unsigned int a = 12;
  unsigned int b = 3;
  unsigned int result = a / b;
}
```

The compiler will translate it to native machine instructions. We can verify this given the minimal symbol table:

```bash
$ nm a.out
0000000100000000 T __mh_execute_header
0000000100000f80 T _main
```

But if I write a program that uses 128 bit types:

```c
int main()
{
  __uint128_t a = 12;
  __uint128_t b = 3;
  __uint128_t result = a / b;
}
```

The symbol table will also include `__udivti3`:

> The starting `__` means that the symbol comes from the compiler or system and isn't meant to be drectly called in user code. In contrast, user code functions start with a single leading underscore like `_main`.
>
> `__udivti3` stands for **u**nsigned **div**ision **t**etra integer 3 (the function's version number). This just means it's the 3rd version of a function for unsigned division for 128 bit integers.

```bash
$ nm a.out
                 U ___udivti3
0000000100000000 T __mh_execute_header
0000000100000f50 T _main
```

As per before, in this case the symbol `__udivti3` is added precisely because these low level runtime libraries provide functions that are too complex to just inline. If they had been inlined we wouldn't have seen it in the symbol table — only in the disassembly of the binary file.

> If you're curious, you can find an almost 200 line C file that implements `__udivti3` in LLVM [here](https://github.com/llvm-mirror/compiler-rt/blob/master/lib/builtins/udivmodti4.c).
>
> Thus, if the compiler inlined the machine instructions for `__udivti3` it would be much harder to make sense of the disassembly of the program.

Under the hood, the compiler also inlines multiple instructions to prepare the `__udivti3` operation because, as a 128 bit operation in a machine with 64 bit registers, it needs multiple registers to load the data.

Here's the full disassembly:

```objdump
0000000100000f50 <_main>:
100000f50: 55                           pushq   %rbp
100000f51: 48 89 e5                     movq    %rsp, %rbp
100000f54: 48 83 ec 30                  subq    $48, %rsp
100000f58: 48 c7 45 f8 00 00 00 00      movq    $0, -8(%rbp)
100000f60: 48 c7 45 f0 0c 00 00 00      movq    $12, -16(%rbp)
100000f68: 48 c7 45 e8 00 00 00 00      movq    $0, -24(%rbp)
100000f70: 48 c7 45 e0 03 00 00 00      movq    $3, -32(%rbp)
100000f78: 48 8b 7d f0                  movq    -16(%rbp), %rdi
100000f7c: 48 8b 75 f8                  movq    -8(%rbp), %rsi
100000f80: 48 8b 55 e0                  movq    -32(%rbp), %rdx
100000f84: 48 8b 4d e8                  movq    -24(%rbp), %rcx
100000f88: e8 10 00 00 00               callq   0x100000f9d <___udivti3+0x100000f9d>
100000f8d: 48 89 55 d8                  movq    %rdx, -40(%rbp)
100000f91: 48 89 45 d0                  movq    %rax, -48(%rbp)
100000f95: 31 c0                        xorl    %eax, %eax
100000f97: 48 83 c4 30                  addq    $48, %rsp
100000f9b: 5d                           popq    %rbp
100000f9c: c3                           retq

Disassembly of section __TEXT,__stubs:

0000000100000f9d <__stubs>:
100000f9d: ff 25 5d 00 00 00            jmpq    *93(%rip)               ## 0x100001000 <___udivti3+0x100001000>
```

To break this down into pieces, the first important part is setting up the 2 128 bit number by loading 2 registers per each `__uint128_t` variable:

```objdump
100000f58: 48 c7 45 f8 00 00 00 00      movq    $0, -8(%rbp)   # Upper 64 bits = 0
100000f60: 48 c7 45 f0 0c 00 00 00      movq    $12, -16(%rbp) # Lower 64 bits = 12
100000f68: 48 c7 45 e8 00 00 00 00      movq    $0, -24(%rbp)  # Upper 64 bits = 0
100000f70: 48 c7 45 e0 03 00 00 00      movq    $3, -32(%rbp)  # Lower 64 bits = 3
```

Since the machine uses the base pointer register in the current function's stack frame (`rbp`) we now need to move these values from memory to the general purpose registers. This is because in `x86-64` there's a calling convention where, when a function is called, arguments are passed in order to certain registers. For example, the first argument is always loaded into `rdi` and the second to `rsi`.

Thus, we then need to load the 2 arguments across 4 registers to be able to call `__udivti3`:

```objdump
100000f78: 48 8b 7d f0                  movq    -16(%rbp), %rdi  # First number lower bits
100000f7c: 48 8b 75 f8                  movq    -8(%rbp), %rsi   # First number upper bits
100000f80: 48 8b 55 e0                  movq    -32(%rbp), %rdx  # Second number lower bits
100000f84: 48 8b 4d e8                  movq    -24(%rbp), %rcx  # Second number upper bits
```

At this point we can call `__udivti3` and store the result in 128 bits of the stack frame before returning it:

```objdump
100000f88: e8 10 00 00 00               callq   ___udivti3       # Call division function
100000f8d: 48 89 55 d8                  movq    %rdx, -40(%rbp)  # Store upper 64 bits of result
100000f91: 48 89 45 d0                  movq    %rax, -48(%rbp)  # Store lower 64 bits of result
```

## Takeaways: binary structure and performance implications

One of the big takeaways I hope you're gotten from this is better understanding what programs are made of. In this case, I showed how the compiler also has code it sometimes inserts to make programs compatible across platforms. This can be very helpful when you get linking errors!

The other takeaway is that we need to be careful of non-native operations for performance-critical systems. Native instructions are more performant, though the compiler does a lot of heavy lifting to optimize a program's performance for us, and also lead to a smaller binary size.
