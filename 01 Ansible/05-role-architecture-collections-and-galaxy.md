# Volume 05: Enterprise Role Architecture, Collections & Execution Environments

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 05: Modular Roles, Ansible Galaxy Collections, Dependency DAGs & Containerized EEs
====================================================================================================
```

---

## 1. Executive Intuition: The Monolithic Playbook Collapse

When operations teams begin automating GPU clusters, they often write monolithic playbooks: a single 4,000-line `deploy_cluster.yml` file containing bare-metal BIOS settings, kernel sysctl flags, NVIDIA driver installs, InfiniBand network setup, Slurm daemons, and Prometheus exporters.

As the cluster scales, the monolithic playbook collapses:
1. **Zero Reusability:** Code written for a DGX H100 cluster cannot be reused on a Grace Hopper GH200 cluster without copying and pasting hundreds of lines.
2. **Untestable Code Paths:** You cannot test the InfiniBand configuration in isolation without running the entire 4,000-line playbook.
3. **Dependency Version Drift:** A change to an OS kernel parameter silently breaks an assumption in the Slurm GPU configuration 2,000 lines later.

```
+-----------------------------------------------------------------------------------------+
|                  MONOLITHIC PLAYBOOK VS. ENTERPRISE COLLECTIONS                         |
+------------------------------------+----------------------------------------------------+
| Monolithic Playbook (Anti-Pattern) | Enterprise Collections & Roles (Standard)          |
+------------------------------------+----------------------------------------------------+
| 4,000 lines in single site.yml     | Decentralized namespaces: `nvidia.cluster_infra`   |
| No semantic versioning             | Strict SemVer 2.0.0 (e.g. `1.4.2`)                 |
| Global variable namespace collision| Encapsulated role `defaults/` and `vars/` scopes   |
| Unversioned Python packages        | Isolated Execution Environments (EE containers)    |
| Testing requires full cluster      | Isolated unit & integration testing via Molecule   |
+------------------------------------+----------------------------------------------------+
```

The enterprise standard decomposes infrastructure into **Modular Roles** packaged into **Ansible Collections** and executed within containerized **Execution Environments (EE)**.

---

## 2. Lineage & Evolution of Ansible Packaging

```
   [2012: Flat Playbook Files]
                 |
           (Single large YAML files with sequential task lists)
                 |
   [2013: Classic Role Directory Standard]
                 |
           (roles/<role_name> with standardized tasks, handlers, vars, defaults, meta)
                 |
   [2016: Ansible Galaxy Community Hub]
                 |
           (Public repository for downloading community roles via requirements.yml)
                 |
   [2019: Ansible Collections (Ansible 2.9+)]
                 |
           (Unified packaging of roles, modules, action plugins, and lookup plugins)
                 |
   [2021: Execution Environments & ansible-builder]
                 |
           (OCI container images packaging Python, OS dependencies, and Collections)
```

---

## 3. First-Principles Mathematics: Role Dependency Resolution DAG

When roles declare dependencies in `meta/main.yml`, Ansible constructs a **Directed Acyclic Graph (DAG)** of execution.

Let $G = (V, E)$ be the dependency graph where:
- $V = \{R_1, R_2, \dots, R_k\}$ is the set of roles.
- $E = \{(R_i, R_j)\}$ denotes that role $R_i$ depends on role $R_j$ (i.e. $R_j$ must execute before $R_i$).

```
                +----------------------------+
                | R_base: Base OS & Hugepages|
                +--------------+-------------+
                               |
                +--------------v-------------+
                | R_mellanox: OFED & RoCEv2  |
                +--------------+-------------+
                               |
                +--------------v-------------+
                | R_nvidia: Drivers & Fabric |
                +--------------+-------------+
                               |
                +--------------v-------------+
                | R_gds: GPUDirect Storage   |
                +----------------------------+
```

### 3.1 Topological Sort & Cycle Detection Math
Ansible evaluates role dependencies using **Kahn's Algorithm** (topological sort):
1. Compute the in-degree for all vertices: $\text{in-degree}(R_i) = \text{number of incoming edges}$.
2. Enqueue all roles with $\text{in-degree} = 0$.
3. While queue is non-empty:
   - Dequeue $R_u$, append to execution order.
   - For each neighbor $R_v$ of $R_u$: decrement $\text{in-degree}(R_v)$. If 0, enqueue.
4. If total executed roles $< |V|$, a **Circular Dependency Cycle** exists (e.g. $A \to B \to A$), and Ansible immediately halts execution.

$$\text{Time Complexity} = O(|V| + |E|)$$

---

## 4. Deep Architecture: Standardized Role Layout

Every role in the AI automation fabric adheres to the standardized directory structure:

```
roles/nvidia_driver_provision/
├── defaults/
│   └── main.yml        # Lowest precedence default variables (overridable by user)
├── vars/
│   └── main.yml        # High precedence internal variables (OS package URLs, etc.)
├── tasks/
│   ├── main.yml        # Master task entrypoint
│   ├── install.yml     # Package repository and driver compilation tasks
│   └── verify.yml      # Post-install verification and NVML health asserts
├── handlers/
│   └── main.yml        # Service restart triggers (e.g., restart nvidia-fabricmanager)
├── templates/
│   ├── fabricmanager.conf.j2  # Jinja2 template for Fabric Manager
│   └── nvidia.rules.j2        # Udev rules template for device nodes
├── files/
│   └── 99-nvidia.conf  # Static configuration files
├── meta/
│   └── main.yml        # Role dependencies and metadata
└── molecule/
    └── default/        # Molecule automated testing scenario
        ├── molecule.yml
        └── converge.yml
```

---

## 5. Concrete Production Lab: The `nvidia_driver_provision` Role

### 5.1 Role Defaults (`defaults/main.yml`)
```yaml
---
# Default configurations for NVIDIA Open Kernel Driver installation
nvidia_driver_branch: "550"
nvidia_driver_version: "550.54.15"
nvidia_open_kernel_modules: true
nvidia_install_fabric_manager: true
nvidia_enable_persistence_mode: true
```

### 5.2 Role Tasks (`tasks/main.yml`)
```yaml
---
- name: 1. Ensure kernel headers match running kernel
  ansible.builtin.apt:
    name: "linux-headers-{{ ansible_kernel }}"
    state: present
    update_cache: true

- name: 2. Install NVIDIA Driver packages
  ansible.builtin.apt:
    name:
      - "cuda-drivers-{{ nvidia_driver_branch }}"
      - "nvidia-driver-{{ nvidia_driver_branch }}{{ '-open' if nvidia_open_kernel_modules else '' }}"
    state: present
  notify: Reload NVIDIA Kernel Modules

- name: 3. Configure NVIDIA Persistence Daemon
  ansible.builtin.systemd:
    name: nvidia-persistenced
    state: started
    enabled: true

- name: 4. Install and configure NVIDIA Fabric Manager (HGX/NVSwitch systems)
  when: nvidia_install_fabric_manager | bool
  block:
    - name: Install fabric-manager package
      ansible.builtin.apt:
        name: "nvidia-fabricmanager-{{ nvidia_driver_branch }}"
        state: present

    - name: Enable and start fabric-manager service
      ansible.builtin.systemd:
        name: nvidia-fabricmanager
        state: started
        enabled: true
      notify: Restart Fabric Manager

- name: 5. Verify Driver and GPU Communication
  ansible.builtin.command: nvidia-smi --query-gpu=driver_version,count --format=csv,noheader
  register: nvidia_smi_out
  changed_when: false
  failed_when: nvidia_driver_version not in nvidia_smi_out.stdout
```

### 5.3 Role Handlers (`handlers/main.yml`)
```yaml
---
- name: Restart Fabric Manager
  ansible.builtin.systemd:
    name: nvidia-fabricmanager
    state: restarted

- name: Reload NVIDIA Kernel Modules
  ansible.builtin.debug:
    msg: "NVIDIA driver updated; node scheduled for graceful reboot."
```

---

## 6. Execution Environments (EE) with `ansible-builder`

To guarantee that playbooks run identically across different engineer laptops, CI/CD pipelines, and AWX clusters without Python dependency hell, execution is packaged into an OCI container image.

### 6.1 Execution Environment Definition (`execution-environment.yml`)

```yaml
version: 3
images:
  base_image:
    name: registry.access.redhat.com/ubi9/ubi-minimal:latest

dependencies:
  ansible_core:
    package_pip: ansible-core==2.16.4
  ansible_runner:
    package_pip: ansible-runner==2.3.4
  galaxy:
    collections:
      - name: community.general
        version: "8.4.0"
      - name: ansible.posix
        version: "1.5.4"
      - name: netbox.netbox
        version: "3.17.1"
      - name: community.hashi_vault
        version: "6.1.0"
  python:
    - pynvml>=11.5.0
    - jmespath>=1.0.1
    - hvac>=2.1.0
    - netaddr>=1.0.0
  system:
    - openssh-clients
    - iproute
    - git
```

### 6.2 Building the Execution Environment
```bash
# Build the production container image
ansible-builder build --tag ai-infra-execution-env:1.0.0 --container-runtime docker
```

---

## 7. Comparative Packaging Matrix

| Packaging Paradigm | Scope | Versioning | Dependency Management | Isolation |
| :--- | :--- | :--- | :--- | :--- |
| **Flat Playbooks** | Single File | Git commit hash | None (Ad-hoc pip/apt) | Zero (Host Python) |
| **Classic Roles** | Modular task set | Git tag | `meta/main.yml` dependencies| Host filesystem |
| **Ansible Collections**| Multi-role + Plugins| SemVer (`galaxy.yml`) | `requirements.yml` resolution| Namespaced |
| **Execution Environment**| Complete Runtime | OCI Container Tag | Hermetic container manifest | **100% Hermetic Isolation** |

---

## 8. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        ROLES & COLLECTIONS SRE DIAGNOSTIC MATRIX                                  |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| `ERROR! the role 'x' was not       | Role not in search path; | Check roles_path in config:       |
| found in ...`.                     | directory name mismatch. | `ansible-config dump | grep ROLES`|
|                                    |                          | Verify folder spelling.           |
+------------------------------------+--------------------------+-----------------------------------+
| Circular dependency error:         | Role A requires B, and B | Inspect `meta/main.yml` across    |
| `Dependency cycle detected`.       | requires A.              | both roles; refactor common tasks |
|                                    |                          | into a third base role.           |
+------------------------------------+--------------------------+-----------------------------------+
| Collection module missing in       | Python dependencies      | Rebuild EE container with missing |
| Execution Environment (EE).        | missing in container.    | pip package declared in manifest. |
+------------------------------------+--------------------------+-----------------------------------+
| Variable collision: Role A         | Both roles use identical | Prefix all role variables with    |
| overwrites variable in Role B.     | variable name without    | role name: `nvidia_driver_version`|
|                                    | role prefix.             | instead of `version`.             |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 9. Verification & Architectural Synthesis Checklist

- [ ] **Modular Role Architecture:** Infrastructure decomposed into independent roles (`nvidia_driver`, `mellanox_ofed`, etc.).
- [ ] **SemVer Dependency Constraints:** All external collection requirements pinned to explicit minor versions in `requirements.yml`.
- [ ] **Variable Scoping Enforced:** Role defaults placed in `defaults/main.yml` with strict role-name prefixes.
- [ ] **Topological DAG Validated:** Role dependencies in `meta/main.yml` free of circular cycles.
- [ ] **Hermetic Container EE:** Execution Environment container image built and validated with `ansible-builder`.
