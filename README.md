# IOS_NXOS_Ansible - Jinja Branch

Repo for using Ansible to deploy and maintain configuration baseline for 3-tier demo environment comprised of IOS-xe and NX-OS devices. Key changes from main branch are using jinja templates for all IOS configurations, specifically the playbooks:
```
- ios_l3_config.yml
- ios_ospf_config.yml
- access_vlans_config.yml
- access_uplinks_config.yml
```
These are imported in the `network_deploy.yml` playbook and may eventually be copied there entirely to clean directory structure.

This repo has 2 overarching playbooks that reference multiple plays/roles - `new_device.yml` and `network_deploy.yml`.

The `new_device.yml` playbook imports individual plays which were left seperate in order to allow admins to use those plays as needed or reference in seperate playbooks (mainly used for configuring network services such as DNS, NTP, etc). 

The `network_deploy.yml` playbook configures the network devices to meet the desired endstate referenced below. This is through the use of roles which reference variables either at the group or host level. Standalone plays can be added to the `network_deploy.yml` playbook as desired by using the ansible builtin module. For example, to capture a backup after the devices are configured, simply add this to the bottom of the `network_deploy.yml` file:

```
- name: Backup Configuration
  ansible.builtin.import_playbook: backup_config.yml
```

- Starting State:
    - Cable according to draw-io diagram. 
    - Update vars files to reflect interface types/IDs
    - Complete initial configs of each device 
- Initial Config:
    - Enable SSH
    - Create a user and password with priv 15
    - Create management vrf
    - Configure management interfaces to use vrf
    - Add default static route for vrf if ansible controller not in same vlan
    - Verify connetivity from Ansible controller to hosts:
     `ansible all -m ping `

- Desired Endstate:
    - Distribution and Core L3 connectivity and OSPF
    - Distribution VPC domain 
    - Distribution to Access L2 trunks using VPC
    - Distribution using HSRP for VLAN SVI's

## Running the Playbook
The `network_deploy.yml` playbook deploys the desired state to all layers of the network (Core, Distribution, Access) via ssh connectivity. To execute the playbook, navigate to the ansible directory on the controller and execute: `ansible-playbook network_deploy.yml`

As network endpoints, features, configurations are changed in the ansible environment you can re-deploy the desired state by either running the entire playbook, limiting to specific hosts, or by tags.

To run the playbook to only adjust the core devices, run: `ansible-playbook network_deploy.yml --limit core`

To run the playbook solely based on tags, run: `ansible-playbook network_deploy.yml --tags "vlan"`

To start the playbook at a specific task, run: `ansible-playbook network_deploy.yml --start-at-task="name of task as written in the specifc role>task>main.yml"`

## Future Developments
Convert all roles to jinja templates, which reduces run time and dependency on ansible-galaxy modules.