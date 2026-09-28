# Volume 13: Multus CNI, Secondary RDMA Networks & Macvlan Automation

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 13: Multi-Network Kubernetes, SR-IOV Device Plugins, NetworkAttachmentDefinition & RDMA Verbs
====================================================================================================
```

---

## 1. Executive Intuition: The Single-Interface Bottleneck

In standard Kubernetes clusters, every Pod receives exactly one network interface: `eth0`. This interface is attached to the primary cluster CNI (e.g. Cilium, Calico, or Flannel) which routes pod-to-pod traffic over a software overlay network (VXLAN, Geneve, or WireGuard).

For AI foundation model training, this architecture is fatal:
1. **Software Overlay Encapsulation Tax:** Pushing hundreds of gigabytes per second of NCCL gradient sync or GPUDirect Storage traffic through a software VXLAN tunnel consumes 100% of the host's CPU cores purely executing kernel packet headers and checksum calculations.
2. **Missing RDMA Hardware Verbs:** Standard Kubernetes overlay CNIs do not expose physical InfiniBand / RoCEv2 verbs (`/dev/infiniband/uverbs*`) into the container namespace.
3. **No Multi-Rail Separation:** An 8-GPU node contains 8 physical 400 Gbps network adapters. A single `eth0` interface cannot utilize the other 7 adapters, wasting $87.5\%$ of the cluster's network investment!

```
+-----------------------------------------------------------------------------------------+
|                  SINGLE CNI OVERLAY VS. MULTUS MULTI-RAIL ARCHITECTURE                  |
+-----------------------------------------------------------------------------------------+
| [Standard Kubernetes CNI (Calico/Cilium)]:                                              |
| GPU Pod [eth0 (VXLAN)] ---> Linux Kernel Overlay Router ---> 1 Physical NIC (12.5% BW)   |
|                                                                                         |
| [Multus Multi-Network CNI (Enterprise AI Architecture)]:                                |
| GPU Pod:                                                                                |
|   |-- eth0 (Primary CNI)       ---> Cluster DNS, Kube-API, Control Plane (Calico/Cilium)|
|   |-- net1 (Secondary RDMA)    ===> Physical ConnectX NIC 0 (400 Gbps Storage Fabric)   |
|   |-- net2 (Secondary RDMA)    ===> Physical ConnectX NIC 1 (400 Gbps Compute Rail 0)   |
|   |-- ...                                                                               |
|   +-- net8 (Secondary RDMA)    ===> Physical ConnectX NIC 7 (400 Gbps Compute Rail 6)   |
|                                                                                         |
| Result: Full 3.2 Terabits/sec hardware line rate directly inside GPU container!         |
+-----------------------------------------------------------------------------------------+
```

Modern AI Kubernetes infrastructure requires **Multus CNI**: a meta-plugin that allows Pods to attach multiple physical network interfaces directly via **SR-IOV** or **Macvlan / IPVlan**.

---

## 2. Lineage & Evolution of Multi-Network Containers

```
   [2015: Single Interface Docker]
                 |
           (Every container bounded to docker0 bridge with NAT port forwarding)
                 |
   [2016: Kubernetes CNI Specification]
                 |
           (Standardized CNI network plugins, but strictly one interface per pod)
                 |
   [2017: Intel SR-IOV Device Plugin]
                 |
           (Hardware PCIe Single Root I/O Virtualization for containers)
                 |
   [2018: Multus CNI (Intel / Red Hat / CNCF)]
                 |
           (Meta-CNI delegating to multiple secondary plugins simultaneously)
                 |
   [2023: NVIDIA Network Operator]
                 |
           (Automated orchestration of Multus, SR-IOV, MOFED, and RDMA device plugins)
```

---

## 3. First-Principles Mathematics: Overlay Encapsulation vs. Secondary RDMA

### 3.1 VXLAN Overhead & CPU Core Saturation
Consider a $100\text{ GB/s}$ ($800\text{ Gbps}$) tensor reduction stream crossing an overlay network:
- VXLAN adds a 50-byte header: Outer Ethernet (14B) + Outer IP (20B) + Outer UDP (8B) + VXLAN Header (8B).
- If standard MTU 1500 is used, the usable inner payload is reduced to $1,450\text{ bytes}$.
- Packet rate:
  $$\text{Packet Rate} = \frac{100 \times 10^9\text{ bytes/sec}}{1,450\text{ bytes}} \approx \mathbf{68,965,517\text{ packets/sec}}$$

Each packet processed by the Linux kernel network stack requires $\sim 150\text{ nanoseconds}$ of CPU handling (sk_buff allocation, route lookup, checksum calculation):

$$\text{CPU Processing Demand} = 68.96 \times 10^6\text{ pkts/s} \times 150 \times 10^{-9}\text{ s} \approx \mathbf{10.34\text{ CPU Cores 100\% Saturated}}$$

### 3.2 Secondary RDMA via Multus
With Multus attaching secondary ConnectX interfaces directly using hardware verbs and MTU 9000:
- Kernel is bypassed completely (Kernel Bypass / zero-copy DMA).
- Host CPU core consumption: **$0.0\text{ Cores}$**.
- Throughput: **$100\%$ hardware line rate**.

---

## 4. Deep Architecture: Multus Delegation & NetworkAttachmentDefinition

Multus acts as a multiplexer. It delegates interface provisioning to underlying CNI plugins based on Kubernetes Custom Resource Definitions (CRDs):

```
+-----------------------------------------------------------------------------+
|                     MULTUS CNI CALL DELEGATION FLOW                         |
+-----------------------------------------------------------------------------+
|  Kubelet (Creating GPU Training Pod)                                        |
|    |                                                                        |
|    | CNI ADD Request: /opt/cni/bin/multus                                   |
|    v                                                                        |
|  Multus Meta-Plugin:                                                        |
|    |                                                                        |
|    |-- 1. Calls Master CNI (e.g. Cilium / Calico) -> Injects eth0 (Overlay) |
|    |                                                                        |
|    |-- 2. Inspects Pod Annotation:                                          |
|    |      k8s.v1.cni.cncf.io/networks: rdma-storage-net                     |
|    |                                                                        |
|    |-- 3. Resolves NetworkAttachmentDefinition "rdma-storage-net"           |
|    |                                                                        |
|    +-- 4. Calls Secondary CNI (macvlan / sriov) -> Injects net1 (RDMA HCA)  |
+-----------------------------------------------------------------------------+
```

---

## 5. Concrete Production Lab: Automated Multus & RDMA Secondary Network Playbook

```yaml
---
# playbook: deploy_multus_rdma.yml
# Provisions Multus CNI, installs CNI plugins, and configures RDMA NetworkAttachmentDefinitions
- name: Orchestrate Multus Secondary RDMA Network Infrastructure
  hosts: kube_nodes
  become: true
  gather_facts: true
  tasks:
    - name: 1. Ensure CNI Plugins Directory Exists
      ansible.builtin.file:
        path: /opt/cni/bin
        state: directory
        mode: '0755'

    - name: 2. Download Standard CNI Reference Plugins (Macvlan, Host-Device, IPAM)
      ansible.builtin.get_url:
        url: "https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz"
        dest: /tmp/cni-plugins.tgz
        mode: '0644'

    - name: Extract CNI Reference Plugins
      ansible.builtin.unarchive:
        src: /tmp/cni-plugins.tgz
        dest: /opt/cni/bin/
        remote_src: true

    - name: 3. Download and Install Multus CNI Meta-Plugin
      ansible.builtin.get_url:
        url: "https://github.com/k8snetworkplumbingwg/multus-cni/releases/download/v4.0.2/multus-cni_4.0.2_linux_amd64.tar.gz"
        dest: /tmp/multus-cni.tar.gz
        mode: '0644'

    - name: Extract Multus CNI Binary
      ansible.builtin.unarchive:
        src: /tmp/multus-cni.tar.gz
        dest: /opt/cni/bin/
        remote_src: true
        extra_opts: ['--strip-components=1']

    - name: 4. Configure Multus CNI Daemon Configuration
      ansible.builtin.copy:
        dest: /etc/cni/net.d/00-multus.conf
        content: |
          {
            "cniVersion": "0.4.0",
            "name": "multus-cni-network",
            "type": "multus",
            "kubeconfig": "/etc/kubernetes/admin.conf",
            "delegates": [
              {
                "cniVersion": "0.4.0",
                "name": "primary-cni",
                "plugins": [
                  {
                    "type": "cilium-cni"
                  }
                ]
              }
            ]
          }

    - name: 5. Deploy RDMA Storage NetworkAttachmentDefinition (Master Node)
      run_once: true
      when: inventory_hostname in groups['kube_control_plane']
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: "k8s.cni.cncf.io/v1"
          kind: NetworkAttachmentDefinition
          metadata:
            name: rdma-storage-net
            namespace: default
          spec:
            config: '{
              "cniVersion": "0.4.0",
              "type": "macvlan",
              "master": "roce0",
              "mode": "bridge",
              "ipam": {
                "type": "whereabouts",
                "range": "10.200.0.0/19",
                "gateway": "10.200.0.1"
              }
            }'
```

### 5.1 Utilizing Secondary RDMA Network in Pod Specification
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-distributed-trainer
  annotations:
    k8s.v1.cni.cncf.io/networks: rdma-storage-net
spec:
  containers:
    - name: pytorch
      image: nvcr.io/nvidia/pytorch:24.04-py3
      resources:
        limits:
          nvidia.com/gpu: 8
          rdma/rdma_shared_device_a: 1
      command: ["torchrun", "train.py"]
```

---

## 6. Comparative Network Injection Matrix

| Architecture | Primary CNI (Overlay) | Host-Device Passthrough | Macvlan via Multus | SR-IOV via Multus |
| :--- | :--- | :--- | :--- | :--- |
| **Interface Name** | `eth0` | `net1` | `net1` | `net1` |
| **Hardware Isolation** | Software Bridge | Exclusive Physical Port| Shared L2 Mac | Hardware Virtual Function |
| **Throughput** | 15–40 Gbps | **400 Gbps** | **385 Gbps** | **395 Gbps** |
| **CPU Overhead** | High (Encapsulation) | **Zero (Kernel Bypass)**| **Zero** | **Zero** |
| **Pod Density** | Unlimited | 1 Pod per Physical Port| High | 127 Pods per Port |

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        MULTUS & SECONDARY NETWORK SRE DIAGNOSTIC MATRIX                           |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| Pod creation stuck in              | Multus failed to call    | Inspect Kubelet CNI logs:         |
| `ContainerCreating`.               | delegate plugin binary   | `journalctl -u kubelet -n 50`     |
|                                    | (binary missing in bin). | Verify `/opt/cni/bin/macvlan`.    |
+------------------------------------+--------------------------+-----------------------------------+
| `net1` present in Pod, but RDMA    | RDMA device plugin not   | Ensure device plugin running:     |
| verbs missing (`ibv_devinfo` empty)| running; `/dev/infiniband`| `kubectl get pods -n kube-system  |
|                                    | not mounted into pod.    |  -l app=k8s-rdma-device-plugin`   |
+------------------------------------+--------------------------+-----------------------------------+
| IP allocation conflict on `net1`:  | IPAM pool exhausted or   | Check Whereabouts IPAM CRD:       |
| "Whereabouts: Out of IP addresses"| stale IP allocation held.| `kubectl get overlappingrangeip-  |
|                                    |                          | reservations -A`                  |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **Multus Binary Installed:** `/opt/cni/bin/multus` present and executable across all cluster nodes.
- [ ] **NetworkAttachmentDefinition Deployed:** CRD configured with high-speed physical master interface (`roce0`).
- [ ] **Dual Interface Injected:** Test pod verifies both `eth0` (Cluster IP) and `net1` (RDMA Storage IP).
- [ ] **RDMA Verbs Accessible:** Inside test container, `ibv_devinfo -v` reports native physical HCA devices.
- [ ] **Hardware Line Rate Achieved:** Network benchmark on `net1` validates $>380\text{ Gbps}$ line rate.
