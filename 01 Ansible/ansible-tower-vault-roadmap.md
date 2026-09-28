# Learning Roadmap: Ansible, Ansible Tower (AWX), and HashiCorp Vault Integration

This guide provides a structured learning pathway, architecture overview, hands-on lab setup commands, and sample code for integrating **Ansible**, **Ansible Tower / AWX**, and **HashiCorp Vault**.

---

## 🗺️ Learning Roadmap

```mermaid
flowchart TD
    subgraph Phase 1: Ansible Core
        A1[Ad-hoc & Inventory] --> A2[Playbooks & Tasks]
        A2 --> A3[Roles & Collections]
        A3 --> A4[ansible-vault native encryption]
    end

    subgraph Phase 2: HashiCorp Vault Core
        V1[Vault Architecture & Engines] --> V2[KV v2 Secrets Engine]
        V2 --> V3[Policies & ACLs]
        V3 --> V4[Authentication: AppRole & Tokens]
    end

    subgraph Phase 3: Ansible + Vault
        AV1[Install community.hashi_vault & hvac] --> AV2[Lookup Plugin hashi_vault]
        AV2 --> AV3[Module: vault_kv2_get]
        AV3 --> AV4[AppRole Auth in Playbooks]
    end

    subgraph Phase 4: Ansible Tower / AWX
        T1[AWX Architecture & RBAC] --> T2[Inventories & Projects]
        T2 --> T3[Credentials & Credential Types]
        T3 --> T4[Job Templates & Workflows]
    end

    subgraph Phase 5: Tower + Vault Enterprise Pattern
        TV1[HashiCorp Vault Credential in Tower] --> TV2[Credential Linking]
        TV2 --> TV3[Dynamic SSH / Secret Injection]
        TV3 --> TV4[Production RBAC & Zero-Trust]
    end

    Phase 1 --> Phase 3
    Phase 2 --> Phase 3
    Phase 3 --> Phase 4
    Phase 4 --> Phase 5
```

---

## ⏱️ Step-by-Step Breakdown

### Phase 1: Ansible Core Mastery
- **Goals**: Understand how Ansible connects, executes, and parses host variables.
- **Key Concepts**:
  - `inventory.ini` and `group_vars`/`host_vars`.
  - Modules (`command`, `copy`, `apt`, `template`, `debug`).
  - Tasks, handlers, facts (`ansible_facts`), conditionals (`when`), loops (`loop`).
  - Native `ansible-vault` (encrypting files and strings with a passphrase) to understand why centralized secret management (HashiCorp Vault) is needed for teams.

### Phase 2: HashiCorp Vault Fundamentals
- **Goals**: Run Vault locally and master secrets retrieval.
- **Key Concepts**:
  - Storage engines, KV v1 vs KV v2 (versioned).
  - Paths: `secret/data/<path>` (KV v2 read) vs `secret/<path>`.
  - Policies (HCL defining `read`, `list`, `write` permissions).
  - Auth methods: `token`, `userpass`, and especially **`approle`** (machine-to-machine authentication with `role_id` and `secret_id`).

### Phase 3: Connecting Ansible with HashiCorp Vault
- **Goals**: Fetch dynamic and static secrets inside playbooks without hardcoding credentials.
- **Key Concepts**:
  - Python library: `hvac`.
  - Collection: `community.hashi_vault`.
  - Lookup plugins vs Action modules:
    - `lookup('community.hashi_vault.hashi_vault', ...)`
    - `community.hashi_vault.vault_kv2_get` module.
  - Safe error handling and masking sensitive task outputs with `no_log: true`.

### Phase 4: Ansible Tower / AWX
- **Goals**: Understand enterprise orchestration, centralization, and RBAC.
- **Key Concepts**:
  - Organization, Teams, Users.
  - Projects (Git sync).
  - Inventories & Dynamic Inventories.
  - Custom Execution Environments (EE) containing `hvac` and collection dependencies.
  - Job Templates, Surveys, and Notifications.

### Phase 5: Tower + HashiCorp Vault Integration
- **Goals**: Never store secrets in Tower or Git; resolve secrets at runtime.
- **Key Concepts**:
  - Native Tower Credential Type: **HashiCorp Vault Secret Lookup**.
  - Credential Linking: A Tower Machine Credential (SSH password/key or sudo password) links to a Vault Credential.
  - Runtime resolution: Tower fetches the secret from Vault when launching the job, injects it into memory, and wipes it after execution.

---

## 🛠️ Hands-on Quickstart Commands

### 1. Prerequisites (Control Machine)

Install Ansible and the `hvac` library (required by `community.hashi_vault`):

```bash
# Install Python packages
pip install ansible hvac

# Install the official HashiCorp Vault Ansible collection
ansible-galaxy collection install community.hashi_vault
```

### 2. Start a Local Vault Dev Server (Docker or Binary)

Using Docker:
```bash
docker run -d \
  --name dev-vault \
  -p 8200:8200 \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=myroottoken' \
  -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' \
  hashicorp/vault:latest
```

Set environment variables:
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='myroottoken'
```

### 3. Write Test Secrets to Vault

```bash
# Check Vault status
vault status

# Write a secret to the KV v2 engine
vault kv put secret/dgx/spark \
  admin_user="dgxadmin" \
  db_password="SparkSecurePassword2026!" \
  api_key="nv-live-993821038"

# Read back the secret
vault kv get secret/dgx/spark
```

### 4. Create an AppRole for Machine-to-Machine Auth

```bash
# Enable approle auth
vault auth enable approle

# Create policy for Ansible
vault policy write ansible-read - <<EOF
path "secret/data/dgx/*" {
  capabilities = ["read"]
}
EOF

# Create role bound to policy
vault write auth/approle/role/ansible-runner \
  secret_id_ttl=24h \
  token_num_uses=50 \
  token_ttl=1h \
  token_max_ttl=4h \
  token_policies="ansible-read"

# Fetch Role ID and Secret ID
ROLE_ID=$(vault read -format=json auth/approle/role/ansible-runner/role-id | jq -r .data.role_id)
SECRET_ID=$(vault write -format=json -f auth/approle/role/ansible-runner/secret-id | jq -r .data.secret_id)

echo "Role ID:   $ROLE_ID"
echo "Secret ID: $SECRET_ID"
```

---

## 📄 Example: Ansible Playbook Fetching from Vault

Save this playbook to test Vault secret retrieval:

```yaml
---
- name: Demonstrate HashiCorp Vault retrieval with community.hashi_vault
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    vault_url: "http://127.0.0.1:8200"
    vault_mount: "secret"
    vault_path: "dgx/spark"
    # In production, pass role_id / secret_id via environment or Tower credentials
    vault_role_id: "{{ lookup('env', 'VAULT_ROLE_ID') | default('my-role-id', true) }}"
    vault_secret_id: "{{ lookup('env', 'VAULT_SECRET_ID') | default('my-secret-id', true) }}"

  tasks:
    - name: Fetch secret using vault_kv2_get module
      community.hashi_vault.vault_kv2_get:
        url: "{{ vault_url }}"
        engine_mount_point: "{{ vault_mount }}"
        path: "{{ vault_path }}"
        auth_method: token
        token: "myroottoken"  # Or use auth_method: approle with role_id and secret_id
      register: spark_secrets
      no_log: true  # Protect secrets from stdout logs

    - name: Use the secret safely
      ansible.builtin.debug:
        msg: "Retrieved admin user: {{ spark_secrets.data.data.admin_user }}"

    - name: Example with lookup plugin
      ansible.builtin.set_fact:
        db_pass: "{{ lookup('community.hashi_vault.hashi_vault', 'secret=secret/data/dgx/spark:data.db_password url=' ~ vault_url ~ ' token=myroottoken') }}"
      no_log: true
```

---

## 🏢 Configuring in Ansible Tower / AWX

1. **Add HashiCorp Vault Credential**:
   - Go to **Credentials** ➔ **Add**.
   - Credential Type: Select **HashiCorp Vault Secret Lookup**.
   - Set **Server URL** (e.g., `https://vault.internal.net:8200`).
   - Authentication method: Choose **AppRole** (Enter `Role ID` and `Secret ID`) or **Token**.
   - Upload CA Certificate if using internal TLS.

2. **Link to a Target Credential (Credential Linking)**:
   - Go to **Credentials** ➔ Add a **Machine** credential.
   - For **Password** or **SSH Private Key**, click the key icon (Search external credential).
   - Select your **HashiCorp Vault Secret Lookup** credential.
   - Enter Path: `secret/data/dgx/spark` and Key: `db_password` or `ssh_key`.

3. **Execution Environments (EE)**:
   - Make sure your AWX / Tower Execution Environment image includes `python3-hvac` and the `community.hashi_vault` collection.
