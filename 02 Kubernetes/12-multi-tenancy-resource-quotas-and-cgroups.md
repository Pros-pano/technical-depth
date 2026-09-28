# 12. Multi-Tenancy, ResourceQuotas & Linux cgroups v2 — Strict 5% Partitioning

In a shared AI computing infrastructure, multiple research teams compete for expensive GPU and CPU compute. Without strict multi-tenancy, a single buggy script can spawn 1,000 parallel threads, consume all memory, trigger host kernel panics, and terminate everyone else's jobs (**Noisy Neighbor Problem**).

This guide explores Kubernetes multi-tenancy models, Linux **cgroups v2** kernel internals, Quality of Service (QoS) classes, and how the **5% compute and storage isolation** is enforced from the Kubernetes API down to the physical silicon.

---

## 📑 Table of Contents
1. [Multi-Tenancy Models: Soft vs. Hard Isolation](#1-multi-tenancy-models-soft-vs-hard-isolation)
2. [Linux Kernel Control Groups (`cgroups v2`) Internals](#2-linux-kernel-control-groups-cgroups-v2-internals)
3. [How Kubernetes Maps Requests & Limits to cgroups](#3-how-kubernetes-maps-requests--limits-to-cgroups)
4. [Quality of Service (QoS) Classes & The OOM Hierarchy](#4-quality-of-service-qos-classes--the-oom-hierarchy)
5. [Enforcing Your Strict 5% Budget: Step-by-Step Math](#5-enforcing-your-strict-5-budget-step-by-step-math)
6. [Systemd Slices: Node-Level Isolation Outside Kubernetes](#6-systemd-slices-node-level-isolation-outside-kubernetes)
7. [Production Failure Scenarios & Throttling Diagnostics](#7-production-failure-scenarios--throttling-diagnostics)
8. [Hands-On cgroup v2 Inspection Labs on Host](#8-hands-on-cgroup-v2-inspection-labs-on-host)

---

## 1. Multi-Tenancy Models: Soft vs. Hard Isolation

```text
+------------------------------------+      +------------------------------------+
|          Soft Multi-Tenancy        |      |          Hard Multi-Tenancy        |
+------------------------------------+      +------------------------------------+
| Different teams in same company    |      | Untrusted / External customers     |
| Separated by K8s Namespaces,       |      | Separated by separate bare-metal   |
| RBAC, ResourceQuotas, and cgroups  |      | hosts, microVMs (Kata / Firecracker)|
| Shared Host Linux Kernel           |      | Isolated Virtual Hardware Kernels  |
| Overhead: < 0.1% (Native Speed)    |      | Overhead: 3-8% CPU overhead        |
+------------------------------------+      +------------------------------------+
```

For your DGX Spark simulation, **Soft Multi-Tenancy via hard cgroups and ResourceQuotas** delivers 100% native bare-metal AI performance without the virtualization penalty of VMware.

---

## 2. Linux Kernel Control Groups (`cgroups v2`) Internals

Control Groups (`cgroups`) are the Linux kernel mechanism that meters and limits resource consumption for a process tree.

In **cgroups v2** (unified hierarchy at `/sys/fs/cgroup/`), resources are controlled by specific files:

```text
/sys/fs/cgroup/kubepods.slice/
├── kubepods-burstable.slice/
│   └── kubepods-burstable-pod<UID>.slice/
│       ├── cpu.max          <── Controls CPU quota & period (CFS throttling)
│       ├── cpu.weight       <── Controls relative CPU priority (shares)
│       ├── memory.max       <── Hard memory ceiling (OOM trigger)
│       ├── memory.high      <── Soft memory threshold (proactive page reclaim)
│       ├── memory.current   <── Real-time bytes currently in use
│       └── io.max           <── Read/Write IOPS & byte bandwidth limits
```

### How `cpu.max` Enforces CPU Limits:
The Linux **Completely Fair Scheduler (CFS)** operates in time periods (default: `100,000` microseconds = 100ms).
- File format: `QUOTA PERIOD`
- If you allocate **1 CPU Core**: `cpu.max = 100000 100000` (Process can execute for 100ms out of every 100ms window).
- If you allocate **0.5 Cores (500m)**: `cpu.max = 50000 100000` (Process can execute for 50ms, then the kernel **pauses execution** for the remaining 50ms).

---

## 3. How Kubernetes Maps Requests & Limits to cgroups

When you specify `resources` in your Pod manifest, `kubelet` directly translates those values into cgroup v2 parameters:

| Kubernetes Manifest Field | Linux cgroup v2 File | Kernel Meaning |
| :--- | :--- | :--- |
| `resources.requests.cpu: "1000m"` | `cpu.weight` | Proportional share of CPU during contention (formula: `((1000 * 1024 / 1000) - 2) * 9999 / 262142 + 1`). |
| `resources.limits.cpu: "3200m"` | `cpu.max` | Hard execution ceiling (`320000 100000`). Once exhausted, CPU cycles are throttled. |
| `resources.requests.memory: "3200Mi"` | `memory.min` / `memory.low` | Protected memory floor (kernel will not reclaim this memory under pressure). |
| `resources.limits.memory: "6400Mi"` | `memory.max` | Hard memory ceiling (`6710886400` bytes). If process allocates more, **OOM Killer** executes. |

---

## 4. Quality of Service (QoS) Classes & The OOM Hierarchy

Kubernetes categorizes every Pod into one of three **QoS Classes**. When a physical node runs out of memory, the Linux kernel terminates pods based on their **`oom_score_adj`**:

```text
Node Memory Pressure Event (RAM 99% Full)
       │
       ▼ (First to be killed)
1. BestEffort Pods (`oom_score_adj = 1000`)
   └── Pods with NO requests and NO limits defined.
       │
       ▼ (Second to be killed)
2. Burstable Pods (`oom_score_adj = 2 to 999`)
   └── Pods where `requests < limits`. Score is proportional to memory usage vs limit.
       │
       ▼ (Last to be killed / Guaranteed Survivability)
3. Guaranteed Pods (`oom_score_adj = -997`)
   └── Pods where `requests == limits` for EVERY container (CPU & RAM).
       └── Protected system pods, critical inference APIs, and master training jobs.
```

---

## 5. Enforcing Your Strict 5% Budget: Step-by-Step Math

Let's compute the exact resource budget for your DGX Spark:

### 1. Compute Math (Based on 64-Core Grace CPU & 128GB RAM)
- **5% CPU Budget**:
  $$64 \text{ Cores} \times 0.05 = 3.2 \text{ Cores} = \mathbf{3200\text{m}}$$
- **5% RAM Budget**:
  $$128 \text{ GiB} \times 0.05 = 6.4 \text{ GiB} = \mathbf{6400\text{Mi}} = 6,710,886,400\text{ bytes}$$

### 2. Storage Math (Based on 1TB NVMe Host Disk)
- **5% Disk Budget**:
  $$1000 \text{ GiB} \times 0.05 = \mathbf{50\text{ GiB}}$$

### 3. GPU Math (1 Physical Blackwell GPU)
- Single GPU configured via Time-Slicing with **10 total slices**:
  $$1 \text{ Slice} = 10\% \text{ of scheduling time (or } \mathbf{1\text{ GPU request}}\text{)}$$

### The Resulting Production Manifest (`5pct-enforcer.yaml`)
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: strict-5pct-quota
  namespace: k3s-alpha
spec:
  hard:
    requests.cpu: "1600m"
    limits.cpu: "3200m"      # Hard ceiling: 3.2 cores
    requests.memory: "3200Mi"
    limits.memory: "6400Mi"  # Hard ceiling: 6.4 GB RAM
    requests.nvidia.com/gpu: "1"
    limits.nvidia.com/gpu: "1"
    requests.storage: "50Gi" # Hard ceiling: 50 GB storage
    persistentvolumeclaims: "4"
```

---

## 6. Systemd Slices: Node-Level Isolation Outside Kubernetes

If you run two separate K3s service instances on the host (e.g. `k3s-alpha.service` and `k3s-beta.service`), you can enforce the 5% limits at the **systemd service level**, completely independent of Kubernetes manifests:

Create `/etc/systemd/system/k3s-alpha.service.d/5pct-cgroup.conf`:
```ini
[Service]
# Linux cgroups v2 resource capping via systemd
CPUAccounting=yes
CPUQuota=320%

MemoryAccounting=yes
MemoryMax=6.4G

# Restrict Disk I/O write bandwidth to 100 MB/sec
IOAccounting=yes
IOReadBandwidthMax=/var/lib/k3s-alpha 100M
IOWriteBandwidthMax=/var/lib/k3s-alpha 100M
```
Apply systemd changes:
```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s-alpha
```

---

## 7. Production Failure Scenarios & Throttling Diagnostics

### Scenario 1: Severe CPU Throttling (`container_cpu_cfs_throttled_periods_total`)
- **Symptom**: An inference server has a response time of 5ms at 10 requests/sec, but jumps to 800ms at 25 requests/sec, even though CPU usage is only at 60%.
- **Root Cause**: The container's CPU limit is set too low. Multi-threaded applications (e.g. PyTorch with 8 OpenMP threads) exhaust their 100ms CFS quota in the first 20ms of the window. The kernel pauses all threads for the remaining 80ms!
- **Triage**:
  ```bash
  # Check if container is being throttled by the Linux kernel:
  cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/.../cpu.stat
  ```
  *Observe `nr_throttled` and `throttled_usec` metrics.*

---

### Scenario 2: Pod Unexpectedly Terminated with `Exit Code 137`
- **Symptom**: AI training script crashes silently at Epoch 3.
- **Triage**:
  ```bash
  kubectl describe pod pytorch-benchmark -n k3s-alpha
  ```
  *Output:*
  ```text
  Last State:     Terminated
    Reason:       OOMKilled
    Exit Code:    137
  ```
- **Resolution**: Check dataset batch size. Increase `limits.memory` in the Pod manifest or decrease the PyTorch `batch_size`.

---

## 8. Hands-On cgroup v2 Inspection Labs on Host

### Lab 1: Read the Live `cpu.max` and `memory.max` for a Running Pod
Run these commands on your DGX Spark host:

1. Find the host PID of your running `pytorch-benchmark` pod:
   ```bash
   CONTAINER_ID=$(sudo crictl ps --name pytorch -q)
   PID=$(sudo crictl inspect $CONTAINER_ID | jq '.info.pid')
   ```
2. Locate the container's cgroup directory in the unified hierarchy:
   ```bash
   CGROUP_PATH=$(cat /proc/$PID/cgroup | cut -d: -f3)
   echo "CGroup Path: /sys/fs/cgroup$CGROUP_PATH"
   ```
3. Inspect the active hardware limits set by the Linux kernel:
   ```bash
   cat /sys/fs/cgroup$CGROUP_PATH/cpu.max
   cat /sys/fs/cgroup$CGROUP_PATH/memory.max
   ```
   *Verify that `cpu.max` equals `200000 100000` (2 CPU cores limit) and `memory.max` equals `4294967296` (4 GiB).*

---

Proceed to [**13-nvidia-hardware-and-driver-stack.md**](13-nvidia-hardware-and-driver-stack.md) to explore the NVIDIA physical hardware, Grace Blackwell GB10 unified memory architecture, and kernel driver modules.
