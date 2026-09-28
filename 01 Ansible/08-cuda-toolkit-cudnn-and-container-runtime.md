# Volume 08: CUDA Toolkit, cuDNN & Container Device Interface (CDI) Automation

```
====================================================================================================
MODULE 08: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 08: CUDA Toolkit 12.x, cuDNN, Containerd Runtime Hooks & Container Device Interface (CDI)
====================================================================================================
```

---

## 1. Executive Intuition: The OCI Container Boundary

Modern distributed training jobs (PyTorch DDP, Megatron-Core, DeepSeek, vLLM) run almost exclusively inside OCI container runtimes (`containerd`, `CRI-O`, `Docker`). However, standard Linux containers are completely isolated from physical hardware by default.

Bridging the physical GPU hardware into an isolated container namespace requires a multi-stage software bridge:
1. **CUDA Toolkit & cuDNN:** Provides user-space compilation headers (`cuda.h`), runtime libraries (`libcudart.so`), and deep learning primitives (`libcudnn.so`).
2. **NVIDIA Container Toolkit (`nvidia-container-toolkit`):** Provides the CLI tool `nvidia-ctk` and the low-level OCI pre-start hooks.
3. **Container Device Interface (CDI):** The modern, open-standard mechanism (spearheaded by the CNCF) that replaces legacy runtime hooks. CDI generates static, declarative YAML specifications (`/etc/cdi/nvidia.yaml`) detailing exact device nodes (`/dev/nvidia0`, `/dev/nvidia-uvm`, `/dev/nvidiactl`) and driver libraries for deterministic injection into containers.

```
+-----------------------------------------------------------------------------------------+
|                        CONTAINER GPU INJECTION EVOLUTION                                |
+-----------------------------------------------------------------------------------------+
| [Legacy nvidia-docker v1/v2 (Deprecated)]:                                              |
| Containerd -> Custom Binary Wrapper -> Dynamic Library Mounting -> Fragile, slow       |
|                                                                                         |
| [Modern Container Device Interface - CDI (Production Standard)]:                        |
| Container Engine (K8s/containerd) reads declarative YAML: /etc/cdi/nvidia.yaml          |
| Injects exact device nodes, IPC capabilities, and NUMA-aligned NICs with zero runtime overhead!|
+-----------------------------------------------------------------------------------------+
```

---

## 2. Lineage & Evolution of Containerized GPU Runtimes

```
   [2016: nvidia-docker v1]
                 |
           (Custom daemon wrapper; mounted host driver libraries directly into container)
                 |
   [2018: nvidia-docker v2 & nvidia-container-runtime]
                 |
           (Integrated with runc via OCI prestart hooks; required Docker daemon.json edits)
                 |
   [2021: NVIDIA Container Toolkit (nvidia-ctk)]
                 |
           (Unified toolkit supporting containerd, CRI-O, and Docker seamlessly)
                 |
   [2023: Container Device Interface (CDI Specification)]
                 |
           (CNCF open standard for hardware device injection; eliminates runtime hacks)
```

---

## 3. First-Principles Mathematics: CUDA Compatibility Matrix

When training foundation models, the CUDA Driver API and CUDA Runtime API adhere to a strict version inequality:

$$V_{\text{Installed Driver}} \ge V_{\text{Compiled CUDA Toolkit}}$$

```
+-----------------------------------------------------------------------------------------+
|                        CUDA FORWARD & BACKWARD COMPATIBILITY                            |
+-----------------------+-----------------------+--------------------+--------------------+
| CUDA Toolkit Version  | Minimum Driver Version| Backward Compatible| Forward Compatible |
+-----------------------+-----------------------+--------------------+--------------------+
| CUDA 12.4             | >= 550.54.14          | YES                | YES (via libcuda)  |
| CUDA 12.2             | >= 535.54.03          | YES                | YES (via libcuda)  |
| CUDA 11.8             | >= 520.61.05          | YES                | YES                |
+-----------------------+-----------------------+--------------------+--------------------+
```

### 3.1 Forward Compatibility Mechanics
If an older driver (e.g. `535.x`) is running on host bare metal, but a container requires CUDA 12.4 features:
- Ansible can deploy the **NVIDIA CUDA Forward Compatibility Package** (`cuda-compat-12-4`).
- At container startup, the forward compatibility library intercepts CUDA Driver API calls, emulating newer driver capabilities without upgrading the host kernel module!

---

## 4. Deep Architecture: Container Device Interface (CDI) Specification

The CDI specification (`/etc/cdi/nvidia.yaml`) explicitly defines hardware device endpoints for OCI runtimes:

```yaml
# /etc/cdi/nvidia.yaml (Generated via nvidia-ctk cdi generate)
cdiVersion: "0.5.0"
kind: "nvidia.com/gpu"
devices:
  - name: "0"
    containerEdits:
      deviceNodes:
        - path: "/dev/nvidia0"
        - path: "/dev/nvidiactl"
        - path: "/dev/nvidia-uvm"
        - path: "/dev/nvidia-uvm-tools"
  - name: "all"
    containerEdits:
      deviceNodes:
        - path: "/dev/nvidia0"
        - path: "/dev/nvidia1"
        - path: "/dev/nvidiactl"
        - path: "/dev/nvidia-uvm"
```

To run a container using CDI without Docker daemon wrappers:
```bash
podman run --device nvidia.com/gpu=all --rm nvcr.io/nvidia/pytorch:24.04-py3 nvidia-smi
```

---

## 5. Concrete Production Lab: Automated Container Toolkit & CDI Playbook

```yaml
---
# playbook: container_runtime_setup.yml
# Installs NVIDIA Container Toolkit, configures containerd, and generates CDI specs
- name: Configure Enterprise NVIDIA Container Runtime & CDI
  hosts: gpu_nodes
  become: true
  gather_facts: true
  tasks:
    - name: 1. Add NVIDIA Container Toolkit Repository GPG Key
      ansible.builtin.get_url:
        url: https://nvidia.github.io/libnvidia-container/gpgkey
        dest: /etc/apt/trusted.gpg.d/nvidia-container-toolkit.asc
        mode: '0644'

    - name: 2. Add NVIDIA Container Toolkit APT Repository
      ansible.builtin.apt_repository:
        repo: "deb [signed-by=/etc/apt/trusted.gpg.d/nvidia-container-toolkit.asc] https://nvidia.github.io/libnvidia-container/stable/deb/$(ARCH) /"
        state: present
        filename: nvidia-container-toolkit

    - name: 3. Install NVIDIA Container Toolkit
      ansible.builtin.apt:
        name:
          - nvidia-container-toolkit
          - containerd
        state: present
        update_cache: true

    - name: 4. Configure containerd with NVIDIA Runtime
      ansible.builtin.command: nvidia-ctk runtime configure --runtime=containerd
      notify: Restart containerd

    - name: 5. Generate Container Device Interface (CDI) Specification
      ansible.builtin.command: nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
      args:
        creates: /etc/cdi/nvidia.yaml

    - name: 6. Configure Docker daemon.json if Docker is installed
      ansible.builtin.command: nvidia-ctk runtime configure --runtime=docker
      notify: Restart Docker
      ignore_errors: true

    - name: 7. Validate Container GPU Access
      ansible.builtin.command: ctr run --rm --gpus 0 docker.io/nvidia/cuda:12.4.1-base-ubuntu22.04 test nvidia-smi
      register: ctr_test
      changed_when: false
      retries: 3
      delay: 5
      until: ctr_test.rc == 0

  handlers:
    - name: Restart containerd
      ansible.builtin.systemd:
        name: containerd
        state: restarted

    - name: Restart Docker
      ansible.builtin.systemd:
        name: docker
        state: restarted
```

---

## 6. Comparative Container GPU Runtime Matrix

| Feature | Legacy `nvidia-docker2` | NVIDIA Container Toolkit (Default) | Container Device Interface (CDI) |
| :--- | :--- | :--- | :--- |
| **Standardization** | Proprietary wrapper | OCI Prestart Hooks | **CNF Open Standard (v1alpha1)**|
| **Runtime Independence** | Docker-only | Docker, containerd, CRI-O | **Any OCI-compliant runtime** |
| **MIG Slicing Injection**| Fragile environment vars | Device UUID parsing | **Direct declarative device node**|
| **Startup Overhead** | High (~250ms) | Moderate (~80ms) | **Sub-millisecond (~1ms)** |
| **Kubernetes Support** | Deprecated | Standard (via GPU Operator)| **Next-Gen K8s Dynamic Allocation**|

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        CONTAINER RUNTIME SRE DIAGNOSTIC MATRIX                                    |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| `docker run --gpus all` fails:     | Docker daemon not        | Run configuration command:        |
| "could not select device driver".  | configured with toolkit. | `nvidia-ctk runtime configure     |
|                                    |                          |  --runtime=docker && systemctl    |
|                                    |                          |  restart docker`                  |
+------------------------------------+--------------------------+-----------------------------------+
| Container fails with: "failed to   | `/dev/nvidia-uvm` node   | Re-create device nodes via:       |
| initialize NVML: Unknown Error".   | missing from host.       | `nvidia-modprobe -u -c=0`         |
+------------------------------------+--------------------------+-----------------------------------+
| CDI device not found:              | CDI spec outdated after  | Re-generate CDI YAML spec:        |
| `nvidia.com/gpu=0: unknown device`.| GPU reboot or MIG slice. | `nvidia-ctk cdi generate          |
|                                    |                          |  --output=/etc/cdi/nvidia.yaml`   |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **NVIDIA Container Toolkit Installed:** `nvidia-ctk --version` verified.
- [ ] **containerd Runtime Configured:** `nvidia-container-runtime` set in `/etc/containerd/config.toml`.
- [ ] **CDI Specification Generated:** `/etc/cdi/nvidia.yaml` contains all physical GPUs and unified memory device paths.
- [ ] **Container Smoke Test Passed:** Container execution tested with `ctr run` or `docker run --gpus all`.
- [ ] **MIG Device Nodes Exposed:** Multi-Instance GPU partitions registered in CDI device lists.
