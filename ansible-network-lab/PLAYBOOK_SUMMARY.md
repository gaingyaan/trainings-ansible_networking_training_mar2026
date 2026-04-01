# Playbook Summary — Day 2 Training

## 📋 Created Playbooks

### Playbook 1: `1_baseline_setup.yaml`

**Section:** Block 1 — Day 1 Remainder  
**Goal:** Verify Ansible environment setup  
**Targets:** localhost (read-only)

**Covers:**

- ✓ YAML fundamentals recap
- ✓ Jinja2 filter demonstrations
- ✓ Configuration file verification
- ✓ Linter installation check

**Run:**

```bash
ansible-playbook playbooks/1_baseline_setup.yaml -v
```

---

### Playbook 2: `2_facts_and_variables.yaml`

**Section:** Block 2 — Variables, Facts & Register  
**Goal:** Discover system facts and demonstrate variable precedence  
**Targets:** linux_hosts

**Covers:**

- ✓ System facts (hostname, OS, IP, memory, CPU)
- ✓ Variable precedence hierarchy
- ✓ The `register` keyword
- ✓ Package facts discovery
- ✓ Variable sources: group_vars → host_vars → play vars → extra vars

**Sections in Playbook:**

1. Host identity facts
2. Hardware and resources
3. Variable precedence analysis
4. Registered task output
5. Package information
6. Complete variable reference

**Run:**

```bash
# Basic
ansible-playbook playbooks/2_facts_and_variables.yaml -v

# With runtime overrides
ansible-playbook playbooks/2_facts_and_variables.yaml -e "environment=production"
```

---

### Playbook 3: `3_conditionals_and_loops.yaml`

**Section:** Block 3 — Conditionals & Loops  
**Goal:** Demonstrate all conditional patterns and loop techniques  
**Targets:** linux_hosts

**Covers:**

- ✓ `when` conditions (equality, contains, defined, negate)
- ✓ AND/OR logic
- ✓ `set_fact` for derived variables
- ✓ Simple list loops
- ✓ Dictionary list loops
- ✓ `loop_control` with labels and index
- ✓ Registering loop results

**Sections in Playbook:**

1. Simple when conditions
2. Negate and logical operators
3. File existence checks
4. set_fact for runtime variables
5. Simple list loops
6. Loops with conditionals
7. Dictionary list loops
8. Loop result registration
9. loop_control options
10. Complete pattern summary

**Run:**

```bash
ansible-playbook playbooks/3_conditionals_and_loops.yaml -vv
ansible-playbook playbooks/3_conditionals_and_loops.yaml --limit ubuntu_hosts
```

---

### Playbook 4: `4_baseline.yaml`

**Section:** Block 4 — Linux Baseline Lab (MAIN LAB)  
**Goal:** Enforce complete baseline configuration  
**Targets:** linux_hosts

**Covers:**

- ✓ Cross-platform playbook design
- ✓ Conditional execution per OS family
- ✓ Package installation (apt for Ubuntu, dnf for Red Hat)
- ✓ File deployment (MOTD)
- ✓ Service management (SSH)
- ✓ Post-change validation with `assert`
- ✓ Safety patterns (`--check`, `--diff`)
- ✓ Tags for selective execution

**Sections in Playbook:**

1. Host identity display
2. Package cache update (Debian only)
3. Package installation (cross-platform)
4. MOTD deployment
5. Service management
6. Validation & assertions
7. Summary report

**Safety Workflow (REQUIRED):**

```bash
# 1. Syntax check
ansible-playbook playbooks/4_baseline.yaml --syntax-check

# 2. Lint
ansible-lint playbooks/4_baseline.yaml

# 3. Dry-run single host
ansible-playbook playbooks/4_baseline.yaml --check --diff --limit ubuntu-test

# 4. Dry-run all
ansible-playbook playbooks/4_baseline.yaml --check --diff

# 5. Apply
ansible-playbook playbooks/4_baseline.yaml --diff

# 6. Validate
ansible-playbook playbooks/4_baseline.yaml --tags validate
```

**What Gets Deployed:**

- Packages: vim, curl, net-tools, tree, htop
- MOTD file at /etc/motd with welcome message
- SSH service: started and enabled

---

## 📁 Updated Inventory Files

### `inventory/group_vars/all.yaml`

```yaml
ansible_python_interpreter: /usr/bin/python3
environment: lab
motd_message: "... welcome message ..."
```

### `inventory/group_vars/ubuntu_hosts.yaml`

```yaml
ansible_user: ansible
packages_required: [vim, curl, net-tools, tree, htop]
```

### `inventory/group_vars/redhat_hosts.yaml`

```yaml
ansible_user: ansible
packages_required: [vim, curl, net-tools, tree, htop]
```

### `inventory/host_vars/ubuntu-test.yaml`

```yaml
server_role: "test"
motd: "Ubuntu test server — lab environment"
```

### `inventory/host_vars/redhat-vm.yaml`

```yaml
server_role: "test"
motd: "Red Hat test server — lab environment"
```

---

## 🔑 Key Concepts Demonstrated

### Concepts by Playbook

| Concept                | Playbook 1 | Playbook 2 | Playbook 3 | Playbook 4 |
| ---------------------- | ---------- | ---------- | ---------- | ---------- |
| Facts (`gather_facts`) | ✓          | ✓          | ✓          | ✓          |
| Jinja2 filters         | ✓          | ✓          | ✓          | ✓          |
| Variables              | ✓          | ✓          | ✓          | ✓          |
| Conditionals (`when`)  | ✓          | ✓          | ✓          | ✓          |
| Loops                  | —          | —          | ✓          | ✓          |
| Register               | —          | ✓          | ✓          | ✓          |
| set_fact               | —          | —          | ✓          | —          |
| Cross-platform         | —          | —          | —          | ✓          |
| Assertions             | —          | —          | —          | ✓          |
| File deployment        | —          | —          | —          | ✓          |
| Service mgmt           | —          | —          | —          | ✓          |

---

## 🚀 Quick Start

```bash
cd ansible-network-lab

# 1. Verify environment
ansible-playbook playbooks/1_baseline_setup.yaml -v

# 2. Explore your inventory
ansible-playbook playbooks/2_facts_and_variables.yaml --limit ubuntu-test

# 3. See loops and conditionals
ansible-playbook playbooks/3_conditionals_and_loops.yaml -vv

# 4. Deploy baseline (safe approach)
ansible-playbook playbooks/4_baseline.yaml --check --diff --limit ubuntu-test
ansible-playbook playbooks/4_baseline.yaml --diff
```

---

## 📚 Documentation

Each playbook includes:

- Clear section headers (numbered in playbook)
- Inline comments explaining concepts
- Multiple examples per pattern
- Practical demonstrations

Full reference guide: [`README_PLAYBOOKS.md`](README_PLAYBOOKS.md)

---

## ✅ Testing Checklist

- [x] Playbook 1: Runs successfully on localhost
- [x] Playbook 2: Displays facts from linux_hosts
- [x] Playbook 3: Demonstrates all conditional/loop patterns
- [x] Playbook 4: Dry-run successful with `--check --diff`
- [x] All playbooks pass `--syntax-check`
- [x] Inventory variables configured correctly

---

## 📝 Notes

All playbooks are **fully functional** and ready to run against:

```
[ubuntu_hosts]
ubuntu-test  ansible_host=172.20.0.57

[redhat_hosts]
redhat-vm    ansible_host=3.128.89.105
```

For network-specific modules (ios_config, eos_config), wait for Day 3 after cEOS environment is ready.

**Concepts are identical** — only module names change for network devices.

---

Generated: Day 2 Training | Ansible for Network Engineers
