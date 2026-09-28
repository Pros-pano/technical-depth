# Volume 16: Kubernetes Bare-Metal Bootstrapping on DGX/HGX with Kubespray

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 16: High-Availability etcd Quorums, containerd CRI, Kubespray Engine & GPU Node Hardening
====================================================================================================
```

---

## 1. Executive Intuition: The Bare-Metal Kubernetes Chasm

Deploying Kubernetes on bare-metal AI supercomputers (NVIDIA DGX / HGX clusters) is fundamentally different from clicking a button on AWS EKS or Google GKE. On bare metal, there is no underlying cloud provider to provision control plane load balancers, manage etcd snapshots, or configure virtual VPC routing:
1. **The etcd Fsync Wall:** etcd is a write-ahead log (WAL) database using the Raft consensus algorithm. In a 1,000-node cluster with high churn, etcd requires sub-10ms disk fsync latency on dedicated NVMe drives. If disk latency spikes, leader elections flap and the API server crashes.
2. **Containerd GPU Runtime Coupling:** The Kubernetes container runtime interface (`containerd`) must be pre-configured with the `nvidia-container-runtime` and Container Device Interface (CDI) hooks before the `kubelet` daemon starts; otherwise, GPU pods fail with device discovery errors.
3. **Control Plane High Availability:** Control plane API traffic must be load-balanced across multiple master nodes using an internal Virtual IP (VIP) managed via **Keepalived** and **HAProxy**.

```
+-----------------------------------------------------------------------------------------+
|                        KUBESPRAY BARE-METAL TOPOLOGY                                    |
+-----------------------------------------------------------------------------------------+
| [Control Plane VIP: 10.200.0.100 (Keepalived / HAProxy)]                                |
|   |-- Master 01 (kube-apiserver + etcd member 1)                                        |
|   |-- Master 02 (kube-apiserver + etcd member 2)                                        |
|   +-- Master 03 (kube-apiserver + etcd member 3)                                        |
|                                                                                         |
| [Bare-Metal GPU Worker Nodes: 8x H100 / B200 Servers]                                   |
|   |-- kubelet (configured with fail-swap-on=false)                                      |
|   |-- containerd (CRI configured with default_runtime_name = "nvidia")                  |
|   |-- Primary CNI: Cilium / Calico (eBPF host routing)                                  |
|   +-- Secondary CNI: Multus (Injecting 400 Gbps RoCEv2 interfaces directly to pods)     |
+-----------------------------------------------------------------------------------------+
```

**Kubespray**—an enterprise, production-grade Ansible framework maintained by the Kubernetes SIGs—is the definitive industry tool for deploying self-healing, bare-metal Kubernetes clusters on physical hardware.

---

## 2. Lineage & Evolution of Bare-Metal Kubernetes

```
   [2015: Manual CoreOS & Flannel]
                 |
           (Manual bash scripts generating TLS certificates and static systemd units)
                 |
   [2016: kubeadm Utility]
                 |
           (Official Kubernetes bootstrapping CLI; handles certs and init, but single-node)
                 |
   [2017: Kubespray Project (Kubernetes SIGs)]
                 |
           (Enterprise Ansible roles coordinating HA etcd, VIPs, CNI, and multi-OS support)
                 |
   [2021: Lightweight Edge Distributions (K3s / RKE2)]
                 |
           (Packaged binary K8s with embedded sqlite/etcd; ideal for standalone DGX Spark)
                 |
   [2024: Automated GPU Operator Integration]
                 |
           (Kubespray orchestrating CDI, NVLink fabrics, and Dynamic Resource Allocation)
```

---

## 3. First-Principles Mathematics: etcd Raft Quorum & Write Latency

etcd uses the **Raft consensus algorithm** to replicate state across cluster masters.

### 3.1 Quorum Size & Fault Tolerance
Let $N$ be the number of etcd members. The cluster requires an absolute majority (Quorum $Q$) to commit transactions:

$$Q = \left\lfloor \frac{N}{2} \right\rfloor + 1$$

$$\text{Fault Tolerance } F = N - Q = \left\lfloor \frac{N - 1}{2} \right\rfloor$$

```
+-----------------------------------------------------------------------------------------+
|                         etcd RAFT QUORUM & FAULT CAPACITY                               |
+-------------------+--------------------+-----------------------+------------------------+
| Cluster Size (N)  | Quorum Required (Q)| Max Faults Tolerated  | Operational Recommendation|
+-------------------+--------------------+-----------------------+------------------------+
| 1                 | 1                  | 0                     | Development / Test only|
| 3                 | 2                  | 1                     | Standard Production    |
| 5                 | 3                  | 2                     | Hyperscale Production  |
| 7                 | 4                  | 3                     | Extreme Scale Clusters |
+-------------------+--------------------+-----------------------+------------------------+
```

> **Why Even Numbers (e.g. N=4) are Forbidden:** A 4-node cluster requires $Q = \lfloor 4/2 \rfloor + 1 = 3$ nodes for quorum. It can tolerate only $4 - 3 = 1$ failure. A 3-node cluster also tolerates 1 failure. Adding a 4th node adds network partition vulnerability without increasing fault tolerance! Always use **3 or 5** members.

### 3.2 Disk fsync Latency Constraint
Every etcd write requires synchronous flush to non-volatile storage via the `fdatasync()` syscall:

$$T_{\text{commit}} = T_{\text{network\_rtt}} + T_{\text{fsync\_disk}}$$

If $T_{\text{fsync\_disk}} > 10\text{ ms}$, etcd logs: `etcdserver: read-only range request took too long`.
If disk latency exceeds the Raft heartbeat interval ($100\text{ ms}$), etcd triggers catastrophic leader re-elections, freezing Kube-API requests.
- **Ansible Invariant:** etcd data directory (`/var/lib/etcd`) must be mounted on dedicated PCIe NVMe drives, isolated from OS logs and container overlays.

---

## 4. Deep Architecture: Kubespray Configuration Hierarchy

Kubespray organizes configuration through structured group variables:

```
inventory/ai-cluster/
├── hosts.yaml              # Node IPs, control-plane vs worker assignments
└── group_vars/
    ├── all/
    │   ├── all.yml         # Global proxy, VIP, and upstream repository mirrors
    │   └── containerd.yml  # Container runtime flags and NVIDIA hooks
    └── k8s_cluster/
        ├── k8s-cluster.yml # Kubernetes version, CNI choice (Cilium/Calico), VIP
        └── addons.yml      # Metrics-server, local-path-provisioner
```

---

## 5. Concrete Production Lab: Automated Kubespray Deployment Playbook

Below is an enterprise Ansible configuration deploying a 3-node HA control plane with containerd GPU hooks enabled.

### 5.1 Kubespray Cluster Configuration (`group_vars/k8s_cluster/k8s-cluster.yml`)
```yaml
---
# Kubernetes Core Settings
kube_version: v1.29.3
kube_network_plugin: cilium
kube_service_addresses: 10.233.0.0/18
kube_pods_subnet: 10.233.64.0/18

# Load Balancer VIP for API Server
loadbalancer_apiserver_localhost: true
loadbalancer_apiserver:
  address: 10.200.0.100
  port: 6443

# High-Performance Node Hardening
kubelet_cgroup_driver: systemd
kubelet_status_update_frequency: 5s
kubelet_node_status_report_frequency: 1m

# GPU Containerd Runtime Integration
container_manager: containerd
containerd_default_runtime: "nvidia"
containerd_additional_runtimes:
  nvidia:
    type: "io.containerd.runc.v2"
    engine: ""
    root: ""
    options:
      BinaryName: "/usr/bin/nvidia-container-runtime"
```

### 5.2 Kubespray Execution Wrapper Playbook (`deploy_k8s_baremetal.yml`)
```yaml
---
- name: Phase 1 - Pre-flight Bare Metal OS Preparation
  hosts: k8s_cluster
  become: true
  tasks:
    - name: 1. Disable Linux Swap (Required by Kubernetes)
      ansible.builtin.command: swapoff -a
      changed_when: false

    - name: Remove swap entry from fstab
      ansible.builtin.lineinfile:
        path: /etc/fstab
        regexp: '\sswap\s'
        state: absent

    - name: 2. Enable Required Linux Kernel Networking Modules
      community.general.modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - overlay
        - br_netfilter

    - name: 3. Configure Kubernetes Kernel Sysctl Parameters
      ansible.posix.sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        state: present
        sysctl_file: /etc/sysctl.d/99-kubernetes-cri.conf
        reload: true
      loop:
        - { name: "net.bridge.bridge-nf-call-iptables", value: "1" }
        - { name: "net.bridge.bridge-nf-call-ip6tables", value: "1" }
        - { name: "net.ipv4.ip_forward", value: "1" }

- name: Phase 2 - Trigger Kubespray Automated Orchestration
  import_playbook: kubespray/cluster.yml
```

---

## 6. Comparative Bare-Metal Orchestration Matrix

| Feature | Managed Cloud K8s (EKS/GKE) | Manual `kubeadm` | Kubespray (Ansible) |
| :--- | :--- | :--- | :--- |
| **Hardware Control** | Zero (Cloud abstracted) | Full | **Full Bare-Metal Control** |
| **High Availability etcd** | Handled by Cloud | Manual config | **Automated Raft Clustering** |
| **GPU / CDI Integration** | Basic AMI | Manual hooks | **Pre-configured in containerd** |
| **Multi-NIC / Multus** | Challenging | Manual | **Native Supported Addon** |
| **Scale Limits** | Cloud Quota bounded | Operational burden | **1,000+ Bare-Metal Nodes** |

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        BARE-METAL KUBERNETES SRE DIAGNOSTIC MATRIX                                |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| `kubelet` fails to start:          | Linux Swap active on     | Disable swap immediately:         |
| "Running with swap on is not supp".| node.                    | `swapoff -a && sed -i '/swap/d'   |
|                                    |                          |  /etc/fstab`                      |
+------------------------------------+--------------------------+-----------------------------------+
| `etcd` logs: "took too long to     | High disk I/O latency on | Benchmark etcd disk latency:      |
| execute fdatasync".                | storage drive.           | `fio --name=etcd --rw=write --bs=4k|
|                                    |                          |  --size=1G --direct=1`            |
+------------------------------------+--------------------------+-----------------------------------+
| Kube-API server VIP unreachable;   | Keepalived split-brain or| Check keepalived VRRP status:     |
| kubectl commands time out.         | HAProxy service dead.    | `systemctl status keepalived`     |
|                                    |                          | Check local VIP binding: `ip a`   |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **Swap Completely Disabled:** `free -m` reports 0 MB swap across all nodes.
- [ ] **etcd Quorum Verified:** `etcdctl endpoint health` confirms healthy consensus across all 3/5 members.
- [ ] **Control Plane VIP Functional:** `kubectl cluster-info` reaches API server across load balancer VIP.
- [ ] **NVIDIA Containerd Runtime Active:** containerd configured with `default_runtime_name = "nvidia"`.
- [ ] **Overlay Networking Functional:** Pod-to-Pod ping across different nodes validates CNI routing.
