---
layout: default
---

# CBD （CXL Block Device）

CBD (CXL Block Device) provides two usage scenarios: single-host and multi-hosts.

(1) Single-host scenario, CBD can use a pmem device as a cache for block devices,
providing a caching mechanism specifically designed for persistent memory.

[more cbd cache detail](cbd_cache.md)
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

(2) Multi-hosts scenario, CBD also provides a cache while taking advantage of
shared memory features, allowing users to access block devices on other nodes across
different hosts.

As shared memory is supported in CXL3.0 spec, we can transfer data via CXL shared memory.
CBD use CXL shared memory to transfer data between node-1 and node-2.
```
+--------------------------------------------------------------------------------------------------------+
|                                           multi-hosts                                                  |
+--------------------------------------------------------------------------------------------------------+
|                                                                                                        |
|                                                                                                        |
| +-------------------------------+                               +------------------------------------+ |
| |          node-1               |                               |              node-2                | |
| +-------------------------------+                               +------------------------------------+ |
| |                               |                               |                                    | |
| |                       +-------+                               +---------+                          | |
| |                       | cbd0  |                               | backend0+------------------+       | |
| |                       +-------+                               +---------+                  |       | |
| |                       | pmem0 |                               | pmem0   |                  v       | |
| |               +-------+-------+                               +---------+----+     +---------------+ |
| |               |    cxl driver |                               | cxl driver   |     |  /dev/sda     | |
| +---------------+--------+------+                               +-----+--------+-----+---------------+ |
|                          |                                            |                                |
|                          |                                            |                                |
|                          |        CXL                         CXL     |                                |
|                          +----------------+               +-----------+                                |
|                                           |               |                                            |
|                                           |               |                                            |
|                                           |               |                                            |
|                 +-------------------------+---------------+--------------------------+                 |
|                 |                         +---------------+                          |                 |
|                 | shared memory device    |  cbd0 cache   |                          |                 |
|                 |                         +---------------+                          |                 |
|                 +--------------------------------------------------------------------+                 |
|                                                                                                        |
+--------------------------------------------------------------------------------------------------------+
```

<br>

## cbd Linux kernel modules

The CBD kernel module is part of the Linux kernel, maintained under the DataTravelGuide organization.

[cbd kernel module source tree](https://github.com/DataTravelGuide/linux)

It has currently sent an RFC version to the Linux community.

[CBD RFC](https://lore.kernel.org/lkml/20240422071606.52637-1-dongsheng.yang@easystack.cn/)

V1 patchset for CBD sent here:

[CBD V1](https://lore.kernel.org/all/20240709130343.858363-1-dongsheng.yang@linux.dev/)

<br>

## cbd userspace tool

The CBD user-space command is a subcommand in the ndctl project and is currently under development.

[cbd userspace tool](https://github.com/DataTravelGuide/ndctl)

<br>

## cbd test suites

cbd-tests is an automated test suite based on the Avocado test framework, specifically designed for CBD. 
It includes performance tests (fio), data read/write tests (xfstests), and other tests (continuously being added).

[cbd-tests](https://github.com/DataTravelGuide/cbd-tests)

<br>

### test results

The test reports for cbd-tests are automatically generated by Avocado.
They include information about the runtime environment, all test results, detailed test processes for each test case, and more.

- V1:
  - [test-result-v1](./test-results/test_result_v1/results.html)
  - [test-result-v1-crc](./test-results/test_result_v1_crc/results.html)

<br><br><br>

