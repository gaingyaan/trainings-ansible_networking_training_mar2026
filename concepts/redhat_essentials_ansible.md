# Red Hat Linux — Essentials for Ansible

### Supplementary Reference — Ansible for Network Engineers

---

> **Purpose of this document:**
> This is a standalone reference covering two things:
>
> 1. Red Hat Linux basics that a network engineer needs to know when working with Ansible in a Red Hat environment
> 2. A side-by-side command reference for switching between Ubuntu and Red Hat
>
> The lab controller remains Ubuntu throughout the programme. The Red Hat VM in the lab is a **managed node** — Ansible connects to it and configures it, just like it does with the Ubuntu Test VM. This document helps you understand what is different on the Red Hat side and how to handle it in playbooks.

---

---

# Part 1 — Red Hat Linux Basics Required for Ansible

## 1.1 What Makes Red Hat Different from Ubuntu

Both Ubuntu and Red Hat Enterprise Linux (RHEL) are Linux distributions. They share the same kernel, the same shell, the same filesystem structure, and the same core utilities. The differences that matter for Ansible are:

| Area             | Ubuntu             | Red Hat / RHEL                        |
| ---------------- | ------------------ | ------------------------------------- |
| Package manager  | `apt`              | `dnf` (RHEL 8+) / `yum` (RHEL 7)      |
| Package format   | `.deb`             | `.rpm`                                |
| Default firewall | `ufw`              | `firewalld`                           |
| Security module  | AppArmor           | **SELinux**                           |
| Service manager  | `systemctl` (same) | `systemctl` (same)                    |
| Shell            | bash (same)        | bash (same)                           |
| Python           | Pre-installed      | Pre-installed (Python 3.x on RHEL 8+) |
| Ansible install  | `apt` or `pip`     | `dnf` or `pip`                        |

For Ansible automation, the only things that regularly affect playbook behaviour are the **package manager** and **SELinux**. Everything else is either identical or irrelevant for managed node operations.

---

## 1.2 Package Management — dnf and yum

### dnf — Default on RHEL 8 and later

`dnf` (Dandified YUM) replaced `yum` as the default package manager on RHEL 8+. It is faster, handles dependency resolution better, and is the one you will encounter on modern Red Hat systems.

```bash
# Update package metadata
sudo dnf check-update

# Install a package
sudo dnf install -y curl

# Remove a package
sudo dnf remove -y curl

# Search for a package
sudo dnf search ansible

# Show package info
sudo dnf info ansible

# List installed packages
sudo dnf list installed

# Update all packages
sudo dnf update -y

# Install a specific version
sudo dnf install -y ansible-2.9.27
```

### yum — Legacy, RHEL 7 and earlier

`yum` commands are syntactically identical to `dnf`. On RHEL 8+, `yum` is a symlink to `dnf` — the commands still work.

```bash
# Same syntax as dnf — works on RHEL 7 and earlier
sudo yum install -y curl
sudo yum remove -y curl
sudo yum update -y
```

### Installing Ansible on Red Hat

```bash
# RHEL 8+ — via dnf (requires EPEL or AAP subscription)
sudo dnf install -y epel-release
sudo dnf install -y ansible

# Any RHEL version — via pip (recommended for consistent version)
sudo dnf install -y python3 python3-pip
pip3 install ansible --user

# Verify
ansible --version
```

---

## 1.3 SELinux — The One That Catches People Out

**SELinux (Security-Enhanced Linux)** is a mandatory access control system built into Red Hat. It enforces security policies at the kernel level — beyond standard file permissions. It is enabled and enforcing by default on all RHEL systems.

**Why it matters for Ansible:**

- Ansible copies files, writes configs, and executes scripts on managed nodes
- SELinux can silently block these operations even when file permissions look correct
- A task that works fine on Ubuntu may fail silently on Red Hat because SELinux denies it

### SELinux Modes

```bash
# Check current SELinux status and mode
getenforce
# Output: Enforcing | Permissive | Disabled

sestatus
# Detailed status including policy type
```

| Mode         | Behaviour                                                 |
| ------------ | --------------------------------------------------------- |
| `Enforcing`  | Blocks and logs policy violations — default on RHEL       |
| `Permissive` | Logs violations but does not block — useful for debugging |
| `Disabled`   | SELinux completely off — not recommended in production    |

### Temporarily Switch to Permissive (for debugging only)

```bash
# Switch to permissive — survives until reboot
sudo setenforce 0

# Switch back to enforcing
sudo setenforce 1
```

### Check SELinux Denials

```bash
# View recent SELinux denials
sudo ausearch -m avc -ts recent

# Human-readable denial messages
sudo sealert -a /var/log/audit/audit.log
```

### SELinux File Contexts — Common Ansible Issue

When Ansible copies a file to a Red Hat managed node, the file may get the wrong SELinux context, causing services to fail even though the file permissions are correct.

```bash
# Check SELinux context of a file
ls -Z /etc/ssh/sshd_config

# Restore correct context for a file
sudo restorecon -v /etc/ssh/sshd_config

# Restore correct context recursively
sudo restorecon -Rv /etc/
```

### Handling SELinux in Ansible Tasks

```yaml
# Copy a file and set the correct SELinux context
- name: Copy SSH config to managed node
  ansible.builtin.copy:
    src: sshd_config
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: "0600"
    seuser: system_u
    serole: object_r
    setype: etc_t
    selevel: s0

# Use sefcontext module to manage SELinux file contexts
- name: Set SELinux context for app directory
  community.general.sefcontext:
    target: "/opt/myapp(/.*)?"
    setype: httpd_sys_content_t
    state: present

# Use seboolean to toggle SELinux booleans
- name: Allow httpd to connect to network
  ansible.posix.seboolean:
    name: httpd_can_network_connect
    state: true
    persistent: true
```

> **Practical note for the lab:** The Red Hat VM in the lab is a managed node. If an Ansible task fails on it but works on the Ubuntu VM, check SELinux first. Run `sudo ausearch -m avc -ts recent` on the Red Hat VM to see if a denial is the cause.

---

## 1.4 firewalld — Red Hat's Default Firewall

Red Hat uses `firewalld` instead of Ubuntu's `ufw`. The concepts are the same — zones and rules — the commands differ.

```bash
# Check firewalld status
sudo systemctl status firewalld

# Check open ports/services
sudo firewall-cmd --list-all

# Open a port permanently
sudo firewall-cmd --permanent --add-port=22/tcp
sudo firewall-cmd --reload

# Open a service by name
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload

# Remove a port
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload
```

### Managing firewalld in Ansible

```yaml
- name: Open SSH port on Red Hat managed node
  ansible.posix.firewalld:
    port: 22/tcp
    permanent: true
    state: enabled
    immediate: true

- name: Enable a service through firewalld
  ansible.posix.firewalld:
    service: https
    permanent: true
    state: enabled
    immediate: true
```

---

## 1.5 systemctl — Service Management (Same on Both)

`systemctl` works identically on Ubuntu and Red Hat. This is one area where there is no difference.

```bash
# Start / stop / restart a service
sudo systemctl start sshd
sudo systemctl stop sshd
sudo systemctl restart sshd

# Enable service to start on boot
sudo systemctl enable sshd

# Disable service from starting on boot
sudo systemctl disable sshd

# Check service status
sudo systemctl status sshd

# Reload service config without restart
sudo systemctl reload sshd
```

Note: the SSH daemon is called `sshd` on Red Hat and `ssh` on Ubuntu — the `systemctl` commands are otherwise identical.

---

## 1.6 File Permissions and SSH Keys

File permission commands are identical on both platforms. The only difference relevant to Ansible is that Red Hat applies SELinux context checks on top of standard permissions.

```bash
# Set permissions on SSH private key
chmod 600 ~/.ssh/id_rsa

# Set permissions on SSH directory
chmod 700 ~/.ssh

# Change ownership
chown ubuntu:ubuntu ~/.ssh/authorized_keys   # Ubuntu
chown ec2-user:ec2-user ~/.ssh/authorized_keys   # RHEL on AWS
chown ansible:ansible ~/.ssh/authorized_keys  # custom user
```

### SSH Setup for Red Hat Managed Node

```bash
# From Ubuntu Controller — copy SSH key to Red Hat managed node
ssh-copy-id ansible@<redhat_vm_ip>

# Test connection
ssh ansible@<redhat_vm_ip> "echo connected && cat /etc/redhat-release"
```

Default SSH user on Red Hat varies:

- RHEL on-premise: whatever user was created during install
- RHEL on AWS EC2: `ec2-user`
- In the lab: use the user configured on the VM (confirm with your lab setup)

---

## 1.7 Python on Red Hat

Ansible requires Python on managed nodes for most modules. Red Hat 8+ ships with Python 3 by default.

```bash
# Check Python version
python3 --version

# Check pip
pip3 --version

# Install pip if missing
sudo dnf install -y python3-pip

# Install a Python package
pip3 install pyserial --user
```

In `ansible.cfg` or `group_vars`, tell Ansible which Python interpreter to use:

```yaml
# inventory/group_vars/redhat_hosts.yaml
---
ansible_python_interpreter: /usr/bin/python3
```

---

## 1.8 Red Hat Subscription — Awareness Only

RHEL requires a subscription to access official Red Hat repositories. Without a subscription, `dnf install` will fail for many packages.

```bash
# Check subscription status
sudo subscription-manager status

# Register with Red Hat (requires Red Hat account)
sudo subscription-manager register --username <username> --password <password>

# List available subscriptions
sudo subscription-manager list --available

# Attach a subscription
sudo subscription-manager attach --auto
```

**Alternative — use EPEL for additional packages without a full subscription:**

```bash
# Enable EPEL (Extra Packages for Enterprise Linux) — free, community-maintained
sudo dnf install -y epel-release

# Now install packages from EPEL
sudo dnf install -y ansible
```

---

---

# Part 2 — Ubuntu vs Red Hat: Parallel Command Reference

## 2.1 Package Management

| Task                    | Ubuntu (`apt`)              | Red Hat (`dnf`)                      |
| ----------------------- | --------------------------- | ------------------------------------ |
| Update package metadata | `sudo apt update`           | `sudo dnf check-update`              |
| Install a package       | `sudo apt install -y <pkg>` | `sudo dnf install -y <pkg>`          |
| Remove a package        | `sudo apt remove -y <pkg>`  | `sudo dnf remove -y <pkg>`           |
| Search for a package    | `apt search <pkg>`          | `dnf search <pkg>`                   |
| Show package info       | `apt show <pkg>`            | `dnf info <pkg>`                     |
| List installed packages | `apt list --installed`      | `dnf list installed`                 |
| Update all packages     | `sudo apt upgrade -y`       | `sudo dnf update -y`                 |
| Install from local file | `sudo dpkg -i <file>.deb`   | `sudo rpm -i <file>.rpm`             |
| Add a repo              | `add-apt-repository`        | `sudo dnf config-manager --add-repo` |

---

## 2.2 Service Management

Identical on both platforms — `systemctl` commands are the same. Only the service name differs for SSH.

| Task               | Ubuntu                              | Red Hat                             |
| ------------------ | ----------------------------------- | ----------------------------------- |
| SSH service name   | `ssh`                               | `sshd`                              |
| Start SSH          | `sudo systemctl start ssh`          | `sudo systemctl start sshd`         |
| Enable SSH on boot | `sudo systemctl enable ssh`         | `sudo systemctl enable sshd`        |
| All other services | `sudo systemctl <action> <service>` | `sudo systemctl <action> <service>` |

---

## 2.3 Firewall

| Task            | Ubuntu (`ufw`)          | Red Hat (`firewalld`)                                                              |
| --------------- | ----------------------- | ---------------------------------------------------------------------------------- |
| Check status    | `sudo ufw status`       | `sudo firewall-cmd --list-all`                                                     |
| Enable firewall | `sudo ufw enable`       | `sudo systemctl start firewalld`                                                   |
| Open a port     | `sudo ufw allow 22/tcp` | `sudo firewall-cmd --permanent --add-port=22/tcp && sudo firewall-cmd --reload`    |
| Close a port    | `sudo ufw deny 22/tcp`  | `sudo firewall-cmd --permanent --remove-port=22/tcp && sudo firewall-cmd --reload` |
| Allow a service | `sudo ufw allow ssh`    | `sudo firewall-cmd --permanent --add-service=ssh && sudo firewall-cmd --reload`    |

---

## 2.4 Ansible Inventory — Group Variables

```yaml
# inventory/group_vars/ubuntu_hosts.yaml
---
ansible_user: ubuntu
ansible_python_interpreter: /usr/bin/python3
ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

```yaml
# inventory/group_vars/redhat_hosts.yaml
---
ansible_user: ansible # or ec2-user depending on setup
ansible_python_interpreter: /usr/bin/python3
ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

---

## 2.5 Ansible Modules — Package Management

| Task            | Ubuntu module             | Red Hat module            | Cross-platform module     |
| --------------- | ------------------------- | ------------------------- | ------------------------- |
| Install package | `ansible.builtin.apt`     | `ansible.builtin.dnf`     | `ansible.builtin.package` |
| Manage service  | `ansible.builtin.service` | `ansible.builtin.service` | `ansible.builtin.service` |
| Copy file       | `ansible.builtin.copy`    | `ansible.builtin.copy`    | `ansible.builtin.copy`    |
| Manage firewall | `community.general.ufw`   | `ansible.posix.firewalld` | —                         |
| Manage SELinux  | — (AppArmor)              | `ansible.posix.seboolean` | —                         |

**Use `ansible.builtin.package` for cross-platform playbooks:**

```yaml
# Works on both Ubuntu and Red Hat — Ansible selects the right package manager
- name: Install curl on any Linux host
  ansible.builtin.package:
    name: curl
    state: present
```

---

## 2.6 Playbook Examples — Ubuntu Only vs Red Hat Only vs Cross-platform

### Ubuntu only

```yaml
- name: Install packages on Ubuntu
  ansible.builtin.apt:
    name:
      - curl
      - vim
      - net-tools
    state: present
    update_cache: true
  when: ansible_os_family == "Debian"
```

### Red Hat only

```yaml
- name: Install packages on Red Hat
  ansible.builtin.dnf:
    name:
      - curl
      - vim
      - net-tools
    state: present
  when: ansible_os_family == "RedHat"
```

### Cross-platform — branch by OS family

```yaml
- name: Install packages — Ubuntu
  ansible.builtin.apt:
    name: "{{ packages }}"
    state: present
    update_cache: true
  when: ansible_os_family == "Debian"

- name: Install packages — Red Hat
  ansible.builtin.dnf:
    name: "{{ packages }}"
    state: present
  when: ansible_os_family == "RedHat"
```

### Cross-platform — single task using package module

```yaml
- name: Install packages — any Linux
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop: "{{ packages }}"
```

---

## 2.7 Key Facts — OS Detection in Playbooks

These facts are populated automatically when `gather_facts: true` (Linux hosts only):

| Fact                                 | Ubuntu example | Red Hat example |
| ------------------------------------ | -------------- | --------------- |
| `ansible_os_family`                  | `Debian`       | `RedHat`        |
| `ansible_distribution`               | `Ubuntu`       | `RedHat`        |
| `ansible_distribution_version`       | `22.04`        | `8.7`           |
| `ansible_distribution_major_version` | `22`           | `8`             |
| `ansible_pkg_mgr`                    | `apt`          | `dnf`           |

```bash
# Check facts for any Linux host
ansible ubuntu_test -m setup -a "filter=ansible_distribution*"
ansible redhat_vm -m setup -a "filter=ansible_distribution*"
ansible redhat_vm -m setup -a "filter=ansible_os_family"
```

---

## 2.8 Inventory — Adding the Red Hat VM

Add to `inventory/hosts.ini`:

```ini
# ── Linux Hosts ────────────────────────────────────────
[ubuntu_hosts]
ubuntu_test  ansible_host=192.168.1.20

[redhat_hosts]
redhat_vm    ansible_host=192.168.1.30

[linux_hosts:children]
ubuntu_hosts
redhat_hosts
```

Add `inventory/group_vars/redhat_hosts.yaml`:

```yaml
---
ansible_user: ansible
ansible_python_interpreter: /usr/bin/python3
ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

Test connectivity:

```bash
# Ping both Linux hosts
ansible linux_hosts -m ping

# Check OS facts on Red Hat VM
ansible redhat_hosts -m setup -a "filter=ansible_distribution*"

# Run a command on Red Hat VM only
ansible redhat_hosts -m shell -a "cat /etc/redhat-release"
```

---

## 2.9 Installing Ansible Collections for Red Hat Support

Some Red Hat-specific modules require additional collections:

```yaml
# Add to requirements.yml
collections:
  - name: ansible.netcommon
    version: ">=5.1.0,<6.0.0"
  - name: cisco.ios
    version: ">=5.0.0,<6.0.0"
  - name: ansible.posix # firewalld, SELinux modules
    version: ">=1.5.0"
  - name: community.general # ufw and other utilities
    version: ">=7.0.0"
```

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## 2.10 Quick Troubleshooting — Red Hat Managed Node

When a task works on Ubuntu but fails on Red Hat, check in this order:

```bash
# 1. Is Python available on the Red Hat VM?
ansible redhat_hosts -m shell -a "python3 --version"

# 2. Is SSH working correctly?
ansible redhat_hosts -m ping -v

# 3. Is SELinux blocking something? (run on the Red Hat VM)
sudo ausearch -m avc -ts recent

# 4. Is firewalld blocking the connection? (run on the Red Hat VM)
sudo firewall-cmd --list-all

# 5. Check what OS family Ansible sees
ansible redhat_hosts -m setup -a "filter=ansible_os_family"

# 6. Check privilege escalation works
ansible redhat_hosts -m shell -a "whoami" --become
```

---

# Reference — Red Hat Quick Cheat Sheet

## Package Management

```bash
sudo dnf install -y <package>
sudo dnf remove -y <package>
sudo dnf update -y
sudo dnf search <package>
sudo dnf info <package>
sudo dnf list installed
```

## SELinux

```bash
getenforce                          # check mode
sudo setenforce 0                   # permissive (debug only)
sudo setenforce 1                   # back to enforcing
sudo ausearch -m avc -ts recent     # view denials
sudo restorecon -Rv <path>          # restore file context
```

## firewalld

```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-port=<port>/tcp
sudo firewall-cmd --permanent --add-service=<service>
sudo firewall-cmd --reload
```

## OS Detection in Playbooks

```yaml
when: ansible_os_family == "RedHat"     # Red Hat, CentOS, Fedora
when: ansible_os_family == "Debian"     # Ubuntu, Debian
when: ansible_distribution == "RedHat"
when: ansible_distribution_major_version == "8"
```

## Cross-platform Package Module

```yaml
ansible.builtin.package:
  name: <package>
  state: present # works on both apt and dnf systems
```

---

_End of Red Hat Essentials Reference_
