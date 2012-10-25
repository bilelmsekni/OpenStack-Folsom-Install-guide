==========================================================
  Install nova-network for OpenStack Folsom
==========================================================

:Version: 1.0
:Keywords: OpenStack Folsom, Nova-network, 2 NICs, Ubuntu Server 12.04 LTE (64 bits).
:Status: Stable

Authors
==========

Copyright (C) Jerry Luk <jerryluk@gmail.com>

This work is licensed under the Creative Commons Attribution-ShareAlike 3.0 Unported License.
 
To view a copy of this license, visit [ http://creativecommons.org/licenses/by-sa/3.0/ ].

Contributors
==========

**Wana contribute ? Read the idea, send your contribution and get your name listed ;)**

0. What is this about?
==============

Many people have been complaining on how they are not being able to deploy OpenStack Folsom without having 3 NICs on the controller node. Fortunatly, there is a smart solution to install OpenStack Folsom with just two NICs on the controller node: We can use nova-network !!

You can use the original gudie to deploy OpenStack Folsom found `here <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/OpenStack_Folsom_Install_Guide_WebVersion.rst>`_ but you **MUST** comment out everything that is quantum related (which is important because once the end-point is created, deleting it will still result in some error behavior with Horizon) in the keystone scripts.

1. How to do it?
====================

First make sure that your hardware meets these requirements:

:Node Role: NICs
:Control Node: eth0 (10.0.1.4), eth1 (192.168.1.101)
:Compute Node: eth0 (10.0.1.2), eth1 (192.168.1.102)

**Note:** eth1 is connected to the external network while eth0 is connected to the internal network.

1.1. The controller node:
-----------------

* Add this script to /etc/network/if-pre-up.d/iptablesload to forward traffic to eth1::

   #!/bin/sh
   iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
   exit 0

* Install these packages::

   nova-api nova-cert nova-compute nova-compute-kvm nova-doc nova-network nova-scheduler rabbitmq-server novnc nova-consoleauth nova-ajax-console-proxy nova-novncproxy

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


* Change your /etc/nova/nova.conf to look like this::
   
   [DEFAULT]

   # LOGS/STATE
   verbose=True
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova

   # AUTHENTICATION
   auth_strategy=keystone

   # SCHEDULER
   scheduler_driver=nova.scheduler.multi.MultiScheduler
   compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler

   # CINDER
   volume_api_class=nova.volume.cinder.API

   # DATABASE
   sql_connection=mysql://novaUser:novaPass@10.0.1.4/nova

   # COMPUTE
   libvirt_type=kvm
   libvirt_use_virtio_for_bridges=True
   start_guests_on_host_boot=True
   resume_guests_state_on_host_boot=True
   api_paste_config=/etc/nova/api-paste.ini
   allow_admin_api=True
   use_deprecated_auth=False
   nova_url=http://10.0.1.4:8774/v1.1/
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # APIS
   ec2_host=10.0.1.4
   ec2_url=http://10.0.1.4:8773/services/Cloud
   keystone_ec2_url=http://10.0.1.4:5000/v2.0/ec2tokens
   s3_host=10.0.1.4
   cc_host=10.0.1.4
   metadata_host=10.0.1.4
   #metadata_listen=0.0.0.0
   enabled_apis=ec2,osapi_compute,metadata

   # RABBITMQ
   rabbit_host=10.0.1.4

   # GLANCE
   image_service=nova.image.glance.GlanceImageService
   glance_api_servers=10.0.1.4:9292

   # NETWORK
   network_manager=nova.network.manager.FlatDHCPManager
   force_dhcp_release=True
   dhcpbridge_flagfile=/etc/nova/nova.conf
   dhcpbridge=/usr/bin/nova-dhcpbridge
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   public_interface=eth1
   flat_interface=eth0
   flat_network_bridge=br100
   fixed_range=10.0.1.129/25
   network_size=128
   flat_network_dhcp_start=10.0.1.129
   flat_injected=False
   connection_type=libvirt
   multi_host=True

   # NOVNC CONSOLE
   novnc_enabled=True
   novncproxy_base_url=http://192.168.1.101:6080/vnc_auto.html
   vncserver_proxyclient_address=10.0.1.4
   vncserver_listen=10.0.1.4

* Don't forget to update the ownership rights of the nova directory::

   chown -R nova. /etc/nova
   chmod 644 /etc/nova/nova.conf

* Add this line to the sudoers file::

   sudo visudo
   #Paste this line anywhere you like:
   nova ALL=(ALL) NOPASSWD:ALL

* Synchronize your database::

   nova-manage db sync

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

* Use the following command to create fixed network::
   
   nova-manage network create private --fixed_range_v4=10.0.1.129/25 --num_networks=1 --bridge=br100 --bridge_interface=eth0 --network_size=128 --multi_host=T

* Create the floating IPs::

   nova-manage floating create --ip_range=192.168.1.201

1.1. The compute node:
-----------------

* Install this packages::

   nova-compute nova-network nova-api-metadata

* Edit your /etc/nova/nova.conf, Don't forget to change vncserver_proxyclient_address and vncserver_listen to match each compute host::

   [DEFAULT]

   # LOGS/STATE
   verbose=True
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova

   # AUTHENTICATION
   auth_strategy=keystone

   # SCHEDULER
   scheduler_driver=nova.scheduler.multi.MultiScheduler
   compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler

   # CINDER
   volume_api_class=nova.volume.cinder.API

   # DATABASE
   sql_connection=mysql://novaUser:novaPass@10.0.1.4/nova

   # COMPUTE
   libvirt_type=kvm
   libvirt_use_virtio_for_bridges=True
   start_guests_on_host_boot=True
   resume_guests_state_on_host_boot=True
   api_paste_config=/etc/nova/api-paste.ini
   allow_admin_api=True
   use_deprecated_auth=False
   nova_url=http://10.0.1.4:8774/v1.1/
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # APIS
   ec2_host=10.0.1.4
   ec2_url=http://10.0.1.4:8773/services/Cloud
   keystone_ec2_url=http://10.0.1.4:5000/v2.0/ec2tokens
   s3_host=10.0.1.4
   cc_host=10.0.1.4
   metadata_host=10.0.1.4

   # RABBITMQ
   rabbit_host=10.0.1.4

   # GLANCE
   image_service=nova.image.glance.GlanceImageService
   glance_api_servers=10.0.1.4:9292

   # NETWORK
   network_manager=nova.network.manager.FlatDHCPManager
   force_dhcp_release=True
   dhcpbridge_flagfile=/etc/nova/nova.conf
   dhcpbridge=/usr/bin/nova-dhcpbridge
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   public_interface=eth1
   flat_interface=eth0
   flat_network_bridge=br100
   fixed_range=10.0.1.129/25
   network_size=128
   flat_network_dhcp_start=10.0.1.129
   flat_injected=False
   connection_type=libvirt
   multi_host=True

   # NOVNC CONSOLE
   novnc_enabled=True
   novncproxy_base_url=http://192.168.1.101:6080/vnc_auto.html
   
   # Change vncserver_proxyclient_address and vncserver_listen to match each compute host
   vncserver_proxyclient_address=10.0.1.2
   vncserver_listen=10.0.1.2

**That's it**, you can now move in to the Cinder install section in the original guide.

2. I have a better idea!
====================

You have a better idea ? Share it with us and get it named after you :)  

