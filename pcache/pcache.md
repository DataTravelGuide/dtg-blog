# pcache: Persistent Memory Block Device Cache

## Overview

`pcache` is a high-performance Linux kernel module designed to use persistent memory (pmem) as a cache layer for block devices. It is a modern, low-overhead alternative to traditional caching solutions like `bcache`, with an architecture optimized for byte-addressable persistent memory accessed via DAX.

Originally developed as `cbd_cache`, `pcache` has been decoupled from the CBD infrastructure to support **any pmem device** as a caching backend. 

## Key Features

### Low Latency Path

`pcache` minimizes software overhead by eliminating asynchronous cache writes. Unlike `bcache`, which uses `submit_bio` and defers metadata updates until I/O completion, `pcache` leverages direct `memcpy` to DAX-mapped memory, followed by immediate metadata updates.

This results in significant latency reduction, especially for small I/O sizes. In 4K random writes, `pcache` achieves latencies as low as 7.72μs, compared to 25.30μs for `bcache`.

### Multi-Tree for High Concurrency

To fully utilize the hardware concurrency of pmem, `pcache` employs a multi-tree architecture:

- Each backend has an **independent cache tree**
- Each tree is partitioned into **multiple RB trees** by logical address ranges

This design ensures minimal contention and scales effectively with multiple I/O threads. In 32-thread tests, `pcache` achieves over 1.4 million IOPS for 4K random writes, far exceeding `bcache`'s 210K.

### Stable and Predictable Performance

The simplified and deterministic I/O path results in:

- Lower standard deviation of latency
- Better tail latency behavior

In benchmark comparisons:

- `pcache` standard deviation: 36.45
- `bcache` standard deviation: 937.81

This makes `pcache` ideal for latency-sensitive workloads.

### Non-Intrusive Design

`pcache` requires no metadata on the backend disk, enabling:

- Use of existing disks without reformatting
- Easy enable/disable of cache without data migration

### Crash-Consistent Writeback

Writebacks in `pcache` follow a **log-structured** model. Overwritten dirty cache data is flushed in order to the backend, preserving crash consistency of the backend disk, even if pmem fails.

This behavior is critical for cloud and disaster recovery use cases.

### Per-Backend Cache Size Configuration

Unlike `bcache`, which pools cache among all backends, `pcache` supports assigning **dedicated cache space per backend**. This improves predictability and avoids interference between workloads.

## Comparison: pcache vs bcache

| Feature                     | pcache                   | bcache                     |
| --------------------------- | ------------------------ | -------------------------- |
| Target cache device         | Persistent Memory (pmem) | Block Device (e.g. SSD)    |
| Access method               | DAX / memcpy             | submit\_bio                |
| 4K write latency            | ~7.72μs                 | ~25.30μs                  |
| 4K write IOPS (numjobs=32)  | ~1400kops               | ~210kops                  |
| Concurrency scaling         | Multi-tree per backend   | Single shared structure    |
| Tail latency stability      | High                     | Low                        |
| Crash consistency (backend) | Guaranteed               | Risk of loss without flush |
| Backend formatting needed   | No                       | Yes                        |
| Per-disk cache allocation   | Yes                      | No (shared pool)           |

## Use Cases

- High-throughput, low-latency I/O workloads
- Cloud storage with pmem-enabled hosts
- Databases using pmem as a caching layer
- Applications requiring deterministic latency

## Conclusion

`pcache` offers a modern caching layer for systems with persistent memory. With low latency, high concurrency, and predictable behavior, it is a strong complement to `bcache` in scenarios where pmem is available.

It does not replace `bcache`, but extends the Linux block caching ecosystem into the persistent memory domain with a lightweight, robust design.

