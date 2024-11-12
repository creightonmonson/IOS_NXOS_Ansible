# IOS_NXOS_Ansible
Repo for using Ansible to deploy and maintain configuration baseline for 3-tier demo environment comprised of IOS-xe and NX-OS devices

- Starting State:
    - Cable according to draw-io diagram. 
    - Update vars files to reflect interface types/IDs
    - Complete initial configs of each device
- Initial Config:
    - Enable SSH
    - Create a user and password with priv 15
    - Create management vrf
    - Configure management interfaces to use vrf
    - Add default static route for vrf
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

## Future Developments
The next step for this repo is to initialize and use IOS netconf and NX-OS API's for connectivity. This will allow us to easily incorporate jinja templates to generate configurations in xml format using appropriate structures. 