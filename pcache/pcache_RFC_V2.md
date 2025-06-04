# PCache RFC V2

This document summarizes the design of **dm-pcache**, a device mapper target that turns a persistent memory (pmem) device into a write‑back cache for a slower block device.

## Key features
- Write‑back caching (current mode)
- 16&nbsp;MiB segments on the pmem cache device
- Optional CRC32 verification for cached data
- Crash‑safe metadata duplicated and protected with CRC and sequence numbers
- Multi‑tree indexing (per CPU backend) for high parallelism
- Pure DAX I/O path with no extra BIO round‑trips
- Log‑structured write‑back preserving backend crash consistency

## Status information
`dmsetup status <dev>` prints:
```
<sb_flags> <seg_total> <cache_segs> <segs_used> \
<gc_percent> <cache_flags> \
<key_head_seg>:<key_head_off> \
<dirty_tail_seg>:<dirty_tail_off> \
<key_tail_seg>:<key_tail_off>
```
Important fields:
- `seg_total` – number of pmem segments
- `cache_segs` – segments used for cache
- `segs_used` – currently allocated segments
- `gc_percent` – GC threshold (0‑90)
- `cache_flags` – bit 0: DATA_CRC, bit 1: INIT_DONE, bits 2‑5: cache mode

## Messages
Adjust the GC trigger:
```
dmsetup message <dev> 0 gc_percent <0-90>
```

## Operation overview
- The pmem space is divided into segments, with per‑CPU allocation heads
- Keys record ranges on the backing device and map them to pmem
- 128 keys form a key‑set; ksets are written sequentially and are crash safe
- Dirty keys are written back asynchronously; a FLUSH/FUA forces metadata commit
- Garbage collection reclaims segments once the usage exceeds `gc_percent`
- CRC32 protects cached data when enabled

## Failure handling
- Uncorrectable pmem errors abort initialization
- Cache full returns `-EBUSY` and requests are retried internally
- After a crash, key‑sets are replayed to rebuild in‑memory trees

## Limitations
- Only write‑back mode is available
- FIFO invalidation only (LRU/ARC planned)
- Table reload not yet supported
- Discard support planned

## Example workflow
1. Create the device with `dmsetup`
2. Format and mount a filesystem
3. Tune the GC threshold
4. Observe status with `dmsetup status`
5. Unmount and remove the device
