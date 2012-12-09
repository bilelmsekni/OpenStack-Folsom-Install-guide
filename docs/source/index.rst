.. toctree::
   :maxdepth: 2

------------------------------------
Basic OpenStack Folsom Install Guide
------------------------------------



.. image:: http://i.imgur.com/VBJL6.png
   :height: 75


.. image:: http://i.imgur.com/GsjzH.png
   :height: 65



:Version: 1.0
:Source: https://github.com/nimbula/Basic-OpenStack-Folsom-Install-Guide
:Keywords: Multi node OpenStack, Folsom, Nova, Keystone, Glance, Horizon, Cinder, KVM, Ubuntu Server 12.10 (64-bit release).




Overview
===========

This guide focuses on providing step-by-step instruction to users who are interested in taking a bare-metal server installation to a fully functioning OpenStack cloud. We will avoid using scripts like TryStack and DevStack and will attempt to configure a “vanilla” OpenStack environment. The only scripts used in this tutorial are slight modifications of the existing keystone scripts available on the official OpenStack GitHub repo.
(https://github.com/openstack/keystone/blob/master/tools/sample_data.sh)

Who should read this guide
------------------------------

This guide is for the system administrator who is installing, configuring and managing the OpenStack “Folsom” cluster infrastructure. A reasonable level of familiarity with the following is assumed:

* The Unix command line

* Installing packages

* Basic networking concepts

Special thanks to
---------------------

`Emilien Macchi <http://www.linkedin.com/profile/view?id=128600871>`_ at eNovance for assistance in merging some portions of this text into the official OpenStack repository.

`Bilel Msekni <http://www.linkedin.com/profile/view?id=136237741>`_ from TELECOM SudParis for allowing me to fork sections of his OpenStack install guide and for the valuable suggestions and input.

Author
----------

`Zachary VanDuyn <http://www.linkedin.com/profile/view?id=132309630&trk=hb_tab_pro_top>`_
Technical Marketing Intern
`Nimbula <http://www.nimbula.com/>`_


Getting Started
=================
**Note:** This guide intentionally uses the 'nova-network' package instead of the newly released 'quantum'. This decision was made in order to reduce the setup time for a basic network configuration. Although the next release plans to freeze nova-network development, the team responsible for overseeing OpenStack networking (Thierry, Vish, Dan) have decided that they will "...continue to support nova-network as it currently exists in Folsom".

You can read more about their decision `here <https://lists.launchpad.net/openstack/msg16368.html>`_.

Hardware Requirements
-------------------------

The following are recommended hardware requirements for both the controller and compute nodes.

Controller Node
-------------------

(runs network, volume, API, scheduler and image services)

* **Processor**: 64-bit x86
* **Memory**: 12GB of RAM
* **Disk Space**: 30GB (SATA, SAS, SSD)
* **Volume Storage**: two disks with 2TB (SATA) for volumes attached to the compute nodes
* **Network**: one 1GB NIC

Compute Node(s)
---------------
(runs virtual instances)

* **Processor**: 64-bit x86
* **Memory**: 32GB of RAM
* **Disk Space**: 30GB (SATA, SAS, SSD)
* **Network**: one 1GB NIC

2 Basic Configuration
=====================

.. image:: http://i.imgur.com/r6tE9.png


3 OpenStack Install
===================

3.1 Control Node Install
------------------------

3.1.1 Updating Your System
**************************
* After you complete the Ubuntu 12.10 installation, go into superuser mode and stay there until this tutorial concludes::

   sudo su

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

3.1.2 Install and configure the MySQL & RabbitMQ
-----------------
* Install MySQL::

   apt-get install mysql-server python-mysqldb

* Configure MySQL to accept all incoming requests::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* Install RabbitMQ::

   apt-get install rabbitmq-server

3.1.3 Install and configure the NTP service
-----------------

* Install the NTP service::

   apt-get install ntp
   
* Configure the NTP server to synchronize between your compute node(s) and the controller node::

   sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
   service ntp restart

3.1.4 Install VLAN, Bridge-Utils, and setup IP Forwarding
-----------------

* Install the VLAN and Bridge-Utils services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding by uncommenting net.ipv4.ip_forward=1 in /etc/sysctl.conf::

   vi /etc/sysctl.conf

* Now, run systcl with the updated configuration::

   sysctl -p 

3.1.5 Install and configure Keystone
-----------------

* Install the Keystone identity service::

   apt-get install keystone

* Create a new MySQL database for Keystone::

   mysql -u root -p
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   quit;
 
* Adapt the connection attribute in the /etc/keystone/keystone.conf to our newly created database.

   vi /etc/keystone/keystone.conf
   #and edit the connection field to show
   connection = mysql://keystoneUser:keystonePass@10.32.14.232/keystone
   
* Restart the identity service then synchronize the database.

   service keystone restart
   keystone-manage db_sync

* Use mseknibilel's scripts 

   wget https://raw.github.com/nimbula/OpenStack-Folsom-Install-guide/master/Keystone_Scripts/keystone_basic.sh
   wget https://raw.github.com/nimbula/OpenStack-Folsom-Install-guide/master/Keystone_Scripts/keystone_endpoints_basic.sh
   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

* In the keystone_basic.sh script, change the $HOST_IP variable to your X.X.X.232 address
* In the keystone_endpoints_basic.sh script, change the $HOST_IP, $EXT_HOST_IP, & $MYSQL_HOST variables to your X.X.X.232 address and then execute the scripts.
 
**Note: Double check your work here, screwing up keystone can be a pain to recover from.**
 
   vi keystone_basic.sh
   vi keystone_endpoints_basic.sh
   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* The keystone_basic.sh script has no output, but keystone_endpoints_basic.sh should kick out something similar to this:
 
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |    OpenStack Compute Service     |
   |      id     | 2801693507a44570a7439245b20ea0cd |
   |     name    |               nova               |
   |     type    |             compute              |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |     OpenStack Volume Service     |
   |      id     | b80f524c06464c0c8af80942a1c94f78 |
   |     name    |              cinder              |
   |     type    |              volume              |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |     OpenStack Image Service      |
   |      id     | 9326c1e4d4bc4e748bd8387fa5279bd0 |
   |     name    |              glance              |
   |     type    |              image               |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |        OpenStack Identity        |
   |      id     | 7fd27d54ac7c476cb36ef7d0002b9fda |
   |     name    |             keystone             |
   |     type    |             identity             |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |      OpenStack EC2 service       |
   |      id     | 7ce8ae8b16774c3f82e0eeecea60520a |
   |     name    |               ec2                |
   |     type    |               ec2                |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |   OpenStack Networking service   |
   |      id     | 8777783c2f9f4ae3a3a6a501833ab021 |
   |     name    |             quantum              |
   |     type    |             network              |
   +-------------+----------------------------------+
   +-------------+-------------------------------------------+
   |   Property  |                   Value                   |
   +-------------+-------------------------------------------+
   |   adminurl  | http://10.32.14.232:8774/v2/$(tenant_id)s |
   |      id     |      ecfcff81220c45ce9f13ca000f1c4fa7     |
   | internalurl | http://10.32.14.232:8774/v2/$(tenant_id)s |
   |  publicurl  | http://10.32.14.232:8774/v2/$(tenant_id)s |
   |    region   |                 RegionOne                 |
   |  service_id |      2801693507a44570a7439245b20ea0cd     |
   +-------------+-------------------------------------------+
   +-------------+-------------------------------------------+
   |   Property  |                   Value                   |
   +-------------+-------------------------------------------+
   |   adminurl  | http://10.32.14.232:8776/v1/$(tenant_id)s |
   |      id     |      420959377cde408a865445b0ea743a19     |
   | internalurl | http://10.32.14.232:8776/v1/$(tenant_id)s |
   |  publicurl  | http://10.32.14.232:8776/v1/$(tenant_id)s |
   |    region   |                 RegionOne                 |
   |  service_id |      b80f524c06464c0c8af80942a1c94f78     |
   +-------------+-------------------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |   http://10.32.14.232:9292/v2    |
   |      id     | f2c0b4ea7bed4a8aa2d44b140df73a0d |
   | internalurl |   http://10.32.14.232:9292/v2    |
   |  publicurl  |   http://10.32.14.232:9292/v2    |
   |    region   |            RegionOne             |
   |  service_id | 9326c1e4d4bc4e748bd8387fa5279bd0 |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |  http://10.32.14.232:35357/v2.0  |
   |      id     | ef0fb3dfa5f74f70a2059dd015e7743d |
   | internalurl |  http://10.32.14.232:5000/v2.0   |
   |  publicurl  |  http://10.32.14.232:5000/v2.0   |
   |    region   |            RegionOne             |
   |  service_id | 7fd27d54ac7c476cb36ef7d0002b9fda |
   +-------------+----------------------------------+
   +-------------+-----------------------------------------+
   |   Property  |                  Value                  |
   +-------------+-----------------------------------------+
   |   adminurl  | http://10.32.14.232:8773/services/Admin |
   |      id     |     e5a40371df6e47e79dc78bb61591fc87    |
   | internalurl | http://10.32.14.232:8773/services/Cloud |
   |  publicurl  | http://10.32.14.232:8773/services/Cloud |
   |    region   |                RegionOne                |
   |  service_id |     7ce8ae8b16774c3f82e0eeecea60520a    |
   +-------------+-----------------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |    http://10.32.14.232:9696/     |
   |      id     | 8396e30ecbf14f3d9bd97d489f7407ea |
   | internalurl |    http://10.32.14.232:9696/     |
   |  publicurl  |    http://10.32.14.232:9696/     |
   |    region   |            RegionOne             |
   |  service_id | 8777783c2f9f4ae3a3a6a501833ab021 |
   +-------------+----------------------------------+

* Let's create our OpenStack credential file and load it so we won't be bothered later.

   vi creds
 
* Paste the following text
 
   export OS_NO_CACHE=1
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://10.32.14.232:5000/v2.0/"

* Load the file
 
   source creds
 
 
* Let's just do a quick test to see if Keystone is up.

   apt-get install curl openssl
   curl http://10.32.14.232:35357/v2.0/endpoints -H 'x-auth-token: ADMIN' | python -m json.tool
* It should kick out something like this:

   {
   "endpoints": [
   {
   "adminurl": "http://10.32.14.232:8776/v1/$(tenant_id)s", 
   "id": "0fe6ddf16ce344989adb22a644befa48", 
   "internalurl": "http://10.32.14.232:8776/v1/$(tenant_id)s", 
   "publicurl": "http://10.32.14.232:8776/v1/$(tenant_id)s", 
   "region": "RegionOne", 
   "service_id": "d0a8dbeac60845aaa1fa043c23177d5e"
   }, 
   {
   "adminurl": "http://10.32.14.232:35357/v2.0", 
   "id": "7811cbbf4c3042f1a6b97d19a9ceace5", 
   "internalurl": "http://10.32.14.232:5000/v2.0", 
   "publicurl": "http://10.32.14.232:5000/v2.0", 
   "region": "RegionOne", 
   "service_id": "00685df9e085427a97837892622ca4b2"
   }, 
   {
   "adminurl": "http://10.32.14.232:8774/v2/$(tenant_id)s", 
   "id": "826a7b77f108414ea4be8eb06d3b0c96", 
   "internalurl": "http://10.32.14.232:8774/v2/$(tenant_id)s", 
   "publicurl": "http://10.32.14.232:8774/v2/$(tenant_id)s", 
   "region": "RegionOne", 
   "service_id": "1a7bd347252049d9921703d45c1182dc"
   }, 
   {
   "adminurl": "http://10.32.14.232:9696/", 
   "id": "b0974d6c9bbb4f2cab281f3ff5bcd412", 
   "internalurl": "http://10.32.14.232:9696/", 
   "publicurl": "http://10.32.14.232:9696/", 
   "region": "RegionOne", 
   "service_id": "fc2b6886fd8241448d4f3b0c9a960bf0"
   }, 
   {
   "adminurl": "http://10.32.14.232:9292/v2", 
   "id": "c49d46bc5a62445ea60dc568abc954bb", 
   "internalurl": "http://10.32.14.232:9292/v2", 
   "publicurl": "http://10.32.14.232:9292/v2", 
   "region": "RegionOne", 
   "service_id": "574f359c07fc449ab6b0b4fad42b2df9"
   }, 
   {
   "adminurl": "http://10.32.14.232:8773/services/Admin", 
   "id": "d50733db9848451596c84b782906cba1", 
   "internalurl": "http://10.32.14.232:8773/services/Cloud", 
   "publicurl": "http://10.32.14.232:8773/services/Cloud", 
   "region": "RegionOne", 
   "service_id": "a0fdb3cd3a234cada512ba0a75a6df56"
   }
   ]
   }
* Now, let's continue by installing the image storage service (Glance)

   apt-get install glance

* Let's create a new MySQL database for Glance

   mysql -u root -p
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
   quit;

* Next, replace the existing filter:authtoken section in /etc/glance/glance-api-paste.ini with:

   vi /etc/glance/glance-api-paste.ini

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.32.14.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass
 
* Then, update /etc/glance/glance-registry-paste.ini with:

   vi /etc/glance/glance-registry-paste.ini

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.32.14.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Open /etc/glance/glance-api.conf and update with the following:

   vi /etc/glance/glance-api.conf

   sql_connection = mysql://glanceUser:glancePass@10.32.14.232/glance

   [paste_deploy]
   flavor = keystone

* Update the /etc/glance/glance-registry.ini

   vi /etc/glance/glance-registry.conf

   sql_connection = mysql://glanceUser:glancePass@10.32.14.232/glance

   [paste_deploy]
   flavor = keystone

* Restart the glance-api and glance-registry services

   service glance-api restart; service glance-registry restart

* ...and sync databases

   glance-manage db_sync

**Note:** You'll probably get a warning, reminding you that 'useexisting' is deprecated. That's normal, don't worry about it.

* Restart the services again to take into account the new modifications

   service glance-registry restart; service glance-api restart

* Now, let's test the Glance installation by installing the cirros cloud image from the Launchpad mirror

   mkdir images
   cd images
   wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
   glance image-create --name NimbulaTest --is-public true --container-format bare --disk-format qcow2 < cirros-0.3.0-x86_64-disk.img

* ...that last command should produce output similar to:

   +-----------------+------------------------------------+
   | Property | Value |
   +----------------+------------------------------------+
   | checksum | 50bdc35edb03a38d91b1b071afb20a3c |
   | container_format | bare |
   | created_at | 2012-12-04T21:52:49 |
   | deleted | False |
   | deleted_at | None |
   | disk_format | qcow2 |
   | id | 9f045abf-3aa4-40d9-a9e1-7ab7bfa3e1ef |
   | is_public | True |
   | min_disk | 0 |
   | min_ram | 0 |
   | name | NimbulaTest |
   | owner | b302c28c0f0e4d2f8f4d99553fc3971f |
   | protected | False |
   | size | 9761280 |
   | status | active |
   | updated_at | 2012-12-04T21:52:50 |
   +----------------+------------------------------------+

* Now let's make sure it uploaded, by using glance's image list

   glance image-list

* ...it should return something like this:

   +--------------------------------------+-------------+-------------+------------------+---------+--------+
   | ID                                   | Name        | Disk Format | Container Format | Size    | Status |
   +--------------------------------------+-------------+-------------+------------------+---------+--------+
   | 74cec29b-76a1-4e89-8060-f0e2623ae5bf | NimbulaTest | qcow2       | bare             | 9761280 | active |
   +--------------------------------------+-------------+-------------+------------------+---------+--------+

* Now, time to install bridge-utils.

   apt-get install -y bridge-utils

* Reconfigure /etc/network/interfaces

   vi /etc/network/interfaces

   # This file describes the network interfaces available on your system
   # and how to activate them. For more information, see interfaces(5).
   # The loopback network interface
   auto lo
   iface lo inet loopback
   # The primary network interface
   auto br100
   iface br100 inet static
           address 10.32.14.232
           netmask 255.255.255.0
           network 10.32.14.0
           broadcast 10.32.14.255
           gateway 10.32.14.1
           # dns-* options are implemented by the resolvconf package, if installed
           dns-nameservers 172.16.0.16
           dns-search mtv.nimbula.org
           bridge_ports eth0
           bridge_stp off
           bridge_maxwait 0
           bridge_fd 0

* Ensure that you setup the bridge and then restart networking

   sudo brctl addbr br100; sudo /etc/init.d/networking restart

* Time to install Nova (and some other packages)

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-network

* We are also going to remove the Quantum endpoint and service, the script we ran earlier assumes that we will use Quantum instead of nova-networks, and having both endpoints on the same installation can cause some serious conflicts.

* ...first, get the endpoint ID

   keystone endpoint-list | grep 9696

* ...it will return output similar to this:

   | b0974d6c9bbb4f2cab281f3ff5bcd412 | RegionOne | http://192.168.161.232:9696/ | http://192.168.161.232:9696/ | http://192.168.161.232:9696/ | 
 
* ...grab the ID from the output and then remove that endpoint

   keystone endpoint-delete b0974d6c9bbb4f2cab281f3ff5bcd412

* ...next, find the Quantum service ID

   keystone service-list | grep quantum

* ...it will return output similar to this:

   | 9e3b400f6531414c93262644f20cfda1 | quantum  |   network    | OpenStack Networking Service |

* ...grab the ID from the output and then remove that service

   keystone service-delete 9e3b400f6531414c93262644f20cfda1
* Prepare a MySQL database for Nova

   mysql -u root -p
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
   quit;

* Now, let's modify the authtoken section in the /etc/nova/api-paste.ini file.

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.32.14.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova

* Next up is the /etc/nova/nova.conf file.

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   scheduler_driver=nova.scheduler.simple.SimpleScheduler
   s3_host=10.32.14.232
   ec2_host=10.32.14.232
   ec2_dmz_host=10.32.14.232
   rabbit_host=10.32.14.232
   cc_host=10.32.14.232
   metadata_host=10.32.14.232
   metadata_listen=0.0.0.0
   nova_url=http://10.32.14.232:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.32.14.232/nova
   ec2_url=http://10.32.14.232:8773/services/Cloud
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
    
   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://10.32.14.232:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=10.32.14.232:9292
   image_service=nova.image.glance.GlanceImageService
    
   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://10.32.14.232:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.32.14.232
   vncserver_listen=0.0.0.0
    
   # NETWORK
   network_manager=nova.network.manager.FlatDHCPManager
   force_dhcp_release=True
   dhcpbridge_flagfile=/etc/nova/nova.conf
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   # Change my_ip to match each host
   my_ip=10.32.14.232
   public_interface=br100
   vlan_interface=eth0
   flat_network_bridge=br100
   flat_interface=eth0
   #Note the different pool, this will be used for instance range
   fixed_range=10.33.14.0/24
    
   # Compute #
   compute_driver=libvirt.LibvirtDriver
    
   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* Now, sync your database

   nova-manage db sync

**Note**: You may get some debug output mentioning 'nova.db.sqlalchemy.migration'. That's normal, don't worry about it.
* Restart all of your nova-* services

   cd /etc/init.d/; for i in $(ls nova-*); do sudo service $i restart; done

* Make sure all of your services are up and happy.

   nova-manage service list

* ...you should get something like this:

   Binary           Host                                 Zone             Status     State Updated_At
   nova-cert        folsom-1                             nova             enabled    :-)   2012-11-06 18:30:58
   nova-consoleauth folsom-1                             nova             enabled    :-)   2012-11-06 18:30:57
   nova-scheduler   folsom-1                             nova             enabled    :-)   2012-11-06 18:31:05
   nova-network     folsom-1                             nova             enabled    :-)   2012-11-06 18:31:09

**Note:** You may get some debug output mentioning 'nova.db.sqlalchemy.migration'. That's normal, don't worry about it.
 
 
* Now, it's time to install Cinder, this new OpenStack project aims at managing the volumes for VM's. It replaces nova-volumes.

   apt-get install cinder-api cinder-scheduler cinder-volume iscsitarget iscsitarget-dkms

* Prepare a MySQL database for Cinder.

   mysql -u root -p
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
   quit;

* Configure api-paste.ini by following this template:

   vi /etc/cinder/api-paste.ini

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 10.32.14.232
   service_port = 5000
   auth_host = 10.32.14.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* Open cinder.conf and change it to the following:

   vi /etc/cinder/cinder.conf

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@10.32.14.232/cinder
   api_paste_confg = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

* Time to synchronize the database, yet again.

   cinder-manage db sync

* Now, let's create a volume group, name it cinder-volumes, and make sure that it persists after a reboot.

   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2

* ...and at the fdisk prompt, enter the following commands.

   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

* Proceed to create the physical volume and then the volume group itself.

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

* Now, let's add this to the rc.local to make sure that we don't lose this volume on a server reboot
* ...add the following to the file before the exit 0 line:

   #the path is wherever you ran the previous two steps + "cinder-volumes"
   losetup /dev/loop2 <PATH_TO_VG>

* Finally, we are almost done. On to the Horizon interface....

   apt-get install openstack-dashboard memcached

* Now, for some reason, they felt that the Ubuntu theme was stable enough to be the default option - it's not.  Disable it by doing the following:

   vi /etc/openstack-dashboard/local_settings.py

   #Comment these lines
   #Enable the Ubuntu theme if it is present.
   #try:
   #   from ubuntu_theme import *
   #except ImportError:
   #   pass

* Horizon now requires a reboot to authenticate properly. Reboot and when the machine is ready, start all of your nova services.

   reboot
   cd /etc/init.d/; for i in $(ls nova-*); do sudo service $i start; done


*** Compute Node ***
Now, all we have to do is add a compute node. Log onto the next available node on your cluster. (repeat for as many nodes as you'd like)
.....start by updating your system as root

Terminal
1
2
3
4
sudo su
apt-get update
apt-get upgrade
apt-get dist-upgrade
Install the NTP service

Terminal
1
apt-get install ntp
Configure the NTP server to follow the controller node.

Terminal
1
2
sed -i 's/server ntp.ubuntu.com/server 10.32.14.232/' /etc/ntp.conf
service ntp restart
Install other miscellaneous services

Terminal
1
apt-get install vlan bridge-utils
Enable IP_Forwarding by uncommenting net.ipv4.ip_forward=1
 
Terminal
1
vi /etc/sysctl.conf
Now, run systcl with the updated configuration

Terminal
1
sysctl -p
Next, check if you can install KVM on your machine
 
Terminal
1
apt-get install cpu-checker
....then run
Terminal
1
kvm-ok
...you should get a response similar to:
Terminal
1
KVM acceleration can be used
Now that we are all clear, let's install kvm and configure it.

Terminal
1
apt-get install -y kvm libvirt-bin pm-utils
Edit the cgroup_device_acl array in the qemu.conf file to:
 
Terminal
1
vi /etc/libvirt/qemu.conf
Terminal
1
2
3
4
5
6
cgroup_device_acl = [
"/dev/null", "/dev/full", "/dev/zero",
"/dev/random", "/dev/urandom",
"/dev/ptmx", "/dev/kvm", "/dev/kqemu",
"/dev/rtc", "/dev/hpet", "/dev/net/tun"
]
Delete the default virtual bridge.

Terminal
1
2
virsh net-destroy default
virsh net-undefine default
Enable live migration by uncommenting the listen_tls = 0, listen_tcp = 1, and auth_tcp = "none" fields in the libvirtd.conf file. Don't touch any of the other existing settings.

Terminal
1
vi /etc/libvirt/libvirtd.conf
Terminal
1
2
3
listen_tls = 0
listen_tcp = 1
auth_tcp = "none"
Edit libvirtd_opts variable in the libvirt-bin.conf file.

Terminal
1
vi /etc/default/libvirt-bin.conf
...find env libvirtd_opts and set it to:

Terminal
1
env libvirtd_opts="-d -l"
Edit the same field in /etc/default/libvirt-bin and again, set it to:
 
Terminal
1
libvirtd_opts="-d -l"
Restart the libvirt service to apply the changes.

Terminal
1
service libvirt-bin restart
Now, time to install nova-network and bridge-utils

Terminal
1
apt-get install nova-network bridge-utils
Now, let's configure our interfaces file similar to our first node. (this time we will just use a different IP)

Terminal
1
vi /etc/network/interfaces
Terminal
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).
# The loopback network interface
auto lo
iface lo inet loopback
# The primary network interface
auto br100
iface br100 inet static
        address 10.32.14.234
        netmask 255.255.255.0
        network 10.32.14.0
        broadcast 10.32.14.255
        gateway 10.32.14.1
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 172.16.0.16
        dns-search mtv.nimbula.org
        bridge_ports eth0
        bridge_stp off
        bridge_maxwait 0
        bridge_fd 0


Now, make sure the br100 is added and restart the networking services.

Terminal
1
brctl add br100; /etc/init.d/networking restart
Now, let's install the compute packages.

Terminal
1
apt-get install nova-api-metadata nova-compute-kvm
Now, modify the authtoken section in api-paste.ini

Terminal
1
vi /etc/nova/api-paste.ini
Terminal
1
2
3
4
5
6
7
8
9
[filter:authtoken]
paste.filter_factory = keystone.middleware.auth_token:filter_factory
auth_host = 10.32.14.232
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = service_pass
signing_dirname = /tmp/keystone-signing-nova
Next up, edit the nova-compute.conf

Terminal
1
vi /etc/nova/nova-compute.conf
Terminal
1
2
[DEFAULT]
libvirt_type=kvm
Now, time for the good ol' nova.conf again

Terminal
1
vi /etc/nova/nova.conf
Terminal
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
[DEFAULT]
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/run/lock/nova
verbose=True
api_paste_config=/etc/nova/api-paste.ini
scheduler_driver=nova.scheduler.simple.SimpleScheduler
s3_host=10.32.14.232
ec2_host=10.32.14.232
ec2_dmz_host=10.32.14.232
rabbit_host=10.32.14.232
cc_host=10.32.14.232
metadata_host=10.32.14.234
metadata_listen=0.0.0.0
nova_url=http://10.32.14.232:8774/v1.1/
sql_connection=mysql://novaUser:novaPass@10.32.14.232/nova
ec2_url=http://10.32.14.232:8773/services/Cloud
root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
 
# Auth
use_deprecated_auth=false
auth_strategy=keystone
keystone_ec2_url=http://10.32.14.232:5000/v2.0/ec2tokens
# Imaging service
glance_api_servers=10.32.14.232:9292
image_service=nova.image.glance.GlanceImageService
 
# Vnc configuration
novnc_enabled=true
novncproxy_base_url=http://10.32.14.232:6080/vnc_auto.html
novncproxy_port=6080
vncserver_proxyclient_address=10.32.14.234
vncserver_listen=0.0.0.0
 
# NETWORK
network_manager=nova.network.manager.FlatDHCPManager
force_dhcp_release=True
dhcpbridge=/usr/bin/nova-dhcpbridge
dhcpbridge_flagfile=/etc/nova/nova.conf
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
# Change my_ip to match each host
my_ip=10.32.14.234
public_interface=br100
vlan_interface=eth0
flat_network_bridge=br100
flat_interface=eth0
#Note the different pool, this will be used for instance range
fixed_range=10.33.14.0/24
 
# Compute #
compute_driver=libvirt.LibvirtDriver
 
# Cinder #
volume_api_class=nova.volume.cinder.API
osapi_volume_listen_port=5900
Resync your databases

Terminal
1
nova-manage db sync
Restart services to update changes

Terminal
1
cd /etc/init.d/; for i in $(ls nova-*); do sudo service $i restart; done
Check to make sure your services are in a good mood.

Terminal
1
nova-manage service list
...you should now see a mix of services running on multiple nodes.
Terminal
1
2
3
4
5
6
7
Binary           Host                                 Zone             Status     State Updated_At
nova-cert        folsom-1                             nova             enabled    :-)   2012-11-08 00:26:03
nova-consoleauth folsom-1                             nova             enabled    :-)   2012-11-08 00:26:03
nova-scheduler   folsom-1                             nova             enabled    :-)   2012-11-08 00:26:04
nova-network     folsom-1                             nova             enabled    :-)   2012-11-08 00:26:04
nova-compute     folsom-2                             nova             enabled    :-)   2012-11-08 00:26:00
nova-network     folsom-2                             nova             enabled    :-)   2012-11-08 00:26:00
Note: You may get some debug output mentioning 'nova.db.sqlalchemy.api'. That's normal, don't worry about it.
 
 
Now, log onto OpenStack horizon by visiting the URL: http://10.32.14.232/horizon and logging in with the credentials:

Terminal
1
2
username: admin
password: admin_pass
Next, navigate to the "Projects" tab on the bottom left of the landing screen.


Now, in the new pane go ahead and click the "Create Project" button in the top right. You will be greeted with a modal dialog like so:



Fill in the fields presented, and don't forget the tabs on top. Make sure you add yourself as a project member.


Now, your new project should be behind the modal grid. Find your new project and the corresponding row. Copy the "Project ID" to your clipboard, we'll use it in the next step.

We are almost finished. Now it's time to create a network and bind it to that project.

Terminal
1
nova-manage network create --label=NimbulaNetwork --fixed_range_v4=10.33.14.0/24 --bridge=br100 --project_id=<InsertProjectIDHere> --num_networks=1 --multi_host=T
Now, you are done. No seriously. Go to http://10.32.14.232/horizon (or whatever your IP is) and then select your project, find an image, and launch and instance.


Tips & Tricks
 

So, it's entirely possible that you screw up your network the first time, maybe you give it the wrong IP Pool, or maybe you assign it to the wrong project. Now, all of your instances are in the error state and you can't delete them. Luckily, the intern already found two very simple and undocumented processes of removing them.

Jump on your first node, open up the terminal as root, and plugin the following commands:

Terminal
1
2
3
service nova-network stop
nova-manage project scrub <ProjectName>
nova-manage network list
 
...it should return something like this
Terminal
1
2
id      IPv4                IPv6            start address   DNS1            DNS2            VlanID          project         uuid           
3       10.33.14.0/24       None            10.33.14.2      8.8.4.4         None            None            Nimbula         8ccdef11-7070-4852-a212-31c3ddedccd3
 
...find the network you want to remove and copy the IPv4 section.
Terminal
1
nova-manage network delete 10.33.14.0/24
Now, we've got to get rid of those error state instances

Terminal
1
nova list | grep ERROR
 
...should return....
Terminal
1
2
3
4
5
+------+------------+--------+--------------------------------+
|  ID  |    Name    | Status |            Networks            |
+------+------------+--------+--------------------------------+
| 1805 | testserver | ERROR  | private=10.4.96.81             |
+------+------------+--------+--------------------------------+

...this returns all of the instances stuck in the error state - plug in their names to the following command:
Terminal
1
2
nova reset-state --active <name>
nova delete <name>
If those two commands don't work, we can take more drastic measures.

Terminal
1
vi DeleteInstances.sh
 
....paste the following in:
Terminal
1
2
3
4
5
6
7
#!/bin/bash
mysql -uroot -ppassword << EOF
use nova;
DELETE a FROM nova.security_group_instance_association AS a INNER JOIN nova.instances AS b ON a.instance_id=b.id where b.uuid='$1';
DELETE FROM nova.instance_info_caches WHERE instance_id='$1';
DELETE FROM nova.instances WHERE uuid='$1';
EOF
 
...save it by hitting ESC, then ":", then x, and hitting Enter - Then make sure it's executable by using the following:
Terminal
1
chmod +x DeleteInstances.sh
 
...and run it
Terminal
1
./DeleteInstances.sh
 
 
Floating IP setup

...first create a dedicated pool
Terminal
1
sudo nova-mange floating create --pool pool_auto_assign --ip_range X.X.X.X/X

...then modify the nova.conf with these flags:
Terminal
1
vi /etc/nova/nova.conf
Terminal
1
2
3
default_floating_pool = pool_auto_assign
floating_range = X.X.X.X/X
auto_assign_floating_ip = True
You may also want to increase the floating IP's quota:

...this is also in the /etc/nova/nova.conf
Terminal
1
quota_floating_ips = 50
 

Then, we need to restart nova-network

Terminal
1
sudo service nova-network restart
You are done! Celebrate!












####### This is the one I'm stealing formatting from
0. What is it?
==============

OpenStack Folsom Install Guide is an easy and tested way to create your own OpenStack plateform. 

Version 3.0

Status: stable 


1. Requirements
====================

:Node Role: NICs
:Control Node: eth0 (100.10.10.51), eth1 (192.168.100.51)
:Network Node: eth0 (100.10.10.52), eth1 (100.20.20.52), eth2 (192.168.100.52)
:Compute Node: eth0 (100.10.10.53), eth1 (100.20.20.53)

**Note 1:** If you don't have 2 NICs on controller node, you can check other branches for 2 NIC installation.

**Note 2:** Compute and Controller nodes can be merged into one node.

**Note 3:** If you are not interrested in Quantum, you can also use this guide but you must follow the nova section found `here <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/Tricks%26Ideas/install_nova-network.rst>`_ instead of the one written in this guide.

**Note 4:** This is my current network architecture, you can add as many compute node as you wish.

.. image:: http://i.imgur.com/aJvZ7.jpg

2. Controller node
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

* Only one NIC on the controller node need internet access::

   #For Exposing OpenStack API over the internet
   auto eth1
   iface eth1 inet static
   address 192.168.100.51
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

   #Not internet connected(used for OpenStack management)
   auto eth0
   iface eth0 inet static
   address 100.10.10.51
   netmask 255.255.255.0

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

2.6. Keystone
-------------------

* Start by the keystone packages::

   apt-get install keystone

* Create a new MySQL database for keystone::

   mysql -u root -p
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   quit;

* Adapt the connection attribute in the /etc/keystone/keystone.conf to the new database::

   connection = mysql://keystoneUser:keystonePass@100.10.10.51/keystone

* Restart the identity service then synchronize the database::

   service keystone restart
   keystone-manage db_sync

* Fill up the keystone database using the two scripts available in the `Scripts folder <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/tree/master/Keystone_Scripts>`_ of this git repository. Beware that you MUST comment every part related to Quantum if you don't intend to install it otherwise you will have trouble with your dashboard later::

   #Modify the HOST_IP and HOST_IP_EXT variables before executing the scripts

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
   export OS_AUTH_URL="http://192.168.100.51:5000/v2.0/"
   # Load it:
   source creds

* To test Keystone, we use a simple curl request::

   apt-get install curl openssl
   curl http://192.168.100.51:35357/v2.0/endpoints -H 'x-auth-token: ADMIN'

2.7. Glance
-------------------

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
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update the /etc/glance/glance-registry-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update /etc/glance/glance-api.conf with::

   sql_connection = mysql://glanceUser:glancePass@100.10.10.51/glance

* And::

   [paste_deploy]
   flavor = keystone

* Update the /etc/glance/glance-registry.conf with::

   sql_connection = mysql://glanceUser:glancePass@100.10.10.51/glance

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

2.8. Quantum
-------------------

* Install the Quantum server::

   apt-get install quantum-server quantum-plugin-openvswitch

* Create a database::

   mysql -u root -p
   CREATE DATABASE quantum;
   GRANT ALL ON quantum.* TO 'quantumUser'@'%' IDENTIFIED BY 'quantumPass';
   quit; 

* Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@100.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type=vlan
   network_vlan_ranges = physnet1:1:4094

* Edit /etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* Restart the quantum server::

   service quantum-server restart

2.9. Nova
-------------------

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
   auth_host = 100.10.10.51
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
   s3_host=100.10.10.51
   ec2_host=100.10.10.51
   ec2_dmz_host=100.10.10.51
   rabbit_host=100.10.10.51
   cc_host=100.10.10.51
   dmz_cidr=169.254.169.254/32
   metadata_host=100.10.10.51
   metadata_listen=0.0.0.0
   nova_url=http://100.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@100.10.10.51/nova
   ec2_url=http://100.10.10.51:8773/services/Cloud 
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://100.10.10.51:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=100.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=192.168.100.51
   vncserver_listen=0.0.0.0 

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://100.10.10.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://100.10.10.51:35357/v2.0
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

2.10. Cinder
-------------------

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
   service_host = 192.168.100.51
   service_port = 5000
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* Edit the /etc/cinder/cinder.conf to::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@100.10.10.51/cinder
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

2.11. Horizon
-------------------

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

You can now access your OpenStack **192.168.100.51/horizon** with credentials **admin:admin_pass**.

**Note:** A reboot might be needed for a successful login

3. Network Node
=========================

3.1. Preparing the Node
------------------

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

* Install ntp service::

   apt-get install ntp

* Configure the NTP server to follow the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server 100.10.10.51/g' /etc/ntp.conf
   service ntp restart  

* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   nano /etc/sysctl.conf
   # Uncomment net.ipv4.ip_forward=1, to save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

3.2.Networking
------------

* 3 NICs must be present::
   

   # VM internet Access
   auto eth2
   iface eth2 inet static
   address 192.168.100.52
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8
   
   # OpenStack management
   auto eth0
   iface eth0 inet static
   address 100.10.10.52
   netmask 255.255.255.0

   # VM Configuration
   auto eth1
   iface eth1 inet static
   address 100.20.20.52
   netmask 255.255.255.0


3.4. OpenVSwitch
------------------

* Install the openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* Create the bridges::

   #br-int will be used for VM integration	
   ovs-vsctl add-br br-int

   #br-eth1 will be used for VM configuration 
   ovs-vsctl add-br br-eth1
   ovs-vsctl add-port br-eth1 eth1

   #br-ex is used to make to VM accessible from the internet
   ovs-vsctl add-br br-ex
   ovs-vsctl add-port br-ex eth2

3.5. Quantum
------------------

* Install the Quantum openvswitch agent, l3 agent and dhcp agent::

   apt-get -y install quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent

* Edit /etc/quantum/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@100.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type=vlan
   network_vlan_ranges = physnet1:1:4094
   bridge_mappings = physnet1:br-eth1

* In addition, update the /etc/quantum/l3_agent.ini::

   auth_url = http://100.10.10.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   metadata_ip = 192.168.100.51
   metadata_port = 8775

* Make sure that your rabbitMQ IP in /etc/quantum/quantum.conf is set to the controller node::
   
   rabbit_host = 100.10.10.51

* Restart all the services::

   service quantum-plugin-openvswitch-agent restart
   service quantum-dhcp-agent restart
   service quantum-l3-agent restart

4. Compute Node
=========================

4.1. Preparing the Node
------------------

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

* Install ntp service::

   apt-get install ntp

* Configure the NTP server to follow the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server 100.10.10.51/g' /etc/ntp.conf
   service ntp restart  

* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   nano /etc/sysctl.conf
   # Uncomment net.ipv4.ip_forward=1, to save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

4.2.Networking
------------

* Perform the following::
   
   # OpenStack management
   auto eth0
   iface eth0 inet static
   address 100.10.10.53
   netmask 255.255.255.0

   # VM Configuration
   auto eth1
   iface eth1 inet static
   address 100.20.20.53
   netmask 255.255.255.0

4.3 KVM
------------------

* make sure that your hardware enables virtualization::

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

4.4. OpenVSwitch
------------------

* Install the openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* Create the bridges::

   #br-int will be used for VM integration	
   ovs-vsctl add-br br-int

   #br-eth1 will be used for VM configuration 
   ovs-vsctl add-br br-eth1
   ovs-vsctl add-port br-eth1 eth1

4.5. Quantum
------------------

* Install the Quantum openvswitch agent::

   apt-get -y install quantum-plugin-openvswitch-agent

* Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@100.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type=vlan
   network_vlan_ranges = physnet1:1:4094
   bridge_mappings = physnet1:br-eth1

* Make sure that your rabbitMQ IP in /etc/quantum/quantum.conf is set to the controller node::
   
   rabbit_host = 100.10.10.51

* Restart all the services::

   service quantum-plugin-openvswitch-agent restart

4.6. Nova
------------------

* Install nova's required components for the compute node::

   apt-get install nova-compute-kvm

* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
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
   s3_host=100.10.10.51
   ec2_host=100.10.10.51
   ec2_dmz_host=100.10.10.51
   rabbit_host=100.10.10.51
   cc_host=100.10.10.51
   dmz_cidr=169.254.169.254/32
   metadata_host=100.10.10.51
   metadata_listen=0.0.0.0
   nova_url=http://100.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@100.10.10.51/nova
   ec2_url=http://100.10.10.51:8773/services/Cloud 
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://100.10.10.51:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=100.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=100.10.10.53
   vncserver_listen=0.0.0.0 

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://100.10.10.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://100.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

5. Your First VM
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

   quantum subnet-create --tenant-id $put_id_of_project_one net_proj_one 50.50.1.0/24

* Create a router for the new tenant::

   quantum router-create --tenant-id $put_id_of_project_one router_proj_one

* Add the router to the subnet::

   quantum router-interface-add $put_router_proj_one_id_here $put_subnet_id_here

* Create your external network with the tenant id belonging to the service tenant (keystone tenant-list to get the appropriate id) ::

   quantum net-create --tenant-id $put_id_of_service_tenant ext_net --router:external=True

* Create a subnet containing your floating IPs::

   quantum subnet-create --tenant-id $put_id_of_service_tenant --allocation-pool start=192.168.100.102,end=192.168.100.126 --gateway 192.168.100.1 ext_net 192.168.100.100/24 --enable_dhcp=False

* Set the router for the external network::

   quantum router-gateway-set $put_router_proj_one_id_here $put_id_of_ext_net_here

VMs gain access to the metadata server locally present in the controller node via the external network. To create that necessary connection perform the following:

* Get the IP address of router proj one::

   quantum port-list -- --device_id <router_proj_one_id> --device_owner network:router_gateway

* Add the following route on controller node only::

   route add -net 50.50.1.0/24 gw $router_proj_one_IP

Unfortunatly, you can't use the dashboard to assign floating IPs to VMs so you need to get your hands a bit dirty to give your VM a public IP.

* Start by allocating a floating ip to the project one tenant::

   quantum floatingip-create --tenant-id $put_id_of_project_one ext_net

* pick the id of the port corresponding to your VM::

   quantum port-list

* Associate the floating IP to your VM::

   quantum floatingip-associate $put_id_floating_ip $put_id_vm_port

**This is it !**, You can now ping you VM and start administrating you OpenStack !

I Hope you enjoyed this guide, please if you have any feedbacks, don't hesitate.

6. Licensing
============

OpenStack Folsom Install Guide by Bilel Msekni is licensed under a Creative Commons Attribution 3.0 Unported License.

.. image:: http://i.imgur.com/4XWrp.png
To view a copy of this license, visit [ http://creativecommons.org/licenses/by/3.0/deed.en_US ].

7. Contacts
===========

Bilel Msekni: bilel.msekni@telecom-sudparis.eu

8. Acknowledgment
=================

This work has been supported by:

* CompatibleOne Project (French FUI project) [http://compatibleone.org/]
* Easi-Clouds (ITEA2 project) [http://easi-clouds.eu/]

9. Credits
=================

This work has been based on:

* Emilien Macchi's Folsom guide [https://github.com/EmilienM/openstack-folsom-guide]
* OpenStack Documentation [http://docs.openstack.org/trunk/openstack-compute/install/apt/content/]
* OpenStack Quantum Install [http://docs.openstack.org/trunk/openstack-network/admin/content/ch_install.html]

10. To do
=======

This guide is just a startup. Your suggestions are always welcomed.

Some of this guide's needs might be:

* Define more Quantum configurations to cover all usecases possible see `here <http://docs.openstack.org/trunk/openstack-network/admin/content/use_cases.html>`_. 




