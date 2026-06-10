# IOS_NXOS_Ansible
Repo for using Ansible to deploy and maintain configuration baseline for 3-tier demo environment comprised of IOS-xe and NX-OS devices. 

This repo has 2 overarching playbooks that reference multiple plays/roles - `new_device.yml` and `network_deploy.yml`.

The `new_device.yml` playbook imports individual plays which were left seperate in order to allow admins to use those plays as needed or reference in seperate playbooks (mainly used for configuring network services such as DNS, NTP, etc). 

The `network_deploy.yml` playbook configures the network devices to meet the desired endstate referenced below. This is through the use of roles which reference variables either at the group or host level. Standalone plays can be added to the `network_deploy.yml` playbook as desired by using the ansible builtin module. For example, to capture a backup after the devices are configured, simply add this to the bottom of the `network_deploy.yml` file:

`- name: Backup Configuration`

`ansible.builtin.import_playbook: backup_config.yml`


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

## IOS-XE Software Upgrades

The `upgrade_iosxe.yml` playbook upgrades IOS-XE devices in the `iosxe` inventory group. It backs up each device's configuration, runs the `upgrade_iosxe` role, and saves the running config to startup when complete.

Upgrade variables are split the same way as the rest of this repo:

- **`group_vars/iosxe.yml`** — settings shared by all IOS-XE devices (target version, file server, timeouts, VRF)
- **`host_vars/<hostname>.yml`** — per-device values such as `ios_image_filename` (image name can differ by chassis/model)
- **`roles/upgrade_iosxe/vars/main.yml`** — role-internal pre/post verification commands

### Image transport: controller push vs remote server

`ios_image_transport` controls **who initiates the file transfer** and **where the image file lives before the upgrade**.

**`remote_server` (default)** — the **network device pulls** the image from a file server on your management network. Ansible SSHes into the switch and runs a `copy scp://...` (or similar) command. The image must already be on a reachable server such as `192.168.1.1`. Ansible does not need a local copy of the `.bin` file.

```
[File server 192.168.1.1]  ──SCP/TFTP──>  [IOS-XE device]
                              (device initiates)
```

**`controller`** — the **Ansible controller pushes** the image directly to the device over SCP. The `.bin` file must exist on the controller at `ios_image_local_path` (default `./images/iosxe/`). The device must allow inbound SCP (`ip scp server enable`).

```
[Ansible controller]  ──SCP──>  [IOS-XE device]
     (controller initiates)
```

Use `remote_server` when you maintain images on a central file server (common in production). Use `controller` when you have the image locally on the Ansible host and want a simpler lab setup without standing up a separate file server.

To switch modes, set in `group_vars/iosxe.yml`:

```yaml
# Device pulls from file server (default)
ios_image_transport: remote_server
ios_image_server: 192.168.1.1
ios_image_path: images/iosxe

# Ansible controller pushes the file
ios_image_transport: controller
ios_image_local_path: ./images/iosxe
```

### Before you run against real devices

1. **Place the image file** in the correct location for your transport mode:
   - `remote_server`: copy the `.bin` to your file server at the path defined by `ios_image_server` and `ios_image_path`
   - `controller`: copy the `.bin` into `./images/iosxe/` on the Ansible controller (or update `ios_image_local_path`)

2. **Set `ios_image_filename`** in each device's `host_vars/<hostname>.yml` if models differ between hosts.

3. **Set `ios_target_version`** in `group_vars/iosxe.yml` to match the version string you expect after the upgrade (used for pre-check skip and post-upgrade validation).

4. **Enable SCP on devices** when using either transport mode that relies on SCP (`ip scp server enable`).

5. **Understand reload behavior**: on IOS-XE 17.x, `install add file ... activate commit` typically reloads the device automatically. `ios_upgrade_reload` defaults to `false` because a manual reload is usually unnecessary. Set it to `true` only if your workflow requires an explicit reload after install.

6. **Run one device at a time** in production: `upgrade_serial: 1` in `group_vars/iosxe.yml` limits the upgrade play to one host at a time. Increase only when you are confident parallel upgrades are safe.

### Running the upgrade playbook

```bash
# Full upgrade: backup, upgrade (one device at a time), save config
ansible-playbook upgrade_iosxe.yml

# Single device
ansible-playbook upgrade_iosxe.yml --limit CORE-1

# Pre-checks only (facts, version display, skip if already at target)
ansible-playbook upgrade_iosxe.yml --tags pre_upgrade

# Copy image only (after pre-checks pass)
ansible-playbook upgrade_iosxe.yml --tags copy_image --limit CORE-1
```

Tagged phases available in the `upgrade_iosxe` role: `pre_upgrade`, `copy_image`, `install`, `reload`, `post_upgrade`.

## Future Developments
The next step for this repo is to initialize and use IOS netconf and NX-OS API's for connectivity. This will allow us to easily incorporate jinja templates to generate configurations in xml format using appropriate structures. 