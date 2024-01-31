---
layout: post
title:  "Fiber channel zoning with Ansible"
date:   2024-01-31 12:00:00 -0400
categories: fc brocade zoning ansible
---

Example repository can be found [here](https://github.com/zeroonebn/brocade-zoning-with-ansible).

In nearly every non trivial configuration, you will have to implement zoning on your fiber channel switches. Zoning is the partitioning of a Fiber Channel fabric into smaller subsets to restrict interference, add security and to simplify management.

On Brocade Fibre Channel switches, zoning can be managed via graphical interface or command line. 

Brocade also provides an Ansible [collection](https://galaxy.ansible.com/ui/repo/published/brocade/fos/docs/) to manage zoning (and more) on Brocade fiber channel switches.

# Zoning objects

Zoning generally involves 3 types of objects:
* Aliases
* Zones
* Configurations

## Aliases
Zoning is all about restricting communications inside a Fibre Channel Fabric, this restriction can be defined based on two elements:
* port
* world wide port name (WWPN)

WWPN zoning is generally prefered nowadays. A WWPN is an 8-byte number that serves as unique identifier for a device port. It is generally represented as colon separated hexadecimal octets (similar to ethernet MAC address).
Using WWPN directly in zoning, while possible, is not recommended since it makes maintenance difficult to say the least.

Aliases allows you to give a meaningful name for one or more WWPN, making maintenance way easier.

## Zones
As their name implies, zones are at the heart of zoning. Communication will be allowed only between WWPN in the same zone. There can (and probably will) be multiple zones. The same WWPN can belong to multiple zones. You can put directly WWPNs in a zone but it is recommended to use aliases instead.

## Configurations
Configurations are how you indicate which zones are active. A configuration (usually) contains multiple zones. You can have multiple configurations defined but there can only be one configuration active at any given time on a fabric : the effective configuration. Only zones in the effective configuration are active and allow communication.

# Zoning with Ansible

The `brocade.fos` Ansible collection allows you to manage all 3 types of objects.

You can install the `brocade.fos` collection by using the following command: `ansible-galaxy collection install brocade.fos`

## Connection and authentication

Since Brocade switches have a restricted shell, the "normal" Ansible inventory mechanism isn't useable. Instead, connection and authentication information must be provided in the playbook with the following parameters:
<pre>credential:
  fos_ip_addr: name_or_ip
  fos_user_name: username
  fos_password: password
  https: true_or_false
vfid: n</pre>

`fos_ip_addr` contains the host name or IP address of the switch.
`fos_user_name` contains the username to connect with.
`fos_password` contains the password associated with the switch user.
`https` contains True to use https to connect to the switch, False to use http.
`vfid` contains the logical switch ID to use. If the switch doesn't have logical switches activated, you must use -1.

## Aliases

Aliases can be manipulated with the `brocade_zoning_alias` module.

The following operations can be performed:
* Create aliases
* Add members (WWPNs) to existing aliases
* Delete aliases
* Remove members (WWPNs) from existing aliases

To create aliases, use the following playbook:
<pre>- hosts: localhost                      # we don't use inventory for the switches
  connection: local                          # Run everything on the control node
  collections:
  - brocade.fos                              # Reference the brocade.fos collection
  gather_facts: False                        # We don't need to gather facts
  vars_prompt:                               # We don't store passwords so prompt switch password
  - name: brocade_password
    prompt: Enter Brocade password

  tasks:
  - name: Create aliases                     # Descriptive name for the task
    brocade_zoning_alias:
      credential:
        fos_ip_addr: switch_name_or_ip       # CHANGEME host name or IP address of the switch
        fos_user_name: admin                 # CHANGEME username to connect to the switch
        fos_password: "{{brocade_password}}" # Use the prompted password
        https: False                         # Set to True to use https instead of http
      vfid: -1                               # CHANGEME use the logical switch ID if activated
      aliases:
      - name: ALIAS_1                        # CHANGEME name of the first alias
        members:
        - 11:22:33:44:55:66:77:88            # CHANGEME WWPN to put in the first alias
      - name: ALIAS_2                        # CHANGEME name of the second alias
        members:
        - 22:22:33:44:55:66:77:88            # CHANGEME first WWPN to put in the second alias
        - 22:22:33:44:55:66:77:89            # CHANGEME second WWPN to put in the second alias</pre>

Lines that contain CHANGEME should be adjusted for your configuration.

The playbook can be run with the following command:
`ansible-playbook name_of_playbook_file`

Ansible will prompt for the password of the switch user. Enter it and press Enter.

After a few seconds, you should get an output similar to the following:
<pre>[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
Enter Brocade password:

PLAY [localhost] **********************************************************************************************************************************************************************************

TASK [Create aliases] *****************************************************************************************************************************************************************************
changed: [localhost]

PLAY RECAP ****************************************************************************************************************************************************************************************
localhost                  : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0</pre>

Ansible normally works with a list of machine to manage called the inventory. Since the Brocade switches have a restricted shell, we can't use the Ansible inventory mechanism. The warning is related to the lack of inventory and can be ignored.

The line `changed: [localhost]` indicates that the task was successful and made a change (created 2 aliases in our case).

If we run the same command again we should get a similar output except for the line `ok: [localhost]` indicating that the task was successful and didn't make any change (since the aliases already exist).

To delete one or more aliases, replace alias: with alias_to_delete: in your playbook.

## Zones

Zones can be manipulated with the `brocade_zoning_zone` module.

The following operations can be performed:
* Create zones
* Add members (WWPNs or preferably aliases) to existing zones
* Delete zones
* Remove members (WWPNs or aliases) from existing zones

To create zones, use the following playbook:
<pre>- hosts: localhost                      # we don't use inventory for the switches
  connection: local                          # Run everything on the control node
  collections:
  - brocade.fos                              # Reference the brocade.fos collection
  gather_facts: False                        # We don't need to gather facts
  vars_prompt:                               # We don't store passwords so prompt switch password
  - name: brocade_password
    prompt: Enter Brocade password

  tasks:
  - name: Create zones                       # Descriptive name for the task
    brocade_zoning_zone:
      credential:
        fos_ip_addr: switch_name_or_ip       # CHANGEME host name or IP address of the switch
        fos_user_name: admin                 # CHANGEME username to connect to the switch
        fos_password: "{{brocade_password}}" # Use the prompted password
        https: False                         # Set to True to use https instead of http
      vfid: -1                               # CHANGEME use the logical switch ID if activated
      zones:
      - name: ALIAS_1_ALIAS2                 # CHANGEME name of the zone
        members:
        - ALIAS_1                            # CHANGEME first member of the zone
        - ALIAS_2                            # CHANGEME second member of the zone</pre>

The playbook can be run with `ansible-playbook`.

Just like the alias playbook, the output indicates if a change has been done.

To delete one or more zone, replace zones: with zones_to_delete: in your playbook.

## Configurations

Configurations can be manipulated with the `brocade_zoning_cfg` modules.

The usual operations can be performed:
* Create configurations
* Add zones to existing configuration
* Delete configurations
* Remove zones from existing configuration

The module can also enable a configuration.

Unlike aliases and zones (for which most actions are create and delete), existing configurations are generally modified (add or remove zones).

To add zones to an existing configuration, use the following playbook:
<pre>- hosts: localhost                      # we don't use inventory for the switches
  connection: local                          # Run everything on the control node
  collections:
  - brocade.fos                              # Reference the brocade.fos collection
  gather_facts: False                        # We don't need to gather facts
  vars_prompt:                               # We don't store passwords so prompt switch password
  - name: brocade_password
    prompt: Enter Brocade password

  tasks:
  - name: Add zone to configuration          # Descriptive name for the task
    brocade_zoning_cfg:
      credential:
        fos_ip_addr: switch_name_or_ip       # CHANGEME host name or IP address of the switch
        fos_user_name: admin                 # CHANGEME username to connect to the switch
        fos_password: "{{brocade_password}}" # Use the prompted password
        https: False                         # Set to True to use https instead of http
      vfid: -1                               # CHANGEME use the logical switch ID if activated
      members_add_only: true                 # If not specified or false, all zones already in the configuration will be removed
      cfgs:
      - name: CFG                            # CHANGEME name of the configuration
        members:
        - ALIAS_1_ALIAS_2                    # CHANGEME zone to add to the configuration</pre>

The playbook can be run with `ansible-playbook`.

The line `member_add_only: true` is extremely important: if it is missing or set to false, the playbook will remove all zones that are not specified in members from the configuration, which is probably not what you want.

To remove zones from the configurations, replace `member_add_only: true` with `member_remove_only: true`.

# References

The Ansible documentation: [https://docs.ansible.com/](https://docs.ansible.com/)

The brocade.fos collection documentation: [https://galaxy.ansible.com/ui/repo/published/brocade/fos/](https://galaxy.ansible.com/ui/repo/published/brocade/fos/)