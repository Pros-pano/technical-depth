# Volume 18: Slurm Workload Manager Orchestration & cgroup GPU Confinement

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 18: Slurm Controller, slurmd Daemons, gres.conf GPU Slicing & cgroup v2 Device Confinement
====================================================================================================
```

---

## 1. Executive Intuition: The Supercomputing Scheduler

While Kubernetes is the default orchestrator for enterprise microservices and inference APIs, **Slurm Workload Manager (Simple Linux Utility for Resource Management)** remains the gold-standard workload scheduler for multi-month foundation model pre-training campaigns (powering clusters at Meta, DeepSeek, and national supercomputing labs):

1. **Deterministic Gang Scheduling:** Slurm starts all 16,384 GPUs in a distributed job at the exact same microsecond, eliminating the resource fragmentation and deadlocks common in Kubernetes pod scheduling.
2. **Hardware cgroup Device Confinement:** Slurm uses Linux kernel cgroups (`cgroup.conf`) to create an impenetrable sandbox around each job: a job requesting 4 GPUs is physically restricted via `/sys/fs/cgroup/devices` from communicating with or accessing the other 4 GPUs on the same motherboard.
3. **Zero Container Runtime Overhead:** Jobs run directly on bare metal (or inside lightweight Singularity/Apptainer containers) with native InfiniBand and GDS access, eliminating Kubelet and CNI networking layers.

```
+-----------------------------------------------------------------------------------------+
|                          SLURM DISTRIBUTED CONTROL TOPOLOGY                             |
+-----------------------------------------------------------------------------------------+
| [Slurm Controller Node: slurmctld]                                                      |
|   - Authenticates via MUNGE cryptographic tokens (/etc/munge/munge.key)                 |
|   - Reads slurm.conf: Trackable Resources (TRES: CPU, RAM, GPU)                         |
|   - High-Speed Scheduler: Evaluates fair-share priority and backfill reservations       |
|                                                                                         |
| [Compute Nodes: slurmd Daemons on DGX H100 Servers]                                     |
|   - gres.conf: Maps physical device nodes /dev/nvidia[0-7] to Slurm GPU resources       |
|   - cgroup.conf: Enforces kernel device cgroup boundaries:                              |
|       ConstrainDevices=yes -> Injects only assigned /dev/nvidiaX into job cgroup        |
|       ConstrainRAMSpace=yes -> Prevents host DRAM OOM from crashing neighbor jobs       |
+-----------------------------------------------------------------------------------------+
```

Automating Slurm across an enterprise AI cluster requires Ansible to manage **MUNGE authentication keys**, **`slurm.conf` topology files**, **`gres.conf` GPU mappings**, and **cgroup confinement rules**.

---

## 2. Lineage & Evolution of High-Performance Batch Schedulers

```
   [1990s: PBS & Platform LSF]
                 |
           (Proprietary enterprise batch systems; static CPU queue slot allocations)
                 |
   [2002: Slurm Project (LLNL / SchedMD)]
                 |
           (Open-source scalable cluster resource management for top supercomputers)
                 |
   [2015: Generic Resource Scheduling (GRES)]
                 |
           (Native support for tracking and allocating coprocessors like NVIDIA GPUs)
                 |
   [2021: Trackable Resources (TRES) & cgroup v2]
                 |
           (Unified tracking of GPU memory, NVLink fabrics, and unified cgroups v2 trees)
                 |
   [2024: PyTorch Elastic & Slurm Auto-Requeue]
                 |
           (Native integration with torchrun, PyTorch DCP, and preemptible batch queues)
```

---

## 3. First-Principles Mathematics: TRES Scheduling & cgroup Memory Limits

Slurm allocates resources using **Consumable Trackable Resources (TRES)** via the `select/cons_tres` plugin.

### 3.1 Memory Confinement Calculation
If an 8-GPU node contains $2\text{ TB}$ ($2,048\text{ GB}$) of host RAM, and a user submits a job requesting 2 GPUs:

$$\text{Allocated RAM} = \text{Total RAM} \times \frac{\text{Requested GPUs}}{\text{Total GPUs}} \times \text{AllowedRAMSpaceRatio}$$

If `AllowedRAMSpace=95`:
$$\text{Max Allowed Host RAM} = 2048\text{ GB} \times \left(\frac{2}{8}\right) \times 0.95 = \mathbf{486.4\text{ Gigabytes}}$$

The Linux kernel cgroup subsystem enforces this ceiling at `/sys/fs/cgroup/memory/slurm/uid_X/job_Y/memory.limit_in_bytes`. If the training job leaks host memory and exceeds $486.4\text{ GB}$, the kernel OOM-killer terminates only that specific job, safeguarding the remaining 6 GPUs on the server.

---

## 4. Deep Architecture: `gres.conf` and `cgroup.conf` Configuration

### 4.1 Generic Resource Configuration (`/etc/slurm/gres.conf`)
Explicitly maps GPU indices, device nodes, and CPU NUMA socket affinities:

```
# 8x NVIDIA H100 SXM5 with NUMA Socket Binding
NodeName=dgx[01-64] Name=gpu Type=h100 File=/dev/nvidia0 Cores=0-63
NodeName=dgx[01-64] Name=gpu Type=h100 File=/dev/nvidia1 Cores=0-63
NodeName=dgx[01-64] Name=gpu Type=h100 File=/dev/nvidia2 Cores=0-63
NodeName=dgx[01-64] Name=gpu Type=h100 File=/dev/nvidia3 Cores=0-63
NodeName=dgx[01-64] Name=gpu Type=h100 File=/dev/nvidia4 Cores=64-127
NodeName=dgx[01-64] Name=gpu Type=h100 File=/dev/nvidia5 Cores=64-127
NodeName=dgx[01-64] Name=gpu Type=h100 File=/dev/nvidia6 Cores=64-127
NodeName=dgx[01-64] Name=gpu Type=h100 File=/dev/nvidia7 Cores=64-127
```

### 4.2 Linux cgroup Confinement (`/etc/slurm/cgroup.conf`)
```
CgroupPlugin=cgroup/v2
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=yes
AllowedRAMSpace=95
MaxRAMPercent=98
```

---

## 5. Concrete Production Lab: Automated Slurm Cluster Deployment Playbook

```yaml
---
# playbook: deploy_slurm_cluster.yml
# Deploys MUNGE authentication, configures slurm.conf, gres.conf, and starts daemons
- name: Phase 1 - Configure MUNGE Authentication Fabric
  hosts: all_slurm_nodes
  become: true
  tasks:
    - name: 1. Install MUNGE Authentication Service
      ansible.builtin.apt:
        name: munge
        state: present

    - name: 2. Deploy Shared MUNGE Secret Key
      ansible.builtin.copy:
        dest: /etc/munge/munge.key
        content: "{{ munge_shared_secret }}"
        owner: munge
        group: munge
        mode: '0400'
      notify: Restart Munge

    - name: 3. Enable and Start MUNGE Service
      ansible.builtin.systemd:
        name: munge
        state: started
        enabled: true

  handlers:
    - name: Restart Munge
      ansible.builtin.systemd:
        name: munge
        state: restarted

- name: Phase 2 - Configure Slurm Compute Nodes (Workers)
  hosts: gpu_nodes
  become: true
  tasks:
    - name: 1. Install slurmd daemon package
      ansible.builtin.apt:
        name: slurmd
        state: present

    - name: 2. Deploy Slurm GRES GPU Mapping (gres.conf)
      ansible.builtin.template:
        src: templates/gres.conf.j2
        dest: /etc/slurm/gres.conf
        owner: slurm
        group: slurm
        mode: '0644'
      notify: Restart Slurmd

    - name: 3. Deploy Slurm cgroup Confinement (cgroup.conf)
      ansible.builtin.copy:
        dest: /etc/slurm/cgroup.conf
        owner: slurm
        group: slurm
        mode: '0644'
        content: |
          CgroupPlugin=cgroup/v2
          ConstrainCores=yes
          ConstrainDevices=yes
          ConstrainRAMSpace=yes
          AllowedRAMSpace=95
      notify: Restart Slurmd

    - name: 4. Deploy Global slurm.conf Configuration
      ansible.builtin.template:
        src: templates/slurm.conf.j2
        dest: /etc/slurm/slurm.conf
        owner: slurm
        group: slurm
        mode: '0644'
      notify: Restart Slurmd

    - name: 5. Enable and Start slurmd Service
      ansible.builtin.systemd:
        name: slurmd
        state: started
        enabled: true

  handlers:
    - name: Restart Slurmd
      ansible.builtin.systemd:
        name: slurmd
        state: restarted
```

---

## 6. Comparative Cluster Scheduler Matrix

| Capability | Kubernetes (K8s) | Slurm Workload Manager | Ray Cluster |
| :--- | :--- | :--- | :--- |
| **Primary Domain** | Microservices & Serving | **Exascale Model Training** | Distributed Python / RL |
| **Scheduling Model** | Declarative Pod Event Loop | **Microsecond Gang Scheduling**| Actor / Task Scheduler |
| **GPU Isolation** | Device Plugin / CDI | **Linux Kernel cgroups v2** | Dynamic Python worker |
| **Bare-Metal MPI / NCCL**| Extra CNI configuration | **Native High-Speed RDMA** | Throughput throttled |
| **Preemption & Requeue** | Eviction / Restart | **Native `scontrol requeue`** | Manual retry logic |

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        SLURM WORKLOAD SRE DIAGNOSTIC MATRIX                                       |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| `slurmd` fails to start:           | MUNGE authentication key | Verify munge communication:       |
| "Failed to authenticate munge".    | mismatch or clock skew.  | `munge -n | unmunge`              |
|                                    |                          | Sync clocks via `chrony`.         |
+------------------------------------+--------------------------+-----------------------------------+
| Node stuck in `DRAIN` state with   | Node rebooted or GRES GPU| Inspect drain reason:             |
| reason: "Unexpected reboot".       | count mismatched in conf.| `sinfo -R`                        |
|                                    |                          | Re-enable: `scontrol update       |
|                                    |                          |  NodeName=<node> State=RESUME`    |
+------------------------------------+--------------------------+-----------------------------------+
| Training job crashes with:         | Job exceeded allocated   | Check cgroup memory counters:     |
| "Killed" (Signal 9).               | cgroup host memory limit.| `cat /sys/fs/cgroup/memory/slurm/ |
|                                    |                          |  uid_*/job_*/memory.max_usage...` |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **MUNGE Authentication Verified:** `munge -n | ssh <node> unmunge` verifies seamless cross-node trust.
- [ ] **GRES GPU Registration Confirmed:** `scontrol show node <name>` reports `Gres=gpu:h100:8`.
- [ ] **cgroups Device Confinement Active:** Job with `--gpus=2` sees only two `/dev/nvidiaX` device nodes.
- [ ] **Interactive GPU Allocation Tested:** `srun --gpus=1 nvidia-smi` successfully returns allocated GPU.
- [ ] **Automated Node Resumption Active:** Slurm daemon automatically reconciles node state post-reboot.
