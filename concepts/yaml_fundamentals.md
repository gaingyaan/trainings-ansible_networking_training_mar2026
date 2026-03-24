## 1.1 YAML Fundamentals **[BASE]**

### Core Rules

- **Spaces only** — never tabs. 2 spaces per indent level.
- **Case sensitive** — `Name` and `name` are different keys.
- `---` marks the start of a YAML document.
- `#` is a comment.

### Scalars, Lists, Dictionaries

```yaml
---
# Scalar
hostname: webserver01
port: 8080
enabled: true

# List
packages:
  - vim
  - curl
  - net-tools

# Dictionary
service:
  name: nginx
  state: started
  enabled: true

# List of dictionaries — most common pattern in Ansible
users:
  - name: alice
    shell: /bin/bash
    groups: sudo
  - name: bob
    shell: /bin/bash
    groups: users
```

### Multi-line Strings

```yaml
# Literal block (|) — preserves newlines
motd_message: |
  Welcome to the lab server.
  Authorised access only.
  All activity is monitored.

# Folded block (>) — newlines become spaces
description: >
  This server runs the lab environment
  for the Ansible training programme.
```

### Common YAML Mistakes

```yaml
# Tab character — YAML will reject this
hostname:	webserver01      # ← tab, not spaces

# Inconsistent indent
packages:
  - vim
   - curl                   # ← extra space, breaks list

# No space after colon
hostname:webserver01         # ← parsed as string, not key-value

# Special character unquoted
description: Server: Lab     # ← colon in value, must quote
description: "Server: Lab"   # ✓ correct
```

**Validate YAML:**

```bash
python3 -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))" inventory/hosts.yaml && echo "YAML OK"
```

---

## 1.2 yamllint + ansible-lint

```bash
# Check if installed
yamllint --version 2>/dev/null || pip3 install yamllint --break-system-packages
ansible-lint --version 2>/dev/null || pip3 install ansible-lint --break-system-packages
```

Create `.yamllint` in project root:

```yaml
---
extends: default

rules:
  line-length:
    max: 120
    level: warning
  truthy:
    allowed-values: ["true", "false"]
    check-keys: false
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
# Run lint
yamllint .
ansible-lint
```

---

## 1.4 Pre-commit Hooks

```bash
pre-commit --version 2>/dev/null || pip3 install pre-commit --break-system-packages
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
pre-commit install
pre-commit run --all-files
```
