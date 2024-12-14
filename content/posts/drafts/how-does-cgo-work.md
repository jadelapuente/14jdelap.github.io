---
title: "How does Go embed C code? Analyzing Cgo in action."
date: "2024-12-07"
description: "Understand how Go embeds and calls C within a Go package by inspecting the symbol tables of the executable files with Cgo."
summary: "Understand how Go embeds and calls C within a Go package by inspecting the symbol tables of the executable files with Cgo."
tags: ["go", "c", "compilation symbols"]
categories: ["go", "c", "software engineering"]
# series: ["AI"]
ShowToc: true
TocOpen: false
draft: true
---

There's a few reasons I've been interested in understanding `Cgo`:

- Understanding language runtime differences and their implications for interoperability (for example, how Go's goroutines and garbage collection make it harder to embed in other languages, while C's minimal runtime makes it easier)
- Understanding how code from 2 different languages can compile to the same binary
- C has historically been the de-facto language for systems programming and has many useful libraries (like the Python API)

Thus, in this article I'll compare 3 "Hello, world!" programs:

- One written in C
- One written in Go
- One written in Go with Cgo

## "Hello, world!" in C

```c
#include <stdio.h>

void hello() {
  printf("Hello from Cgo!\n");
}

int main() {
  hello();
}
```

After compiling the above program `ls -l` tells us that it's 8KB in size (to be exact, 8448 bytes).

```bash
$ ls -l c_hello_world/a.out
-rwxr-xr-x  1 jadlp  staff  8448 Dec 12 21:43 c_hello_world/a.out
```

`size` helps us better understand the binary's composition:

```bash
$ bash c_hello_world/a.out
__TEXT  __DATA  __OBJC  others  dec     hex
4096    0       0       4294975488      4294979584      100003000
```

Both tell us that:

- The code in `__TEXT` section is `4096` bytes, likely because it's the size of a block and thus the minimum number of bytes
- `dec` shows us that the maximum the sum of all segments is

## "Hello, world!" in Go

## "Hello, world!" in Cgo
