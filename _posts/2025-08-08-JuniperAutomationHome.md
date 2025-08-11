---
title: "Juniper Network Automation: Ansible, PyEZ, JSNAPy, and SaltStack"
date: 2025-08-08 12:00:00 -500
categories: [JuniperAutomation, overview]
tags: [juniper, junos, automation, ansible, pyez, jsnapy, saltstack, netconf, devops]
---

# Juniper Network Automation: Ansible, PyEZ, JSNAPy, and SaltStack

In modern networking, automation is a requirement.  
Juniper’s ecosystem provides mature tools to reduce manual effort, minimize errors, validate intent, and accelerate delivery.

## Table of Contents
- [1. Ansible juniper.device Collection](#1-ansible-juniperdevice-collection)
- [2. PyEZ](#2-pyez)
- [3. JSNAPy](#3-jsnapy)
- [4. SaltStack](#4-saltstack)
- [5. Comparison Matrix](#5-comparison-matrix)
- [6. When to Use Which Tool](#6-when-to-use-which-tool)
- [7. Sample End-to-End Pipeline](#7-sample-end-to-end-pipeline)
- [8. Summary](#8-summary)

---

## 1. Ansible `juniper.device` Collection

Official Ansible collection for Junos (agentless, NETCONF or local execution).

Docs: https://ansible-juniper-collection.readthedocs.io/

### Key Capabilities
- Facts collection (system / RE / chassis)
- Config load (merge / replace / set), diff & commit
- Operational CLI commands
- NETCONF RPC execution
- JSNAPy test invocation
- Software image upgrades
- Local (on-box) PyEZ path or remote NETCONF

### Installation

```bash
ansible-galaxy collection install juniper.device
pip install junos-eznc ncclient
```

### Inventory (INI)

```ini
[junos_devices]
router1 ansible_host=192.0.2.1
router2 ansible_host=192.0.2.2

[junos_devices:vars]
ansible_user=netadmin
ansible_password=juniper123
ansible_connection=local
ansible_python_interpreter=/usr/bin/python3
```

### Example: Gather Facts

```yaml
---
- name: Gather Junos facts
  hosts: junos_devices
  gather_facts: no
  collections:
    - juniper.device
  tasks:
    - name: Retrieve facts
      juniper.device.facts:
      register: facts_output

    - name: Show facts
      debug:
        var: facts_output
```

### Notable Modules

| Module  | Purpose                                   |
|---------|-------------------------------------------|
| facts   | Gather system / RE / chassis facts        |
| config  | Load, diff, commit configurations         |
| command | Run CLI operational commands              |
| rpc     | Execute NETCONF RPCs                      |
| jsnapy  | Run JSNAPy validation suites              |
| software| Manage Junos OS upgrades                  |
| ping    | Test reachability from the device         |

---

## 2. PyEZ

Python library (junos-eznc) for programmatic Junos interaction; best for custom logic.

### Features
- Open NETCONF session (`Device`)
- Structured facts access
- Invoke RPCs (XML / JSON)
- Load / compare / commit configs (`Config`)
- Table/View abstraction for structured data
- Embed in Python services / CI tooling

### Install

```bash
pip install junos-eznc
```

### Example

```python
from jnpr.junos import Device
from jnpr.junos.utils.config import Config

dev = Device(host='192.0.2.1', user='netadmin', password='juniper123')
dev.open()
print(dev.facts)
with Config(dev, mode='exclusive') as cu:
    cu.load('set system host-name demo-router', format='set')
    if cu.diff():
        cu.commit()
dev.close()
```

---

## 3. JSNAPy

State validation & snapshot comparison framework.

### Use Cases
- Pre / post change validation
- Compliance & golden-state checks
- Regression testing
- Intent assurance

### Install

```bash
pip install jsnapy
```

### Workflow

1. Define tests (e.g. `test_bgp.yml`)
2. Pre-change snapshot:
   ```bash
   jsnapy --snap pre_change -f test_bgp.yml -t hosts.yml
   ```
3. Apply changes
4. Post-change snapshot + compare:
   ```bash
   jsnapy --check pre_change post_change -f test_bgp.yml -t hosts.yml
   ```

### Sample Test (Excerpt)

```yaml
tests:
  - test_bgp_neighbors:
      rpc: get-bgp-summary-information
      iterate:
        id: peer-address
        xpath: //bgp-peer
        tests:
          - is-equal: peer-state, Established
```

---

## 4. SaltStack

Event-driven automation & orchestration using Junos proxy minions.

### Strengths
- Real-time reactor (event -> action)
- Scales to many devices centrally
- Declarative state enforcement
- Extensible modules & returners

### Minimal Libraries

```bash
pip install junos-eznc jxmlease
```

### Example

```bash
salt 'router1' junos.cli "show version"
```

---

## 5. Comparison Matrix

| Aspect                | Ansible (juniper.device)        | PyEZ                      | JSNAPy                       | SaltStack                         |
|-----------------------|----------------------------------|---------------------------|------------------------------|-----------------------------------|
| Style                 | Declarative playbooks            | Imperative Python         | Validation snapshots         | Event / state driven              |
| Transport             | NETCONF / local                  | NETCONF                   | NETCONF                      | NETCONF via proxy                 |
| Best For              | Multi-step workflows             | Custom tooling            | Intent & state validation    | Real-time orchestration           |
| Learning Curve        | Low–Medium                       | Medium                    | Low                          | Medium–High                       |
| Diff Support          | Config diff built-in             | Manual (script)           | Snapshot comparison          | State reports                     |
| Multi-Vendor          | Yes                              | No                        | Limited (Junos focus)        | Limited (focus on managed types)  |
| Extensibility         | Roles / collections              | Full Python               | YAML tests                   | Modules / reactors / engines      |
| CI/CD Integration     | Common                           | Common                    | Validation stage             | Possible (extra setup)            |
| Event Handling        | Indirect                         | Scripted                  | None (batch)                 | Native                            |

---

## 6. When to Use Which Tool

| Goal                               | Recommended Tool            |
|-----------------------------------|-----------------------------|
| Fast repeatable config rollout    | Ansible                     |
| Deep custom device logic          | PyEZ                        |
| Pre/post change assurance         | JSNAPy                      |
| Event-triggered remediation       | SaltStack                   |
| Unified multi-vendor automation   | Ansible                     |
| Python service integration        | PyEZ                        |
| Compliance & drift detection      | JSNAPy (+ Ansible)          |
| Large scale realtime orchestration| SaltStack                   |

---

## 7. Sample End-to-End Pipeline

1. JSNAPy: pre-change validation  
2. Ansible: deploy config (juniper.device.config)  
3. JSNAPy: post-change comparison (intent confirmation)  
4. PyEZ: custom inventory export or complex logic  
5. SaltStack: reactor watches for anomalies & triggers rollback / alert  

---

## 8. Summary

Use each where it excels:
- Ansible: orchestration & standardized config
- PyEZ: granular programmable control
- JSNAPy: confidence & compliance validation
- SaltStack: scale & event-driven operations

Combined they form a robust Junos automation toolchain.