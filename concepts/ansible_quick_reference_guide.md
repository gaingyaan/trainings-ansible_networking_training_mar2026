# Ansible Quick Reference Guide
### Ansible for Network Engineers — 4-Day Workshop

---

## Table of Contents

1. [DevOps & CI Concepts](#1-devops--ci-concepts)
2. [Git Essentials](#2-git-essentials)
3. [YAML Fundamentals](#3-yaml-fundamentals)
4. [Jinja2 Fundamentals](#4-jinja2-fundamentals)
5. [Linting & Code Quality](#5-linting--code-quality)
6. [Pre-commit Hooks](#6-pre-commit-hooks)
7. [Ansible Installation & Configuration](#7-ansible-installation--configuration)
8. [Collections](#8-collections)
9. [Inventory](#9-inventory)
10. [Ad-hoc Commands](#10-ad-hoc-commands)
11. [Playbook Anatomy](#11-playbook-anatomy)
12. [Variables & Facts](#12-variables--facts)
13. [Conditionals & the `when` Keyword](#13-conditionals--the-when-keyword)
14. [Loops](#14-loops)
15. [Safety Patterns](#15-safety-patterns)
16. [Role Structure](#16-role-structure)
17. [Jinja2 Templates in Roles](#17-jinja2-templates-in-roles)
18. [Handlers](#18-handlers)
19. [Error Handling — block/rescue/always](#19-error-handling--blockrescuealways)
20. [Molecule Testing Framework](#20-molecule-testing-framework)
21. [Ansible Vault](#21-ansible-vault)
22. [Network Device Modules](#22-network-device-modules)
23. [Post-change Validation](#23-post-change-validation)
24. [Dynamic Inventory — Concepts](#24-dynamic-inventory--concepts)
25. [CI Pipeline — Concepts](#25-ci-pipeline--concepts)
26. [Windows Automation via WinRM](#26-windows-automation-via-winrm)
27. [Multi-platform Reference](#27-multi-platform-reference)

---

## 1. DevOps & CI Concepts

DevOps bridges **Development** and **Operations**. For network engineers, it means moving from **CLI-by-hand** to **playbook-driven, version-controlled, auditable change management**.

**Key Principles:**
- **Automation** — replace repetitive manual work with playbooks and pipelines
- **Version Control** — every config change is tracked, reviewable, and reversible
- **Continuous Integration (CI)** — automated checks run on every proposed change before it touches a device
- **Infrastructure as Code (IaC)** — network config treated like application code

**Traditional vs. CI-Driven Operations:**

| Traditional | CI-Driven |
|---|---|
| SSH to device, type config manually | Write playbook, push to repo, pipeline runs |
| Change lives only in running config | Every change is a commit — full audit trail |
| Rollback means remembering old config | `git revert` + re-run the playbook |
| No peer review | MR/PR enforces review before deploy |
| Errors found in production | Lint and dry-run catch errors before production |

**CI Pipeline Workflow:**

```
Engineer pushes playbook change
         │
         ▼
  ┌─────────────┐
  │    lint     │  ← ansible-lint, yamllint
  └──────┬──────┘
         │ pass
         ▼
  ┌─────────────┐
  │    test     │  ← Molecule — role logic validation
  └──────┬──────┘
         │ pass
         ▼
  ┌─────────────┐
  │  dry-run    │  ← ansible-playbook --check --diff
  └──────┬──────┘
         │ reviewed
         ▼
  ┌─────────────┐
  │  approval   │  ← Merge Request / Pull Request peer review
  └──────┬──────┘
         │ approved
         ▼
  ┌─────────────┐
  │   deploy    │  ← Playbook runs against production devices
  └─────────────┘
```

**Toolchain Overview:**

| Category | Tool | Purpose |
|---|---|---|
| Version Control | Git | Track every change; enables MR/PR workflow |
| Lint & Style | `ansible-lint`, `yamllint` | Enforce consistent, error-free playbooks |
| Testing | Molecule | Validate role logic without a real device |
| Configuration Management | Ansible | Push config to many devices reliably |
| CI Runner | GitLab CI, GitHub Actions, Bitbucket Pipelines | Automate the lint → test → dry-run pipeline |
| Secrets Management | Ansible Vault | Encrypt credentials stored in the repo |

**Ansible Architecture:**

```
[ Ansible Controller (Ubuntu VM) ]
         │
         │  SSH / NETCONF / network_cli
         │
   ┌─────┴──────────────────────────────┐
   │                                    │
[ Network Devices ]        [ Ubuntu/Linux VMs ]
```

- **Control Node** — Ubuntu Controller VM. All playbooks run here.
- **Managed Nodes** — network devices, Linux VMs. Nothing installed on them.
- **Agentless** — Ansible connects, runs the task, disconnects.

---

## 2. Git Essentials

```bash
# Install and configure
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Initialise a new repo
mkdir ~/ansible-network-lab && cd ~/ansible-network-lab
git init

# Create project structure
mkdir -p inventory/group_vars inventory/host_vars roles playbooks
touch inventory/hosts.ini ansible.cfg .gitignore README.md

# First commit
git add .
git commit -m "feat: initial project skeleton"
```

**.gitignore:**

```
*.retry
*.log
.vault_pass
*.pyc
__pycache__/
.env
artifacts/
```

**Commit Message Convention:**

```
<type>: <short description>

feat     — new playbook, role, or feature
fix      — bug fix
lint     — style/lint fix only, no logic change
docs     — README or comment updates
refactor — restructuring without behaviour change
chore    — dependency updates, config changes
```

**Branching and MR/PR Workflow:**

Never commit directly to `main`. Branch → commit → push → MR → merge.

```bash
git checkout -b feature/add-ntp-role
git add .
git commit -m "feat: add eos_ntp role"
git push origin feature/add-ntp-role
# Create MR/PR in your Git platform, get it reviewed, merge
git checkout main
git merge feature/add-ntp-role
git push origin main
```

**Common Git Commands:**

```bash
git status
git log --oneline
git diff
git checkout -b <branch>
git merge <branch>
git revert <commit>
```

---

## 3. YAML Fundamentals

### Core Rules

- **Spaces only** — never tabs. 2 spaces per indent level.
- **Case sensitive** — `Name` and `name` are different keys.
- `---` marks the start of a YAML document.
- `#` is a comment.

### Scalars, Lists, and Dictionaries

```yaml
---
# Scalar (string, number, boolean)
hostname: R1
vlan_id: 10
ssh_enabled: true

# List
ntp_servers:
  - 10.0.0.100
  - 10.0.0.101

# Dictionary
mgmt_interface:
  name: GigabitEthernet0/0
  ip: 192.168.1.1
  mask: 255.255.255.0

# List of dictionaries — most common pattern in Ansible
interfaces:
  - name: GigabitEthernet0/0
    ip: 10.1.1.1
    mask: 255.255.255.0
  - name: Loopback0
    ip: 1.1.1.1
    mask: 255.255.255.255
```

### Multi-line Strings

```yaml
# Literal block (|) — preserves newlines. Use for device config blocks.
ios_config_block: |
  interface GigabitEthernet0/0
   description WAN Uplink
   ip address 10.0.0.1 255.255.255.0
   no shutdown

# Folded block (>) — newlines become spaces. Use for long descriptions.
change_description: >
  This change adds NTP servers to all routers
  as per CR-2024-041.
```

### Common YAML Mistakes

```yaml
# Tab character — YAML will reject this
hostname:	R1             # ← tab, not spaces

# Inconsistent indent
interfaces:
  - name: Gi0/0
     ip: 10.0.0.1         # ← extra space breaks structure

# No space after colon
hostname:R1               # ← parsed as string, not key-value

# Special character unquoted
description: Router: Core  # ← colon in value, must quote
description: "Router: Core"  # ✓ correct
```

**Validate YAML:**

```bash
python3 -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))" inventory/hosts.yaml && echo "YAML OK"
```

---

## 4. Jinja2 Fundamentals

### Variable Substitution

```yaml
# Variable substitution with {{ }}
- name: Show device info
  debug:
    msg: "Configuring {{ hostname }} at {{ mgmt_ip }}"
```

### Common Filters

```yaml
"{{ hostname | upper }}"                           # R1 → R1
"{{ hostname | lower }}"                           # R1 → r1
"{{ description | default('No description') }}"    # fallback if undefined
"{{ ntp_servers | join(', ') }}"                   # 10.0.0.100, 10.0.0.101
"{{ interfaces | length }}"                        # count items
"{{ intf | replace('GigabitEthernet', 'Gi') }}"    # Gi0/0
"{{ ip | ansible.netcommon.ipaddr('address') }}"   # extract IP from CIDR
"{{ ip | ansible.netcommon.ipaddr('netmask') }}"   # extract mask from CIDR
```

### Conditionals and Loops in Templates

```jinja2
{% for server in ntp_servers %}
ntp server {{ server }}
{% endfor %}

{% if interface.enabled %}
interface {{ interface.name }}
 no shutdown
{% endif %}

{# This is a comment — not rendered #}
```

---

## 5. Linting & Code Quality

### yamllint

```bash
pip3 install yamllint --break-system-packages
```

Create `.yamllint` in the project root:

```yaml
---
extends: default

rules:
  line-length:
    max: 120
    level: warning
  truthy:
    allowed-values: ['true', 'false']
    check-keys: false
  comments:
    min-spaces-from-content: 1
```

```bash
yamllint inventory/hosts.yaml
yamllint .          # check all YAML in the project
```

### ansible-lint

```bash
pip3 install ansible-lint --break-system-packages
```

Create `.ansible-lint`:

```yaml
---
warn_list:
  - experimental

skip_list:
  - yaml[line-length]

exclude_paths:
  - .git/
  - molecule/
```

```bash
ansible-lint playbooks/ntp.yaml
ansible-lint                # lint everything
ansible-lint roles/eos_ntp/ # lint a role
```

**Common ansible-lint Rules:**

| Rule | What it catches |
|---|---|
| `no-free-form` | `shell: command arg` instead of proper key/value |
| `name[missing]` | Tasks without a `name:` field |
| `yaml[truthy]` | `yes`/`no` instead of `true`/`false` |
| `no-changed-when` | `shell`/`command` tasks missing `changed_when` |
| `fqcn` | Module name not fully qualified |
| `role-name` | Role name contains invalid characters or uppercase |
| `var-naming` | Variable names use hyphens instead of underscores |

---

## 6. Pre-commit Hooks

Pre-commit hooks run automatically on every `git commit`. If lint fails, the commit is blocked.

```bash
pip3 install pre-commit --break-system-packages
```

Create `.pre-commit-config.yaml`:

```yaml
---
repos:
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint
        args: [--config-file, .yamllint]

  - repo: https://github.com/ansible/ansible-lint
    rev: v24.2.0
    hooks:
      - id: ansible-lint
        args: [--config-file, .ansible-lint]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-merge-conflict
```

```bash
pre-commit install          # install hooks into the repo
pre-commit run --all-files  # run manually against all files
pre-commit run yamllint     # run a specific hook
```

---

## 7. Ansible Installation & Configuration

```bash
sudo apt update && sudo apt install -y python3 python3-pip
pip3 install ansible --break-system-packages
ansible --version
```

**`ansible.cfg`** (place in project root — takes precedence over system-wide config):

```ini
[defaults]
inventory           = ./inventory
host_key_checking   = False
retry_files_enabled = False
stdout_callback     = yaml
interpreter_python  = auto_silent
vault_password_file = .vault_pass    # optional: set vault file path

[privilege_escalation]
become        = True
become_method = sudo
become_user   = root

[persistent_connection]
connect_timeout  = 30
command_timeout  = 30
```

**Configuration file lookup order:**
1. `ANSIBLE_CONFIG` environment variable (if set)
2. `ansible.cfg` in the **current directory** ← recommended
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

| Setting | Purpose |
|---|---|
| `inventory = ./inventory` | Use the inventory directory in this project |
| `host_key_checking = False` | Skip SSH host key prompts — safe in lab |
| `stdout_callback = yaml` | Human-readable task output |
| `become = True` | Use sudo for privilege escalation on Linux |
| `connect_timeout` | Seconds to wait when connecting |
| `command_timeout` | Seconds to wait for a command response |

---

## 8. Collections

```yaml
# requirements.yml
---
collections:
  - name: ansible.netcommon
    version: ">=5.1.0,<6.0.0"
  - name: arista.eos
    version: ">=4.0.0,<5.0.0"
  - name: cisco.ios
    version: ">=5.0.0,<6.0.0"
```

```bash
ansible-galaxy collection install -r requirements.yml
ansible-galaxy collection list
ansible-doc arista.eos.eos_command    # read module docs
ansible-doc -l arista.eos            # list all modules in a collection
```

**Why pin versions?** Unpinned installs mean a playbook that works today may break next month when a collection releases an update. Pinning ensures lab, staging, and production all run the same tested code.

---

## 9. Inventory

### Directory Structure

```
inventory/
├── hosts.ini            ← host definitions and group membership
├── group_vars/
│   ├── all.yaml         ← variables for every host
│   ├── arista_nodes.yaml  ← connection vars for Arista devices
│   ├── vault_arista.yaml  ← encrypted credentials (Vault)
│   ├── ubuntu_hosts.yaml
│   └── redhat_hosts.yaml
└── host_vars/
    ├── R1.yaml          ← variables specific to R1 only
    └── R2.yaml
```

### hosts.ini

```ini
# ── Linux Hosts ────────────────────────────────────────
[ubuntu_hosts]
ubuntu-test  ansible_host=192.168.100.20

[redhat_hosts]
redhat-vm    ansible_host=192.168.100.30

[linux_hosts:children]
ubuntu_hosts
redhat_hosts

# ── Network Devices ─────────────────────────────────────
[arista_nodes]
R1  ansible_host=172.20.20.2
R2  ansible_host=172.20.20.3

[network_devices:children]
arista_nodes
```

> Credentials and connection variables go in `group_vars/` — **never** in `hosts.ini`.

### group_vars/all.yaml

```yaml
---
ansible_python_interpreter: /usr/bin/python3
environment: lab
```

### group_vars/arista_nodes.yaml (plain-text portion)

```yaml
---
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: arista.eos.eos
ansible_user: admin
ansible_become: true
ansible_become_method: enable

ntp_servers:
  - 10.0.0.100
  - 10.0.0.101
```

### group_vars/ubuntu_hosts.yaml

```yaml
---
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/id_rsa

packages_required:
  - vim
  - curl
  - net-tools
  - tree
```

### host_vars/R1.yaml

```yaml
---
ansible_host: 172.20.20.2
router_id: 1.1.1.1
site: site-a
role: core
```

### Variable Precedence (lowest → highest)

```
group_vars/all
group_vars/<group>
host_vars/<host>        ← wins over group
play vars:
extra vars (-e)         ← wins over everything
```

### Inventory Inspection Commands

```bash
ansible-inventory --graph               # tree view
ansible-inventory --list                # full JSON with all variables
ansible-inventory --host R1             # single host variables
ansible arista_nodes --list-hosts       # list hosts in a group
```

### network_cli Connection Plugin

`network_cli` is the Ansible connection plugin for SSH sessions to network devices.

| | Linux SSH | network_cli |
|---|---|---|
| Connection plugin | `ssh` (default) | `ansible.netcommon.network_cli` |
| Connects to | Linux shell | Network device CLI |
| Remote execution | Copies Python module, runs it | Sends CLI commands, parses output |
| Requires Python on target | Yes | **No** |
| Privilege escalation | sudo | `enable` (network device enable mode) |

---

## 10. Ad-hoc Commands

```
ansible <host-pattern> -m <module> -a "<arguments>" [options]
```

```bash
# Test Linux connectivity
ansible ubuntu_hosts -m ping

# Run a shell command
ansible ubuntu_hosts -m shell -a "uptime"

# Arista show command
ansible arista_nodes -m arista.eos.eos_command -a "commands='show version'"
ansible arista_nodes -m arista.eos.eos_command -a "commands='show ip interface brief'"

# Gather Arista facts
ansible R1 -m arista.eos.eos_facts -a "gather_subset=min"
ansible R1 -m arista.eos.eos_facts -a "gather_subset=all"

# Debug a variable
ansible ubuntu_hosts -m debug -a "msg={{ packages_required }}"
```

**Useful Flags:**

```bash
-v / -vv / -vvv    # increase verbosity for debugging
--check            # dry run — no changes made
--diff             # show what would change
--limit R1         # run against one host only
-e "key=value"     # pass extra variable
```

---

## 11. Playbook Anatomy

```yaml
---
# ── Play header ──────────────────────────────────────────────────
- name: Enforce baseline packages on Linux hosts    # human-readable name
  hosts: ubuntu_hosts                               # which inventory group
  gather_facts: true                                # collect system info first
  become: true                                      # use sudo

  # ── Play-level variables ────────────────────────────────────────
  vars:
    custom_package: htop

  # ── Tasks ───────────────────────────────────────────────────────
  tasks:

    - name: Update apt cache                        # every task needs a name
      ansible.builtin.apt:                          # fully qualified module name (FQCN)
        update_cache: true
        cache_valid_time: 3600

    - name: Install required packages
      ansible.builtin.apt:
        name: "{{ packages_required }}"             # variable from group_vars
        state: present

    - name: Check disk usage
      ansible.builtin.shell:
        cmd: df -h /
      register: disk_usage                          # capture output
      changed_when: false                           # read-only — never counts as a change
```

### Key Playbook Fields

| Field | Where | Purpose |
|---|---|---|
| `name` | play, task | Human-readable label |
| `hosts` | play | Which inventory group to target |
| `gather_facts` | play | Auto-collect system info before tasks run |
| `become` | play or task | Privilege escalation — sudo on Linux |
| `vars` | play | Variables scoped to this play only |
| `tasks` | play | Ordered list of tasks |
| `register` | task | Save task output into a named variable |
| `when` | task | Conditional — run only if expression is true |
| `loop` | task | Repeat task for each item in a list |
| `changed_when` | task | Define what counts as a change |
| `ignore_errors` | task | Continue even if this task fails |
| `notify` | task | Trigger a handler if this task reports changed |

### Playbook Best Practices

```yaml
# ALWAYS use Fully Qualified Collection Names (FQCN)
ansible.builtin.apt:      # ✓ correct
apt:                      # ✗ avoid — ambiguous

# ALWAYS name every task
- name: Install vim editor        # ✓ correct
  ansible.builtin.apt:
    name: vim

# AVOID free-form module syntax
- name: Check uptime                  # ✓ correct
  ansible.builtin.shell:
    cmd: uptime
# NOT: ansible.builtin.shell: uptime  # ✗ avoid

# Use true/false not yes/no
become: true       # ✓
become: yes        # ✗

# Add changed_when to shell/command tasks
- name: Get hostname
  ansible.builtin.shell:
    cmd: hostname
  changed_when: false   # this task never makes a change
```

**Syntax check before every run:**

```bash
ansible-playbook playbooks/baseline.yaml --syntax-check
```

---

## 12. Variables & Facts

### Variable Types and Where to Define Them

| Source | When to use |
|---|---|
| `group_vars/<group>.yaml` | Shared config for a group — packages, settings |
| `host_vars/<host>.yaml` | Per-host overrides — hostname, role, specific values |
| Play `vars:` block | Variables for one play only |
| `register` | Output of a task captured for use in the same play |
| Extra vars (`-e`) | Runtime overrides — highest priority |

### Facts — Discovered at Runtime

When `gather_facts: true`, Ansible runs the `setup` module and populates variables automatically.

```bash
ansible ubuntu-test -m setup
ansible ubuntu-test -m setup -a "filter=ansible_distribution*"
```

**Common Linux Facts:**

```yaml
ansible_hostname              # system hostname
ansible_os_family             # Debian (Ubuntu) or RedHat
ansible_distribution          # Ubuntu / RedHat / Rocky
ansible_distribution_version  # 22.04 / 8.7
ansible_default_ipv4.address  # primary IP address
ansible_memtotal_mb           # total RAM in MB
ansible_processor_vcpus       # number of vCPUs
ansible_python_version        # Python version on managed node
ansible_facts.packages        # installed packages (after package_facts)
```

**Common Arista EOS Facts:**

```yaml
ansible_net_version      # EOS software version
ansible_net_model        # hardware model
ansible_net_serialnum    # serial number
ansible_net_hostname     # configured hostname
ansible_net_interfaces   # dict of all interfaces + state
ansible_net_neighbors    # CDP/LLDP neighbours
ansible_net_config       # running config as a string
```

### The `register` Keyword

```yaml
tasks:
  - name: Check if a file exists
    ansible.builtin.stat:
      path: /etc/motd
    register: motd_stat

  - name: Show file info (always debug full structure first)
    debug:
      var: motd_stat

  - name: Show just whether file exists
    debug:
      msg: "MOTD file exists: {{ motd_stat.stat.exists }}"

  - name: Conditional — show size only if file exists
    debug:
      msg: "MOTD size: {{ motd_stat.stat.size }} bytes"
    when: motd_stat.stat.exists
```

**Structure of common registered results:**

```yaml
# From ansible.builtin.shell
result:
  stdout: "output as a string"
  stdout_lines: ["line1", "line2"]
  stderr: ""
  rc: 0         # return code — 0 = success
  failed: false
  changed: true

# From ansible.builtin.stat
result:
  stat:
    exists: true
    size: 1024
    mode: "0644"
    owner: root
```

### set_fact — Deriving Variables at Runtime

```yaml
- name: Set package manager fact
  set_fact:
    pkg_manager: "{{ 'apt' if ansible_os_family == 'Debian' else 'dnf' }}"

- name: Calculate disk space in GB
  ansible.builtin.shell:
    cmd: df / --output=avail | tail -1
  register: disk_avail
  changed_when: false

- name: Set disk space fact in GB
  set_fact:
    disk_available_gb: "{{ (disk_avail.stdout | int / 1024 / 1024) | round(1) }}"
```

---

## 13. Conditionals & the `when` Keyword

`when` makes a task conditional. Uses Jinja2 syntax **without** `{{ }}`:

```yaml
# OS family check — most common pattern for cross-platform playbooks
- name: Install packages — Ubuntu
  ansible.builtin.apt:
    name: "{{ packages_required }}"
    state: present
  when: ansible_os_family == "Debian"

- name: Install packages — Red Hat
  ansible.builtin.dnf:
    name: "{{ packages_required }}"
    state: present
  when: ansible_os_family == "RedHat"
```

**Common `when` Patterns:**

```yaml
when: ansible_os_family == "Debian"           # equality
when: "'Ubuntu' in ansible_distribution"      # string contains
when: custom_package is defined               # variable is defined
when: motd_stat.stat.exists                   # stat result
when: not motd_stat.stat.exists               # negate
when: command_result.rc == 0                  # return code check
when: command_result.failed == false          # failed check

# AND — all must be true (list format)
when:
  - ansible_os_family == "Debian"
  - ansible_distribution_version == "22.04"

# OR
when: >
  ansible_os_family == "Debian" or
  ansible_os_family == "RedHat"
```

---

## 14. Loops

### Simple List Loop

```yaml
- name: Install packages one by one
  ansible.builtin.package:             # cross-platform
    name: "{{ item }}"
    state: present
  loop: "{{ packages_required }}"
  loop_control:
    label: "{{ item }}"               # cleaner output
```

### Loop over a List of Dictionaries

```yaml
vars:
  lab_users:
    - name: alice
      shell: /bin/bash
      groups: sudo
    - name: bob
      shell: /bin/bash
      groups: users

tasks:
  - name: Create lab users
    ansible.builtin.user:
      name: "{{ item.name }}"
      shell: "{{ item.shell }}"
      groups: "{{ item.groups }}"
      state: present
    loop: "{{ lab_users }}"
    loop_control:
      label: "{{ item.name }}"
```

### Registering Results from a Loop

When `register` is used with a loop, results are in a `results` list.

```yaml
- name: Check if config files exist
  ansible.builtin.stat:
    path: "{{ item }}"
  loop:
    - /etc/hosts
    - /etc/motd
    - /etc/resolv.conf
  register: file_checks
  loop_control:
    label: "{{ item }}"

- name: Show which files exist
  debug:
    msg: "{{ item.item }}: {{ 'exists' if item.stat.exists else 'MISSING' }}"
  loop: "{{ file_checks.results }}"
  loop_control:
    label: "{{ item.item }}"
```

### loop_control Options

```yaml
loop_control:
  label: "{{ item.name }}"    # what to show in output instead of full item
  pause: 1                    # seconds between iterations
  index_var: loop_idx         # current loop index as a variable
```

### Cross-Platform Package Management

```yaml
# Option A — Separate tasks with when (explicit, allows per-platform options)
- name: Install packages — Ubuntu
  ansible.builtin.apt:
    name: "{{ packages_required }}"
    state: present
    update_cache: true
  when: ansible_os_family == "Debian"

- name: Install packages — Red Hat
  ansible.builtin.dnf:
    name: "{{ packages_required }}"
    state: present
  when: ansible_os_family == "RedHat"

# Option B — Single task using package module (clean for simple cases)
- name: Install packages — any Linux
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop: "{{ packages_required }}"
```

---

## 15. Safety Patterns

### Recommended Execution Workflow

```bash
# 1. Syntax check
ansible-playbook playbooks/site_baseline.yaml --syntax-check

# 2. Lint
ansible-lint playbooks/site_baseline.yaml

# 3. Dry run — one host
ansible-playbook playbooks/site_baseline.yaml --check --diff --limit R1

# 4. Dry run — all hosts
ansible-playbook playbooks/site_baseline.yaml --check --diff

# 5. Apply
ansible-playbook playbooks/site_baseline.yaml --diff

# 6. Verify
ansible-playbook playbooks/validate_baseline.yaml
```

### check_mode (`--check`)

Simulates the playbook — no changes are made. Control per task:

```yaml
- name: Always run this even in --check (read-only)
  ansible.builtin.shell:
    cmd: df -h
  check_mode: false
  changed_when: false

- name: Only runs during --check (informational)
  debug:
    msg: "Dry run — would install packages on {{ inventory_hostname }}"
  when: ansible_check_mode
```

### diff Mode (`--diff`)

Shows before/after content for file-based tasks (`copy`, `template`, `lineinfile`):

```diff
TASK [Deploy MOTD file]
--- before: /etc/motd
+++ after:
@@ @@
-Old message
+Authorised access only.
```

### assert Module

Validates expected state — fails the play with a clear message if conditions are not met.

```yaml
- name: Assert required packages are present
  assert:
    that:
      - item in ansible_facts.packages
    fail_msg: "Package {{ item }} missing on {{ inventory_hostname }}"
    success_msg: "Package {{ item }} confirmed on {{ inventory_hostname }}"
  loop: "{{ packages_required }}"
  loop_control:
    label: "{{ item }}"

- name: Assert Python 3 is available
  assert:
    that:
      - ansible_python_version is defined
      - ansible_python_version.startswith('3')
    fail_msg: "Python 3 required but found: {{ ansible_python_version }}"
    success_msg: "Python 3 confirmed: {{ ansible_python_version }}"
```

---

## 16. Role Structure

A **role** packages tasks, variables, templates, and handlers into a reusable, standardised directory structure.

```
roles/
└── eos_ntp/
    ├── tasks/
    │   └── main.yaml        ← entry point — task list (required)
    ├── templates/
    │   └── ntp.j2           ← Jinja2 template for device config
    ├── defaults/
    │   └── main.yaml        ← default variable values — lowest precedence
    ├── vars/
    │   └── main.yaml        ← internal constants — higher precedence
    ├── handlers/
    │   └── main.yaml        ← tasks triggered by notify
    ├── meta/
    │   └── main.yaml        ← role metadata
    └── molecule/
        └── default/         ← Molecule test scenario
```

Only `tasks/main.yaml` is required. Everything else is created as needed.

```bash
# Create skeleton
ansible-galaxy role init roles/eos_ntp
```

### Calling a Role from a Playbook

```yaml
---
- name: Apply NTP role to all network devices
  hosts: arista_nodes
  gather_facts: false

  roles:
    - eos_ntp
```

Or with inline role variables:

```yaml
  roles:
    - role: eos_ntp
      vars:
        ntp_servers:
          - 10.0.0.100
```

Or using `include_role` in a task block (preferred for `block/rescue`):

```yaml
  tasks:
    - name: Apply NTP role
      ansible.builtin.include_role:
        name: eos_ntp
```

### Role Defaults vs Vars

```yaml
# roles/eos_ntp/defaults/main.yaml ← operators can override these
---
ntp_servers:
  - 10.0.0.100
  - 10.0.0.101
ntp_source_interface: Management0
ntp_auth_enabled: false
```

```yaml
# roles/eos_ntp/vars/main.yaml ← internal constants, not intended to be overridden
---
ntp_stratum: 4
```

**Rule:** values operators should customise → `defaults/`. Internal implementation constants → `vars/`.

### meta/main.yaml

```yaml
---
galaxy_info:
  author: "your name"
  description: "Enforce NTP configuration on Arista EOS devices"
  license: MIT
  min_ansible_version: "2.14"
dependencies: []
```

---

## 17. Jinja2 Templates in Roles

Templates (`.j2` files) live in `roles/<name>/templates/`. They render device configuration from variable data at runtime.

### NTP Template — Arista EOS

```jinja2
{# roles/eos_ntp/templates/ntp.j2 #}
{% for server in ntp_servers %}
ntp server {{ server }}
{% endfor %}

{% if ntp_source_interface is defined %}
ntp local-interface {{ ntp_source_interface }}
{% endif %}

{% if ntp_auth_enabled %}
ntp authenticate
{% endif %}
! NTP config rendered for {{ inventory_hostname | upper }}
```

### Banner Template — Arista EOS

```jinja2
banner motd
{{ banner_motd }}
EOF
```

### SNMP Template — Arista EOS

```jinja2
snmp-server community {{ snmp_community_ro }} ro
snmp-server location {{ snmp_location }}
snmp-server contact {{ snmp_contact }}
{% if snmp_trap_host is defined %}
snmp-server host {{ snmp_trap_host }} version 2c {{ snmp_community_ro }}
{% endif %}
```

### Using a Template in a Task

```yaml
- name: Render and apply NTP configuration
  arista.eos.eos_config:
    src: "{{ role_path }}/templates/ntp.j2"
    save_when: changed
```

---

## 18. Handlers

A handler runs only when **notified** — and only **once** at the end of the play regardless of how many tasks notify it.

```yaml
# roles/eos_ntp/handlers/main.yaml
---
- name: Save EOS config
  arista.eos.eos_command:
    commands:
      - write memory
```

```yaml
# In tasks/main.yaml
- name: Apply NTP config
  arista.eos.eos_config:
    src: "{{ role_path }}/templates/ntp.j2"
  notify: Save EOS config    # triggers handler only if this task reports changed
```

---

## 19. Error Handling — block/rescue/always

```yaml
tasks:

  - name: Apply and validate config with rollback
    block:

      - name: Take pre-change backup
        arista.eos.eos_config:
          backup: true
        register: backup_result

      - name: Apply NTP role
        ansible.builtin.include_role:
          name: eos_ntp

      - name: Apply banner role
        ansible.builtin.include_role:
          name: eos_banner

      - name: Apply SNMP role
        ansible.builtin.include_role:
          name: eos_snmp

    rescue:

      - name: Log failure
        ansible.builtin.debug:
          msg: >
            FAILURE on {{ inventory_hostname }}: {{ ansible_failed_result.msg | default('unknown error') }}

      - name: Restore pre-change config
        arista.eos.eos_config:
          src: "{{ backup_result.backup_path }}"
          save_when: always
        when: backup_result.backup_path is defined

    always:

      - name: Capture final device state as artifact
        arista.eos.eos_command:
          commands:
            - show running-config
        register: final_state

      - name: Save artifact
        ansible.builtin.copy:
          content: "{{ final_state.stdout[0] }}"
          dest: "./artifacts/{{ inventory_hostname }}_final_{{ ansible_date_time.date }}.txt"
        delegate_to: localhost
```

| Section | When it runs |
|---|---|
| `block` | Always — the main task sequence |
| `rescue` | Only if a task in `block` fails |
| `always` | Always — regardless of block or rescue outcome |

---

## 20. Molecule Testing Framework

Molecule is a testing framework for Ansible roles. It provides a structured workflow to validate that a role is correct before it touches a real device.

```
lint → syntax → converge → verify → (destroy)
```

### Install

```bash
pip3 install molecule --break-system-packages
molecule --version
```

### Initialise a Scenario

```bash
cd roles/eos_ntp
molecule init scenario --driver-name delegated
```

Creates:

```
roles/eos_ntp/
└── molecule/
    └── default/
        ├── molecule.yml    ← scenario config
        ├── converge.yml    ← playbook Molecule runs to apply role
        └── verify.yml      ← playbook Molecule runs to assert state
```

### molecule.yml (Delegated Driver)

```yaml
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml

driver:
  name: delegated
  options:
    managed: false

platforms:
  - name: R1

provisioner:
  name: ansible
  inventory:
    hosts:
      all:
        hosts:
          R1:
            ansible_host: 172.20.20.2
            ansible_connection: ansible.netcommon.network_cli
            ansible_network_os: arista.eos.eos
            ansible_user: admin
            ansible_password: admin
            ansible_become: true
            ansible_become_method: enable

verifier:
  name: ansible
```

### converge.yml

```yaml
---
- name: Converge — apply eos_ntp role
  hosts: all
  gather_facts: false

  vars:
    ntp_servers:
      - 10.0.0.100
      - 10.0.0.101
    ntp_source_interface: Management0

  roles:
    - role: ../../..
```

### verify.yml

```yaml
---
- name: Verify — confirm NTP config is present
  hosts: all
  gather_facts: false

  tasks:

    - name: Check NTP configuration
      arista.eos.eos_command:
        commands:
          - show running-config | include ntp
      register: ntp_output

    - name: Assert NTP server lines present
      assert:
        that:
          - "'ntp server 10.0.0.100' in ntp_output.stdout[0]"
          - "'ntp server 10.0.0.101' in ntp_output.stdout[0]"
        fail_msg: "NTP config not applied on {{ inventory_hostname }}"
        success_msg: "NTP config verified on {{ inventory_hostname }}"
```

### Running Molecule

```bash
cd roles/eos_ntp

molecule lint       # yamllint + ansible-lint on the role
molecule syntax     # ansible-playbook --syntax-check
molecule converge   # runs the role against the test instance
molecule verify     # runs assertions to confirm expected state
molecule test       # full sequence
```

The delegated driver skips create/destroy — assumes the host is already running. Points Molecule at existing lab devices.

---

## 21. Ansible Vault

Ansible Vault encrypts files or individual variables using AES-256. Encrypted content is safe to commit to Git.

```
Plain-text credentials file
        │
        │  ansible-vault encrypt
        ▼
Encrypted file (safe to commit to Git)
        │
        │  ansible-playbook --vault-password-file .vault_pass
        ▼
Ansible decrypts in memory → runs playbook
```

### Vault Commands

```bash
# Create a new encrypted file
ansible-vault create inventory/group_vars/vault_arista.yaml

# Encrypt an existing file
ansible-vault encrypt inventory/group_vars/vault_arista.yaml

# View encrypted file (decrypts in memory)
ansible-vault view inventory/group_vars/vault_arista.yaml

# Edit encrypted file
ansible-vault edit inventory/group_vars/vault_arista.yaml

# Decrypt back to plain text (use with caution)
ansible-vault decrypt inventory/group_vars/vault_arista.yaml

# Change the vault password
ansible-vault rekey inventory/group_vars/vault_arista.yaml

# Encrypt a single inline string variable
ansible-vault encrypt_string 'admin' --name 'ansible_password'
```

### The Split File Pattern (Recommended)

Split `group_vars` into two files — one plain-text, one encrypted.

**`inventory/group_vars/arista_nodes.yaml`** (plain text — safe to commit):

```yaml
---
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: arista.eos.eos
ansible_user: admin
ansible_become: true
ansible_become_method: enable
```

**`inventory/group_vars/vault_arista.yaml`** (will be encrypted):

```yaml
---
ansible_password: admin
ansible_become_password: ""
```

Ansible automatically loads both files for the group. Variables from the vault file behave identically to variables in the plain-text file.

### Running Playbooks with Vault

```bash
# Prompt for vault password at runtime (interactive)
ansible-playbook playbooks/site_baseline.yaml --ask-vault-pass

# Use a vault password file (non-interactive — for scripts and CI)
echo "trainingvault" > .vault_pass
chmod 600 .vault_pass
ansible-playbook playbooks/site_baseline.yaml --vault-password-file .vault_pass

# Set in ansible.cfg to make vault transparent
# vault_password_file = .vault_pass
```

### Vault Best Practices

```bash
# 1. Always add .vault_pass to .gitignore before creating it
echo ".vault_pass" >> .gitignore

# 2. Verify no plain-text credentials are staged
git diff --staged | grep -i "ansible_password"  # should return nothing

# 3. Rekey periodically or after staff changes
ansible-vault rekey inventory/group_vars/vault_arista.yaml

# 4. Separate vault files per environment
# inventory/group_vars/vault_arista_prod.yaml
# inventory/group_vars/vault_arista_lab.yaml

# 5. Verify vault is working
ansible-playbook playbooks/site_baseline.yaml --check
```

---

## 22. Network Device Modules

### Arista EOS — Key Modules

```bash
# Discover available modules
ansible-doc -l arista.eos

# Common modules:
# arista.eos.eos_facts       — gather device facts
# arista.eos.eos_config      — push config lines or a template
# arista.eos.eos_command     — run show commands, capture output
# arista.eos.eos_interfaces  — manage interface state (resource module)
# arista.eos.eos_vlans       — manage VLANs (resource module)
# arista.eos.eos_l3_interfaces — manage L3 interfaces
# arista.eos.eos_ntp_global  — manage NTP (resource module)
# arista.eos.eos_banner      — manage banner
# arista.eos.eos_snmp_server — manage SNMP (resource module)
```

### Resource Modules vs Free-form Config

| Approach | Module | When to use |
|---|---|---|
| **Resource modules** | `eos_vlans`, `eos_interfaces`, `eos_ntp_global` | Structured, idempotent, declarative — preferred |
| **Free-form config push** | `eos_config` with `src:` or `lines:` | When no resource module exists, or for complex configs |

**Resource module `state` parameter:**
- `merged` — add/update, leave others alone
- `replaced` — replace the resource entirely
- `deleted` — remove the named resource
- `gathered` — read current state into `register`
- `rendered` — generate config without connecting (for review)

### eos_config Parameters

```yaml
arista.eos.eos_config:    # or cisco.ios.ios_config
  src: "{{ role_path }}/templates/x.j2"  # Jinja2 template
  lines: ["config line"]                 # inline config lines
  parents: "interface Eth1"              # parent context for lines
  save_when: changed                     # write mem only if changed
  backup: true                           # save running config before change
  replace: line                          # default — add missing lines
  replace: block                         # replace entire parent block
```

### eos_command

```yaml
- name: Run show commands
  arista.eos.eos_command:
    commands:
      - show running-config | include ntp
      - show running-config | section banner
      - show ip interface brief
      - show version
  register: cmd_output

- name: Show output
  debug:
    msg: "{{ cmd_output.stdout[0] }}"   # stdout is a list, one entry per command
```

### eos_vlans — Resource Module Example

```yaml
- name: Ensure VLANs are configured
  arista.eos.eos_vlans:
    config:
      - vlan_id: 10
        name: Mgmt
        state: active
      - vlan_id: 20
        name: Users
        state: active
    state: merged
```

### eos_facts — Useful Variables

| Variable | Content |
|---|---|
| `ansible_net_version` | EOS software version |
| `ansible_net_model` | Hardware model |
| `ansible_net_serialnum` | Serial number |
| `ansible_net_hostname` | Configured hostname |
| `ansible_net_interfaces` | Dict of all interfaces + state |
| `ansible_net_neighbors` | CDP/LLDP neighbours |
| `ansible_net_config` | Running config as a string |

```yaml
- name: Gather EOS facts
  arista.eos.eos_facts:
    gather_subset:
      - min        # identity only (fast)
      # - interfaces
      # - all

- name: Display hostname and version
  ansible.builtin.debug:
    msg: >
      Host: {{ inventory_hostname }}
      EOS version: {{ ansible_net_version }}
      Model: {{ ansible_net_model }}

- name: List all interfaces
  ansible.builtin.debug:
    msg: "Interface: {{ item.key }}"
  loop: "{{ ansible_net_interfaces | dict2items }}"
  loop_control:
    label: "{{ item.key }}"
```

### Navigating Ansible Documentation

```bash
# Terminal docs
ansible-doc arista.eos.eos_command
ansible-doc arista.eos.eos_config -s    # examples only

# Online
# https://docs.ansible.com/ansible/latest/collections/arista/eos/
# https://docs.ansible.com/ansible/latest/collections/ansible/builtin/
```

**How to read a module page:**
1. **Synopsis** — one-line description
2. **Parameters** — every option, required vs optional, type, default
3. **Notes** — platform-specific gotchas
4. **Examples** — copy-paste starting points
5. **Return Values** — what `register` gives you

---

## 23. Post-change Validation

Applying config is only half the task — validate that the change produced the expected outcome.

**Validation Pattern:**
1. Run `eos_command` with show commands to read actual device state
2. `register` the output
3. `assert` specific strings are present in the output
4. Save output as a timestamped **artifact** for audit trail

```yaml
---
- name: Post-change validation — Arista nodes
  hosts: arista_nodes
  gather_facts: false

  tasks:

    - name: Gather post-change device state
      arista.eos.eos_command:
        commands:
          - show running-config | include ntp
          - show running-config | section banner
          - show running-config | include snmp
          - show ntp status
      register: validation_output

    - name: Assert NTP servers are configured
      ansible.builtin.assert:
        that:
          - "'ntp server 10.0.0.100' in validation_output.stdout[0]"
          - "'ntp server 10.0.0.101' in validation_output.stdout[0]"
        fail_msg: "NTP server config missing on {{ inventory_hostname }}"
        success_msg: "NTP servers confirmed on {{ inventory_hostname }}"

    - name: Assert banner is set
      ansible.builtin.assert:
        that:
          - "'Authorised' in validation_output.stdout[1]"
        fail_msg: "Banner not set on {{ inventory_hostname }}"
        success_msg: "Banner confirmed on {{ inventory_hostname }}"

    - name: Assert SNMP community is configured
      ansible.builtin.assert:
        that:
          - "'snmp-server community' in validation_output.stdout[2]"
        fail_msg: "SNMP config missing on {{ inventory_hostname }}"
        success_msg: "SNMP community confirmed on {{ inventory_hostname }}"

    - name: Save validation output as artifact
      ansible.builtin.copy:
        content: |
          Validation run: {{ ansible_date_time.iso8601 }}
          Host: {{ inventory_hostname }}
          ---
          NTP config:
          {{ validation_output.stdout[0] }}
          ---
          Banner config:
          {{ validation_output.stdout[1] }}
          ---
          SNMP config:
          {{ validation_output.stdout[2] }}
          ---
          NTP status:
          {{ validation_output.stdout[3] }}
        dest: "./artifacts/{{ inventory_hostname }}_validation_{{ ansible_date_time.date }}.txt"
      delegate_to: localhost
```

```bash
# Create artifacts directory — exclude from Git
mkdir -p artifacts
echo "artifacts/" >> .gitignore
```

---

## 24. Dynamic Inventory — Concepts

Static `hosts.ini` files do not scale at large numbers of devices. **Dynamic inventory** generates the host list automatically from an external source of truth.

```
ansible-playbook site_baseline.yaml
        │
        │  Ansible calls inventory plugin
        ▼
  ┌─────────────┐
  │   NetBox    │  ← queries devices, groups by site/role/platform
  └──────┬──────┘
        │  returns host list + variables
        ▼
Ansible builds inventory in memory
runs playbook against real-time host list
```

**Common Sources of Truth:**

| System | What it stores |
|---|---|
| **NetBox** | Devices, interfaces, IPs, VLANs, sites, roles — the network CMDB |
| **Nautobot** | Same as NetBox (fork), adds workflow automation |
| **Infoblox** | IPAM + DNS |
| **ServiceNow CMDB** | Enterprise asset and configuration management |

**NetBox Dynamic Inventory Config (reference):**

```yaml
# inventory/netbox.yaml  ← replaces hosts.ini
---
plugin: netbox.netbox.nb_inventory
api_endpoint: http://netbox.lab.local
token: "{{ lookup('env', 'NETBOX_TOKEN') }}"

group_by:
  - device_roles
  - sites
  - platforms

compose:
  ansible_host: primary_ip4.address | ansible.netcommon.ipaddr('address')
  ansible_network_os: "arista.eos.eos"
```

```bash
ansible-galaxy collection install netbox.netbox
ansible-inventory -i inventory/netbox.yaml --graph
ansible-playbook -i inventory/netbox.yaml playbooks/site_baseline.yaml
```

**Static vs. Dynamic:**

| | Static (`hosts.ini`) | Dynamic (NetBox/Nautobot) |
|---|---|---|
| Setup | Zero — edit a text file | Requires a running NetBox instance |
| Accuracy | Manual — can drift from reality | Live — always reflects current state |
| Scalability | Painful at >50 devices | Scales to thousands |
| Good for | Labs, small environments | Production, large fleets |

---

## 25. CI Pipeline — Concepts

A CI pipeline (e.g., Bitbucket Pipelines, GitLab CI, GitHub Actions) automates the quality gates that run on every push or pull request.

```
git push
    │
    ▼
[Pipeline starts]
    │
    ├─ Step 1: lint          → yamllint + ansible-lint
    ├─ Step 2: syntax check  → ansible-playbook --syntax-check
    ├─ Step 3: dry-run       → --check --diff (manual trigger)
    │
    ▼
[Pass → engineer applies manually]
[Fail → engineer is notified before any device is touched]
```

**Conceptual Bitbucket Pipelines file:**

```yaml
# .bitbucket-pipelines.yml
image: python:3.11

pipelines:
  default:
    - step:
        name: Lint
        script:
          - pip install ansible ansible-lint yamllint
          - ansible-galaxy collection install -r requirements.yml
          - yamllint .
          - ansible-lint
    - step:
        name: Syntax Check
        script:
          - pip install ansible
          - ansible-galaxy collection install -r requirements.yml
          - ansible-playbook playbooks/site_baseline.yaml --syntax-check
    - step:
        name: Dry Run
        trigger: manual            # human must approve before this runs
        script:
          - pip install ansible
          - ansible-galaxy collection install -r requirements.yml
          - echo "$VAULT_PASSWORD" > .vault_pass   # injected from Bitbucket Repository Variable
          - chmod 600 .vault_pass
          - ansible-playbook playbooks/site_baseline.yaml --check --diff
          - rm -f .vault_pass
```

**Vault in CI Pipelines:**

In a CI pipeline, the vault password is stored as a **CI secret/variable** — never in the repo. The pipeline injects it at runtime, uses it, then immediately discards it.

```
CI secret store holds: VAULT_PASSWORD = "trainingvault"
        │
        │  Pipeline injects it at runtime
        ▼
  echo "$VAULT_PASSWORD" > .vault_pass
  ansible-playbook site_baseline.yaml --check --diff
  rm -f .vault_pass          ← discarded after use
```

**Trust boundary:** The repo contains only encrypted secrets (`vault_arista.yaml`). The CI system holds the decryption key stored as a secret. Anyone with repo access sees only AES-256 ciphertext.

---

## 26. Windows Automation via WinRM

WinRM (Windows Remote Management) is to Windows what SSH is to Linux/network devices.

**Requirements on Windows host:**

```powershell
winrm quickconfig -q
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
winrm set winrm/config/service/auth '@{Basic="true"}'
```

**Inventory for Windows:**

```ini
[windows]
windows-vm  ansible_host=192.168.x.x

[windows:vars]
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_winrm_scheme=http
ansible_port=5985
ansible_user=Administrator
ansible_password=<password>
```

**Common Windows Modules:**

```yaml
# Test connectivity
- name: Ping Windows host
  ansible.windows.win_ping:

# Install software
- name: Install software package
  ansible.windows.win_package:
    path: C:\installers\app.exe
    state: present

# Manage services
- name: Ensure WinRM service is running
  ansible.windows.win_service:
    name: WinRM
    state: started
    start_mode: auto

# Run PowerShell
- name: Run PowerShell command
  ansible.windows.win_shell: |
    Get-Process | Select-Object -First 5
  register: ps_output
```

---

## 27. Multi-platform Reference

The same role and playbook structure applies across all network platforms — only collection names and connection variables change.

| Platform | Collection | `ansible_network_os` | Config module | Facts module |
|---|---|---|---|---|
| Arista EOS | `arista.eos` | `arista.eos.eos` | `arista.eos.eos_config` | `arista.eos.eos_facts` |
| Cisco IOS/XE | `cisco.ios` | `cisco.ios.ios` | `cisco.ios.ios_config` | `cisco.ios.ios_facts` |
| Juniper Junos | `junipernetworks.junos` | `junipernetworks.junos.junos` | `junipernetworks.junos.junos_config` | `junipernetworks.junos.junos_facts` |
| Palo Alto | `paloaltonetworks.panos` | — (XML API) | `panos_config_element` | `panos_facts` |
| F5 BIG-IP | `f5networks.f5_modules` | — (REST API) | `bigip_command` | `bigip_facts` |

**What changes per platform:** collection name, `ansible_network_os`, module names, template CLI syntax.

**What stays the same:** role structure, inventory layout, playbook anatomy, Jinja2 engine, Molecule scenario structure, pre-commit lint pipeline.

### Arista vs Cisco — Quick Module Comparison

| Task | Arista EOS | Cisco IOS |
|---|---|---|
| `ansible_network_os` | `arista.eos.eos` | `cisco.ios.ios` |
| Config push | `arista.eos.eos_config` | `cisco.ios.ios_config` |
| Show commands | `arista.eos.eos_command` | `cisco.ios.ios_command` |
| Facts | `arista.eos.eos_facts` | `cisco.ios.ios_facts` |
| VLANs | `arista.eos.eos_vlans` | `cisco.ios.ios_vlans` |
| Interfaces | `arista.eos.eos_interfaces` | `cisco.ios.ios_interfaces` |
| Banner | `arista.eos.eos_banner` | `cisco.ios.ios_banner` |

---

## Quick Reference — Key Commands

### Playbook Execution

```bash
ansible-playbook <playbook.yaml> --syntax-check
ansible-playbook <playbook.yaml> --check
ansible-playbook <playbook.yaml> --check --diff
ansible-playbook <playbook.yaml> --diff
ansible-playbook <playbook.yaml> --limit R1
ansible-playbook <playbook.yaml> -e "key=value"
ansible-playbook <playbook.yaml> -v / -vv / -vvv
```

### Ad-hoc

```bash
ansible <hosts> -m <module> -a "<args>"
ansible all -m ping
ansible arista_nodes -m arista.eos.eos_command -a "commands='show version'"
```

### Inventory

```bash
ansible-inventory --graph
ansible-inventory --list
ansible-inventory --host <hostname>
ansible <group> --list-hosts
```

### Collections

```bash
ansible-galaxy collection install -r requirements.yml
ansible-galaxy collection list
ansible-doc <module>
ansible-doc -l <collection>
```

### Vault

```bash
ansible-vault encrypt <file>
ansible-vault view <file>
ansible-vault edit <file>
ansible-vault rekey <file>
ansible-vault encrypt_string 'value' --name 'variable_name'
```

### Molecule

```bash
molecule init scenario --driver-name delegated
molecule lint
molecule syntax
molecule converge
molecule verify
molecule test
```

### Git

```bash
git init / git clone <url>
git checkout -b <branch>
git add . / git add <file>
git commit -m "type: message"
git push origin <branch>
git merge <branch>
git log --oneline
git status
```

### Lint

```bash
yamllint .
ansible-lint
ansible-lint roles/<name>/
pre-commit install
pre-commit run --all-files
```

---

*Ansible for Network Engineers — 4-Day Workshop Quick Reference*
