# NCCL Setup and Validation Guide (Kubernetes + NVIDIA Network Operator + MPI Operator)

This document describes the complete setup required to run NCCL AllReduce validation jobs on Kubernetes using InfiniBand/RDMA networking.

---

# 1. Install NVIDIA/Mellanox OFED Driver

## Configure RDMA Modules to Persist Across Reboot

```bash
cat > /etc/modules-load.d/mlx5.conf << 'EOF'
mlx5_core
mlx5_ib
ib_uverbs
rdma_ucm
ib_umad
nvidia_peermem
EOF
```

---

## Prepare Node for OFED Installation

### Cordoning and Draining Kubernetes Node

```bash
kubectl cordon <node>

kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

---

### Stop Storage and RDMA Services

```bash
unmount <wekavolume>

systemctl stop ibacm srp_daemon

weka local stop -f
```

---

### Verify No Process is Using InfiniBand Devices

```bash
lsof /dev/infiniband/uverbs* 2>/dev/null
```

Expected output:

```text
EMPTY
```

---

### Unload Existing RDMA Driver Stack

```bash
modprobe -r ib_srp
modprobe -r ib_iser
modprobe -r rpcrdma
modprobe -r rdma_ucm
modprobe -r ib_umad
modprobe -r ib_cm
modprobe -r iw_cm
modprobe -r rdma_cm
//modprobe -r ib_uverbs
//modprobe -r mlx5_ib
//modprobe -r mlx5_core
```

---

### Verify Drivers are Fully Unloaded

```bash
lsmod | grep -E "mlx5|ib_core"
```

Expected:

```text
No output
```

---

## Install MOFED on Host

Download and install MOFED:

```bash
wget https://content.mellanox.com/ofed/MLNX_OFED-24.10-0.7.0.0/MLNX_OFED_LINUX-24.10-0.7.0.0-ubuntu22.04-x86_64.tgz

tar xzf MLNX_OFED_LINUX-24.10-0.7.0.0-ubuntu22.04-x86_64.tgz

cd MLNX_OFED_LINUX-24.10-0.7.0.0-ubuntu22.04-x86_64

./mlnxofedinstall --without-fw-update --force
```

---

### Remove Intel RDMA Driver (If Present)

```bash
modprobe -r irdma
```

---

### Restart OpenIB Stack

```bash
/etc/init.d/openibd start
```

---

## Reboot Requirement

Perform a **power cycle** of the node after installation.

---

## Post-Reboot Validation

### Verify Required Modules Loaded

```bash
lsmod | grep -E "mlx5_core|mlx5_ib|ib_uverbs|ib_core"
```

---

### Verify RDMA Interfaces

```bash
rdma link | wc -l
```

Expected:

```text
8
```

---

### Verify Bond Interface

```bash
cat /proc/net/bonding/bond0 | grep "MII Status"
```

Expected:

```text
MII Status: up
```

---

### Verify MOFED Driver

```bash
modinfo mlx5_ib | grep filename
```

This confirms whether the node is using:

* Inbox driver
* MOFED driver

---

### Verify OpenIB Service

```bash
/etc/init.d/openibd status
```

---

### Load GPUDirect RDMA Peer Memory Module

```bash
sudo modprobe nvidia_peermem
```

Verify:

```bash
lsmod | grep nvidia_peermem
```

Expected:

```text
nvidia_peermem
```

---

# 2. Install NVIDIA Network Operator

## Helm Values

Create `values.yaml`

```yaml
nfd:
  enabled: true
```

---

## Install Network Operator

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install network-operator nvidia/network-operator \
  -n nvidia-network-operator \
  --create-namespace \
  -f values.yaml
```

---

## Create NicClusterPolicy

Create `nicclusterpolicy.yaml`

```yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  sriovDevicePlugin:
    image: sriov-network-device-plugin
    repository: nvcr.io/nvidia/mellanox
    version: network-operator-v25.7.0
    imagePullSecrets: []
    config: |
      {
        "resourceList": [
          {
            "resourcePrefix": "nvidia.com",
            "resourceName": "hostdev",
            "selectors": {
              "vendors": ["15b3"],
              "isRdma": true
            }
          }
        ]
      }

  secondaryNetwork:
    cniPlugins:
      image: plugins
      repository: nvcr.io/nvidia/mellanox
      version: network-operator-v25.7.0
      imagePullSecrets: []

    multus:
      image: multus-cni
      repository: nvcr.io/nvidia/mellanox
      version: network-operator-v25.7.0
      imagePullSecrets: []

    ipamPlugin:
      image: whereabouts
      repository: nvcr.io/nvidia/mellanox
      version: network-operator-v25.7.0
      imagePullSecrets: []
```

Apply:

```bash
kubectl apply -f nicclusterpolicy.yaml
```

---

## Verify Network Operator

```bash
kubectl get nicclusterpolicy
```

Expected:

```text
STATE = ready
```

---

### Verify RDMA Resource Exposure

```bash
kubectl describe node <worker-node>
```

Expected resource:

```text
nvidia.com/hostdev
```

---

# 3. Install MPI Operator

Install MPI Operator CRDs and controller:

```bash
kubectl apply --server-side \
  -f https://raw.githubusercontent.com/kubeflow/mpi-operator/master/deploy/v2beta1/mpi-operator.yaml
```

---

## Verify MPI Operator

```bash
kubectl get pods -n mpi-operator
```

Expected:

```text
mpi-operator Running
```

---

# 4. Deploy NCCL AllReduce Validation Job

Apply NCCL validation workload:

```bash
kubectl apply -f nccl-all-reduce.yaml
```

---

## Monitor Job Status

```bash
kubectl get pods
```

---

## View NCCL Logs

Identify launcher pod:

```bash
kubectl get pods
```

Example:

```text
nccl-allreduce-launcher
```

Follow logs:

```bash
kubectl logs -f <launcher-pod-name>
```

Example:

```bash
kubectl logs -f nccl-allreduce-launcher
```

---

# 5. Validation Checklist

Verify all of the following before running NCCL benchmarks:

| Check                         | Expected   |
| ----------------------------- | ---------- |
| MOFED Installed               | Yes        |
| openibd Running               | Yes        |
| nvidia_peermem Loaded         | Yes        |
| RDMA Links                    | 8          |
| Bond Interface                | Up         |
| Network Operator              | Ready      |
| NicClusterPolicy              | Ready      |
| SR-IOV Device Plugin          | Running    |
| Multus                        | Running    |
| MPI Operator                  | Running    |
| `nvidia.com/hostdev` Resource | Present    |
| NCCL Job Pods                 | Running    |
| NCCL AllReduce                | Successful |

---

# Useful Troubleshooting Commands

### RDMA Devices

```bash
ibv_devices
```

```bash
ibstat
```

```bash
rdma link
```

---

### Kubernetes RDMA Resources

```bash
kubectl get nodes -o json | jq '.items[].status.allocatable'
```

---

### Network Operator

```bash
kubectl get pods -n nvidia-network-operator
```

```bash
kubectl logs -n nvidia-network-operator deployment/network-operator
```

---

### MPI Jobs

```bash
kubectl get mpijobs
```

```bash
kubectl describe mpijob <job-name>
```

---

### NCCL Debugging

Inside NCCL job containers:

```bash
export NCCL_DEBUG=INFO
export NCCL_IB_DISABLE=0
export NCCL_P2P_DISABLE=0
export NCCL_NET_GDR_LEVEL=SYS
```

Verify NCCL detects InfiniBand instead of falling back to TCP sockets.
