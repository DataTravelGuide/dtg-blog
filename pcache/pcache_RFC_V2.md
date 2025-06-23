# PCache RFC V2

This document summarizes **dm-pcache** RFC v2.  The most notable change from
[RFC v1](https://lore.kernel.org/lkml/20250414014505.20477-1-dongsheng.yang@linux.dev/)
is that the cache has been ported to the **Device Mapper** framework and is now
exposed as a standard DM target.  The code lives under
`drivers/md/dm-pcache/` in the kernel tree and lets a persistent memory (pmem)
region act as a write-back cache in front of a slower block device.

## Mail
https://lore.kernel.org/lkml/20250605142306.1930831-1-dongsheng.yang@linux.dev/

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

We used the `pcache` test suite from **dtg-tests** to validate the target in various scenarios. The tests create pcache devices with different parameters, verify data read and write correctness and run xfstests under each configuration. Detailed results are available in the [test-result](./pcache_rfc_v2_result/results.html).
