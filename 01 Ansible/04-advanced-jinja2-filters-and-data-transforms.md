# Volume 04: Advanced Jinja2 Filters, JMESPath Queries & Data Transformation

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 04: JMESPath Projections, Complex Telemetry Parsing, Netplan Templating & Custom Python Filters
====================================================================================================
```

---

## 1. Executive Intuition: The Telemetry Parsing Crisis

In modern AI supercomputing, automation playbooks constantly ingest complex, nested, non-uniform telemetry streams:
- `nvidia-smi -q -x`: A 2-megabyte deeply nested XML document describing clocks, thermals, power, and PCIe link counters for 8 GPUs.
- `ibv_devinfo -v` / `ibstat`: Multi-port InfiniBand HCA link states, GID tables, and active rates.
- `ip -j link` / `ip -j route`: Modern Linux kernel JSON interface and routing tables.
- `lshw -json` / `lspci -Dvmmn`: Hierarchical PCIe root complex bus and NUMA socket layouts.

Attempting to parse these nested structures with fragile shell pipelines (`awk`, `sed`, `grep`, `cut`) leads to silent production catastrophes: a trailing whitespace or unexpected XML tag silently evaluates to empty string, causing Ansible to write broken configuration files across hundreds of servers.

```
+-----------------------------------------------------------------------------------------+
|                        DECLARATIVE TELEMETRY TRANSFORMATION                             |
+-----------------------------------------------------------------------------------------+
| Raw Nested JSON / XML Telemetry (nvidia-smi, ip -j, netbox)                             |
|   |                                                                                     |
|   v                                                                                     |
| In-Memory Data Pipeline (Ansible Engine)                                                |
|   |-- JMESPath Projections: Extracting exactly the PCIe BDF and Power Cap               |
|   |-- IP Address Filters: Computing RoCEv2 secondary subnet offsets                     |
|   +-- Custom Python Filter Plugins: Mapping GPU index to NUMA CPU cores                 |
|   v                                                                                     |
| Deterministic Configuration Generation (/etc/netplan/01-roce.yaml, /etc/cufile.json)    |
+-----------------------------------------------------------------------------------------+
```

High-reliability AI automation demands declarative transformation engines: **JMESPath Projections (`json_query`)**, **Network CIDR Math (`ansible.utils.ipaddr`)**, and **Custom Python Filter Plugins**.

---

## 2. Lineage & Evolution of Ansible Data Transformation

```
   [2012: Basic String Interpolation]
                 |
           (Basic {{ var }} substitution; primitive filters like | lower, | default)
                 |
   [2015: Jinja2 Loops & Conditionals]
                 |
           (Embedded {% for host in groups['gpu'] %} templates; string munging)
                 |
   [2017: JMESPath Query Engine (json_query)]
                 |
           (Declarative JSON transformation based on RFC specification for query paths)
                 |
   [2020: Ansible Utils Collection (ipaddr)]
                 |
           (Dedicated collection for RFC 1918 / CIDR subnet calculation)
                 |
   [2024: Native Python Typed Filter Plugins]
                 |
           (Type-safe, compiled Python filter plugins directly executed in control node memory)
```

---

## 3. First-Principles Mathematics: Network CIDR Math & Subnet Offsets

When configuring secondary high-speed networks (such as 8 dedicated RoCEv2 or InfiniBand storage interfaces per DGX node), IP addressing follows strict mathematical offsets.

### 3.1 Subnet Derivation Formula
Given a base supernet $S = \text{10.200.0.0/16}$, each of the 8 storage rails requires an isolated Layer-3 subnet:

$$\text{Rail Prefix Length} = 16 + \lceil \log_2(8) \rceil = 16 + 3 = 19$$

Rail $k$ ($0 \le k \le 7$) has subnet prefix:
$$\text{Base IP}_k = \text{Base Supernet} + (k \times 2^{32 - 19}) = \text{Base Supernet} + (k \times 8192)$$

```
+-----------------------------------------------------------------------------------------+
|                         DGX RAIL-ALIGNED STORAGE SUBNET MAP                             |
+--------+------------------+---------------------+---------------------+-----------------+
| Rail # | Interface Name   | Subnet CIDR (/19)   | Subnet Range        | Broadcast IP    |
+--------+------------------+---------------------+---------------------+-----------------+
| Rail 0 | roce0 (NIC 0)    | 10.200.0.0/19       | 10.200.0.1 - 31.254 | 10.200.31.255   |
| Rail 1 | roce1 (NIC 1)    | 10.200.32.0/19      | 10.200.32.1 - 63.254| 10.200.63.255   |
| Rail 2 | roce2 (NIC 2)    | 10.200.64.0/19      | 10.200.64.1 - 95.254| 10.200.95.255   |
| Rail 3 | roce3 (NIC 3)    | 10.200.96.0/19      | 10.200.96.1 - 127.254| 10.200.127.255  |
+--------+------------------+---------------------+---------------------+-----------------+
```

Instead of hardcoding hundreds of IP addresses in inventory files, Jinja2 network filters compute these addresses dynamically from the node index ($N_{\text{id}}$):

$$\text{IP}(N_{\text{id}}, k) = \text{Subnet}_k + N_{\text{id}}$$

---

## 4. Deep Architecture: JMESPath Query Patterns for AI Telemetry

JMESPath is a query language for JSON. It allows projection, filtering, and slicing of deep structures:

### 4.1 Extracting GPU Bus IDs and Memory Utilization
Given JSON output from `nvidia-smi --query-gpu=pci.bus_id,memory.total,memory.free --format=json`:

```yaml
# Extract all GPU Bus IDs where free memory is less than 10,000 MiB
- name: Query memory-constrained GPUs
  ansible.builtin.set_fact:
    constrained_gpus: "{{ gpu_telemetry | community.general.json_query(query) }}"
  vars:
    query: "gpus[?memory_free < `10000`].pci_bus_id"
```

### 4.2 Flattening InfiniBand Port States Across Multi-HCA Servers
```yaml
# Extract all active 400 Gbps InfiniBand interfaces
- name: Extract healthy InfiniBand ports
  ansible.builtin.set_fact:
    healthy_ib_ports: "{{ ib_status | community.general.json_query('adapters[*].ports[?state==`ACTIVE` && rate==`400 Gb/s`].interface') | flatten }}"
```

---

## 5. Concrete Production Lab: Custom Python Filter Plugin (`filter_plugins/gpu_filters.py`)

Below is a complete, production-grade Ansible filter plugin that computes NUMA node affinity bindings, parses PCIe BDF addresses, and calculates rail-aligned IP addresses.

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-
"""
Production Lab: Custom Ansible Filter Plugin for GPU & Fabric Systems.
Place in `filter_plugins/gpu_filters.py` in your playbook or role directory.
"""

import ipaddress
import re

class FilterModule(object):
    def filters(self):
        return {
            'gpu_numa_affinity': self.gpu_numa_affinity,
            'bdf_to_sysfs_path': self.bdf_to_sysfs_path,
            'rail_storage_ip': self.rail_storage_ip,
        }

    def gpu_numa_affinity(self, gpu_index: int, gpus_per_socket: int = 4) -> dict:
        """
        Calculates optimal CPU NUMA node and CPU core range for a given GPU index.
        HGX 8-GPU nodes pair GPUs 0-3 with NUMA 0, and GPUs 4-7 with NUMA 1.
        """
        socket_id = 0 if gpu_index < gpus_per_socket else 1
        core_start = socket_id * 64
        core_end = core_start + 63
        
        return {
            "gpu_index": gpu_index,
            "numa_socket": socket_id,
            "core_mask": f"{core_start}-{core_end}",
            "optimal_nic": f"mlx5_{gpu_index}"
        }

    def bdf_to_sysfs_path(self, bdf_string: str) -> str:
        """
        Normalizes short BDF (e.g. '41:00.0') to full Linux sysfs path
        (e.g. '/sys/bus/pci/devices/0000:41:00.0').
        """
        clean_bdf = bdf_string.strip()
        if not clean_bdf.startswith("0000:"):
            clean_bdf = f"0000:{clean_bdf}"
        return f"/sys/bus/pci/devices/{clean_bdf}"

    def rail_storage_ip(self, base_supernet: str, rail_index: int, node_offset: int) -> str:
        """
        Dynamically computes rail-aligned IP address for multi-rail storage networks.
        """
        supernet = ipaddress.ip_network(base_supernet)
        # Subnet into 8 isolated /19s
        subnets = list(supernet.subnets(new_prefix=19))
        if rail_index >= len(subnets):
            raise ValueError(f"Rail index {rail_index} exceeds available subnets")
        
        target_subnet = subnets[rail_index]
        # Target host IP = network address + node_offset
        host_ip = target_subnet.network_address + node_offset
        return f"{host_ip}/{target_subnet.prefixlen}"

if __name__ == '__main__':
    # Local verification
    f = FilterModule()
    filters = f.filters()
    
    # Test NUMA affinity
    affinity = filters['gpu_numa_affinity'](5)
    print(f"GPU 5 Affinity: {affinity}")
    assert affinity['numa_socket'] == 1
    
    # Test Rail IP calculation
    rail_ip = filters['rail_storage_ip']("10.200.0.0/16", rail_index=2, node_offset=14)
    print(f"Rail 2 Node 14 IP: {rail_ip}")
    assert rail_ip == "10.200.64.14/19"
```

---

## 6. Concrete Production Lab: Dynamic Netplan Storage Template (`netplan_roce.j2`)

```jinja2
# {{ ansible_managed }}
# Auto-generated high-performance RoCEv2 multi-rail network configuration
network:
  version: 2
  renderer: networkd
  ethernets:
{% for i in range(8) %}
    roce{{ i }}:
      match:
        name: roce{{ i }}
      addresses:
        - {{ '10.200.0.0/16' | rail_storage_ip(i, host_node_id | int) }}
      mtu: 9000
      routes:
        - to: {{ '10.200.0.0/16' | rail_storage_ip(i, 0) | ansible.utils.ipaddr('network') }}/19
          via: {{ '10.200.0.0/16' | rail_storage_ip(i, 1) | ansible.utils.ipaddr('address') }}
          metric: 10{{ i }}
      dhcp4: false
      dhcp6: false
{% endfor %}
```

---

## 7. Comparative Filter Processing Matrix

| Processing Technique | Implementation | Performance | Maintainability | Error Safety |
| :--- | :--- | :--- | :--- | :--- |
| **Shell Command Pipelines** | `shell: "grep ... \| awk ..."` | Very Slow (Subprocesses) | Very Low (Fragile) | Poor (Silent empty string) |
| **Basic Jinja2 String Splitting** | `{{ var.split(',')[0] }}` | Fast | Low (Unreadable) | Poor (IndexError) |
| **JMESPath (`json_query`)** | Declarative query string | Fast (Optimized C/Python) | High | High (Returns None on miss)|
| **Custom Python Filter Plugin** | Pure Python Class | **Fastest (Native memory)** | **Highest (Unit-testable)** | **Type-safe with exceptions** |

---

## 8. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        JINJA2 & DATA TRANSFORM SRE DIAGNOSTIC MATRIX                              |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| `json_query: You need to install   | `jmespath` Python package| Install requirement on control:   |
| "jmespath" on control machine`.    | missing in virtualenv.   | `pip install jmespath`            |
+------------------------------------+--------------------------+-----------------------------------+
| Template error: `UndefinedError:   | Variable missing or mistyped| Run task in debug mode:        |
| 'dict object' has no attribute 'x'`. in dynamic hostvars.     | `ansible -m debug -a "var=dict"`  |
|                                    |                          | Use `| default('fallback')`.      |
+------------------------------------+--------------------------+-----------------------------------+
| `FilterModule` not loaded:         | Plugin directory name    | Ensure folder is named:           |
| `TemplateFilterError: no filter`.  | misspelled or misplaced. | `filter_plugins/` adjacent to     |
|                                    |                          | playbook or inside role root.     |
+------------------------------------+--------------------------+-----------------------------------+
| Netplan configuration fails with   | Indentation error in     | Validate generated YAML syntax:   |
| YAML parse error on target host.   | Jinja2 template loop.    | `yamllint /etc/netplan/*.yaml`    |
|                                    |                          | Test with `netplan try`.          |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 9. Verification & Architectural Synthesis Checklist

- [ ] **JMESPath Standardized:** Nested JSON/XML telemetry parsed using `json_query` rather than shell pipelines.
- [ ] **Custom Filters Tested:** Python filter plugins verified in unit test suite (`gpu_numa_affinity`, `rail_storage_ip`).
- [ ] **Network CIDR Math Automated:** Multi-rail storage IP assignments calculated dynamically from node offsets.
- [ ] **Type-Safe Templates:** Jinja2 templates include `default()` guards for all optional variables.
- [ ] **Zero-Shell Parsing:** All raw command outputs registered as structured facts and processed in control node memory.
