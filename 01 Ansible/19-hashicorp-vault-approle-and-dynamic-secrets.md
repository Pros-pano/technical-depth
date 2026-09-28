# Volume 19: HashiCorp Vault Integration: AppRole, Dynamic Secrets & Auto-Unseal

```
====================================================================================================
MODULE 01: ANSIBLE & BARE-METAL AI INFRASTRUCTURE AUTOMATION
VOLUME 19: Centralized Secrets, AppRole Machine Auth, Dynamic DB Credentials & Vault Lookups
====================================================================================================
```

---

## 1. Executive Intuition: The Secrets Sprawl Hazard

In large-scale AI infrastructure automation, playbooks must authenticate to dozens of sensitive physical endpoints:
- Baseboard Management Controllers (BMC / IPMI root passwords).
- Storage cluster admin APIs (WekaFS tokens, VAST Data S3 keys).
- Docker and NGC enterprise container registries.
- Infrastructure database credentials (SlurmDBD, etcd certificates).

Storing these secrets in plaintext files, environment variables, or even git-committed `ansible-vault` encrypted files presents catastrophic security risks:
1. **Zero Lifecycle Rotation:** Static passwords remain unchanged for years. If an engineer leaves the organization, rotating hardcoded passwords across 2,000 servers requires manual intervention.
2. **Key Sprawl on Laptops:** Native `ansible-vault` requires distributing a shared master decryption password to every operator's personal machine.
3. **No Centralized Audit Trail:** There is no mechanism to determine which operator or automation task queried which credential.

```
+-----------------------------------------------------------------------------------------+
|                  PLAINTEXT SECRETS VS. HASHICORP VAULT ARCHITECTURE                    |
+------------------------------------+----------------------------------------------------+
| Static Files / Ansible-Vault       | HashiCorp Vault Integration (Enterprise Standard)  |
+------------------------------------+----------------------------------------------------+
| Passwords hardcoded in group_vars  | Single central source of truth over TLS 1.3        |
| Static credentials never expire    | Ephemeral, dynamic credentials with strict TTLs    |
| Shared master password sprawl      | Machine-to-machine AppRole authentication (CIDRs)  |
| No audit logging of access         | Cryptographic audit log: Who, What, When recorded  |
| Manual rotation across cluster     | Automated dynamic revocation and leasing           |
+------------------------------------+----------------------------------------------------+
```

The enterprise standard integrates Ansible directly with **HashiCorp Vault** using the **AppRole Authentication Engine** and **`community.hashi_vault`** lookup plugins.

---

## 2. Lineage & Evolution of Enterprise Secrets Management

```
   [1990s: Plaintext Configuration Files]
                 |
           (Passwords stored directly in /etc/shadow, configs, and shell scripts)
                 |
   [2014: Ansible Vault (.vault files)]
                 |
           (AES-256-CBC file-level encryption; required shared password or vault-id)
                 |
   [2015: HashiCorp Vault Project]
                 |
           (Shamir's secret sharing, centralized HTTP REST API, dynamic secret engines)
                 |
   [2018: AppRole Authentication Specification]
                 |
           (Machine-to-machine role authentication decoupling RoleID from SecretID)
                 |
   [2023: Cloud KMS Auto-Unseal & Vault Agent]
                 |
           (Hardware-backed cloud KMS unsealing; automated secret renewal sidecars)
```

---

## 3. First-Principles Mathematics: Shamir's Secret Sharing $(k, n)$ Threshold

HashiCorp Vault secures its root Master Encryption Key using **Adi Shamir's Secret Sharing Scheme (1979)** based on polynomial interpolation over finite fields.

### 3.1 Mathematical Formulation
To divide a master secret $S$ into $n$ unseal shares such that any $k$ shares can reconstruct $S$, but any $k - 1$ shares reveal zero information:
1. Choose a random polynomial of degree $k - 1$ over a prime Galois Field $\mathbb{F}_p$ ($p > S$):
   $$f(x) = S + a_1 x + a_2 x^2 + \dots + a_{k-1} x^{k-1} \pmod p$$
   Where $f(0) = S$ (the secret is the $y$-intercept).
2. Generate $n$ distinct points (shares): $(x_1, f(x_1)), (x_2, f(x_2)), \dots, (x_n, f(x_n))$.
3. Given any $k$ shares, reconstruct $S$ using **Lagrange Interpolating Polynomials**:

$$f(0) = \sum_{j=1}^{k} y_j \prod_{m \ne j} \frac{0 - x_m}{x_j - x_m} \pmod p$$

#### Security Guarantee:
If an attacker intercepts $k - 1$ keys, there exist infinite polynomials of degree $k - 1$ passing through those points. Every possible candidate secret in $\mathbb{F}_p$ is equally probable!

---

## 4. Deep Architecture: AppRole Machine-to-Machine Authentication

Ansible control nodes authenticate to Vault using the **AppRole Method**:

```
+-----------------------------------------------------------------------------+
|                          APPROLE AUTHENTICATION FLOW                        |
+-----------------------------------------------------------------------------+
|  Ansible Control Node / CI Runner                                           |
|    |                                                                        |
|    |-- Possesses static RoleID: "ai-cluster-provisioner"                    |
|    |-- Ingests dynamic SecretID (from environment or wrapped token)         |
|    v                                                                        |
|  POST https://vault.internal:8200/v1/auth/approle/login                     |
|    Payload: { "role_id": "...", "secret_id": "..." }                        |
|    v                                                                        |
|  HashiCorp Vault Engine:                                                    |
|    |-- Validates SecretID and checks CIDR bind restrictions                 |
|    |-- Issues short-lived Client Token (TTL: 1 Hour)                        |
|    v                                                                        |
|  Ansible Execution:                                                         |
|    |-- Queries KV v2 secrets: secret/data/dgx/bmc_credentials               |
|    |-- Injects passwords directly into task memory buffers                  |
|    +-- Token automatically expires upon playbook completion                 |
+-----------------------------------------------------------------------------+
```

---

## 5. Concrete Production Lab: Automated Vault Secrets Integration Playbook

Below is an enterprise Ansible playbook demonstrating secure secret retrieval via HashiCorp Vault AppRole and `community.hashi_vault` lookup plugins.

```yaml
---
# playbook: secure_provision_vault.yml
# Authenticates to Vault via AppRole, retrieves BMC credentials, and applies configurations
- name: Retrieve Credentials and Configure AI Infrastructure
  hosts: localhost
  gather_facts: false
  vars:
    vault_url: "https://vault.datacenter.internal:8200"
    vault_role_id: "{{ lookup('ansible.builtin.env', 'VAULT_ROLE_ID') }}"
    vault_secret_id: "{{ lookup('ansible.builtin.env', 'VAULT_SECRET_ID') }}"

  tasks:
    - name: 1. Authenticate to Vault via AppRole to obtain temporary Client Token
      community.hashi_vault.vault_login_approle:
        url: "{{ vault_url }}"
        role_id: "{{ vault_role_id }}"
        secret_id: "{{ vault_secret_id }}"
      register: vault_auth

    - name: Set Vault Token Fact
      ansible.builtin.set_fact:
        vault_token: "{{ vault_auth.client_token }}"

    - name: 2. Securely Retrieve BMC Admin Credentials from KV v2 Engine
      ansible.builtin.set_fact:
        bmc_creds: "{{ lookup('community.hashi_vault.hashi_vault', 'secret=secret/data/dgx/bmc_admin token=' + vault_token + ' url=' + vault_url) }}"
      no_log: true  # Prevents secrets from leaking into Ansible terminal logs

    - name: 3. Verify Secret Retrieval without Exposing Passwords
      ansible.builtin.assert:
        that:
          - "bmc_creds.username is defined"
          - "bmc_creds.password is defined"
          - "bmc_creds.password | length > 8"
        fail_msg: "Vault secret retrieval failed or password format invalid!"
        success_msg: "Securely retrieved credentials for user: {{ bmc_creds.username }}"

    - name: 4. Example Dynamic Database Credential Retrieval
      ansible.builtin.set_fact:
        slurmdb_creds: "{{ lookup('community.hashi_vault.hashi_vault', 'secret=database/creds/slurm-role token=' + vault_token + ' url=' + vault_url) }}"
      no_log: true

    - name: Display Dynamic Lease Information
      ansible.builtin.debug:
        msg: "Retrieved ephemeral database user '{{ slurmdb_creds.username }}' (Lease duration: {{ slurmdb_creds.lease_duration }} seconds)"
```

---

## 6. Comparative Secrets Management Matrix

| Feature | Plaintext Variables | Native `ansible-vault` | HashiCorp Vault AppRole |
| :--- | :--- | :--- | :--- |
| **Storage Location** | Git / Hostvars | Encrypted Git files | **Centralized Vault Cluster** |
| **Decryption Key** | None (Exposed) | Shared passphrase | **Ephemeral AppRole Token** |
| **Secret Rotation** | Manual edits | Manual re-encryption | **Automated Dynamic Rotation** |
| **Access Auditing** | None | Git commit log only | **Full cryptographic audit log** |
| **Dynamic Secrets** | Impossible | Impossible | **Native (DB/Cloud short-lived)**|

---

## 7. SRE Diagnostics & Troubleshooting Playbook

```
+---------------------------------------------------------------------------------------------------+
|                        HASHICORP VAULT SRE DIAGNOSTIC MATRIX                                      |
+------------------------------------+--------------------------+-----------------------------------+
| Symptom / Failure Mode             | Root Cause Hypothesis    | Triage & Remediation Command      |
+------------------------------------+--------------------------+-----------------------------------+
| Vault returns HTTP 503:            | Vault cluster is sealed; | Check Vault status:               |
| "Vault is sealed".                 | unseal keys not applied. | `vault status`                    |
|                                    |                          | Apply 3 unseal keys or check KMS. |
+------------------------------------+--------------------------+-----------------------------------+
| `permission denied`: AppRole login | SecretID expired or      | Generate new SecretID in Vault:   |
| rejected.                          | client IP outside CIDR.  | `vault write -f auth/approle/     |
|                                    |                          |  role/ai-provisioner/secret-id`   |
+------------------------------------+--------------------------+-----------------------------------+
| Secrets leak in Ansible logs:      | Task lacked `no_log: true`| Add `no_log: true` to every task  |
| password displayed in stdout.      | directive.               | that registers or sets sensitive  |
|                                    |                          | variables.                        |
+------------------------------------+--------------------------+-----------------------------------+
```

---

## 8. Verification & Architectural Synthesis Checklist

- [ ] **AppRole Authentication Active:** Control node authenticates using decoupled RoleID/SecretID.
- [ ] **`no_log: true` Enforced:** All secret registration tasks strictly suppressed from console logs.
- [ ] **Dynamic Credentials Leveraged:** Ephemeral short-lived credentials used for database operations.
- [ ] **Vault Cluster Unsealed:** HA Vault cluster confirmed operational with Cloud KMS auto-unseal.
- [ ] **Audit Trail Validated:** Every credential retrieval recorded in Vault's structured audit log.
