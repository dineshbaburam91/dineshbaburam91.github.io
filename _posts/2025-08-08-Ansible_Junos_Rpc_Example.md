---
title: "Ansible Junos RPC Module: Running RedHat junipernetworks.junos Modules in juniper.device Collection"
date: 2025-08-10 12:00:00 -500
categories: [JuniperAutomation, ansible, junos]
tags: [ansible, junos, juniper, automation, rpc, netconf, pyez, compatibility]
---

# Ansible Junos RPC Module: Running RedHat junipernetworks.junos Modules in juniper.device Collection

The `juniper.device` collection allows continued use of RedHat style module paths (`junipernetworks.junos.junos_rpc`). Existing playbooks run unmodified while new automation may adopt `juniper.device.rpc`. You can also call the RedHat RPC task using the `juniper.device` namespace (`juniper.device.junos_rpc`). This post shows:

1. RedHat module path playbook (`junipernetworks.junos.junos_rpc`)
2. Native module path playbook (`juniper.device.rpc`)
3. Executing RedHat RPC via juniper.device namespace
4. Inventory patterns (NETCONF / PyEZ / local)
5. RPC naming patterns
6. RPC usage cheat sheet
7. Troubleshooting
8. Summary

---

## 1. RedHat Module Path Playbook (Unchanged)

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

### Sample Output (Excerpt)

```
<rpc-reply>
  <system-information>...</system-information>
</rpc-reply>
```

---

## 2. Native Module Path Playbook (`juniper.device.rpc`)

### Inventory (PyEZ Transport)

```ini
[junos]
pyez_rpc ansible_host=x.x.x.x ansible_user=<user> ansible_ssh_pass=<pass> ansible_port=22 \
ansible_connection=juniper.device.pyez ansible_command_timeout=300

[all:vars]
ansible_python_interpreter=<python interpreter path>
```

(Local/on-box)
```ini
[junos]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
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

    - name: Show single RPC
      debug:
        var: rpc_single.stdout

    - name: Execute multiple RPCs
      juniper.device.rpc:
        rpcs:
          - get-system-information
          - get-chassis-inventory
          - get-interface-information terse
        format: xml
      register: rpc_multi

    - name: Show multi RPC output
      debug:
        var: rpc_multi.stdout
```

### Run

```bash
ansible-playbook -i inventory_native pb.native_rpc.yml
```

### Sample Output (Excerpt)

```
stdout[0]: <system-information>...</system-information>
stdout[1]: <chassis-inventory>...</chassis-inventory>
stdout[2]: <interface-information>...</interface-information>
```

---

## 3. Executing RedHat RPC Using juniper.device Namespace

You can invoke the RedHat RPC module through `juniper.device.junos_rpc`.

```yaml
---
- name: Run RedHat RPC via juniper.device namespace
  hosts: junos
  gather_facts: false
  connection: ansible.netcommon.netconf
  collections:
    - juniper.device

  tasks:
    - name: Get system information
      juniper.device.junos_rpc:
        rpc: get-system-information
      register: compat_rpc

    - debug:
        var: compat_rpc.xml
```

---

## 4. Directory Structure Example

```
_playbooks/
  pb.redhat_rpc.yml
  pb.native_rpc.yml
  pb.compat_namespace_rpc.yml
inventory_rh
inventory_native
```

---

## 5. RPC Naming Patterns

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

## 6. RPC Usage Cheat Sheet

| Goal | RedHat Path | Native Path |
|------|-------------|-------------|
| Single RPC | rpc: get-system-information | rpcs: [get-system-information] |
| Multiple RPCs | One task per RPC | rpcs: [list, of, rpcs] |
| XML output | (default) | format: xml |
| Basic text (CLI alt) | Use junos_command | N/A (rpc is NETCONF) |
| Increase timeout | ansible_command_timeout in inventory | same |
| Namespace compatibility | junipernetworks.junos.junos_rpc | juniper.device.junos_rpc |

Common return fields:
- RedHat path: xml (full <rpc-reply>), changed (False), failed
- Native path: stdout (list of inner XML blocks), rpcs (list), changed (False), failed

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| ConnectionError get_capabilities | NETCONF disabled | set system services netconf ssh |
| Empty stdout/xml | Misspelled RPC | Verify with show rpc |
| Timeout on inventory RPC | Large chassis / many FPCs | Increase ansible_command_timeout |
| Warning: NoneType .get | Loader quirk | Ignore if task succeeds |
| Auth failure | Bad credentials / SSH | ssh -s netconf <device-ip> |
| Multi-RPC fails mid-run | One RPC invalid | Validate list individually |

Quick diagnostics:

```bash
ssh -s netconf <device-ip>
nc -vz <device-ip> 830
```

Inventory timeout example:
```ini
ansible_command_timeout=300
```

---

## 8. Summary

- RedHat `junipernetworks.junos.junos_rpc` runs unchanged within the `juniper.device` collection.
- Native `juniper.device.rpc` supports multiple RPCs per task and cleaner inner XML output.
- You can also call the RedHat RPC via `juniper.device.junos_rpc` namespace.
- Both approaches coexist; adopt native for new automation while keeping existing playbooks intact.
