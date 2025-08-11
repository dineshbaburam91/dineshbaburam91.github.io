---
title: "Ansible Junos RPC Module: RedHat vs Native Collection Examples"
date: 2025-08-08 12:00:00 -500
categories: [JuniperAutomation, ansible, junos]
tags: [ansible, junos, juniper, automation, rpc, netconf, pyez]
---

# Ansible Junos RPC Module: RedHat vs Native Collection Examples

This guide demonstrates how to execute Junos RPC commands using both the RedHat Ansible `junipernetworks.junos.junos_rpc` module and the native `juniper.device.rpc` module from the Juniper device collections.

## Method 1: RedHat Ansible junos_rpc via juniper.device Collections

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

### Step 2: Write the Playbook for Junos RPC

Create a playbook to execute RPC commands using the RedHat collection:

```yaml
---
- name: Functional Test - junos_rpc module
  hosts: all
  gather_facts: false
  connection: ansible.netcommon.netconf
  collections:
    - junipernetworks.junos
 
  tasks:
    - name: Get system information using RPC
      junipernetworks.junos.junos_rpc:
        rpc: get-system-information
      register: system_info
 
    - name: Assert system information RPC returned XML
      debug:
        var: system_info
```

### Step 3: Execute the Playbook

Run the playbook with the following command:

```bash
ansible-playbook -i inventory_rh pb.juniper_junos_rh_rpc.yml
```

### Expected Output

```bash
[WARNING]: errors were encountered during the plugin load for junipernetworks.junos.junos_rpc: ["'NoneType' object has no attribute 'get'"]
[WARNING]: errors were encountered during the plugin load for debug: ["'NoneType' object has no attribute 'get'"]

PLAY [Functional Test - junos_rpc module] ****************************************************************************************

TASK [Get system information using RPC] ******************************************************************************************
[WARNING]: errors were encountered during the plugin load for junipernetworks.junos.junos: ["'NoneType' object has no attribute 'get'"]
ok: [10.220.17.17]

TASK [Assert system information RPC returned XML] ********************************************************************************
ok: [10.220.17.17] => {
    "system_info": {
        "changed": false,
        "failed": false,
        "output": [
            "<rpc-reply message-id=\"urn:uuid:418b4192-936d-47c7-8bda-db7b7156d8cc\"><system-information><host-name>evoeventtestb</host-name><hardware-model>mx960</hardware-model><os-name>junos</os-name><os-version>25.4I-20250615_dev_common.0.1157</os-version><serial-number>VM685037CBB2</serial-number></system-information></rpc-reply>"
        ],
        "xml": "<rpc-reply message-id=\"urn:uuid:418b4192-936d-47c7-8bda-db7b7156d8cc\"><system-information><host-name>evoeventtestb</host-name><hardware-model>mx960</hardware-model><os-name>junos</os-name><os-version>25.4I-20250615_dev_common.0.1157</os-version><serial-number>VM685037CBB2</serial-number></system-information></rpc-reply>"
    }
}

PLAY RECAP ***********************************************************************************************************************
10.220.17.17               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Method 2: Native Junos RPC via juniper.device Collection

### Step 1: Create the Inventory File

Create an inventory file using the native PyEZ connection:

```ini
[junos]
pyez_connection_testcases  ansible_host=x.x.x.x  ansible_user=xxx  ansible_ssh_pass=xxxx ansible_port=22 ansible_connection=juniper.device.pyez ansible_command_timeout=300

[all:vars]
ansible_python_interpreter=<python interpreter path>
```

### Step 2: Write the Playbook for Native Junos RPC

Create a playbook using the native Juniper device collection:

```yaml
---
- name: juniper.device.rpc module
  hosts: all
  gather_facts: false

  tasks:
    - name: "Execute single RPC get-system-information without any kwargs"
      juniper.device.rpc:
        rpcs:
          - "get-system-information"
      register: rpc_output 
      ignore_errors: true
      tags: [test1]

    - name: Check RPC output 
      debug:
        var: rpc_output.stdout
```

### Step 3: Execute the Native Playbook

Run the native RPC playbook:

```bash
ansible-playbook -i inventory pb.juniper_junos_native_rpc.yml
```

### Expected Output (Cleaner XML)

```bash
PLAY [juniper.device.rpc module] *************************************************************************************************

TASK [Execute single RPC get-system-information without any kwargs] ************************************************************
ok: [pyez_connection_testcases]

TASK [Check RPC output] **********************************************************************************************************
ok: [pyez_connection_testcases] => {
    "rpc_output.stdout": "<system-information>\n  <host-name>evoeventtestb</host-name>\n  <hardware-model>mx960</hardware-model>\n  <os-name>junos</os-name>\n  <os-version>25.4I-20250615_dev_common.0.1157</os-version>\n  <serial-number>VM685037CBB2</serial-number>\n</system-information>\n"
}

PLAY RECAP ***********************************************************************************************************************
pyez_connection_testcases : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Comparison: RedHat vs Native RPC Collections

| Feature | RedHat Collection | Native Collection |
|---------|------------------|-------------------|
| **Connection Type** | `ansible.netcommon.netconf` | `juniper.device.pyez` |
| **RPC Syntax** | Single RPC string | List of RPCs |
| **Output Format** | Full RPC reply with metadata | Clean XML output |
| **Multiple RPCs** | One per task | Multiple in single task |
| **Error Handling** | Standard Ansible | Enhanced PyEZ handling |
| **Performance** | Standard NETCONF | Optimized PyEZ |

## Key Differences

### RedHat Collection Output
- **Full RPC Reply**: Includes `<rpc-reply>` wrapper and message IDs
- **Both formats**: Provides both `output` array and `xml` string
- **Standard structure**: Follows Ansible network module patterns

### Native Collection Output
- **Clean XML**: Direct system information without RPC wrapper
- **Formatted output**: Properly indented XML structure
- **Multiple RPC support**: Can execute multiple RPCs in single task
- **Streamlined**: Only essential data in response

## Advanced RPC Examples

### Multiple RPCs with Native Collection

```yaml
---
- name: Execute multiple RPCs
  hosts: all
  gather_facts: false
  
  tasks:
    - name: "Execute multiple RPCs"
      juniper.device.rpc:
        rpcs:
          - "get-system-information"
          - "get-chassis-inventory"
          - "get-interface-information terse"
        format: "xml"
      register: multi_rpc_output

    - name: Display results
      debug:
        var: multi_rpc_output
```

### RPC with Arguments (RedHat Collection)

```yaml
---
- name: RPC with arguments
  hosts: all
  gather_facts: false
  connection: ansible.netcommon.netconf
  collections:
    - junipernetworks.junos
  
  tasks:
    - name: Get interface information with arguments
      junipernetworks.junos.junos_rpc:
        rpc: get-interface-information
        args:
          interface-name: ge-0/0/0
          terse: true
      register: interface_info

    - name: Display interface info
      debug:
        var: interface_info
```

## Best Practices

### When to Use RedHat Collection
- **Multi-vendor environments** where consistency is important
- **Standard Ansible workflows** that require uniform module behavior
- **Simple RPC calls** with basic requirements
- **Integration with existing** Red Hat Ansible workflows

### When to Use Native Collection
- **Juniper-only environments** where optimization matters
- **Complex RPC operations** requiring multiple calls
- **Performance-critical** automation tasks
- **Advanced PyEZ features** and error handling

## Troubleshooting

### Common Warning Messages
The warning `"'NoneType' object has no attribute 'get'"` in the RedHat collection is typically harmless and doesn't affect RPC execution.

### RPC Execution Issues

#### 1. Invalid RPC Names
```bash
# Correct RPC format
get-system-information
get-chassis-inventory

# Incorrect format (avoid hyphens in wrong places)
get_system_information
```

#### 2. Connection Problems
- Verify NETCONF is enabled: `set system services netconf ssh`
- Check network connectivity on port 830
- Ensure proper authentication credentials

#### 3. Timeout Issues
```yaml
# Increase timeout for long-running RPCs
ansible_command_timeout: 300
```


