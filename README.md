# NCCL Tests Setup and Execution Guide

NVIDIA Collective Communications Library (NCCL) provides high-performance communication primitives optimized for NVIDIA GPUs and multi-GPU systems. The `nccl-tests` repository provides benchmarks to measure communication performance within a node and across multiple nodes.

---

## References

* NCCL Tests: https://github.com/NVIDIA/nccl-tests
* NCCL Repository: https://github.com/NVIDIA/nccl
* NCCL Installation Guide: https://docs.nvidia.com/deeplearning/nccl/install-guide/index.html

---

## Prerequisites

### Hardware Requirements

* NVIDIA GPUs (A100, H100, L40S, etc.)
* NVIDIA Driver installed
* CUDA-compatible system

For multi-node testing:

* InfiniBand or RoCE network
* Passwordless SSH between nodes
* MPI installed on all nodes

---

# Method 1: NCCL Tests Using Docker (Single Node)

## Pull NVIDIA CUDA Container

```bash
docker pull nvcr.io/nvidia/cuda:12.4.1-cudnn-devel-ubuntu22.04
```

## Launch Container

```bash
docker run -it --rm \
  --gpus all \
  --shm-size=16g \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  nvcr.io/nvidia/cuda:12.4.1-cudnn-devel-ubuntu22.04 bash
```

---

## Install Dependencies

```bash
apt update

apt install -y \
  git \
  build-essential \
  cmake \
  gcc-12 \
  g++-12 \
  libnccl2 \
  libnccl-dev \
  rdma-core \
  infiniband-diags \
  ibverbs-utils \
  iproute2
```

---

## Clone NCCL Tests

```bash
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests
```

---

## Build NCCL Tests

### Generic Build

```bash
make MPI=0 CUDA_HOME=/usr/local/cuda
```

### Build for H100 (SM90)

```bash
make MPI=0 \
  CUDA_HOME=/usr/local/cuda \
  NVCC_GENCODE="-gencode=arch=compute_90,code=sm_90"
```

---

## Run Single-Node NCCL Benchmark

Example: Run on all 8 GPUs of a server.

```bash
./build/all_reduce_perf \
  -b 8M \
  -e 1G \
  -f 2 \
  -g 8
```

### Parameters

| Parameter | Description                        |
| --------- | ---------------------------------- |
| `-b`      | Starting message size              |
| `-e`      | Maximum message size               |
| `-f`      | Message size multiplication factor |
| `-g`      | Number of GPUs per process         |

---

# Method 2: NCCL Tests with MPI (Single Node and Multi-Node)

This method is required for benchmarking communication across multiple nodes.

---

## Install CUDA Toolkit

### Download CUDA

```bash
wget https://developer.download.nvidia.com/compute/cuda/12.4.0/local_installers/cuda_12.4.0_550.54.14_linux.run
```

### Install CUDA

```bash
sudo sh cuda_12.4.0_550.54.14_linux.run
```

### Configure Environment Variables

```bash
echo 'export PATH=/usr/local/cuda-12.4/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc

source ~/.bashrc
```

### Verify Installation

```bash
nvcc -V
```

---

## Install PyTorch (Optional)

```bash
pip install torch==2.6.0 \
  torchvision==0.21.0 \
  torchaudio==2.6.0 \
  --index-url https://download.pytorch.org/whl/cu124
```

---

## Install MPI

```bash
sudo apt update

sudo apt install -y \
  openmpi-bin \
  libopenmpi-dev
```

### Verify Installation

```bash
mpirun --version
mpicc --version
which mpirun
```

---

## Install NCCL

```bash
sudo apt install \
  libnccl2=2.20.5-1+cuda12.4 \
  libnccl-dev=2.20.5-1+cuda12.4
```

### Verify Installation

```bash
dpkg -l | grep nccl
ldconfig -p | grep nccl
find / -name "nccl.h" 2>/dev/null
```

---

## Clone NCCL Tests

```bash
cd /opt
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests
```

---

## Configure MPI Environment

```bash
export CPATH=/usr/lib/x86_64-linux-gnu/openmpi/include:$CPATH
export LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/openmpi/lib:$LIBRARY_PATH
```

---

## Build NCCL Tests with MPI Support

```bash
make -j$(nproc) \
  MPI=1 \
  CUDA_HOME=/usr/local/cuda-12.4 \
  NCCL_HOME=/usr
```

---

# Single-Node MPI Benchmark

Run one MPI process per GPU.

Example: 8 GPUs on a single server.

```bash
mpirun \
  -np 8 \
  --allow-run-as-root \
  ./build/all_reduce_perf \
  -b 8 \
  -e 8G \
  -f 2 \
  -g 1
```

### Parameters

| Parameter  | Description            |
| ---------- | ---------------------- |
| `-np 8`    | Launch 8 MPI processes |
| `-g 1`     | One GPU per process    |
| Total GPUs | 8                      |

---

# Multi-Node Benchmark

## Step 1: Configure Passwordless SSH

All nodes must be able to SSH into each other without passwords.

Example:

```bash
ssh node1
ssh node2
```

---

## Step 2: Create a Host File

Example `hosts.txt`:

```text
node1 slots=8
node2 slots=8
```

or

```text
192.168.1.101 slots=8
192.168.1.102 slots=8
```

---

## Step 3: Verify MPI Connectivity

```bash
mpirun \
  -np 2 \
  -hostfile hosts.txt \
  hostname
```

Expected output:

```text
node1
node2
```

---

## Run Multi-Node NCCL Benchmark

Example:

```bash
mpirun \
  -np 16 \
  -N 2 \
  -hostfile hosts.txt \
  ./build/all_reduce_perf \
  -b 8 \
  -e 8G \
  -f 2 \
  -g 1
```

### Parameters

| Parameter   | Description                 |
| ----------- | --------------------------- |
| `-np 16`    | Total MPI processes         |
| `-N 2`      | MPI processes per node      |
| `-hostfile` | List of participating nodes |
| `-g 1`      | One GPU per process         |

---

# Recommended NCCL Environment Variables

For InfiniBand or RoCE deployments:

```bash
export NCCL_DEBUG=INFO
export NCCL_IB_DISABLE=0
export NCCL_SOCKET_IFNAME=eth0
export NCCL_IB_GID_INDEX=3
export NCCL_NET_GDR_LEVEL=2
```

---

# Useful Diagnostic Commands

## GPU Topology

```bash
nvidia-smi topo -m
```

## Check GPU Availability

```bash
nvidia-smi
```

## Check InfiniBand Devices

```bash
ibstat
ibv_devices
ibdev2netdev
```

## Check MPI Installation

```bash
mpirun --version
mpicc --version
```

---

# Additional NCCL Benchmarks

### AllReduce

```bash
./build/all_reduce_perf -b 8M -e 8G -f 2 -g 8
```

### AllGather

```bash
./build/all_gather_perf -b 8M -e 8G -f 2 -g 8
```

### Broadcast

```bash
./build/broadcast_perf -b 8M -e 8G -f 2 -g 8
```

### ReduceScatter

```bash
./build/reduce_scatter_perf -b 8M -e 8G -f 2 -g 8
```

---

# Troubleshooting

## NCCL Libraries Not Found

```bash
ldconfig -p | grep nccl
dpkg -l | grep nccl
```

## MPI Compilation Errors

```bash
which mpicc
which mpirun
```

Ensure the following variables are set:

```bash
export CPATH=/usr/lib/x86_64-linux-gnu/openmpi/include:$CPATH
export LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/openmpi/lib:$LIBRARY_PATH
```

## InfiniBand Devices Not Detected

```bash
ibstat
ibv_devices
```

## Enable NCCL Debugging

```bash
export NCCL_DEBUG=INFO
```

Then rerun the benchmark to inspect transport selection and communication paths.
