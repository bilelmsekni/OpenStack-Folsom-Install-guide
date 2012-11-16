==========================================================
  OpenStack Folsom Install Guide
==========================================================

:Version: 2.5
:Source: https://github.com/mseknibilel/OpenStack-Folsom-Install-guide
:Keywords: Multi node OpenStack, Folsom, Quantum, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM, Ubuntu Server 12.10 (64 bits).

Authors
==========

Copyright (C) Bilel Msekni <bilel.msekni@telecom-sudparis.eu>

Contributors
==========

* Roy Sowa <Roy.Sowa@ssc-spc.gc.ca>
* Marco Consonni <marco_consonni@hp.com>
* Dennis E Miyoshi <dennis.miyoshi@hp.com>
* Houssem Medhioub <houssem.medhioub@it-sudparis.eu>
* Djamal Zeghlache <djamal.zeghlache@telecom-sudparis.eu>

Wana contribute ? Read the guide, send your contribution and get your name listed ;)

Table of Contents
=================

::

  0. What is it?
  1. Requirements
  2. Getting Ready
  3. Keystone 
  4. Glance
  5. OpenVSwitch
  6. Quantum
  7. Nova
  8. Cinder
  9. Horizon
  10. Adding a compute node
  11. Start your first VM
  12. Licencing
  13. Contacts
  14. Acknowledgement
  15. To do

0. What is it?
==============

OpenStack Folsom Install Guide is an easy and tested way to create your own OpenStack plateform. 

Version 2.5

Status: stable 


1. Requirements
====================

:Node Role: NICs
:Control Node: eth0 (192.168.100.232), eth1 (100.10.10.232), eth2(Must be internet connected)
:Compute Node: eth0 (192.168.100.233), eth1 (100.10.10.233)

**Note 1:** If you don't have 2 NICs on controller node, you can check the milestone branch for 2 NIC installation.

**Note 2:** If you are not interrested in Quantum, you can also use this guide but you must follow the nova section found `here <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/Tricks%26Ideas/install_nova-network.rst>`_ instead of the one written in this guide.

**Note 3:** This is my current network architecture, you can add as many compute node as you wish.

.. image:: http://i.imgur.com/Ueh5Z.jpg

2. Getting Ready
===============

2.1. Preparing Ubuntu 12.10
-----------------

* After you install Ubuntu 12.10 Server 64bits, Go to the sudo mode and don't leave it until the end of this guide::

   sudo su

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

2.2.Networking
------------

* Both NICs on the controller node need internet access::

   #For OpenStack management
   auto eth0
   iface eth0 inet static
   address 192.168.100.232
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

   #Not internet connected(used for VM configuration)
   auto eth1
   iface eth1 inet static
   address 100.10.10.232
   netmask 255.255.255.0

   #Will generously be bridged to the br-ex later:
   auto eth2
   iface eth2 inet manual
   up ifconfig $IFACE 0.0.0.0 up
   up ip link set $IFACE promisc on
   down ip link set $IFACE promisc off
   down ifconfig $IFACE down

2.3. MySQL & RabbitMQ
------------

* Install MySQL::

   apt-get install mysql-server python-mysqldb

* Configure mysql to accept all incoming requests::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* Install RabbitMQ::

   apt-get install rabbitmq-server 

2.4. Node synchronization
------------------

* Install other services::

   apt-get install ntp

* Configure the NTP server to synchronize between your compute nodes and the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
   service ntp restart  

2.5. Others
-------------------
* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   nano /etc/sysctl.conf
   # Uncomment net.ipv4.ip_forward=1, to save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

3. Keystone
=====================================================================

This is how we install OpenStack's identity service:

* Start by the keystone packages::

   apt-get install keystone

* Create a new MySQL database for keystone::

   mysql -u root -p
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   quit;

* Adapt the connection attribute in the /etc/keystone/keystone.conf to the new database::

   connection = mysql://keystoneUser:keystonePass@192.168.100.232/keystone

* Restart the identity service then synchronize the database::

   service keystone restart
   keystone-manage db_sync

* Fill up the keystone database using the two scripts available in the `Scripts folder <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/tree/master/Keystone_Scripts>`_ of this git repository. Beware that you MUST comment every part related to Quantum if you don't intend to install it otherwise you will have trouble with your dashboard later::

   #Modify the HOST_IP variable before executing the scripts

   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* Create a simple credential file and load it so you won't be bothered later::

   nano creds
   #Paste the following:
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://192.168.100.232:5000/v2.0/"
   # Load it:
   source creds

* To test Keystone, we use a simple curl request::

   apt-get install curl openssl
   curl http://192.168.100.232:35357/v2.0/endpoints -H 'x-auth-token: ADMIN'

4. Glance
=====================================================================

* After installing Keystone, we continue with installing image storage service a.k.a Glance::

   apt-get install glance

* Create a new MySQL database for Glance::

   mysql -u root -p
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
   quit;

* Update /etc/glance/glance-api-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update the /etc/glance/glance-registry-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update /etc/glance/glance-api.conf with::

   sql_connection = mysql://glanceUser:glancePass@192.168.100.232/glance

* And::

   [paste_deploy]
   flavor = keystone

* Update the /etc/glance/glance-registry.conf with::

   sql_connection = mysql://glanceUser:glancePass@192.168.100.232/glance

* And::

   [paste_deploy]
   flavor = keystone

* Restart the glance-api and glance-registry services::

   service glance-api restart; service glance-registry restart

* Synchronize the glance database::

   glance-manage db_sync

* Restart the services again to take into account the new modifications::

   service glance-registry restart; service glance-api restart

* To test Glance's well installation, we upload a new image to the store. Start by downloading the cirros cloud image to your node and then uploading it to Glance::

   mkdir images
   cd images
   wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
   glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 < cirros-0.3.0-x86_64-disk.img

* Now list the images to see what you have just uploaded::

   glance image-list

5. OpenVSwitch
=====================================================================

* Install the openVSwitch::

   apt-get install -y openvswitch-switch

* Create the bridges::

   #br-ex is used to make to VM accessible from the internet
   ovs-vsctl add-br br-ex
   ovs-vsctl add-port br-ex eth2
   
   #br-int is used for VM integration 
   ovs-vsctl add-br br-int

   #Used for VM configuration
   ovs-vsctl add-br br-eth1
   ovs-vsctl add-port br-eth1 eth1

6. Quantum
=====================================================================

Quantum literaly eliminated the network overhead i used to deal with during the nova-network era.

* Install the Quantum server::

   apt-get install quantum-server quantum-plugin-openvswitch quantum-plugin-openvswitch-agent

* Create a database::

   mysql -u root -p
   CREATE DATABASE quantum;
   GRANT ALL ON quantum.* TO 'quantumUser'@'%' IDENTIFIED BY 'quantumPass';
   quit; 

* Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@192.168.100.232/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type=vlan
   network_vlan_ranges = physnet1:1:4094
   bridge_mappings = physnet1:br-eth1

* Install quantum DHCP and l3 agents::

   apt-get -y install quantum-dhcp-agent quantum-l3-agent

* Edit /etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* In addition, update the /etc/quantum/l3_agent.ini::

   auth_url = http://192.168.100.232:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* Restart all the services::

   service quantum-server restart
   service quantum-dhcp-agent restart
   service quantum-l3-agent restart
   service quantum-plugin-openvswitch-agent restart

7. Nova
=================

* Start by installing nova components::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy

* Prepare a Mysql database for Nova::

   mysql -u root -p
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
   quit;

* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova

* Modify the /etc/nova/nova.conf like this::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   scheduler_driver=nova.scheduler.simple.SimpleScheduler
   s3_host=192.168.100.232
   ec2_host=192.168.100.232
   ec2_dmz_host=192.168.100.232
   rabbit_host=192.168.100.232
   cc_host=192.168.100.232
   metadata_host=192.168.100.232
   metadata_listen=0.0.0.0
   nova_url=http://192.168.100.232:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@192.168.100.232/nova
   ec2_url=http://192.168.100.232:8773/services/Cloud 
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://192.168.100.232:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=192.168.100.232:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.232:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=192.168.100.232
   vncserver_listen=0.0.0.0 

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://192.168.100.232:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://192.168.100.232:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* Synchronize your database::

   nova-manage db sync

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

8. Cinder
=================

Although Cinder is a replacement of the old nova-volume service, its installation is now a seperated from the nova install process.

* Install the required packages::

   apt-get install cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* Configure the iscsi services::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* Restart the services::
   
   service iscsitarget start
   service open-iscsi start

* Prepare a Mysql database for Cinder::

   mysql -u root -p
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
   quit;

* Configure /etc/cinder/api-paste.ini like the following::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 192.168.100.232
   service_port = 5000
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* Edit the /etc/cinder/cinder.conf to::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@192.168.100.232/cinder
   api_paste_confg = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

* Then, synchronize your database::

   cinder-manage db sync

* Finally, don't forget to create a volumegroup and name it cinder-volumes::

   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2
   #Type in the followings:
   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

* Proceed to create the physical volume then the volume group::

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

**Note:** Beware that this volume group gets lost after a system reboot. (Click `Here <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/Tricks%26Ideas/load_volume_group_after_system_reboot.rst>`_ to know how to load it after a reboot) 

* Restart the cinder services::

   service cinder-volume restart
   service cinder-api restart

9. Horizon
============

* To install horizon, proceed like this ::

   apt-get install openstack-dashboard memcached


* If you don't like the OpenStack ubuntu theme, you can disabled it and go back to the default look::

   nano /etc/openstack-dashboard/local_settings.py
   #Comment these lines
   #Enable the Ubuntu theme if it is present.
   #try:
   #    from ubuntu_theme import *
   #except ImportError:
   #    pass

* Reload Apache and memcached::

   service apache2 restart; service memcached restart

You can now access your OpenStack **192.168.100.232/horizon** with credentials **admin:admin_pass**.

**Note:** A reboot might be needed for a successful login

10. Adding a compute node
=========================

10.1. Preparing the Node
------------------

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

* Install ntp service::

   apt-get install ntp

* Configure the NTP server to follow the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server 192.168.100.232/g' /etc/ntp.conf
   service ntp restart  

* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   nano /etc/sysctl.conf
   # Uncomment net.ipv4.ip_forward=1, to save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

10.2.Networking
------------

* It's recommended to have two NICs but only one needs to be internet connected::
   
   # Connected to the internet
   auto eth0
   iface eth0 inet static
   address 192.168.100.233
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

   # Not connected to internet
   auto eth1
   iface eth1 inet static
   address 100.10.10.233
   netmask 255.255.255.0

10.3 KVM
------------------

* KVM is needed as the hypervisor that will be used to create virtual machines. Before you install KVM, make sure that your hardware enables virtualization::

   apt-get install cpu-checker
   kvm-ok

* Normally you would get a good response. Now, move to install kvm and configure it::

   apt-get install -y kvm libvirt-bin pm-utils

* Edit the cgroup_device_acl array in the /etc/libvirt/qemu.conf file to::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* Delete default virtual bridge ::

   virsh net-destroy default
   virsh net-undefine default

* Enable live migration by updating /etc/libvirt/libvirtd.conf file::

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"

* Edit libvirtd_opts variable in /etc/init/libvirt-bin.conf file::

   env libvirtd_opts="-d -l"

* Edit /etc/default/libvirt-bin file ::

   libvirtd_opts="-d -l"

* Restart the libvirt service to load the new values::

   service libvirt-bin restart

10.4. OpenVSwitch
------------------

* Install the openVSwitch::

   apt-get install -y openvswitch-switch

* Create the bridges::

   #br-int will be used for VM integration	
   ovs-vsctl add-br br-int

   #br-eth1 will be used for VM configuration 
   ovs-vsctl add-br br-eth1
   ovs-vsctl add-port br-eth1 eth1

10.5. Quantum
------------------

We don't need to install the hole quantum server here, just the openVSwitch plugin's agent

* Install the Quantum openvswitch agent::

   apt-get -y install quantum-plugin-openvswitch-agent

* Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@192.168.100.232/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type=vlan
   network_vlan_ranges = physnet1:1:4094
   bridge_mappings = physnet1:br-eth1

* Make sure that your rabbitMQ IP in /etc/quantum/quantum.conf is set to the controller node::
   
   rabbit_host = 192.168.100.232

* Restart all the services::

   service quantum-plugin-openvswitch-agent restart

10.6. Nova
------------------

* Install nova's required components for the compute node::

   apt-get install nova-api-metadata nova-compute-kvm

* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova

* Edit /etc/nova/nova-compute.conf file ::
   
   [DEFAULT]
   libvirt_type=kvm
   libvirt_ovs_bridge=br-int
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   libvirt_use_virtio_for_bridges=True

* Modify the /etc/nova/nova.conf like this::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   scheduler_driver=nova.scheduler.simple.SimpleScheduler
   s3_host=192.168.100.232
   ec2_host=192.168.100.232
   ec2_dmz_host=192.168.100.232
   rabbit_host=192.168.100.232
   cc_host=192.168.100.232
   metadata_host=192.168.100.233
   metadata_listen=0.0.0.0
   nova_url=http://192.168.100.232:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@192.168.100.232/nova
   ec2_url=http://192.168.100.232:8773/services/Cloud 
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
   

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://192.168.100.232:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=192.168.100.232:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.232:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=192.168.100.233
   vncserver_listen=0.0.0.0 

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://192.168.100.232:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://192.168.100.232:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* Synchronize your database::

   nova-manage db sync

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

11. Your First VM
============

To start your first VM, we first need to create a new tenant, user, internal and external network. SSH to your controller node and perform the following.

* Create a new tenant ::

   keystone tenant-create --name project_one

* Create a new user and assign the member role to it in the new tenant (keystone role-list to get the appropriate id)::

   keystone user-create --name=user_one --pass=user_one --tenant-id $put_id_of_project_one --email=user_one@domain.com
   keystone user-role-add --tenant-id $put_id_of_project_one  --user-id $put_id_of_user_one --role-id $put_id_of_member_role

* Create a new network for the tenant::

   quantum net-create --tenant-id $put_id_of_project_one net_proj_one --provider:network_type vlan --provider:physical_network physnet1 --provider:segmentation_id 1024

* Create a new subnet inside the new tenant network::

   quantum subnet-create --tenant-id $put_id_of_project_one net_proj_one 10.10.10.0/24

* Create a router for the new tenant::

   quantum router-create --tenant-id $put_id_of_project_one router_proj_one

* Add the router to the subnet::

   quantum router-interface-add $put_router_proj_one_id_here $put_subnet_id_here

You can now start creating VMs but they will not be accessible from the internet. If you like them to be so, perform the following:

* Create your external network with the tenant id belonging to the service tenant (keystone tenant-list to get the appropriate id) ::

   quantum net-create --tenant-id $put_id_of_service_tenant ext_net --router:external=True

* Create a subnet containing your floating IPs::

   quantum subnet-create --tenant-id $put_id_of_service_tenant --gateway 192.168.100.1 ext_net 192.168.100.100/28 --enable_dhcp=False

* Set the router for the external network::

   quantum router-gateway-set $put_router_proj_one_id_here $put_id_of_ext_net_here

* update your br-ex::

   ip addr flush dev br-ex
   ip addr add 192.168.100.100/28 dev br-ex
   ip link set br-ex up

Unfortunatly, you can't use the dashboard to assign floating IPs to VMs so you need to get your hands a bit dirty to give your VM a public IP.

* Start by allocating a floating ip to the project one tenant::

   quantum floatingip-create --tenant-id $put_id_of_project_one ext_net

* pick the id of the port corresponding to your VM::

   quantum port-list

* Associate the floating IP to your VM::

   quantum floatingip-associate $put_id_floating_ip $put_id_vm_port

**This is it !**, You can now ping you VM and start administrating you OpenStack !

I Hope you enjoyed this guide, please if you have any feedbacks, don't hesitate.

12. Licensing
============

OpenStack Folsom Install Guide by Bilel Msekni is licensed under a Creative Commons Attribution 3.0 Unported License.

.. image:: http://i.imgur.com/4XWrp.png
To view a copy of this license, visit [ http://creativecommons.org/licenses/by/3.0/deed.en_US ].

13. Contacts
===========

Bilel Msekni: bilel.msekni@telecom-sudparis.eu

14. Acknowledgment
=================

This work has been based on:

* Emilien Macchi's Folsom guide [https://github.com/EmilienM/openstack-folsom-guide]
* OpenStack Documentation [http://docs.openstack.org/trunk/openstack-compute/install/apt/content/]
* OpenStack Quantum Install [http://docs.openstack.org/trunk/openstack-network/admin/content/ch_install.html]

15. To do
=======

This guide is just a startup. Your suggestions are always welcomed.

Some of this guide's needs might be:

* Define more Quantum configurations to cover all usecases possible see `here <http://docs.openstack.org/trunk/openstack-network/admin/content/use_cases.html>`_. 




