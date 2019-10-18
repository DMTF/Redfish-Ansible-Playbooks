Copyright 2019 DMTF. All rights reserved.

# Redfish Ansible OEM Extensions

## About

This document describes an approach for extending the standard Redfish Ansible modules in order to support OEM extensions used by various vendors.

## The Approach

With the current Redfish Ansible support, there are three main modules (`redfish_command.py`, `redfish_config.py` and `redfish_info.py`). These three modules are relatively thin modules that establish the commands and their parameters and then call out to a utility module, `redfish_utils.py`, that has the bulk of the logic to interact with the Redfish services.

The `redfish_utils.py` module implements a `RedfishUtils` class that has many methods to perform the various Redfish operations. With this architecture, a new vendor module that needs to add a specific Oem feature can create a new class (e.g. `ContosoRedfishUtils`) that extends the existing `RedfishUtils`. New method(s) can be added to that class (or existing methods overridden) while still being able to leverage all the existing standard methods in the parent class. Then a new vendor module (e.g. `contoso_redfish_command.py`) can be written that is basically a sparse version of `redfish_command.py` that only needs to handle the new or changed vendor commands.

Here is an example module that does what is described above. This example provides a non-standard `CreateBiosConfigJob` command. Here is the code:

```
DOCUMENTATION = '''
---
module: contoso_redfish_command
version_added: "2.7"
short_description: Manages Out-Of-Band controllers using Contoso Oem Redfish APIs
description:
  - Builds Redfish URIs locally and sends them to remote OOB controllers to
    perform an action.
  - For use with operations that require Contoso Oem extensions 
options:
  category:
    required: true
    description:
      - Category to execute on OOB controller
  command:
    required: true
    description:
      - List of commands to execute on OOB controller
  baseuri:
    required: true
    description:
      - Base URI of OOB controller
  user:
    required: true
    description:
      - User for authentication with OOB controller
  password:
    required: true
    description:
      - Password for authentication with OOB controller

author: ""
'''

EXAMPLES = '''
  - name: Create BIOS config job
    contoso_redfish_command:
      category: Systems
      command: CreateBiosConfigJob
      baseuri: "{{ baseuri }}"
      user: "{{ user }}"
      password: "{{ password }}"
'''

RETURN = '''
msg:
    description: Message with action result or error description
    returned: always
    type: string
    sample: "Action was successful"
'''

import re
from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils.redfish_utils import RedfishUtils, HEADERS
from ansible.module_utils._text import to_native

class ContosoRedfishUtils(RedfishUtils):

    def create_bios_config_job(self):
        result = {}
        key = "Bios"
        jobs = "Jobs"

        # Search for 'key' entry and extract URI from it
        response = self.get_request(self.root_uri + self.systems_uri)
        if response['ret'] is False:
            return response
        result['ret'] = True
        data = response['data']

        if key not in data:
            return {'ret': False, 'msg': "Key %s not found" % key}

        bios_uri = data[key]["@odata.id"]

        # Extract proper URI
        response = self.get_request(self.root_uri + bios_uri)
        if response['ret'] is False:
            return response
        result['ret'] = True
        data = response['data']
        set_bios_attr_uri = data["@Redfish.Settings"]["SettingsObject"][
            "@odata.id"]

        payload = {"TargetSettingsURI": set_bios_attr_uri}
        response = self.post_request(
            self.root_uri + self.manager_uri + "/" + jobs,
            payload, HEADERS)
        if response['ret'] is False:
            return response

        response_output = response['resp'].__dict__
        job_id = response_output["headers"]["Location"]
        job_id = re.search("JID_.+", job_id).group()
        # Currently not passing job_id back to user but patch is coming
        return {'ret': True, 'msg': "Config job %s created" % job_id}


CATEGORY_COMMANDS_ALL = {
    "Systems": ["CreateBiosConfigJob"],
    "Accounts": [],
    "Manager": []
}


def main():
    result = {}
    module = AnsibleModule(
        argument_spec=dict(
            category=dict(required=True),
            command=dict(required=True, type='list'),
            baseuri=dict(required=True),
            user=dict(required=True),
            password=dict(required=True, no_log=True)
        ),
        supports_check_mode=False
    )

    category = module.params['category']
    command_list = module.params['command']

    # admin credentials used for authentication
    creds = {'user': module.params['user'],
             'pswd': module.params['password']}

    # Build root URI
    root_uri = "https://" + module.params['baseuri']
    rf_uri = "/redfish/v1/"
    rf_utils = ContosoRedfishUtils(creds, root_uri)

    # Check that Category is valid
    if category not in CATEGORY_COMMANDS_ALL:
        module.fail_json(msg=to_native("Invalid Category '%s'. Valid Categories = %s" % (category, CATEGORY_COMMANDS_ALL.keys())))

    # Check that all commands are valid
    for cmd in command_list:
        # Fail if even one command given is invalid
        if cmd not in CATEGORY_COMMANDS_ALL[category]:
            module.fail_json(msg=to_native("Invalid Command '%s'. Valid Commands = %s" % (cmd, CATEGORY_COMMANDS_ALL[category])))

    # Organize by Categories / Commands

    if category == "Systems":
        # execute only if we find a System resource
        result = rf_utils._find_systems_resource(rf_uri)
        if result['ret'] is False:
            module.fail_json(msg=to_native(result['msg']))

        for command in command_list:
            if command == "CreateBiosConfigJob":
                # execute only if we find a Managers resource
                result = rf_utils._find_managers_resource(rf_uri)
                if result['ret'] is False:
                    module.fail_json(msg=to_native(result['msg']))
                result = rf_utils.create_bios_config_job()

    # Return data back or fail with proper message
    if result['ret'] is True:
        del result['ret']
        module.exit_json(changed=True, msg='Action was successful')
    else:
        module.fail_json(msg=to_native(result['msg']))


if __name__ == '__main__':
    main()
``` 

Given this new vendor module, playbooks like this can be written that call a mix of tasks using standard and vendor modules. For example, here is a playbook that sets some BIOS attributes using a standard module and then applies the attributes using a vendor module:

```
---
- hosts: myhosts
  connection: local
  name: Set BIOS attribute - choose below
  gather_facts: False

  # Updating BIOS settings requires creating a configuration job to implement,
  # which then reboots the system.

  # BIOS attributes that have been tested
  #
  # Name      Value
  # --------------------------------------------------
  # OsWatchdogTimer   Disabled / Enabled
  # ProcVirtualization    Disabled / Enabled
  # MemTest     Disabled / Enabled
  # SriovGlobalEnable   Disabled / Enabled

  vars:
  - attribute_name1: SriovGlobalEnable
  - attribute_value1: Disabled
  - attribute_name2: ProcVirtualization
  - attribute_value2: Enabled

  tasks:

  - name: Set BIOS attribute {{ attribute_name1 }} to {{ attribute_value1 }}
    redfish_config:
      category: Systems
      command: SetBiosAttributes
      bios_attr_name: "{{ attribute_name1 }}"
      bios_attr_value: "{{ attribute_value1 }}"
      baseuri: "{{ baseuri }}"
      user: "{{ user }}"
      password: "{{ password }}"

  - name: Set BIOS attribute {{ attribute_name2 }} to {{ attribute_value2 }}
    redfish_config:
      category: Systems
      command: SetBiosAttributes
      bios_attr_name: "{{ attribute_name2 }}"
      bios_attr_value: "{{ attribute_value2 }}"
      baseuri: "{{ baseuri }}"
      user: "{{ user }}"
      password: "{{ password }}"

  - name: Schedule Config Job - Reboot
    contoso_redfish_command:
      category: Systems
      command: CreateBiosConfigJob
      baseuri: "{{ baseuri }}"
      user: "{{ user }}"
      password: "{{ password }}"                                    
```
 
Note that the first 2 tasks use a standard module (`redfish_config`) and the last task uses a vendor module (`contoso_redfish_command`).

And here is an example playbook to set the BIOS attributes using all standard modules:

```
---
- hosts: myhosts
  connection: local
  name: Set BIOS attribute - choose below
  gather_facts: False

  # Updating BIOS settings requires creating a configuration job to implement,
  # which then reboots the system.

  # BIOS attributes that have been tested
  #
  # Name      Value
  # --------------------------------------------------
  # OsWatchdogTimer   Disabled / Enabled
  # ProcVirtualization    Disabled / Enabled
  # MemTest     Disabled / Enabled
  # SriovGlobalEnable   Disabled / Enabled

  vars:
  - attribute_name1: SriovGlobalEnable
  - attribute_value1: Disabled
  - attribute_name2: ProcVirtualization
  - attribute_value2: Enabled

  tasks:

  - name: Set BIOS attribute {{ attribute_name1 }} to {{ attribute_value1 }}
    redfish_config:
      category: Systems
      command: SetBiosAttributes
      bios_attr_name: "{{ attribute_name1 }}"
      bios_attr_value: "{{ attribute_value1 }}"
      baseuri: "{{ baseuri }}"
      user: "{{ user }}"
      password: "{{ password }}"

  - name: Set BIOS attribute {{ attribute_name2 }} to {{ attribute_value2 }}
    redfish_config:
      category: Systems
      command: SetBiosAttributes
      bios_attr_name: "{{ attribute_name2 }}"
      bios_attr_value: "{{ attribute_value2 }}"
      baseuri: "{{ baseuri }}"
      user: "{{ user }}"
      password: "{{ password }}"

  - name: Reboot
    redfish_command:
      category: Systems
      command: PowerReboot
      baseuri: "{{ baseuri }}"
      user: "{{ user }}"
      password: "{{ password }}"
```
