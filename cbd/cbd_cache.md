---
layout: default
---

# CBD Cache

## 1 What is cbd cache
`cbd cache` is a **lightweight** solution that uses persistent memory as block device cache. 
It works similar with `bcache`, where `bcache` uses block devices as cache drives, but
`cbd cache` only supports persistent memory devices for caching. It accesses the cache device
through DAX and incorporates several design features tailored for persistent memory scenarios,
such as multi-cache tree structures and atomic insertion of cached data.

**Note**: `cbd cache` is not intended to replace your `bcache`. Instead, it offers an alternative
 solution specifically suited for scenarios where you want to use persistent memory devices
 as block device cache.
 
Another caching technique for accessing persistent memory using DAX is **dm-writeback**,
but it is designed for scenarios based on **device-mapper**. On the other hand, **cbd cache**
and **bcache** are caching solutions for block device scenarios. Therefore, I did not do
a comparative analysis between **cbd cache** and **dm-writeback**.

```
+-----------------------------------------------------------------+
|                         single-host                             |
+-----------------------------------------------------------------+
|                                                                 |
|                                                                 |
|                                                                 |
|                                                                 |
|                                                                 |
|                        +-----------+     +------------+         |
|                        | /dev/cbd0 |     | /dev/cbd1  |         |
|                        |           |     |            |         |
|  +---------------------|-----------|-----|------------|-------+ |
|  |                     |           |     |            |       | |
|  |      /dev/pmem0     | cbd0 cache|     | cbd1 cache |       | |
|  |                     |           |     |            |       | |
|  +---------------------|-----------|-----|------------|-------+ |
|                        |+---------+|     |+----------+|         |
|                        ||/dev/sda ||     || /dev/sdb ||         |
|                        |+---------+|     |+----------+|         |
|                        +-----------+     +------------+         |
+-----------------------------------------------------------------+
```
## 2 Features of cbd cache
### 2.1 light software overhead cache write (low latency)
For cache write, handling a write request typically involves the following steps: 
(1) Allocating cache space -> (2) Writing data to the cache -> (3) Recording cache index metadata -> (4) Returning the result.

In cache modules using block devices as the cache (e.g., bcache), the steps of (2) writing data to the cache and 
(3) recording cache index metadata are asynchronous. 

During step (2), `submit_bio` is issued to the cache block device, and after the `bi_end_io` callback completes,
a new process continues with step (3). This incurs significant overhead for persistent memory cache.

However, **cbd cache**, which is designed for persistent memory, does not require asynchronous operations.
It can directly proceed with steps (3) and (4) after completing the `memcpy` through DAX. 

This makes a significant difference for small IO. In the case of 4K random writes,
**cbd cache** achieves a latency of only 7.72μs (compared to 25.30μs for bcache in the same test, offering a 300% improvement).

At the same time, the **memcpy** in dax access reduces a amount of software overhead compared to **submit_bio**,
and this difference be observed in large IO scenarios.

Further comparative results for various scenarios are shown in the table below.

```bash
+------------+-------------------------+--------------------------+
| numjobs=1  |         randwrite       |       randread           |
| iodepth=1  +------------+------------+-------------+------------+
| (latency)  |  cbd cache |  bcache    |  cbd cache  |  bcache    |
+------------+------------+------------+-------------+------------+
|  bs=512    |    6.10us  |    23.08us |      4.82us |     5.57us |
+------------+------------+------------+-------------+------------+
|  bs=1K     |    6.35us  |    21.68us |      5.38us |     6.05us |
+------------+------------+------------+-------------+------------+
|  bs=4K     |    7.72us  |    25.30us |      6.06us |     6.00us |
+------------+------------+------------+-------------+------------+
|  bs=8K     |    8.92us  |    27.73us |      7.24us |     7.35us |
+------------+------------+------------+-------------+------------+
|  bs=16K    |   12.25us  |    34.04us |      9.06us |     9.10us |
+------------+------------+------------+-------------+------------+
|  bs=32K    |   16.77us  |    49.24us |     14.10us |    16.18us |
+------------+------------+------------+-------------+------------+
|  bs=64K    |   30.52us  |    63.72us |     30.69us |    30.38us |
+------------+------------+------------+-------------+------------+
|  bs=128K   |   51.66us  |   114.69us |     38.47us |    39.10us |
+------------+------------+------------+-------------+------------+
|  bs=256K   |  110.16us  |   204.41us |     79.64us |    99.98us |
+------------+------------+------------+-------------+------------+
|  bs=512K   |  205.52us  |   398.70us |    122.15us |   131.97us |
+------------+------------+------------+-------------+------------+
|  bs=1M     |  493.57us  |   623.31us |    233.48us |   246.56us |
+------------+------------+------------+-------------+------------+
```
### 2.2 multi-queue and multi cache tree (high iops)
For persistent memory, the hardware concurrency is very high. If an indexing tree is used to manage space indexing,
the indexing will become a bottleneck for concurrency.

cbd cache independently manages its own indexing tree for each backend. Meanwhile, the indexing tree for the
cache corresponding to each backend is divided into multiple RB trees based on the logical address space.
All IO operations will find the corresponding indexing tree based on their offset. This design increases
concurrency while ensuring that the depth of the indexing tree does not become too large.

From testing, in a scenario with 32 numjobs, cbd cache achieved nearly 1,400K IOPS for 4K random write (under the
same test scenario, the IOPS of bcache was around 210K, meaning CBD Cache provided an improvement of over 600%).

More detailed comparison results are as follows:
```
+------------+-------------------------+--------------------------+
|  bs=4K     |         randwrite       |       randread           |
| iodepth=1  +------------+------------+-------------+------------+
|  (iops)    |  cbd cache |  bcache    |  cbd cache  |  bcache    |
+------------+------------+------------+-------------+------------+
|  numjobs=1 |    93652   |    38055   |    154101   |     142292 |
+------------+------------+------------+-------------+------------+
|  numjobs=2 |   205255   |    79322   |    317143   |     221957 |
+------------+------------+------------+-------------+------------+
|  numjobs=4 |   430588   |   124439   |    635760   |     513443 |
+------------+------------+------------+-------------+------------+
|  numjobs=8 |   852865   |   160980   |   1226714   |     505911 |
+------------+------------+------------+-------------+------------+
|  numjobs=16|  1140952   |   226094   |   2058178   |     996146 |
+------------+------------+------------+-------------+------------+
|  numjobs=32|  1418989   |   214447   |   2892710   |    1361308 |
+------------+------------+------------+-------------+------------+
```
### 2.3 better performance stablility (less stdev)
CBD Cache, through a streamlined design, simplifies and makes the IO process more controllable,
which allows for stable performance output.

For example, in CBD Cache, the writeback does not need to walk through the indexing tree,
meaning that the writeback process will not suffer from increased IO latency due to conflict
in the indexing tree. In contrast, Bcache's writeback process needs to traverse the indexing
tree to find the dirty keys before performing the writeback.

From testing, under random write, CBD Cache achieves an average latency of 6.80µs, with a max
latency of 2794µs and a latency standard deviation of 36.45 (under the same test, Bcache has an
average latency of 24.28µs, but a max latency of 474,622µs and a standard deviation as high as 937.81.
This means that in terms of standard deviation, CBD Cache achieved approximately 30 times the improvement).

```
  write: IOPS=39.1k, BW=153MiB/s (160MB/s)(5120MiB/33479msec); 0 zone resets
    slat (usec): min=4, max=157364, avg=12.47, stdev=138.93
    clat (nsec): min=1168, max=474615k, avg=11808.80, stdev=927287.74
     lat (usec): min=11, max=474622, avg=24.28, stdev=937.81
    clat percentiles (nsec):
     |  1.00th=[   1256],  5.00th=[   1304], 10.00th=[   1320],
     | 20.00th=[   1400], 30.00th=[   1448], 40.00th=[   1672],
     | 50.00th=[   8640], 60.00th=[   9152], 70.00th=[   9664],
     | 80.00th=[  10048], 90.00th=[  11328], 95.00th=[  19072],
     | 99.00th=[  27776], 99.50th=[  36608], 99.90th=[ 173056],
     | 99.95th=[ 856064], 99.99th=[2039808]
   bw (  KiB/s): min=28032, max=214664, per=99.69%, avg=156122.03, stdev=51649.87, samples=66
   iops        : min= 7008, max=53666, avg=39030.53, stdev=12912.50, samples=66
  lat (usec)   : 2=41.55%, 4=4.59%, 10=32.70%, 20=16.37%, 50=4.45%
  lat (usec)   : 100=0.10%, 250=0.17%, 500=0.02%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.03%, 4=0.01%, 10=0.01%, 20=0.01%, 50=0.01%
  lat (msec)   : 100=0.01%, 250=0.01%, 500=0.01%
  cpu          : usr=11.93%, sys=38.61%, ctx=1311384, majf=0, minf=382
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,1310718,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=153MiB/s (160MB/s), 153MiB/s-153MiB/s (160MB/s-160MB/s), io=5120MiB (5369MB), run=33479-33479msec

Disk stats (read/write):
    bcache0: ios=0/1305444, sectors=0/10443552, merge=0/0, ticks=0/21789, in_queue=21789, util=65.13%, aggrios=0/0, aggsectors=0/0, aggrmerge=0/0, aggrticks=0/0, aggrin_queue=0, aggrutil=0.00%
  ram0: ios=0/0, sectors=0/0, merge=0/0, ticks=0/0, in_queue=0, util=0.00%
  pmem0: ios=0/0, sectors=0/0, merge=0/0, ticks=0/0, in_queue=0, util=0.00%
```

```
  write: IOPS=133k, BW=520MiB/s (545MB/s)(5120MiB/9848msec); 0 zone resets
    slat (usec): min=3, max=2786, avg= 5.84, stdev=36.41
    clat (nsec): min=852, max=132404, avg=959.09, stdev=436.60
     lat (usec): min=4, max=2794, avg= 6.80, stdev=36.45
    clat percentiles (nsec):
     |  1.00th=[  884],  5.00th=[  900], 10.00th=[  908], 20.00th=[  916],
     | 30.00th=[  924], 40.00th=[  924], 50.00th=[  932], 60.00th=[  940],
     | 70.00th=[  948], 80.00th=[  964], 90.00th=[ 1004], 95.00th=[ 1064],
     | 99.00th=[ 1192], 99.50th=[ 1432], 99.90th=[ 6688], 99.95th=[ 7712],
     | 99.99th=[12480]
   bw (  KiB/s): min=487088, max=552928, per=99.96%, avg=532154.95, stdev=18228.92, samples=19
   iops        : min=121772, max=138232, avg=133038.84, stdev=4557.32, samples=19
  lat (nsec)   : 1000=89.09%
  lat (usec)   : 2=10.76%, 4=0.03%, 10=0.09%, 20=0.03%, 50=0.01%
  lat (usec)   : 100=0.01%, 250=0.01%
  cpu          : usr=23.93%, sys=76.03%, ctx=61, majf=0, minf=16
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,1310720,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=520MiB/s (545MB/s), 520MiB/s-520MiB/s (545MB/s-545MB/s), io=5120MiB (5369MB), run=9848-9848msec

Disk stats (read/write):
  cbd0: ios=0/1280334, sectors=0/10242672, merge=0/0, ticks=0/0, in_queue=0, util=43.07%
```
### 2.4 no need of formating for your existing disk
As a lightweight block storage caching technology, `cbd cache` does not require storing metadata on backend disk.
This allows users to easily add caching to existing disks without the need for any formatting operations and data
migration. They can also easily stop using the `cbd cache` without complications, The backend disk can be used 
independently as a raw disk.

### 2.5 backend device is crash-consistency
The writeback mechanism of `cbd cache` strictly follows a log-structured approach when writeback data. 
Even if dirty cache data is overwritten by new data (e.g., the old data from 0-4K is A, and new data 
overwrites 0-4K with B), the old data A is writeback first, followed by writeback the new data B to overwrite 
on the backend disk. This ensures that the backend disk maintains crash consistency. In the event of a failure 
of the `pmem` device, the data on the backend disk remains usable, though crash consistency is maintained while 
losing the data in the cache. This feature is particularly useful in cloud storage for disaster recovery scenarios.

It is important to note that this approach may lead to cache space efficiency issues if there are many overwrite operations.
However, modern file systems, such as Btrfs and F2FS, take wear leveling of the disk into account, so they tend
to avoid writing repeatedly to the same area. This means that there will not be a large number of overwrite writes
for the disk. Additionally, modern databases, especially those using LSM engines, rarely perform overwrite operations.

Additionally, there is an entry on the TODO list regarding providing an option for users to choose whether to
maintain backend consistency. If there is strong requirement, we can certainly offer this parameter in the future,
allowing users to opt out of maintaining backend crash consistency. This would optimize the handling of cache overwrite
scenarios, which is technically feasible.

### 2.6 specified cache space for each disk
For each backend, when enabling caching, the `cache_size` parameter must be specified. This is different from `bcache`,
where all backing devices can dynamically share the cache space within a single cache device. This improves cache utilization
by achieving optimal efficiency through time-sharing. However, this can lead to an issue where cache behavior becomes unpredictable.
In enterprise applications, it's important to have a more precise understanding of the performance of each disk.
When multiple disks dynamically share the cache, the exact amount of cache each disk receives becomes uncertain. `cbd cache` assigns
a dedicated cache space for each disk, ensuring that the cache is exclusive and not affected by others, making the cache behavior more predictable.


<br><br><br>

