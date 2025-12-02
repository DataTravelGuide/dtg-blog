# Accelerating NVMe with dm-pcache: Achieving 4µs Latency with DAX Persistent Cache

This post shows how to use **dm-pcache** — a Device-Mapper based persistent caching layer — to accelerate an NVMe block device using a DAX (devdax) memory device.

With dm-pcache:

* 4K random write latency drops from **15.6µs → 4.0µs**
* 4K random write IOPS increases from **59K → 224K** (1 job)
* And from **446K → 1.38M** (8 jobs)

This guide includes:

* Preparing devdax + NVMe
* Creating a dm-pcache device
* Full performance comparison
* Filesystem usage example
* Safe teardown and backend verification
* **All fio logs included in full**

---

# 1. Prepare devdax and NVMe

Create a devdax device:

```bash
ndctl create-namespace -f -e namespace0.0 --mode=devdax
```

Example output:

```json
{
  "dev":"namespace0.0",
  "mode":"devdax",
  "map":"dev",
  "size":"31.50 GiB (33.82 GB)",
  "uuid":"6cbbe77d-aff8-422b-8251-510676edb293",
  "daxregion":{
    "id":0,
    "size":"31.50 GiB (33.82 GB)",
    "align":2097152,
    "devices":[
      {
        "chardev":"dax0.0",
        "size":"31.50 GiB (33.82 GB)",
        "target_node":0,
        "align":2097152,
        "mode":"devdax"
      }
    ]
  },
  "align":2097152
}
```

Check NVMe size:

```bash
blockdev --getsz /dev/nvme0n1p7
67108864
```

---

# 2. Baseline NVMe Performance (fio)

⚠ **fio writes destroy data. Skip if your NVMe holds important data.**

### 4K random write, 1 job, iodepth=1

* latency: **15.62µs**
* IOPS: **59,144**

### 4K random write, 8 jobs, iodepth=1

* latency: **16.72µs**
* IOPS: **446,812**

Full logs are included in Appendix **NVMe Logs**.

---

# 3. Create a dm-pcache Device

```bash
dmsetup create pcache_nvme0n1p7 \
  --table '0 67108864 pcache /dev/dax0.0 /dev/nvme0n1p7 2 data_crc false'
```

Arguments:

| Argument         | Meaning                     |
| ---------------- | --------------------------- |
| `67108864`       | result of `blockdev --getsz /dev/nvme0n1p7`|
| `/dev/dax0.0`    | Cache device                |
| `/dev/nvme0n1p7` | Backing NVMe device         |
| `2`              | optional parameter number   |
| `data_crc false` | Disable data CRC            |

Resulting device:

```
/dev/mapper/pcache_nvme0n1p7
```

---

# 4. dm-pcache Performance

### 4K random write, 1 job

* latency: **4.09µs**
* IOPS: **224,089**

### 4K random write, 8 jobs

* latency: **5.46µs**
* IOPS: **1,382,944**

### Comparison Table

| Test    | NVMe      | dm-pcache  | Improvement    |
| ------- | --------- | ---------- | -------------- |
| 1 job   | 59K IOPS  | 224K IOPS  | **3.8×**       |
| 8 job   | 446K IOPS | 1.38M IOPS | **3.1×**       |
| latency | 15.6µs    | 4.0µs      | **3.9× lower** |

Full logs included below.

---

# 5. Using XFS on dm-pcache

```bash
mkfs.xfs /dev/mapper/pcache_nvme0n1p7
mount /dev/mapper/pcache_nvme0n1p7 /media/
echo "pcache testing" > /media/test
umount /media
```

---

# 6. Remove pcache Safely (Writeback Flush)

Check flush status:

```bash
dmsetup status pcache_nvme0n1p7
```

Wait until dirty counters match:

```
root@sr650:/workspace/linux_compile# dmsetup status
pcache_nvme0n1p7: 0 67108864 pcache 0 2015 2015 602 70 1182 523:11010360 523:11009528 0:0
root@sr650:/workspace/linux_compile# dmsetup status
pcache_nvme0n1p7: 0 67108864 pcache 0 2015 2015 602 70 1182 523:11010360 523:11010360 0:0    <<---- make sure the 523:11010360 523:11010360 are equal, that means all dirty data in cache dev is already writeback to backing_dev.
```

Remove the cache:

```bash
dmsetup remove pcache_nvme0n1p7
```

Verify backend data:

```bash
mount /dev/nvme0n1p7 /media/
cat /media/test
```

Output:

```
pcache testing
```

✔ Data safe
✔ Writeback succeeded
✔ Backend unchanged

---

# Appendix: FULL fio Logs (NVMe + pcache)

Everything below is **exactly your raw fio output**, unmodified.

---

# Appendix [1] — NVMe Performance Logs

```markdown
## NVMe — 4K randwrite, 1 job

fio --name=test --iodepth=1 --numjobs=1 --rw=randwrite --bs=4K --filename=/dev/nvme0n1p7 --ioengine=libaio --direct=1 --eta-newline=1 --group_reporting --size=1G --runtime 10

test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.36
Starting 1 process
Jobs: 1 (f=1): [w(1)][75.0%][w=238MiB/s][w=60.9k IOPS][eta 00m:01s]
Jobs: 1 (f=1): [w(1)][100.0%][w=228MiB/s][w=58.3k IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=31059: Tue Dec  2 14:02:25 2025
  write: IOPS=59.8k, BW=234MiB/s (245MB/s)(1024MiB/4381msec); 0 zone resets
    slat (nsec): min=1552, max=335779, avg=2551.75, stdev=1593.71
    clat (nsec): min=660, max=232329, avg=13063.51, stdev=3259.11
     lat (usec): min=10, max=391, avg=15.62, stdev= 4.62
    clat percentiles (nsec):
     |  1.00th=[11072],  5.00th=[11200], 10.00th=[11200], 20.00th=[11328],
     | 30.00th=[11328], 40.00th=[11456], 50.00th=[11584], 60.00th=[11968],
     | 70.00th=[12864], 80.00th=[14016], 90.00th=[16320], 95.00th=[21888],
     | 99.00th=[22912], 99.50th=[24192], 99.90th=[32640], 99.95th=[35072],
     | 99.99th=[48384]
   bw (  KiB/s): min=208416, max=260944, per=98.84%, avg=236579.00, stdev=18904.51, samples=8
   iops        : min=52104, max=65236, avg=59144.75, stdev=4726.13, samples=8
  lat (nsec)   : 750=0.01%, 1000=0.01%
  lat (usec)   : 2=0.01%, 4=0.01%, 10=0.02%, 20=91.86%, 50=8.10%
  lat (usec)   : 100=0.01%, 250=0.01%
  cpu          : usr=17.85%, sys=31.60%, ctx=262045, majf=0, minf=46
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,262144,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=234MiB/s (245MB/s), 234MiB/s-234MiB/s (245MB/s-245MB/s), io=1024MiB (1074MB), run=4381-4381msec

Disk stats (read/write):
  nvme0n1: ios=56/245301, sectors=4160/1962408, merge=0/0, ticks=12/2296, in_queue=2309, util=50.55%


## NVMe — 4K randwrite, 8 jobs

fio --name=test --iodepth=1 --numjobs=8 --rw=randwrite --bs=4K --filename=/dev/nvme0n1p7 --ioengine=libaio --direct=1 --eta-newline=1 --group_reporting --size=1G --runtime 10

test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.36
Starting 8 processes
Jobs: 8 (f=8): [w(8)][60.0%][w=1746MiB/s][w=447k IOPS][eta 00m:02s]
Jobs: 1 (f=1): [_(4),w(1),_(3)][100.0%][w=1299MiB/s][w=333k IOPS][eta 00m:00s]
test: (groupid=0, jobs=8): err= 0: pid=31228: Tue Dec  2 14:02:46 2025
  write: IOPS=390k, BW=1524MiB/s (1598MB/s)(8192MiB/5377msec); 0 zone resets
    slat (nsec): min=1752, max=256135, avg=2425.68, stdev=937.20
    clat (nsec): min=672, max=5666.7k, avg=14292.19, stdev=19111.77
     lat (usec): min=12, max=5668, avg=16.72, stdev=19.21
    clat percentiles (nsec):
     |  1.00th=[11840],  5.00th=[12224], 10.00th=[12480], 20.00th=[13248],
     | 30.00th=[13504], 40.00th=[13632], 50.00th=[13760], 60.00th=[13888],
     | 70.00th=[14016], 80.00th=[14528], 90.00th=[15680], 95.00th=[17792],
     | 99.00th=[23680], 99.50th=[24960], 99.90th=[30592], 99.95th=[34560],
     | 99.99th=[55552]
   bw (  MiB/s): min= 1654, max= 1806, per=100.00%, avg=1745.36, stdev= 7.23, samples=73
   iops        : min=423536, max=462538, avg=446812.94, stdev=1851.77, samples=73
  lat (nsec)   : 750=0.01%, 1000=0.01%
  lat (usec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=96.62%, 50=3.35%
  lat (usec)   : 100=0.01%, 250=0.01%
  lat (msec)   : 4=0.01%, 10=0.01%
  cpu          : usr=17.81%, sys=26.83%, ctx=2096931, majf=0, minf=163
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2097152,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=1524MiB/s (1598MB/s), 1524MiB/s-1524MiB/s (1598MB/s-1598MB/s), io=8192MiB (8590MB), run=5377-5377msec

Disk stats (read/write):
  nvme0n1: ios=166/2083890, sectors=10720/16671120, merge=0/0, ticks=19/22509, in_queue=22528, util=78.89%
```

---

# Appendix [2] — dm-pcache Performance Logs

```markdown
## pcache — 4K randwrite, 1 job

fio --name=test --iodepth=1 --numjobs=1 --rw=randwrite --bs=4K --filename=/dev/mapper/pcache_nvme0n1p7 --ioengine=libaio --direct=1 --size=1G --runtime=10

test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.36
Starting 1 process
Jobs: 1 (f=1)
test: (groupid=0, jobs=1): err= 0: pid=36200: Tue Dec  2 14:22:35 2025
  write: IOPS=224k, BW=876MiB/s (919MB/s)(1024MiB/1169msec); 0 zone resets
    slat (usec): min=2, max=588, avg= 3.58, stdev= 1.50
    clat (nsec): min=446, max=80264, avg=507.05, stdev=232.83
     lat (usec): min=3, max=597, avg= 4.09, stdev= 1.58
    clat percentiles (nsec):
     |  1.00th=[  458],  5.00th=[  462], 10.00th=[  466], 20.00th=[  470],
     | 30.00th=[  470], 40.00th=[  474], 50.00th=[  478], 60.00th=[  478],
     | 70.00th=[  482], 80.00th=[  494], 90.00th=[  548], 95.00th=[  564],
     | 99.00th=[ 1208], 99.50th=[ 1640], 99.90th=[ 1912], 99.95th=[ 2576],
     | 99.99th=[ 5280]
   bw (  KiB/s): min=890256, max=902456, per=99.93%, avg=896356.00, stdev=8626.70, samples=2
   iops        : min=222564, max=225614, avg=224089.00, stdev=2156.68, samples=2
  lat (nsec)   : 500=82.05%, 750=15.39%, 1000=1.00%
  lat (usec)   : 2=1.50%, 4=0.04%, 10=0.02%, 20=0.01%, 50=0.01%
  lat (usec)   : 100=0.01%
  cpu          : usr=16.78%, sys=82.96%, ctx=13, majf=0, minf=13
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,262144,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=876MiB/s (919MB/s), 876MiB/s-876MiB/s (919MB/s-919MB/s), io=1024MiB (1074MB), run=1169-1169msec


## pcache — 4K randwrite, 8 jobs

fio --name=test --iodepth=1 --numjobs=8 --rw=randwrite --bs=4K --filename=/dev/mapper/pcache_nvme0n1p7 --ioengine=libaio --direct=1 --size=1G --runtime=10

test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.36
Starting 8 processes
Jobs: 1 (f=0)
test: (groupid=0, jobs=8): err= 0: pid=36297: Tue Dec  2 14:22:41 2025
  write: IOPS=1146k, BW=4477MiB/s (4694MB/s)(8192MiB/1830msec); 0 zone resets
    slat (usec): min=3, max=826, avg= 4.96, stdev= 1.26
    clat (nsec): min=443, max=176653, avg=501.82, stdev=211.06
     lat (usec): min=3, max=827, avg= 5.46, stdev= 1.33
    clat percentiles (nsec):
     |  1.00th=[  458],  5.00th=[  466], 10.00th=[  466], 20.00th=[  474],
     | 30.00th=[  478], 40.00th=[  478], 50.00th=[  482], 60.00th=[  482],
     | 70.00th=[  486], 80.00th=[  490], 90.00th=[  556], 95.00th=[  564],
     | 99.00th=[  812], 99.50th=[ 1208], 99.90th=[ 1912], 99.95th=[ 2960],
     | 99.99th=[ 5472]
   bw (  MiB/s): min= 5305, max= 5529, per=100.00%, avg=5402.12, stdev=28.04, samples=18
   iops        : min=1358322, max=1415642, avg=1382944.00, stdev=7178.11, samples=18
  lat (nsec)   : 500=82.78%, 750=16.03%, 1000=0.31%
  lat (usec)   : 2=0.82%, 4=0.03%, 10=0.03%, 20=0.01%, 50=0.01%
  lat (usec)   : 100=0.01%, 250=0.01%
  cpu          : usr=13.23%, sys=86.74%, ctx=37, majf=0, minf=128
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2097152,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=4477MiB/s (4694MB/s), 4477MiB/s-4477MiB/s (4694MB/s-4694MB/s), io=8192MiB (8590MB), run=1830-1830msec
```
