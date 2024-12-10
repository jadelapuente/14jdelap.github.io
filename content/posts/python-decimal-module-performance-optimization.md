---
title: "Python performance optimization: 6x speedup by fixing `decimal` module linking error"
date: "2024-12-07"
description: "Learn how fixing a Python decimal module linking error led to 6x performance improvement: a detailed guide on debugging C extensions and symbol errors in Python interpreters."
summary: "Learn how fixing a Python decimal module linking error led to 6x performance improvement: a detailed guide on debugging C extensions and symbol errors in Python interpreters."
tags: ["python", "compilation", "linking", "symbols"]
categories: ["python", "c", "software engineering"]
# series: ["AI"]
ShowToc: true
TocOpen: false
---

```
Could you implement these?

Image Alt Text


The code blocks could benefit from more descriptive alt text for better accessibility and SEO
Currently missing image alt text for code snippets


Schema Markup


While the article has basic schema markup, it could be enhanced with:

Code snippet schema
Technical article schema
Tutorial schema
More detailed article metadata
```

At my company I led a recent migration of our Python interpreter between C standard libraries. We found that a critical, performance-sensitive journey that heavily used decimals had improved by 6 times (!) after the migration finished between our 2 interpreters. Fixing the old interpreter was still important because it is heavily used by clients using older versions of our software.

The root cause? A multi-year bug in how our compiler dynamically linked at runtime to the `_decimal` C library when the interpreter tried to import `decimal`.

> A helpful companion is the more beginner-friendly article [How to diagnose and mitigate linking errors]({{< relref "/posts/how-to-diagnose-and-mitigate-linking-errors.md" >}})

## Why are we talking about C if this is Python?

Just download the source code of the Python wheel `grpcio` and count the number of C files:

```bash
# Executed from /grpcio-1.66.0
$ find -name "*.c" | wc -l 839
```

Almost 840 C files! I almost feel catfished, I thought this was a Python — not a C — wheel.

This is because Python heavily uses C extensions in both the interpreter and Python wheels for 2 reasons:

1. Performance
2. Access to system calls provided by C but not Python

These Python C extensions are a form of a foreign function interface: functions that allow a programming language to make a function call to a function in another language.

## Finding the Python symbol linking error

We didn't get an explicit error that something was failing — we only saw the 6x performance difference I mentioned above when comparing two different Python interpreters.

I confirmed the issue by running `strace` on the old interpreter as it imported `decimal` — which revealed the missing symbol `__udivti3`:

```bash
$ strace -e trace=open,openat /.../usr/bin/python3.10 -c "import _decimal" 2>&1 | grep decimal
open("/.../usr/lib/python3.10/lib-dynload/_decimal.cpython-310-x86_64-linux-musl.so", O_RDONLY|O_CLOEXEC) = 3
ImportError: Error relocating /.../usr/lib/python3.10/lib-dynload/_decimal.cpython-310-x86_64-linux-musl.so: __udivti3: symbol not found
```

Another way of surfacing the error is by directly importing the `_decimal` in the interpreter, which bypasses the error handling from `import decimal`:

```python
>>> import _decimal
Traceback (most recent las call):
  File "<stdin>", line 1, lin <module>
ImportError: Error relocating /.../_decimal.cpython-310-x86_64-linux-musl.so: __udivti3: symbol not found
```

## Understanding how Python's `decimal` module is imported

Python's native module `decimal` is a wrapper around 2 functionally-identical backends:

- `_decimal`: a C implementation (which itself is a wrapper around [`mpdecimal`](https://www.bytereef.org/mpdecimal/))
- `_pydecimal`: a pure-Python implementation of the same decimal library

When you run `import decimal`, Python first tries to load `_decimal` and if it fails falls back on `_pydecimal`.

```python
# cpython/Lib/decimal.py
try:
    from _decimal import *
    from _decimal import __version__  # noqa: F401
    from _decimal import __libmpdec_version__  # noqa: F401
except ImportError:
    import _pydecimal
    import sys
    _pydecimal.__doc__ = __doc__
    sys.modules[__name__] = _pydecimal
```

See the above code on CPython's GitHub ([link](https://github.com/python/cpython/blob/main/Lib/decimal.py)).

Thus, this error hadn't been seen before because when we imported `decimal` it silently failed and didn't notify us of this problem.

To put the performance of both libraries in context, relative to `_pydecimal` `_decimal` is:

- 30x faster for I/O heavy operations
- 80x faster for numerical computations

## How I found the symbol missed by the dynamic linker

### Sanity checks: comparing the old and new interpreters

My investigation began with a sanity check that we needed the `__udivti3` symbol. Since the error prints the affected file name it's trivial to do this by inspecting the binary file's symbol table with `nm`:

```
$ nm /.../usr/lib/python3.10/lib-dynload/_decimal.cpython-310-x86_64-linux-musl.so | grep _udivti3
        U __udivti3
```

Since the new interpreter has a very different build, my second sanity check was to confirm that it too needed this `__udivti3`. This would allow me to discard some hypothesis for how to mitigate the error, such as if this was due to how we compiled the interpreter.

```
$ nm /.../usr/lib/python3.10/lib-dynload/_decimal.cpython-310-x86_64-linux-gnu.so | grep _udivti3
0000000000045950 t __udivti3
```

What is *very* interesting about this comparison is that the old interpreter expects `__udivti3` to be provided by a dynamic library (since it's an unreferenced symbol) whereas the new interpreter already has it statically linked.

### A dead end: inspecting the binaries of the interpreter's dynamic libraries

Since `__udivti3` is unreferenced the compiler will expect it at runtime. The question is: from where?

An effective way to find the dynamic libraries expected by a binary is to use `ldd` ("list dynamic dependencies").

```bash
$ ldd /.../usr/lib/python3.10/lib-dynload/_decimal.cpython-310-x86_64-linux-musl.so
linux-vdso.so.1 (0x00007ffec49fe000)
libpython3.10.so => /lib/x86_64-linux-gnu/libpython3.10.so (0x00007f334600000)
libc.so => /opt/tm/tools/musl/1.1.18-2/usr/lib/libc.so (0x00007f334000000)
libexpat.so.1 => /lib/x86_64-linux-gnu/libexpat.so.1 (0x00007f3345cf000)
libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007f3345b3000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f3344cc000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f333c00000) /lib/ld-musl-x86_64.so.1 => /lib64/ld-linux-x86-64.so.2 (0x00007f3344c52000)
```

However, none of these file's symbol tables contained `__udivti3`:

```bash
nm -D /lib/x86_64-linux-gnu/libpython3.10.so | grep _udivti3
nm -D /opt/tm/tools/musl/1.1.18-2/usr/lib/libc.so | grep _udivti3
nm -D /lib/x86_64-linux-gnu/libexpat.so.1 | grep _udivti3
nm -D /lib/x86_64-linux-gnu/libz.so.1 | grep _udivti3
nm -D /lib/x86_64-linux-gnu/libm.so.6 | grep _udivti3
nm -D /lib/x86_64-linux-gnu/libc.so.6 | grep _udivti3
nm -D /lib64/ld-linux-x86-64.so.2 | grep _udivti3
```

### Breakthrough: the compiler's low-level runtime library

There's no way to sugarcoat this part: this involved a lot of Googling and wasn't linear. From a few hours jumping through different references I pieced together 3 things:

1. `__udivti3` is a compiler-provided function that enables 128-bit division on machines that don't natively support it
2. The `x86-64` instruction set lacks a native instruction for 128-bit division
3. Compilers provide low-level types for situations like these

Based on the above, I found the symbol in one of my compiler's objects:

```bash
$ nm /.../clang/compiler-rt/lib/linux/libclang_rt.builtins-x86_64.a | grep _udivti3
0000000000000000 T __udivti3
```

## Mitigating the symbol error in the old interpreter

There was significant value in mitigating the problem in the old interpreter. Given the above, it was clear that the root cause was that our build system wasn't able to reference `libclang_rt.builtins-x86_64.a` at build time.

Thus, the fix was adding `deps = ["@//cc/clang:compiler-rt_builtins"]` to our build.

## Why did this missing symbol import error go undetected?

### CPython catched the import error at runtime

What was happening in our old interpreter is that we were

- Importing `decimal`, which first tried to import `_decimal`
- `_decimal` failed due to an `ImportError`
- The error was caught and instead Python imported `_pydecimal`

Thus, the Python interpreter guaranteed `decimal` functionality by catching the error (see the `try`/`except` above) and loading `_pydecimal`. By catching the error it took years for us to notice what was going on — which led to years of significant performance loss of my company's systems.

### Our C library couldn't error at compile-time because it was expecting `__udivti3` from the compiler at runtime

The build system worked as expected. It built the binary, which expected the symbol to be eventually resolved by a dynamic linker. As such, it couldn't fail at compile time because of this — only at runtime, when CPython catched the error.

## Creating tests for import errors

There's two ways you can approach this.

The first is to create unit tests that try to import specific libraries that you know have C extensions.

```python
# First import your test case
import unittest

class DecimalImportTest(unittest.TestCase):
    def test_can_import_decimal(self):
        try:
            import _decimal
        except:
            return self.fail("Could not import _decimal module")
```

The limitations of this approach are that:

1. It requires awareness of whether a library is implemented as a C extension or in Python
2. You need to write a test for every library
3. Given the first 2, it creates more room for omissions or errors

A better way to test this is to programmatically import and catch errors. You can also extend this approach to test all 3rd party Python dependencies are being loaded correctly.

## Takeaways

1. Python has lots of C extensions that can be easily missed
2. Missing C extensions in your interpreter or wheels is expensive
3. You can test missing C files in Python

## Further reading

[How to diagnose and mitigate linking errors]({{< relref "/posts/how-to-diagnose-and-mitigate-linking-errors.md" >}}) is a lighter introduction to compilation and linking.
