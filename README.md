Copyright 2019 DMTF. All rights reserved.

# Redfish Ansible Playbooks

## About

This repository contains a set of example Ansible playbooks for invoking the Redfish Ansible modules.

## Prerequisites

To use these playbooks, you first need to install the 2.10 or later release of Ansible. The Redfish Ansible modules are included in the Ansible distribution. Instructions for installing Ansible can be found here:

[Ansible Installation Guide](https://docs.ansible.com/ansible/2.10/installation_guide/intro_installation.html)

## Release process

The playbooks in the `2.9` branch of this repository are designed to work with the [latest stable 2.9 release of Ansible](https://docs.ansible.com/ansible/2.9/reference_appendices/release_and_maintenance.html). The playbooks in the `master` branch of this repository are designed to work with the [2.10 or later  Ansible release](https://docs.ansible.com/ansible/2.10/reference_appendices/release_and_maintenance.html).

## Running the playbooks

1. [Clone](https://help.github.com/en/articles/cloning-a-repository) this repository
2. Edit the example [inventory.yml](inventory.yml) file to specify the hostname/ip and credentials of your Redfish service(s).
3. Run a playbook, for example:
```
ansible-playbook -i inventory.yml playbooks/systems/get_system_inventory.yml
```


## Redfish Ansible modules

The Redfish Ansible modules are maintained in the main [Ansible Collections community.general GitHub repository](https://github.com/ansible-collections/community.general).

The three Redfish modules are summarized here:

1. [redfish_command](https://docs.ansible.com/ansible/2.10/collections/community/general/redfish_command_module.html) (source: [redfish_command.py](https://github.com/ansible-collections/community.general/blob/main/plugins/modules/remote_management/redfish/redfish_command.py))

	The `redfish_command` module performs Out-Of-Band (OOB) controller operations like log management, adding/deleting/modifying users, and power operations (e.g. on, off, reboot, etc.).

2. [redfish_config](https://docs.ansible.com/ansible/2.10/collections/community/general/redfish_config_module.html) (source: [redfish_config.py](https://github.com/ansible-collections/community.general/blob/main/plugins/modules/remote_management/redfish/redfish_config.py))

	The `redfish_config` module performs OOB controller operations like setting the BIOS configuration.

3. [redfish_info](https://docs.ansible.com/ansible/2.10/collections/community/general/redfish_info_module.html) (source: [redfish_info.py](https://github.com/ansible-collections/community.general/blob/main/plugins/modules/remote_management/redfish/redfish_info.py))

	The `redfish_info` module retrieves information about the OOB controller like Systems inventory and Accounts inventory.

All three of the above modules use this utility module: [redfish_utils.py](https://github.com/ansible-collections/community.general/blob/main/plugins/module_utils/redfish_utils.py)


## Contributing

For the Redfish Ansible modules: File issues or submit pull requests in the [Ansible Collections community.general repo](https://github.com/ansible-collections/community.general) following the [Ansible Community Guide](https://docs.ansible.com/ansible/latest/community/index.html) and the [developer guide for collections](https://docs.ansible.com/ansible/devel/dev_guide/developing_collections.html#contributing-to-collections).

For the Redfish Ansible playbooks: File issues or submit pull requests in this github repository.

## OEM Extensions

See [OEM_EXTENSIONS.md](OEM_EXTENSIONS.md) for an outine of how to extend the standard Redfish Ansible modules to support OEM extensions.

## Acknowledgements

These playbooks are based on the ones provided in https://github.com/dell/redfish-ansible-module


