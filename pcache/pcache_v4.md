# PCache v4

This document summarizes **dm-pcache** patch set v4.  It describes the
current feature set, architecture and workflow of the persistent cache and
highlights changes made since earlier revisions.

## Mail
https://www.spinics.net/lists/dm-devel/msg63536.html

## Code
https://github.com/DataTravelGuide/linux/tree/pcache_v4

## Changelog

### V4 from V3
- Revert to using **mempool** for allocating `cache_key` and
  `backing_dev_req` objects.
- Introduce `backing_bvec_cache` and `backing_dev->bvec_pool` to provide
  bvecs for write‑back requests.
- Drop return-value checks for `bio_init_clone()` as no integrity flags are
  used and the call cannot fail.
- Remove return-value checks from `backing_dev_req_alloc()` and
  `cache_key_alloc()`.

### V3 from V2
- Rebase onto `dm-6.17`.
- Add the missing `bitfield.h` include.
- Move `kmem_cache` instances from per-device to per-module scope.
- Fix a memory leak spotted via failslab testing.
- Retry `pcache_request` in `defer_req()` when memory allocation fails.

### V2 from V1
- Add `req_alloc()` and `req_init()` helpers in `backing_dev.c` to decouple
  allocation from initialization.
- Introduce `pre_alloc_key` and `pre_alloc_req` in the walk context so keys
  and requests can be preallocated prior to tree walking.
- Use `mempool_alloc(..., GFP_NOIO)` for `cache_key` and `backing_dev_req`
  allocations.
- Coding-style updates.

### V1 from RFC-V2
- Switch to **crc32c** for data validation.
- Retry only when the cache is full; requests are queued on a `defer_list`
  waiting for invalidation.
- Redesign the table format for easier extensibility.
- Remove `__packed` annotations.
- Use `spin_lock_irq` in `req_complete_fn()` and avoid
  `spin_lock_irqsave()`.
- Fix a bug in `backing_dev_bio_end()` concerning `spin_lock_irqsave()`.
- Call `queue_work()` inside the spinlock.
- Introduce `inline_bvecs` in `backing_dev_req` and allocate other bvecs via
  `kmalloc_array()`.
- Compute `->off` with `dm_target_offset()` before use.

## Key features
- Write‑back caching (current mode)
- 16&nbsp;MiB segments on the pmem cache device
- Optional CRC32 verification for cached data
- Crash‑safe metadata duplicated and protected with CRC and sequence numbers
- Multi‑tree indexing (per CPU backend) for high parallelism
- Pure DAX I/O path with no extra BIO round‑trips
- Log‑structured write‑back preserving backend crash consistency

## Architecture overview
The implementation is composed of three layers:

1. **pmem access layer** – reads use `copy_mc_to_kernel()` so media errors are
   detected; writes go through `memcpy_flushcache()` to ensure durability on
   persistent memory.
2. **cache-logic layer** – manages 16&nbsp;MiB segments with log‑structured
   allocation, maintains multiple RB-tree indexes for parallelism, verifies data
   CRCs, handles background write‑back and garbage collection, and replays
   key-sets from `key_tail` after a crash.
3. **dm-pcache target integration** – exposes a table line
   ``pcache <pmem_dev> <origin_dev> writeback <true|false>`` and advertises
   support for `PREFLUSH`/`FUA`.  Discard and dynamic reload are not yet
   implemented.  Runtime GC control is available via
   ``dmsetup message <dev> 0 gc_percent <0-90>``.

## Status information
`dmsetup status <dev>` prints:
```text
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
```bash
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
```bash
# 1. create devices
pmem=/dev/pmem0
ssd=/dev/sdb

# 2. map a pcache device
dmsetup create pcache_sdb --table \
  "0 $(blockdev --getsz $ssd) pcache $pmem $ssd writeback true"

# 3. format and mount
mkfs.ext4 /dev/mapper/pcache_sdb
mount /dev/mapper/pcache_sdb /mnt

# 4. tune GC to 80%
dmsetup message pcache_sdb 0 gc_percent 80

# 5. monitor status
watch -n1 'dmsetup status pcache_sdb'

# 6. shutdown
umount /mnt
dmsetup remove pcache_sdb
```

## Test result
We used the `pcache` test suite from **dtg-tests** to validate the target in
various scenarios. The tests create pcache devices with different parameters,
verify data read and write correctness and run xfstests under each
configuration. Detailed results are available in the test reports:

- [Coverage report](./pcache_cov_v4/index.html)
- [Test result](./pcache_v4_result/results.html)

