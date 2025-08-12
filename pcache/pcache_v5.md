# PCache v5

This document summarizes **dm-pcache** patch set v5.  It is based on v6.16 of
the Linux kernel and contains the latest updates and testing results for the
persistent cache target.

## Mail
> Hi Mikulas,
> This is V5 for dm-pcache, please consider merging.
>
> As you requested, I've squashed these patches into a single patch based on
> v6.16. If anything needs tweaking while merging, feel free to modify it
> directly. I can also send a rebased version on top of the linux-dm branch if
> you prefer.
>
> This patch has been tested extensively. Once it's merged into linux-dm I'll
> keep running ongoing tests against dm-6.18.

[Full cover letter on dm-devel](https://marc.info/?l=dm-devel&m=175498716132537)

## Code
https://github.com/DataTravelGuide/linux/tree/pcache_v5

## Changelog

### V5 from V4
- Fix `get_n_vecs()` bug when the buffer is not page‑aligned; use
  `bio_add_max_vecs()` instead.
- Use `kvcalloc()` for allocations larger than 32,768 bytes.
- For mempool allocations, try `GFP_NOWAIT` first and fall back to
  `GFP_NOIO`.

## Testing
The following tests were executed against dm-pcache v5:

1. **dtg-tests for pcache** – includes dmsetup operations, failslab,
   fault injection (`fail_make_request`) and backing device delay tests,
   plus xfstests (`generic/rw`) under various parameters.
2. **Striped pmem as cache device** – dtg-tests suite executed with a
   striped pmem configuration.
3. **KASAN and memleak modes** – all of the above tests run with KASAN
   and memory leak detection enabled.
4. **gcov coverage** – reported 100% function coverage and 90%+ line
   coverage; remaining uncovered code includes `BUG()` lines and
   difficult‑to‑reach error paths.  The `mempool_alloc` path with
   `GFP_NOIO` was exercised.

## Results
- [dtg-tests (virtio)](https://datatravelguide.github.io/dtg-blog/pcache/pcache_v5_result_virtio/results.html)
- [dtg-tests (striped pmem)](https://datatravelguide.github.io/dtg-blog/pcache/pcache_v5_result_ram/results.html)
- [KASAN & memleak tests](https://datatravelguide.github.io/dtg-blog/pcache/pcache_v5_result_kasan/results.html)
- [Coverage report](https://datatravelguide.github.io/dtg-blog/pcache/pcache_cov_v5/dm-pcache/index.html)

