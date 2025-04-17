---
title: "What are Unix pipes and why do they matter? An analysis through Python"
date: "2024-12-27"
description: "A primer on what are pipes and how Unix implements these for inter-process communication"
summary: "A primer on what are pipes and how Unix implements these for inter-process communication"
tags: ["unix"]
categories: ["software engineering"]
ShowToc: true
TocOpen: false
draft: true
---

There's a lot of situations where we might want to work with multiple processes, especially in Python because we can't (yet!) implement parallelism using a single process working on multiple cores. Parallelism means that 2 tasks execute at the same time, and this is impossible in Python as long as the Global Interpreter Lock (GIL) is in place.

> CPython's memory management isn't thread-safe because it implemented reference counting to garbage collect cyclic objects (Python the language doesn't specify garbage collection, so different implementations have their own garbage collection). This is analogous to what Ruby's first reference language implementation did before To avoid concurrency problems with multiple libraries mutating the state of these variables, CPython added the GIL in the first major version to ensure that 2 threads or interpreters within the same process can't reference the same global variable at the same time (hence the word "lock": it acquires a lock on global variables).  A side effect is that it makes single-threaded programs faster at the expense of multi-threaded programs being unable to execute in parallel.
>
> This means that historically parallelism in Python has been achieved through multiple processes, typically provided by the `multiprocessing` library. However, there's a lot of ongoing work to remove the GIL from CPython: 3.12 allows multiple interpreters within a single process to each have their own GIL (see [PEP 684](https://peps.python.org/pep-0684/)) and 3.13 intorduces an experimental feature to build an interpreter *without* a GIL (see [PEP 703](https://peps.python.org/pep-0703/)).

Given that processes have their own virtual address space, a process can't refer to the same symbols in another process because their virtual memories are isolated due to memory virtualization. This is where inter-process communication comes in: they're mechanisms for different processes to send and receive information.

Thus, this primer is relevant because `multiprocessing` (and similar libraries in most languages) heavily use pipes for inter-process communication (IPC).

I'll first show how pipes can be used to send information between processes using Python. I'll then explore the key characteristics of pipes, and how they differ from regular files. Finally, I'll discuss at a high level how they're implemented as circular buffers in kernel memory and how that connects to their key characteristics.

## How to send data through pipes

This is a simple program to create child processes that communicate via pipes to send data from one process to the other:

```python
from multiprocessing import Process, Pipe

def sender(conn):
    """Process that sends data through the pipe"""
    messages = ["Hello", "World", "Done"]
    for msg in messages:
        print(f"Sending: {msg}")
        conn.send(msg)  # Send string through the pipe
    conn.close()


def receiver(conn):
    """Process that receives data from the pipe"""
    while True:
        try:
            msg = conn.recv()  # Blocks until data is available
            print(f"Received: {msg}")
        except EOFError:  # Sender has closed the connection
            break

if __name__ == "__main__":
    # Create pipes
    parent_conn, child_conn = Pipe()

    # Create processes
    p1 = Process(target=sender, args=(child_conn,))
    p2 = Process(target=receiver, args=(parent_conn,))

    # Start processes
    p1.start()
    p2.start()

    # Wait for completion
    p1.join()
    p2.join()

```

What it's doing is:

1. Creating the reader and writer pipes
2. Creating the processes with 2 arguments: the functions they'll execute and their pipe connections
3. Start the processes, close the pipes, and join them when they're done

## What are Unix pipes?

Pipes are a key mechanism for IPC because they allow the sending process to send data sequentially to the receiving process.

Pipes are implemented as a [standard Unix file types](https://en.wikipedia.org/wiki/Unix_file_types#FIFO_(named_pipe)). They have several differences to regular files, such as:

- Storing data in memory rather than on disk
- Enabling uni-directional communication
- A sequential rather than random access of data

I'll explore their key characteristics in the following subsections.

### Files stored in memory rather than on disk

```python
import os
import time

r, w = os.pipe()
pid = os.getpid()

print(f"PID: {pid}")

time.sleep(5000)  # Keep process running
```

To explore the concept of a file stored in memory, the above program creates an anonymous pipe and waits for 5000 seconds so that we can examine its file connections with `lsof`.

```bash
$ lsof -p 14269
...   FD   TYPE             DEVICE SIZE/OFF     NODE NAME
...  cwd    DIR                1,8      224 85103852 /Users/jadlp/repos/experimental/pipes
...  txt    REG                1,8     8824 76636094 /Users/jadlp/.pyenv/versions/3.12.2/bin/python3.12
...  txt    REG                1,8   143328 75315401 /usr/local/Cellar/gettext/0.22.5/lib/libintl.8.dylib
...  txt    REG                1,8  6544528 76636095 /Users/jadlp/.pyenv/versions/3.12.2/lib/libpython3.12.dylib
...    0u   CHR              16,13   0t7971      671 /dev/ttys013
...    1u   CHR              16,13   0t7971      671 /dev/ttys013
...    2u   CHR              16,13   0t7971      671 /dev/ttys013
...    3   PIPE 0xff0f28fce5f755d8    16384          ->0xa2f31527b9cd9fdd
...    4   PIPE 0xa2f31527b9cd9fdd    16384          ->0xff0f28fce5f755d8
```

The output shows the file connections currently open by the process:

- The current working directory of the process (`cwd` of file type `DIR`) is `/Users/jadlp/repos/experimental/pipes` (directories are files, hence the file connection)
- 3 regular (`REG`) files it's dynamically linked to
- 3 character device (`CHR`) files connected to the terminal for standard input (`0u`), standard output (`1u`), and standard error (`2u`)
- 2 pipe (`PIPE`) files with hexadecimal identifiers

> The 3 regular files aren't the only ones that are dynamically linked to this process: `libSystem.B.dylib` is also dynamically linked. That doesn't appear in this output because `libSystem.B.dylib` is loaded into memory during process initialization, all of its symbols are resolved, and the file connection is closed because it isn't needed at runtime. This commonly happens with system libraries because they provide low-level functionality that's frequently used by programs to run.
>
> Inversely, these 3 regular files remain with an open connection for symbol resolution and relocation at runtime because they aren't fully loaded into memory. This is **an optimization to reduce initial startup time and memory usage** by resolving symbols only when they're needed.

The table shows us that pipes exist in memory rather than on disk, which is suggested by:

- No inode number under `NODE`, which is the unique identifier for files in the filesystem, precisely because anonymous pipes are in memory
- A hexadecimal rather than device number under `DEVICE` because it's the memory location in which the pipe file is stored in memory

### Files for uni- rather than bi-directional communication and data flow

Both regular files or sockets can arguably be used for bi-directional communication because the same file can be read or written to. Pipes are different because you need a dedicated pipe to write to and another dedicated pipe to read from. You'll error if you try to write from your read pipe or vice verse!

```python
import os

r, w = os.pipe()

# Try writing to read end
try:
    os.write(r, b"test")
except OSError as e:
    print(f"Writing to read end: {e}") # Writing to read end: [Errno 9] Bad file descriptor

```

### Files with sequential rather than random access patterns

Regular files have random access patterns: you can read memory in any order. Pipes are more restrictive in that you need to read data sequentially in a first-in, first-out order (FIFO). This is why pipe files are also refered to as FIFO files.

### Files with single rather than multiple data access

A corrollary of being FIFO is that data can only be read **once** from a pipe: after you read from a pipe that data gets discarded from the pipe so that you can read the next data. This is unlike files, where multiple processes can read the same data (as long as it's not changed).

```python
# This writes to an actual file on disk
with open("data.txt", "w") as f:
    f.write("hello")

# This reads from disk
with open("data.txt", "r") as f:
    data = f.read()
```

The key difference is that pipes store data **in memory rather than on disk** to save on I/O operations on disk (for reference: the latency of an HDD's seek time is around 2 miliseconds, which would add a big overhead to IPC). It also means data is lost if it's there when the process stops.

### Files with varying visibility and lifetimes

The caveat here is that you can create 2 types of pipe files:

- Anonymous pipes, which only live in a single process
- Names pipes, which are available to any process in the file system

In both cases they're stored in memory. The differences are that:

- Named pipes are available to any process in the filesystem
- Named pipes persist until they're deleted or the machine crashes or turns off (since it's volatile memory), rather than when its process ends

Pipes are also:

- Transient because data is removed once it's read
- Unidirectional
- OS manages memory buffer size and flow control

# Comparison of Pipes vs Regular Files

| Characteristic | Pipes | Regular Files |
|----------------|-------|---------------|
| **Storage Location** | Memory (kernel buffer) | Disk |
| **Persistence** | Anonymous: Temporary until process ends; Named: Until deleted | Until deleted |
| **Data Flow** | Unidirectional (one-way) | Unidirectional per file descriptor |
| **Seeking** | Not supported (sequential only) | Supported (random access) |
| **Multiple Access** | Data consumed when read | Many processes can read same data |
| **Buffer Size** | OS dependent (16KB-several MB) | Limited by disk space |
| **Blocking Behavior** | Blocks when full/empty | Blocks on disk I/O or full disk |
| **Creation** | `os.pipe()` or `os.mkfifo()` | `open()` with file path |
| **Reading Empty** | Blocks until data available | Returns immediately with EOF |
| **Writing Full** | Blocks until space available | Fails if disk full |
| **Primary Use** | Inter-process communication | Data storage |
| **Lifetime** | Anonymous: Process lifetime; Named: Until deleted | Until deleted |
| **Access Pattern** | First in, first out (FIFO) | Random access |
| **Visibility** | Anonymous: Process only; Named: Filesystem | Filesystem |
| **Performance** | Faster (memory-based) | Slower (disk-based) |
| **Data Reuse** | Data can be read only once | Data can be read multiple times |
| **Error Handling** | SIGPIPE on write to closed pipe | No special signal handling |
| **Buffer Management** | Managed by OS | Managed by filesystem |
| **File Descriptor Type** | Special file type in /proc | Regular file in filesystem |
| **Maximum Size** | Limited by pipe buffer | Limited by filesystem/disk |

## Example Code Demonstrating Key Differences

```python
# Regular File
with open("file.txt", "w") as f:
    f.write("data")     # Writes to disk
    f.seek(0)          # Can seek
    f.write("new")     # Can overwrite

# Pipe
r, w = os.pipe()
os.write(w, b"data")   # Writes to memory
# Cannot seek
# Cannot overwrite previous data
data = os.read(r, 4)   # Data is consumed
```

# Notes

Pipes are implemented as (usually circular) buffers within the kernel. If the buffer becomes overloaded because the consumer is too reading too slowly, the producer will be told to *wait* by the system.

Where pipes are used:

- Shell-scripting (e.g. `grep kernel | less`, where `|` is for a pipe)

Unix pipes can be creating with 2 commands:

1. `pipe()` for an anonymous pipe
2. `mkfifo()` for a named pipe

Anonymous pipes exist as long as a file descriptor in the creating process is open on it. Named pipes exist until explicitly deleted or the system shutsdown.

They both return 2 file descriptors (recap: what is a fd? Why are 2 needed?)

Pipes exist as long as either end of the pipe is open (but is broken if only 1 end exists and you try to use it). It is cleaned after you close() both ends. => Why doesn't the OS just clean up the pipe if either end closes if it'll just error? => widowing pipes sends a signal SIGPIPE

To pass fd across processes you need to use fork() from parent to child.

<https://wiki.osdev.org/Unix_Pipes#:~:text=Implementation,second%20creates%20a%20named%20pipe>.
