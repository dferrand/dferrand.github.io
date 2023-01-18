---
layout: post
title:  "Setup a POWER server using HMC CLI"
date:   2023-01-18 10:00:00 -0500
categories: power hmc cli
---

The common way to setup a POWER server using the HMC GUI (web interface). While it is the most intuitive way to do it, it can also be done using the HMC CLI, this has the following advantages:
- **Stability:** While the GUI is always changing, sometimes radically, the CLI is very stable, new features are introduced but in a forward compatible way.
- **Repeatability:** It is very easy to setup several identical servers with CLI, a simple find/replace or scripting will allow to repeat the process.
- **Conciseness:** For documentation purposes, the CLI commands will generally be much more concise than many screenshots from the GUI, it is also searchable.
- **Globalization:** The GUI is displayed in the web browser's language which is nice for the user but can be problemactic since your screenshots will be in one language only. On the other hand, the CLI has no translation, it is english only.

# Connecting POWER server to HMC

If you have a private HMC network where the HMC acts as a DHCP server (which is the recommended setup), the POWER server should automatically appear in the HMC as soon as it is connected the the private network and has power.

If the HMC is not the DHCP server, you must first setup the POWER management IP addresses in the ASMI interface and then create the connection in the HMC. The command varies slightly depending on wether the POWER server has a FSP (before POWER10 or E1080) or an eBMC (from POWER10 on, except E1080)

**Before POWER10 or E1080:** 
<pre>mksysconn --ip <i>ip_address</i></pre>

**From POWER10 on, except E1080:** 
<pre>mksysconn --ip <i>ip_address</i> --passwd <i>admin</i></pre>

Where *ip_address* is the IP address of the POWER server and *admin* the password of the `admin` eBMC user (admin by default).

If  you have dual HMC, you have to run this command on both HMCs.

The Connection should now appear in the `lssysconn -r all` output:

<pre>
hscroot@hmc:~> lssysconn -r all
resource_type=sys,type_model_serial_num=0000-BMC*10013019,sp_type=ebmc,ipaddr=10.0.130.19,user_name=admin,alt_ipaddr=unavailable,state=Pending Authentication - Password Updates Required,connection_error_code=Password expired 0806-0000-00000000,vmi_ipaddr=unavailable,vmi_state=unavailable
</pre>

This indicates that the connection is working but the default password needs to be changed. If this is the case, run the following command:

<pre>chsyspwd -m <i>0000-BMC*10013019</i> -t access --passwd <i>admin</i> --newpasswd '<i>new_password</i>'</pre>

Where _0000-BMC*10013019_ is the value for type_model_serial_num you get in the lssysconn output, *admin* is the current password and *new_password* the new password you want to assign to the POWER server. Keep this password in a safe place, you will need it to reconnect to an HMC.

If the password is not the default password, the lssysconn output will indicate a failed authentication state:

<pre>
hscroot@hmc:~> lssysconn -r all
resource_type=sys,type_model_serial_num=0000-BMC*10013019,sp_type=ebmc,ipaddr=10.0.130.19,user_name=admin,alt_ipaddr=unavailable,state=Failed Authentication,connection_error_code=Authentication failed  0243-0000-00000000,vmi_ipaddr=unavailable,vmi_state=unavailable
</pre>

In this case, run the following command:

**Before POWER10 or E1080:** 
<pre>chsyspwd -m <i>8286-41A*7812345</i> -t access --newpasswd '<i>password</i></pre>

**From POWER10 on, except E1080:** 
<pre>chsyspwd -m <i>0000-BMC*10013019</i> -u admin --newpasswd '<i>password</i>'</pre>

Where _8286-41A*7812345_ or _0000-BMC*10013019_ is the value for type_model_serial_num you get in the lssysconn output and _password_ the password of the POWER server.

After setting up the password, the lssysconn output should indicate a state of connected:
<pre>
hscroot@hmc:~> lssysconn -r all
resource_type=sys,type_model_serial_num=9105-22A*7812345,sp_type=ebmc,ipaddr=10.0.130.19,user_name=admin,alt_ipaddr=unavailable,state=Connected,vmi_ipaddr=unavailable,vmi_state=unavailable
</pre>

## Setting the VMI address(es)

This section only applies to eBMC systems (POWER10 on, except E1080).

eBMC system have another set of IP addresses used for communication between the HMC and the POWER server called VMI (Virtualization Management Interface) those IP will allow **one day** the HMC to manage a POWER server even if the eBMC is failing.

Those IP addresses are on the eBMC ports (eth0 and eth1) and can use DHCP or static address. Contrary to the eBMC addresses, those are not setup to use DHCP by default. You need to setup a static address or enable DHCP for the HMC to be able to manage the POWER server. If the HMC is not able to communicate with the VMI, the POWER server will show as No Connection in the HMC once it is powered on.

To enable DHCP:
<pre>chsysconn -m <i>9105-22A*7812345</i> -r vmi -o set -i eth0 -t ipv4dhcp</pre>

To set a static IP address:
<pre>chsysconn -m <i>9105-22A*7812345</i> -r vmi -o set -i eth0 -t static --ip <i>ip_address</i> -g <i>gateway_ip</i> --nm <i>network_mask</i></pre>

Where _9105-22A*7812345_ is the type, model and serial number of the POWER server, *ip_address* the VMI IP address to set, *gateway_ip* the default gateway and *network_mask* the network mask. This will set the configuration for the first eBMC ethernet port, to set the configuration for the second eBMC ethernet port, replace eth0 with eth1.

# Basic server setup

By default, the POWER system will be named after its type, model and serial number. To set a name of your choosing to the server, run the following command:
<pre>chsyscfg -r sys -m <i>9105-22A*7812345</i> -i new_name=<i>PWR_NAME</i></pre>

Where _9105-22A*7812345_ is the type, model and serial number of the POWER server and *PWR_NAME* the name of the server.

For the next step, the server needs to be in STANDBY state, first power off the server:
<pre>chsysstate -r sys -m <i>PWR_NAME</i> -o off --immed</pre>

The previous command will return immediately but the POWER server will need some time to complete its power off. Run the following command until the output is `Power Off`:
<pre>lssyscfg -r sys -m <i>PWR_NAME</i> -F state</pre>

Once the POWER server is powered off, run the following command to start it in standby mode:
<pre>chsysstate -r sys -m <i>PWR_NAME</i> -o onstandby</pre>

This command will also return immediately but the POWER server will need some time to reach standby state. Run the following command until the output is `Standby`:
<pre>lssyscfg -r sys -m <i>PWR_NAME</i> -F state</pre>

By default IBM generally ships POWER servers with one LPAR using all the system resources, the following command will delete all partitions, partition profiles and system profiles from the POWER server:
<pre>rstprofdata -l 4 -m <i>PWR_NAME</i></pre>

The following command will set the power off and on policies for the POWER server:
<pre>chsyscfg -r sys -m <i>PWR_NAME</i> -i "power_off_policy=1,power_on_lpar_start_policy=autostart"</pre>
This sets the power off policy to 1 which means the server will not power off when the last partition is powered off. If you want the server to power off, set this to 0.
The power on policy is set to autostart, which will automatically start partitions set to autostart when the POWER server is powered on. The other possible values are userinit and autorecovery.

# Collecting physical I/O adapter information

Before creating the Virtual I/O Servers (VIOSes) it is necessary to collect information about  the physical I/O adapters that will be assigned to the VIOSes. Run the following command to list the slots of the POWER server:
<pre>lshwres -r io --rsubtype slot -m <i>PWR_NAME</i> -F drc_name,drc_index,description</pre>

Here is an example output:
<pre>
U78DA.ND0.WZS013T-P0-C11,21010010,NVMe JBOF Card
U78DA.ND0.WZS013T-P0-C10,21010018,PCIe3 2 PORT 25/10 Gb NIC&ROCE SFP28 ADAPTER
U78DA.ND0.WZS013T-P2-C10,21030102,800GB NVMe Gen2 U.2 Slim SSD
U78DA.ND0.WZS013T-P2-C11,21040103,800GB NVMe Gen2 U.2 Slim SSD
U78DA.ND0.WZS013T-P2-C12,21050104,Empty slot
U78DA.ND0.WZS013T-P2-C13,21060105,Empty slot
U78DA.ND0.WZS013T-P0-T18,2101001A,Universal Serial Bus UHC Spec
U78DA.ND0.WZS013T-P0-C9,21010020,PCIe3 2-Port 16Gb FC Adapter
U78DA.ND0.WZS013T-P0-C8,21010021,NVMe JBOF Card
U78DA.ND0.WZS013T-P0-C7,21010028,PCIe3 2-Port 16Gb FC Adapter
U78DA.ND0.WZS013T-P1-C2,21030212,800GB NVMe Gen2 U.2 Slim SSD
U78DA.ND0.WZS013T-P1-C3,21040213,800GB NVMe Gen2 U.2 Slim SSD
U78DA.ND0.WZS013T-P1-C4,21050214,Empty slot
U78DA.ND0.WZS013T-P1-C5,21060215,Empty slot
U78DA.ND0.WZS013T-P0-C4,21010030,Empty slot
U78DA.ND0.WZS013T-P0-C3,21010038,Empty slot
U78DA.ND0.WZS013T-P0-C2,21010040,PCIe3 2-Port 16Gb FC Adapter
U78DA.ND0.WZS013T-P0-C1,21010041,PCIe3 2-Port 16Gb FC Adapter
U78DA.ND0.WZS013T-P0-C0,21010048,PCIe3 2 PORT 25/10 Gb NIC&ROCE SFP28 ADAPTER
</pre>

Each line represents a resource. A resource is generally a PCIe adapter or a NVMe drive. The first field is the address that will indicate the resource location, the second is the DRC index that will be used during the partition creation. The third  field is the description of the resource.

You need to decide which resource to assign to each VIOS and get the corresponding DRC indexes.

# Creating the VIOS partitions

The following command will create the first VIOS partition:
<pre>
mksyscfg -r lpar -m <i>PWR_NAME</i> -i "name=VIOS1,lpar_id=1,profile_name=default,lpar_env=vioserver,msp=1,time_ref=1,sync_curr_profile=1,min_mem=4096,desired_mem=16384,max_mem=24576,proc_mode=shared,min_procs=1,desired_procs=1,max_procs=4,min_proc_units=0.5,desired_proc_units=1,max_proc_units=4,uncap_weight=255,\"io_slots=21010018//1,21010020//1,21010028//1,21010040//1,21010041//1\",auto_start=1,max_virtual_slots=1000,sharing_mode=uncap"
</pre>

Here is the description of the parameters:
- **name** the name of the VIOS partition
- **lpar_id** the ID of the VIOS partition. There is no hard constraint but a rule of thumb is to create the VIOSes with the first IDs (starting at 1)
- **map** set to 1 if this VIOS is setup as a Mover Service Partition (in charge of transferring the content of memory during a Live Partition Mobility operation). Unless you have more than 2 VIOSes, it is generally a good idea to enable it for all VIOSes. Set to 0 to disable this functionnality for this VIOS.
- **time_ref** set to 1 if this VIOS is a time reference. The POWER hypervisor itself has a clock but is not able to use an NTP server, to keep its clock on time, it will use one or more partition as a time reference, those partition can and should use a NTP server. Since VIOSes are generally always on and can't be moved via Live Partition Mobility, they are ideal candidates as time references. A rule of thumb is to setup all VIOSes as time reference. Set to 0 to disable this functionnality for this VIOS.
- **sync_curr_profile** set to 1 to automatically update the partition profile when a Dynamic LPAR operation is performed. Generally keep this set to 1.
- **min_mem, desired_mem and max_mem** the minimum, desired and maximum memory sizes in megabytes for this VIOS.
- **proc_mode** the processor mode for the VIOS. Shared processor mode is generally the most flexible option. Set to `ded` to use dedicated processor mode.
- **min_procs, desired_procs, max_procs** the minimum, desired and maximum vCPU (if shared processor mode) or processors (if dedicated processor mode).
- **min_proc_units, desired_proc_units, max_proc_units** the minimum, desired and maximum processor entitlement. Only applies to shared processor mode.
- **uncap_weight** the weight of this VIOS when the hypervisor will dispatch available processor cycles above the entitlement. The rule of thumb is that VIOSes should never be processor starved so they should always have the highest weight (255). Only applies to shared processor mode.
- **io_slots** the list of physical resources assigned to this VIOS. Each entry is composed of the DRC index found in the previous section, the 1 after // indicates that the resource is required, meaning it can't be removed from the VIOS via Dynamic LPAR which is generally the case for VIOS resources, set to 0 instead to specify a resource as not required.
- **auto_start** set to 1 so the VIOS will automatically be started with the POWER server (if its power on policy is set to autostart or autorecovery). Set to 0 for not having the VIOS automatically start with the POWER server.
- **max_virtual_slots** the maximum number of virtual slots for the VIOS. Generally set this to a high value since changing this value requires a VIOS restart.
- **sharing_mode** defines if the partition is allowed to use more CPU cycles than its entitled capacity. A VIOS should generally be set to uncap. Set to cap if you want the VIOS to not be allowed to used more CPU cycles than its entitled capacity. Only applies to shared processor mode.

The second VIOS would be created with the following command:
<pre>
mksyscfg -r lpar -m <i>PWR_NAME</i> -i "name=VIOS2,lpar_id=2,profile_name=default,lpar_env=vioserver,msp=1,time_ref=1,sync_curr_profile=1,min_mem=4096,desired_mem=16384,max_mem=24576,proc_mode=shared,min_procs=1,desired_procs=1,max_procs=4,min_proc_units=0.5,desired_proc_units=1,max_proc_units=4,uncap_weight=255,\"io_slots=21010040//1,21010041//1,21010048//1,21030102//1,21040103//1\",auto_start=1,max_virtual_slots=1000,sharing_mode=uncap"
</pre>

That's all for this post. We'll see how to install the VIOS software from the HMC CLI.