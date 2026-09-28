# Complete Step-by-Step Ansible Beginner's Guide & Learning Curriculum

Welcome to Ansible! This guide takes you from zero knowledge to confident automation engineer. It walks through every step in detail with practical explanations, commands, examples, and curated learning resources.

---

## 📑 Table of Contents
1. [Core Mental Model: How Ansible Actually Works](#1-core-mental-model-how-ansible-actually-works)
2. [Step 1: Setting Up the Control Machine](#step-1-setting-up-the-control-machine)
3. [Step 2: SSH Setup & Passwordless Authentication](#step-2-ssh-setup--passwordless-authentication)
4. [Step 3: Creating and Understanding Inventories](#step-3-creating-and-understanding-inventories)
5. [Step 4: Running Ad-Hoc Commands](#step-4-running-ad-hoc-commands)
6. [Step 5: Writing and Running Your First Playbook](#step-5-writing-and-running-your-first-playbook)
7. [Step 6: Mastering Essential Playbook Concepts](#step-6-mastering-essential-playbook-concepts)
8. [Step 7: Modular Code with Roles and Collections](#step-7-modular-code-with-roles-and-collections)
9. [Step 8: Secrets with Native Ansible Vault](#step-8-secrets-with-native-ansible-vault)
10. [Curated Learning Materials, Books & Labs](#curated-learning-materials-books--labs)

---

## 1. Core Mental Model: How Ansible Actually Works

Before typing commands, understand three foundational principles:

1. **Agentless Architecture**:
   - Unlike tools like Puppet, Chef, or Datadog, Ansible requires **no agent daemon** installed on managed nodes.
   - It connects from your **Control Node** (Mac/Linux workstation) to **Managed Nodes** (servers, DGX systems, VMs) over standard **SSH**, copies small Python execution modules to `/tmp`, runs them, returns JSON results, and removes the temporary files.

2. **Idempotency**:
   - An Ansible task defines the **desired end state**, not the procedural steps.
   - If a package is already installed or a file already has the right content, Ansible reports `ok` (green) and does nothing. If changes are needed, it executes them and reports `changed` (yellow). Running a playbook 10 times in a row should produce 0 changes on runs 2 through 10.

3. **Declarative YAML**:
   - Playbooks are written in YAML (`.yml`), making automation human-readable and version-controllable.

---

## Step 1: Setting Up the Control Machine

### What is the Control Node?
The machine where you install Ansible and run commands from (e.g., your Mac). *Note: Managed nodes only need Python 3 and an SSH daemon.*

### Step-by-Step Procedure:
Always use a Python virtual environment to avoid dependency conflicts with your operating system:

```bash
# 1. Create a dedicated directory for your Ansible virtual environment
python3 -m venv ~/.ansible-env

# 2. Activate the virtual environment
source ~/.ansible-env/bin/activate

# 3. Upgrade pip
pip install --upgrade pip

# 4. Install Ansible Core and linting tools
pip install ansible ansible-lint

# 5. Verify the installation
ansible --version
```

> **Tip**: Add `source ~/.ansible-env/bin/activate` to your `~/.zshrc` or activate it whenever working on Ansible projects.

---

## Step 2: SSH Setup & Passwordless Authentication

Ansible relies on passwordless SSH key authentication for seamless automation.

### Step-by-Step Procedure:

1. **Generate an SSH key pair on your Control Node** (if you don't already have one):
   ```bash
   ssh-keygen -t ed25519 -C "ansible-admin" -f ~/.ssh/ansible_id_ed25519
   ```

2. **Copy the public key to each managed host**:
   ```bash
   ssh-copy-id -i ~/.ssh/ansible_id_ed25519.pub dgxadmin@<TARGET_HOST_IP>
   ```

3. **Verify manual SSH login without password**:
   ```bash
   ssh -i ~/.ssh/ansible_id_ed25519 dgxadmin@<TARGET_HOST_IP>
   ```

4. **Configure Passwordless Sudo on Managed Nodes (Optional but recommended)**:
   On the managed machine, run `sudo visudo` and add:
   ```text
   dgxadmin ALL=(ALL) NOPASSWD:ALL
   ```

---

## Step 3: Creating and Understanding Inventories

The inventory tells Ansible **which machines to manage**, **what groups they belong to**, and **how to reach them**.

### Step-by-Step Procedure:

Create an `inventory.ini` file:

```ini
# Individual hosts and their specific connection IP/FQDN
[dgx_spark]
spark-node-01 ansible_host=192.168.1.50
spark-node-02 ansible_host=192.168.1.51

[web_servers]
web-01 ansible_host=192.168.1.60

# Group of groups (Meta-group)
[production:children]
dgx_spark
web_servers

# Variables applicable to an entire group
[dgx_spark:vars]
ansible_user=dgxadmin
ansible_ssh_private_key_file=~/.ssh/ansible_id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

### Inspect Your Inventory:
Ansible provides built-in tools to inspect and graph your inventory:
```bash
# View all parsed hosts and variables in JSON
ansible-inventory -i inventory.ini --list

# View a clean visual graph of your groups
ansible-inventory -i inventory.ini --graph
```

---

## Step 4: Running Ad-Hoc Commands

Ad-hoc commands are one-liner Ansible commands used for quick inspection, testing, or one-off operations without writing a full playbook.

### Syntax Pattern:
```bash
ansible <target-pattern> -i <inventory> -m <module-name> -a "<module-arguments>"
```

### Essential Ad-Hoc Commands to Practice:

1. **Test Connectivity (`ping` module)**:
   *Note: This is an Ansible Python ping, not an ICMP network ping.*
   ```bash
   ansible dgx_spark -i inventory.ini -m ping
   ```

2. **Run a Shell Command (`command` module)**:
   ```bash
   ansible dgx_spark -i inventory.ini -m command -a "uptime"
   ```

3. **Check Disk Space**:
   ```bash
   ansible dgx_spark -i inventory.ini -m command -a "df -h /"
   ```

4. **Check NVIDIA GPU Status (on DGX)**:
   ```bash
   ansible dgx_spark -i inventory.ini -m command -a "nvidia-smi"
   ```

5. **Gather Host Facts (`setup` module)**:
   Ansible collects detailed system information (OS, RAM, CPU cores, IPs):
   ```bash
   ansible dgx_spark -i inventory.ini -m setup -a "filter=ansible_distribution*"
   ```

---

## Step 5: Writing and Running Your First Playbook

A **Playbook** maps a group of hosts to a series of tasks.

### Step-by-Step Walkthrough:

Create a file named `site-init.yml`:

```yaml
---
- name: Initial DGX node configuration & validation
  hosts: dgx_spark
  become: true  # Run tasks with sudo elevation

  vars:
    required_packages:
      - curl
      - htop
      - git
      - bc

  tasks:
    - name: Update apt cache (Debian/Ubuntu)
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Ensure baseline system utilities are installed
      ansible.builtin.apt:
        name: "{{ required_packages }}"
        state: present

    - name: Check GPU driver and hardware status
      ansible.builtin.command: nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
      register: gpu_info
      changed_when: false

    - name: Display detected GPU model
      ansible.builtin.debug:
        msg: "GPU Hardware: {{ gpu_info.stdout }}"
```

### Running the Playbook:

1. **Check Syntax**:
   ```bash
   ansible-playbook -i inventory.ini site-init.yml --syntax-check
   ```

2. **Run in Dry-Run / Check Mode** (Simulates changes without applying):
   ```bash
   ansible-playbook -i inventory.ini site-init.yml --check
   ```

3. **Execute the Playbook**:
   ```bash
   ansible-playbook -i inventory.ini site-init.yml
   ```

---

## Step 6: Mastering Essential Playbook Concepts

To become proficient, master these core building blocks:

### 1. Variables & Fact Gathering
```yaml
vars:
  http_port: 8080

tasks:
  - name: Print custom variable and system fact
    ansible.builtin.debug:
      msg: "Host {{ inventory_hostname }} has {{ ansible_processor_vcpus }} CPUs and listens on port {{ http_port }}"
```

### 2. Conditionals (`when`)
Execute tasks only when specific conditions are met:
```yaml
- name: Install Debian package
  ansible.builtin.apt:
    name: htop
    state: present
  when: ansible_os_family == "Debian"
```

### 3. Loops (`loop`)
Iterate over lists or dictionaries:
```yaml
- name: Create required monitoring directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - /opt/monitoring
    - /opt/monitoring/logs
    - /opt/monitoring/scripts
```

### 4. Handlers & Notifiers
Handlers run **only once at the end of the play**, and **only if a task triggered a change**:
```yaml
tasks:
  - name: Update configuration file
    ansible.builtin.template:
      src: templates/app.conf.j2
      dest: /etc/app/app.conf
    notify: Restart application service

handlers:
  - name: Restart application service
    ansible.builtin.systemd:
      name: app-service
      state: restarted
```

### 5. Jinja2 Templating (`template` module)
Dynamically generate config files from template files (`.j2`):
```jinja2
# templates/app.conf.j2
server_name = {{ inventory_hostname }}
listen_port = {{ http_port }}
max_memory = {{ (ansible_memtotal_mb * 0.8) | round | int }}MB
```

---

## Step 7: Modular Code with Roles and Collections

As your automation grows, keep playbooks clean by organizing code into **Roles**.

### Role Directory Structure
Generate a standard role template:
```bash
ansible-galaxy role init roles/gpu_monitoring
```

This creates:
```text
roles/gpu_monitoring/
├── defaults/     # Lowest priority default variables
│   └── main.yml
├── vars/         # Higher priority role variables
│   └── main.yml
├── tasks/        # Main list of tasks to execute
│   └── main.yml
├── handlers/     # Handlers (service restarts)
│   └── main.yml
├── templates/    # Jinja2 template files (.j2)
├── files/        # Static files copied to target
└── meta/         # Role metadata and dependencies
```

### Using Roles in a Playbook:
```yaml
---
- name: Deploy complete cluster stack
  hosts: dgx_spark
  become: true
  roles:
    - role: common_setup
    - role: gpu_monitoring
```

---

## Step 8: Secrets with Native Ansible Vault

Before moving to HashiCorp Vault, learn native **`ansible-vault`** to understand basic encryption:

```bash
# 1. Encrypt an existing file
ansible-vault encrypt secrets.yml

# 2. View an encrypted file
ansible-vault view secrets.yml

# 3. Edit an encrypted file
ansible-vault edit secrets.yml

# 4. Encrypt a single string to paste inside a playbook
ansible-vault encrypt_string 'MySuperPassword' --name 'db_password'

# 5. Run playbook prompting for vault password
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

---

## 📚 Curated Learning Materials, Books & Labs

### 1. The Gold Standard Book
- **"Ansible for DevOps" by Jeff Geerling**: The definitive practical guide used across the industry. Free companion code repository available on GitHub.

### 2. Free Hands-on Interactive Labs (No setup required)
- **[Killercoda Ansible Scenarios](https://killercoda.com/playgrounds/scenario/ansible)**: Interactive, browser-based Linux environments pre-loaded with Ansible.
- **[Red Hat Interactive Learning](https://www.redhat.com/en/interactive-walkthroughs/ansible)**: Guided hands-on walkthroughs covering playbooks and automation concepts.

### 3. Official Documentation (Bookmark these!)
- **[Ansible Getting Started Guide](https://docs.ansible.com/ansible/latest/getting_started/index.html)**: Clear conceptual and tutorial documentation.
- **[Ansible Built-in Module Index](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html)**: Reference for modules like `copy`, `file`, `template`, `apt`, `systemd`.
- **[Ansible Community Galaxy](https://galaxy.ansible.com/)**: Repository of community-developed roles and collections (e.g. `community.hashi_vault`, `nvidia.nvidia_driver`).

### 4. High-Quality Video Courses
- **Jeff Geerling's "Ansible 101" (YouTube)**: Comprehensive, free multi-part video series by the author of Ansible for DevOps.
- **NetworkChuck & LearnLinuxTV**: Great visual overviews for beginners covering SSH keys, inventory files, and basic modules.

### 5. Certification Pathway (For Career Growth)
- **Red Hat Certified Specialist in Ansible Automation (EX294)**: The premier industry credential validating real-world enterprise Ansible proficiency.
