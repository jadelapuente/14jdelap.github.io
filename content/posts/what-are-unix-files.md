---
title: "What are regular Unix files?"
date: "2024-12-27"
description: "A primer on regular files and their key characteristics"
summary: "A primer on regular files and their key characteristics"
tags: ["unix"]
categories: ["software engineering"]
ShowToc: true
TocOpen: false
draft: true
---

This is meant to be a brief primer on regular files, which are easy to overlook in their simplicity. This isn't meant to be exhaustive, and there's a lot I won't talk about (inodes!, file descriptors!, POSIX permissioning!).

Regular files are one of the 7 [standard Unix file types](https://en.wikipedia.org/wiki/Unix_file_types):

| File Type     | Purpose                                   | Common Examples                               |
|---------------|-----------------------------------------------|----------------------------------------------|
| Regular       | Persistent data storage                       | `.txt`, `.jpg`, `.mp3`        |
| Directory     | Structuring the file system     | `/home`, `/etc`, `/usr`                            |
| Block         | Provide buffered access to hardware devices  | Hard drives (`/dev/sda`), SSDs                 |
| Character     | Provide unbuffered access to hardware devices| Terminals (`/dev/tty`), printers               |
| Pipe          | Uni-directional inter-process communication  | Named pipes created with `mkfifo`              |
| Socket        | Bi-directional inter-process communication   | Unix domain sockets, network sockets         |
| Symbolic link | Pointing to another file or directory          | Shortcut files created with `ln -s`            |

As the table hints at, they serve a crucial purpose: permanent data storage in file systems. As such, I'll try to convey 3 key characteristics of regular files:

- They're unstructured streams of bytes
- They have random access patterns
- They allow multiple access

## Regular files as unstructured streams of bytes on blocks

If you take 1 idea from this, it's that files are streams of bytes with no underlying structure written on blocks.

Filesystems use "blocks" as the lowest unit of physical storage. Blocks are typically 4KB in size, and files are stored in collections of 1 or more blocks. Thus, each file stores the blocks it uses in its metadata so that when the OS interacts with a file it reads and writes to its blocks.

Thus, the operating system just provides blocks to read to and write from, and you can manipulate its data through fairly flexible system call APIs (e.g. `open`, `read`, `write`). This means that it's up to the *applications* opening a file to interpret the bytes of a file and see if the data is as expected or corrupted — which is why corrupted files happen in the first place!

### Images as regular files

For example, PNG images have a structured format that specifies how bytes should be structured for the computer to interpret the file (see [PNG file format](https://en.wikipedia.org/wiki/PNG#File_format)). Yet that doesn't stop me from being able to create an arbitrary file with `.png` that doesn't conform to the format or modify an existing one to corrupt it!

### Code as regular files

Another example is that we can all write wrong code, like a `.c` file with Python code:

```c
print("This isn't C!")
```

I can still save it because the file system allows me to indiscriminately edit the contents of my files (assuming I have the appropriate permissions). However, if I try to compile the C program I'll get compile-time errors because it doesn't conform to the C code that the compiler expects.

### Postgres as regular files

A good example of this idea are SQL databases like Postgres, which heavily use regular files to store data.

Naturally, the application logic of Postgres — just like any program — is composed of a group of source code in regular files that are compiled to binary (link to [GitHub](https://github.com/postgres/postgres)). In my Unix file system my Postgres binary is in `/usr/local/opt/postgresql@14`.

Yet it's not just the source code: the underlying data read and written through Postgres also goes to regular files. Like other databases, Postgres typically stores its data in `/var/` directories to signal files that change during the program's normal operations (mine is in `/usr/local/var/postgresql@14`). Postgres databases themselves are organized within the `/base/` subdirectory, with each subdirectory being named after the object ID (OID) associated with each database in Postgres.

This is a simplified tree view of how said directory tree might look like:

```bash
postgresql@14/
├── base/                  # Database files
│   ├── 1/                 # Database OID
│   │   ├── 12345          # Table file
│   │   └── 12345_fsm      # FSM file
├── ...
```

Where, again, each OID is associated by Postgres to a database name — so there's a 1-to-1 mapping between them.

```psql
FROM pg_database;
  oid  |  datname
-------+-----------
 14034 | postgres
 12345 | template1
 14033 | template0
 16385 | homework
 ...
```

For example, if I `ls` in one of the subdirectories for a database's files I get a list of files that stores the database's data:

```bash
➜  16385 ls -alh
total 17024
drwx------  296 jadlp  admin   9.3K Oct 29  2022 .
drwx------    7 jadlp  admin   224B Oct 29  2022 ..
-rw-------    1 jadlp  admin   8.0K Oct 29  2022 112
-rw-------    1 jadlp  admin   8.0K Oct 29  2022 113
-rw-------    1 jadlp  admin   112K Oct 29  2022 1247
-rw-------    1 jadlp  admin    24K Oct 29  2022 1247_fsm
...
```

And each file is a regular file that can be read from or written to according to its file permissions (note how `file 1247` describes the file as `data`):

```bash
➜  16385 file 1247
1247: data
➜  16385 stat 1247
16777224 49754225 -rw------- 1 jadlp admin 0 114688 "Dec 30 15:34:17 2024" "Oct 29 10:08:36 2022" "Oct 29 10:08:36 2022" "Oct 29 10:08:35 2022" 4096 224 0 1247
```

We can even use `psql` to find that the file `1247` stores the data for `pg_type`, which is a critical system catalog with Postgres type definitions:

```psql
postgres=# SELECT relname FROM pg_class WHERE oid = 1247;
 relname
---------
 pg_type
(1 row)
```

So deleting this file would mean the database wouldn't be able to interpretet the data or run queries! How could it without understanding its data types?

We can easily verify this (don't try this at home!):

```bash
➜  16385 rm 1247
```

And then try to use a query with a data type in that database in Postgres.

```psql
homework=# SELECT 1;
2024-12-30 22:02:44.498 GMT [60150] ERROR:  could not open file "base/16385/1247": No such file or directory
2024-12-30 22:02:44.498 GMT [60150] STATEMENT:  SELECT 1;
ERROR:  could not open file "base/16385/1247": No such file or directory
```

### Structured vs unstructured file types: contrasting regular and directory files

This feature of the user being able to write any data to regular files doesn't apply to all file types. For example, you can't just open a directory file and write to it like a regular file. This is why the below program will output `Error occurred: [Errno 21] Is a directory: 'test_dir'`:

```python
import os

def try_write_to_directory():
    # Create a test directory
    os.makedirs("test_dir", exist_ok=True)
    print("Created directory: test_dir")

    try:
        # Try to open the directory for writing
        print("\nAttempting to open directory for writing...")
        fd = os.open("test_dir", os.O_WRONLY)

        # We won't reach this code due to the error
        print("Writing to directory...")
        os.write(fd, b"This shouldn't work")
        os.close(fd)

    except OSError as e:
        print(f"Error occurred: {e}")

    # Clean up
    os.rmdir("test_dir")
    print("\nCleaned up test directory")

if __name__ == "__main__":
    try_write_to_directory()
```

The reason for this is that directories are a structured file format. To protect it, Unix provides APIs that give less control to the user to prevent them from corrupting their file system — unlike regular files, which don't have those protections. That's why you can't use `open` or `write` on directories but a more controlled API like `opendir` and `readdir` for both safety and convenience.

## Random access

Our Postgres example hinted at another key feature of regular files: random access.

Random access means that we can read byte data at any location in the file's byte stream. This contrasts with sequential access, where we need to read data sequentially — so you can't just read an arbitrary byte without reading all previous bytes before it. This is why the `lseek()` system call exists: to offset a byte position when reading from a file.

```python
f.seek(3)        # Move the file pointer to byte 3 of the file `f`
f.seek(129)      # Move the file pointer to byte 129 of the file
print(f.read(1)) # Read 1 byte from position 129
```

As a contrast to regular files, pipes need to be read sequentially! You can't just jump at an arbitrary data point in a pipe's memory.

## Multiple access

Multiple access means that multiple processes can have the file connection to read and write to the file. Since regular files handle persistent storage, this is helpful in cases like:

- Multiple services writing to the same log files
- Multiple database systems reading the same tables
- Multiple processes reading the same shared config

As a corollary, multiple access is why we have concurrency risk and locking mechanisms to address those risks. If you have multiple processes writing to the same file you have a potential race condition that could create inconsistencies.

Again, pipes are a contrast: there can only be 1 reader and 1 writer rather than multiple.

## Conclusion

The key idea I wanted to convey is that our systems run on regular files because persistent storage *depends* on regular files to store data on the filesystem's blocks. Files are just some metadata and collections of blocks, where as long as you have permissions you can write anything because applications are responsible for interpreting the data.

As I'll explore in future articles, there's a need for other file types with different trade-offs than regular files. For starters, there's other use cases where

- Data doesn't need to persist
- Access patterns should be sequential rather than random
- Latency is critical

In cases like these, regular files are probably not the best fit because they involve writing to disk. Reading and writing from memory has a much lower latency (100 nanoseconds vs 100K nanoseconds to 2M nanoseconds, depending on the memory device), so that's a big motivation for other file types with different trade-offs.
