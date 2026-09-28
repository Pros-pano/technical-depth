# 13. NVIDIA Hardware & Driver Stack — Grace Blackwell GB10 & Unified Memory

To operate at the highest level of AI infrastructure engineering, you must understand the physical silicon, coherent interconnects, and Linux kernel drivers that power modern accelerators.

This guide explores the NVIDIA DGX hardware lineage, the **Grace Blackwell (GB10)** architecture powering your DGX Spark, unified memory over NVLink-C2C, and the kernel module subsystem.

---

## 📑 Table of Contents
1. [NVIDIA DGX System Lineage](#1-nvidia-dgx-system-lineage)
2. [DGX Spark & Grace Blackwell (GB10) Architecture](#2-dgx-spark--grace-blackwell-gb10-architecture)
3. [NVLink-C2C: The End of the PCIe Bottleneck](#3-nvlink-c2c-the-end-of-the-pcie-bottleneck)
4. [The Linux Kernel Driver Modules Subsystem](#4-the-linux-kernel-driver-modules-subsystem)
5. [CUDA Driver API vs. CUDA Runtime API](#5-cuda-driver-api-vs-cuda-runtime-api)
6. [GPU Topology & Interconnect Bus Hierarchies](#6-gpu-topology--interconnect-bus-hierarchies)
7. [Production Hardware Diagnostics & Performance Telemetry](#7-production-hardware-diagnostics--performance-telemetry)
8. [Hands-On Hardware Inspection Labs](#8-hands-on-hardware-inspection-labs)

---

## 1. NVIDIA DGX System Lineage

| Platform | GPU Architecture | Intra-Node Interconnect | Memory Subsystem | Key Architectural Leap |
| :--- | :--- | :--- | :--- | :--- |
| **DGX-1** (2016) | 8x Tesla P100 / V100 | Hybrid Cube Mesh NVLink (160-300 GB/s) | HBM2 (16GB-32GB per GPU) | First turnkey deep learning supercomputer. |
| **DGX-2** (2018) | 16x V100 (SXM3) | NVSwitch Crossbar (2.4 TB/s bisection) | 512GB total unified HBM2 | First single-node multi-GPU crossbar switch. |
| **DGX A100** (2020) | 8x A100 (SXM4) | NVLink 3 + NVSwitch (600 GB/s per GPU) | 320GB - 640GB total HBM2e | First hardware **Multi-Instance GPU (MIG)**. |
| **DGX H100** (2022) | 8x H100 (SXM5) | NVLink 4 + NVSwitch (900 GB/s per GPU) | 640GB total HBM3 | Transformer Engine with native FP8 precision. |
| **DGX Spark** (Current)| **Grace Blackwell (GB10)** | **NVLink-C2C (900 GB/s)** | **Coherent Unified Memory** | **Grace ARM CPU + Blackwell GPU unified.** |
| **DGX GB200 NVL72** | 72x Blackwell GPUs | NVLink 5 (1.8 TB/s per GPU) | 13.5 TB coherent memory | Liquid-cooled single rack acting as one giant GPU. |

---

## 2. DGX Spark & Grace Blackwell (GB10) Architecture

The **DGX Spark** represents a fundamental departure from traditional x86 server architecture:

```text
+-----------------------------------------------------------------------------------+
|                            DGX Spark (GB10 Architecture)                          |
|                                                                                   |
|  +---------------------------+                     +---------------------------+  |
|  |     NVIDIA Grace CPU      |   NVLink-C2C        |    NVIDIA Blackwell GPU   |  |
|  |   - 72x ARM Neoverse V2   | <=================> |   - 5th Gen Tensor Cores  |  |
|  |   - Scalable Vector SVE2  |     900 GB/s        |   - FP4 / FP8 Precision   |  |
|  +---------------------------+   Bidirectional     +---------------------------+  |
|                \                                                 /                |
|                 \                                               /                 |
|                  +---------------------------------------------+                  |
|                  |       Unified Coherent Memory Subsystem     |                  |
|                  |     (Shared High-Speed Physical Memory)     |                  |
|                  +---------------------------------------------+                  |
+-----------------------------------------------------------------------------------+
```

---

## 3. NVLink-C2C: The End of the PCIe Bottleneck

In legacy server architectures (Intel Xeon / AMD EPYC + PCIe GPU):
- Data must travel across the **PCIe Gen 4 / Gen 5 bus** (bandwidth limited to 32 GB/s - 64 GB/s).
- Before running a CUDA kernel, the CPU must allocate memory in system RAM, map it, and explicitly issue `cudaMemcpy(..., cudaMemcpyHostToDevice)`.
- If a dataset or model weight tensor exceeds GPU VRAM, the training script crashes with `CUDA out of memory`.

### In DGX Spark (GB10 Architecture):
- **NVLink-C2C (Chip-to-Chip)** connects the Grace CPU and Blackwell GPU with **900 GB/s bandwidth** (up to 14x faster than PCIe Gen 5).
- **Hardware Cache-Coherent Memory**: The CPU and GPU share the same physical address space.
- A pointer allocated by the CPU (`malloc()`) is directly readable and writable by the GPU without copying.

---

## 4. The Linux Kernel Driver Modules Subsystem

The host Linux kernel communicates with the NVIDIA accelerator via loadable kernel modules:

```text
User Applications (PyTorch, Triton, vLLM)
       │
       ▼
CUDA Driver API (`libcuda.so.1`)
       │
       ▼ IOCTL Syscalls (/dev/nvidiactl, /dev/nvidia0, /dev/nvidia-uvm)
Linux Kernel Boundary
       │
       ├── `nvidia.ko`: Core hardware initialization, clocks, power, PCIe registers
       ├── `nvidia-uvm.ko`: Unified Virtual Memory page table manager & fault handler
       ├── `nvidia-modeset.ko`: Display timing (disabled on headless servers)
       └── `nvidia-peermem.ko`: Direct RDMA bridge between Mellanox NICs and GPU
```

### Inspecting Active Modules on DGX Spark:
```bash
lsmod | grep -E "nvidia|nv_peer_mem"
```

---

## 5. CUDA Driver API vs. CUDA Runtime API

Understanding this distinction is essential for containerized AI platforms:

| Metric | CUDA Driver API (`libcuda.so.1`) | CUDA Runtime API (`libcudart.so`) |
| :--- | :--- | :--- |
| **Location** | Installed on **Host Operating System** with Driver | Bundled **Inside Container Image** (e.g. PyTorch) |
| **Language** | Low-level C API | High-level C++ API |
| **Stability** | Strictly backward compatible | Can be upgraded per container |
| **Interface** | Interacts directly with `/dev/nvidiactl` | Calls `libcuda.so` under the hood |

### CUDA Forward Compatibility:
Thanks to NVIDIA's forward compatibility model, a newer CUDA runtime container (e.g. CUDA 12.4 in PyTorch 2.4) can run smoothly on an older host driver (e.g. Driver 535) by dynamically mounting a forward compatibility driver package.

---

## 6. GPU Topology & Interconnect Bus Hierarchies

In multi-GPU nodes, how GPUs are linked physically determines communication speed:

```bash
nvidia-smi topo -m
```

### Legend of Matrix Connections:
- **`SYS`**: Connected via Host System Bus (Slowest, travels through CPU/RAM).
- **`PIX`**: Connected via a single PCIe bridge.
- **`PXB`**: Connected via multiple PCIe bridges.
- **`NV#`**: Connected via **NVLink** (Fastest, full non-blocking bandwidth).

On DGX Spark, because the Grace CPU and Blackwell GPU share NVLink-C2C, memory access bypasses standard PCIe switch topologies.

---

## 7. Production Hardware Diagnostics & Performance Telemetry

### 1. Real-Time Hardware Performance Query
```bash
nvidia-smi --query-gpu=timestamp,name,pstate,temperature.gpu,utilization.gpu,utilization.memory,memory.total,memory.free,memory.used,power.draw \
  --format=csv -l 1
```

### 2. Checking for Hardware Throttling
```bash
nvidia-smi -q -d PERFORMANCE
```
*Look for `Clocks Throttle Reasons`:*
- `Sw Power Cap`: GPU reached its maximum TDP power limit.
- `Hw Slowdown`: Severe thermal overload (fan failure or cooling issue).
- `Sync Boost`: Clocks synchronized across NVLink.

---

## 8. Hands-On Hardware Inspection Labs

### Lab 1: Inspect Unified Memory Coherence
On your DGX Spark host, observe how system RAM and GPU memory interact:
```bash
# Query unified memory:
free -h

# Query GPU memory:
nvidia-smi --query-gpu=memory.total,memory.used,memory.free --format=csv
```

### Lab 2: Inspect PCIe Bus Generation and Width
Verify that the GPU is operating at full PCIe/bus link speed:
```bash
sudo lspci -vvv -d 10de:* | grep -E "LnkCap|LnkSta"
```
*Verify that `LnkSta` (Link Status) matches `LnkCap` (Link Capability) — e.g. Speed 32GT/s, Width x16.*

---

Proceed to [**14-nvidia-container-toolkit-and-gpu-virtualization.md**](14-nvidia-container-toolkit-and-gpu-virtualization.md) to explore the NVIDIA Container Toolkit (`nvidia-ctk`), CDI, and the deep comparison between Time-Slicing, MIG, and MPS.
