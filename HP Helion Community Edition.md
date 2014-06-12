HP Helion OpenStack Community Edition (Multi-node Baremetal)
=========

Notes taken during setup of HP Helion OpenStack (Community Edition).

Introduction
--
[HP Helion OpenStack](https://helion.hpwsportal.com/) is an OpenStack distribution that utilizes the TripleO program to install, upgrade and operate the cloud environment.

TripleO (OpenStack on OpenStack) uses OpenStack to deploy OpenStack on hardware servers (bare metal). Since OpenStack has ability to provision machines (eg: deploy Operating System to VM and configure its storage and networking), TripleO leverages these tools (nova, neutron, glance, heat etc) to deploy the actual hardware infrastructure hosting the OpenStack cloud. Nova with [Ironic](https://wiki.openstack.org/wiki/Ironic) provide the ability to provision software images onto real hardware. [Heat](https://wiki.openstack.org/wiki/Heat) provides the orchestration engine to launch the various OpenStack services based on a template file. For example, we can provide a Heat template which describes an OpenStack deployment that composes of 2 nodes swift, 3 nodes KVM compute and a single OpenStack controller. With TripleO, we are re-using the same OpenStack components and tools (nova, glance & heat) to deploy OpenStack onto hardware. You can find out more from this [TripleO Lightning Talk].

##Hardware Requirement

We will be using HP ProLiant DL/SL/BL/ML servers with iLO3/iLO4. Each server is recommended to have the following requirement :
 - 12 x CPU Cores
 - 32 GB RAM
 - 2 TB local disk (can be multiple disk in raid using HP Smart Array)
 - 1x 10 GB Nic with PXE boot capability

###Preparing the Hardware

- Ensure the BIOS has the current date and time
- Set the server to pxe boot from the first network card (eth0)
- Ensure an iLO admin account is created that is capable of power on/off function

Collect the following information for each server:
- mac address of first network card (eth0) that will take part in the pxe boot
- iLO address
- iLO admin account name and password
- IP address of iLO

##Deployment Overview

In this deployment, we will setup a 7-nodes baremetal HP Helion OpenStack Community Edition. HP Helion OpenStack Community  Edition can scale up to 30 compute nodes.
- 1x undercloud all-in-one OpenStack for baremetal deployment
- 1x overcloud controller node
- 2x overcloud swift nodes
- 3x overcloud compute nodes (KVM)

Additionally, we need another server to host the **seed** VM. This server (the seed host) must have the following:
 - 8 GB RAM
 - 100 GB Disk
 - OS: Ubuntu 13.10 or 14.04, Debian/Testing (Jessie)
 

![alt text](http://h06af3c3bd24692eee027972ac3415ae2.cdn.hpcloudsvc.com/helion-CE-deploy-2.png "HP Helion OpenStack 7 Nodes")


###Network Setup
As shown in the diagram, there are two physical networks. The first network is for the IPMI traffic that allows the **seed** VM and **undercloud** to power on/off the bare metal servers.
 
The other network is for the OpenStack public network (default : 192.0.2.0/24). This is the "public" network for our OpenStack API access network, Horizon web portal and the floating IP addresses for the VMs that are deployed on the **overcloud**. For this simple setup, we are also using this network for the pxe booting/iscsi copy process performed by the **seed** and **undercloud**.
To bridge the network between the **seed** VM and bare metal servers, an openvswitch bridge called _brbm_ is created by the installer and attached to a physical network port of the seed host (default eth0).


###Theory of Operation
Install the supported OS (eg: Ubuntu 13.10/14.04) on the seed host. The first command of the installer script (`hp_ced_start_seed.sh`) will create the **seed** VM and bridge it to the physical network port on the seed host. The **seed VM** is a hLinux VM with Keystone, Glance (to store the undercloud image), Nova, Ironic, Heat and Nuetron. It is an all-in-one OpenStack for the purpose of deploying the **undercloud** node.

Once the **seed** VM is booted and ready, we issue a second command (`hp_ced_installer.sh`) in the **seed** VM to start the deployment of **undercloud** and **overcloud**. This process will start by uploading the **undercloud** image to glance repository. This is followed by HEAT deploying the **undercloud** image to a bare metal server. All *overcloud* and *undercloud* servers will run HP hLinux. The first entry from the baremetal file, _baremetal.csv_ will be selected and it will be powered on via IPMI. This server then proceed to pxe boot over the network. After the server is booted up, it will use iscsi to mount the **undercloud** image from the **seed** VM and write it to the local disk. Finally, the server will reboot to local disk to become the **undercloud** node.

After the reboot, the **undercloud** node will configure itself to be a functional OpenStack cloud with Nova, Ironic, Horizon, Heat, Glance and Neutron. This is followed by uploading of **overcloud-controller**, **overcloud-swift**, **overcloud-compute** images to the **undercloud** Glance. An overcloud heat template will then be used to drive the deployment of the **overcloud**. This deployment will target the remaining entries from _baremetal.csv_. All the servers follow a similar process, where they will be powered up, pxe boot and mount the image from the **undercloud** using iscsi and write it to the local disk.


##Preparing the Seed host

Verify that:
- root has ssh public/private key (use _ssh-keygen_ to generate one)
- Seed host is capable of running kvm (use _virt-host-validate_ to verify)
- bridging interface (eth0 in our example) of the seed host need to have a valid IP
- seed host can reach the internet (ping 8.8.8.8 to validate) This is needed if there are updated packages to download. [optional]


##Starting the Seed VM
Download HP Helion OpenStack Community Edition at https://helion.hpwsportal.com

Next, we need to copy the installer into the seed host and extract the content into the root account's home directory.

####Extract the installer to _/root_ directory
```sh
## issue this command in the seed host
cd /root
tar xzvfp hp_helion_openstack_community_baremetal.tgz
```
####Start creating the **seed** VM
```bash
## issue this command in the seed host
bash -x /root/tripleo/tripleo-incubator/scripts/hp_ced_start_seed.sh
```
####Wait for the script to finish
```
+ echo 'Thu Jun 12 14:57:00 SGT 2014 --- completed setup seed'
Thu Jun 12 14:57:00 SGT 2014 --- completed setup seed
                               - 
```
####Validate that **seed** VM is running
```sh
## issue this command in the seed host
virsh list


Id    Name                           State
----------------------------------------------------
 8     seed                           running
```

####Validating the bridging interface
The seed host needs to bridge the **seed VM** to the physical network so that it could reach all the bare metal servers. Without this bridging of *brbm* bridge to a physical interface *eth0*, the bare metal servers cannot pxe boot and install the undercloud/overcloud.

```sh
## issue this command in the seed host
ovs-vsctl show
ce83871a-9a58-4df1-96cf-7143b5df1c91
    Bridge brbm
        Port "eth0"
            Interface "eth0"
        Port brbm
            Interface brbm
                type: internal
    ovs_version: "1.10.2"

# noticed that the seed host eth0 is connected to the bridge brbm
```
## Deploying the UnderCloud and OverCloud

### Login to the Seed VM
The seed VM has two network interfaces, the first port, eth0 is connected to the openvswith bridge *brbm* with the IP address 192.168.122.x, while the second port eth1 is connected to the 

```sh
## issue this command in the seed host
ssh 192.0.2.1
```
Once you are in the **seed** VM, you will see the _root@hLInux_ prompt.


you can monitor the OpenStack configuration of this **seed** VM by using the `tail -f` command. You should see that the _os-refresh-config_ process. Once it is done, you will see *Completed phase migration* in the *os-collect-config.log* file.

```sh
## issue this command in the seed VM
tail -10f /var/log/upstart/os-collect-config.log
```




### ILO/ipmi verification
Before we start deploying the **undercloud** and **overcloud** to the bare metal servers, let's ensure that we can power control these servers using the **ipmitool**. 

Issue the following commands in the **seed VM**. You should see an output that shows that either the server is powered on or powered down. This test ensure that you have network access to the iLO network and the right iLO credential.

```sh
## issue this command in the seed VM
/usr/bin/ipmitool -I lanplus -H <ilo IP> -U <ilo admin user> -P <password> power status
```

###Bare Metal Servers enrolment 
Create a file **/root/baremetal.csv** in the **seed** VM. The format is as follows:

```
<pxe nic mac_address>,<ilo admin>,<ilopassword>,<iloipaddress>,<#cpus>,<memory_MB>,<diskspace_GB>
```
An example of _baremetal.csv_ :

```
     78:e7:d1:22:5d:58,admin,password,192.168.11.1,12,32768,2048
     78:e7:d1:22:5d:10,admin,password,192.168.11.5,12,32768,2048
     78:e7:d1:22:52:90,admin,password,192.168.11.3,12,32768,2048
     78:e7:d1:22:5d:c0,admin,password,192.168.11.2,12,32768,2048
     78:e7:d1:22:5d:a8,admin,password,192.168.11.4,12,32768,2048
     78:e7:d1:22:52:9b,admin,password,192.168.11.6,12,32768,2048
     78:e7:d1:22:52:9e,admin,password,192.168.11.7,12,32768,2048

```
Take note that _diskspace_GB_ needs to be at least 512 GB and cannot be a value exceeding the physical disk capacity. Each entry in this file denotes a single bare metal server. The first entry will be selected for the **undercloud**. The remaining entries will be used for **overcloud**. From these remaining servers, 2 will be selected for **overcloud swift**, 1 will be selected for **overcloud controller** and the remaining will be **overcloud compute** nodes.

###Start Undercloud and OverCloud Deployment
From the **seed** VM, issue the following command as root.
Go take a coffee.
```sh
## issue this command in the seed VM
bash -x /root/tripleo/tripleo-incubator/scripts/hp_ced_installer.sh
```

####Successful setup
This is the output we'll see once the demo VM is created and assigned a floating IP in the **overcloud**.
```
+ echo 'HP - demo VM complete - Thu Jun 12 07:59:47 UTC 2014'
HP - demo VM complete - Thu Jun 12 07:59:47 UTC 2014
++ date
+ echo 'HP - completing - Thu Jun 12 07:59:47 UTC 2014'
HP - completing - Thu Jun 12 07:59:47 UTC 2014
++ date
+ echo 'HP - completed - Thu Jun 12 07:59:47 UTC 2014'
HP - completed - Thu Jun 12 07:59:47 UTC 2014


```
####login to the Undercloud
From the **seed** VM:
```sh
## issue this command in the seed VM
ssh heat-admin@192.0.2.2
```
####Status of Overcloud Deployment
From the **undercloud**, you can find out the IP address of the **overcloud** controller, hence you can access the horizon portal with this IP.
```sh
## issue this command in the undercloud node
source  stackrc && nova list

+-----------------------------+---------------------------------+--------+------------+-------------+---------------------+
| ID                          | Name                            | Status | Task State | Power State | Networks            |
+-----------------------------+---------------------------------+--------+------------+-------------+---------------------+
| c68f764b-ac7e-4bd4-be74-5c7 | overcloud-NovaCompute0vdwkxviq  | ACTIVE | -          | Running     | ctlplane=192.0.2.23 |
| 1459cdae-da0d-4912-d20-5af0 | overcloud-NovaCompute1-til55p2  | ACTIVE | -          | Running     | ctlplane=192.0.2.24 |
| 932545b4-8a4e-4dbf-bafd487d | overcloudNovaCompute2-fnipzw7p  | ACTIVE | -          | Running     | ctlplane=192.0.2.28 |
| 240673a8-235b-49ba-8cacd0ce | overcloud-NovaComputewpwpxepkm  | ACTIVE | -          | Running     | ctlplane=192.0.2.27 |
| f0546168-b6ac-c21d235387a2d | overcloud-NovaComputewvvof6sia  | ACTIVE | -          | Running     | ctlplane=192.0.2.29 |
| 93f3f7aa-13e1-4231-9364-e1c | overcloud-SwiftStorage0-vndttd4 | ACTIVE | -          | Running     | ctlplane=192.0.2.25 |
| 36a3b14a-be17-4bdf-a2ff-0e7 | overcloud-SwiftStorage1-ugsgnl5 | ACTIVE | -          | Running     | ctlplane=192.0.2.30 |
| 2ace2c68-b88f-47b4-b3a2e850 | overcloud-controller0-7jg7kkh   | ACTIVE | -          | Running     | ctlplane=192.0.2.26 |
+-----------------------------+---------------------------------+--------+------------+-------------+---------------------+

```


[TripleO Lightning Talk]:http://www.slideshare.net/cmsj1/tripleo-lightning-talk-29149308


