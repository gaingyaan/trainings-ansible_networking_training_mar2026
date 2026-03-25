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