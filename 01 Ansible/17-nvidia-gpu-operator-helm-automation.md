# Volume 17: NVIDIA GPU Operator & Network Operator Helm Automation

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 17: GPU Operator Helm Values, Driver DaemonSets, GFD Node Labeling & Network Operator
====================================================================================================
```

---

## 1. Executive Intuition: The Containerized Driver Architecture

In enterprise Kubernetes clusters, manually managing GPU drivers, CUDA libraries, and monitoring daemons across worker nodes creates configuration drift. The **NVIDIA GPU Operator** automates the entire GPU lifecycle declaratively using Kubernetes Custom Resources and Helm:

Instead of installing packages directly into the host OS, the GPU Operator deploys a coordinated set of DaemonSets:
1. **NVIDIA Driver Container:** Compiles or loads pre-compiled kernel modules (`nvidia.ko`, `nvidia-uvm.ko`) into the host kernel from inside a privileged container.
2. **NVIDIA Container Toolkit Container:** Injects `nvidia-ctk` and configures the host container runtime.
3. **NVIDIA Device Plugin:** Advertises `nvidia.com/gpu` allocatable resources to the Kube-API server.
4. **NVIDIA GPU Feature Discovery (GFD):** Auto-discovers GPU architecture and applies labels (e.g., `nvidia.com/gpu.family: hopper`, `nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3`).
5. **NVIDIA DCGM Exporter:** Exposes Prometheus metrics on port 9400.
6. **NVIDIA Fabric Manager DaemonSet:** Configures NVSwitch crossbars for SXM nodes.

```
+-----------------------------------------------------------------------------------------+
|                        NVIDIA GPU OPERATOR ARCHITECTURE                                 |
+-----------------------------------------------------------------------------------------+
| Ansible Control Node                                                                    |
|   |-- Deploys Helm Chart: nvidia/gpu-operator                                           |
|   +-- Generates Values File: custom-gpu-operator-values.yaml                            |
|                                                                                         |
| Kubernetes Cluster Target                                                               |
|   |-- ClusterPolicy CRD: gpu-cluster-policy                                             |
|   |     |-- daemonset/nvidia-driver-daemonset (Loads kmods into Linux kernel)           |
|   |     |-- daemonset/nvidia-container-toolkit-daemonset (Configures containerd CDI)    |
|   |     |-- daemonset/nvidia-device-plugin-daemonset (Exposes nvidia.com/gpu: 8)        |
|   |     |-- daemonset/gpu-feature-discovery (Applies node labels for scheduling)        |
|   |     +-- daemonset/nvidia-dcgm-exporter (Prometheus port 9400)                       |
+-----------------------------------------------------------------------------------------+
```

Automating the GPU Operator via Ansible guarantees reproducible, zero-touch deployment across thousands of nodes.

---

## 2. Lineage & Evolution of Kubernetes GPU Provisioning

```
   [2017: Static Kubernetes Device Plugin]
                 |
           (Single DaemonSet registering GPUs; required manual host driver pre-installation)
                 |
   [2019: NVIDIA GPU Operator v1.0]
                 |
           (Helm-driven operator automating drivers, toolkit, and device plugin)
                 |
   [2021: NVIDIA Network Operator]
                 |
           (Automates MOFED, SR-IOV device plugin, and Multus CNI for RDMA fabrics)
                 |
   [2024: Dynamic Resource Allocation (DRA) & CDI Standard]
                 |
           (K8s DRA replacing static integer counting with fine-grained GPU capability claims)
```

---

## 3. First-Principles Mathematics: Host Driver vs. Containerized Driver

When deploying the GPU Operator, infrastructure architects must choose between two distinct execution models:

```
+-----------------------------------------------------------------------------------------+
|                  HOST-INSTALLED DRIVER VS. CONTAINERIZED DRIVER                         |
+-----------------------+----------------------------------+------------------------------+
| Attribute             | Host-Installed Driver            | Operator Driver Container    |
+-----------------------+----------------------------------+------------------------------+
| Driver Source         | OS Package Manager (apt / dnf)   | Privileged Docker Container  |
| Node Reboot Required  | Yes (on driver upgrade)          | No (Kmod unloaded/reloaded)  |
| Boot Time             | Instantaneous (~5 seconds)       | Delayed (pulls 2GB image)    |
| Hermetic Upgrades     | Node-by-node maintenance drain   | Declarative Helm values patch|
| Secure Boot Support   | Standard MOK signing             | Requires custom secret certs |
| Production Fit        | **Mission-Critical Pre-Training**| **Cloud-Native / Dynamic**   |
+-----------------------+----------------------------------+------------------------------+
```

> **Production Recommendation:** For foundation model pre-training clusters running tightly coupled InfiniBand/NVSwitch fabrics, deploy **Host-Installed Drivers** (configured via Volume 07) and configure the GPU Operator with `driver.enabled=false`. This eliminates container startup delays and ensures 100% deterministic kernel module loading at system boot.

---

## 4. Concrete Production Lab: Automated GPU Operator Helm Deployment Playbook

```yaml
---
# playbook: deploy_gpu_operator.yml
# Provisions NVIDIA GPU Operator via Helm with customized cluster policy
- name: Orchestrate NVIDIA GPU Operator via Helm
  hosts: kube_control_plane[0]
  become: true
  gather_facts: false
  tasks:
    - name: 1. Add NVIDIA Official Helm Repository
      kubernetes.core.helm_repository:
        name: nvidia
        repo_url: "https://helm.ngc.nvidia.com/nvidia"

    - name: 2. Deploy Tuned GPU Operator Values Template
      ansible.builtin.copy:
        dest: /tmp/gpu-operator-values.yaml
        content: |
          # Production GPU Operator Values for DGX/HGX SuperPOD
          operator:
            defaultRuntime: containerd

          # Disable driver container if host driver pre-installed (Recommended for SXM)
          driver:
            enabled: false

          toolkit:
            enabled: true

          devicePlugin:
            enabled: true
            arguments:
              - "--pass-device-specs=true"

          dcgmExporter:
            enabled: true
            serviceMonitor:
              enabled: true

          gfd:
            enabled: true

          # Enable Fabric Manager for SXM NVSwitches
          fabricManager:
            enabled: false  # Pre-installed on host via Volume 07

          # Enable GPUDirect Storage (GDS) Driver
          gds:
            enabled: true

          node-feature-discovery:
            worker:
              tolerations:
                - key: "node-role.kubernetes.io/master"
                  operator: "Exists"
                  effect: "NoSchedule"

    - name: 3. Install or Upgrade NVIDIA GPU Operator via Helm
      kubernetes.core.helm:
        name: gpu-operator
        chart_ref: nvidia/gpu-operator
        release_namespace: gpu-operator
        create_namespace: true
        values_files:
          - /tmp/gpu-operator-values.yaml
        state: present
        wait: true
        timeout: 10m

    - name: 4. Wait for Node GPU Capacity Registration
      kubernetes.core.k8s_info:
        kind: Node
      register: node_info
      retries: 20
      delay: 15
      until: >
        node_info.resources |
        selectattr('status.allocatable', 'defined') |
        selectattr('status.allocatable["nvidia.com/gpu"]', 'defined') |
        list | length > 0

    - name: 5. Display Cluster Allocatable GPU Inventory
      ansible.builtin.debug:
        msg: "Node {{ item.metadata.name }} has {{ item.status.allocatable['nvidia.com/gpu'] }} allocatable GPUs"
      loop: "{{ node_info.resources }}"
      when: item.status.allocatable['nvidia.com/gpu'] is defined
```

---

## 5. Comparative Orchestration Matrix

| Capability | Bare-Metal Scripting | Standalone DaemonSets | NVIDIA GPU Operator |
| :--- | :--- | :--- | :--- |
| **Lifecycle Management** | Manual upgrades | Manual YAML edits | **Automated Helm Rollout** |
| **Hardware Labeling** | Manual `kubectl label`| None | **Automated GFD Discovery** |
| **MIG Dynamic Slicing** | Manual CLI slicing | Not supported | **Automated MIG Config CRD** |
| **Prometheus Exporter** | Separate daemon | Separate DaemonSet | **Integrated ServiceMonitor**|
| **Multi-Cluster Uniformity**| Low | Moderate | **100% Declarative Policy** |

---

## 6. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        GPU OPERATOR SRE DIAGNOSTIC MATRIX                                         |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| Driver DaemonSet in CrashLoopBackOff| Driver container trying  | Inspect pod logs:                 |
| with "kernel module already loaded"| to load kmod when host   | `kubectl logs -n gpu-operator     |
|                                    | driver already active.   |  -l app=nvidia-driver-daemonset`  |
|                                    |                          | Set `driver.enabled=false`.       |
+------------------------------------+--------------------------+-----------------------------------+
| Nodes report 0 allocatable GPUs in | Device Plugin pod hung   | Restart Device Plugin DaemonSet:  |
| `kubectl describe node`.           | or cannot access CDI.    | `kubectl rollout restart ds       |
|                                    |                          |  -n gpu-operator nvidia-device-..`|
+------------------------------------+--------------------------+-----------------------------------+
| GFD fails to label nodes:          | Node Feature Discovery   | Check NFD worker daemonset:       |
| `nvidia.com/gpu.product` missing.  | (NFD) worker not running.| `kubectl get ds -n gpu-operator   |
|                                    |                          |  -l app.kubernetes.io/name=node-..`|
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 7. Verification & Architectural Synthesis Checklist

- [ ] **Helm Release Succeeded:** `helm status -n gpu-operator gpu-operator` reports `STATUS: deployed`.
- [ ] **All DaemonSets Ready:** Device plugin, GFD, and DCGM exporter report 100% desired/ready pods.
- [ ] **Node Allocatable Validated:** `kubectl get nodes -o json` confirms `nvidia.com/gpu: 8` on every GPU worker.
- [ ] **GFD Labels Applied:** Nodes labeled with exact GPU architecture (`nvidia.com/gpu.family: hopper`).
- [ ] **Prometheus Scraping Active:** ServiceMonitor scrapes GPU metrics into cluster Prometheus.
