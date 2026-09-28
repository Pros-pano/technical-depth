# Volume 06: Bare-Metal OS Provisioning, Redfish API, IOMMU & Kernel Hugepages

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 06: Out-of-Band BMC Automation, Redfish REST APIs, Kernel Parameters & 1GB Hugepages
====================================================================================================
```

---

## 1. Executive Intuition: The Out-of-Band Foundation

An AI server (such as an NVIDIA DGX H100 or HGX B200) cannot be configured like a standard cloud virtual machine. Before the Linux operating system boots, dozens of critical physical, electrical, and bus-level parameters must be precisely aligned in the server's BIOS/UEFI:
1. **IOMMU Passthrough:** If `intel_iommu=on iommu=pt` or `amd_iommu=on` is not declared in the kernel, GPUDirect RDMA and GPUDirect Storage transactions will be trapped by hypervisor translation layers, degrading network and storage throughput by up to $60\%$.
2. **PCIe Power Throttling (ASPM):** Active State Power Management (ASPM) powers down idle PCIe lanes to save watts. In an AI cluster, transient idle periods during gradient descent sync cause PCIe links to drop into low-power states, introducing millisecond wake-up latencies that stall all-reduce barriers.
3. **Memory TLB Thrashing:** Managing 2 Terabytes of host memory using standard $4\text{ KB}$ pages requires millions of page table entries, thrashing the CPU Translation Lookaside Buffer (TLB). Pre-allocating **1 GB Hugepages** or **2 MB Hugepages** is mandatory for high-speed DMA transfers.

```
+-----------------------------------------------------------------------------------------+
|                    THE BARE-METAL PROVISIONING HIERARCHY                                |
+-----------------------------------------------------------------------------------------+
| [Out-of-Band Management: BMC (Baseboard Management Controller)]                         |
|   - Protocol: DMTF Redfish REST API over HTTPS (Port 443)                               |
|   - Ansible: community.general.redfish_config / redfish_command                         |
|   - Configuration: Enable SR-IOV, Disable PCIe ASPM, Lock CPU Performance Governor       |
|                                                                                         |
| [Pre-Boot Network Phase: PXE / iPXE / UEFI HTTP Boot]                                   |
|   - Automated OS Installation (Ubuntu 22.04/24.04 Server LTS or Rocky Linux 9)          |
|                                                                                         |
| [In-Band Kernel Hardening: Linux OS Level]                                              |
|   - GRUB Cmdline: intel_iommu=on iommu=pt pcie_aspm=off processor.max_cstate=0          |
|   - Memory: Allocate 512 GB of 2MB Hugepages (vm.nr_hugepages = 262144)                 |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Lineage & Evolution of Bare-Metal Provisioning

```
   [1998: IPMI v1.0 / v2.0 (RMCP+)]
                 |
           (Raw UDP packets on port 623; complex hex sensors; insecure cipher suite 0)
                 |
   [2015: DMTF Redfish Specification]
                 |
           (RESTful JSON over HTTPS; vendor-neutral schema for chassis, BIOS, and power)
                 |
   [2018: Ansible Redfish Collection]
                 |
           (community.general.redfish: Declarative Ansible modules managing BMCs)
                 |
   [2024: Open Compute Project (OCP) OpenBMC]
                 |
           (Linux-based BMC firmware running native REST/gRPC interfaces on DGX systems)
```

---

## 3. First-Principles Mathematics: Page Table Footprint & Hugepages Physics

In an enterprise GPU server equipped with $2\text{ TB}$ ($2,048\text{ GB}$) of host DRAM:

### 3.1 Standard 4 KB Page Table Overhead
The number of virtual-to-physical memory translations required:

$$N_{\text{pages}} = \frac{2 \times 1024^3\text{ KB}}{4\text{ KB}} = 536,870,912\text{ pages}$$

Each 64-bit Page Table Entry (PTE) consumes 8 bytes. In a 4-level x86-64 page table structure (PML4 $\to$ PDPT $\to$ PD $\to$ PT), the total physical memory consumed purely by kernel page tables is:

$$\text{PTE Memory} = 536,870,912 \times 8\text{ bytes} \approx 4,294,967,296\text{ bytes} \approx \mathbf{4.0\text{ Gigabytes}}$$

Furthermore, modern CPU L1/L2 TLBs can cache only $1,024 - 2,048$ entries ($4\text{ MB} - 8\text{ MB}$ total reach). Every GPU DMA access outside this reach incurs a high-latency **4-level Page Table Walk** across host memory buses ($\sim 80\text{ ns}$).

### 3.2 Sizing 2 MB and 1 GB Hugepages
With **2 MB Hugepages**:
$$N_{\text{pages}} = \frac{2 \times 1024\text{ GB}}{2\text{ MB}} = 1,048,576\text{ pages}$$
$$\text{PTE Memory} = 1,048,576 \times 8\text{ bytes} \approx \mathbf{8.0\text{ Megabytes}}\quad (99.8\%\text{ reduction!})$$

With **1 GB Hugepages**:
$$N_{\text{pages}} = \frac{2 \times 1024\text{ GB}}{1\text{ GB}} = 2,048\text{ pages}$$
$$\text{PTE Memory} = 2,048 \times 8\text{ bytes} \approx \mathbf{16.0\text{ Kilobytes}}$$

The CPU TLB can now hold the entire $2\text{ TB}$ physical address space simultaneously, eliminating TLB misses during multi-hundred GB/s GPUDirect Storage transfers.

#### Sizing Equation for Ansible sysctl:
To allocate $512\text{ GB}$ of $2\text{ MB}$ Hugepages in `/etc/sysctl.d/99-hugepages.conf`:

$$\text{vm.nr\_hugepages} = \frac{512 \times 1024\text{ MB}}{2\text{ MB}} = \mathbf{262,144}$$

---

## 4. Deep Architecture: Out-of-Band Redfish REST State Machine

Ansible communicates with the BMC over an isolated Out-of-Band (OOB) management network.

```
+-----------------------------------------------------------------------------+
|                     REDFISH BIOS AUTOMATION WORKFLOW                        |
+-----------------------------------------------------------------------------+
|  Ansible Control Node                                                       |
|    |                                                                        |
|    | (1) POST /redfish/v1/SessionService/Sessions (Auth Token generated)    |
|    v                                                                        |
|  DGX BMC (Baseboard Management Controller)                                  |
|    |                                                                        |
|    | (2) PATCH /redfish/v1/Systems/Self/Bios/Settings                       |
|    |     Payload: { "Attributes": { "WorkloadProfile": "HPC",               |
|    |                                "SriovEnable": "Enabled",               |
|    |                                "PcieAspm": "Disabled" } }              |
|    |                                                                        |
|    | (3) BMC creates Pending Configuration Object                           |
|    v                                                                        |
|  Ansible Control Node                                                       |
|    |                                                                        |
|    | (4) POST /redfish/v1/Systems/Self/Actions/ComputerSystem.Reset         |
|    |     Payload: { "ResetType": "GracefulRestart" }                        |
|    v                                                                        |
|  DGX Server: Cold boots, flashes UEFI NVRAM, trains PCIe Gen5 buses         |
+-----------------------------------------------------------------------------+
```

---

## 5. Concrete Production Lab: End-to-End Bare-Metal Provisioning Playbook

```yaml
---
# playbook: bare_metal_foundation.yml
# Provisions BMC BIOS parameters via Redfish and hardens Linux kernel for AI
- name: Phase 1 - Out-of-Band BIOS Configuration via Redfish
  hosts: bmc_nodes
  gather_facts: false
  connection: local
  tasks:
    - name: 1. Ensure High-Performance Compute BIOS attributes
      community.general.redfish_config:
        category: Systems
        command: SetBiosAttributes
        baseuri: "{{ bmc_ip }}"
        username: "{{ bmc_user }}"
        password: "{{ bmc_password }}"
        bios_attributes:
          WorkloadProfile: "HPC"
          SriovEnable: "Enabled"
          IntelVirtualizationTechnology: "Enabled"
          PcieAspmSupport: "Disabled"
          EnergyPerfBias: "MaxPerformance"
      register: bios_result

    - name: 2. Trigger Graceful Restart if BIOS configuration changed
      when: bios_result.changed
      community.general.redfish_command:
        category: Systems
        command: PowerReboot
        baseuri: "{{ bmc_ip }}"
        username: "{{ bmc_user }}"
        password: "{{ bmc_password }}"

- name: Phase 2 - In-Band Linux Kernel Hardening & Memory Tuning
  hosts: gpu_nodes
  become: true
  gather_facts: true
  tasks:
    - name: 1. Configure Linux Kernel Boot Parameters (GRUB)
      ansible.builtin.lineinfile:
        path: /etc/default/grub
        regexp: '^GRUB_CMDLINE_LINUX_DEFAULT='
        line: 'GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on iommu=pt pcie_aspm=off processor.max_cstate=0 intel_idle.max_cstate=0 transparent_hugepage=never default_hugepagesz=1G hugepagesz=1G hugepages=256"'
      notify: Update GRUB

    - name: 2. Allocate 2MB Hugepages via Sysctl
      ansible.posix.sysctl:
        name: vm.nr_hugepages
        value: "262144"  # 512 GB in 2MB pages
        state: present
        sysctl_file: /etc/sysctl.d/99-ai-memory.conf
        reload: true

    - name: 3. Maximize Network & IPC Memory Buffers
      ansible.posix.sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        state: present
        sysctl_file: /etc/sysctl.d/99-ai-sysctl.conf
        reload: true
      loop:
        - { name: "net.core.rmem_max", value: "67108864" }        # 64 MB
        - { name: "net.core.wmem_max", value: "67108864" }        # 64 MB
        - { name: "net.core.optmem_max", value: "67108864" }
        - { name: "net.ipv4.tcp_rmem", value: "4096 87380 67108864" }
        - { name: "net.ipv4.tcp_wmem", value: "4096 65536 67108864" }
        - { name: "vm.max_map_count", value: "2621440" }          # Required for PyTorch/CUDA
        - { name: "fs.file-max", value: "2097152" }

    - name: 4. Lock CPU Scaling Governor to Maximum Performance
      ansible.builtin.apt:
        name: cpufrequtils
        state: present

    - name: Set governor in /etc/default/cpufrequtils
      ansible.builtin.copy:
        dest: /etc/default/cpufrequtils
        content: |
          GOVERNOR="performance"
      notify: Restart CPUFreq

  handlers:
    - name: Update GRUB
      ansible.builtin.command: update-grub

    - name: Restart CPUFreq
      ansible.builtin.systemd:
        name: cpufrequtils
        state: restarted
```

---

## 6. Comparative Bare-Metal Configuration Matrix

| Setting | Default Linux Server | Tuned AI Supercomputer Node | Impact of Misconfiguration |
| :--- | :--- | :--- | :--- |
| **IOMMU Mode** | `intel_iommu=off` | `intel_iommu=on iommu=pt` | GDS and InfiniBand P2P fail completely |
| **PCIe ASPM** | `pcie_aspm=default` (Active)| `pcie_aspm=off` (Disabled) | Multi-millisecond PCIe latency spikes |
| **Hugepages** | Transparent (THP Enabled) | **Explicit 1GB/2MB Locked** | Severe memory fragmentation & CUDA OOM |
| **CPU Governor** | `powersave` / `schedutil` | **`performance`** | CPU clock throttling stalls DataLoader |
| **C-States** | Deep sleep (C6/C8 allowed) | **Locked to C0 (`max_cstate=0`)** | Core wake-up latency ruins barrier sync |

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        BARE-METAL OS SRE DIAGNOSTIC MATRIX                                        |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| `nvidia-fs` driver fails to load:  | IOMMU passthrough        | Verify active kernel boot params: |
| "IOMMU disabled or missing pt".    | disabled in GRUB.        | `cat /proc/cmdline | grep iommu`  |
|                                    |                          | Must contain `iommu=pt`.          |
+------------------------------------+--------------------------+-----------------------------------+
| Hugepages allocation fails:        | Memory fragmented; kernel| Pre-allocate pages at boot via    |
| `vm.nr_hugepages` rejected.        | cannot find contiguous   | GRUB cmdline (`hugepages=...`)    |
|                                    | physical 2MB blocks.     | instead of runtime sysctl.        |
+------------------------------------+--------------------------+-----------------------------------+
| Redfish API returns 401:           | Expired token or default | Test Redfish auth via curl:       |
| `Unauthorized`.                    | OEM password changed.    | `curl -k -u user:pass https://$BMC|
|                                    |                          | /redfish/v1/Systems/Self`         |
+------------------------------------+--------------------------+-----------------------------------+
| Random GPU all-reduce stalls on    | CPU core downclocking    | Check current CPU core frequency: |
| specific workers.                  | under powersave governor.| `cat /proc/cpuinfo | grep MHz`    |
|                                    |                          | Force governor to performance.    |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **Redfish BIOS Hardened:** Out-of-band automation validated for PCIe ASPM disablement and SR-IOV.
- [ ] **IOMMU Passthrough Confirmed:** `dmesg | grep -i iommu` confirms `IOMMU enabled, passthrough active`.
- [ ] **Hugepages Locked:** `/proc/meminfo` confirms `HugePages_Total` matches target memory allocation.
- [ ] **CPU Frequency Fixed:** All CPU cores verified running at maximum non-throttled frequency in C0 state.
- [ ] **Network Buffers Sized:** Linux sysctl `rmem_max` and `wmem_max` tuned to $64\text{ MB}$ for high-BDP fabrics.
