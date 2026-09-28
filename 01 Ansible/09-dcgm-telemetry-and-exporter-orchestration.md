# Volume 09: Data Center GPU Manager (DCGM) & Telemetry Orchestration

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 09: DCGM nv-hostengine, Field Groups, Prometheus dcgm-exporter & NVVS Diagnostics
====================================================================================================
```

---

## 1. Executive Intuition: The Polling Loop Trap

In naive GPU cluster monitoring, administrators often set up cron jobs or shell scripts that execute `nvidia-smi` every few seconds. In an exascale cluster (16,384 GPUs), this is a fatal design:
1. **Process Fork Overhead:** Every `nvidia-smi` invocation spawns a new binary process, establishes an NVML context, polls the hardware via IOCTLs, and tears down the context ($\sim 15\text{ ms} - 50\text{ ms}$ CPU penalty).
2. **Missing Transient Anomalies:** High-frequency electrical spikes, thermal throttling micro-events, and NVLink single-lane retraining happen in microseconds, completely invisible to 15-second `nvidia-smi` polling.
3. **Double-Bit ECC Blindness:** A memory cell uncorrectable error (DBE) will corrupt model weights silently unless an active hardware register monitor immediately triggers an evacuation event.

```
+-----------------------------------------------------------------------------------------+
|                  NVIDIA-SMI POLLING VS. DCGM CONTINUOUS TELEMETRY                       |
+------------------------------------+----------------------------------------------------+
| Periodic nvidia-smi (Anti-Pattern) | NVIDIA DCGM Architecture (Standard)                |
+------------------------------------+----------------------------------------------------+
| 15ms fork overhead per poll        | Sub-microsecond MMIO hardware sampling             |
| CPU-intensive process boots        | Persistent background daemon (nv-hostengine)       |
| Coarse 10-60 second resolution     | Configurable 100ms - 1000ms high-frequency buffers |
| No historical trending in RAM      | In-memory ring buffer (up to 24 hours of data)     |
| No automated diagnostic validation | Integrated NVIDIA System Validation Suite (NVVS)   |
+------------------------------------+----------------------------------------------------+
```

The enterprise standard deploys **NVIDIA Data Center GPU Manager (DCGM)**: a persistent background daemon (`nv-hostengine`) coupled with **`dcgm-exporter`** to stream high-cardinality Prometheus telemetry into cluster-wide monitoring platforms.

---

## 2. Lineage & Evolution of GPU Monitoring

```
   [2008: nvclock & nvml.dll]
                 |
           (Ad-hoc overclocking tools and early NVML C APIs)
                 |
   [2012: nvidia-smi dmon]
                 |
           (First native CLI daemon monitoring tool built into driver packages)
                 |
   [2018: NVIDIA DCGM v1.x]
                 |
           (Introduction of nv-hostengine, client/server telemetry architecture)
                 |
   [2021: dcgm-exporter for Kubernetes]
                 |
           (Prometheus-native exporter transforming DCGM field IDs into OpenMetrics)
                 |
   [2024: Unified Telemetry Architecture (DCGM + OpenTelemetry)]
                 |
           (Microsecond GPU trace emission integrated into distributed tracing frameworks)
```

---

## 3. First-Principles Mathematics: Polling Overhead vs. MMIO Sampling

Let $N$ be the number of servers, with 8 GPUs per server:
- If an agent runs `nvidia-smi` every $1\text{ second}$:
  $$\text{Syscall & Context Creation Time} \approx 25\text{ ms per execution}$$
  $$\text{CPU Time Burned} = 2.5\%\text{ of a dedicated host CPU core 24/7 per node}$$
  Across 1,000 nodes, the cluster burns **25 full CPU cores continuously** purely executing polling forks!

- In contrast, DCGM's `nv-hostengine` holds an open device file descriptor (`/dev/nvidiactl`) and reads hardware counter registers directly via PCIe Memory Mapped I/O (MMIO):
  $$\text{Sampling Latency} \le 40\ \mu\text{s}$$
  $$\text{CPU Footprint} < 0.1\%\text{ of a single core}$$

---

## 4. Deep Architecture: DCGM Field IDs & Exporter Pipeline

```
+-----------------------------------------------------------------------------+
|                        DCGM OBSERVABILITY TOPOLOGY                          |
+-----------------------------------------------------------------------------+
|  NVIDIA Physical Hardware (H100 / B200 / NVLink)                            |
|    |                                                                        |
|    v (Continuous Register Sampling: Clocks, Power, Thermals, NVLink)        |
|  nv-hostengine Daemon (Systemd Unit, Local TCP Port 5555)                   |
|    |-- Internal In-Memory Time-Series Ring Buffer                           |
|    |-- Health Watchdog Engine: Evaluates XID, ECC, and Thermal thresholds   |
|    v                                                                        |
|  dcgm-exporter (Prometheus Exporter Daemon / Port 9400)                     |
|    |-- Reads Field Group Configuration: /etc/dcgm-exporter/dcgm-fields.csv  |
|    |-- Maps Internal Field IDs -> Prometheus Metric Strings                 |
|    v                                                                        |
|  Prometheus Scraper: Scrapes http://<node-ip>:9400/metrics every 5 seconds  |
+-----------------------------------------------------------------------------+
```

### 4.1 Essential DCGM Field IDs for Foundation Model Training
```
Field ID | Metric Name                     | Description
---------+---------------------------------+----------------------------------------
1001     | DCGM_FI_DEV_SM_CLOCK            | Current Streaming Multiprocessor Clock
1006     | DCGM_FI_DEV_GPU_TEMP            | Core Silicon Temperature (deg C)
1007     | DCGM_FI_DEV_POWER_USAGE         | Real-time Power Draw in Watts
1008     | DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION | Cumulative Joules Consumed
1011     | DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL  | Aggregate NVLink Throughput (Bytes/sec)
203      | DCGM_FI_DEV_ECC_DBE_VOL_DEV     | Volatile Double-Bit ECC Errors (Uncorrectable)
204      | DCGM_FI_DEV_ECC_SBE_VOL_DEV     | Volatile Single-Bit ECC Errors (Correctable)
```

---

## 5. Concrete Production Lab: Automated DCGM & Exporter Deployment Role

```yaml
---
# playbook: deploy_dcgm_monitoring.yml
# Provisions NVIDIA DCGM, starts nv-hostengine, and deploys dcgm-exporter
- name: Orchestrate Enterprise DCGM & Prometheus Telemetry
  hosts: gpu_nodes
  become: true
  gather_facts: true
  tasks:
    - name: 1. Install Datacenter GPU Manager (DCGM) Package
      ansible.builtin.apt:
        name: datacenter-gpu-manager
        state: present
        update_cache: true

    - name: 2. Enable and Start nv-hostengine Daemon
      ansible.builtin.systemd:
        name: nvidia-dcgm
        state: started
        enabled: true

    - name: 3. Verify DCGM Daemon Connectivity
      ansible.builtin.command: dcgmi discovery -l
      register: dcgmi_disc
      changed_when: false
      retries: 3
      delay: 2
      until: dcgmi_disc.rc == 0

    - name: 4. Deploy Custom DCGM Field Configuration
      ansible.builtin.copy:
        dest: /etc/dcgm-exporter-fields.csv
        content: |
          # DCGM Custom Field Map for AI Supercomputers
          DCGM_FI_DEV_SM_CLOCK, gauge, gpu_sm_clock_mhz
          DCGM_FI_DEV_MEM_CLOCK, gauge, gpu_memory_clock_mhz
          DCGM_FI_DEV_MEMORY_TEMP, gauge, gpu_memory_temp_c
          DCGM_FI_DEV_GPU_TEMP, gauge, gpu_temp_c
          DCGM_FI_DEV_POWER_USAGE, gauge, gpu_power_watts
          DCGM_FI_DEV_GPU_UTIL, gauge, gpu_utilization_ratio
          DCGM_FI_DEV_MEM_COPY_UTIL, gauge, gpu_memory_copy_utilization_ratio
          DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL, counter, gpu_nvlink_throughput_bytes_total
          DCGM_FI_DEV_ECC_DBE_VOL_DEV, counter, gpu_ecc_double_bit_errors_total
          DCGM_FI_DEV_XID_ERRORS, gauge, gpu_active_xid_code

    - name: 5. Download and Install dcgm-exporter Binary
      ansible.builtin.get_url:
        url: "https://github.com/NVIDIA/dcgm-exporter/releases/download/v3.3.5-3.4.0/dcgm-exporter_3.3.5-3.4.0_linux_amd64.tar.gz"
        dest: /tmp/dcgm-exporter.tar.gz
        mode: '0644'

    - name: Unpack dcgm-exporter
      ansible.builtin.unarchive:
        src: /tmp/dcgm-exporter.tar.gz
        dest: /usr/local/bin/
        remote_src: true
        extra_opts: ['--strip-components=1']

    - name: 6. Create dcgm-exporter Systemd Service Unit
      ansible.builtin.copy:
        dest: /etc/systemd/system/dcgm-exporter.service
        content: |
          [Unit]
          Description=NVIDIA DCGM Exporter for Prometheus
          After=nvidia-dcgm.service
          Wants=nvidia-dcgm.service

          [Service]
          Type=simple
          ExecStart=/usr/local/bin/dcgm-exporter -f /etc/dcgm-exporter-fields.csv -a 0.0.0.0:9400
          Restart=always
          RestartSec=5s

          [Install]
          WantedBy=multi-user.target
      notify: Restart dcgm-exporter

    - name: 7. Enable and Start dcgm-exporter Service
      ansible.builtin.systemd:
        name: dcgm-exporter
        state: started
        enabled: true
        daemon_reload: true

    - name: 8. Verify Prometheus Metrics Scrape Endpoint
      ansible.builtin.uri:
        url: http://localhost:9400/metrics
        status_code: 200
        timeout: 5
      register: metrics_probe
      retries: 5
      delay: 2
      until: metrics_probe.status == 200

  handlers:
    - name: Restart dcgm-exporter
      ansible.builtin.systemd:
        name: dcgm-exporter
        state: restarted
```

---

## 6. Automated NVIDIA System Validation (NVVS) Health Diagnostic

Ansible can invoke DCGM's integrated validation suite to execute physical hardware tests (Level 1 Quick, Level 2 Medium, Level 3 Comprehensive):

```yaml
# Execute NVVS Level 3 Diagnostic before scheduling production training
- name: Execute Full Hardware Validation (NVVS Level 3)
  ansible.builtin.command: dcgmi diag -r 3
  register: nvvs_results
  changed_when: false
  failed_when: "'FAIL' in nvvs_results.stdout"
```

*NVVS Diagnostic Coverage:*
- PCIe Bus Bandwidth & Replay Test
- Memory Stress (Bandwidth & Page Retirement test)
- High-Stress GEMM Test (Max thermal and power dissipation)
- NVLink Loopback & Ping-Pong test across all NVSwitches

---

## 7. Comparative Observability Architecture Matrix

| Telemetry Platform | Mechanism | Overhead | Sampling Granularity | Prometheus Native | Hardware Scope |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`nvidia-smi` Poller** | Subprocess fork | High (~25ms) | 10–60 Seconds | No (Custom script) | GPU Core only |
| **Direct PyNVML** | Python CTypes | Moderate (~2ms) | 1–5 Seconds | Manual bridge | GPU Core only |
| **NVIDIA DCGM Exporter**| **In-kernel MMIO** | **Negligible (<40us)**| **100ms - 1000ms** | **Native OpenMetrics**| **GPU, NVLink, NVSwitch**|
| **OS Node Exporter** | procfs parsing | Low | 15 Seconds | Native | CPU / Host RAM only |

---

## 8. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        DCGM TELEMETRY SRE DIAGNOSTIC MATRIX                                       |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| `dcgm-exporter` fails with:        | `nv-hostengine` daemon   | Verify service state:             |
| "could not connect to DCGM host".  | crashed or stopped.      | `systemctl status nvidia-dcgm`    |
|                                    |                          | Restart `nv-hostengine`.          |
+------------------------------------+--------------------------+-----------------------------------+
| Port 9400 unreachable from         | Host firewall blocking   | Open port in ufw / iptables:      |
| Prometheus scraper server.         | TCP 9400.                | `ufw allow from <prom_ip> to any  |
|                                    |                          |  port 9400 proto tcp`             |
+------------------------------------+--------------------------+-----------------------------------+
| Metric `gpu_ecc_double_bit_errors` | Physical HBM3e hardware  | Immediately cordon node and run:  |
| increments above zero.             | memory cell degradation. | `dcgmi diag -r 3`                 |
|                                    |                          | Schedule GPU replacement (RMA).   |
+------------------------------------+--------------------------+-----------------------------------+
| `dcgmi` commands hang indefinitely| Driver hung on deadlocked| Check kernel ring buffer:         |
| in uninterruptible sleep.          | PCIe transaction.        | `dmesg -T | grep -i xid`          |
|                                    |                          | Hard reset host via BMC.          |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 9. Verification & Architectural Synthesis Checklist

- [ ] **DCGM Daemon Active:** `systemctl is-active nvidia-dcgm` returns `active`.
- [ ] **Exporter Unit Running:** `dcgm-exporter.service` running and serving metrics on port 9400.
- [ ] **Prometheus Ingestion Confirmed:** Scrape response verified to contain `gpu_power_watts` and `gpu_nvlink_throughput_bytes_total`.
- [ ] **Hardware Diagnostic Qualified:** `dcgmi diag -r 1` executes cleanly with zero failures.
- [ ] **ECC Error Alerting Wired:** Alerting rules configured to notify SREs immediately on any Double-Bit Error (DBE).
