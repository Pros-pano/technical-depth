# Volume 21 — Ansible Testing, Linting & Molecule CI/CD Gating

> **AI Supercomputing Ansible Masterclass · 01 Ansible · Volume 21 of 25**

---

## 1. Executive Intuition

An untested Ansible role in a GPU cluster is a latent disaster. A single
misconfigured `sysctl` pushed to 800 nodes during a maintenance window can
lock out SSH across the entire fabric in under 3 minutes. The **Molecule +
Testinfra** testing stack brings software-engineering discipline to
infrastructure: you write a scenario, spin an ephemeral container or VM,
run the role, then *assert* the state with Python test functions — the same
way a developer asserts unit tests before merging code.

This volume covers the full testing pyramid for Ansible: `yamllint` → 
`ansible-lint` → `Molecule scenario` → `Testinfra assertions` → `CI/CD
gating` in GitHub Actions and GitLab CI, culminating in a complete Python lab
that generates Molecule scenario boilerplate for any role.

---

## 2. Lineage & Evolution

```
2014 ──► ansible-lint 1.x (basic YAML hygiene checks)
2016 ──► Molecule 1.x (Vagrant-backed role testing)
2018 ──► Molecule 2.x: Docker driver; Testinfra integration
2019 ──► Molecule 3.x: driver plugins (podman, delegated, EC2)
2020 ──► ansible-lint 5.x: profile system (min, basic, production)
2021 ──► Molecule 3.4: scenario matrix, parallel testing
2022 ──► ansible-lint 6.x: FQCN enforcement rules
2023 ──► Molecule 6.x: native pytest integration; auto-discovery
2024 ──► ansible-lint 24.x: yamllint plugin merger; --fix mode
2025 ──► Molecule 7.x: OCI-first driver, Podman 5.x pods
```

---

## 3. First-Principles Mathematics

### 3.1 Testing Pyramid — Cost vs Coverage

Define 4 testing layers with cost $c_i$ and defect-catch rate $d_i$:

| Layer | $c_i$ (s per run) | $d_i$ |
|---|---|---|
| yamllint | 0.5 | 0.20 |
| ansible-lint | 2.0 | 0.35 |
| Molecule (container) | 45 | 0.35 |
| Molecule (VM/metal) | 300 | 0.10 |

Expected defects caught with budget $B$ seconds:

$$
D(B) = \sum_{i : c_i \le B} d_i
$$

For $B = 60\;\text{s}$ (fast CI gate): $D = 0.20 + 0.35 + 0.35 = 0.90$.
Only 10% of defects escape the 60-second gate.

### 3.2 Idempotency Score

A role's idempotency score $\mathcal{I}$ measures drift between two
consecutive runs on an unchanged system:

$$
\mathcal{I} = 1 - \frac{\text{changed tasks}_{\text{run2}}}{\text{total tasks}}
$$

**Perfect idempotency:** $\mathcal{I} = 1.0$.
**Molecule's idempotency check** asserts $\text{changed tasks}_{\text{run2}} = 0$,
i.e., $\mathcal{I} = 1.0$ is a hard gate.

### 3.3 Lint Rule Density

$$
\text{LRD} = \frac{\text{violations}}{\text{KLOC}}
$$

Target: $\text{LRD} < 5$ for production roles (< 5 violations per 1000 lines).
A new role averaging 30 YAML lines per task with 200 tasks = 6 000 lines →
fewer than 30 violations allowed.

---

## 4. Deep Architecture

```
Developer workstation
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  molecule test --scenario-name default                           │
│                                                                  │
│  Phase 1: create         → docker/podman spin ephemeral instance │
│  Phase 2: prepare        → install prerequisites                 │
│  Phase 3: converge       → apply the role under test             │
│  Phase 4: idempotency    → re-run; assert 0 changed tasks        │
│  Phase 5: verify         → pytest + Testinfra assertions         │
│  Phase 6: side_effect    → (optional) inject failure             │
│  Phase 7: cleanup        → destroy container                     │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼  CI/CD Pipeline (GitHub Actions / GitLab CI)
┌────────────────────┐   ┌──────────────┐   ┌──────────────────────┐
│ yamllint           │──►│ ansible-lint │──►│ molecule test (matrix│
│ 0.5s / always      │   │ 2s / always  │   │ scenarios)           │
│ gate: exit 0       │   │ gate: exit 0 │   │ gate: all tests pass │
└────────────────────┘   └──────────────┘   └──────────────────────┘
        │                                            │
        ▼                                            ▼
   PR Blocked                                  Merge allowed
   (red check)                                 (green check)
```

**Molecule Scenario Directory Layout:**
```
roles/
  nvidia_driver/
    tasks/main.yml
    defaults/main.yml
    molecule/
      default/
        molecule.yml        ← driver, platforms, provisioner config
        converge.yml        ← playbook that calls the role
        verify.yml          ← pytest/Testinfra assertions
        prepare.yml         ← pre-role setup (optional)
```

---

## 5. Concrete Production Lab

```python
#!/usr/bin/env python3
"""
molecule_boilerplate_generator.py
Generates a complete Molecule testing scaffold for any Ansible role.

Usage: python3 molecule_boilerplate_generator.py --role nvidia_driver
       python3 molecule_boilerplate_generator.py  (runs self-tests)
"""

import os, sys, textwrap, argparse
from pathlib import Path

# ── Template library ──────────────────────────────────────────────────────────

def molecule_yml(role_name: str, image: str = "ubuntu:22.04") -> str:
    return textwrap.dedent(f"""\
        ---
        dependency:
          name: galaxy

        driver:
          name: podman

        platforms:
          - name: instance-{role_name}
            image: {image}
            pre_build_image: true
            privileged: false
            cgroupns_mode: host
            volumes:
              - /sys/fs/cgroup:/sys/fs/cgroup:rw
            command: /sbin/init

        provisioner:
          name: ansible
          config_options:
            defaults:
              interpreter_python: auto_silent
              callback_whitelist: profile_tasks
          env:
            ANSIBLE_FORCE_COLOR: "1"

        verifier:
          name: ansible

        lint: |
          set -e
          yamllint .
          ansible-lint
    """)


def converge_yml(role_name: str) -> str:
    return textwrap.dedent(f"""\
        ---
        - name: Converge
          hosts: all
          gather_facts: true
          become: true

          roles:
            - role: {role_name}
    """)


def verify_yml(role_name: str, checks: list) -> str:
    """Generate a Testinfra-style verify playbook using ansible.builtin.assert."""
    task_list = ""
    for check in checks:
        task_list += textwrap.dedent(f"""\
            - name: "{check['name']}"
              ansible.builtin.{check['module']}:
                {check['args']}
              {check.get('extra', '')}

        """)
    return textwrap.dedent(f"""\
        ---
        - name: Verify {role_name}
          hosts: all
          gather_facts: true

          tasks:
            {task_list.strip()}
    """)


def yamllint_config() -> str:
    return textwrap.dedent("""\
        ---
        extends: default
        rules:
          line-length:
            max: 160
            level: warning
          truthy:
            allowed-values: ['true', 'false']
        ignore: |
          .tox/
          .cache/
    """)


def ansible_lint_config() -> str:
    return textwrap.dedent("""\
        ---
        profile: production
        warn_list:
          - yaml[line-length]
        skip_list: []
        use_default_rules: true
        offline: false
        verbosity: 1
    """)


def github_ci_workflow(role_name: str) -> str:
    return textwrap.dedent(f"""\
        name: CI — {role_name}

        on:
          push:
            branches: [main, dev]
          pull_request:

        jobs:
          lint:
            runs-on: ubuntu-latest
            steps:
              - uses: actions/checkout@v4
              - uses: actions/setup-python@v5
                with:
                  python-version: "3.11"
              - run: pip install yamllint ansible-lint
              - run: yamllint .
              - run: ansible-lint

          molecule:
            runs-on: ubuntu-latest
            needs: lint
            strategy:
              matrix:
                image:
                  - ubuntu:22.04
                  - rockylinux:9
            steps:
              - uses: actions/checkout@v4
              - uses: actions/setup-python@v5
                with:
                  python-version: "3.11"
              - run: pip install molecule molecule-plugins[podman] ansible pytest testinfra
              - run: molecule test
                env:
                  MOLECULE_IMAGE: ${{{{ matrix.image }}}}
    """)


# ── Scaffold writer ───────────────────────────────────────────────────────────

NVIDIA_DRIVER_CHECKS = [
    {
        "name": "NVIDIA driver module is loaded",
        "module": "command",
        "args": "cmd: lsmod",
        "extra": "register: lsmod_out\n  failed_when: \"'nvidia' not in lsmod_out.stdout\"",
    },
    {
        "name": "nvidia-smi returns exit 0",
        "module": "command",
        "args": "cmd: nvidia-smi",
        "extra": "",
    },
    {
        "name": "DCGM exporter port 9400 is listening",
        "module": "wait_for",
        "args": "port: 9400\n                timeout: 5",
        "extra": "",
    },
]


def scaffold_role(role_name: str, output_dir: str, checks: list = None):
    if checks is None:
        checks = [
            {
                "name": f"Service {role_name} is running",
                "module": "service",
                "args": f"name: {role_name}\n                state: started",
                "extra": "",
            }
        ]

    base = Path(output_dir) / "molecule" / "default"
    base.mkdir(parents=True, exist_ok=True)

    (base / "molecule.yml").write_text(molecule_yml(role_name))
    (base / "converge.yml").write_text(converge_yml(role_name))
    (base / "verify.yml").write_text(verify_yml(role_name, checks))

    # yamllint + ansible-lint configs at role root
    role_root = Path(output_dir)
    (role_root / ".yamllint.yml").write_text(yamllint_config())
    (role_root / ".ansible-lint").write_text(ansible_lint_config())

    # GitHub Actions workflow
    gha_dir = role_root / ".github" / "workflows"
    gha_dir.mkdir(parents=True, exist_ok=True)
    (gha_dir / f"ci-{role_name}.yml").write_text(github_ci_workflow(role_name))

    print(f"[OK] Molecule scaffold generated at: {output_dir}")
    print(f"     molecule/default/molecule.yml")
    print(f"     molecule/default/converge.yml")
    print(f"     molecule/default/verify.yml")
    print(f"     .yamllint.yml")
    print(f"     .ansible-lint")
    print(f"     .github/workflows/ci-{role_name}.yml")


# ── Verification tests ────────────────────────────────────────────────────────

def test_molecule_yml_contains_driver():
    yml = molecule_yml("test_role")
    assert "driver:" in yml
    assert "podman" in yml

def test_converge_yml_references_role():
    yml = converge_yml("my_role")
    assert "my_role" in yml
    assert "become: true" in yml

def test_verify_yml_generates_tasks():
    checks = [{"name": "check x", "module": "command",
               "args": "cmd: echo ok", "extra": ""}]
    yml = verify_yml("r", checks)
    assert "check x" in yml
    assert "ansible.builtin.command" in yml

def test_yamllint_config_max_line():
    cfg = yamllint_config()
    assert "max: 160" in cfg

def test_ansible_lint_profile_production():
    cfg = ansible_lint_config()
    assert "production" in cfg

def test_github_ci_matrix_images():
    wf = github_ci_workflow("myrole")
    assert "ubuntu:22.04" in wf
    assert "rockylinux:9" in wf

def test_idempotency_score_formula():
    total_tasks = 20
    changed_run2 = 0
    score = 1 - changed_run2 / total_tasks
    assert score == 1.0

def test_lint_rule_density():
    violations = 20
    kloc = 6.0  # 6000 lines / 1000
    lrd = violations / kloc
    assert lrd < 5.0, f"LRD={lrd} exceeds threshold"

def test_scaffold_creates_files(tmp_path):
    scaffold_role("test_role", str(tmp_path),
                  checks=[{"name": "ping", "module": "ping",
                           "args": "", "extra": ""}])
    assert (tmp_path / "molecule" / "default" / "molecule.yml").exists()
    assert (tmp_path / ".yamllint.yml").exists()
    assert (tmp_path / ".ansible-lint").exists()

def run_tests():
    import tempfile
    tests = [
        test_molecule_yml_contains_driver,
        test_converge_yml_references_role,
        test_verify_yml_generates_tasks,
        test_yamllint_config_max_line,
        test_ansible_lint_profile_production,
        test_github_ci_matrix_images,
        test_idempotency_score_formula,
        test_lint_rule_density,
        lambda: test_scaffold_creates_files(Path(tempfile.mkdtemp())),
    ]
    passed = 0
    for t in tests:
        name = getattr(t, "__name__", "test_scaffold_creates_files")
        try:
            t()
            print(f"  [PASS] {name}")
            passed += 1
        except AssertionError as e:
            print(f"  [FAIL] {name}: {e}")
    print(f"\n{passed}/{len(tests)} tests passed.")
    return passed == len(tests)


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Molecule boilerplate generator")
    parser.add_argument("--role", help="Role name to scaffold (skip=run tests only)")
    parser.add_argument("--out",  help="Output directory", default=".")
    args = parser.parse_args()

    if args.role:
        checks = NVIDIA_DRIVER_CHECKS if "nvidia" in args.role else None
        scaffold_role(args.role, args.out, checks=checks)
    else:
        print("── Running verification tests ──")
        ok = run_tests()
        sys.exit(0 if ok else 1)
```

---

## 6. Comparative Matrix — Testing Tools

| Tool | Speed | Scope | Auto-fix | GPU/HW aware |
|---|---|---|---|---|
| yamllint | < 1 s | Syntax & style | `--fix` (v1.35) | N/A |
| ansible-lint | 2–5 s | Best practices, FQCN | `--fix` | Partial (custom rules) |
| Molecule (Docker) | 30–90 s | Full role execution | No | No (mocked) |
| Molecule (bare-metal delegated) | 5–15 min | Real hardware | No | ✅ |
| Testinfra (Python) | < 5 s | OS state assertions | No | Via ssh/paramiko |
| pytest-ansible | < 10 s | Module unit tests | No | Yes |

---

## 7. SRE Diagnostics Playbook

| Symptom | Root Cause | Diagnostic | Remediation |
|---|---|---|---|
| Molecule `converge` hangs | PID 1 is not systemd in container | `molecule --debug converge` | Use `command: /sbin/init` and `privileged: true` |
| ansible-lint FQCN violation | Role uses short module names (`copy` not `ansible.builtin.copy`) | `ansible-lint --list-rules | grep fqcn` | Auto-fix: `ansible-lint --fix fqcn` |
| Idempotency check fails | Task always reports `changed` | `molecule idempotency` output | Add `changed_when: false` or fix module logic |
| yamllint `truthy` warning | `yes`/`no` instead of `true`/`false` | `yamllint -d .yamllint.yml .` | Replace `yes`/`no` in all YAML files |
| Molecule image pull timeout | Registry unreachable in CI | `podman pull <image>` manually | Mirror image to internal registry; set `REGISTRY_URL` |

---

## 8. Verification Checklist

- [ ] `yamllint .` exits 0 with no `[error]` lines across all role YAML
- [ ] `ansible-lint` exits 0 with `profile: production` enabled
- [ ] `molecule test` completes all 7 phases (create → cleanup) in < 5 min
- [ ] Idempotency phase reports `0 changed tasks` on second converge run
- [ ] Verify phase has ≥ 3 Testinfra assertions per role
- [ ] GitHub Actions workflow triggers on both push and pull_request events
- [ ] Matrix tests cover ≥ 2 OS platforms (Ubuntu 22.04 + Rocky 9 minimum)
- [ ] `python3 molecule_boilerplate_generator.py` prints `9/9 tests passed`
- [ ] `.ansible-lint` committed to the role repository root
