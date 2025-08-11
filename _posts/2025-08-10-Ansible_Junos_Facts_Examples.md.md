---
title: "Ansible Junos Facts Module: RedHat vs Native Collection Examples"
date: 2025-08-10 14:00:00 -500
categories: [JuniperAutomation, ansible, junos]
tags: [ansible, junos, juniper, automation, facts, netconf, pyez]
---

# Ansible Junos Facts Module: RedHat vs Native Collection Examples

This guide demonstrates how to execute Junos facts collection using both the mergered RedHat Ansible `juniper.device.junos_facts` module and the native `juniper.device.facts` module from the Juniper device collections.

## RedHat Ansible junos_facts via juniper.device Collections

### Step 1: Create the Inventory File

Create an inventory file with NETCONF connection parameters:

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

### Step 2: Write the Playbook for Junos Facts

Create a playbook to gather facts using the RedHat collection:

```yaml
---
- name: Functional Test - junos_facts module
  hosts: all
  gather_facts: false
  connection: netconf
  collections:
    - junipernetworks.junos
 
  tasks:
    - name: Collect default set of facts (min)
      junipernetworks.junos.junos_facts:
      register: default_facts
 
    - name: Print the facts
      debug:
        var: default_facts
```

### Step 3: Execute the Playbook

Run the playbook with the following command:

```bash
ansible-playbook -i inventory_rh pb.juniper_junos_facts.yml
```

### Expected Output

```bash
[WARNING]: errors were encountered during the plugin load for junipernetworks.junos.junos_facts: ["'NoneType' object has no attribute 'get'"]

PLAY [Functional Test - junos_facts module] **************************************************************************************

TASK [Collect default set of facts (min)] ****************************************************************************************
ok: [10.220.17.17]

TASK [Print the facts] ***********************************************************************************************************
ok: [10.220.17.17] => {
    "default_facts": {
        "ansible_facts": {
            "ansible_net_api": "netconf",
            "ansible_net_gather_network_resources": [],
            "ansible_net_gather_subset": ["default"],
            "ansible_net_hostname": "evoeventtestb",
            "ansible_net_model": "mx960",
            "ansible_net_python_version": "3.13.5",
            "ansible_net_serialnum": "VM685037CBB2",
            "ansible_net_system": "junos",
            "ansible_net_version": "25.4I-20250615_dev_common.0.1157",
            "ansible_network_resources": {}
        },
        "changed": false,
        "failed": false
    }
}

PLAY RECAP ***********************************************************************************************************************
10.220.17.17               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

##  Native Junos Facts via juniper.device Collection

### Step 1: Create the Inventory File

Create an inventory file using the native PyEZ connection:

```ini
[junos]
pyez_connection_testcases  ansible_host=x.x.x.x  ansible_user=xxx  ansible_ssh_pass=xxxx ansible_port=22 ansible_connection=juniper.device.pyez ansible_command_timeout=300

[all:vars]
ansible_python_interpreter=<python interpreter path>
```

### Step 2: Write the Playbook for Native Junos Facts

Create a playbook using the native Juniper device collection:

```yaml
---
- name: Test juniper.device.facts module
  hosts: all
  gather_facts: false
  tasks:
    - name: "Gather Facts"
      juniper.device.facts:
      ignore_errors: true
      register: default_facts 

    - name: Check TEST 1
      debug:
        var: default_facts
```

### Step 3: Execute the Native Playbook

Run the native facts playbook:

```bash
ansible-playbook -i inventory pb.juniper_junos_native_facts.yml
```

### Expected Output (Detailed)

The native collection provides much more detailed information:

```bash
PLAY [Test juniper.device.facts module] ******************************************************************************************

TASK [Gather Facts] **************************************************************************************************************
ok: [pyez_connection_testcases]

TASK [Check TEST 1] **************************************************************************************************************
ok: [pyez_connection_testcases] => {
    "default_facts": {
        "ansible_facts": {
            "junos": {
                "HOME": "/root",
                "RE0": {
                    "last_reboot_reason": "Router rebooted after a normal shutdown.",
                    "mastership_state": "master",
                    "model": "RE-VMX",
                    "status": "OK",
                    "up_time": "16 days, 22 hours, 21 minutes, 35 seconds"
                },
                "RE1": {
                    "last_reboot_reason": "Router rebooted after a normal shutdown.",
                    "mastership_state": "backup", 
                    "model": "RE-VMX",
                    "status": "OK",
                    "up_time": "16 days, 22 hours, 21 minutes, 17 seconds"
                },
                "hostname": "evoeventtestb",
                "model": "MX960",
                "serialnumber": "VM685037CBB2",
                "version": "25.4I-20250615_dev_common.0.1157"
                // ... additional detailed facts
            }
        },
        "changed": false,
        "failed": false
    }
}
```

## Troubleshooting

### Common Warning Messages
The warning `"'NoneType' object has no attribute 'get'"` in the RedHat collection is typically harmless and doesn't affect functionality.

### Connection Issues
- Ensure NETCONF is enabled on the target device
- Verify network connectivity on port 830 (NETCONF) or 22 (SSH)
- Check authentication credentials
- Confirm Python interpreter path is correct

Both methods successfully gather device facts, but the native collection provides significantly more detailed and Junos-specific information, making it ideal for comprehensive device management and monitoring tasks.