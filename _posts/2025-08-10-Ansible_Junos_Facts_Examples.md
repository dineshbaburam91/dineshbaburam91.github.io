---
title: "Ansible Junos Facts Module: Running RedHat junipernetworks.junos Modules in juniper.device Collection"
date: 2025-08-10 14:00:00 -500
categories: [JuniperAutomation, ansible, junos]
tags: [ansible, junos, juniper, automation, facts, netconf, pyez, compatibility]
---

# Ansible Junos Facts Module: Running RedHat junipernetworks.junos Modules in juniper.device Collection

The `juniper.device` collection allows continued use of RedHat style module paths (`junipernetworks.junos.junos_facts`). Existing playbooks run unmodified while new automation may adopt `juniper.device.facts`. This post shows:

1. RedHat module path playbook (`junipernetworks.junos.junos_facts`)
2. Native module path playbook (`juniper.device.facts`)
3. Inventory patterns (NETCONF and PyEZ)
4. Returned data fields
5. Troubleshooting and adoption tips

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
- name: Gather Junos facts (RedHat module path)
  hosts: junos
  gather_facts: false
  connection: ansible.netcommon.netconf
  collections:
    - junipernetworks.junos

  tasks:
    - name: Collect default subset
      junipernetworks.junos.junos_facts:
      register: rh_facts

    - name: Show facts
      debug:
        var: rh_facts
```

### Run

```bash
ansible-playbook -i inventory_rh pb.redhat_facts.yml
```

### Sample Output (Excerpt)

```
ansible_net_hostname: evoeventtestb
ansible_net_model: mx960
ansible_net_system: junos
ansible_net_version: 25.4I-20250615_dev_common.0.1157
```

---

## 2. Native Module Path Playbook (`juniper.device.facts`)

### Inventory (PyEZ Transport)

```ini
[junos]
pyez_facts ansible_host=x.x.x.x ansible_user=<user> ansible_ssh_pass=<pass> ansible_port=22 \
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
- name: Gather Junos facts (native module path)
  hosts: junos
  gather_facts: false
  collections:
    - juniper.device

  tasks:
    - name: Run native facts
      juniper.device.facts:
      register: native_facts

    - name: Show native facts
      debug:
        var: native_facts.ansible_facts.junos
```

### Run

```bash
ansible-playbook -i inventory_native pb.native_facts.yml
```

### Sample Output (Excerpt)

```
hostname: evoeventtestb
model: MX960
serialnumber: VM685037CBB2
version: 25.4I-20250615_dev_common.0.1157
RE0.mastership_state: master
RE1.mastership_state: backup
```

---

## 3. Directory Structure Example

```
_playbooks/
  pb.redhat_facts.yml
  pb.native_facts.yml
inventory_rh
inventory_native
```

---

## 4. Returned Data Fields

| RedHat Path Key (ansible_facts) | Native Path Key (ansible_facts.junos) | Notes |
|---------------------------------|----------------------------------------|-------|
| ansible_net_hostname            | hostname                               | Device host-name |
| ansible_net_model               | model                                  | Chassis model |
| ansible_net_version             | version                                | Junos version |
| ansible_net_serialnum           | serialnumber                           | Serial |
| (not present)                   | RE0 / RE1 maps                         | Routing engine details |
| (not present)                   | up_time                                | Human readable uptime |
| ansible_net_system              | (implicit junos)                       | OS identifier |

Native output contains deeper per-RE and platform metadata.

---

## 5. Facts Cheat Sheet (Native)

| Goal | Approach |
|------|----------|
| Basic platform data | Use task with no params |
| Access hostname | native_facts.ansible_facts.junos.hostname |
| RE status | native_facts.ansible_facts.junos.RE0.status |
| Serial number | native_facts.ansible_facts.junos.serialnumber |
| Conditional logic | when: native_facts.ansible_facts.junos.RE0.mastership_state == 'master' |

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Warning: NoneType .get | Loader quirk in RedHat path | Ignore if task succeeds |
| Connection failure | NETCONF disabled | set system services netconf ssh |
| Timeout | Large device / slow RE | Increase ansible_command_timeout |
| Missing RE data (native) | Single RE platform | Normal |
| Empty facts | Auth / privilege issue | Validate credentials via NETCONF |

Quick tests:

```bash
ssh -s netconf <device-ip>
nc -vz <device-ip> 830
```

---

## 7. Summary

- RedHat `junipernetworks.junos.junos_facts` module path works unchanged inside `juniper.device`.
- Native `juniper.device.facts` provides extended Junos-specific details (RE, uptime, richer keys).
- Run both side by side; adopt native path for new automation without breaking