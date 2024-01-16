---
layout: post
title:  "LPAR provisionning with Ansible"
date:   2024-01-16 14:00:00 -0400
categories: power hmc phyp ansible
---

If you prefer a video, follow this [link](https://youtu.be/tlbesjbGDpM).

Example repository can be found [here](https://github.com/zeroonebn/lpar-provisioning-with-ansible).

Wikipedia describes Ansible as a suite of software tools that enables infrastructure as code. It is an open source software that helps with provisioning, configuration management and application deployment.

Ansible is extremely extensible, third parties can develop modules and plugins to manage a wide variety of systems. Modules and plugins can be grouped in a collection and collections can be shared on a central repository called [Ansible Galaxy](https://galaxy.ansible.com). IBM maintains the [power_hmc collection](https://galaxy.ansible.com/ui/repo/published/ibm/power_hmc/) that helps performing a lot of operations on a Hardware Management Console from ansible.

# Idempotency

A big part of working with Ansible is writing and using playbooks. Playbooks are text files written in YAML that contain a list of tasks to be executed.

It is possible write playbooks that are idempotent, meaning that running a playbook multiple times will not change the result beyond the first execution. Be warned that it is very possible to write playbooks that are not idempotent (voluntarily or not).

# Control node

A playbook is executed on a control node. The controle node is a Linux or unix-like (AIX, IBM i for example) system. It can be run from a Windows system via the Windows Subsystem for Linux (WSL).

# Setting up the control node

Ansible is mostly agentless, meaning there is no need to install anything on systems that are managed by Ansible. You generally only need to install Ansible on the control node. The control node can be the administrator worksation or a management server.

## Linux

Most Linux distribution have a package called **ansible** that can be installed:
- For deb based distributions (Debian, Ubuntu, ...): <pre>sudo apt install ansible</pre>
- For rpm based distributions (Fedora, RedHat, Suse, ...): <pre>dnf install ansible</pre>

## AIX

The easiest way to install Ansible on AIX is via the AIX toolbox for Open Source Software following this [guide](https://www.ibm.com/support/pages/node/6585774).

Ansible can then be installed with the follozing command: <pre>dnf install ansible<pre>

## IBM i

On IBM i, install the open source packages following this [guide](http://ibm.biz/ibmi-rpms).

Ansible can then be installed with the following command: <pre>yum install ansible<pre>

Ansible will be accessible in PASE, the easiest way to access it is via ssh.

# Installing the power_hmc collection

Once Ansible is installed, the **power_hmc** collection can be installed with the following command:
<pre>ansible-galaxy collection install ibm.power_hmc</pre>

# Creating a LPAR with ansible

Using your favourite editor, create the file <pre>first_lpar.yaml</pre> with the following content:

<pre>- hosts: localhost        # We don't use Ansible inventory from HMC
  connection: local       # Run everything on the control node
  collections:
  - ibm.power_hmc         # Reference the power_hmc collection
  gather_facts: False     # We don't need to gather facts
  vars_prompt:            # We don't store passwords, so prompt for the HMC password
    - name: hmc_password
      prompt: Enter HMC password

  tasks:
    - name: Create first lpar                   # Descriptive name of the task
      powervm_lpar_instance:                    # This is the name of the module provided by the power_hmc collection
        hmc_host: hmc_host_or_ip                # CHANGEME host name of IP of the HMC
        hmc_auth:
          username: hscroot                     # CHANGEME username to user to connect to the HMC
          password: '{% raw %}{{ hmc_password }}{% endraw %}'        # Use the password entered by the user
        system_name: S1022                      # CHANGEME name of the POWER server
        vm_name: LPAR_NAME                      # CHANGEME name of the LPAR to create
        virt_network_config:
          - network_name: VLAN638-ETHERNET0     # CHANGEME name of the VLAN as it appears in the HMC enhanced interface
        npiv_config:
          - vios_name: VIOS1                    # CHANGEME name of the first VIOS
            fc_port: fcs0                       # CHANGEME name of the physical FC port in the first VIOS
          - vios_name: VIOS2                    # CHANGEME name of the second VIOS
            fc_port: fcs3                       # CHANGEME name of the physical FC port in the second VIOS
        os_type: ibmi                           # The type of LPAR, can be ibmi, aix, linux or aix_linux
        shared_proc_pool: IBMi                  # CHANGEME name of the shared processor pool
        max_proc_unit: 4                        # Maximum processing units
        min_proc_unit: 0.05                     # Minimum processing units
        proc_unit: 0.05                         # (Desired) processing units
        max_proc: 4                             # Maximum VP
        min_proc: 1                             # Minimum VP
        proc: 1                                 # (Desired) VP
        max_mem: 12288                          # Maximum memory (in MB)
        min_mem: 4096                           # Minimum memory (in MB)
        mem: 8192                               # (Desired) memory (in MB)
        state: present                          # Indicates we want the partition to be created if needed
</pre>

Playbooks are in YAML, columning is important! Please read the following about [YAML](https://yaml.org/).

Lines that contain CHANGEME should be adapted to your configuration.

The playbook can be run with the following command:
<pre>ansible-playbook first_lpar.yaml</pre>

Ansible will prompt for the HMC password. Enter it and press enter.

After a few seconds, you should get an output similar to the following:
<pre>[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
Enter HMC password: 

PLAY [localhost] **************************************************************************************************************************************************************************************************

TASK [Create first lpar] ******************************************************************************************************************************************************************************************
changed: [localhost]

PLAY RECAP ********************************************************************************************************************************************************************************************************
localhost                  : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
</pre>

Ansible normally works with a list of machine to manage called the inventory. Since the HMC only has a restricted shell, we can't use the Ansible inventory mechanism. The 2 warnings are related to the lack of inventory and can be ignored.

The line <pre>changed: [localhost]</pre> indicates that the task was successful and made a change (created the LPAR in our case).

If we run the same command again we should get a similar output except for the line <pre>ok: [localhost]</pre> indicating that the task was successful and didn't make any change (since the LPAR was already existing).

I noticed two limitations on the module currently:
* It is not possible to set the tagged IO for IBM i LPAR from Ansible. The load source adapter has to be set manually in the HMC.
* The processing units has to be a multiple of the minimum processing unit per core (0.05 since POWER7+, 0.1 before that).

# Deleting a LPAR with Ansible

Using your favourite editor, create the file <pre>remove_first_lpar.yaml</pre> with the following content:

<pre>- hosts: localhost        # We don't use Ansible inventory from HMC
  connection: local       # Run everything on the control node
  collections:
  - ibm.power_hmc         # Reference the power_hmc collection
  gather_facts: False     # We don't need to gather facts
  vars_prompt:            # We don't store passwords, so prompt for the HMC password
    - name: hmc_password
      prompt: Enter HMC password

  tasks:
    - name: Remove first lpar                   # Descriptive name of the task
      powervm_lpar_instance:                    # This is the name of the module provided by the power_hmc collection
        hmc_host: hmc_host_or_ip                # CHANGEME host name of IP of the HMC
        hmc_auth:
          username: hscroot                     # CHANGEME username to user to connect to the HMC
          password: '{% raw %}{{ hmc_password }}{% endraw %}'        # Use the password entered by the user
        system_name: S1022                      # CHANGEME name of the POWER server
        vm_name: LPAR_NAME                      # CHANGEME name of the LPAR to remove
        state: absent                           # Indicates we want the partition to be deleted if existing
</pre>

Lines that contain CHANGEME should be adapted to your configuration. **Warning, this will delete the partition, do not indicate a  partition you want to keep.**

This file is similar to the one used to create the partition, we simply removed the LPAR configuration information and changed state from present to absent.

The playbook can be run with the following command:
<pre>ansible-playbook remove_first_lpar.yaml</pre>

This should delete the partition indicated on the vm_name line.

# References

The Ansible documentation: [https://docs.ansible.com/](https://docs.ansible.com/)

The power_hmc collection documentation: [https://galaxy.ansible.com/ui/repo/published/ibm/power_hmc/docs/](https://galaxy.ansible.com/ui/repo/published/ibm/power_hmc/docs/)

Ansible for IBM Power Redbook: [https://www.redbooks.ibm.com/abstracts/sg248551.html](https://www.redbooks.ibm.com/abstracts/sg248551.html)