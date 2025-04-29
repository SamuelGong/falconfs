# FalconFS

This is awesome!

[![Build](https://github.com/falcon-infra/falconfs/actions/workflows/build.yml/badge.svg)](https://github.com/falcon-infra/falconfs/actions/workflows/build.yml)
[![License](https://img.shields.io/badge/License-Mulan%20PSL%202-green)](LICENSE)

FalconFS is a high-performance distributed file system (DFS) optimized for AI workloads by addressing three key challenges:  

1. **Massive small files** – Its high-performance distributed metadata engine dramatically improves I/O throughput of handling massive small files (e.g., images), eliminating storage bottlenecks in AI data preprocessing and model training.

2. **High throughput requirement** – In tiered storage (i.e., DRAM, SSD and elastic object store), FalconFS can aggregates near-compute DRAM and SSDs to provide over TB/s high throughput for AI workloads (e.g., KV Cache Offloading, model training and data preprocessing).

3. **Large scale** - FalconFS can scale to thousands of NPUs due to its scale-out metadata engine and scale-up single metadata performance.

Through the above advantages, FalconFS delivers an ideal storage solution for modern AI pipelines.

## Documents

- [FalconFS Design](./docs/design.md)
- [FalconFS Cluster Test Setup Guide](./deploy/ansible/README.md)

## Performance

**Test Environment Configuration:**
- **CPU:** 2 x Intel Xeon 3.00GHz, 12 cores
- **Memory:** 16 x DDR4 2933 MHz 16GB
- **Storage:** 2 x NVMe SSD
- **Network:** 2 x 100GbE
- **OS:** Ubuntu 20.04 Server 64-bit

> **ℹ️ Note**  
> This experiment uses an optimized Linux fuse module. The relevant code will be open-sourced later.

We conduct the experiments in a cluster of 13 dual-socket machines, whose configuration is shown above. To better simulate large scale deployment in data centers, we have the following setups:
- First, to expand the test scale, we abstract each machine into two nodes, with each node bound to one socket, one SSD, and one NIC, scaling up the testbed to 26 nodes.
- Second, to simulate the resource ratio in real deployment, we reduce the server resources to 4 cores per node. So that we can:
  - generate sufficient load to stress the servers with a few client nodes.
  - correctly simulate the 4:1 ratio between CPU cores and NVMe SSDs in typical real deployments.
In the experiments below, we run 4 metadata nodes and 12 data nodes for each DFS instance and saturate them with 10 client nodes. All DFSs do not enable metadata or data replication.

**Compared Systems:**
- CephFS 12.2.13.
- JuiceFS 1.2.1, with TiKV 1.16.1 as the metadata engine and data store.
- Lustre 2.15.6.


<br>
<div style="text-align: center;">
    <font size="5">
        <b>Throughput of File data IO.</b>
    </font>
    <br>We evaluate the performance of accessing small files with different file sizes. As shown in following figures, Y-axis is the throughput normalized to that of FalconFS. Thanks to FalconFS's higher metadata performance, it outperforms other DFSs in small file access. For files no larger than 64 KB, FalconFS achieves 7.35--21.23x speedup over CephFS, 0.86--24.87x speedup over JuiceFS and 1.12--1.85x speedup over Lustre. For files whose size is larger than 256 KiB, the performance of FalconFS is bounded by the aggregated SSD bandwidth. 
</div>

![alt text](./docs/images/read-throughput.png)![alt text](./docs/images/write-throughput.png)
<br>

<div style="text-align: center;">
    <font size="5">
        <b>MLPerf ResNet-50 Training Storage Benchmark.</b>
    </font>
    <br> We simulate training ResNet-50 model on a dataset containing 10 million files, each file contains one 131 KB object, which is a typical scenario for deep learning model training in production. MLPerf has been modified to avoid merging small files into large ones, simulating real-world business scenarios while reducing the overhead associated with merge and copy operations. The FalconFS client utilizes an optimized FUSE module to minimize overhead, and the module will be open-sourced in the near future. Taking 90% accelerator utilization as the threshold, FalconFS supports up to 80 accelerators while Lustre can only support 32 accelerators on the experiment hardware.
</div>

![alt text](./docs/images/mlperf.png)

<br>

## Build

suppose at the `~/code` dir
``` bash
git clone https://github.com/falcon-infra/falconfs.git
cd falconfs
git submodule update --init --recursive # submodule update postresql
./patches/apply.sh
docker run -it --rm -v `pwd`:/root/code -w /root/code/falconfs ghcr.io/falcon-infra/falconfs-dev:0.1.0 /bin/zsh
./build.sh
ln -s /root/code/falconfs/falcon/build/compile_commands.json . # use for clangd
```

test

``` bash
./build.sh test
```

clean

``` bash
cd falconfs
./build.sh clean
```

incermental build and clean

``` bash
cd falconfs
./build.sh build pg # only build pg
./build.sh clean pg # only clean pg
./build.sh build falcon # only build falconfs
./build.sh clean falcon # only clean falconfs
./build.sh build falcon --debug # build falconfs with debug
```

## Debug

### build and start FalconFS in the docker

> **⚠️ Warning**  
> This only for debug mode, do not use no_root_check.patch in production!

no root check debug, suppose at the `~/code` dir
``` bash
docker run --privileged -d -it --name falcon-dev -v `pwd`:/root/code -w /root/code/falconfs ghcr.io/falcon-infra/falconfs-dev:0.1.0
docker exec -it --detach-keys="ctrl-z,z" falcon-dev /bin/zsh
git -C third_party/postgres apply ../../patches/no_root_check.patch
./build.sh clean
./build.sh build --debug
source deploy/falcon_env.sh
./deploy/falcon_start.sh
```

### debug falcon meta server

- first login to cn: `psql -d postgres -p $cnport`
- when in the pg cli
``` bash
select pg_backend_pid(); # to get pid, then use gdb to attach the pid
SELECT falcon_plain_mkdir('/test'); # to trigger mkdir meta operation
```

### run some test and stop

``` bash
./.github/workflows/smoke_test.sh /tmp/falcon_mnt
./deploy/falcon_stop.sh
```

## Copyright
Copyright (c) 2025 Huawei Technologies Co., Ltd.
