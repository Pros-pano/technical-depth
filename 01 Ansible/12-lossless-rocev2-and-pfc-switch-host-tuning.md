# Volume 12: Lossless RoCEv2 Network Tuning: PFC Priority 3, ECN & MTU 9000

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 12: Lossless Ethernet, Priority Flow Control (PFC), DCQCN Congestion & Jumbo Frames
====================================================================================================
```

---

## 1. Executive Intuition: The Lossless Ethernet Imperative

While native InfiniBand handles flow control through hardware credit tokens, RDMA over Converged Ethernet (RoCEv2) runs over standard IP/UDP packet fabrics. However, RoCEv2 hardware implementations are notoriously intolerant of packet loss:
1. **The Go-Back-N Cliff:** When a switch drops a single packet during an RDMA transfer, the receiving HCA discards all subsequent out-of-order packets. The sender must rewind and retransmit the entire window from the dropped packet. Bandwidth instantly collapses from $400\text{ Gbps}$ down to $<5\text{ Gbps}$.
2. **PFC Deadlocks & Pause Storms:** Priority Flow Control (PFC) pauses upstream switches when ingress buffers fill. If configured improperly across cyclic routing paths, pause frames propagate indefinitely, freezing all network traffic.
3. **Congestion Collapse:** Without Explicit Congestion Notification (ECN) and Data Center QCN (DCQCN), senders inject packets at line rate until buffers overflow.

```
+-----------------------------------------------------------------------------------------+
|                         ROCEv2 LOSSLESS MULTI-TIER COUPLING                             |
+-----------------------------------------------------------------------------------------+
| Traffic Separation:                                                                     |
| Priority 3 (Lossless Storage): [RoCEv2 NVMe-oF / GDS] -> Protected by PFC & DCQCN       |
| Priority 0 (Best Effort):     [K8s, SSH, DNS, Logs]   -> Standard droppable TCP/IP      |
|                                                                                         |
| Host Configuration Chain (Ansible Automated):                                           |
| 1. Netplan: MTU 9000 on physical roce0 - roce7 interfaces                               |
| 2. mlnx_qos: Set PFC mask 0,0,0,1,0,0,0,0 (Priority 3 only!)                           |
| 3. mlnx_qos: Map DSCP 26 -> Priority 3                                                  |
| 4. Kernel sysctl: Enforce BBR/DCTCP and high-concurrency socket buffers                 |
+-----------------------------------------------------------------------------------------+
```

Automating an enterprise AI RoCEv2 fabric requires Ansible to configure **Host Interface MTUs**, **Hardware PFC Queues**, and **DSCP-to-Priority Mappings**.

---

## 2. Lineage & Evolution of RoCEv2 Networking

```
   [1980s: Classical Lossy Ethernet]
                 |
           (Best-effort packet delivery; TCP sliding windows handle all congestion)
                 |
   [2008: Data Center Bridging (DCB / IEEE 802.1Qbb)]
                 |
           (Priority-based Flow Control introduced for Fibre Channel over Ethernet - FCoE)
                 |
   [2014: RoCEv2 (Infiniband over UDP 4791)]
                 |
           (IB verbs encapsulated in routable Layer-3 UDP packets)
                 |
   [2015: DCQCN (Data Center Quantized Congestion Notification)]
                 |
           (Hardware end-to-end feedback loop combining switch ECN marking with NIC CNPs)
                 |
   [2024: NVIDIA Spectrum-X & Adaptive Packet Spraying]
                 |
           (Lossless Ethernet augmented with fine-grained telemetry and packet spraying)
```

---

## 3. First-Principles Mathematics: PFC Headroom & BDP Buffer Sizing

To guarantee zero packet loss without triggering pause storms, the switch and NIC port ingress buffers must be sized to accommodate all in-flight bytes while a PFC pause frame travels across the wire.

### 3.1 Headroom Buffer Sizing Formula
Let:
- $C$ = Link bandwidth ($400\text{ Gbps} = 50\text{ GB/s} = 50 \times 10^9\text{ bytes/sec}$)
- $T_{\text{RTT}}$ = Round-trip propagation time across optical fiber ($T_{\text{RTT}} \approx 2 \times \frac{\text{Distance}}{2 \times 10^8\text{ m/s}}$)
- $T_{\text{reaction}}$ = Switch pause generation time + NIC response and drain time ($\sim 1.5\ \mu\text{s}$)
- $S_{\text{MTU}}$ = Maximum Transmission Unit ($9,216\text{ bytes}$ for Jumbo Frames)

$$\text{Buffer}_{\text{headroom}} = (C \times T_{\text{RTT}}) + (C \times T_{\text{reaction}}) + S_{\text{MTU}}$$

#### Concrete Numerical Calculation (100-meter Datacenter Run):
- $T_{\text{RTT}} = 1.0\ \mu\text{s}$
- Total reaction time $T_{\text{total}} = 1.0\ \mu\text{s} + 1.5\ \mu\text{s} = 2.5\ \mu\text{s}$

$$\text{Buffer}_{\text{headroom}} = (50 \times 10^9\text{ B/s} \times 2.5 \times 10^{-6}\text{ s}) + 9,216\text{ B}$$
$$\text{Buffer}_{\text{headroom}} = 125,000\text{ bytes} + 9,216\text{ bytes} \approx \mathbf{134.2\text{ Kilobytes}}$$

> **Switch Invariant:** Every 400 Gbps switch port must reserve at least **$140\text{ KB}$ of dedicated headroom buffer** for Priority 3. If headroom is undersized, packets drop and RoCEv2 throughput collapses.

---

## 4. Deep Architecture: DSCP-to-PFC Priority Mapping

In Layer-3 networks, Ethernet Priority Code Point (PCP, 3 bits) is lost when crossing IP routers. Therefore, Quality of Service (QoS) must be carried in the IP header's **Differentiated Services Code Point (DSCP, 6 bits)** field:

```
+-----------------------------------------------------------------------------+
|                      DSCP TO HARDWARE QUEUE MAPPING                         |
+-----------------------------------------------------------------------------+
|  Application Socket / Verbs (GPUDirect Storage, NCCL)                       |
|    |                                                                        |
|    v                                                                        |
|  Sets IP ToS / DSCP Field: DSCP 26 (Storage) or DSCP 46 (Compute)           |
|    |                                                                        |
|    v                                                                        |
|  Mellanox ConnectX HCA Hardware:                                            |
|    |-- Inspects incoming packet DSCP field (Value: 26)                      |
|    |-- Maps DSCP 26 -> Priority 3 (Lossless Storage Queue)                  |
|    |-- Applies PFC Flow Control rules strictly to Priority 3                |
|    +-- Passes Priority 0 packets (SSH, Logs) as Best-Effort (Droppable)    |
+-----------------------------------------------------------------------------+
```

---

## 5. Concrete Production Lab: Automated RoCEv2 Host Tuning Role

Below is an enterprise Ansible role that configures Netplan with MTU 9000 and deploys a systemd oneshot service that executes `mlnx_qos` to lock PFC Priority 3 and map DSCP 26 on all ConnectX interfaces.

### 5.1 Playbook Tasks (`tasks/main.yml`)
```yaml
---
- name: 1. Ensure Mellanox QoS utilities present
  ansible.builtin.apt:
    name: mlnx-tools
    state: present

- name: 2. Discover all Mellanox RoCEv2 Ethernet interfaces
  ansible.builtin.shell: >
    ls -l /sys/class/net/ | grep -E "mlx5_[0-9]" | awk '{print $9}'
  register: roce_interfaces_raw
  changed_when: false

- name: Parse interface list
  ansible.builtin.set_fact:
    roce_interfaces: "{{ roce_interfaces_raw.stdout_lines }}"

- name: 3. Configure Persistent Netplan MTU 9000 on RoCE Interfaces
  ansible.builtin.copy:
    dest: /etc/netplan/90-roce-mtu.yaml
    content: |
      network:
        version: 2
        ethernets:
      {% for iface in roce_interfaces %}
          {{ iface }}:
            mtu: 9000
      {% endfor %}
  notify: Apply Netplan

- name: 4. Deploy RoCEv2 Hardware QoS Tuning Script
  ansible.builtin.copy:
    dest: /usr/local/bin/tune-roce-qos.sh
    mode: '0755'
    content: |
      #!/usr/bin/env bash
      set -euo pipefail
      
      {% for iface in roce_interfaces %}
      echo "Configuring RoCEv2 QoS on {{ iface }}..."
      # Set MTU 9000
      ip link set dev {{ iface }} mtu 9000
      
      # Enable PFC strictly on Priority 3 (Storage Queue), disable all others
      mlnx_qos -i {{ iface }} --pfc 0,0,0,1,0,0,0,0
      
      # Map DSCP 26 to Priority 3
      mlnx_qos -i {{ iface }} --dscp2prio 26:3
      
      # Set Trust state to DSCP (trust L3 IP headers rather than L2 802.1p)
      mlnx_qos -i {{ iface }} --trust dscp
      {% endfor %}

- name: 5. Create Systemd Service to Enforce RoCE QoS at Boot
  ansible.builtin.copy:
    dest: /etc/systemd/system/roce-qos-tuning.service
    content: |
      [Unit]
      Description=Enforce Mellanox Lossless RoCEv2 QoS & PFC Tuning
      After=network-online.target
      Wants=network-online.target

      [Service]
      Type=oneshot
      ExecStart=/usr/local/bin/tune-roce-qos.sh
      RemainAfterExit=true

      [Install]
      WantedBy=multi-user.target
  notify: Reload Systemd

- name: 6. Enable and Execute RoCE QoS Tuning Service
  ansible.builtin.systemd:
    name: roce-qos-tuning
    state: started
    enabled: true
    daemon_reload: true

- name: 7. Audit Hardware PFC Counters Post-Tuning
  ansible.builtin.command: "mlnx_qos -i {{ item }}"
  loop: "{{ roce_interfaces }}"
  register: qos_audits
  changed_when: false
  failed_when: "'pfc: 0,0,0,1,0,0,0,0' not in qos_audits.stdout"

handlers:
  - name: Apply Netplan
    ansible.builtin.command: netplan apply

  - name: Reload Systemd
    ansible.builtin.systemd:
      daemon_reload: true
```

---

## 6. Comparative Transport Matrix

| Feature | Lossy TCP/IP | Standard RoCEv2 (Untuned) | Lossless RoCEv2 (Tuned) |
| :--- | :--- | :--- | :--- |
| **Packet Drop Policy** | Sliding Window Backoff | Hardware Go-Back-N Drop | **PFC Lossless Queue (Zero Drop)** |
| **Congestion Control** | CUBIC / BBR | None | **DCQCN (ECN + CNPs)** |
| **MTU Size** | 1,500 bytes | 1,500 bytes | **9,000 bytes (Jumbo Frames)** |
| **Effective Throughput** | ~85 Gbps (CPU bound) | ~15 Gbps (Drop collapse) | **~385 Gbps (96% Line Rate)** |
| **Tail Latency (p99)** | 1,500 microseconds | 20,000 microseconds | **12 microseconds** |

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        ROCEv2 NETWORK SRE DIAGNOSTIC MATRIX                                       |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| Cluster-wide latency spikes;       | PFC Pause Storm: Faulty  | Inspect pause counters:           |
| non-storage services time out.     | cable or NIC flooding    | `ethtool -S <iface> | grep pause` |
|                                    | PFC frames upstream.     | Enable switch PFC watchdog timers.|
+------------------------------------+--------------------------+-----------------------------------+
| NVMe-oF disconnects during large   | MTU mismatch: Host MTU   | Run trace path ping with DF bit:  |
| checkpoint writes.                 | 9000, but switch port    | `ping -M do -s 8972 <storage_ip>` |
|                                    | set to 1500 (blackhole). | Align MTU across all switches.    |
+------------------------------------+--------------------------+-----------------------------------+
| High CNP packet rates; RoCEv2      | Over-aggressive ECN      | Adjust switch ECN WRED thresholds:|
| throughput throttled below 50%.    | threshold causing false  | Raise `min_threshold` buffer limit|
|                                    | congestion backoff.      | to absorb transient bursts.       |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **PFC Enabled on Priority 3 Only:** Verified with `mlnx_qos -i <iface>` showing mask `0,0,0,1,0,0,0,0`.
- [ ] **DSCP 26 Mapped:** DSCP to priority mapping verified with `mlnx_qos --dscp2prio 26:3`.
- [ ] **Jumbo Frames Active:** `ip link show` confirms MTU 9000 across all high-speed interfaces.
- [ ] **Zero Buffer Drops:** `ethtool -S <iface>` confirms `rx_out_of_buffer` and `rx_discards` are zero.
- [ ] **Switch Watchdog Tuned:** Switch PFC watchdog configured with $200\text{ ms}$ timeout to prevent deadlocks.
