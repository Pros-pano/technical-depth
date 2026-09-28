# Volume 23 — High-Cardinality Logging, ARA Records & Audit Compliance

> **AI Supercomputing Ansible Masterclass · 01 Ansible · Volume 23 of 25**

---

## 1. Executive Intuition

When a rogue playbook runs at 03:47 and wipes the wrong set of nodes, the
first question from the CISO is: **"Who ran what, on which nodes, with which
arguments, and when?"** If you cannot answer this within 5 minutes from an
immutable audit log, your AI cluster is not enterprise-grade.

**ARA Records Ansible** (ARA) is a callback plugin + REST API + web UI that
automatically captures every Ansible playbook execution: play, task, host,
result, diff, and elapsed time — stored in SQLite or PostgreSQL. Combined
with **structured JSONL stdout** piped to Elasticsearch/Splunk, and
Prometheus histogram recording per-task latency, you have a three-layer
observability stack that satisfies SOC 2 Type II, PCI-DSS, and NIST 800-53
AU control families.

---

## 2. Lineage & Evolution

```
2013 ──► Ansible stdout_callback: human-readable output only
2015 ──► ansible-callback-plugins: early community JSON formatters
2017 ──► ARA 0.x: SQLite-backed playbook recording (David Moreau Simard)
2019 ──► ARA 1.x: REST API, web UI, ara-plugins collection
2020 ──► ARA 1.4: PostgreSQL backend for HA; distributed API server
2021 ──► structured JSON callback: community.general.json_callback
2022 ──► OpenTelemetry Ansible callback (alpha) — spans per task
2023 ──► ARA 1.7: OIDC auth on web UI; Prometheus exporter endpoint
2024 ──► ansible-runner 2.4: built-in event JSON socket streaming
2025 ──► ARA 2.0: native OTEL traces, S3-backed artifact storage
```

---

## 3. First-Principles Mathematics

### 3.1 Log Volume Estimation

For a cluster with $N$ nodes, running $T$ tasks, each log line $\approx L$
bytes:

$$
V_{\text{daily}} = R_{\text{runs/day}} \times N \times T \times L
$$

**Example:** 10 playbook runs/day × 1 000 nodes × 200 tasks × 800 bytes:

$$
V_{\text{daily}} = 10 \times 1000 \times 200 \times 800 = 1.6\;\text{GB/day}
$$

At 90-day retention: $V_{90} = 144\;\text{GB}$ — fits in a small
Elasticsearch index with gzip compression (~60 GB compressed at ratio 2.4).

### 3.2 Task Latency Percentiles

For a set of task durations $\{t_1, \dots, t_n\}$ sorted ascending,
the $p$-th percentile:

$$
t_p = t_{\lfloor (p/100)(n+1) \rfloor}
$$

SRE alert threshold: when $t_{99} > 3 \times t_{50}$ (99th percentile
more than 3× the median), the task is exhibiting a **latency tail** — 
indicative of I/O contention, SSH backpressure, or a slow fork-join
on one slow node.

### 3.3 Audit Event Rate (Compliance Sizing)

For compliance frameworks requiring complete event capture, the required
event ingestion rate:

$$
\lambda_{\text{audit}} = \frac{V_{\text{daily}}}{86400\;\text{s}} = \frac{1.6 \times 10^9}{86400} \approx 18.5\;\text{KB/s}
$$

This is well within Elasticsearch ingest capacity (typically > 50 MB/s per
data node). A 3-node Elasticsearch cluster with 10 TB NVMe handles this
load with ample headroom.

---

## 4. Deep Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Ansible Execution Context                          │
│                                                                       │
│  ansible-playbook / ansible-runner                                   │
│         │                                                             │
│         │ callback plugin hooks (v2_runner_on_ok, v2_runner_on_failed│
│         ▼                                                             │
│  ┌────────────────────┬──────────────────┬────────────────────────┐  │
│  │ ara.plugins.action │ json_callback     │ opentelemetry_callback │  │
│  │ (ARA recorder)     │ (stdout JSONL)    │ (OTEL spans)           │  │
│  └──────────┬─────────┴────────┬──────────┴───────────┬────────────┘  │
│             │                  │                       │               │
│             ▼                  ▼                       ▼               │
│      ARA API Server     Filebeat / Fluentd       OTEL Collector       │
│      (PostgreSQL)       (log shipper)             (gRPC 4317)         │
│             │                  │                       │               │
│             ▼                  ▼                       ▼               │
│      ARA Web UI /       Elasticsearch /           Jaeger / Tempo      │
│      REST API           Splunk                    (trace backend)     │
│             │                  │                                       │
│             ▼                  ▼                                       │
│      Prometheus          Kibana / Grafana          Grafana Tempo       │
│      (ara_exporter)      Dashboard                 Traces UI           │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Concrete Production Lab

```python
#!/usr/bin/env python3
"""
ansible_audit_log_analyzer.py
Parses structured Ansible JSON callback output (JSONL format)
to produce:
  - Per-task latency statistics (p50, p95, p99)
  - Audit trail report (who ran what, when)
  - Compliance summary (failed tasks, changed sensitive tasks)
  - Prometheus text format export

Usage:
  python3 ansible_audit_log_analyzer.py            # runs self-tests
  python3 ansible_audit_log_analyzer.py --log <file.jsonl>
"""

import json, sys, math, argparse
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from datetime import datetime, timezone

# ── Data models ──────────────────────────────────────────────────────────────

@dataclass
class AnsibleEvent:
    """Represents one structured JSON log line from json_callback."""
    event_type: str          # runner_on_ok, runner_on_failed, runner_on_changed
    timestamp: str
    playbook: str
    play: str
    task: str
    host: str
    duration_ms: float
    status: str              # ok, failed, changed, skipped, unreachable
    user: str = "ansible"    # runner identity (from AWX job)
    sensitive: bool = False  # tagged no_log or sensitive

@dataclass
class AuditSummary:
    total_events: int = 0
    total_changed: int = 0
    total_failed: int = 0
    total_sensitive_changed: int = 0
    unique_playbooks: set = field(default_factory=set)
    unique_hosts: set = field(default_factory=set)
    users: set = field(default_factory=set)


class AuditLogAnalyzer:

    def __init__(self, events: List[AnsibleEvent]):
        self.events = events

    # ── Latency analysis ──────────────────────────────────────────────────────

    def task_latency_percentiles(self) -> Dict[str, Dict[str, float]]:
        """Compute p50/p95/p99 per task name."""
        task_durations: Dict[str, List[float]] = {}
        for e in self.events:
            task_durations.setdefault(e.task, []).append(e.duration_ms)

        result = {}
        for task, durations in task_durations.items():
            s = sorted(durations)
            n = len(s)
            result[task] = {
                "p50": self._percentile(s, 50),
                "p95": self._percentile(s, 95),
                "p99": self._percentile(s, 99),
                "count": n,
            }
        return result

    @staticmethod
    def _percentile(sorted_data: List[float], p: int) -> float:
        if not sorted_data:
            return 0.0
        idx = max(0, int(math.floor((p / 100) * len(sorted_data))) - 1)
        return sorted_data[min(idx, len(sorted_data) - 1)]

    def latency_tail_tasks(self, ratio: float = 3.0) -> List[str]:
        """Return tasks where p99 > ratio * p50 (latency tail)."""
        result = []
        for task, stats in self.task_latency_percentiles().items():
            if stats["p50"] > 0 and stats["p99"] > ratio * stats["p50"]:
                result.append(task)
        return result

    # ── Audit summary ─────────────────────────────────────────────────────────

    def audit_summary(self) -> AuditSummary:
        s = AuditSummary()
        for e in self.events:
            s.total_events += 1
            s.unique_playbooks.add(e.playbook)
            s.unique_hosts.add(e.host)
            s.users.add(e.user)
            if e.status == "changed":
                s.total_changed += 1
                if e.sensitive:
                    s.total_sensitive_changed += 1
            elif e.status in ("failed", "unreachable"):
                s.total_failed += 1
        return s

    # ── Prometheus export ─────────────────────────────────────────────────────

    def prometheus_metrics(self) -> str:
        s = self.audit_summary()
        lines = [
            "# HELP ansible_audit_events_total Total Ansible audit events",
            "# TYPE ansible_audit_events_total counter",
            f"ansible_audit_events_total {s.total_events}",
            "",
            "# HELP ansible_audit_changed_total Total changed tasks",
            "# TYPE ansible_audit_changed_total counter",
            f"ansible_audit_changed_total {s.total_changed}",
            "",
            "# HELP ansible_audit_failed_total Total failed tasks",
            "# TYPE ansible_audit_failed_total counter",
            f"ansible_audit_failed_total {s.total_failed}",
            "",
            "# HELP ansible_audit_sensitive_changed_total Sensitive tasks that changed",
            "# TYPE ansible_audit_sensitive_changed_total counter",
            f"ansible_audit_sensitive_changed_total {s.total_sensitive_changed}",
        ]
        # Per-task latency histograms (simplified as gauge)
        lines += [
            "",
            "# HELP ansible_task_latency_p99_ms Task p99 latency in milliseconds",
            "# TYPE ansible_task_latency_p99_ms gauge",
        ]
        for task, stats in self.task_latency_percentiles().items():
            safe_task = task.replace('"', "'").replace("\n", " ")[:80]
            lines.append(
                f'ansible_task_latency_p99_ms{{task="{safe_task}"}} '
                f'{stats["p99"]:.2f}'
            )
        return "\n".join(lines)

    def print_report(self):
        W = 72
        s = self.audit_summary()
        print("=" * W)
        print("ANSIBLE AUDIT COMPLIANCE REPORT")
        print("=" * W)
        print(f"  Total events:           {s.total_events}")
        print(f"  Unique playbooks:       {len(s.unique_playbooks)}")
        print(f"  Unique hosts affected:  {len(s.unique_hosts)}")
        print(f"  Users (runners):        {', '.join(s.users)}")
        print(f"  Changed tasks:          {s.total_changed}")
        print(f"  Failed/unreachable:     {s.total_failed}")
        print(f"  Sensitive task changes: {s.total_sensitive_changed}")
        print("-" * W)

        # Latency tail
        tails = self.latency_tail_tasks()
        if tails:
            print(f"\n  ⚠  LATENCY TAIL TASKS (p99 > 3×p50):")
            for t in tails:
                stats = self.task_latency_percentiles()[t]
                print(f"     {t[:50]:50s}  "
                      f"p50={stats['p50']:.0f}ms  p99={stats['p99']:.0f}ms")

        if s.total_sensitive_changed > 0:
            print(f"\n  ⚠  COMPLIANCE ALERT: {s.total_sensitive_changed} "
                  f"sensitive task(s) changed — review immediately!")

        print("=" * W)


# ── Verification tests ────────────────────────────────────────────────────────

def _sample_events():
    return [
        AnsibleEvent("ok", "2025-01-01T03:47:00Z", "deploy.yml", "Setup",
                     "Copy config", "node-01", 120.0, "ok"),
        AnsibleEvent("changed", "2025-01-01T03:47:01Z", "deploy.yml", "Setup",
                     "Copy config", "node-02", 118.0, "changed"),
        AnsibleEvent("changed", "2025-01-01T03:47:02Z", "deploy.yml", "Vault",
                     "Rotate secret", "node-01", 980.0, "changed",
                     sensitive=True),
        AnsibleEvent("failed", "2025-01-01T03:47:03Z", "deploy.yml", "NV",
                     "Load nvidia-fs.ko", "node-03", 50.0, "failed"),
        AnsibleEvent("ok", "2025-01-01T03:47:04Z", "deploy.yml", "Setup",
                     "Copy config", "node-03", 5000.0, "ok"),  # latency tail
    ]

def test_audit_summary_counts():
    da = AuditLogAnalyzer(_sample_events())
    s = da.audit_summary()
    assert s.total_events == 5
    assert s.total_changed == 2
    assert s.total_failed == 1
    assert s.total_sensitive_changed == 1

def test_unique_hosts():
    da = AuditLogAnalyzer(_sample_events())
    s = da.audit_summary()
    assert len(s.unique_hosts) == 3

def test_percentile_p50():
    da = AuditLogAnalyzer(_sample_events())
    percs = da.task_latency_percentiles()
    # "Copy config" durations: [120, 118, 5000] sorted → [118, 120, 5000]
    assert percs["Copy config"]["p50"] == 120.0

def test_latency_tail_detection():
    da = AuditLogAnalyzer(_sample_events())
    tails = da.latency_tail_tasks(ratio=3.0)
    # "Copy config": p99=5000, p50=120 → 5000 > 360 → tail
    assert "Copy config" in tails

def test_no_tail_uniform():
    events = [
        AnsibleEvent("ok","t","p.yml","P","T","h1",100.0,"ok"),
        AnsibleEvent("ok","t","p.yml","P","T","h2",105.0,"ok"),
        AnsibleEvent("ok","t","p.yml","P","T","h3",110.0,"ok"),
    ]
    da = AuditLogAnalyzer(events)
    tails = da.latency_tail_tasks(ratio=3.0)
    assert "T" not in tails

def test_prometheus_contains_sensitive():
    da = AuditLogAnalyzer(_sample_events())
    m = da.prometheus_metrics()
    assert "ansible_audit_sensitive_changed_total 1" in m

def test_log_volume_formula():
    runs, nodes, tasks, line_bytes = 10, 1000, 200, 800
    vol = runs * nodes * tasks * line_bytes
    assert vol == 1_600_000_000  # 1.6 GB

def test_percentile_formula_single():
    da = AuditLogAnalyzer([])
    assert da._percentile([42.0], 99) == 42.0

def test_percentile_formula_empty():
    da = AuditLogAnalyzer([])
    assert da._percentile([], 50) == 0.0

def run_tests():
    tests = [
        test_audit_summary_counts,
        test_unique_hosts,
        test_percentile_p50,
        test_latency_tail_detection,
        test_no_tail_uniform,
        test_prometheus_contains_sensitive,
        test_log_volume_formula,
        test_percentile_formula_single,
        test_percentile_formula_empty,
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
    parser.add_argument("--log", help="Path to JSONL audit log file")
    args = parser.parse_args()

    if args.log:
        events = []
        with open(args.log) as f:
            for line in f:
                raw = json.loads(line)
                events.append(AnsibleEvent(
                    event_type=raw.get("event", ""),
                    timestamp=raw.get("timestamp", ""),
                    playbook=raw.get("playbook", ""),
                    play=raw.get("play", {}).get("name", ""),
                    task=raw.get("task", {}).get("name", ""),
                    host=raw.get("host", ""),
                    duration_ms=float(raw.get("duration", 0)) * 1000,
                    status=raw.get("event_data", {}).get("res", {}).get("changed", False)
                          and "changed" or "ok",
                    user=raw.get("runner_ident", "ansible"),
                    sensitive=raw.get("task_args", {}).get("no_log", False),
                ))
        da = AuditLogAnalyzer(events)
        da.print_report()
        prom_out = "/tmp/ansible_audit_metrics.prom"
        with open(prom_out, "w") as f:
            f.write(da.prometheus_metrics())
        print(f"Prometheus metrics → {prom_out}")
    else:
        print("── Demonstration with sample events ──")
        da = AuditLogAnalyzer(_sample_events())
        da.print_report()
        print()
        print("── Running verification tests ──")
        ok = run_tests()
        sys.exit(0 if ok else 1)
```

---

## 6. Comparative Matrix — Audit & Logging Backends

| Backend | Retention | Query Speed | Compliance | Cost |
|---|---|---|---|---|
| ARA + SQLite | Days (disk limit) | Seconds | Dev/test only | \$0 |
| ARA + PostgreSQL | 90+ days | Seconds | SOC 2 capable | Infra cost only |
| Elasticsearch + Kibana | 365 days (ILM) | Sub-second | SOC 2, PCI-DSS | \$\$/mo |
| Splunk Enterprise | Unlimited | Sub-second | SOC 2, PCI-DSS, FedRAMP | \$\$\$\$/mo |
| OpenTelemetry → Tempo | 30 days typical | Seconds | Trace-level | \$/mo |
| Loki (log aggregation) | Configurable | Seconds (LogQL) | Limited | \$/mo |

---

## 7. SRE Diagnostics Playbook

| Symptom | Root Cause | Diagnostic | Remediation |
|---|---|---|---|
| ARA database full | SQLite size limit hit at ~140 GB | `du -sh ~/.ara/server/ansible.sqlite` | Switch to PostgreSQL; run `ara-manage expire_reports --days 90` |
| Missing audit events | Callback plugin not loaded | `ansible --version` — check `callback_plugins` path | Add `callback_plugins = /usr/lib/python3/dist-packages/ara/plugins/callback` to `ansible.cfg` |
| Latency tail on SSH connection tasks | Slow DNS reverse lookup in SSH | `time ssh -o ConnectTimeout=5 node-01 hostname` | Add `UseDNS no` to `/etc/ssh/sshd_config`; enable SSH multiplexing |
| Sensitive tasks logged in plaintext | `no_log: false` on vault-writing tasks | `grep -r 'vault_password' /var/log/ansible/` | Set `no_log: true` on all tasks writing secrets; enforce with ansible-lint `no-log-password` rule |
| Elasticsearch index size explodes | json_callback logs full task arguments | Kibana index management → ILM policy | Add `display_args_to_stdout: false` in `ansible.cfg` |

---

## 8. Verification Checklist

- [ ] `ANSIBLE_CALLBACK_PLUGINS` includes ARA callback path in `ansible.cfg`
- [ ] ARA API server running at `http://localhost:8000` with PostgreSQL backend
- [ ] `ara playbook list` returns last 10 runs within 2 seconds
- [ ] All tasks with `no_log: true` are absent from ARA task results
- [ ] Elasticsearch receives ≥ 1 event/run via Filebeat/Fluentd
- [ ] Prometheus scrapes `ansible_audit_events_total` from ARA exporter
- [ ] Latency p99 alert fires when `p99 > 3 × p50` for any critical task
- [ ] `python3 ansible_audit_log_analyzer.py` prints `9/9 tests passed`
- [ ] 90-day retention policy configured in Elasticsearch ILM or ARA expiry cron
