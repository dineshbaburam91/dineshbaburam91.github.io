---
title: "Ansible Junos RPC Module: Running RedHat junipernetworks.junos Modules in juniper.device Collection"
date: 2025-08-08 12:00:00 -500
categories: [JuniperAutomation, ansible, junos]
tags: [ansible, junos, juniper, automation, rpc, netconf, pyez, compatibility]
---

# Ansible Junos RPC Module: Running RedHat junipernetworks.junos Modules in juniper.device Collection

The `juniper.device` collection allows continued use of RedHat style module paths (`junipernetworks.junos.junos_rpc`). This enables existing playbooks to run without edits while new automation can adopt `juniper.device.rpc`. This post shows:

1. RedHat module path playbook (`junipernetworks.junos.junos_rpc`)
2. Native module path playbook (`juniper.device.rpc`)
3. Inventory patterns (NETCONF and PyEZ)
4. RPC usage examples
5. Troubleshooting and gradual adoption tips

---

## 1. RedHat Module Path Playbook (Unmodified)

### Inventory (NETCONF Transport)

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

### Playbook

```yaml
---
- name: Run RPC using RedHat module path
  hosts: junos
  gather_facts: false
  connection: ansible.netcommon.netconf
  collections:
    - junipernetworks.junos

  tasks:
    - name: Get system information
      junipernetworks.junos.junos_rpc:
        rpc: get-system-information
      register: sysinfo

    - name: Show RPC result
      debug:
        var: sysinfo.xml
```

### Run

```bash
ansible-playbook -i inventory_rh pb.redhat_rpc.yml
```

---

## 2. Native Module Path Playbook (`juniper.device.rpc`)

### Inventory (PyEZ Transport Example)

```ini
[junos]
device1 ansible_host=x.x.x.x ansible_user=<user> ansible_ssh_pass=<pass> ansible_port=22 ansible_connection=juniper.device.pyez ansible_command_timeout=300

[all:vars]
ansible_python_interpreter=<python interpreter path>
```

### Playbook

```yaml
---
- name: Run RPCs using native module path
  hosts: junos
  gather_facts: false
  collections:
    - juniper.device

  tasks:
    - name: Execute single RPC
      juniper.device.rpc:
        rpcs:
          - get-system-information
      register: rpc_single

    - debug:
        var: rpc_single.stdout

    - name: Execute multiple RPCs
      juniper.device.rpc:
        rpcs:
          - get-system-information
          - get-chassis-inventory
          - get-interface-information terse
        format: xml
      register: rpc_multi

    - debug:
        var: rpc_multi
```

### Run

```bash
ansible-playbook -i inventory_native pb.native_rpc.yml
```

---

## 3. Repository Layout Example

```
_playbooks/
  pb.redhat_rpc.yml
  pb.native_rpc.yml
inventory_rh
inventory_native
```

---

## 4. RPC Naming Patterns

Correct (hyphenated, lower-case):
```
get-system-information
get-chassis-inventory
get-interface-information
get-interface-information terse
```

Avoid:
```
get_system_information
Get-System-Information
```

---

## 5. Troubleshooting

| Symptom | Check | Fix |
|---------|-------|-----|
| ConnectionError get_capabilities | NETCONF enabled | set system services netconf ssh |
| Timeout on large RPC | Long inventory/interface queries | Increase ansible_command_timeout |
| Empty output | RPC spelling or permissions | Verify RPC name (show rpc) |
| Warning: NoneType .get | Loader quirk | Ignore if task succeeds |
| Auth failure | Credentials / SSH reachability | ssh -s netconf <device-ip> |

### Quick Diagnostics

```bash
ssh -s netconf <device-ip>
nc -vz <device-ip> 830
```

Increase timeout (inventory example):
```ini
ansible_command_timeout=300
```

---

## 6. Summary

- RedHat `junipernetworks.junos.junos_rpc` modules run unchanged under the `juniper.device` collection.
- Native `juniper.device.rpc` offers multi-RPC execution and PyEZ transport use.
- Both paths can coexist; adopt native naming for new automation while keeping existing playbooks operational.
