---
title: "Ansible Junos Configuration: RedHat junipernetworks.junos Logging vs juniper.device Config"
date: 2025-08-10 12:00:00 -500
categories: [JuniperAutomation, ansible, junos]
tags: [ansible, junos, juniper, automation, logging, config, netconf, pyez, rollback, diff, compatibility]
---

# Ansible Junos Configuration: RedHat junipernetworks.junos Logging vs juniper.device Config

The `juniper.device` collection allows continued use of RedHat style logging module paths (`junipernetworks.junos.junos_logging_global`) while new playbooks can use the native configuration module `juniper.device.config`.  
This post shows:
1. RedHat logging playbook (junos_logging_global states: merged / replaced / overridden / gathered / rendered / deleted)
2. Execute RedHat logging module via `juniper.device` namespace
3. Native configuration playbook (merge / replace / commit / check / rollback / retrieve)
4. Inventory examples (NETCONF vs PyEZ)
5. Troubleshooting
6. Summary

---

## 1. RedHat Logging Playbook (junos_logging_global)

### Inventory (NETCONF)

```ini
[junos]
<Device IP>

[junos:vars]
ansible_network_os=juniper.device.junos
ansible_ssh_user=<username>
ansible_ssh_pass=<password>
ansible_port=22
ansible_connection=ansible.netcommon.netconf

[all:vars]
ansible_python_interpreter=<python interpreter path>
```

### Playbook (Logging Lifecycle)

```yaml
---
- name: Manage system logging (RedHat module path)
  hosts: junos
  gather_facts: false
  connection: ansible.netcommon.netconf
  collections:
    - junipernetworks.junos

  vars:
    ri_name: "LOG-INST-A"
    src_addr: "192.0.2.10"
    host_addr: "192.0.2.200"

  tasks:
    - name: Ensure routing instance exists (prereq)
      junipernetworks.junos.junos_config:
        lines:
          - "set routing-instances {{ ri_name }} instance-type virtual-router"
        update: merge
        comment: "Create routing instance for syslog host"

    - name: MERGED - Full logging configuration
      junipernetworks.junos.junos_logging_global:
        state: merged
        config:
          allow_duplicates: true
          source_address: "{{ src_addr }}"
          routing_instance: "{{ ri_name }}"
          log_rotate_frequency: 30
          files:
            - name: "file_main"
              any: { level: "info" }
            - name: "file_audit"
              authorization: { level: "any" }
          hosts:
            - name: "loghost1"
              source_address: "{{ host_addr }}"
              routing_instance: "{{ ri_name }}"
              any: { level: "any" }
          users:
            - name: "user1"
              any: { level: "any" }

    - name: REPLACED - Replace subset
      junipernetworks.junos.junos_logging_global:
        state: replaced
        config:
          files:
            - name: "file_main"
              any: { level: "notice" }
          users:
            - name: "user1"
              any: { level: "any" }

    - name: OVERRIDDEN - Override full logging stanza
      junipernetworks.junos.junos_logging_global:
        state: overridden
        config:
          files:
            - name: "file_override"
              any: { level: "warning" }
          users:
            - name: "user1"

    - name: GATHERED - Get current logging state
      junipernetworks.junos.junos_logging_global:
        state: gathered
      register: gathered_logging

    - debug:
        var: gathered_logging.gathered

    - name: RENDERED - Convert gathered to XML form
      junipernetworks.junos.junos_logging_global:
        state: rendered
        config: "{{ gathered_logging.gathered }}"
      register: rendered_logging

    - debug:
        var: rendered_logging.rendered

    - name: DELETED - Remove all logging config
      junipernetworks.junos.junos_logging_global:
        state: deleted
        config: {}
```

### Run

```bash
ansible-playbook -i inventory_rh pb.redhat_logging.yml -vv
```

---

## 2. Execute RedHat Logging Using juniper.device Namespace

You can invoke the same RedHat logging module through the `juniper.device` namespace (`juniper.device.junos_logging_global`) without changing task logic.

### Playbook (Namespace Compatibility)

```yaml
---
- name: Manage logging via juniper.device namespace (compatibility)
  hosts: junos
  gather_facts: false
  connection: ansible.netcommon.netconf
  collections:
    - juniper.device

  vars:
    ri_name: "LOG-INST-A2"
    src_addr: "192.0.2.20"

  tasks:
    - name: MERGED - Minimal logging (compat call)
      juniper.device.junos_logging_global:
        state: merged
        config:
          source_address: "{{ src_addr }}"
          routing_instance: "{{ ri_name }}"
          files:
            - name: "compat_file"
              any: { level: "info" }
      register: compat_merge

    - debug:
        var: compat_merge.changed

    - name: GATHERED - Show current logging (compat call)
      juniper.device.junos_logging_global:
        state: gathered
      register: compat_gather

    - debug:
        var: compat_gather.gathered
```

### Run

```bash
ansible-playbook -i inventory_rh pb.compat_logging_namespace.yml -vv
```

---

## 3. Native Configuration Playbook (juniper.device.config)

### Inventory (PyEZ Transport)

```ini
[junos]
cfg_target ansible_host=x.x.x.x ansible_user=<user> ansible_ssh_pass=<pass> ansible_port=22 \
ansible_connection=juniper.device.pyez ansible_command_timeout=300

[all:vars]
ansible_python_interpreter=<python interpreter path>
```

(Local/on-box alternative)
```ini
[junos]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
```

### Playbook (Config Workflow)

```yaml
---
- name: Native configuration operations
  hosts: junos
  gather_facts: false
  collections:
    - juniper.device

  vars:
    new_hostname: "EDGE-AUTO-1"

  tasks:
    - name: Merge hostname
      juniper.device.config:
        load: merge
        lines:
          - "set system host-name {{ new_hostname }}"
        comment: "Set hostname"
      register: host_merge

    - debug: { var: host_merge.diff }

    - name: Merge services from file
      juniper.device.config:
        load: merge
        src: files/system_services.conf
        format: text
        comment: "Enable services"
      register: services_merge

    - debug: { var: services_merge.diff }

    - name: Replace banner subtree
      juniper.device.config:
        load: replace
        lines:
          - 'system { login { message "Authorized Access Only"; } }'
        replace: "system login"
        comment: "Replace banner"
      register: banner_replace

    - debug: { var: banner_replace.diff }

    - name: Commit changes
      juniper.device.config:
        commit: true
        comment: "Standard commit"
      register: commit_result

    - debug: { var: commit_result }

    - name: Retrieve committed config (set format)
      juniper.device.config:
        retrieve: committed
        format: set
      register: committed_set

    - debug: { var: committed_set.config }

    - name: Check (no commit) extra login option
      juniper.device.config:
        load: merge
        lines:
          - "set system login retry-options tries-before-disconnect 3"
        check: true
      register: check_only

    - debug: { var: check_only.diff }

    - name: Rollback last commit (example)
      juniper.device.config:
        rollback: 1
        commit: true
        comment: "Rollback last change"
      when: host_merge.changed or banner_replace.changed
      register: rollback_result

    - debug: { var: rollback_result }
      when: rollback_result is defined
```

### Run

```bash
ansible-playbook -i inventory_native pb.native_config.yml -vv
```

---

## 4. Directory Structure Example

```
_playbooks/
  pb.redhat_logging.yml
  pb.compat_logging_namespace.yml
  pb.native_config.yml
files/
  system_services.conf
inventory_rh
inventory_native
```

Sample system_services.conf:
```
system {
  services {
    netconf { ssh; }
  }
}
```

---

## 5. Troubleshooting

| Issue | Check | Fix |
|-------|-------|-----|
| NETCONF errors | Service enabled? Port 830 reachable? | set system services netconf ssh |
| No diff | Lines identical? | Adjust lines / add -vv |
| Rollback fails | Rollback file exists? | show system rollback |
| PyEZ timeout | ansible_command_timeout too low | Increase to 300+ |
| Logging states unexpected | State sequence correct? | Use gathered to inspect |

Quick tests:
```bash
ssh -s netconf <device-ip>
nc -vz <device-ip> 830
show system rollback
```

---

## 6. Summary

- RedHat logging module (`junipernetworks.junos.junos_logging_global`) runs unchanged.
- The same module can be executed via `juniper.device.junos_logging_global` namespace.
- Native `juniper.device.config` handles general configuration workflows (merge, replace, check, commit, retrieve, rollback).
- Use all side by side; migrate gradually