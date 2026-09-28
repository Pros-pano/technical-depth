# Volume 11: InfiniBand Fabric Automation: MOFED, OpenSM, PKeys & IPoIB

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 11: NVIDIA DOCA-OFED, OpenSM Subnet Management, Fat-Tree Routing & IPoIB Connected Mode
====================================================================================================
```

---

## 1. Executive Intuition: The Managed Fabric Requirement

InfiniBand (Quantum-2 NDR 400 Gbps / Quantum-X800 800 Gbps) is the premier low-latency, high-bandwidth interconnect for foundation model training. However, InfiniBand operates on a fundamental paradigm that separates it from standard Ethernet: **InfiniBand switches are physically incapable of forwarding packets autonomously without a centralized Subnet Manager (OpenSM)**.

When an InfiniBand cable is plugged into a switch:
1. **The Dark Fabric State:** The switch ports illuminate physically, but no traffic can flow. The nodes have no Local Identifiers (LIDs), and switch forwarding tables are blank.
2. **The OpenSM Engine:** A designated Subnet Manager must sweep the fabric, discover the multi-tier Fat-Tree topology, calculate deadlock-free routing tables using algorithms like **Fat-Tree (`ftree`)**, assign 16-bit LIDs to every Host Channel Adapter (HCA), and program the Linear Forwarding Tables (LFTs) in every switch ASIC.
3. **Partition Isolation (PKeys):** Tenant isolation and storage segregation are enforced at the hardware level using 16-bit **Partition Keys (PKeys)** managed by OpenSM's partition tables.

```
+-----------------------------------------------------------------------------------------+
|                        INFINIBAND FABRIC CONTROL TOPOLOGY                               |
+-----------------------------------------------------------------------------------------+
| [Centralized Fabric Management Layer: OpenSM Service (Primary / Standby)]               |
|   - Sweeps fabric every 10 seconds via Subnet Management Packets (SMPs)                 |
|   - Computes credit-loop-free routing paths (Algorithm: ftree)                         |
|   - Distributes Linear Forwarding Tables (LFTs) to Quantum-2 Switches                   |
|   - Enforces Partition Keys: partitions.conf (Storage PKey: 0x8001, Compute: 0x8002)    |
|                                                                                         |
| [Compute Nodes: DGX H100 / HGX B200 Servers]                                            |
|   - Driver: NVIDIA DOCA-OFED / MLNX_OFED (mlx5_core, mlx5_ib, ib_uverbs)                |
|   - High-Speed Verbs: Native RDMA for NCCL AllReduce / GPUDirect RDMA                   |
|   - IP-over-InfiniBand (IPoIB): ib0 / ib1 configured in Connected Mode (MTU 65520)      |
+-----------------------------------------------------------------------------------------+
```

Automating an enterprise InfiniBand cluster requires Ansible to handle **DOCA-OFED compilation**, **High-Availability OpenSM deployment**, and **IPoIB network optimization**.

---

## 2. Lineage & Evolution of InfiniBand Management

```
   [2000: InfiniBand Trade Association (IBTA)]
                 |
           (Standardized credit-based flow control, hardware verbs, and subnet management)
                 |
   [2010: Mellanox OFED (MLNX_OFED)]
                 |
           (Enterprise Linux distribution packaging for InfiniBand verbs and IPoIB)
                 |
   [2018: InfiniBand Quantum HDR 200 Gbps & SHARP]
                 |
           (Scalable Hierarchical Aggregation and Reduction Protocol for in-network math)
                 |
   [2022: Quantum-2 NDR 400 Gbps & DOCA-OFED]
                 |
           (NVIDIA unifies networking stack under DOCA-OFED with automated tuning)
```

---

## 3. First-Principles Mathematics: IPoIB Connected Mode vs. Datagram Mode

IP-over-InfiniBand (IPoIB) encapsulates standard IP packets inside InfiniBand transport. It operates in two modes:

### 3.1 Datagram Mode (Unreliable Datagram - UD)
- Packets are mapped directly onto InfiniBand UD transport.
- Maximum Transmission Unit (MTU) is strictly bounded by the physical IB link MTU: **$2,044\text{ bytes}$** or **$4,092\text{ bytes}$**.
- Generating a $100\text{ GB/s}$ ingestion stream with $4,092\text{ B}$ MTU generates:
  $$\text{Packet Rate} = \frac{100 \times 10^9\text{ B/s}}{4,092\text{ B}} \approx \mathbf{24,437,927\text{ packets/sec}}$$
  This creates massive CPU interrupt overhead, stalling system responsiveness.

### 3.2 Connected Mode (Reliable Connection - RC)
- Emulates a virtual streaming connection between endpoints.
- Maximum Transmission Unit (MTU) expands to **$65,520\text{ bytes}$**!
- Generating the same $100\text{ GB/s}$ stream with $65,520\text{ B}$ MTU generates:
  $$\text{Packet Rate} = \frac{100 \times 10^9\text{ B/s}}{65,520\text{ B}} \approx \mathbf{1,526,251\text{ packets/sec}}$$

$$\text{Packet / Interrupt Reduction Factor} = \frac{24.4\text{M}}{1.52\text{M}} \approx \mathbf{16.0\times}$$

> **Performance Axiom:** In AI storage networks running over IPoIB, running in Datagram mode collapses throughput by over $60\%$. Ansible must explicitly configure `mode = connected` and `mtu = 65520`.

---

## 4. Deep Architecture: OpenSM Fat-Tree Routing Algorithm (`ftree`)

In a multi-tier Clos network (Leaf-Spine topology), naive routing algorithms (like MinHop) cause bandwidth bottlenecks by routing multiple flows through the same spine switch while leaving adjacent spines idle.

The **Fat-Tree (`ftree`) Routing Engine** mathematically guarantees:
1. **Full Bisection Bandwidth:** Traffic between compute nodes is evenly balanced across all available uplink paths.
2. **Deadlock Freedom:** Enforces strict upward-then-downward routing, eliminating cyclic buffer dependencies.
3. **Downward Path Balancing:** Preserves uniform port utilization on downward leaf links.

```
# /etc/opensm/opensm.conf snippet
routing_engine ftree
f_rebalance true
sweep_interval 10
```

---

## 5. Concrete Production Lab: Automated OFED & OpenSM Deployment Playbook

```yaml
---
# playbook: infiniband_fabric_orchestration.yml
# Provisions NVIDIA DOCA-OFED, configures IPoIB Connected Mode, and deploys OpenSM
- name: Phase 1 - Deploy OpenSM Fabric Subnet Manager (Manager Nodes)
  hosts: infiniband_managers
  become: true
  tasks:
    - name: 1. Install OpenSM Daemon
      ansible.builtin.apt:
        name: opensm
        state: present

    - name: 2. Configure High-Performance Fat-Tree OpenSM Configuration
      ansible.builtin.copy:
        dest: /etc/opensm/opensm.conf
        content: |
          # OpenSM Configuration for AI SuperPOD
          routing_engine ftree
          sweep_interval 10
          log_file /var/log/opensm.log
          log_max_size 1024
          f_rebalance true
          priority 15  # High priority for primary manager
      notify: Restart OpenSM

    - name: 3. Deploy Fabric Partition Table (PKeys)
      ansible.builtin.copy:
        dest: /etc/opensm/partitions.conf
        content: |
          # Global Default Partition
          Default=0x7fff, ipoib, defmember=full : ALL=full ;
          # Storage Dedicated Partition
          Storage=0x8001, ipoib, defmember=full : ALL=full ;
      notify: Restart OpenSM

    - name: 4. Enable and Start OpenSM Service
      ansible.builtin.systemd:
        name: opensm
        state: started
        enabled: true

  handlers:
    - name: Restart OpenSM
      ansible.builtin.systemd:
        name: opensm
        state: restarted

- name: Phase 2 - Configure Compute Node HCAs & IPoIB (GPU Nodes)
  hosts: gpu_nodes
  become: true
  tasks:
    - name: 1. Ensure DOCA-OFED Kernel Modules Loaded
      community.general.modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - ib_core
        - mlx5_core
        - mlx5_ib
        - ib_uverbs
        - ib_ipoib

    - name: 2. Configure IPoIB Connected Mode & MTU 65520 via Udev
      ansible.builtin.copy:
        dest: /etc/udev/rules.d/99-ipoib.rules
        content: |
          ACTION=="add", SUBSYSTEM=="net", NAME=="ib*", RUN+="/sbin/ip link set dev $name mode connected", RUN+="/sbin/ip link set dev $name mtu 65520"
      notify: Trigger Udev Rules

    - name: 3. Verify InfiniBand HCA Physical Link State
      ansible.builtin.command: ibstat
      register: ibstat_out
      changed_when: false
      failed_when: "'State: Active' not in ibstat_out.stdout"

    - name: 4. Audit Fabric Link Quality via perfquery
      ansible.builtin.command: perfquery -r
      register: perf_out
      changed_when: false
      failed_when: "'SymbolErrorCounter' in perf_out.stdout and 'SymbolErrorCounter: 0' not in perf_out.stdout"

  handlers:
    - name: Trigger Udev Rules
      ansible.builtin.command: udevadm trigger
```

---

## 6. Comparative Fabric Architecture Matrix

| Feature | InfiniBand Quantum-2 (NDR) | Lossless RoCEv2 (Ethernet) | Standard TCP/IP |
| :--- | :--- | :--- | :--- |
| **Control Plane** | **Centralized OpenSM (LIDs)** | Distributed BGP / EVPN | Distributed IP Routing |
| **Lossless Guarantee** | **Link-layer credit tokens** | Priority Flow Control (PFC) | None (Packet drop & retry) |
| **Latency (4KB Verbs)** | **0.8 - 1.5 microseconds** | 8.0 - 15.0 microseconds | 45.0 - 80.0 microseconds |
| **Routing Algorithm** | Deterministic Fat-Tree (`ftree`)| Equal-Cost Multi-Path (ECMP)| Standard shortest path |
| **Ingestion MTU** | **65,520 (Connected Mode)** | 9,000 (Jumbo Frames) | 1,500 (Standard) |

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        INFINIBAND FABRIC SRE DIAGNOSTIC MATRIX                                    |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| HCA port stuck in `Initializing`:  | OpenSM not running or    | Check Subnet Manager status:      |
| Physical link UP, but no traffic.  | LID allocation full.     | `systemctl status opensm`         |
|                                    |                          | Verify master OpenSM logs.        |
+------------------------------------+--------------------------+-----------------------------------+
| IPoIB throughput capped at 15 Gbps | Interface operating in   | Check IPoIB mode:                 |
| on 400 Gbps Quantum-2 adapter.     | Datagram mode (MTU 2044) | `cat /sys/class/net/ib0/mode`     |
|                                    | instead of Connected.    | Set `echo connected > .../mode`.  |
+------------------------------------+--------------------------+-----------------------------------+
| InfiniBand link experiences        | Dirty optical transceiver| Run hardware performance query:   |
| intermittent packet drop spikes.   | or damaged MPO-12 fiber. | `perfquery -C <port>`             |
|                                    |                          | Replace failing optical cable.    |
+------------------------------------+--------------------------+-----------------------------------+
| PKey mismatch error:               | OpenSM partitions.conf   | Query active PKey membership:     |
| "Permission denied during RDMA".   | missing node GUID.       | `smpquery pkey <lid>`             |
|                                    |                          | Add node GUID to partitions.conf. |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **OpenSM Service Active:** Master and standby OpenSM services running with Fat-Tree routing engine.
- [ ] **IPoIB Connected Mode:** `/sys/class/net/ib*/mode` confirms `connected` with MTU $65,520$.
- [ ] **Physical Links Trained:** `ibstat` asserts all ports operating at native link speed ($400\text{ Gbps}$ NDR).
- [ ] **Symbol Errors Zero:** `perfquery` validates zero symbol errors or link recovery events across all ports.
- [ ] **PKey Partitions Enforced:** Storage and compute traffic isolated via dedicated hardware PKeys.
