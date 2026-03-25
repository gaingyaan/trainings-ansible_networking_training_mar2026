## 2.2 Playbook Anatomy

```yaml
---
# ── Play header ──────────────────────────────────────────────────
- name: Ensure baseline packages are installed # human-readable name
  hosts: ubuntu_hosts # which inventory group
  gather_facts: true # collect system info first
  become: true # use sudo

  # ── Play-level variables ────────────────────────────────────────
  vars:
    custom_package: htop

  # ── Tasks ───────────────────────────────────────────────────────
  tasks:
    - name: Update apt cache # every task needs a name
      ansible.builtin.apt: # fully qualified module name
        update_cache: true
        cache_valid_time: 3600

    - name: Install required packages
      ansible.builtin.apt:
        name: "{{ packages_required }}" # variable from group_vars
        state: present

    - name: Install custom package
      ansible.builtin.apt:
        name: "{{ custom_package }}"
        state: present

    - name: Check disk usage
      ansible.builtin.shell:
        cmd: df -h /
      register: disk_usage # capture output
      changed_when: false # shell tasks always report changed — fix this

    - name: Show disk usage
      debug:
        var: disk_usage.stdout_lines
```

---

---

## 2.3 Key Playbook Fields

| Field           | Where        | Purpose                                           |
| --------------- | ------------ | ------------------------------------------------- |
| `name`          | play, task   | Human-readable label — appears in output and logs |
| `hosts`         | play         | Which inventory group to target                   |
| `gather_facts`  | play         | Auto-collect system info before tasks run         |
| `become`        | play or task | Privilege escalation — sudo on Linux              |
| `vars`          | play         | Variables scoped to this play only                |
| `tasks`         | play         | Ordered list of tasks                             |
| `register`      | task         | Save task output into a named variable            |
| `when`          | task         | Conditional — run only if expression is true      |
| `loop`          | task         | Repeat task for each item in a list               |
| `changed_when`  | task         | Define what counts as a change                    |
| `ignore_errors` | task         | Continue even if this task fails                  |

---

## 2.4 YAML Best Practices for Playbooks

### Always use Fully Qualified Collection Names (FQCN)

```yaml
# WRONG — ambiguous
- name: Install package
  apt:
    name: vim

# CORRECT
- name: Install package
  ansible.builtin.apt:
    name: vim
```

### Always name every task

```yaml
# WRONG
- ansible.builtin.apt:
    name: vim

# CORRECT
- name: Install vim editor
  ansible.builtin.apt:
    name: vim
```

### Avoid free-form module syntax

```yaml
# WRONG
- name: Check uptime
  ansible.builtin.shell: uptime

# CORRECT
- name: Check uptime
  ansible.builtin.shell:
    cmd: uptime
```

### Use true/false not yes/no

```yaml
# WRONG
become: yes
gather_facts: no

# CORRECT
become: true
gather_facts: false
```

### Always add changed_when to shell/command tasks

```yaml
# WRONG — always reports changed even when nothing changed
- name: Get hostname
  ansible.builtin.shell:
    cmd: hostname

# CORRECT
- name: Get hostname
  ansible.builtin.shell:
    cmd: hostname
  changed_when: false # this task never makes a change — tell Ansible that
```
