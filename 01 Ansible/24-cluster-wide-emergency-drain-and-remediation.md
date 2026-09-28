# Volume 24 — Cluster-Wide Emergency Drain, Node Fencing & XID Remediation

> **AI Supercomputing Ansible Masterclass · 01 Ansible · Volume 24 of 25**

---

## 1. Executive Intuition

At 02:13 on a Tuesday, one H100 node begins emitting XID 79 (GPU reset) at
40 Hz. Within 60 seconds, its NCCL ring collapses, the 512-GPU training job
crashes, and 511 healthy GPUs sit idle burning \$12 000/hr. The SRE on call
has 3 minutes to fence the bad node before the next job scheduler wave picks
it up and re-queues against the same broken hardware.

**Ansible emergency response playbooks** automate exactly this: detect the
XID storm via DCGM telemetry, drain the node from Slurm (preventing new job
allocation), fence it from the Kubernetes GPU operator (cordon + taint),
collect diagnostics (nvidia-bug-report, DCGM health check, dmesg), and open
a hardware ticket — all in under 90 seconds, without paging a human.

This volume covers: XID taxonomy, automated fencing choreography, MUNGE-
authenticated Slurm drain over Ansible, GPU reset loop remediation, and a
Python lab implementing the full emergency response engine.

---

## 2. Lineage & Evolution

```
2015 ──► Manual Slurm `scontrol update NodeName=X State=DRAIN`
2017 ──► Ansible playbooks for ad-hoc node drain (community patterns)
2018 ──► DCGM 1.6: health check API → scriptable XID detection
2019 ──► NCCL 2.4: watchdog timeout → job abort on ring collapse
2020 ──► Prometheus DCGM exporter: XID counter as time-series metric
2021 ──► Alertmanager → AWX webhook: automated playbook trigger
2022 ──► NVIDIA Fabric Manager 22.x: NVSwitch isolation API
2023 ──► Slurm 23.11: REST API for drain (no SSH to slurmctld needed)
2024 ──► GPU Operator 24.x: node taint via Kubernetes label-selector
2025 ──► DCGM-Exporter 3.4: XID storm detection with debounce filter
```

---

## 3. First-Principles Mathematics

### 3.1 XID Storm Detection Threshold

DCGM exports `DCGM_FI_DEV_XID_ERRORS` as a monotonic counter. Compute
the rate over a window $w$ seconds:

$$
\dot{X} = \frac{\Delta X}{\Delta t}
$$

Define a storm when:

$$
\dot{X} > \theta_{\text{xid}}\;\text{events/s}
$$

With $\theta_{\text{xid}} = 1$ (1 XID per second sustained over $w = 30$ s),
most transient resets (XID 31, driver recovery) are filtered. Only persistent
failures (XID 79 = GPU reset required) breach the threshold.

### 3.2 Drain Time Budget

Total drain + fence time must be less than the Slurm cycle time $T_{\text{sched}}$
to prevent the scheduler from allocating the faulty node to the next job:

$$
T_{\text{drain}} + T_{\text{fence}} + T_{\text{cordon}} < T_{\text{sched}}
$$

Typical values:
- $T_{\text{drain}} = 10\;\text{s}$ (Slurm drain via REST API)
- $T_{\text{fence}} = 5\;\text{s}$ (NVSwitch isolation via FM)
- $T_{\text{cordon}} = 3\;\text{s}$ (kubectl cordon + taint)
- $T_{\text{sched}} = 60\;\text{s}$ (Slurm scheduling cycle)

$$
10 + 5 + 3 = 18\;\text{s} \ll 60\;\text{s} \quad \checkmark
$$

### 3.3 NCCL Ring Recovery Time

After draining the faulty node, the remaining $N-1$ nodes must reconstruct
the NCCL ring. Ring rebuild time is proportional to the number of remaining
nodes and the all-reduce latency:

$$
T_{\text{rebuild}} \approx 2 \times N_{\text{remaining}} \times \alpha_{\text{IB}}
$$

Where $\alpha_{\text{IB}} \approx 1\;\mu\text{s}$ per hop on InfiniBand. For
$N_{\text{remaining}} = 511$:

$$
T_{\text{rebuild}} \approx 2 \times 511 \times 1\;\mu\text{s} = 1.022\;\text{ms}
$$

NCCL ring rebuild is effectively instantaneous — the dominant cost is the
**job requeue** and **checkpoint restore**, not the ring itself.

---

## 4. Deep Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│            XID Emergency Response Automation                          │
│                                                                       │
│  DCGM Prometheus Exporter                                            │
│    DCGM_FI_DEV_XID_ERRORS{gpu="0",host="node-07"} rate > 1/s        │
│         │                                                             │
│         ▼ Alertmanager fires                                          │
│  AWX Webhook → Emergency Workflow Job                                │
│                                                                       │
│  ┌──── Phase 1: Fence (< 30 s) ──────────────────────────────────┐  │
│  │  Task 1: Slurm drain node                                      │  │
│  │    slurm_rest_api PUT /slurm/v0.0.39/node/{node}              │  │
│  │    state: DRAIN, reason: "XID-79 auto-fence"                   │  │
│  │                                                                 │  │
│  │  Task 2: Kubernetes cordon + taint                             │  │
│  │    kubectl cordon node-07                                       │  │
│  │    kubectl taint nodes node-07 xid-fault:NoSchedule            │  │
│  │                                                                 │  │
│  │  Task 3: NVSwitch isolation (Fabric Manager API)               │  │
│  │    nv-hostengine --isolate node-07                             │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌──── Phase 2: Diagnose (30 s – 5 min) ──────────────────────────┐ │
│  │  Task 4: nvidia-bug-report.sh → /tmp/nv-bugreport-node07.gz   │  │
│  │  Task 5: dcgmi diag -r 3 → full GPU diagnostic                │  │
│  │  Task 6: dmesg -T --level=err → kernel error ring             │  │
│  │  Task 7: Fetch GPU serial numbers for RMA filing              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌──── Phase 3: Notify & Ticket ──────────────────────────────────┐  │
│  │  Task 8: Slack #ops-critical webhook                           │  │
│  │  Task 9: ServiceNow / PagerDuty ticket creation                │  │
│  │  Task 10: Archive diagnostics to GCS / S3                     │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Concrete Production Lab

```python
#!/usr/bin/env python3
"""
gpu_emergency_response_engine.py
Simulates the Ansible-driven GPU emergency response pipeline:
  - XID storm detection from DCGM metrics
  - Fencing decision logic
  - Playbook generation for drain + cordon + diagnose
  - Timeline audit trail

Run: python3 gpu_emergency_response_engine.py
"""

import json, time, sys
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from enum import Enum

# ── XID taxonomy ─────────────────────────────────────────────────────────────

class XIDSeverity(Enum):
    INFO     = "info"
    WARNING  = "warning"
    CRITICAL = "critical"
    FATAL    = "fatal"

XID_CATALOG = {
    8:  ("CPU bus error",       XIDSeverity.CRITICAL),
    31: ("GPU memory page fault", XIDSeverity.WARNING),
    48: ("DBE (Double Bit Error)", XIDSeverity.CRITICAL),
    74: ("NVLink error",        XIDSeverity.CRITICAL),
    79: ("GPU reset required",  XIDSeverity.FATAL),
    92: ("High single-bit ECC", XIDSeverity.WARNING),
    94: ("Uncontained error",   XIDSeverity.FATAL),
}

@dataclass
class DCGMMetricPoint:
    host: str
    gpu_id: int
    xid_code: int
    timestamp: float  # epoch seconds
    counter: int      # cumulative XID counter

@dataclass
class FencingDecision:
    host: str
    xid_code: int
    rate_per_second: float
    severity: XIDSeverity
    should_fence: bool
    reason: str

# ── Storm detector ────────────────────────────────────────────────────────────

class XIDStormDetector:

    def __init__(self,
                 threshold_rate: float = 1.0,
                 window_seconds: float = 30.0):
        self.threshold_rate = threshold_rate
        self.window_seconds = window_seconds
        # {(host, gpu_id): [(timestamp, counter), ...]}
        self._history: Dict[tuple, List] = {}

    def ingest(self, point: DCGMMetricPoint):
        key = (point.host, point.gpu_id)
        self._history.setdefault(key, []).append(
            (point.timestamp, point.counter)
        )
        # Prune old points outside window
        cutoff = point.timestamp - self.window_seconds
        self._history[key] = [
            p for p in self._history[key] if p[0] >= cutoff
        ]

    def rate(self, host: str, gpu_id: int, xid_code: int) -> float:
        key = (host, gpu_id)
        pts = self._history.get(key, [])
        if len(pts) < 2:
            return 0.0
        delta_t = pts[-1][0] - pts[0][0]
        delta_c = pts[-1][1] - pts[0][1]
        if delta_t <= 0:
            return 0.0
        return delta_c / delta_t

    def evaluate(self, point: DCGMMetricPoint) -> FencingDecision:
        self.ingest(point)
        r = self.rate(point.host, point.gpu_id, point.xid_code)
        sev_tuple = XID_CATALOG.get(point.xid_code,
                                    ("Unknown XID", XIDSeverity.WARNING))
        sev = sev_tuple[1]
        desc = sev_tuple[0]

        should_fence = (
            r > self.threshold_rate
            or sev == XIDSeverity.FATAL
        )
        reason = (
            f"XID {point.xid_code} ({desc}): "
            f"rate={r:.2f}/s threshold={self.threshold_rate}/s "
            f"severity={sev.value}"
        )
        return FencingDecision(
            host=point.host,
            xid_code=point.xid_code,
            rate_per_second=r,
            severity=sev,
            should_fence=should_fence,
            reason=reason,
        )


# ── Playbook generator ────────────────────────────────────────────────────────

class EmergencyPlaybookGenerator:

    def __init__(self, slurm_api_url: str = "http://slurmctld:6820"):
        self.slurm_api_url = slurm_api_url

    def fence_playbook(self, host: str, xid_code: int) -> dict:
        """Generate Ansible playbook YAML (as dict) for emergency fencing."""
        return {
            "name": f"Emergency GPU Fence — {host} XID-{xid_code}",
            "hosts": host,
            "gather_facts": False,
            "become": True,
            "vars": {
                "target_node": host,
                "xid_code": xid_code,
                "slurm_api_url": self.slurm_api_url,
                "drain_reason": f"auto-fence XID-{xid_code} {time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime())}",
            },
            "tasks": [
                {
                    "name": "Drain node in Slurm via REST API",
                    "ansible.builtin.uri": {
                        "url": f"{self.slurm_api_url}/slurm/v0.0.39/node/{{{{ target_node }}}}",
                        "method": "POST",
                        "headers": {
                            "Content-Type": "application/json",
                            "X-SLURM-USER-NAME": "slurm",
                        },
                        "body_format": "json",
                        "body": {
                            "state": ["drain"],
                            "reason": "{{ drain_reason }}",
                        },
                        "status_code": [200],
                    },
                    "delegate_to": "localhost",
                },
                {
                    "name": "Cordon Kubernetes node",
                    "ansible.builtin.command": {
                        "cmd": "kubectl cordon {{ target_node }}",
                    },
                    "delegate_to": "localhost",
                },
                {
                    "name": "Taint Kubernetes node — NoSchedule",
                    "ansible.builtin.command": {
                        "cmd": "kubectl taint nodes {{ target_node }} "
                               f"xid-fault=xid-{xid_code}:NoSchedule --overwrite",
                    },
                    "delegate_to": "localhost",
                },
                {
                    "name": "Collect nvidia-bug-report",
                    "ansible.builtin.command": {
                        "cmd": "nvidia-bug-report.sh --output /tmp/nv-bugreport-{{ target_node }}.gz",
                    },
                    "async": 120,
                    "poll": 10,
                },
                {
                    "name": "Collect DCGM diagnostic (level 3)",
                    "ansible.builtin.command": {
                        "cmd": "dcgmi diag -r 3 -j",
                    },
                    "register": "dcgm_diag",
                    "ignore_errors": True,
                },
                {
                    "name": "Collect kernel error dmesg",
                    "ansible.builtin.command": {
                        "cmd": "dmesg -T --level=err,crit,emerg",
                    },
                    "register": "dmesg_errs",
                },
                {
                    "name": "Send Slack alert",
                    "community.general.slack": {
                        "token": "{{ lookup('env','SLACK_BOT_TOKEN') }}",
                        "channel": "#ops-critical",
                        "msg": f":rotating_light: *GPU FENCE* `{{{{ target_node }}}}` "
                               f"XID-{xid_code} — auto-drained. Diagnostics collecting.",
                        "color": "danger",
                    },
                    "delegate_to": "localhost",
                    "ignore_errors": True,
                },
            ],
        }

    def generate_yaml_text(self, host: str, xid_code: int) -> str:
        """Render playbook as YAML text."""
        pb = self.fence_playbook(host, xid_code)
        lines = [
            "---",
            f"- name: \"{pb['name']}\"",
            f"  hosts: {pb['hosts']}",
            f"  gather_facts: {str(pb['gather_facts']).lower()}",
            f"  become: {str(pb['become']).lower()}",
            "",
            "  vars:",
        ]
        for k, v in pb["vars"].items():
            val = f'"{v}"' if isinstance(v, str) else str(v)
            lines.append(f"    {k}: {val}")
        lines += ["", "  tasks:"]
        for t in pb["tasks"]:
            name = t.get("name", "")
            lines.append(f'    - name: "{name}"')
        return "\n".join(lines)


# ── Verification tests ────────────────────────────────────────────────────────

def test_xid_79_always_fatal():
    det = XIDStormDetector()
    # Single XID 79 — FATAL regardless of rate
    p = DCGMMetricPoint("node-07", 0, 79, time.time(), 1)
    dec = det.evaluate(p)
    assert dec.severity == XIDSeverity.FATAL
    assert dec.should_fence

def test_xid_31_below_threshold_no_fence():
    det = XIDStormDetector(threshold_rate=1.0)
    now = time.time()
    # One event in 30s window → rate=0 (need 2 points)
    p = DCGMMetricPoint("node-01", 0, 31, now, 1)
    dec = det.evaluate(p)
    assert not dec.should_fence

def test_xid_31_storm_triggers_fence():
    det = XIDStormDetector(threshold_rate=1.0, window_seconds=30.0)
    now = time.time()
    # Counter goes from 0 to 60 in 30 seconds → rate=2/s > threshold=1
    det.ingest(DCGMMetricPoint("node-02", 0, 31, now - 30, 0))
    p = DCGMMetricPoint("node-02", 0, 31, now, 60)
    dec = det.evaluate(p)
    assert dec.rate_per_second == pytest_approx(2.0, 0.1)
    assert dec.should_fence

def pytest_approx(val, tol):
    """Inline approx check."""
    class _A:
        def __eq__(self, other):
            return abs(other - val) <= tol
    return _A()

def test_rate_zero_single_point():
    det = XIDStormDetector()
    p = DCGMMetricPoint("node-03", 1, 48, time.time(), 5)
    dec = det.evaluate(p)
    assert dec.rate_per_second == 0.0

def test_playbook_contains_drain_task():
    gen = EmergencyPlaybookGenerator()
    pb = gen.fence_playbook("node-07", 79)
    task_names = [t["name"] for t in pb["tasks"]]
    assert any("Drain" in n for n in task_names)

def test_playbook_contains_cordon():
    gen = EmergencyPlaybookGenerator()
    pb = gen.fence_playbook("node-07", 79)
    task_names = [t["name"] for t in pb["tasks"]]
    assert any("Cordon" in n for n in task_names)

def test_playbook_yaml_text():
    gen = EmergencyPlaybookGenerator()
    yaml_txt = gen.generate_yaml_text("node-07", 79)
    assert "node-07" in yaml_txt
    assert "gather_facts: false" in yaml_txt

def test_drain_time_budget():
    # Total fence time must be < Slurm scheduling cycle
    T_drain, T_fence, T_cordon, T_sched = 10, 5, 3, 60
    assert T_drain + T_fence + T_cordon < T_sched

def test_nccl_ring_rebuild_time():
    alpha_ib_us = 1  # microseconds per hop
    n_remaining = 511
    T_rebuild_ms = (2 * n_remaining * alpha_ib_us) / 1000
    assert T_rebuild_ms < 5.0  # well under 5 ms

def run_tests():
    tests = [
        test_xid_79_always_fatal,
        test_xid_31_below_threshold_no_fence,
        test_xid_31_storm_triggers_fence,
        test_rate_zero_single_point,
        test_playbook_contains_drain_task,
        test_playbook_contains_cordon,
        test_playbook_yaml_text,
        test_drain_time_budget,
        test_nccl_ring_rebuild_time,
    ]
    passed = 0
    for t in tests:
        try:
            t()
            print(f"  [PASS] {t.__name__}")
            passed += 1
        except AssertionError as e:
            print(f"  [FAIL] {t.__name__}: {e}")
        except Exception as e:
            print(f"  [FAIL] {t.__name__}: unexpected {type(e).__name__}: {e}")
    print(f"\n{passed}/{len(tests)} tests passed.")
    return passed == len(tests)


if __name__ == "__main__":
    print("── XID Emergency Response Engine — Demo ──")
    det = XIDStormDetector(threshold_rate=1.0)
    gen = EmergencyPlaybookGenerator()

    # Simulate XID 79 storm on node-07 GPU 0
    now = time.time()
    for i, (t_offset, count) in enumerate([(0,0),(10,5),(20,15),(30,40)]):
        p = DCGMMetricPoint("node-07", 0, 79, now - 30 + t_offset, count)
        dec = det.evaluate(p)
        if dec.should_fence:
            print(f"\n  ⚠  FENCE TRIGGERED: {dec.reason}")
            yaml_txt = gen.generate_yaml_text(dec.host, dec.xid_code)
            out = "/tmp/emergency_fence_node07.yml"
            with open(out, "w") as f:
                f.write(yaml_txt)
            print(f"  Playbook written → {out}")
            break

    print()
    print("── Running verification tests ──")
    ok = run_tests()
    sys.exit(0 if ok else 1)
```

---

## 6. Comparative Matrix — XID Codes Requiring Immediate Fence

| XID | Description | Auto-Fence? | Typical Root Cause |
|---|---|---|---|
| 8 | CPU bus error | YES | PCIe root complex failure |
| 31 | GPU memory page fault | Rate-based | Buggy CUDA kernel or corrupted ECC |
| 48 | Double bit ECC error | YES | DRAM failure — GPU replacement needed |
| 74 | NVLink error | YES | NVLink cable or retimer fault |
| 79 | GPU reset required | YES (FATAL) | Hardware hang — RMA candidate |
| 92 | High single-bit ECC | Rate-based | Approaching failure — schedule maintenance |
| 94 | Uncontained error | YES (FATAL) | System integrity compromised — immediate isolation |

---

## 7. SRE Diagnostics Playbook

| Symptom | Root Cause | Diagnostic | Remediation |
|---|---|---|---|
| XID 79 repeating after GPU reset | GPU hardware defect | `nvidia-smi --query-gpu=ecc.errors.uncorrected.aggregate.total --format=csv` | RMA GPU — file hardware ticket; replace node from spare pool |
| Slurm drain succeeds but scheduler re-queues to same node | GRES plugin cache stale | `scontrol show node <node>` → check `State=DRAIN` | Restart `slurmctld` with `systemctl restart slurmctld`; verify `State=DRAIN+DOWN` |
| kubectl cordon fails — API server unreachable | Control plane split | `kubectl cluster-info` from Ansible controller | Use Ansible to cordon via direct etcd write (break-glass); restore control plane |
| nvidia-bug-report.sh hangs > 120 s | Driver wedged — GPU hung | `timeout 90 nvidia-bug-report.sh` | Collect `dmesg` only; schedule driver reload after fence |
| NVSwitch isolation API fails | Fabric Manager not running | `systemctl status nvidia-fabricmanager` | Start FM: `systemctl start nvidia-fabricmanager`; re-issue isolation |

---

## 8. Verification Checklist

- [ ] DCGM Prometheus alert rule fires within 30 s of XID 79 rate > 1/s
- [ ] AWX webhook triggers emergency workflow within 5 s of Alertmanager firing
- [ ] Slurm drain completes with `State=DRAIN` visible in `scontrol show node`
- [ ] Kubernetes node shows `Unschedulable=true` after cordon
- [ ] `xid-fault:NoSchedule` taint present on affected node
- [ ] `nvidia-bug-report.gz` archived to object storage within 3 min
- [ ] Slack #ops-critical receives alert with node name and XID code
- [ ] Total fence time (drain + cordon + taint) < 30 s
- [ ] `python3 gpu_emergency_response_engine.py` prints `9/9 tests passed`
