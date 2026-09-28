# Volume 15: Parallel File System Client Orchestration: WekaFS, VAST & Lustre

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 15: WekaFS Agent, VAST NFS-over-RDMA, Lustre LNet Multi-Rail & Systemd Mount Units
====================================================================================================
```

---

## 1. Executive Intuition: The Shared Storage Bottleneck

In an exascale AI supercomputer, thousands of GPUs must simultaneously ingest training shards and flush multi-terabyte checkpoints. Connecting compute nodes to standard enterprise NAS (NFS over TCP) produces an immediate cluster failure:
1. **Single-Stream TCP Choke:** A standard NFS mount uses a single TCP connection, capping throughput at $\sim 1.2 - 1.8\text{ GB/s}$ regardless of whether the node is equipped with a 400 Gbps network adapter.
2. **Kernel VFS Lock Deadlocks:** When 64 DataLoader workers on a node concurrently issue `stat()` system calls over standard NFS, the kernel client locks up, causing processes to hang in uninterruptible sleep (`D-state`).
3. **Shutdown Hangs:** If an NFS or parallel filesystem server becomes unreachable during a cluster reboot, unmounting hangs indefinitely, preventing node evacuation.

```
+-----------------------------------------------------------------------------------------+
|                  PARALLEL STORAGE CLIENT ARCHITECTURES IN AI                            |
+-----------------------------------------------------------------------------------------+
| [WekaFS User-Space Client]:                                                             |
| Direct user-space microkernel bypass -> Native GDS -> Microsecond InfiniBand RDMA       |
|                                                                                         |
| [VAST Data Universal Storage Client]:                                                   |
| NFSv3 over RDMA (proto=rdma, port=20049) with nconnect=8 (8 parallel RDMA queue pairs)   |
|                                                                                         |
| [Lustre Exascale Client]:                                                               |
| Native Lustre LNet multi-rail RDMA driver striping across tens of Object Storage Targets|
+-----------------------------------------------------------------------------------------+
```

Automating high-performance storage clients requires Ansible to orchestrate **WekaFS Cluster Agents**, **VAST NFS-RDMA Mounts**, **Lustre LNet Drivers**, and **Systemd Mount Units**.

---

## 2. Lineage & Evolution of High-Performance Storage Clients

```
   [1984: Sun NFSv2 / NFSv3]
                 |
           (Stateless RPC over UDP/TCP; single socket connection; high CPU tax)
                 |
   [2003: Lustre Parallel File System]
                 |
           (LNet message layer for multi-rail InfiniBand; client-side file striping)
                 |
   [2015: NFS-over-RDMA (RFC 5666)]
                 |
           (Direct memory placement over InfiniBand/RoCEv2; eliminates kernel TCP stack)
                 |
   [2020: WekaFS Matrix User-Space POSIX Client]
                 |
           (Complete kernel VFS bypass; user-space polling loops and direct GDS integration)
```

---

## 3. First-Principles Mathematics: NFS-RDMA Multi-Connection (`nconnect`) Scaling

Standard NFSv3/v4 mounts serialize all I/O transactions through a single transport connection. In high-bandwidth networks ($400\text{ Gbps}$), a single connection cannot saturate the link due to TCP window scaling and single-core interrupt limits.

### 3.1 The `nconnect` Bandwidth Multiplier
The Linux kernel `nconnect` mount option instructs the client to establish $K$ independent transport connections (queue pairs) to the storage server:

$$B_{\text{client}} = \min(B_{\text{fabric}},\ K \times B_{\text{channel}})$$

```
+-----------------------------------------------------------------------------------------+
|                         NFS OVER RDMA THROUGHPUT SCALING                                |
+-----------------------+--------------------+--------------------+-----------------------+
| Mount Options         | Transport Type     | Active Connections | Peak Read Throughput  |
+-----------------------+--------------------+--------------------+-----------------------+
| proto=tcp, nconnect=1 | Standard TCP       | 1 Socket           | ~1.4 GB/sec           |
| proto=tcp, nconnect=8 | Multi-Path TCP     | 8 Sockets          | ~8.2 GB/sec           |
| proto=rdma, nconnect=1| Single RDMA QP     | 1 Queue Pair       | ~11.5 GB/sec          |
| proto=rdma, nconnect=8| Multi-Path RDMA    | 8 Queue Pairs      | ~44.8 GB/sec (Line!)  |
+-----------------------+--------------------+--------------------+-----------------------+
```

$$\text{Throughput Improvement } (\text{NFS-TCP 1} \to \text{NFS-RDMA 8}) = \frac{44.8}{1.4} = \mathbf{32.0\times}$$

---

## 4. Deep Architecture: Systemd Automated Mount Units (`.mount` / `.automount`)

Hardcoding parallel filesystem mounts in `/etc/fstab` is an operational hazard: if a storage server is temporarily offline during boot, systemd halts the entire server boot sequence.

Ansible deploys native **Systemd Mount Units** (`mnt-ai-storage.mount`) with strict dependency ordering:
- `After=network-online.target time-sync.target`
- `Wants=network-online.target`
- `TimeoutSec=15`: Fails quickly rather than hanging node reboots indefinitely.

```
+-----------------------------------------------------------------------------+
|                     SYSTEMD MOUNT DEPENDENCY TOPOLOGY                       |
+-----------------------------------------------------------------------------+
|  network-online.target (InfiniBand & RoCEv2 interfaces UP with IP addresses)|
|    |                                                                        |
|    v                                                                        |
|  mnt-ai-storage.mount (Mounts /mnt/ai-storage via NFS-RDMA / WekaFS)        |
|    |                                                                        |
|    v                                                                        |
|  containerd.service / slurmctld.service (Dependent AI workloads launch)    |
+-----------------------------------------------------------------------------+
```

---

## 5. Concrete Production Lab: Automated Multi-PFS Client Role

Below is an enterprise Ansible role that automates client deployment for both **WekaFS** and **VAST Data NFS-over-RDMA**.

### 5.1 Role Tasks (`tasks/main.yml`)
```yaml
---
# Tasks for parallel filesystem client orchestration
- name: 1. Deploy VAST Data NFS-over-RDMA Mount
  when: storage_backend == "vast"
  block:
    - name: Ensure NFS common utilities installed
      ansible.builtin.apt:
        name: nfs-common
        state: present

    - name: Create mount directory
      ansible.builtin.file:
        path: /mnt/vast-storage
        state: directory
        mode: '0777'

    - name: Deploy Systemd Mount Unit for VAST NFS-RDMA
      ansible.builtin.copy:
        dest: /etc/systemd/system/mnt-vast\x2dstorage.mount
        content: |
          [Unit]
          Description=VAST Data High-Performance NFS-over-RDMA Storage Mount
          After=network-online.target
          Wants=network-online.target

          [Mount]
          What={{ vast_vip_address }}:/ai_corpus
          Where=/mnt/vast-storage
          Type=nfs
          Options=proto=rdma,port=20049,nconnect=8,rsize=1048576,wsize=1048576,hard,intr,noatime
          TimeoutSec=15

          [Install]
          WantedBy=multi-user.target
      notify: Reload Systemd Mounts

    - name: Enable and Start VAST Mount Unit
      ansible.builtin.systemd:
        name: mnt-vast\x2dstorage.mount
        state: started
        enabled: true
        daemon_reload: true

- name: 2. Deploy WekaFS User-Space Matrix Client
  when: storage_backend == "weka"
  block:
    - name: Download and execute WekaFS agent install
      ansible.builtin.shell: >
        curl -k https://{{ weka_backend_ip }}:14000/dist/v1/install | sh
      args:
        creates: /usr/bin/weka

    - name: Create WekaFS mount directory
      ansible.builtin.file:
        path: /mnt/weka-storage
        state: directory
        mode: '0777'

    - name: Mount WekaFS Filesystem with GDS Enabled
      ansible.builtin.command: >
        weka fs mount {{ weka_fs_name }} /mnt/weka-storage -o net=roce0 -o gds
      register: weka_mount_res
      changed_when: false
      failed_when: "'Mounted' not in weka_mount_res.stdout and weka_mount_res.rc != 0"

- name: 3. Verify Storage Mount Responsiveness
  ansible.builtin.command: stat /mnt/{{ 'vast-storage' if storage_backend == 'vast' else 'weka-storage' }}
  register: stat_res
  changed_when: false
  failed_when: stat_res.rc != 0

handlers:
  - name: Reload Systemd Mounts
    ansible.builtin.systemd:
      daemon_reload: true
```

---

## 6. Comparative Client Performance Matrix

| Client Architecture | Transport Protocol | Max Ingestion BW | GDS Direct-to-HBM | Metadata IOPS Rate |
| :--- | :--- | :--- | :--- | :--- |
| **Standard NFSv4 (TCP)** | L4 TCP (`proto=tcp`) | 1.8 GB/s | No (CPU Bounce) | 2,500 ops/sec |
| **VAST NFS-over-RDMA** | L3 RoCEv2 (`proto=rdma`)| **44.5 GB/s** | Supported (`cufile`) | 45,000 ops/sec |
| **Lustre LNet Client** | InfiniBand Verbs | **48.0 GB/s** | Experimental | 80,000 ops/sec |
| **WekaFS Matrix Client** | Custom Microkernel | **54.0 GB/s** | **Native Zero-Copy** | **250,000 ops/sec** |

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        PARALLEL STORAGE CLIENT SRE DIAGNOSTIC MATRIX                              |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| System hangs when listing mount:   | Storage server down;     | Issue unmount with force/lazy:    |
| `ls /mnt/storage` deadlocks.       | process stuck in D-state.| `umount -l /mnt/storage`          |
|                                    |                          | Verify network connectivity.      |
+------------------------------------+--------------------------+-----------------------------------+
| VAST mount fails with:             | RDMA protocol rejected;  | Test port 20049 over RDMA:        |
| `Protocol not supported`.          | server lacks RDMA export.| Mount with `proto=tcp` temporarily|
|                                    |                          | to isolate network issue.         |
+------------------------------------+--------------------------+-----------------------------------+
| Weka client fails with:            | `roce0` interface        | Check interface operational state:|
| `Cannot find network device`.      | renamed or down.         | `ip link show roce0`              |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **RDMA Transport Active:** NFS mounts verified running with `proto=rdma` on port 20049.
- [ ] **Multi-Pathing Enabled:** `nconnect=8` declared in mount options.
- [ ] **Systemd Mount Units Deployed:** Native `.mount` units replace legacy `/etc/fstab` entries.
- [ ] **Graceful Timeout Sized:** `TimeoutSec=15` prevents hanging system shutdowns.
- [ ] **GDS Acceleration Qualified:** WekaFS and VAST client mounts pass `cufile` direct I/O assertions.
