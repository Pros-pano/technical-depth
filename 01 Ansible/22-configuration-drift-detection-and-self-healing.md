# Volume 22 — Configuration Drift Detection & Self-Healing

> **AI Supercomputing Ansible Masterclass · 01 Ansible · Volume 22 of 25**

---

## 1. Executive Intuition

An AI cluster certified at 09:00 Monday can quietly drift into an unsafe
state by 15:00 Tuesday. A sysadmin manually loads `nvidia_peermem` to debug
a job. A kernel update in a nightly cron pulls in a new `kernel-headers`
version that mismatches the pinned NVIDIA driver. A storage team bumps MTU
to 1500 on a RoCE interface. None of these generate alerts.

**Configuration drift** is the silent accumulation of deviation between
declared state (Ansible playbooks) and actual state (running nodes). In AI
clusters this directly translates to: degraded GPU-to-GPU bandwidth,
unexpected NCCL fallbacks from RDMA to TCP, checkpointing failures, and
XID 79 GPU reset storms.

This volume covers Ansible's `--check` mode, `--diff` mode, continuous drift
detection via scheduled AWX jobs, self-healing remediation playbooks, and a
complete Python lab that produces a production-grade drift audit report.

---

## 2. Lineage & Evolution

```
2013 ──► Ansible 1.x: --check mode introduced (dry-run)
2014 ──► --diff flag: show textual changes to files/templates
2017 ──► AWX 1.x: scheduled jobs enable periodic --check runs
2019 ──► Ansible 2.8: check_mode per-task override
2020 ──► ARA Records: drift tracking via job history database
2021 ──► AAP 2.0: Compliance job templates with drift scoring
2022 ──► ansible-lint check-mode rule enforcement
2023 ──► community.general drift_report callback plugin (alpha)
2024 ──► OpenTelemetry spans emitted per-task for drift dashboards
2025 ──► Autonomous self-healing via AWX workflow job triggers
```

---

## 3. First-Principles Mathematics

### 3.1 Drift Score

For a cluster of $N$ nodes, each with $M$ asserted configuration items,
define the drift matrix $D_{n,m} \in \{0, 1\}$ where 1 = drifted:

$$
\text{Drift}_n = \frac{\sum_{m=1}^{M} D_{n,m}}{M}
$$

Cluster-wide drift score:

$$
\Delta_{\text{cluster}} = \frac{1}{N} \sum_{n=1}^{N} \text{Drift}_n
$$

Alert threshold: $\Delta_{\text{cluster}} > 0.02$ (more than 2% of
configuration items drifted cluster-wide) triggers a remediation job.

### 3.2 Mean Time to Drift (MTTD)

$$
\text{MTTD} = \frac{1}{\lambda_{\text{drift}}}
$$

Where $\lambda_{\text{drift}}$ is the empirical drift introduction rate
(events per hour). For a 1 000-node GPU cluster with typical human
operations, $\lambda_{\text{drift}} \approx 0.1\;\text{hr}^{-1}$:

$$
\text{MTTD} = \frac{1}{0.1} = 10\;\text{hours}
$$

With a 1-hour drift scan interval, expected drift items caught per scan:

$$
\mathbb{E}[\text{caught}] = \lambda_{\text{drift}} \times 1\;\text{hr} \times N = 0.1 \times 1000 = 100
$$

### 3.3 Self-Healing Convergence

With $k$ drifted nodes each requiring $t_r$ seconds to remediate, and
Ansible forks $= f$:

$$
T_{\text{remediate}} = \left\lceil \frac{k}{f} \right\rceil \times t_r
$$

With $k = 50$, $f = 25$ forks, $t_r = 120\;\text{s}$:

$$
T = \lceil 50/25 \rceil \times 120 = 2 \times 120 = 240\;\text{s}
$$

50 drifted nodes converge in **4 minutes**.

---

## 4. Deep Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│              Continuous Drift Detection Loop (AWX)                 │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ Scheduled Job Template (every 60 min)                    │     │
│  │   playbook: drift_audit.yml                              │     │
│  │   flags: --check --diff                                  │     │
│  │   tags: drift_critical                                   │     │
│  └──────────────────────┬───────────────────────────────────┘     │
│                          │                                         │
│                    check mode run                                  │
│                          │                                         │
│       ┌──────────────────▼──────────────────┐                     │
│       │ Callback Plugin: drift_collector     │                     │
│       │ - accumulates changed_tasks per host │                     │
│       │ - writes JSON report to /tmp/drift/  │                     │
│       └──────────────────┬──────────────────┘                     │
│                          │                                         │
│              ┌───────────▼────────────┐                           │
│              │ Drift Score Evaluator  │                           │
│              │ Δ_cluster > 0.02?      │                           │
│              └──────┬────────┬────────┘                           │
│                     │ YES    │ NO                                  │
│                     ▼        ▼                                     │
│            Trigger          Log OK to                              │
│            Remediation      Prometheus                             │
│            Workflow         (drift_score gauge)                    │
│                 │                                                  │
│    ┌────────────▼────────────────────┐                            │
│    │ Self-Healing Workflow            │                            │
│    │  Step 1: Notify (#ops-alerts)   │                            │
│    │  Step 2: remediate_drift.yml    │                            │
│    │          (real run, no --check) │                            │
│    │  Step 3: re-run drift_audit.yml │                            │
│    │  Step 4: Assert Δ == 0          │                            │
│    └─────────────────────────────────┘                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## 5. Concrete Production Lab

```python
#!/usr/bin/env python3
"""
ansible_drift_analyzer.py
Parses Ansible JSON callback output (--check mode) to compute per-node
and cluster-wide drift scores, generate remediation priority lists,
and export Prometheus-ready metrics.

Usage:
  python3 ansible_drift_analyzer.py                # run self-tests
  python3 ansible_drift_analyzer.py --report <json_file>
"""

import json, sys, os, argparse
from dataclasses import dataclass, field
from typing import Dict, List, Tuple
from pathlib import Path

# ── Data models ──────────────────────────────────────────────────────────────

@dataclass
class TaskResult:
    task_name: str
    host: str
    status: str          # "changed", "ok", "failed", "skipped"
    diff: str = ""
    is_critical: bool = False

@dataclass
class NodeDrift:
    hostname: str
    total_tasks: int
    changed_tasks: int
    failed_tasks: int
    critical_drift: int  # tasks tagged drift_critical that changed

    @property
    def drift_score(self) -> float:
        if self.total_tasks == 0:
            return 0.0
        return self.changed_tasks / self.total_tasks

    @property
    def severity(self) -> str:
        s = self.drift_score
        if self.failed_tasks > 0 or self.critical_drift > 0:
            return "CRITICAL"
        if s > 0.10:
            return "HIGH"
        if s > 0.02:
            return "MEDIUM"
        if s > 0.00:
            return "LOW"
        return "CLEAN"


class DriftAnalyzer:

    def __init__(self, results: List[TaskResult]):
        self.results = results
        self._node_map: Dict[str, NodeDrift] = {}
        self._analyze()

    def _analyze(self):
        # Count tasks per host
        task_counts: Dict[str, Dict] = {}
        for r in self.results:
            if r.host not in task_counts:
                task_counts[r.host] = {
                    "total": 0, "changed": 0,
                    "failed": 0, "critical": 0
                }
            c = task_counts[r.host]
            c["total"] += 1
            if r.status == "changed":
                c["changed"] += 1
                if r.is_critical:
                    c["critical"] += 1
            elif r.status == "failed":
                c["failed"] += 1

        for host, c in task_counts.items():
            self._node_map[host] = NodeDrift(
                hostname=host,
                total_tasks=c["total"],
                changed_tasks=c["changed"],
                failed_tasks=c["failed"],
                critical_drift=c["critical"],
            )

    @property
    def cluster_drift_score(self) -> float:
        if not self._node_map:
            return 0.0
        return sum(n.drift_score for n in self._node_map.values()) / len(self._node_map)

    def nodes_requiring_remediation(self, threshold: float = 0.0) -> List[NodeDrift]:
        """Return nodes with drift_score > threshold, sorted by severity."""
        SEVERITY_ORDER = {"CRITICAL": 0, "HIGH": 1, "MEDIUM": 2, "LOW": 3, "CLEAN": 4}
        drifted = [n for n in self._node_map.values() if n.drift_score > threshold
                   or n.failed_tasks > 0]
        return sorted(drifted, key=lambda n: SEVERITY_ORDER[n.severity])

    def prometheus_metrics(self) -> str:
        """Emit Prometheus text format metrics."""
        lines = [
            "# HELP ansible_drift_cluster_score Cluster-wide drift score [0..1]",
            "# TYPE ansible_drift_cluster_score gauge",
            f"ansible_drift_cluster_score {self.cluster_drift_score:.6f}",
            "",
            "# HELP ansible_drift_node_score Per-node drift score",
            "# TYPE ansible_drift_node_score gauge",
        ]
        for n in self._node_map.values():
            lines.append(
                f'ansible_drift_node_score{{host="{n.hostname}",'
                f'severity="{n.severity}"}} {n.drift_score:.6f}'
            )
        lines += [
            "",
            "# HELP ansible_drift_critical_items Critical drift items per node",
            "# TYPE ansible_drift_critical_items gauge",
        ]
        for n in self._node_map.values():
            lines.append(
                f'ansible_drift_critical_items{{host="{n.hostname}"}} {n.critical_drift}'
            )
        return "\n".join(lines)

    def print_report(self):
        WIDTH = 72
        print("=" * WIDTH)
        print("ANSIBLE DRIFT AUDIT REPORT")
        print("=" * WIDTH)
        print(f"  Cluster drift score:  {self.cluster_drift_score:.4f}  "
              f"(threshold 0.0200)")
        print(f"  Nodes analyzed:       {len(self._node_map)}")
        needs_fix = self.nodes_requiring_remediation()
        print(f"  Nodes needing fix:    {len(needs_fix)}")
        print("-" * WIDTH)
        print(f"  {'HOSTNAME':<30} {'SCORE':>6} {'CHG':>4} {'FAIL':>4} {'CRIT':>4} SEV")
        print("-" * WIDTH)
        for n in sorted(self._node_map.values(), key=lambda x: -x.drift_score):
            print(f"  {n.hostname:<30} {n.drift_score:>6.4f} "
                  f"{n.changed_tasks:>4} {n.failed_tasks:>4} "
                  f"{n.critical_drift:>4} {n.severity}")
        print("=" * WIDTH)

        if self.cluster_drift_score > 0.02:
            print("\n  ⚠  ALERT: Cluster drift exceeds 2% threshold!")
            print("     Trigger: awx workflow launch --workflow 'Self-Heal Drift'\n")
        else:
            print("\n  ✓  Cluster within drift tolerance.\n")


# ── Verification tests ────────────────────────────────────────────────────────

def _make_results():
    return [
        TaskResult("sysctl net.core.rmem_max", "gpu-node-01", "changed", is_critical=True),
        TaskResult("nvidia driver version pin", "gpu-node-01", "changed"),
        TaskResult("hugepages 1GB config",      "gpu-node-01", "ok"),
        TaskResult("MTU 9000 on rdma0",         "gpu-node-02", "ok"),
        TaskResult("fabric-manager service",    "gpu-node-02", "ok"),
        TaskResult("nvidia driver version pin", "gpu-node-03", "failed"),
    ]

def test_node_drift_score():
    results = _make_results()
    da = DriftAnalyzer(results)
    # gpu-node-01: 2 changed / 3 total = 0.666
    assert abs(da._node_map["gpu-node-01"].drift_score - 2/3) < 1e-9

def test_clean_node():
    results = _make_results()
    da = DriftAnalyzer(results)
    assert da._node_map["gpu-node-02"].drift_score == 0.0
    assert da._node_map["gpu-node-02"].severity == "CLEAN"

def test_critical_severity():
    results = _make_results()
    da = DriftAnalyzer(results)
    assert da._node_map["gpu-node-01"].severity == "CRITICAL"

def test_failed_node_severity():
    results = _make_results()
    da = DriftAnalyzer(results)
    assert da._node_map["gpu-node-03"].severity == "CRITICAL"

def test_cluster_drift_score():
    results = _make_results()
    da = DriftAnalyzer(results)
    # 3 nodes: 0.666, 0.0, 0.0 → avg ≈ 0.222
    assert da.cluster_drift_score > 0.02

def test_nodes_requiring_remediation():
    results = _make_results()
    da = DriftAnalyzer(results)
    needs_fix = da.nodes_requiring_remediation()
    hostnames = [n.hostname for n in needs_fix]
    assert "gpu-node-01" in hostnames
    assert "gpu-node-03" in hostnames
    assert "gpu-node-02" not in hostnames

def test_prometheus_metrics_format():
    results = _make_results()
    da = DriftAnalyzer(results)
    metrics = da.prometheus_metrics()
    assert "ansible_drift_cluster_score" in metrics
    assert "gpu-node-01" in metrics

def test_drift_score_formula():
    changed = 5
    total = 100
    score = changed / total
    assert abs(score - 0.05) < 1e-9

def test_remediation_time_formula():
    import math
    k, f, t_r = 50, 25, 120
    T = math.ceil(k / f) * t_r
    assert T == 240

def run_tests():
    tests = [
        test_node_drift_score,
        test_clean_node,
        test_critical_severity,
        test_failed_node_severity,
        test_cluster_drift_score,
        test_nodes_requiring_remediation,
        test_prometheus_metrics_format,
        test_drift_score_formula,
        test_remediation_time_formula,
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
    parser = argparse.ArgumentParser()
    parser.add_argument("--report", help="Path to Ansible JSON callback output")
    args = parser.parse_args()

    if args.report:
        with open(args.report) as f:
            raw = json.load(f)
        results = [
            TaskResult(
                task_name=item["task"],
                host=item["host"],
                status=item["status"],
                diff=item.get("diff", ""),
                is_critical=item.get("tags", {}).get("drift_critical", False),
            )
            for item in raw
        ]
        da = DriftAnalyzer(results)
        da.print_report()
        prom_out = "/tmp/ansible_drift_metrics.prom"
        with open(prom_out, "w") as f:
            f.write(da.prometheus_metrics())
        print(f"Prometheus metrics → {prom_out}")
    else:
        print("── Running verification tests ──")
        results = _make_results()
        da = DriftAnalyzer(results)
        da.print_report()
        print()
        ok = run_tests()
        sys.exit(0 if ok else 1)
```

---

## 6. Comparative Matrix — Drift Detection Approaches

| Approach | Detection Latency | Remediation | Blast Radius Control | Audit Trail |
|---|---|---|---|---|
| Manual periodic playbook | Hours/days | Manual | None | None |
| AWX scheduled `--check` | 60 min (configurable) | AWX workflow | Instance Groups | PostgreSQL job history |
| Osquery fleet | Near-real-time (15 s) | External (Chef/Puppet) | Query filter | SQLite / Kafka |
| InSpec / Cinc Auditor | On-demand | External | Profile filter | JSON report |
| Prometheus node-exporter + alerting | Real-time (15 s) | Alert → AWX webhook | Alertmanager routes | Prometheus TSDB |
| **Ansible drift_audit.yml (this vol)** | 60 min | Self-healing workflow | Limit + tags | ARA + Prometheus |

---

## 7. SRE Diagnostics Playbook

| Symptom | Root Cause | Diagnostic | Remediation |
|---|---|---|---|
| Drift score spikes to 0.40 cluster-wide | Kernel update pulled new nvidia-headers | `grep changed /tmp/drift/*.json \| sort \| uniq -c \| sort -rn` | Pin kernel with `yum versionlock` or `apt-mark hold` |
| Single node always drifts on sysctl | Local admin manually overriding via `/etc/rc.local` | `--diff` output shows file change each run | Remove rc.local override; enforce via `ansible.posix.sysctl` with `sysctl_file: /etc/sysctl.d/99-ansible.conf` |
| AWX check-mode job shows 0 changed but system is broken | Task uses `changed_when: false` incorrectly | `ansible-playbook --diff --check` from CLI; compare outputs | Fix `changed_when` predicate to reflect actual state change |
| Self-healing loop: fix triggers drift again | Two playbooks fighting over same config | `ara result list --task <id>` to trace | Deduplicate source of truth; single playbook owns each item |
| Drift report shows FAILED tasks | Module incompatibility with new OS minor version | `journalctl -u ansible-runner` on exec node | Pin OS minor version or update collection |

---

## 8. Verification Checklist

- [ ] `drift_audit.yml` runs in `--check --diff` mode without making changes
- [ ] Callback plugin writes per-run JSON drift report to `/tmp/drift/`
- [ ] Prometheus gauge `ansible_drift_cluster_score` exported and scraped
- [ ] AWX workflow triggers remediation when $\Delta_{\text{cluster}} > 0.02$
- [ ] Remediation job uses `--limit @/tmp/drift/drifted_hosts.txt` (only affected nodes)
- [ ] Post-remediation re-audit job asserts `changed_tasks = 0` for all nodes
- [ ] `python3 ansible_drift_analyzer.py` prints `9/9 tests passed`
- [ ] Critical tasks tagged `drift_critical` in playbooks for elevated alerting
- [ ] Drift history retained ≥ 90 days in AWX / ARA for compliance audit
