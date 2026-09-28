# Deep Dive: HashiCorp Vault — Complete Learning Material

A comprehensive guide covering architecture, secrets engines, authentication, policies, operational procedures, integration patterns, and production hardening for HashiCorp Vault.

---

## 📑 Table of Contents

1. [What Is HashiCorp Vault & Why You Need It](#1-what-is-hashicorp-vault--why-you-need-it)
2. [Architecture & Core Concepts](#2-architecture--core-concepts)
3. [Installation & Lab Setup](#3-installation--lab-setup)
4. [Vault CLI Essentials](#4-vault-cli-essentials)
5. [Secrets Engines Deep Dive](#5-secrets-engines-deep-dive)
6. [KV Secrets Engine (v1 vs v2)](#6-kv-secrets-engine-v1-vs-v2)
7. [Dynamic Secrets](#7-dynamic-secrets)
8. [Authentication Methods](#8-authentication-methods)
9. [Policies & ACLs](#9-policies--acls)
10. [Token Management](#10-token-management)
11. [AppRole Authentication (Machine-to-Machine)](#11-approle-authentication-machine-to-machine)
12. [Vault Agent & Auto-Auth](#12-vault-agent--auto-auth)
13. [Response Wrapping](#13-response-wrapping)
14. [Audit Logging](#14-audit-logging)
15. [Seal / Unseal & Auto-Unseal](#15-seal--unseal--auto-unseal)
16. [High Availability & Storage Backends](#16-high-availability--storage-backends)
17. [Namespaces (Enterprise)](#17-namespaces-enterprise)
18. [Integrating Vault with Ansible](#18-integrating-vault-with-ansible)
19. [Integrating Vault with Ansible Tower / AWX](#19-integrating-vault-with-ansible-tower--awx)
20. [Production Hardening Checklist](#20-production-hardening-checklist)
21. [Troubleshooting](#21-troubleshooting)
22. [Lab Exercises](#22-lab-exercises)
23. [Learning Resources](#23-learning-resources)

---

## 1. What Is HashiCorp Vault & Why You Need It

### The Problem

```text
❌ Secrets sprawl:
   - Passwords hardcoded in scripts, playbooks, and config files
   - SSH keys on every engineer's laptop
   - API tokens in environment variables or .env files
   - Database credentials in plaintext Ansible group_vars
   - No audit trail of who accessed what secret
   - No rotation — same password for years
   - No encryption at rest
```

### The Solution

HashiCorp Vault is a **centralized secrets management system** that:

| Capability | Description |
| :--- | :--- |
| **Centralized secrets** | Single source of truth for all credentials |
| **Dynamic secrets** | Generate short-lived credentials on demand (DB users, cloud keys) |
| **Encryption as a Service** | Encrypt/decrypt data without exposing encryption keys |
| **Lease & TTL** | Every secret has an expiration — automatic revocation |
| **Audit logging** | Every access is logged with who, what, when |
| **Fine-grained ACLs** | Policies control exactly who can access which secrets |
| **Multiple auth methods** | Tokens, LDAP, GitHub, Kubernetes, AppRole, AWS IAM, etc. |

### Vault vs Ansible Vault (Native)

| Feature | HashiCorp Vault | Ansible Vault (`ansible-vault`) |
| :--- | :--- | :--- |
| Type | Centralized secrets server | File-level encryption tool |
| Scope | Organization-wide | Single playbook project |
| Dynamic secrets | Yes (DB creds, cloud keys) | No |
| Access control | Fine-grained policies | Single passphrase |
| Audit trail | Full audit log | None |
| Secret rotation | Automatic with leases | Manual |
| Multi-user | Yes (RBAC, teams) | Shared passphrase |
| API | Full REST API | CLI only |

> **Summary**: `ansible-vault` encrypts files. HashiCorp Vault manages secrets at enterprise scale.

---

## 2. Architecture & Core Concepts

```mermaid
graph TD
    subgraph "Vault Server"
        API[HTTP API Layer]
        BARRIER[Encryption Barrier]
        CORE[Core - Path Routing]

        subgraph "Secrets Engines"
            KV[KV v2 - Static Secrets]
            DB[Database - Dynamic Creds]
            PKI[PKI - TLS Certificates]
            TRANSIT[Transit - Encryption]
            SSH_E[SSH - Signed Keys]
        end

        subgraph "Auth Methods"
            TOKEN[Token Auth]
            APPROLE[AppRole]
            LDAP_A[LDAP]
            K8S[Kubernetes]
            GITHUB[GitHub]
        end

        subgraph "System Backends"
            AUDIT[Audit Devices]
            POLICY[Policy Store]
            SYS[sys/ System Backend]
        end

        STORAGE[(Storage Backend)]
    end

    CLIENT[Client - CLI / SDK / Ansible] -->|HTTPS| API
    API --> BARRIER
    BARRIER --> CORE
    CORE --> KV
    CORE --> DB
    CORE --> PKI
    CORE --> TRANSIT
    CORE --> SSH_E
    CORE --> TOKEN
    CORE --> APPROLE
    CORE --> LDAP_A
    CORE --> K8S
    CORE --> GITHUB
    CORE --> AUDIT
    CORE --> POLICY
    CORE --> SYS
    BARRIER --> STORAGE
```

### Core Concepts

| Concept | Description |
| :--- | :--- |
| **Secrets Engine** | A component mounted at a path that stores, generates, or encrypts data. Like a "plugin" that handles a specific type of secret. |
| **Auth Method** | A mechanism for authenticating to Vault. Returns a token upon successful authentication. |
| **Token** | The primary authentication mechanism. Every request to Vault requires a token. All auth methods ultimately produce a token. |
| **Policy** | An HCL/JSON document that grants or denies capabilities (`read`, `write`, `list`, `delete`) on specific paths. |
| **Path** | Every operation in Vault is performed on a path (e.g., `secret/data/dgx/spark`). Paths map to secrets engines and auth methods. |
| **Lease** | A metadata structure with a TTL. When a lease expires, Vault revokes the associated secret. |
| **Seal/Unseal** | Vault starts in a "sealed" state where it cannot decrypt storage. "Unsealing" provides the master key to decrypt. |
| **Barrier** | An encryption layer that encrypts everything before it reaches storage. Uses AES-256-GCM. |
| **Storage Backend** | Where Vault persists its encrypted data (Consul, Raft integrated, PostgreSQL, file, etc.). |

### How a Secret Read Works

```mermaid
sequenceDiagram
    participant C as Client (Ansible)
    participant V as Vault API
    participant A as Auth Method
    participant P as Policy Engine
    participant S as Secrets Engine
    participant D as Audit Device

    C->>V: POST /v1/auth/approle/login (role_id + secret_id)
    V->>A: Validate AppRole credentials
    A->>V: Return token with attached policies
    V->>C: Return client token

    C->>V: GET /v1/secret/data/dgx/spark (with token)
    V->>D: Log access attempt
    V->>P: Check token's policies against path
    P->>V: Permission granted (policy allows read)
    V->>S: Retrieve secret from KV engine
    S->>V: Return secret data
    V->>D: Log successful read
    V->>C: Return secret data (JSON)
```

---

## 3. Installation & Lab Setup

### Option A: Dev Server (Fastest — In-Memory, NOT for Production)

```bash
# Install Vault CLI (macOS)
brew install hashicorp/tap/vault

# Or download binary
curl -fsSL https://releases.hashicorp.com/vault/1.17.3/vault_1.17.3_darwin_arm64.zip -o vault.zip
unzip vault.zip
sudo mv vault /usr/local/bin/

# Verify
vault --version

# Start dev server (auto-unseals, root token = "root")
vault server -dev -dev-root-token-id="root"
```

In a new terminal:
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

# Verify
vault status
```

### Option B: Docker (Recommended for Persistent Lab)

```bash
# Run Vault in dev mode with Docker
docker run -d \
  --name vault-lab \
  --cap-add=IPC_LOCK \
  -p 8200:8200 \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=root' \
  -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' \
  hashicorp/vault:latest

export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

vault status
```

### Option C: Production-like Setup (Raft Storage)

Create a config file `vault-config.hcl`:
```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = "true"  # Enable TLS in production!
}

api_addr     = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui           = true
```

```bash
# Create data directory
sudo mkdir -p /opt/vault/data
sudo chown vault:vault /opt/vault/data

# Start Vault
vault server -config=vault-config.hcl

# In another terminal: Initialize (creates unseal keys + root token)
vault operator init -key-shares=5 -key-threshold=3

# Unseal with 3 of 5 keys
vault operator unseal <KEY_1>
vault operator unseal <KEY_2>
vault operator unseal <KEY_3>

# Login with root token
vault login <ROOT_TOKEN>
```

### Vault Web UI

Access the Vault web UI at `http://127.0.0.1:8200/ui` — great for visual exploration of secrets engines, policies, and auth methods.

---

## 4. Vault CLI Essentials

### General Commands

```bash
# Check Vault status
vault status

# Login (token, userpass, etc.)
vault login root
vault login -method=userpass username=admin

# Check current token information
vault token lookup

# View server configuration
vault read sys/config/state/sanitized

# List enabled secrets engines
vault secrets list

# List enabled auth methods
vault auth list

# Read help for any path
vault path-help secret/
vault path-help auth/approle/
```

### Environment Variables

```bash
export VAULT_ADDR='http://127.0.0.1:8200'    # Vault server address
export VAULT_TOKEN='root'                      # Authentication token
export VAULT_NAMESPACE='admin/'                # Namespace (Enterprise)
export VAULT_SKIP_VERIFY='true'                # Skip TLS verification (dev only)
export VAULT_FORMAT='json'                     # Output format (json, table, yaml)
```

---

## 5. Secrets Engines Deep Dive

Secrets engines are mounted at a **path** and handle different types of secrets.

### Built-In Secrets Engines

| Engine | Path | Purpose |
| :--- | :--- | :--- |
| **KV v1** | `secret/` | Static key-value store (no versioning) |
| **KV v2** | `secret/` | Static key-value store with versioning |
| **Database** | `database/` | Dynamic database credentials (MySQL, PostgreSQL, etc.) |
| **AWS** | `aws/` | Dynamic AWS IAM credentials |
| **GCP** | `gcp/` | Dynamic GCP service account keys |
| **PKI** | `pki/` | X.509 TLS certificate generation |
| **Transit** | `transit/` | Encryption as a service (encrypt/decrypt/sign) |
| **SSH** | `ssh/` | SSH key signing and OTP |
| **TOTP** | `totp/` | Time-based one-time passwords |
| **Cubbyhole** | `cubbyhole/` | Per-token private storage (automatically deleted with token) |

### Enabling a Secrets Engine

```bash
# Enable KV v2 at a custom path
vault secrets enable -path=dgx-secrets -version=2 kv

# Enable Database engine
vault secrets enable database

# Enable Transit engine
vault secrets enable transit

# List all enabled engines
vault secrets list -detailed

# Disable (remove) an engine
vault secrets disable dgx-secrets/
```

---

## 6. KV Secrets Engine (v1 vs v2)

### KV v1 (Simple, No Versioning)

```bash
# Enable KV v1
vault secrets enable -path=kv-v1 -version=1 kv

# Write a secret
vault kv put kv-v1/app/config username="admin" password="s3cret"

# Read a secret
vault kv get kv-v1/app/config

# Delete a secret
vault kv delete kv-v1/app/config
```

### KV v2 (Versioned — Recommended)

KV v2 is the default in dev mode. It automatically versions secrets.

```bash
# Write a secret (creates version 1)
vault kv put secret/dgx/spark \
  admin_user="dgxadmin" \
  db_password="SparkPass2026!" \
  api_key="nv-live-993821038"

# Read latest version
vault kv get secret/dgx/spark

# Read specific version
vault kv get -version=1 secret/dgx/spark

# Read a specific field
vault kv get -field=db_password secret/dgx/spark

# Read as JSON
vault kv get -format=json secret/dgx/spark

# Update (creates version 2 — old version preserved)
vault kv put secret/dgx/spark \
  admin_user="dgxadmin" \
  db_password="NewPassword2026!" \
  api_key="nv-live-993821038"

# Patch a single field (KV v2 only — doesn't overwrite other fields)
vault kv patch secret/dgx/spark db_password="PatchedPassword!"

# List secrets at a path
vault kv list secret/dgx/

# View secret metadata (versions, timestamps, custom metadata)
vault kv metadata get secret/dgx/spark

# Delete latest version (soft delete — can be undeleted)
vault kv delete secret/dgx/spark

# Undelete version 2
vault kv undelete -versions=2 secret/dgx/spark

# Permanently destroy a specific version
vault kv destroy -versions=1 secret/dgx/spark

# Delete ALL versions and metadata permanently
vault kv metadata delete secret/dgx/spark
```

### KV v2 API Path Mapping

This is a critical concept that trips up beginners:

| CLI Command | Actual API Path | Notes |
| :--- | :--- | :--- |
| `vault kv put secret/dgx/spark` | `PUT /v1/secret/data/dgx/spark` | `data/` is injected |
| `vault kv get secret/dgx/spark` | `GET /v1/secret/data/dgx/spark` | `data/` is injected |
| `vault kv list secret/dgx/` | `LIST /v1/secret/metadata/dgx/` | `metadata/` is injected |
| `vault kv metadata get secret/dgx/spark` | `GET /v1/secret/metadata/dgx/spark` | `metadata/` is injected |

> ⚠️ When writing **Vault policies**, you must use the **full API path** including `data/` or `metadata/`.

### Setting Maximum Versions

```bash
# Limit to 10 versions per secret (saves storage)
vault kv metadata put -max-versions=10 secret/dgx/spark

# Set globally for the engine
vault write secret/config max_versions=10
```

---

## 7. Dynamic Secrets

Dynamic secrets are generated on demand and automatically revoked after their TTL expires.

### Example: Dynamic PostgreSQL Credentials

```bash
# 1. Enable the database engine
vault secrets enable database

# 2. Configure PostgreSQL connection
vault write database/config/my-postgres-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="readonly-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.lab.local:5432/appdb?sslmode=disable" \
  username="vault_admin" \
  password="vault_admin_pass"

# 3. Create a role (template for generated credentials)
vault write database/roles/readonly-role \
  db_name=my-postgres-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# 4. Generate dynamic credentials (new user/password each time!)
vault read database/creds/readonly-role
# Key                Value
# ---                -----
# lease_id           database/creds/readonly-role/abc123
# lease_duration     1h
# username           v-approle-readonly-abc123xyz
# password           A1b2C3d4E5-auto-generated

# 5. After 1 hour, Vault automatically REVOKES the user from PostgreSQL
```

### Example: Dynamic AWS IAM Credentials

```bash
vault secrets enable aws

vault write aws/config/root \
  access_key=AKIAIOSFODNN7EXAMPLE \
  secret_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  region=us-east-1

vault write aws/roles/s3-reader \
  credential_type=iam_user \
  policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
EOF

# Generate temporary AWS credentials
vault read aws/creds/s3-reader
```

### Lease Management

```bash
# List all active leases
vault list sys/leases/lookup/database/creds/readonly-role/

# Renew a lease (extend TTL)
vault lease renew database/creds/readonly-role/abc123

# Revoke a specific lease (immediately delete the credential)
vault lease revoke database/creds/readonly-role/abc123

# Revoke ALL leases under a prefix
vault lease revoke -prefix database/creds/readonly-role/
```

---

## 8. Authentication Methods

Every auth method validates identity and returns a **Vault token** with attached policies.

### Built-In Auth Methods

| Method | Use Case | How It Works |
| :--- | :--- | :--- |
| **Token** | Default, always enabled | Direct token presentation |
| **Userpass** | Human users (dev/lab) | Username + password |
| **AppRole** | Machines, CI/CD, Ansible | `role_id` + `secret_id` → token |
| **LDAP** | Enterprise SSO | Validates against LDAP/AD directory |
| **GitHub** | Developer teams | GitHub personal access token |
| **Kubernetes** | Pods in K8s | Service account JWT token |
| **AWS IAM** | EC2 instances, Lambda | AWS IAM identity |
| **GCP** | GCE instances, Cloud Functions | GCP service account identity |
| **TLS Certificates** | Client certificate mutual TLS | X.509 client cert |
| **OIDC** | SSO (Okta, Azure AD, Google) | OpenID Connect flow |

### Userpass Auth (Lab Setup)

```bash
# Enable userpass
vault auth enable userpass

# Create a user
vault write auth/userpass/users/alice \
  password="AliceP@ss2026" \
  policies="kv-reader"

# Login as alice
vault login -method=userpass username=alice password="AliceP@ss2026"
```

### LDAP Auth (Enterprise)

```bash
vault auth enable ldap

vault write auth/ldap/config \
  url="ldaps://ldap.example.com" \
  userdn="ou=users,dc=example,dc=com" \
  groupdn="ou=groups,dc=example,dc=com" \
  groupattr="cn" \
  userattr="uid" \
  starttls=true \
  certificate=@ldap-ca.pem

# Map LDAP group to Vault policy
vault write auth/ldap/groups/devops policies="devops-admin"
```

---

## 9. Policies & ACLs

Policies are written in **HCL** (HashiCorp Configuration Language) and define what a token can do.

### Policy Syntax

```hcl
# Allow reading static secrets for DGX
path "secret/data/dgx/*" {
  capabilities = ["read", "list"]
}

# Allow listing secret paths
path "secret/metadata/dgx/*" {
  capabilities = ["list"]
}

# Allow generating dynamic database credentials
path "database/creds/readonly-role" {
  capabilities = ["read"]
}

# Deny access to root credentials
path "secret/data/root/*" {
  capabilities = ["deny"]
}

# Allow managing own token
path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

### Available Capabilities

| Capability | HTTP Verb | Description |
| :--- | :--- | :--- |
| `create` | POST | Create new data |
| `read` | GET | Read data |
| `update` | PUT/POST | Modify existing data |
| `delete` | DELETE | Delete data |
| `list` | LIST | List keys/entries at a path |
| `deny` | — | Explicitly deny access (overrides all) |
| `sudo` | — | Required for certain `sys/` administrative operations |

### Managing Policies

```bash
# Write a policy from a file
vault policy write ansible-reader policy-ansible.hcl

# Write a policy inline
vault policy write kv-reader - <<EOF
path "secret/data/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/*" {
  capabilities = ["list"]
}
EOF

# List all policies
vault policy list

# Read a policy
vault policy read ansible-reader

# Delete a policy
vault policy delete ansible-reader

# Test a policy (check what a token can do)
vault token capabilities secret/data/dgx/spark
```

### Built-In Policies

| Policy | Description |
| :--- | :--- |
| `root` | Superuser — can do everything. Only assigned to the initial root token. |
| `default` | Automatically attached to every token. Grants basic self-management (lookup-self, renew-self). |

### Policy Design Best Practices

1. **Principle of Least Privilege** — grant only what's needed
2. **Use path prefixes** — `secret/data/team-a/*` instead of `secret/data/*`
3. **Never share the root token** — create admin policies instead
4. **Deny before allow** — `deny` always wins
5. **Separate read and write** — different policies for consumers vs producers
6. **Use templating** for multi-tenant setups:
   ```hcl
   path "secret/data/{{identity.entity.name}}/*" {
     capabilities = ["create", "read", "update", "delete", "list"]
   }
   ```

---

## 10. Token Management

### Token Hierarchy

```mermaid
graph TD
    ROOT[Root Token] --> T1[Admin Token - TTL:24h]
    T1 --> T2[Service Token A - TTL:1h]
    T1 --> T3[Service Token B - TTL:1h]
    T2 --> T4[Child Token - TTL:30m]

    style ROOT fill:#ff4444
    style T1 fill:#ff8800
    style T2 fill:#44aa44
    style T3 fill:#44aa44
    style T4 fill:#4488cc
```

When a parent token is revoked, **all child tokens are also revoked** (unless orphaned).

### Token Types

| Type | Stored in Backend? | Has Parent? | Renewable? | Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Service Token** | Yes | Yes | Yes | Standard use |
| **Batch Token** | No (lightweight) | No | No | High-performance, short-lived |
| **Orphan Token** | Yes | No | Yes | Independent lifecycle |
| **Periodic Token** | Yes | Varies | Yes (indefinitely) | Long-running services |

### Token Operations

```bash
# Create a token with specific policies and TTL
vault token create \
  -policy="ansible-reader" \
  -ttl=2h \
  -display-name="ansible-automation"

# Lookup token details
vault token lookup <TOKEN>
vault token lookup -self

# Renew a token (extend TTL)
vault token renew <TOKEN>
vault token renew -self

# Revoke a token (and all children)
vault token revoke <TOKEN>

# Revoke your own token
vault token revoke -self

# Create an orphan token (no parent)
vault token create -orphan -policy="ansible-reader" -ttl=4h

# Create a periodic token (renewable indefinitely)
vault token create -period=1h -policy="ansible-reader"
```

---

## 11. AppRole Authentication (Machine-to-Machine)

AppRole is the **recommended auth method for automation** (Ansible, CI/CD, scripts).

### How AppRole Works

```mermaid
sequenceDiagram
    participant ADMIN as Admin (Human)
    participant V as Vault Server
    participant A as Ansible (Machine)

    ADMIN->>V: 1. Create AppRole "ansible-runner" with policies
    ADMIN->>V: 2. Fetch Role ID (stable identifier)
    V->>ADMIN: Return Role ID
    ADMIN->>A: 3. Deliver Role ID (config file, env var)

    ADMIN->>V: 4. Generate Secret ID (one-time or limited-use)
    V->>ADMIN: Return Secret ID
    ADMIN->>A: 5. Deliver Secret ID (secure channel)

    A->>V: 6. POST /auth/approle/login (role_id + secret_id)
    V->>A: 7. Return client token (with attached policies, TTL)

    A->>V: 8. GET /secret/data/dgx/spark (with token)
    V->>A: 9. Return secret data
```

### Key Concepts

| Component | Description | Analogy |
| :--- | :--- | :--- |
| **Role ID** | Stable identifier for the role | Like a username |
| **Secret ID** | One-time or limited-use credential | Like a password |
| **Token** | Returned after successful login | Like a session cookie |

### Setup AppRole Step by Step

```bash
# 1. Enable AppRole auth
vault auth enable approle

# 2. Create a policy for Ansible
vault policy write ansible-policy - <<EOF
# Read static secrets
path "secret/data/dgx/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/dgx/*" {
  capabilities = ["list"]
}

# Generate dynamic DB credentials
path "database/creds/readonly-role" {
  capabilities = ["read"]
}

# Self-management
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
EOF

# 3. Create the AppRole
vault write auth/approle/role/ansible-runner \
  token_policies="ansible-policy" \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=24h \
  secret_id_num_uses=10 \
  token_num_uses=0        # 0 = unlimited uses during TTL

# 4. Fetch the Role ID (stable — doesn't change)
vault read auth/approle/role/ansible-runner/role-id
# role_id    abc12345-def6-7890-ghij-klmnopqrstuv

# 5. Generate a Secret ID (one-time credential)
vault write -f auth/approle/role/ansible-runner/secret-id
# secret_id          xyz98765-abcd-4321-efgh-ijklmnopqrst
# secret_id_ttl      24h
# secret_id_num_uses 10

# 6. Test login
vault write auth/approle/login \
  role_id="abc12345-def6-7890-ghij-klmnopqrstuv" \
  secret_id="xyz98765-abcd-4321-efgh-ijklmnopqrst"
# token               s.XXXXXXXXXXXXXXXXXXXXXXXX
# token_policies      ["ansible-policy", "default"]
# token_ttl           1h
```

### AppRole Security Best Practices

1. **Separate delivery channels** for Role ID and Secret ID (different systems/paths)
2. **Limit `secret_id_num_uses`** — use 1 for maximum security (one-time password)
3. **Bind to CIDR** — restrict which IP addresses can use the role:
   ```bash
   vault write auth/approle/role/ansible-runner \
     secret_id_bound_cidrs="10.0.0.0/24" \
     token_bound_cidrs="10.0.0.0/24"
   ```
4. **Rotate Secret IDs regularly** — automate regeneration
5. **Use short TTLs** — 1h token TTL, 24h secret ID TTL

---

## 12. Vault Agent & Auto-Auth

Vault Agent runs as a sidecar/daemon and handles:
- **Auto-authentication** (automatically logs in and renews tokens)
- **Caching** (reduces API calls to Vault)
- **Templating** (renders secrets into config files)

### Agent Configuration

```hcl
# vault-agent-config.hcl
pid_file = "/tmp/vault-agent.pid"

auto_auth {
  method "approle" {
    config = {
      role_id_file_path   = "/etc/vault/role-id"
      secret_id_file_path = "/etc/vault/secret-id"
      remove_secret_id_file_after_reading = true
    }
  }

  sink "file" {
    config = {
      path = "/tmp/vault-token"
      mode = 0600
    }
  }
}

cache {
  use_auto_auth_token = true
}

template {
  source      = "/etc/vault/templates/db-config.ctmpl"
  destination = "/opt/app/config/db.conf"
  perms       = 0600
  command     = "systemctl restart myapp"
}

vault {
  address = "http://vault.lab.local:8200"
}
```

### Template File (`db-config.ctmpl`)

```hcl
{{ with secret "secret/data/dgx/spark" }}
DB_USER={{ .Data.data.admin_user }}
DB_PASS={{ .Data.data.db_password }}
{{ end }}

{{ with secret "database/creds/readonly-role" }}
DYNAMIC_DB_USER={{ .Data.username }}
DYNAMIC_DB_PASS={{ .Data.password }}
{{ end }}
```

```bash
# Run Vault Agent
vault agent -config=vault-agent-config.hcl
```

---

## 13. Response Wrapping

Response wrapping adds an extra layer of security for secret delivery.

```bash
# Wrap a secret read — returns a single-use wrapping token instead of the secret
vault kv get -wrap-ttl=5m secret/dgx/spark
# wrapping_token:    s.WrapXXXXXXXXXXXXXXXXXXXX
# wrapping_ttl:      5m

# Unwrap (can only be done ONCE)
VAULT_TOKEN=s.WrapXXXXXXXXXXXXXXXXXXXX vault unwrap

# If anyone tries to unwrap again, it fails — tamper-evident
```

---

## 14. Audit Logging

```bash
# Enable file audit device
vault audit enable file file_path=/var/log/vault-audit.log

# Enable syslog
vault audit enable syslog

# List enabled audit devices
vault audit list

# Audit log entry (JSON, one per line)
# Every single request and response is logged:
# {
#   "type": "request",
#   "auth": { "token": "hmac-sha256:xxx", "policies": ["ansible-policy"] },
#   "request": { "path": "secret/data/dgx/spark", "operation": "read" },
#   "response": { "data": { ... } }  // Data is HMAC'd (hashed), not plaintext
# }
```

> **Key**: Audit logs HMAC (hash) all sensitive data. You can verify a specific value was accessed by computing its HMAC, but you cannot extract secrets from the audit log.

---

## 15. Seal / Unseal & Auto-Unseal

### Manual Unseal (Shamir's Secret Sharing)

When Vault starts, it is **sealed** — it has encrypted data but no key to decrypt it.

```bash
# Initialize Vault (first time only)
vault operator init -key-shares=5 -key-threshold=3
# Generates 5 unseal keys and 1 root token
# CRITICAL: Store these securely! (separate locations, different people)

# Unseal (requires 3 of 5 keys)
vault operator unseal <KEY_1>
vault operator unseal <KEY_2>
vault operator unseal <KEY_3>

# Check seal status
vault status
# Sealed: false ← Ready to accept requests
```

### Auto-Unseal (Production)

Instead of manual keys, Vault can use an external KMS to auto-unseal:

```hcl
# AWS KMS auto-unseal
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "12345678-1234-1234-1234-123456789012"
}

# GCP Cloud KMS auto-unseal
seal "gcpckms" {
  project     = "my-project"
  region      = "global"
  key_ring    = "vault-keyring"
  crypto_key  = "vault-key"
}
```

---

## 16. High Availability & Storage Backends

### Storage Backend Options

| Backend | HA Support | Notes |
| :--- | :--- | :--- |
| **Raft (Integrated)** | ✅ Built-in | Recommended. No external dependency. |
| **Consul** | ✅ | Traditional choice, requires Consul cluster |
| **PostgreSQL** | ❌ | Simple, no HA |
| **MySQL** | ❌ | Simple, no HA |
| **S3** | ❌ | Object storage (durable, not HA) |
| **File** | ❌ | Development only |

### Raft HA Cluster

```text
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Vault Node 1 │◄──►│ Vault Node 2 │◄──►│ Vault Node 3 │
│   (Leader)   │    │  (Standby)   │    │  (Standby)   │
│  Read/Write  │    │  Read Only   │    │  Read Only   │
└──────────────┘    └──────────────┘    └──────────────┘
      ▲                   ▲                   ▲
      └───────────┬───────┘                   │
                  │                           │
           Load Balancer ◄────────────────────┘
                  │
              Clients
```

---

## 17. Namespaces (Enterprise)

Namespaces provide **multi-tenancy** — each team gets an isolated Vault environment.

```bash
# Create namespaces
vault namespace create team-gpu
vault namespace create team-platform

# Write secret in a namespace
VAULT_NAMESPACE=team-gpu vault kv put secret/config api_key="gpu-key"

# Access is completely isolated — team-platform cannot see team-gpu secrets
```

---

## 18. Integrating Vault with Ansible

### Install Dependencies

```bash
pip install hvac
ansible-galaxy collection install community.hashi_vault
```

### Method 1: Lookup Plugin

```yaml
---
- name: Fetch secrets using lookup plugin
  hosts: dgx_spark
  become: true
  gather_facts: false

  vars:
    vault_addr: "http://127.0.0.1:8200"
    vault_token: "{{ lookup('env', 'VAULT_TOKEN') }}"

  tasks:
    - name: Retrieve DB password from Vault
      ansible.builtin.set_fact:
        db_password: >-
          {{ lookup('community.hashi_vault.hashi_vault',
             'secret/data/dgx/spark:data.db_password',
             url=vault_addr,
             token=vault_token) }}
      no_log: true

    - name: Use the password
      ansible.builtin.debug:
        msg: "Password retrieved successfully (hidden)"
```

### Method 2: vault_kv2_get Module

```yaml
---
- name: Fetch secrets using vault_kv2_get module
  hosts: dgx_spark
  become: true

  tasks:
    - name: Read DGX secrets from Vault
      community.hashi_vault.vault_kv2_get:
        url: "http://127.0.0.1:8200"
        engine_mount_point: secret
        path: dgx/spark
        auth_method: approle
        role_id: "{{ lookup('env', 'VAULT_ROLE_ID') }}"
        secret_id: "{{ lookup('env', 'VAULT_SECRET_ID') }}"
      register: dgx_secrets
      no_log: true
      delegate_to: localhost
      run_once: true

    - name: Deploy configuration using Vault secrets
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/app/config.conf
      vars:
        app_db_user: "{{ dgx_secrets.data.data.admin_user }}"
        app_db_pass: "{{ dgx_secrets.data.data.db_password }}"
      no_log: true
```

### Method 3: Environment Variables

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_ROLE_ID='abc12345-...'
export VAULT_SECRET_ID='xyz98765-...'

ansible-playbook -i inventory.ini deploy.yml
```

```yaml
# The collection auto-detects environment variables:
- name: Fetch with env auto-detection
  community.hashi_vault.vault_kv2_get:
    engine_mount_point: secret
    path: dgx/spark
    auth_method: approle
  register: secrets
  no_log: true
```

---

## 19. Integrating Vault with Ansible Tower / AWX

### Step-by-Step Tower Integration

1. **Create a HashiCorp Vault Credential in Tower**:
   - Navigate to **Credentials** → **Add**
   - **Credential Type**: `HashiCorp Vault Secret Lookup`
   - **Server URL**: `https://vault.internal.net:8200`
   - **Authentication Path**: `approle` (or leave default for token)
   - **Role ID**: `abc12345-...`
   - **Secret ID**: `xyz98765-...`
   - (Optional) **API Version**: `v2`
   - (Optional) **CA Certificate**: Upload if using internal TLS

2. **Create a Machine Credential Linked to Vault**:
   - Navigate to **Credentials** → **Add**
   - **Credential Type**: `Machine`
   - **Username**: `dgxadmin`
   - For **Password**, click the 🔑 icon ("Search external credential")
   - Select your **HashiCorp Vault Secret Lookup** credential
   - **Secret Path**: `secret/data/dgx/spark`
   - **Secret Key**: `db_password`

3. **Attach to Job Template**:
   - Open your Job Template → **Credentials** section
   - Add the linked Machine Credential
   - Tower will **fetch the password from Vault at launch time**

### How It Works at Runtime

```mermaid
sequenceDiagram
    participant U as User
    participant T as Tower/AWX
    participant V as HashiCorp Vault
    participant N as DGX Spark Node

    U->>T: 1. Click "Launch" on Job Template
    T->>V: 2. Authenticate (AppRole login)
    V->>T: 3. Return client token
    T->>V: 4. GET secret/data/dgx/spark
    V->>T: 5. Return secret data
    T->>T: 6. Inject password into credential (in-memory only)
    T->>N: 7. SSH with injected credentials
    N->>T: 8. Task results
    T->>T: 9. Wipe credentials from memory
    T->>U: 10. Display job results
```

> **Key**: The secret is **never stored in Tower's database** — it exists only in memory during job execution.

---

## 20. Production Hardening Checklist

- ☑️ **Enable TLS** — Never run Vault without HTTPS in production
- ☑️ **Use Raft or Consul** for HA storage backend
- ☑️ **Configure auto-unseal** via cloud KMS (AWS KMS, GCP CKMS, Azure Key Vault)
- ☑️ **Enable audit logging** — at least 2 audit devices (file + syslog/splunk)
- ☑️ **Revoke the root token** after initial setup
- ☑️ **Use AppRole for automation** — never embed root tokens
- ☑️ **Set short TTLs** — 1-4h for tokens, 1-24h for dynamic secrets
- ☑️ **Bind AppRoles to CIDRs** — restrict source IPs
- ☑️ **Limit `secret_id_num_uses`** — ideally 1 (one-time use)
- ☑️ **Rotate encryption keys** periodically (`vault operator rotate`)
- ☑️ **Monitor lease counts** — detect runaway applications
- ☑️ **Backup regularly** — `vault operator raft snapshot save backup.snap`
- ☑️ **Separate admin and operational policies** — admins should not have `read` on secrets
- ☑️ **Use response wrapping** for initial secret delivery

---

## 21. Troubleshooting

### Common Errors

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `permission denied` | Token's policy doesn't allow the operation | Check `vault token capabilities <path>` and update policy |
| `missing client token` | No `VAULT_TOKEN` set or expired | Re-authenticate: `vault login` |
| `Vault is sealed` | Vault restarted and needs unsealing | Run `vault operator unseal` with threshold keys |
| `secret not found` | Wrong path (common KV v2 mistake) | For API: use `secret/data/...` not `secret/...` |
| `role_id or secret_id not found` | AppRole misconfigured | Verify: `vault read auth/approle/role/<name>/role-id` |
| `429 Too Many Requests` | Rate limiting | Implement backoff, reduce request frequency |

### Debugging Commands

```bash
# Check Vault server health
vault status

# Check your current token's identity and policies
vault token lookup -self

# Check what capabilities your token has on a path
vault token capabilities secret/data/dgx/spark

# Read audit log for recent activity
sudo tail -20 /var/log/vault-audit.log | python3 -m json.tool

# Check active leases
vault list sys/leases/lookup/

# Check Raft cluster status (if using Raft)
vault operator raft list-peers
```

---

## 22. Lab Exercises

### Exercise 1: Dev Server Setup
Start a Vault dev server. Write 3 secrets to `secret/dgx/`. Read them back. Delete one. Verify it's gone.

### Exercise 2: KV v2 Versioning
Write a secret, update it 3 times. Read version 2 specifically. Delete version 3. Undelete version 3.

### Exercise 3: Policy Authoring
Write a policy that allows read-only access to `secret/data/dgx/*` and nothing else. Create a token with that policy. Verify it can read but not write.

### Exercise 4: AppRole Complete Flow
Set up AppRole with a policy. Fetch Role ID and Secret ID. Use `curl` to POST to `/v1/auth/approle/login`. Use the returned token to read a secret.

### Exercise 5: Ansible Integration
Write a playbook that fetches a secret from Vault using the `vault_kv2_get` module with AppRole auth. Deploy the secret as a config file using `template`. Verify `no_log: true` hides the value.

### Exercise 6: Dynamic Secrets
Enable the database secrets engine. Configure a PostgreSQL connection. Create a role. Generate dynamic credentials. Verify them against the database.

### Exercise 7: Audit Trail
Enable file audit logging. Perform several read/write operations. Examine the audit log to trace the operations.

---

## 23. Learning Resources

### Official Documentation
- **[Vault Getting Started Tutorial](https://developer.hashicorp.com/vault/tutorials/getting-started)** — Step-by-step interactive tutorial
- **[Vault Documentation](https://developer.hashicorp.com/vault/docs)** — Complete reference
- **[Vault API Reference](https://developer.hashicorp.com/vault/api-docs)** — Full REST API documentation

### Interactive Labs
- **[HashiCorp Learn Platform](https://developer.hashicorp.com/vault/tutorials)** — Free hands-on tutorials covering every topic
- **[Katacoda / Killercoda Vault Scenarios](https://killercoda.com/)** — Browser-based labs

### Video Courses
- **HashiCorp's Official YouTube Channel** — Getting started series
- **"HashiCorp Vault with ArgoCD and Kubernetes"** — Practical DevOps integration

### Books
- **"HashiCorp Vault"** by Cameron Huysmans — Practical guide covering all engines and patterns

### Certification
- **HashiCorp Certified: Vault Associate (003)** — Industry credential validating Vault knowledge
  - [Exam Study Guide](https://developer.hashicorp.com/vault/tutorials/associate-cert)
  - Covers: architecture, seal/unseal, auth methods, policies, secrets engines, tokens, leases
