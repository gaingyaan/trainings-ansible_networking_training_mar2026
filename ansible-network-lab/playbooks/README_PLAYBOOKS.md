# Ansible Day 2 Training Playbooks

## Working Playbooks for All Sections

This directory contains four working playbooks implementing all concepts from Day 2 of the Ansible training:

- YAML Fundamentals & Jinja2
- Playbook Anatomy & Variables
- Conditionals & Loops
- Linux Baseline Configuration Lab

---

## Playbook Overview

| Playbook                          | Concepts                                            | Targets     | Status        |
| --------------------------------- | --------------------------------------------------- | ----------- | ------------- |
| **1_baseline_setup.yaml**         | Environment verification, Jinja2 filters            | localhost   | Read-only     |
| **2_facts_and_variables.yaml**    | Facts discovery, variable precedence, register      | linux_hosts | Informational |
| **3_conditionals_and_loops.yaml** | when conditions, set_fact, loops, loop_control      | linux_hosts | Informational |
| **4_baseline.yaml**               | Full lab: cross-platform deploy, validation, assert | linux_hosts | Deployable    |

---

## Playbook 1: Environment Setup

**File:** `1_baseline_setup.yaml`

**Purpose:** Verify Ansible installation, linters, and configuration files.

**Concepts:**

- Ansible facts and `gather_facts`
- Jinja2 filters (`upper`, `lower`, `length`)
- Shell command execution
- Conditional task execution
- File existence checks

**Usage:**

```bash
# Syntax check
ansible-playbook playbooks/1_baseline_setup.yaml --syntax-check

# Run (read-only)
ansible-playbook playbooks/1_baseline_setup.yaml -v
```

**Output:** Displays Ansible version, installed linters, config file status, and demonstrates Jinja2 filters.

---

## Playbook 2: Facts & Variables Discovery

**File:** `2_facts_and_variables.yaml`

**Purpose:** Discover system facts, demonstrate variable precedence, and show the `register` keyword.

**Concepts:**

- System facts (`ansible_hostname`, `ansible_distribution`, `ansible_processor_vcpus`, etc.)
- Facts modules: `setup`, `stat`, `package_facts`
- Variable precedence (group_vars → host_vars → play vars → extra vars)
- The `register` keyword and task output
- Jinja2 templating in debug output
- Extra vars runtime overrides (`-e`)

**Usage:**

```bash
# Run all hosts
ansible-playbook playbooks/2_facts_and_variables.yaml -v

# Override variables at runtime
ansible-playbook playbooks/2_facts_and_variables.yaml \
  -e "environment=production" \
  -e "custom_var=test_value"

# Limit to one host
ansible-playbook playbooks/2_facts_and_variables.yaml --limit ubuntu-test
```

**Output:** Comprehensive facts dump, variable sources demonstrated, package info, disk usage, MOTD content.

**Key Sections:**

1. Host identity facts
2. Hardware and resources
3. Variable precedence analysis
4. Registered task output
5. Package information
6. Complete variable reference

---

## Playbook 3: Conditionals & Loops

**File:** `3_conditionals_and_loops.yaml`

**Purpose:** Demonstrate conditional execution, runtime variables, and loop patterns.

**Concepts:**

- `when` conditions: equality, contains, defined checks
- Negate conditions (`not`)
- AND/OR logic in `when`
- `set_fact` for derived variables
- Simple list loops
- List of dictionary loops
- `loop_control` options (`label`, `index_var`)
- Registering loop results

**Usage:**

```bash
# Full run with verbosity
ansible-playbook playbooks/3_conditionals_and_loops.yaml -vv

# Run only on Ubuntu
ansible-playbook playbooks/3_conditionals_and_loops.yaml --limit ubuntu_hosts

# Run only on RedHat
ansible-playbook playbooks/3_conditionals_and_loops.yaml --limit redhat_hosts
```

**Output:** Demonstrations of all conditional patterns, set_fact usage, complete loop patterns.

**Key Sections:**

1. Simple when conditions
2. Negate and logical operators (AND/OR)
3. File existence checks
4. set_fact for derived variables
5. Simple list loops
6. Loops with conditional
7. Dictionary list loops
8. Loop result registration
9. loop_control options
10. Pattern summary

---

## Playbook 4: Linux Baseline Configuration (Main Lab)

**File:** `4_baseline.yaml`

**Purpose:** Enforce complete baseline configuration across Ubuntu and Red Hat VMs.

**Concepts:**

- Cross-platform playbook design
- Conditional tasks per OS family
- Variable usage (group_vars, host_vars)
- File deployment (`copy` module)
- Service management (`service` module)
- Post-change validation with `assert`
- Package verification
- Safety patterns (`--check`, `--diff`)
- Tags for selective execution

**Inventory Requirements:**

```
inventory/
├── group_vars/
│   ├── all.yaml (motd_message, environment)
│   ├── ubuntu_hosts.yaml (packages_required, ansible_user)
│   └── redhat_hosts.yaml (packages_required, ansible_user)
└── host_vars/
    ├── ubuntu-test.yaml (server_role, motd)
    └── redhat-vm.yaml (server_role, motd)
```

**Safety Workflow (RECOMMENDED):**

```bash
# Step 1: Syntax check
ansible-playbook playbooks/4_baseline.yaml --syntax-check

# Step 2: Lint check
ansible-lint playbooks/4_baseline.yaml

# Step 3: Dry-run on one host with diff
ansible-playbook playbooks/4_baseline.yaml \
  --check --diff --limit ubuntu-test

# Step 4: Dry-run all hosts
ansible-playbook playbooks/4_baseline.yaml --check --diff

# Step 5: Apply changes
ansible-playbook playbooks/4_baseline.yaml --diff

# Step 6: Validate (runs assertions)
ansible-playbook playbooks/4_baseline.yaml --tags validate
```

**Usage Options:**

```bash
# Run full playbook
ansible-playbook playbooks/4_baseline.yaml

# Dry-run
ansible-playbook playbooks/4_baseline.yaml --check

# Show diffs
ansible-playbook playbooks/4_baseline.yaml --diff

# Ubuntu only
ansible-playbook playbooks/4_baseline.yaml --limit ubuntu-test

# RedHat only
ansible-playbook playbooks/4_baseline.yaml --limit redhat-vm

# Install packages only
ansible-playbook playbooks/4_baseline.yaml --tags packages

# Validate only
ansible-playbook playbooks/4_baseline.yaml --tags validate

# Dry-run on Ubuntu with full verbosity
ansible-playbook playbooks/4_baseline.yaml \
  --check --diff --limit ubuntu-test -vvv
```

**What It Does:**

1. **Pre-flight:** Display play targets and execution mode (dry-run vs live)
2. **Host Identity:** Display system facts (OS, version, IP, resources)
3. **Package Updates:** Update apt cache (Ubuntu only)
4. **Package Installation:** Install required packages per OS
5. **File Deployment:** Deploy MOTD message
6. **Service Management:** Ensure SSH is running and enabled
7. **Validation:** Assert all changes succeeded
   - Verify packages installed
   - Check MOTD file exists and has content
   - Verify SSH is active
   - Check filesystem usage
8. **Report:** Summary of baseline completion

**Output Example:**

```
TASK [5.2 — Verify required packages are installed]
ok: [ubuntu-test] => (item=vim)
ok: [ubuntu-test] => (item=curl)
ok: [ubuntu-test] => (item=net-tools)
ok: [ubuntu-test] => (item=tree)
```

---

## Inventory Setup

Your inventory is pre-configured with:

**File:** `inventory/hosts.ini`

```ini
[ubuntu_hosts]
ubuntu-test  ansible_host=172.20.0.57

[redhat_hosts]
redhat-vm    ansible_host=3.128.89.105

[linux_hosts:children]
ubuntu_hosts
redhat_hosts
```

**Group Variables** (apply to all hosts in a group):

- `inventory/group_vars/all.yaml` — All hosts
  - `ansible_python_interpreter: /usr/bin/python3`
  - `environment: lab`
  - `motd_message:` — Message of the Day

- `inventory/group_vars/ubuntu_hosts.yaml` — Ubuntu only
  - `packages_required:` — [vim, curl, net-tools, tree, htop]

- `inventory/group_vars/redhat_hosts.yaml` — RedHat only
  - `packages_required:` — [vim, curl, net-tools, tree, htop]

**Host Variables** (override group vars for one host):

- `inventory/host_vars/ubuntu-test.yaml`
  - `server_role: test`
  - `motd:` — Ubuntu-specific MOTD

- `inventory/host_vars/redhat-vm.yaml`
  - `server_role: test`
  - `motd:` — RedHat-specific MOTD

---

## Module Reference

Modules used in these playbooks:

| Module          | Usage                        | Playbook   |
| --------------- | ---------------------------- | ---------- |
| `debug`         | Display information          | All        |
| `assert`        | Validate conditions          | 4          |
| `stat`          | Check file properties        | 2, 3, 4    |
| `shell`         | Run shell commands           | 1, 2, 3, 4 |
| `apt`           | Install packages (Ubuntu)    | 4          |
| `dnf`           | Install packages (RedHat)    | 4          |
| `copy`          | Deploy files                 | 4          |
| `service`       | Manage services              | 4          |
| `package_facts` | Gather installed packages    | 2, 4       |
| `user`          | Create users (template only) | —          |
| `set_fact`      | Create runtime variables     | 3          |

---

## Common Patterns Used

### 1. Cross-Platform Conditionals

```yaml
when: ansible_os_family == "Debian"
when: ansible_os_family == "RedHat"
```

### 2. Variable from Facts

```yaml
msg: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
```

### 3. Loop with Label

```yaml
loop: "{{ packages_required }}"
loop_control:
  label: "{{ item }}"
```

### 4. Register and Use

```yaml
register: result
when: result.stat.exists
```

### 5. Assert with Feedback

```yaml
assert:
  that:
    - condition
  fail_msg: "Error message"
  success_msg: "Success message"
```

---

## Tips & Troubleshooting

### Does playbook have syntax errors?

```bash
ansible-playbook playbooks/1_baseline_setup.yaml --syntax-check
```

### Want to see what would change without applying?

```bash
ansible-playbook playbooks/4_baseline.yaml --check --diff --limit ubuntu-test
```

### Want to see detailed execution flow?

```bash
ansible-playbook playbooks/2_facts_and_variables.yaml -vvv
```

### Want to run only verification tasks?

```bash
ansible-playbook playbooks/4_baseline.yaml --tags validate
```

### Want to override a variable at runtime?

```bash
ansible-playbook playbooks/2_facts_and_variables.yaml -e "environment=production"
```

### Inventory not found?

Ensure `ansible.cfg` in project root contains:

```ini
[defaults]
inventory = ./inventory
```

Verify:

```bash
ansible-inventory --graph
ansible linux_hosts -m ping
```

---

## Learning Path

**Recommended execution order:**

1. **Start with setup verification** (read-only, safe)

   ```bash
   ansible-playbook playbooks/1_baseline_setup.yaml -v
   ```

2. **Explore facts and variables** (informational, identifies your inventory)

   ```bash
   ansible-playbook playbooks/2_facts_and_variables.yaml --limit ubuntu-test
   ```

3. **Learn conditionals and loops** (informational, all patterns demonstrated)

   ```bash
   ansible-playbook playbooks/3_conditionals_and_loops.yaml -vv
   ```

4. **Run the complete lab** (deployable, use safety patterns)
   ```bash
   # Dryrun first
   ansible-playbook playbooks/4_baseline.yaml --check --diff
   # Then apply
   ansible-playbook playbooks/4_baseline.yaml --diff
   ```

---

## Expected Results

After running `4_baseline.yaml`:

✓ Packages installed: vim, curl, net-tools, tree, htop  
✓ MOTD file deployed to /etc/motd  
✓ SSH service running and enabled  
✓ All assertions pass

Verify:

```bash
ansible linux_hosts -m shell -a "cat /etc/motd"
ansible linux_hosts -m shell -a "which vim curl tree"
ansible linux_hosts -m shell -a "systemctl is-active ssh"
```

---

## Next Steps

- **Day 3:** Roles, Templates, Handlers, and Network device modules
- **Day 4:** Advanced patterns, error handling, debugging

For questions, review the training document: `day2_ansible_v4_no_network.md`

---

_Generated: Day 2 Training | Ansible for Network Engineers_
