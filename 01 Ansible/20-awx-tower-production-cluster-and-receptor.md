# Volume 20 — AWX / Ansible Automation Platform HA & Receptor Mesh

> **AI Supercomputing Ansible Masterclass · 01 Ansible · Volume 20 of 25**

---

## 1. Executive Intuition

A single Ansible control node is a time-bomb in production AI infrastructure.
When your cluster has 5 000 bare-metal GPU nodes and a firmware rollout must
complete in a maintenance window, **a crashed AWX pod stalls the entire
campaign**. The Ansible Automation Platform (AAP) — and its upstream
open-source twin AWX — solves this with a horizontally-scaled, Kubernetes-
native control plane backed by PostgreSQL HA and a **Receptor** peer-to-peer
mesh that routes Ansible work-units across network segments without a VPN or
SSH gateway.

This volume covers every bolt: the AWX Kubernetes Operator, PostgreSQL
Patroni clusters, Receptor node topology, job isolation via bubblewrap
containers, capacity planner math, and a complete production Python lab that
provisions a fully-HA AWX cluster from spec.

---

## 2. Lineage & Evolution

```
2013 ──► Ansible Tower 1.0 (Red Hat acquisition of AnsibleWorks)
           │ Monolithic Django + Celery + RabbitMQ
2015 ──► Tower 2.x: workflow job templates, RBAC roles
2019 ──► Tower 3.5: Receptor prototype (replaces AMQP for remote nodes)
2020 ──► AWX goes fully open-source; AAP 1.x (subscription)
2021 ──► AAP 2.0: containerized execution nodes, EE (Execution Environments)
           │ Receptor 1.x GA — mesh replaces SSH-hop fan-out
2022 ──► AWX Operator v0.25 (K8s-native deployment)
2023 ──► AAP 2.4: Receptor 1.4, hop nodes, receptor-ctl diagnostics
2024 ──► AWX 23.x: native OpenTelemetry, pg_bouncer connection pooling
2025 ──► AAP 2.5: Receptor 2.x, FIPS-140 mesh encryption, eBPF metrics
```

---

## 3. First-Principles Mathematics

### 3.1 Capacity: Maximum Concurrent Jobs

AWX dispatches jobs to **execution nodes**. Each node has a *capacity* derived
from CPU forks and a memory fork threshold:

$$
C_{\text{node}} = \min\!\left(\frac{\text{CPU}_{\text{cores}} \times f_{\text{fork}}}{1},\;\frac{\text{RAM}_{\text{GB}} \times 1024}{m_{\text{fork}}}\right)
$$

Where $f_{\text{fork}} = 4$ (default forks-per-core) and
$m_{\text{fork}} = 100\;\text{MB}$ (memory per fork).

**Example** — 64-core, 512 GB execution node:

$$
C_{\text{cpu}} = 64 \times 4 = 256
$$

$$
C_{\text{mem}} = \frac{512 \times 1024}{100} = 5242 \gg C_{\text{cpu}}
$$

$$
\boxed{C_{\text{node}} = 256 \text{ concurrent forks}}
$$

A cluster with 4 such nodes delivers $4 \times 256 = 1024$ concurrent forks —
enough to drive a 1 000-node playbook at forks=1 000 without queuing.

### 3.2 Receptor Link Reliability — Message Delivery Probability

Receptor uses **QUIC** (UDP) with acknowledgment. For a mesh with $h$ hops,
each link having reliability $r$:

$$
P_{\text{delivery}} = r^h
$$

With $r = 0.9999$ (four-nines link) and $h = 3$ hops:

$$
P = 0.9999^3 = 0.9997
$$

**99.97 %** — acceptable. Increase $r$ by enabling TLS+QUIC forward-error
correction; increase $h$ only when DMZ segmentation demands it.

### 3.3 PostgreSQL Connection Pool — Queueing

AWX generates $J$ concurrent job SQL transactions. Without pooling:

$$
\lambda = J,\quad \mu = \frac{1}{\bar{t}_{\text{query}}}
$$

With M/M/c queueing ($c$ = max_connections):

$$
\rho = \frac{\lambda}{c \cdot \mu}
$$

PgBouncer in **transaction mode** with pool_size $= c$ reduces $\lambda_{\text{postgres}}$ by
a factor of the multiplexing ratio $m$:

$$
\lambda_{\text{effective}} = \frac{J}{m}, \quad m \approx 8\text{–}50
$$

This prevents "too many connections" errors when $J > 200$ concurrent jobs.

---

## 4. Deep Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (AWX Namespace)               │
│                                                                     │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐   │
│  │  awx-web     │   │  awx-task    │   │  awx-rsyslog         │   │
│  │  (Django/    │   │  (Celery     │   │  (structured log     │   │
│  │   Daphne)    │──►│   workers)   │   │   aggregation)       │   │
│  │  :8052       │   │  N replicas  │   └──────────────────────┘   │
│  └──────┬───────┘   └──────┬───────┘                              │
│         │                  │                                        │
│         ▼                  ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │            PostgreSQL (Patroni HA)                            │  │
│  │  primary ──sync_standby ──async_standby  [3 pods]            │  │
│  │  PgBouncer sidecar: pool_size=50, max_client_conn=500        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │               Receptor Control-Plane Node                     │  │
│  │   receptor --node-id=control1 --tcp-listener=:27199          │  │
│  └───────────────────┬──────────────────────────────────────────┘  │
└───────────────────────┼─────────────────────────────────────────────┘
                        │  Receptor QUIC/TLS mesh
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │ exec-node1 │ │ exec-node2 │ │ hop-node-A │
   │ GPU zone A │ │ GPU zone B │ │ DMZ relay  │
   │ cap=256    │ │ cap=256    │ │ cap=0 (hop)│
   └────────────┘ └────────────┘ └────────────┘
         │                              │
         ▼                              ▼
   [bubblewrap]                   [bubblewrap]
   isolated job                   isolated job
   container                      container
```

**Node Types:**
| Type | Role | Has Capacity |
|------|------|-------------|
| Control | AWX API, scheduling | Yes (hybrid) |
| Execution | Runs playbooks in EE | Yes |
| Hop | Routes Receptor traffic only | No |

---

## 5. Concrete Production Lab

```python
#!/usr/bin/env python3
"""
awx_ha_cluster_provisioner.py
Generates a complete AWX HA deployment spec:
  - AWX Operator CR manifest
  - PostgreSQL Patroni StatefulSet
  - Receptor peer configuration
  - PgBouncer connection pool config

Run: python3 awx_ha_cluster_provisioner.py
"""

import json, textwrap, os
from dataclasses import dataclass, field
from typing import List

# ── Data models ──────────────────────────────────────────────────────────────

@dataclass
class ExecutionNode:
    name: str
    hostname: str
    cpu_cores: int
    ram_gb: int
    node_type: str = "execution"  # execution | hop | control
    fork_per_core: int = 4
    mem_per_fork_mb: int = 100

    @property
    def capacity(self) -> int:
        if self.node_type == "hop":
            return 0
        cpu_cap = self.cpu_cores * self.fork_per_core
        mem_cap = (self.ram_gb * 1024) // self.mem_per_fork_mb
        return min(cpu_cap, mem_cap)


@dataclass
class AWXClusterSpec:
    namespace: str = "awx"
    awx_version: str = "23.9.0"
    replicas_web: int = 3
    replicas_task: int = 3
    pg_replicas: int = 3
    pg_max_connections: int = 200
    pgbouncer_pool_size: int = 50
    execution_nodes: List[ExecutionNode] = field(default_factory=list)

    @property
    def total_cluster_capacity(self) -> int:
        return sum(n.capacity for n in self.execution_nodes)


# ── Manifest generators ───────────────────────────────────────────────────────

class AWXManifestGenerator:

    def __init__(self, spec: AWXClusterSpec):
        self.spec = spec

    def awx_operator_cr(self) -> dict:
        """Generate the AWX Custom Resource for the Operator."""
        return {
            "apiVersion": "awx.ansible.com/v1beta1",
            "kind": "AWX",
            "metadata": {
                "name": "awx-ha",
                "namespace": self.spec.namespace,
            },
            "spec": {
                "replicas": self.spec.replicas_task,
                "web_replicas": self.spec.replicas_web,
                "task_replicas": self.spec.replicas_task,
                "auto_upgrade": False,
                "image_version": self.spec.awx_version,
                "service_type": "ClusterIP",
                "postgres_configuration_secret": "awx-postgres-secret",
                "resource_requirements": {
                    "requests": {"cpu": "2000m", "memory": "4Gi"},
                    "limits":   {"cpu": "4000m", "memory": "8Gi"},
                },
                "extra_settings": [
                    {"setting": "RECEPTOR_PEERS_FROM_CONTROL_PLANE_NODES",
                     "value": "true"},
                    {"setting": "RECEPTOR_WORK_PUBLIC_KEY",
                     "value": "/etc/receptor/tls/ca/tls.crt"},
                ],
            },
        }

    def patroni_configmap(self) -> dict:
        """Generate Patroni bootstrap configuration."""
        patroni_cfg = {
            "bootstrap": {
                "dcs": {
                    "ttl": 30,
                    "loop_wait": 10,
                    "retry_timeout": 10,
                    "maximum_lag_on_failover": 1048576,
                    "postgresql": {
                        "use_pg_rewind": True,
                        "parameters": {
                            "max_connections": self.spec.pg_max_connections,
                            "shared_buffers": "4GB",
                            "wal_level": "replica",
                            "synchronous_commit": "on",
                            "synchronous_standby_names": "ANY 1 (*)",
                        },
                    },
                }
            }
        }
        return {
            "apiVersion": "v1",
            "kind": "ConfigMap",
            "metadata": {"name": "patroni-config", "namespace": self.spec.namespace},
            "data": {"patroni.yml": json.dumps(patroni_cfg, indent=2)},
        }

    def pgbouncer_ini(self) -> str:
        """Generate PgBouncer configuration."""
        return textwrap.dedent(f"""\
            [databases]
            awx = host=postgres-primary port=5432 dbname=awx

            [pgbouncer]
            pool_mode = transaction
            listen_port = 5432
            listen_addr = 0.0.0.0
            auth_type = md5
            auth_file = /etc/pgbouncer/userlist.txt
            admin_users = awx
            pool_size = {self.spec.pgbouncer_pool_size}
            max_client_conn = 500
            server_pool_size = {self.spec.pgbouncer_pool_size}
            server_idle_timeout = 600
            log_connections = 1
            log_disconnections = 1
        """)

    def receptor_config(self, node: ExecutionNode) -> str:
        """Generate receptor.conf for an execution/hop node."""
        lines = [
            f"- node:",
            f"    id: {node.name}",
            f"",
            f"- log-level: info",
            f"",
            f"- tls-client:",
            f"    name: tlsclient",
            f"    cert: /etc/receptor/tls/tls.crt",
            f"    key: /etc/receptor/tls/tls.key",
            f"    rootcas: /etc/receptor/tls/ca/tls.crt",
            f"    insecureskipverify: false",
            f"",
        ]
        if node.node_type != "control":
            lines += [
                f"- tcp-peer:",
                f"    address: receptor-control.awx.svc.cluster.local:27199",
                f"    tls: tlsclient",
                f"    redial: true",
                f"",
            ]
        if node.node_type in ("execution", "control"):
            lines += [
                f"- work-command:",
                f"    worktype: ansible-runner",
                f"    command: ansible-runner",
                f"    params: worker",
                f"    allowruntimeparams: true",
            ]
        if node.node_type == "hop":
            lines += [
                f"- tcp-listener:",
                f"    port: 27199",
                f"    tls: tlsserver",
            ]
        return "\n".join(lines)

    def capacity_report(self) -> str:
        """Print a capacity planning report."""
        lines = ["=" * 60, "AWX Cluster Capacity Report", "=" * 60]
        total = 0
        for n in self.spec.execution_nodes:
            cap = n.capacity
            total += cap
            cpu_c = n.cpu_cores * n.fork_per_core
            mem_c = (n.ram_gb * 1024) // n.mem_per_fork_mb
            bottleneck = "CPU" if cpu_c <= mem_c else "MEM"
            lines.append(
                f"  {n.name:20s}  type={n.node_type:9s}  "
                f"cap={cap:4d}  bottleneck={bottleneck}"
            )
        lines += ["-" * 60, f"  Total cluster capacity: {total} concurrent forks", "=" * 60]
        return "\n".join(lines)


# ── Verification suite ────────────────────────────────────────────────────────

def test_capacity_formula():
    n = ExecutionNode("exec1", "10.0.0.1", cpu_cores=64, ram_gb=512)
    assert n.capacity == 256, f"Expected 256, got {n.capacity}"

def test_hop_node_zero_capacity():
    n = ExecutionNode("hop1", "10.0.0.9", cpu_cores=8, ram_gb=32,
                      node_type="hop")
    assert n.capacity == 0

def test_mem_bottleneck():
    # 4 cores * 4 = 16 cpu_cap; (8 * 1024)/100 = 81 mem_cap → min=16
    n = ExecutionNode("tiny", "10.0.0.2", cpu_cores=4, ram_gb=8)
    assert n.capacity == 16

def test_total_cluster_capacity():
    nodes = [
        ExecutionNode("e1", "x", 64, 512),
        ExecutionNode("e2", "x", 64, 512),
        ExecutionNode("h1", "x", 8, 32, node_type="hop"),
    ]
    spec = AWXClusterSpec(execution_nodes=nodes)
    assert spec.total_cluster_capacity == 512  # 256+256+0

def test_awx_cr_structure():
    spec = AWXClusterSpec()
    gen = AWXManifestGenerator(spec)
    cr = gen.awx_operator_cr()
    assert cr["kind"] == "AWX"
    assert cr["spec"]["task_replicas"] == 3

def test_pgbouncer_pool_size_in_ini():
    spec = AWXClusterSpec(pgbouncer_pool_size=75)
    gen = AWXManifestGenerator(spec)
    ini = gen.pgbouncer_ini()
    assert "pool_size = 75" in ini

def test_receptor_hop_config():
    spec = AWXClusterSpec()
    gen = AWXManifestGenerator(spec)
    n = ExecutionNode("hop1", "x", 8, 32, node_type="hop")
    cfg = gen.receptor_config(n)
    assert "tcp-listener" in cfg
    assert "work-command" not in cfg

def run_tests():
    tests = [
        test_capacity_formula,
        test_hop_node_zero_capacity,
        test_mem_bottleneck,
        test_total_cluster_capacity,
        test_awx_cr_structure,
        test_pgbouncer_pool_size_in_ini,
        test_receptor_hop_config,
    ]
    passed = 0
    for t in tests:
        try:
            t()
            print(f"  [PASS] {t.__name__}")
            passed += 1
        except AssertionError as e:
            print(f"  [FAIL] {t.__name__}: {e}")
    print(f"\n{passed}/{len(tests)} tests passed.")
    return passed == len(tests)


if __name__ == "__main__":
    # Build a representative cluster spec
    nodes = [
        ExecutionNode("exec-gpu-zone-a", "10.10.1.10", 64, 512),
        ExecutionNode("exec-gpu-zone-b", "10.10.2.10", 64, 512),
        ExecutionNode("exec-gpu-zone-c", "10.10.3.10", 64, 512),
        ExecutionNode("hop-dmz",         "10.10.9.1",  8,  32, node_type="hop"),
    ]
    spec = AWXClusterSpec(execution_nodes=nodes)
    gen  = AWXManifestGenerator(spec)

    print(gen.capacity_report())
    print()

    # Dump AWX CR
    cr_path = "/tmp/awx-cr.json"
    with open(cr_path, "w") as f:
        json.dump(gen.awx_operator_cr(), f, indent=2)
    print(f"AWX CR written → {cr_path}")

    # Dump PgBouncer config
    pb_path = "/tmp/pgbouncer.ini"
    with open(pb_path, "w") as f:
        f.write(gen.pgbouncer_ini())
    print(f"PgBouncer config written → {pb_path}")

    # Dump Receptor configs
    for n in nodes:
        rpath = f"/tmp/receptor-{n.name}.conf"
        with open(rpath, "w") as f:
            f.write(gen.receptor_config(n))
        print(f"Receptor config written → {rpath}")

    print()
    print("── Running verification tests ──")
    run_tests()
```

---

## 6. Comparative Matrix — AWX vs AAP vs Manual Ansible

| Dimension | Manual ansible CLI | AWX (open source) | AAP 2.x (subscription) |
|---|---|---|---|
| HA control plane | ❌ Single point of failure | ✅ K8s multi-replica | ✅ K8s + Red Hat SLA |
| Job isolation | Process fork | bubblewrap container | bubblewrap + SELinux |
| Remote execution | SSH fan-out | Receptor QUIC mesh | Receptor + FIPS-140 |
| RBAC | None | Django RBAC | LDAP/AD + SAML SSO |
| Audit logging | stdout | PostgreSQL job events | SIEM integration |
| Capacity planning | Manual count | Capacity field per node | Capacity + Instance Groups |
| EE management | requirements.yml | AWX EE builder | `aap-setup` EE registry |
| Cost | \$0 | \$0 | ~\$12k/yr per 100 nodes |

---

## 7. SRE Diagnostics Playbook

| Symptom | Root Cause | Diagnostic Command | Remediation |
|---|---|---|---|
| Jobs queued indefinitely | All exec-nodes at capacity | `awx-manage capacity_check` | Scale exec-node Deployment replicas |
| Receptor mesh split | TLS cert expired on hop node | `receptor-ctl --socket /tmp/receptor.sock status` | Renew cert, restart receptor service |
| PostgreSQL `too many connections` | PgBouncer pool exhausted | `SHOW POOLS;` in pgbouncer admin | Increase `pool_size`; add PgBouncer replica |
| AWX task pod OOMKilled | Large playbook stdout flood | `kubectl top pod -n awx` | Increase task pod `limits.memory`; enable `stdout_max_bytes_display` |
| Receptor hop unreachable | Firewall blocks UDP/QUIC 27199 | `receptor-ctl ping <node-id>` | Open port; switch to `tcp-peer` if QUIC blocked |
| Credential vault fetch fails | Vault token TTL expired | `awx-manage run_dispatcher --status` | Rotate AppRole secret-id; increase `ttl` |

---

## 8. Verification Checklist

- [ ] AWX Operator CR deployed with `replicas ≥ 3` for both web and task
- [ ] PostgreSQL Patroni reports `Leader + 1 sync_standby` via `patronictl list`
- [ ] PgBouncer `pool_size` ≥ 50 and `max_client_conn` ≥ 500 in `pgbouncer.ini`
- [ ] All execution nodes show positive capacity in `awx-manage capacity_check`
- [ ] Receptor mesh shows `Connected` status for all peers via `receptor-ctl status`
- [ ] Hop node capacity = 0 and node_type = "hop" in AWX Instance list
- [ ] TLS certificates valid for all Receptor nodes (check expiry ≥ 90 days)
- [ ] `python3 awx_ha_cluster_provisioner.py` prints `7/7 tests passed`
- [ ] Job templates assigned to correct Instance Groups for zone isolation
