---
title: "Writing a performant web crawler: why systems knowledge is important"
date: "2024-12-29"
description: "A simple but high-performance web crawler based on systems insights"
summary: "A simple but high-performance web crawler based on systems insights"
tags: ["unix", "python", "go"]
categories: ["software engineering"]
ShowToc: true
TocOpen: false
draft: true
---

I recently went through a technical interview where I wrote a web crawler that was 100 times more performant than the next best submission (that eventually shrunk to ≈5 times better, but still!). What I find shocking is that this was for a senior role at a high-profile AI company paying at the top of the market, so I would've expected multiple submissions comparable to mine.

My objective with this is to show:

1. How I break down an engineering problem
2. Key technical insights required for a competitive solution

This means summarizing years of learning, so I hope others find this useful and learn from it.

## Defining the problem: what is a web crawler?

## Crawling as depth-first search on a directed graph (digraph)

## Key ideas

- BFS search of a digraph
- Set of hashes to minimize memory use ()
- Parallelism for CPU-bound tasks
  - Processing each website
  - Each worker having its own network connections
- Parallelism leads to choice of Go over Python
- Filtering out
  - Non-HTTP links
  - Query parameters
  - Duplicate links
- Concurrency for I/O bound tasks
  - Use different goroutine workers while they're idle
    - Waiting for HTTP responses (done by Go's network poller)
    - Waiting for the mutex to write to the set
    - Waiting on channel operations if the queue is empty
  - Cancel workers
  - Load balance work via the URLQueue's work-stealing mechanism (push work to the least loaded queue)
- Worker pool design
