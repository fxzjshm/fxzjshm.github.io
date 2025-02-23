---
layout: post
title: Benchmark UCX
date: 2024-07-19
author: Fxzjshm
category: Linux
tags: [Linux]
---

# UCX bechmark
## env vars
Refer to their FAQ: <https://openucx.readthedocs.io/en/master/faq.html>

* debug:
```bash
export UCX_LOG_LEVEL=info
export UCX_PROTO_INFO=y
```

* select device:
```bash
export UCX_NET_DEVICES=rocep65s0:1  # or sth. like mlx5_0:1
```
get device info from `ucx_info -d` & `lstopo`

* select protocol (RoCE, TCP, etc.)
  * note: for GPU RDMA, must enable `rocm_copy`/`cuda_copy`, otherwise `ucx_perftest` cannot allocate GPU memory for testing
```bash
export UCX_TLS=rc_v  # or tcp, shm, etc.
```

## compile UCX
* tag `v1.17.0`

```bash
./autogen.sh
./contrib/configure-release --prefix=/opt/ucx --enable-mt --with-verbs --enable-devel-headers --enable-examples --enable-cma --with-cuda=/usr/local/cuda --with-rocm=/opt/rocm
```

## host memory: RoCEv2 w1 <-> w2 <-> w3 <-> w4 <-> w5 <-> w1: 10125 MB/s
* move network controller to appropriate PCIe position
* enable RoCEv2 (priority flow control, etc.) on switch
  * e.g. on Dell switch we used, see <https://www.dell.com/support/manuals/en-us/dell-emc-smartfabric-os10/smartfabric-os-user-guide-10-5-2-6/configure-roce-on-the-switch?guid=guid-ddfa4455-8f64-4014-8543-ceb6719c904d&lang=en-us>
```bash
/opt/ucx/bin/ucx_perftest -c 0
```
```bash
/opt/ucx/bin/ucx_perftest -t ucp_put_bw 172.21.1.11 -s 1073741824 -c 3 -w 3 -n 10000
```

<!-- more -->

If add another w1 <-> w2, it jitters much:

<details>

```
+--------------+--------------+------------------------------+---------------------+-----------------------+
|              |              |       overhead (usec)        |   bandwidth (MB/s)  |  message rate (msg/s) |
+--------------+--------------+----------+---------+---------+----------+----------+-----------+-----------+
|    Stage     | # iterations | 50.0%ile | average | overall |  average |  overall |  average  |  overall  |
+--------------+--------------+----------+---------+---------+----------+----------+-----------+-----------+
[thread 0]                34      0.272 56927.562 56927.562    17987.77   17987.77          18          18
[thread 0]                42      0.280 135449.260 71884.076     7560.03   14245.16           7          14
[thread 0]                43      0.286 2240514.994 122317.353      457.04    8371.67           0           8
[thread 0]                52      0.323 121289.227 122139.408     8442.63    8383.86           8           8
[thread 0]                59      0.594 299897.841 143229.392     3414.50    7149.37           3           7
[thread 0]                64     44.577 426787.615 165382.378     2399.32    6191.71           2           6
[thread 0]                68 104199.285 534186.721 187076.751     1916.93    5473.69           2           5
[thread 0]                73 105799.818 427048.206 203513.152     2397.86    5031.62           2           5
[thread 0]                82 107617.851 117423.773 194064.318     8720.55    5276.60           9           5
[thread 0]                84 108411.667 1110605.001 215886.715      922.02    4743.23           1           5
[thread 0]                89 109011.394 432309.008 228045.271     2368.68    4490.34           2           4
[thread 0]                93 109389.568 519582.987 240584.527     1970.81    4256.30           2           4
[thread 0]                99 109859.320 366978.685 248244.779     2790.35    4124.96           3           4
[thread 0]               103 110529.342 1050885.499 279415.293      974.42    3664.80           1           4
```

</details>

## GPU RDMA: pursai-9654 <-> gpu8
Notice, this may require PCIe switch.
For AMD GPU, seems no firmware lock; in contrary, requires professional GPU for NVIDIA.

### 1. ROCm Platform
#### 1. Check Firmware Setup
Enable PCIe resizable bar & tune MMIO address setup, refer to: <https://github.com/openucx/ucx/wiki/Build-and-run-ROCM-UCX-OpenMPI>

Bar address of GPUs should be < 44 bits for gfx900 / gfx906 devices, otherwise segmentation fault will still happen.

#### 2. run test
```bash
/opt/ucx/bin/ucx_perftest -c 0
```
```bash
/opt/ucx/bin/ucx_perftest -t ucp_get 10.0.1.2 -s 1073741824 -c 3 -w 3 -n 10000 -m rocm
```
gives 26 GB/s on 400 GbE (limited by PCIe 4 of GPU) w/o host memory activity.

<details>

```
+--------------+--------------+------------------------------+---------------------+-----------------------+
|              |              |        latency (usec)        |   bandwidth (MB/s)  |  message rate (msg/s) |
+--------------+--------------+----------+---------+---------+----------+----------+-----------+-----------+
|    Stage     | # iterations | 50.0%ile | average | overall |  average |  overall |  average  |  overall  |
+--------------+--------------+----------+---------+---------+----------+----------+-----------+-----------+
[1722588781.634052] [ps:655046:0]         libperf.c:2090 UCX  DIAG  UCT tests also copy one-byte value from host memory to rocm send memory, which may impact performance results
[1722588781.634062] [ps:655046:0]         libperf.c:2097 UCX  DIAG  UCT tests also copy one-byte value from rocm recv memory to host memory, which may impact performance results
[1722588781.659752] [ps:655046:0]     ucp_context.c:2190 UCX  INFO  Version 1.17.0 (loaded from /opt/ucx/lib/libucp.so.0)
[1722588784.566871] [ps:655046:0]          parser.c:2314 UCX  INFO  UCX_* env variables: UCX_NET_DEVICES=mlx5_1:1 UCX_PROTO_INFO=y UCX_LOG_LEVEL=info
[1722588785.740799] [ps:655046:0]      ucp_worker.c:1888 UCX  INFO    perftest inter-node cfg#2 rma(rc_mlx5/mlx5_1:1)
[1722588785.755380] [ps:655046:0]   +---------------------------+-------------------------------------------------------------+
[1722588785.755391] [ps:655046:0]   | perftest inter-node cfg#2 | remote memory write by ucp_put* from host memory to rocm    |
[1722588785.755395] [ps:655046:0]   +---------------------------+------------------------------------+------------------------+
[1722588785.755400] [ps:655046:0]   |                     0..2K | short                              | rc_mlx5/mlx5_1:1/path0 |
[1722588785.755402] [ps:655046:0]   |                 2049..inf | zero-copy                          | rc_mlx5/mlx5_1:1/path0 |
[1722588785.755405] [ps:655046:0]   +---------------------------+------------------------------------+------------------------+
[1722588785.755468] [ps:655046:0]   +---------------------------+------------------------------------------------------------------------------+
[1722588785.755471] [ps:655046:0]   | perftest inter-node cfg#2 | remote memory write by ucp_put*(fast-completion) from host memory to rocm    |
[1722588785.755473] [ps:655046:0]   +---------------------------+-----------------------------------------------------+------------------------+
[1722588785.755476] [ps:655046:0]   |                     0..2K | short                                               | rc_mlx5/mlx5_1:1/path0 |
[1722588785.755479] [ps:655046:0]   |                2049..8256 | copy-in                                             | rc_mlx5/mlx5_1:1/path0 |
[1722588785.755481] [ps:655046:0]   |                 8257..inf | zero-copy                                           | rc_mlx5/mlx5_1:1/path0 |
[1722588785.755508] [ps:655046:0]   +---------------------------+-----------------------------------------------------+------------------------+
[1722588785.755562] [ps:655046:0]   +---------------------------+--------------------------------------------------------------------+
[1722588785.755565] [ps:655046:0]   | perftest inter-node cfg#2 | remote memory write by ucp_put*(multi) from host memory to rocm    |
[1722588785.755567] [ps:655046:0]   +---------------------------+-------------------------------------------+------------------------+
[1722588785.755570] [ps:655046:0]   |                    0..587 | short                                     | rc_mlx5/mlx5_1:1/path0 |
[1722588785.755573] [ps:655046:0]   |                  588..inf | zero-copy                                 | rc_mlx5/mlx5_1:1/path0 |
[1722588785.755576] [ps:655046:0]   +---------------------------+-------------------------------------------+------------------------+
[1722588785.755809] [ps:655046:0]      ucp_worker.c:1888 UCX  INFO    perftest self cfg#3 rma(self/memory rc_mlx5/mlx5_1:1)
[1722588785.770022] [ps:655046:0]   +---------------------+-------------------------------------------------------------+
[1722588785.770029] [ps:655046:0]   | perftest self cfg#3 | remote memory write by ucp_put* from host memory to rocm    |
[1722588785.770033] [ps:655046:0]   +---------------------+------------------------------------+------------------------+
[1722588785.770035] [ps:655046:0]   |               0..2K | short                              | rc_mlx5/mlx5_1:1/path0 |
[1722588785.770038] [ps:655046:0]   |           2049..inf | zero-copy                          | rc_mlx5/mlx5_1:1/path0 |
[1722588785.770040] [ps:655046:0]   +---------------------+------------------------------------+------------------------+
[1722588785.770104] [ps:655046:0]   +---------------------+------------------------------------------------------------------------------+
[1722588785.770107] [ps:655046:0]   | perftest self cfg#3 | remote memory write by ucp_put*(fast-completion) from host memory to rocm    |
[1722588785.770109] [ps:655046:0]   +---------------------+-----------------------------------------------------+------------------------+
[1722588785.770112] [ps:655046:0]   |               0..2K | short                                               | rc_mlx5/mlx5_1:1/path0 |
[1722588785.770115] [ps:655046:0]   |          2049..8256 | copy-in                                             | rc_mlx5/mlx5_1:1/path0 |
[1722588785.770118] [ps:655046:0]   |           8257..inf | zero-copy                                           | rc_mlx5/mlx5_1:1/path0 |
[1722588785.770121] [ps:655046:0]   +---------------------+-----------------------------------------------------+------------------------+
[1722588785.770172] [ps:655046:0]   +---------------------+--------------------------------------------------------------------+
[1722588785.770175] [ps:655046:0]   | perftest self cfg#3 | remote memory write by ucp_put*(multi) from host memory to rocm    |
[1722588785.770177] [ps:655046:0]   +---------------------+-------------------------------------------+------------------------+
[1722588785.770180] [ps:655046:0]   |              0..587 | short                                     | rc_mlx5/mlx5_1:1/path0 |
[1722588785.770182] [ps:655046:0]   |            588..inf | zero-copy                                 | rc_mlx5/mlx5_1:1/path0 |
[1722588785.770184] [ps:655046:0]   +---------------------+-------------------------------------------+------------------------+
[1722588785.780923] [ps:655046:0]   +---------------------------+-------------------------------------------------------------------+
[1722588785.780930] [ps:655046:0]   | perftest inter-node cfg#2 | remote memory read by ucp_get*(multi) into rocm/GPU0 from rocm    |
[1722588785.780933] [ps:655046:0]   +---------------------------+------------------------------------------+------------------------+
[1722588785.780936] [ps:655046:0]   |                         0 | copy-out                                 | rc_mlx5/mlx5_1:1/path0 |
[1722588785.780938] [ps:655046:0]   |                     1..64 | software emulation                       | rc_mlx5/mlx5_1:1/path0 |
[1722588785.780941] [ps:655046:0]   |                   65..inf | zero-copy                                | rc_mlx5/mlx5_1:1/path0 |
[1722588785.780943] [ps:655046:0]   +---------------------------+------------------------------------------+------------------------+
[thread 0]                28  38116.175 36737.178 36737.178    27873.67   27873.67          27          27
[thread 0]                55  38115.524 38211.999 37461.181    26797.87   27334.96          26          27
[thread 0]                82  38115.123 38095.112 37669.915    26880.09   27183.50          26          27
[thread 0]               109  38115.173 38212.370 37804.284    26797.61   27086.88          26          26
[thread 0]               136  38115.083 38094.891 37861.978    26880.24   27045.60          26          26
[thread 0]               163  38115.173 38195.663 37917.251    26809.33   27006.18          26          26
[thread 0]               190  38115.083 38221.333 37960.463    26791.32   26975.44          26          26
```

</details>

#### 3. check memory usage
```bash
sudo pcm-memory
```

<details>

```
|---------------------------------------||---------------------------------------|
|--             Socket  0             --||--             Socket  1             --|
|---------------------------------------||---------------------------------------|
|--     Memory Channel Monitoring     --||--     Memory Channel Monitoring     --|
|---------------------------------------||---------------------------------------|
|-- Mem Ch  0: Reads (MB/s):     5.94 --||-- Mem Ch  0: Reads (MB/s):     6.58 --|
|--            Writes(MB/s):     5.91 --||--            Writes(MB/s):     6.49 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  2: Reads (MB/s):     5.96 --||-- Mem Ch  2: Reads (MB/s):     6.15 --|
|--            Writes(MB/s):     5.91 --||--            Writes(MB/s):     6.07 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  3: Reads (MB/s):     5.96 --||-- Mem Ch  3: Reads (MB/s):     6.83 --|
|--            Writes(MB/s):     5.90 --||--            Writes(MB/s):     6.77 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  5: Reads (MB/s):     5.98 --||-- Mem Ch  5: Reads (MB/s):     6.14 --|
|--            Writes(MB/s):     5.92 --||--            Writes(MB/s):     6.08 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  6: Reads (MB/s):     5.98 --||-- Mem Ch  6: Reads (MB/s):     6.58 --|
|--            Writes(MB/s):     5.91 --||--            Writes(MB/s):     6.49 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  8: Reads (MB/s):     5.98 --||-- Mem Ch  8: Reads (MB/s):     6.38 --|
|--            Writes(MB/s):     5.91 --||--            Writes(MB/s):     6.34 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  9: Reads (MB/s):     5.98 --||-- Mem Ch  9: Reads (MB/s):     6.19 --|
|--            Writes(MB/s):     5.93 --||--            Writes(MB/s):     6.08 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch 11: Reads (MB/s):     5.96 --||-- Mem Ch 11: Reads (MB/s):     6.28 --|
|--            Writes(MB/s):     5.92 --||--            Writes(MB/s):     6.16 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- NODE 0 Mem Read (MB/s) :    47.74 --||-- NODE 1 Mem Read (MB/s) :    51.14 --|
|-- NODE 0 Mem Write(MB/s) :    47.32 --||-- NODE 1 Mem Write(MB/s) :    50.49 --|
|-- NODE 0 PMM Read (MB/s):      0.00 --||-- NODE 1 PMM Read (MB/s):      0.00 --|
|-- NODE 0 PMM Write(MB/s):      0.00 --||-- NODE 1 PMM Write(MB/s):      0.00 --|
|-- NODE 0 Memory (MB/s):       95.07 --||-- NODE 1 Memory (MB/s):      101.62 --|
|---------------------------------------||---------------------------------------|
|---------------------------------------||---------------------------------------|
|--            System DRAM Read Throughput(MB/s):         98.88                --|
|--           System DRAM Write Throughput(MB/s):         97.81                --|
|--             System PMM Read Throughput(MB/s):          0.00                --|
|--            System PMM Write Throughput(MB/s):          0.00                --|
|--                 System Read Throughput(MB/s):         98.88                --|
|--                System Write Throughput(MB/s):         97.81                --|
|--               System Memory Throughput(MB/s):        196.69                --|
|---------------------------------------||---------------------------------------|
```

</details>

### 2. CUDA Platform
#### 1. Check if `nvidia_peermem` is correctly loaded
```bash
sudo modprobe nvidia_peermem
```
sometimes it gives symbol mismatch in `dmesg`:
```
[31832.683273] nvidia_peermem: disagrees about version of symbol ib_register_peer_memory_client
[31832.683276] nvidia_peermem: Unknown symbol ib_register_peer_memory_client (err -22)
```
this is possibly caused by installing Mellanox OFED driver after installing NVIDIA driver, and NVIDIA driver is using `ib_*` in kernel. Rebuild NVIDIA driver:
```bash
sudo dkms status
sudo dkms unbuild nvidia/...
sudo dkms build nvidia/...
```
Why always so many weird things for NVIDIA ??

#### 2. run test
```bash
/opt/ucx/bin/ucx_perftest -c 0
```
```bash
/opt/ucx/bin/ucx_perftest -t ucp_put_bw 10.0.1.2 -s 1073741824 -c 3 -w 3 -n 10000 -m cuda
```
gives 11.5 GB/s on 100 GbE w/o host memory activity.

<details>

```
+--------------+--------------+------------------------------+---------------------+-----------------------+
|              |              |       overhead (usec)        |   bandwidth (MB/s)  |  message rate (msg/s) |
+--------------+--------------+----------+---------+---------+----------+----------+-----------+-----------+
|    Stage     | # iterations | 50.0%ile | average | overall |  average |  overall |  average  |  overall  |
+--------------+--------------+----------+---------+---------+----------+----------+-----------+-----------+
[1721319196.742086] [ps:143851:0]         libperf.c:2090 UCX  DIAG  UCT tests also copy one-byte value from host memory to cuda send memory, which may impact performance results
[1721319196.742098] [ps:143851:0]         libperf.c:2097 UCX  DIAG  UCT tests also copy one-byte value from cuda recv memory to host memory, which may impact performance results
[1721319196.991262] [ps:143851:0]     ucp_context.c:2190 UCX  INFO  Version 1.17.0 (loaded from /opt/ucx/lib/libucp.so.0)
[1721319197.230251] [ps:143851:0]          parser.c:2314 UCX  INFO  UCX_* env variables: UCX_NET_DEVICES=mlx5_0:1 UCX_PROTO_INFO=y UCX_LOG_LEVEL=info
[1721319202.977060] [ps:143851:0]      ucp_worker.c:1888 UCX  INFO    perftest inter-node cfg#1 rma(rc_mlx5/mlx5_0:1)
[1721319202.993450] [ps:143851:0]   +---------------------------+-------------------------------------------------------------+
[1721319202.993461] [ps:143851:0]   | perftest inter-node cfg#1 | remote memory write by ucp_put* from host memory to cuda    |
[1721319202.993465] [ps:143851:0]   +---------------------------+------------------------------------------+------------------+
[1721319202.993468] [ps:143851:0]   |                     0..2K | short                                    | rc_mlx5/mlx5_0:1 |
[1721319202.993470] [ps:143851:0]   |                 2049..inf | zero-copy                                | rc_mlx5/mlx5_0:1 |
[1721319202.993473] [ps:143851:0]   +---------------------------+------------------------------------------+------------------+
[1721319202.993537] [ps:143851:0]   +---------------------------+------------------------------------------------------------------------------+
[1721319202.993541] [ps:143851:0]   | perftest inter-node cfg#1 | remote memory write by ucp_put*(fast-completion) from host memory to cuda    |
[1721319202.993549] [ps:143851:0]   +---------------------------+-----------------------------------------------------------+------------------+
[1721319202.993551] [ps:143851:0]   |                     0..2K | short                                                     | rc_mlx5/mlx5_0:1 |
[1721319202.993553] [ps:143851:0]   |                2049..8256 | copy-in                                                   | rc_mlx5/mlx5_0:1 |
[1721319202.993555] [ps:143851:0]   |                 8257..inf | zero-copy                                                 | rc_mlx5/mlx5_0:1 |
[1721319202.993558] [ps:143851:0]   +---------------------------+-----------------------------------------------------------+------------------+
[1721319202.993611] [ps:143851:0]   +---------------------------+--------------------------------------------------------------------+
[1721319202.993615] [ps:143851:0]   | perftest inter-node cfg#1 | remote memory write by ucp_put*(multi) from host memory to cuda    |
[1721319202.993617] [ps:143851:0]   +---------------------------+-------------------------------------------------+------------------+
[1721319202.993619] [ps:143851:0]   |                    0..587 | short                                           | rc_mlx5/mlx5_0:1 |
[1721319202.993622] [ps:143851:0]   |                  588..inf | zero-copy                                       | rc_mlx5/mlx5_0:1 |
[1721319202.993624] [ps:143851:0]   +---------------------------+-------------------------------------------------+------------------+
[1721319202.993869] [ps:143851:0]      ucp_worker.c:1888 UCX  INFO    perftest self cfg#2 rma(self/memory rc_mlx5/mlx5_0:1)
[1721319203.007122] [ps:143851:0]   +---------------------+-------------------------------------------------------------+
[1721319203.007130] [ps:143851:0]   | perftest self cfg#2 | remote memory write by ucp_put* from host memory to cuda    |
[1721319203.007133] [ps:143851:0]   +---------------------+------------------------------------------+------------------+
[1721319203.007135] [ps:143851:0]   |               0..2K | short                                    | rc_mlx5/mlx5_0:1 |
[1721319203.007137] [ps:143851:0]   |           2049..inf | zero-copy                                | rc_mlx5/mlx5_0:1 |
[1721319203.007139] [ps:143851:0]   +---------------------+------------------------------------------+------------------+
[1721319203.007207] [ps:143851:0]   +---------------------+------------------------------------------------------------------------------+
[1721319203.007211] [ps:143851:0]   | perftest self cfg#2 | remote memory write by ucp_put*(fast-completion) from host memory to cuda    |
[1721319203.007213] [ps:143851:0]   +---------------------+-----------------------------------------------------------+------------------+
[1721319203.007216] [ps:143851:0]   |               0..2K | short                                                     | rc_mlx5/mlx5_0:1 |
[1721319203.007219] [ps:143851:0]   |          2049..8256 | copy-in                                                   | rc_mlx5/mlx5_0:1 |
[1721319203.007222] [ps:143851:0]   |           8257..inf | zero-copy                                                 | rc_mlx5/mlx5_0:1 |
[1721319203.007224] [ps:143851:0]   +---------------------+-----------------------------------------------------------+------------------+
[1721319203.007280] [ps:143851:0]   +---------------------+--------------------------------------------------------------------+
[1721319203.007284] [ps:143851:0]   | perftest self cfg#2 | remote memory write by ucp_put*(multi) from host memory to cuda    |
[1721319203.007286] [ps:143851:0]   +---------------------+-------------------------------------------------+------------------+
[1721319203.007289] [ps:143851:0]   |              0..587 | short                                           | rc_mlx5/mlx5_0:1 |
[1721319203.007291] [ps:143851:0]   |            588..inf | zero-copy                                       | rc_mlx5/mlx5_0:1 |
[1721319203.007294] [ps:143851:0]   +---------------------+-------------------------------------------------+------------------+
[1721319203.015890] [ps:143851:0]   +---------------------------+------------------------------------------------------------------+
[1721319203.015901] [ps:143851:0]   | perftest inter-node cfg#1 | remote memory write by ucp_put*(multi) from cuda/GPU0 to cuda    |
[1721319203.015903] [ps:143851:0]   +---------------------------+-----------------------------------------------+------------------+
[1721319203.015905] [ps:143851:0]   |                         0 | short                                         | rc_mlx5/mlx5_0:1 |
[1721319203.015907] [ps:143851:0]   |                    1..inf | zero-copy                                     | rc_mlx5/mlx5_0:1 |
[1721319203.015909] [ps:143851:0]   +---------------------------+-----------------------------------------------+------------------+
[thread 0]                44      0.160 24257.161 24257.161    42214.34   42214.34          41          41
[thread 0]                56      0.180 88938.236 38117.392    11513.61   26864.38          11          26
[thread 0]                68  88926.031 88943.839 47086.765    11512.88   21747.09          11          21
[thread 0]                80  88934.854 88940.243 53364.787    11513.35   19188.68          11          19
[thread 0]                92  88935.065 88939.766 58005.001    11513.41   17653.65          11          17
[thread 0]               104  88935.636 88943.819 61574.865    11512.89   16630.16          11          16
[thread 0]               116  88936.517 88943.581 64406.111    11512.92   15899.11          11          16
[thread 0]               128  88936.387 88938.336 66706.007    11513.60   15350.94          11          15
[thread 0]               140  88939.872 88942.011 68611.951    11513.12   14924.51          11          15
[thread 0]               152  88937.448 88937.402 70216.591    11513.72   14583.45          11          14
[thread 0]               164  88936.387 88935.852 71586.293    11513.92   14304.41          11          14
```

</details>

#### 3. check memory bandwidth used
for Intel CPUs,
```bash
sudo pcm-memory
```
gives something like

<details>

```
|---------------------------------------||---------------------------------------|
|--             Socket  0             --||--             Socket  1             --|
|---------------------------------------||---------------------------------------|
|--     Memory Channel Monitoring     --||--     Memory Channel Monitoring     --|
|---------------------------------------||---------------------------------------|
|-- Mem Ch  0: Reads (MB/s):    20.07 --||-- Mem Ch  0: Reads (MB/s):    12.15 --|
|--            Writes(MB/s):    15.17 --||--            Writes(MB/s):     9.91 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  2: Reads (MB/s):    20.06 --||-- Mem Ch  2: Reads (MB/s):    12.24 --|
|--            Writes(MB/s):    15.23 --||--            Writes(MB/s):     9.96 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  3: Reads (MB/s):    20.00 --||-- Mem Ch  3: Reads (MB/s):    12.41 --|
|--            Writes(MB/s):    15.07 --||--            Writes(MB/s):    10.19 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  5: Reads (MB/s):    20.11 --||-- Mem Ch  5: Reads (MB/s):    12.62 --|
|--            Writes(MB/s):    15.16 --||--            Writes(MB/s):    10.41 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  6: Reads (MB/s):    20.15 --||-- Mem Ch  6: Reads (MB/s):    12.55 --|
|--            Writes(MB/s):    15.19 --||--            Writes(MB/s):    10.35 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  8: Reads (MB/s):    20.19 --||-- Mem Ch  8: Reads (MB/s):    12.52 --|
|--            Writes(MB/s):    15.21 --||--            Writes(MB/s):    10.23 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch  9: Reads (MB/s):    20.17 --||-- Mem Ch  9: Reads (MB/s):    12.20 --|
|--            Writes(MB/s):    15.26 --||--            Writes(MB/s):     9.87 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- Mem Ch 11: Reads (MB/s):    20.12 --||-- Mem Ch 11: Reads (MB/s):    12.28 --|
|--            Writes(MB/s):    15.20 --||--            Writes(MB/s):     9.99 --|
|--      PMM Reads(MB/s)   :     0.00 --||--      PMM Reads(MB/s)   :     0.00 --|
|--      PMM Writes(MB/s)  :     0.00 --||--      PMM Writes(MB/s)  :     0.00 --|
|-- NODE 0 Mem Read (MB/s) :   160.87 --||-- NODE 1 Mem Read (MB/s) :    98.97 --|
|-- NODE 0 Mem Write(MB/s) :   121.50 --||-- NODE 1 Mem Write(MB/s) :    80.91 --|
|-- NODE 0 PMM Read (MB/s):      0.00 --||-- NODE 1 PMM Read (MB/s):      0.00 --|
|-- NODE 0 PMM Write(MB/s):      0.00 --||-- NODE 1 PMM Write(MB/s):      0.00 --|
|-- NODE 0 Memory (MB/s):      282.37 --||-- NODE 1 Memory (MB/s):      179.88 --|
|---------------------------------------||---------------------------------------|
|---------------------------------------||---------------------------------------|
|--            System DRAM Read Throughput(MB/s):        259.85                --|
|--           System DRAM Write Throughput(MB/s):        202.41                --|
|--             System PMM Read Throughput(MB/s):          0.00                --|
|--            System PMM Write Throughput(MB/s):          0.00                --|
|--                 System Read Throughput(MB/s):        259.85                --|
|--                System Write Throughput(MB/s):        202.41                --|
|--               System Memory Throughput(MB/s):        462.25                --|
|---------------------------------------||---------------------------------------|
```

</details>
