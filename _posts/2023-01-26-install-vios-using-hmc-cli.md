---
layout: post
title:  "Install VIOS using HMC CLI"
date:   2023-01-26 12:00:00 -0500
categories: power hmc cli vios
---

In the [previous blog post](https://bigblueblog.xyz/power/hmc/cli/2023/01/18/setup-power-with-hmc-cli/), we did the basic setup of a a POWER server using the HMC CLI. In this post we'll see one way to install the VIOS from the HMC.

# What is a VIOS

VIOS stands for Virtual I/O Server, it is a virtual appliance that provides the following functions:
* sharing of physical I/O resources (disk, tape and network)
* memory transfer for Live Partition Mobility
* paging capability for Active Memory Sharing (memory overallaction)

While VIOS is part of PowerVM it is not in itself a hypervisor. On POWER, the hypervisor is called PHYP (PowerVM Hypervisor), it is integrated in the firmware of most POWER Servers. VIOS runs as an LPAR, it is based on AIX (7.2 for VIOS 3). VIOS is included in PowerVM, you need an up to date PowerVM maintenance to have support and updates for VIOS. You do not need AIX (or any other operating system) license for your VIOS LPARs.

VIOS supports failover for all its functions and load balacing for most of them using two VIOS LPARs. A dual VIOS configuration is strongly recommended for better performance and availability. Single VIOS, while supported, is strongly discouraged since it will require downtimes for all partitions using the VIOS.

It is also possible to have more than 2 VIOS on a system (for example a pair of VIOS for production LPARs another one for non-production LPARs).

# How to install a VIOS

There are several methods to install a VIOS:
* Install from a physical DVD
* Install from a USB key
* Network install from a NIM server
* Network install from the HMC

This post will detail how to install a VIOS from the HMC.

# NIM

NIM stands for Network Installation Manager, it is part of AIX operating system and allows to manage the installation and update of AIX on one or more machines. Since VIOS is based on AIX, NIM is also capable of installing VIOS via the network.

While most AIX customers have one or more NIM server, IBM i and linux customers are less likely to have one. NIM itself does not need a license but it runs on an AIX LPAR that must be licensed.

The HMC can act as a NIM server to install VIOS for customers who don't have (or don't want to use) a NIM server.

The VIOS network installation (with a NIM server or an HMC) performs those operations:
* The VIOS LPAR will PING the server (NIM or HMC) to verify connectivity
* The VIOS LPAR issues a BOOTP request to the server (NIM or HMC) to know which file to download on the next step
* The VIOS LPAR downloads the VIOS kernel and a RAMdisk image using the TFTP protocol
* The VIOS LPAR mounts several directories using NFS
* The VIOS LPAR performs the VIOS installation over NFS
* The VIOS LPAR reboots to its newly installed disk

There are 4 different protocols used during the installation. If the VIOS and the server (NIM or HMC) are on the same VLAN, there is no problem, if there is a router or a firewall between them, things can get more difficult:
* PING and TFTP are very easy to route and enable in a firewall
* NFS is very easy to route but quite painful for firewalls, there multiple ports involved, most of them dynamically negotiated. Fortunately, most firewalls handle it pretty well nowadays.
* BOOTP can be a real problem, there is a broadcast packet involved. Some routers and firewalls are simply unable to handle those packets.

If possible, have your VIOS and HMC on the same VLAN, it will make your life much easier.

# VIOS installation and network link aggregation

Another possible caveat is with link aggregation: it is common to have link aggregation on your VIOS for failover and load balancing. While this is a good idea when you're in production, this can lead to multiple failures during installation. To make things even more complicated, protocols have differents names with different switch vendors. Here are the different aggregation methods and there protential problems and workaround:
* **Teaming** This is the simplest aggregation method, it requires no configuration and the switch. This is not supported on the VIOS, don't use it, it will probably work but this will break the Enhanced+ GUI feature of the HMC, just don't do it.
* **Standard** Also known as Etherchannel, this is the oldest aggregation method supported. In this mode, the ports will always come up but the switch might think the aggregation is up and send packets to multiple adapters, since during installation, the VIOS doesn't do any aggregation, thinks might work (if you're very lucky) or partially (ping and bootp might work but not tftp) which might make you question your sanity or not at all. The workaround here is to disable all the ports but the one you're using for installation on the switch or remove the aggregation configuration on the switch for the installation.
* **802.3ad** Also known as LACP, this is the most recent and standard aggregation method supported. There are two modes (but some switches only handle active mode):
  * **Active** The switch port will only come up if there is a successful 802.3ad negotiation. Since the VIOS will not do any negotiation during installation, nothing will work. The workaround is to remove the aggregation altogether during installation.
  * **Passive** The switch port will come up as a standard port if the server doesn't do a 802.3ad negotiation but does an aggregation if the server does the negotiation. This is the ideal setting since you will be able to install the VIOS without having to do any change on the switch.

# Getting VIOS installation image

VIOS installation image can be downloaded from the [Entitled Systems Support (ESS)](https://www.ibm.com/eserver/ess) website. You will need an IBM ID and have a customer number with a POWER server with PowerVM registered with your ID.

Next click on "My Entitled Software", then "Software Downloads". In Category, choose "AIX", "IBM i" or "Linux" and click on the magnifying glass. This should populate the list of products, check 5765-VE3 (PowerVM Enterprise Edition V3) and click on Continue. On the next screen there will probably be multiple products. One of them will be named IBM PowerVM V3 / VIOS" followed by the version of vios available at the time. Click on the "packages" link on this line which expand the available packages. You can either download the 2 DVD images or the single Flash image, the HMC is able to use either. After accepting the license, you'll be able to download the selected file(s).

Those files need to be placed on a machine the HMC can copy the files from using NFS or sftp (ssh). A third option is to write the Flash image to a USB stick and put that USB stick in the HMC.

# Importing the VIOS image in the HMC repository

Before installing the VIOS, the images need to be imported in the HMC repository, this will copy the images on the HMC disks and make them usable for installation.

To import a file using sftp, use the following command:
<pre>cpviosimg -r sftp -n <i>vios31410</i> -f <i>Virtual_IO_Server_Base_Install_3.1.4.10_Flash_122022_LCD8250311.iso</i> -h <i>server_ip</i> -u <i>user</i> --passwd <i>password</i> -d <i>/path/to/image</i></pre>

With the following parameters:
- **vios31410** is the name the repository entry will have. Try to choose a meaningfull name.
- **Virtual_IO_Server_Base_Install_3.1.4.10_Flash_122022_LCD8250311.iso** is the name of the image file downloaded from ESS. If you downloaded the DVD images, you must specify the name of both files separated with a coma, the first image (1 of 2) must be specified first.
- **server_ip** is the ip or hostname of the sftp server where the images can be found.
- **user** is the username used to connect to the sftp server.
- **password** is the password used to connect to the sftp server.
- **/path/to/image** is the path of the directory containing the image on the sftp server.

To import a file using NFS, use the following command:
<pre>cpviosimg -r nfs -n <i>vios31410</i> -h <i>server_ip</i> -l <i>/path/to/image</i> -f <i>Virtual_IO_Server_Base_Install_3.1.4.10_Flash_122022_LCD8250311.iso</i></pre>

With the following parameters:
- **vios31410** is the name the repository entry will have. Try to choose a meaningfull name.
- **Virtual_IO_Server_Base_Install_3.1.4.10_Flash_122022_LCD8250311.iso** is the name of the image file downloaded from ESS. If you downloaded the DVD images, you must specify the name of both files separated with a coma, the first image (1 of 2) must be specified first.
- **server_ip** is the ip or hostname of the nfs server where the images can be found.
- **/path/to/image** is the path of the directory containing the image on the nfs server.

To import an image from a USB stick, after writing the image to the USB stick (typically on a personnal computer) and connecting it to the HMC USB port, run the following command:

<pre>lsmediadev</pre>

This will list the names of the devices attached to the HMC, for example /dev/sdb1.

Then mount the usb device with the following command:

<pre>mount /dev/sdb1</pre>

Finally import the image with the following command:

<pre>vpviosimg -r usb -n <i>vios31410</i> --device /dev/sdb1</pre>
Where **vios31410** is the name the repository entry will have.

# A word about secure boot

The network installation process is incompatible with secure boot. This means that, before starting the installation (even to get the ethernet ports MAC addresses), secure boot needs to be disabled. To disable secure boot, use the following command:
<pre>chsyscfg   -r lpar -m <i>power_server</i> -i "lpar_id=<i>id</i>,secure_boot=0"</pre>
With the following parameters:
- **power_server** is the name of the POWER server.
- **id** is the id of the VIOS LPAR

Once the VIOS installation is completed, secure boot can be enabled with the following command:
<pre>chsyscfg   -r lpar -m <i>power_server</i> -i "lpar_id=<i>id</i>,secure_boot=2"</pre>

# Identifying the ethernet ports MAC addresses

If the VIOS has more than one ethernet port (which is probably the case) and you want a fully unattended install, you need to pass the MAC address of the ethernet port to use.

To list the ethernet ports and their MAC addresses for a VIOS LPAR, run the following command:

<pre>lpar_netboot -M -n -t ent -A <i>partition_name</i> <i>profile_name</i> <i>power_name</i></pre>
With the following parameters:
- **partition_name** the name of the VIOS LPAR.
- **profile_name** the name of the VIOS LPAR profile.
- **power_name** the name of the POWER server.

This will output the following information:
<pre># Connecting to XXXXXXXXXX.
# Connected
# Checking for power off.
# Power off complete.
# Power on XXXXXXXX to Open Firmware.
# Power on complete.
# Getting adapter location codes.
# Type   Location Code   MAC Address     Full Path Name  Ping Result     Device Type
ent U9105.22A.XXXXXXX-V1-C2-T0 621072d17102 /vdevice/l-lan@30000002 n/a virtual
ent U9105.22A.XXXXXXX-V1-C3-T0 621072d17103 /vdevice/l-lan@30000003 n/a virtual
ent U9105.22A.XXXXXXX-V1-C4-T0 621072d17104 /vdevice/l-lan@30000004 n/a virtual
ent U78DA.ND0.WZS013T-P0-C10-T0 b83fd22b9a04 /pci@800000020000018/ethernet@0 n/a physical
ent U78DA.ND0.WZS013T-P0-C10-T1 b83fd22b9a05 /pci@800000020000018/ethernet@0,1 n/a physical</pre>

Each of the last 5 lines describe an ethernet port, the first three are virtual ethernet ports, the last 2 are the 2 ports of a physical adapter.
The second column indicates the location of the port and the third column contains the MAC address of the port.

In this example, if we want to use the first port of the card in slot C10 (address C10-T0), the MAC address will be b83fd22b9a04.

# Installing the VIOS (finally!)

The command to install a VIOS from the HMC is `installios`. If used without parameter, the command will start a wizard prompting you for LPAR and network information to perform installation.

To perform an unattended install, use the command like this:

<pre>installios -s <i>power_name</i> -p <i>partition_name</i> -r <i>profile_name</i> -i <i>vios_ip</i> -d <i>'/extra/viosimages/vios31410/dvdimage.v1.iso'</i> -g <i>gateway_ip</i> -S <i>netmask</i> -m <i>b8:3f:d2:2b:9a:04</i></pre>
With the following parameters:
- **power_name** the name of the POWER server.
- **partition_name** the name of the VIOS LPAR.
- **profile_name** the name of the VIOS LPAR profile.
- **vios_ip** the ip address for the VIOS.
- **'/extra/viosimages/vios31410/dvdimage.v1.iso'** the path to the image on the HMC filesystem. Replace vios31410 with the name of the repository entry.
- **gateway_ip** the default gateway for the VIOS.
- **netmask** the subnet mask for the VIOS.
- **b8:3f:d2:2b:9a:04** the MAC address of the VIOS ethernet port to use for installation. The MAC is found in the previous section with the lpar_netboot command. Add a `:` every othe digit.

If needed, you can add the following parameters:
- <strong>-V <i>vlan_id</i></strong> to specify a VLAN ID for the VIOS ethernet port. By default, the VIOS will configure its IP address without VLAN tagging. If you need to use a tagged VLAN during installation, you must specify it with this option.
- <strong>-E <i>bosinst_pvid</i></strong> to specify the VIOS disk onto which the installation will be done. The bosinst_pvid can be found using the `lspv` command on the VIOS, which can be challenging when installing the VIOS for the first time. I recommend making sure the VIOS only see its installation disk during installation (by removing any SAN mapping if using vscsi or shared storage pools, you shouldn't have any if using NPIV).
- **-n** to instruct the installer not to configure the ip address after installation. VIOS IP configuration will probably need to be  modified later (to use Shared Ethernet Adapter and/or link aggregation) so this option will save you one command later.

The installios command is pretty verbose and will keep you updated on the installation process. When the command completes, the VIOS should be up and running.

That's all for this post. We'll see how to finish the VIOS customization next time.