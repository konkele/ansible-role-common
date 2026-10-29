# Common Ansible Role

The **`common` role** provides baseline operating system configuration for Ubuntu and Debian-based hosts. It is intended to be applied early in a playbook to establish a consistent system foundation for downstream roles.

This role follows a deterministic, inventory-driven design that emphasizes predictability, idempotence, and long-term maintainability.

---

## Features

* **User Management**
  * Create system and human users
  * Optional UID assignment
  * Automatic primary-group creation
  * Manage SSH authorized keys, with configurable exclusivity
  * Optional passwordless sudo access
  * User tagging for classification and automation
  * Merge user definitions across global, group, and host scopes
  * Persist reconciled user metadata for downstream roles

* **Timezone Management**
  * Enforce a consistent system timezone

* **DNS Configuration**
  * Configure systemd-resolved search domains
  * Uses drop-in configuration under `/etc/systemd/resolved.conf.d`

* **Firewall Management**
  * Optional integration with `geerlingguy.firewall`
  * Enabled by default
  * Can be disabled per inventory scope

* **Package Management**
  * Install a configurable list of packages
  * Skip installation when the package list is empty
  * Can be disabled per inventory scope

* **NFS Storage**
  * Install NFS client packages
  * Create mount points
  * Configure persistent NFS mounts

* **Local Disk Management**
  * Detect existing filesystems
  * Format uninitialized disks
  * Configure UUID-based persistent mounts
  * Remove mounts and fstab entries
  * Optional mount-point cleanup

* **System Updates**
  * Refresh package metadata
  * Perform distribution upgrades
  * Remove obsolete packages
  * Automatically reboot when required
  * Executed only when explicitly tagged

* **Metadata Persistence**
  * Writes merged user state to `/etc/ansible/common_users.yml`
  * Provides a structured source of truth for downstream roles

---

## Firewall Behavior

Firewall management is implemented as a soft dependency.

When enabled, the role imports the `geerlingguy.firewall` role. Firewall management can be disabled globally, per group, or per host:

```yaml
common_firewall_manage: false
```

When disabled, the role makes no changes to firewall configuration and does not assume ownership of existing firewall state.

---

## Package Management

Define packages to install using common_packages in your inventory, typically in group_vars/all.yml to apply them to all hosts:

```yaml
# group_vars/all.yml

common_packages:
  - curl
  - htop
  - jq
  - unzip
```

Packages are installed using APT with state: present. Existing installed packages are not upgraded by this task, and an empty list skips package installation.

To install packages only for a specific group, define common_packages in that group's inventory file instead:

```yaml
# group_vars/docker.yml

common_packages:
  - docker.io
  - docker-compose
```

Package management is enabled by default. To disable it for a host or group:

```yaml
common_packages_manage: false
```

Run only the package installation tasks:

```bash
ansible-playbook site.yml --tags packages
Package installation is separate from system upgrades, which must be explicitly run using the update tag.
```

---

## User Definition Model

This role intentionally does **not** allow `common_users` to be defined directly in inventory.

Ansible does not automatically merge list variables across inventory scopes. Instead, define users through three explicit variables:

| Scope | Variable | Intended Location |
|---|---|---|
| Global | `common_users_global` | `group_vars/all.yml` |
| Group | `common_users_group` | `group_vars/<group>.yml` |
| Host | `common_users_host` | `host_vars/<host>.yml` |

The role processes these inputs in a fixed order:

```text
common_users_global
        ↓
common_users_group
        ↓
common_users_host
```

User definitions are merged by `name`. When the same user appears in multiple scopes:

- **Scalar values:** Later definitions override earlier values, so host-level values have the highest precedence.
- **List values:** Lists are merged using Ansible's `append_rp` behavior, retaining values from multiple scopes while removing duplicates.

Users can define `authorized_keys` to manage SSH public keys. The `authorized_keys_exclusive` variable defaults to `true` and controls whether unlisted keys are removed.

---

## Inventory Examples

### Global Users

```yaml
# group_vars/all.yml

common_users_global:
  - name: homelab
    uid: 1000
    sudo: true
```

### Group-Specific Users

```yaml
# group_vars/database.yml

common_users_group:
  - name: postgres
    uid: 101
    system: true
```

### Host-Specific Overrides

```yaml
# host_vars/db01.yml

common_users_host:
  - name: postgres
    uid: 999
```

---

## DNS Examples

### Configure Search Domains

```yaml
common_dns_search:
  - lab.local
  - corp.example.com
```

Resulting configuration:

```ini
[Resolve]
Domains=lab.local corp.example.com
```

---

## NFS Examples

### Mount an NFS Export

```yaml
common_nfs_mounts:
  - src: nas01:/exports/media
    dest: /mnt/media
    opts: rw,sync,hard,intr
    state: mounted
```

### Multiple NFS Mounts

```yaml
common_nfs_mounts:
  - src: nas01:/exports/media
    dest: /mnt/media

  - src: nas01:/exports/backups
    dest: /mnt/backups
```

---

## Local Disk Examples

### Format and Mount a Disk

```yaml
common_local_disks:
  - device: /dev/sdb
    fs_type: ext4
    mount_point: /mnt/data
    state: mounted
```

### Customize Mount Options

```yaml
common_local_disks:
  - device: /dev/sdb
    fs_type: ext4
    mount_point: /mnt/data
    mount_opts: defaults,noatime
    owner: root
    group: root
```

### Remove a Mount

```yaml
common_local_disks:
  - device: /dev/sdb
    mount_point: /mnt/data
    state: absent
```

### Remove a Mount and Delete the Directory

```yaml
common_local_disks:
  - device: /dev/sdb
    mount_point: /mnt/data
    state: absent
    purge: true
```

---

## Persisted User Metadata

After execution, the role writes the reconciled user definition to:

```text
/etc/ansible/common_users.yml
```

This file becomes the authoritative local user model for downstream roles.

### Example

```yaml
common_users:
  - name: homelab
    uid: 1000
    gid: 1000
    home: /home/homelab
    shell: /bin/bash
    sudo: true
    authorized_keys_exclusive: true
    authorized_keys:
      - ssh-ed25519 AAAA...
    tags:
      - dev

  - name: postgres
    uid: 999
    gid: 999
    home: /var/lib/postgresql
    shell: /usr/sbin/nologin
    sudo: false
    authorized_keys: []
    authorized_keys_exclusive: true
    system: true
    tags: []
```

---

## Downstream Role Integration

Roles that depend on user metadata should validate its existence before proceeding.

```yaml
- name: Verify common user metadata exists
  ansible.builtin.stat:
    path: /etc/ansible/common_users.yml
  register: common_users_file

- name: Load common user metadata
  ansible.builtin.include_vars:
    file: /etc/ansible/common_users.yml
  when: common_users_file.stat.exists

- name: Abort if metadata is missing
  ansible.builtin.fail:
    msg: >
      The common role must be executed before this role.
      Missing /etc/ansible/common_users.yml.
  when: not common_users_file.stat.exists
```

---

## Example Playbook

```yaml
- name: Baseline Configuration
  hosts: all
  become: true

  roles:
    - role: common
```

### Run System Updates

Updates are intentionally tagged `never` to prevent accidental execution.

Run updates explicitly:

```bash
ansible-playbook site.yml --tags update
```

### Install Common Packages

Define common_packages in inventory and run the package tasks explicitly:

```bash
ansible-playbook site.yml --tags packages
```

---

## Design Principles

* Inventory expresses desired state
* User definitions merge deterministically across inventory scopes
* Host-level values always take precedence
* Shared metadata is persisted for reuse and auditing
* Storage configuration is UUID-based wherever possible
* Optional subsystems can be independently enabled or disabled
* Avoid implicit Ansible behaviors that become problematic at scale

---

## Tags

| Tag | Description |
|------|-------------|
| `users` | User and SSH configuration |
| `timezone` | Timezone configuration |
| `dns` | DNS search-domain configuration |
| `nfs` | NFS client configuration and mounts |
| `disks` | Local disk formatting and mounting |
| `firewall` | Firewall configuration |
| `packages` | Install packages listed in common_packages |
| `update` | Package updates and reboot handling (must be explicitly tagged) |

---