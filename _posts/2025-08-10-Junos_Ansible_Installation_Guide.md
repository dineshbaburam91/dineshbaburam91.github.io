---
title: Junos Ansible Installation Guide to run junipernetworks.junos Modules in juniper.device Collection
date: 2025-08-09 12:00:00 -500
categories: [JuniperAutomation, junos, ansible]
tags: [junos, juniper, automation, ansible, installation, guide]
---

# Junos Ansible Installation Guide to run junipernetworks.junos Modules in juniper.device Collection

This guide walks you through the installation process for Junos Ansible modules using the merged collection approach.

## Prerequisites

Before starting the installation, ensure you have the following components installed:

- **Python 3.12 or later**
- **Ansible** - Automation platform
- **junos-eznc >= 2.6.0** - Python library for Junos automation
- **jsnapy >= 1.3.6** - Junos Snapshot Administrator in Python
- **jxmlease** - XML parsing library
- **xmltodict** - XML to dictionary converter
- **looseversion** - Version comparison utility

## Installation Steps

### 1. Create the Virtual Environment

Create a dedicated virtual environment for the Ansible modules:

```bash
mkdir ansible_modules_redirection_ansible_merger_branch_venv
cd ansible_modules_redirection_ansible_merger_branch_venv
python3.13 -m venv venv
source venv/bin/activate
```

### 2. Clone the JNPR Ansible Merger Repository

Clone the repository with the merger branch:

```bash
git clone https://github.com/Juniper/ansible-junos-stdlib.git -b jnpr-ansible-merger
```

### 3. Install the Merged Collection

Install the merged collection using ansible-galaxy:

```bash
ansible-galaxy collection install git+https://github.com/Juniper/ansible-junos-stdlib.git#/ansible_collections/juniper/device,jnpr-ansible-merger
```

### 4. Install the JuniperNetworks.junos Collection

Install the official JuniperNetworks collection:

```bash
ansible-galaxy collection install junipernetworks.junos
```

### 5. Remove Conflicting Plugins

Remove the conflicting plugins directory:

```bash
rm -rf /root/.ansible/collections/ansible_collections/junipernetworks/junos/plugins
```

### 6. Update the JuniperNetworks.junos Collection Meta Runtime

Follow these steps to update the runtime configuration:

1. **Remove existing content** from the runtime.yaml file located at:
   ```
   /root/.ansible/collections/ansible_collections/junipernetworks/junos/meta/runtime.yml
   ```

2. **Add the new content** from the merger branch runtime.yml:
   - Source: [runtime.yml from jnpr-ansible-merger branch](https://github.com/Juniper/ansible-junos-stdlib/blob/jnpr-ansible-merger/ansible_collections/junipernetworks/junos/meta/runtime.yml)

> **Note:** This step is not required post merger.

### 7. Execute Your Playbooks

You can now execute your Ansible playbooks with the installed collections.

## Troubleshooting

If you encounter any issues during installation, ensure that:

- **Python Version**: Your Python version is 3.12 or later
- **Network Connectivity**: You have proper network connectivity to GitHub
- **Ansible Compatibility**: Your Ansible version is compatible with the collections
- **Permissions**: You have appropriate permissions to install packages and modify directories

### Common Issues

1. **Collection Installation Fails**: Check your internet connection and GitHub access
2. **Permission Denied**: Ensure you're running commands with appropriate privileges
3. **Version Conflicts**: Verify all prerequisite versions are met

## Next Steps

After completing the installation, you can start creating and running Ansible playbooks for Junos device automation.