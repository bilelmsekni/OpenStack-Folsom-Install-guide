==========================================================
  OpenStack Folsom Install Guide
==========================================================

:Version: 0.2
:Source: https://github.com/mseknibilel/OpenStack-Folsom-Install-guide
:Keywords: Multi node OpenStack, Folsom, Quantum, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM, Ubuntu Server 12.10 (64 bits),Virtual Box, Sand-Boxing, VmWare, Virtual Networks.

Authors
==========

Copyright (C) Pranav Salunke <pps.pranav@gmail.com>


Table of Contents
=================

::

  0. What is it?
  1. Requirements
  2. Setup Your VM Environment
  3. Add Virtual Networks
  4. Install SSH and FTP
  5. Install Your VM's Instances
  6. Its about to get sticky
  7. Controller Node
  8. Network Node
  9. Compute Node
  10. Configure the internal networks
  11. Word Of Advice.

0. What is it?
==============
Well this guide will help you to setup your own OpenStack Cloud on Virtual Machines and Virtual Networks. 
These Virtual Machines and Virtual Networks will be given equal privilege as a physical machine on a physical network.

This Guide is not responsible for teaching OpenStack, Networking , Virtualization and Linux related concepts.

For learning more follow these links:

OpenStack:
  1.I am using **OpenStack Folsom Install Guide** by  **SkiBLE mseknibilel** as it is well written, easy and tested by 
  Open Source geeks, with regular updates.
  You can find OpenStack Folsom Install Guide link `here <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide>`_
  
  2.If you want to blow your brains out then you can refer the OpenStack Official Website which contains all the related 
  Documentation, API's, guides , Wiki's etc. Access the OpenStack Official Website `here <http://www.openstack.org/>`_


Networking:
  1.Basic Networking concepts are necessary but can be ignored, if you want to dig into networking I would 
  suggest `Computer Networks (5th Edition) by Andrew S. Tanenbaum <http://www.amazon.com/Computer-Networks-5th-Andrew-Tanenbaum/dp/0132126958>`_  is an awesome one to learn networking 
  
  2.For learning Virtual Networking or Networking for Virtual Machines the following guide by Virtual Box `here <http://www.virtualbox.org/manual/ch06.html>`_  should suffice.
  **Note :** This is required as further in the guide these Concepts will be handy and you need to know what kind of networks you are setting up as there will be nesting of networks , meaning Virtual Networks inside Virtual Networks.

Virtualization:
  1.Learn, go through the Virtual Box Guide `here <http://www.virtualbox.org/manual/UserManual.html>`_, this should be sufficient to provide a fair idea on how to Virtualization.
  
  2.Virtual Box provides User Interface and API for using it, although API will provide more flexibility, UI will have lesser learning curve, its up to you. I will try to provide both if time permits but I have to remind my-selves that this guide is meant for OpenStack sand-boxing :).
  You can access the API's for advanced networking `here <https://www.virtualbox.org/wiki/Advanced_Networking_Linux>`_.

Linux:
  1.You will need some basic knowledge of Linux otherwise you will go through tremendous torture of blindly following these Guides and if in case come to an error/dead lock, you will get stuck for silly reasons. There are many books, docs available and I don't know which one to recommend so please `Google <https://www.google.com/>`_ it.


Version 0.2

Status: Beta


1. Requirements
====================
Basic Requirements 
:VT Enabled PC:Intel ix or Amd QuadCore
:4GB Ram:DDR2/DDR3 

If you dont know wether your processor is VT enabled, you could check it by installing **cpu checker**
::
        $sudo apt-get install cpu-checker
        $sudo kvm-ok
if your pc does not support VT it will show 
::
        INFO: Your CPU does not support KVM extensions
        KVM acceleration can NOT be used
        
Don't worry you will still be able to use Virtual Box but it will be very slow, so I must consider putting the requirements to be Patience or VT enabled processor ;).

Well there are many ways to configure you OpenStack installation but I am going to follow `OpenStack-Folsom-Install-guide <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/OpenStack_Folsom_Install_Guide_WebVersion.rst>`_


There are two different types of configurations that are possible for setting up of Virtual Networks.

**1. Bridged Connections :** 
------------
Bridged Connection connects your VM as if its a physical machine. This means that your machine will be able to use
internet and can be traced from other machines from internet. So if your network has a physical switch or you can
spare a few IP addresses then I would suggest bridged connection
                            
Advantage of bridged connections is that your networks remain the same and you are free of the hassels of creating
virtual networks.


:Node Role: NICs
:Control Node: eth0 (100.10.10.51), eth1 (192.168.100.51)
:Network Node: eth0 (100.10.10.52), eth1 (100.20.20.52), eth2 (192.168.100.52)
:Compute Node: eth0 (100.10.10.53), eth1 (100.20.20.53)



.. image:: http://i.imgur.com/aJvZ7.jpg

**Note:** If you are using bridged connections you may skip this section as there is no need to set up host-only connections.

**2. Host Only Connections:** 
------------
Host only connections provide an internet network between your host and the Virtual Machine instances
up and running on your host machine. This network is not traceable by other networks.

The following are the host only connections that you will be setting up later on :

  1. vboxnet1 - Openstack Management Network - Host static IP 100.10.10.1 
  2. vboxnet2 - VM Conf. Network - Host Static IP 100.20.20.1
  3. vboxnet3 - VM External Network Access (Host Machine)

    .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/Diagrams/WIth%20Host%20only.png


2. Setup Your VM Environment
==============

Well a few of these sections will be full of screenshots because it is essential for people to understand some of the networking
related configurations so please bear with me since its quite necessary to put it up.

Before you can start configuring your Environment you need to download some of the following stuff:

  1. `Oracle Virtual Box <https://www.virtualbox.org/wiki/Downloads>`_
        Note: You cannot set up a amd64 VM on a x86 machine. 
        
  2. `Ubuntu 12.04 Server or Ubuntu 12.10 Server <http://www.ubuntu.com/download/server>`_
        Note: You need a x86 image for VM's if kvm-ok fails, even though you are on amd64 machine.

  3. My host machine is Ubuntu 12.04 amd64 (Core2duo (VT not supported)) and Ubuntu 12.10 amd64 (Intel i5 2nd gen (VT enabled))
        Please do consider using quad core processors as they are VT enabled. Which is required for virtualization.
        At the worst case go for a dual core processor.

**Note:** Even Though Im using Ubuntu as Host, the same is applicable to Windows or other Linux Hosts. 

If you have i5 or i7 2nd gen processor you can have VT technology inside VM's provided by VmWare. This means that your OpenStack
nodes will give positive result on KVM-OK. (Nesting of type-2 Hypervisors).
Rest of the configurations remain same except for the UI and few other trivial differences.

3. Configure Virtual Networks 
==============

**1. Setting up Virtual Network** :
------------

  **Note:** If you are using Bridged Connections Please Ignore this section.

  Step 1:
    Start **Virtual Box**

  Step 2:
    **File>Preferences** 
    Select **Network** Option.
  Step 3: 
    Click on **Create Host Only Networks** - Create three networks. They will be automatically named as
      vboxnet0, vboxnet1, vboxnet2
        
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/1.%20Virtual%20Network/1-Create%20Host%20only%20Network.png

  Step 4:
    Select vboxnet0 and click on edit, select **Adapter Tab**
      Set the IPv4 address as  **100.10.10.1**
      Leave the other options as it is.
      
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/1.%20Virtual%20Network/2-Give%20Static%20Ip%20to%20Host.png
    
    Select **DHCP Server** tab
      Unselect the **Enable Server** option
      
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/1.%20Virtual%20Network/3-%20Configure%20DHCP.png

**2. Set up Network Interface Cards(NIC) on Virtual Machines** :
------------      
  Step 1:
    Control Node
      Create a new Virtual Machine ... select the appropriate options
      
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/2.%20Setup%20VM/Control%20Node/1-%20Basic%20Info.png
    
    Ram Required for this node is 512 MB, if you have more ram feel free to allocate itbut remember that your Compute Node needs
    the highest amount of RAM and Processor so I usually save up for the compute node...reduce the processor allocation pool
      
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/2.%20Setup%20VM/Control%20Node/2-%20Resource%20Allocation.png
    
    For **Bridged Connections** set up two NIC cards as bridged connections and the settings as shown by the diagram...
      eth0 - 100.10.10.51 (IP addresses are not allocated now)
      eth1 - 192.168.100.51 (IP addresses are not allocated now)
      
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/2.%20Setup%20VM/Control%20Node/7-%20Bridge%20Connection.png
      
      Note: Internet is available to bridged connected VM's directly so no need to setup a seperate NIC for internet.
    For **Host Only Connections** set up three NIC cards as per the given diagram.
      eth0 - OpenStack Management Network - 100.10.10.51 (IP addresses are not allocated now)
      
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/2.%20Setup%20VM/Control%20Node/3-%20control-nw1.png
      
      eth1 - Expose OpenStack API - 192.168.100.51 (IP addresses are not allocated now)
      
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/2.%20Setup%20VM/Control%20Node/4%20-%20control-nw2.png
      
      eth2 - Virtual Box NAT (Network Address Translation) - for internet Connection. (IP addresses are not allocated now)
      
      .. image:: https://raw.github.com/cloud-rack/cloud-rack-docs/master/ScreenShots/2.%20Setup%20VM/Control%20Node/5%20-control-nw3.png

  Step 2:
    Network Node
      Create a new Virtual Machine ... configure it similar to the Control Node except for the networking part.
      
        **For bridged connections** Create three NIC's connect them to bridge network as done above.

        **For Host-Only Connections** Create four NIC's 
          1. eth0 - OpenStack Management Network - 100.10.10.52 (IP addresses are not allocated now)
          2. eth1 - OpenStack VM Conf. Network - 100.20.20.52 (IP addresses are not allocated now)
          3. eth2 - Expose OpenStack to external networks - 192.168.100.52 (IP addresses are not allocated now)
          4. eth3 - NAT - for internet connection.
  Step 3:
    Compute Node:
      Create a new Virtual Machine ... configure it as follows:
        If possible give it about **1gb - 4 gb of ram** depending how much extra RAM you have
        Give as many Processor Cores you can spare with **100% processor Execution Capacity**

        **For bridged connections** Create two NIC's connect them to bridge network as done above.

        **For Host-Only Connections** Create four NIC's 
          1. eth0 - OpenStack Management Network - 100.10.10.53 (IP addresses are not allocated now)
          2. eth1 - OpenStack VM Conf. Network - 100.20.20.53 (IP addresses are not allocated now)
          3. eth2 - NAT - for internet connection.


**Note:** For Host Only Connections - Please do remember to select the NIC card which has the internet access NAT - which is
::
  During Installation of Ubuntu Server on the Virtual Machine Nodes you will be asked for the Network Interface to be 
  Selected for Internet. Make sure you select the proper one.
  1. Control Node :
      Select eth2
  2. Network Node :
      Select eth3
  3. Compute Node :
      Select eth2

**Note:** You can select the network interface orders as per your choice but to make life simpler I have followed `OpenStack-Folsom-Install-Guide by  SkiBLE mseknibilel <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide>`_ 

**Warning:**  You have to select the MAC addresses of the NIC cards before you start the installation of Ubuntu server. And make sure
              that the MAC address are not changed once you start the installation. This leads to Network Interface variable name registory error
              inside the kernel network configurations and you will have to manually edit it , let alone the hell of SSH Key conflicts due
              to change in MAC address after installation of the OS's and OpenStack packages on your VM's.
            



4. Install SSH and FTP
==============
I feel that there is a need to install SSH and FTP so that you could use your remote shell to login into the machine and use
your terminal which is more convenient that using the Virtual Machines tty through the Virtual Box's  UI. You get a few added
comforts like copy - paste stuff into the remote terminal which is not possible directly on VM.

FTP is for transferring files to and fro ... you can also use SFTP or install FTPD on both HOST and VM's.

Installation of SSH and FTP with its configuration is out of scope of this GUIDE and I may put it up but it depends upon my free time.
If someone wants to contribute to this - please welcome. 

**Note:** Please set up the Networks from inside the VM before trying to SSH and FTP into the machines. I would suggest setting
it up at once just after the installation of the Server on VM's is over.


5. Install Your VM's Instances
==============

1. Control Node: Install **SSH server** when asked for **Custom Software to Install**. Rest of the packages are not required and may
   come in the way of OpenStack packages - like DNS servers etc. (not necessary). Unless you know what you are doing.

2. Quantum/Network Node: Install **SSH server** when asked for **Custom Software to Install**. Rest of the packages are not required and may
   come in the way of OpenStack packages - like DNS servers etc. (not necessary). Unless you know what you are doing.

3. Control Node: Install **SSH server** and **Virtual Machines Host** when asked for **Custom Software to Install**. Rest of the packages are not required and may
   come in the way of OpenStack packages - like DNS servers etc. (not necessary). Unless you know what you are doing.

6. Its about to get sticky
==============

Well there are a few warnings that I must give you out of experience due to stupid habits that normal Users like me have -
1. Never Shutdown your Virtual Machine - just save its state Virtual Box and VmWare both provide it.
      In past this has broken NOVA packages , NOVA database, other errors have risen. I had to go restart each and every NOVA service on Control and Compute node. Believe me sometimes they can be pain in ass as they refuse to start up on reboot.
      Once you configure up the messy part of Quantum Floating Ip's etc., honestly you dont want to re do it cause the settings get lost on reboot/shutdown.
      Linux Servers are meant to be running 24x7 ... so no need for restarts until required. 
2. If you are using bridged connection over a different physical router and have a seperate Internet connection/network ... then you can put up additional network interface NAT connections on your VM's for giving them Internet Access.
3. VmWare NAT connection has minimal functionality issues. Virtual Box NAT connection is a bad boy - will disconnect or not work properly many times. So if your VM's are not getting internet connection do not panic ... follow these steps
::
    // Use ping command to see whether internet is on.
    $ping google.com
    // If its not connected restart networking service-
    $sudo service networking restart
    // Now Ping again
    $ping google.com

This should reconnect your network about 99% of the times. If you are really unlucky you must be having some other problems or your internet connection itself is not functioning... well try to avoid immature decisions. Believe me you dont want to mess up your existing setup.


7. Controller Node
==============

7.1. Preparing Ubuntu 12.10/12.04
------------

* If your installation is Ubuntu 12.04 Server,
   
   To access Folsom from Ubuntu archive, please add the following entries to your /etc/apt/sources.list:
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main
   For more information `follow this link <http://www.ubuntu.com/download/help/cloud-archive-instructions>`_ steps to access OpenStack Folsom archives

* After you install Ubuntu 12.10 Server 64bits,

   sudo su

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade


7.2.Networking
------------

Configure your network by editing :: /etc/network/interfaces file

* Only one NIC on the controller node need internet access::
  
    # NAT should be preconfigured otherwise can copy the following ...
    # This file describes the network interfaces available on your system
    # and how to activate them. For more information, see interfaces(5).

    # The loopback network interface
    auto lo
    iface lo inet loopback
    
    # The primary network interface - Virtual Box NAT connection
    auto eth2
    iface eth2 inet dhcp
    
    # Virtual Box vboxnet0 - Openstack Management Network
    auto eth0
    iface eth0 inet static
    address 100.10.10.51
    netmask 255.255.255.0
    gateway 100.10.10.1
  
    # Virtual Box vboxnet2 - for exposing Openstack API over external network
    auto eth1
    iface eth1 inet static
    address 192.168.100.51
    netmask 255.255.255.0
    gateway 192.168.100.1



For the remaining Installation Follow `OpenStack-Folsom-Install-guide 2. Control Node <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/OpenStack_Folsom_Install_Guide_WebVersion.rst>`_


8. Network Node
==============

8.1. Preparing the Node
------------------


* If your installation is Ubuntu 12.04 Server,
   
   To access Folsom from Ubuntu archive, please add the following entries to your /etc/apt/sources.list:
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main
   For more information `follow this link <http://www.ubuntu.com/download/help/cloud-archive-instructions>`_ steps to access OpenStack Folsom archives

* After you install Ubuntu 12.10 Server 64bits,

   sudo su

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

8.2.Networking
------------

* 4 NICs must be present::
   
    # This file describes the network interfaces available on your system
    # and how to activate them. For more information, see interfaces(5).

    # The loopback network interface
    auto lo
    iface lo inet loopback

    # The primary network interface - Virtual Box NAT connection
    auto eth3
    iface eth3 inet dhcp


    # vboxnet0  - OpenStack Management Netowork
    auto eth0
    iface eth0 inet static
    address 100.10.10.52
    netmask 255.255.255.0
    gateway 100.10.10.1

    # vboxnet1 - OpenStack VM Conf. Network
    auto eth1
    iface eth1 inet static
    address 100.20.20.52
    netmask 255.255.255.0
    gateway 100.20.20.1

    # vboxnet2 - Expose OpenStack API's to external network.
    auto eth2
    iface eth2 inet static
    address 192.168.100.52
    netmask 255.255.255.0
    gateway 192.168.100.1


For the remaining Installation Follow `OpenStack-Folsom-Install-guide 3. Network Node <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/OpenStack_Folsom_Install_Guide_WebVersion.rst>`_


9. Compute Node
==============

9.1. Preparing the Node
------------------


* If your installation is Ubuntu 12.04 Server,
   
   To access Folsom from Ubuntu archive, please add the following entries to your /etc/apt/sources.list:
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main
   For more information `follow this link <http://www.ubuntu.com/download/help/cloud-archive-instructions>`_ steps to access OpenStack Folsom archives

* After you install Ubuntu 12.10 Server 64bits,

   sudo su

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

9.2.Networking
------------

* 3 NICs must be present::
                                           

    # This file describes the network interfaces available on your system
    # and how to activate them. For more information, see interfaces(5).
    
    # The loopback network interface
    auto lo
    iface lo inet loopback
    
    # The primary network interface - Virtual Box NAT connection
    auto eth2
    iface eth2 inet dhcp
    
    # Virtual Box vboxnet0 - Openstack Management Network
    auto eth0
    iface eth0 inet static
    address 100.10.10.53
    netmask 255.255.255.0
    gateway 100.10.10.1
    
    # Virtual Box vboxnet1 - for exposing Openstack API over external network
    auto eth1
    iface eth1 inet static
    address 100.20.20.53
    netmask 255.255.255.0
    gateway 100.20.20.1
    
    
    
For the remaining Installation Follow `OpenStack-Folsom-Install-guide 4. Compute Node <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/OpenStack_Folsom_Install_Guide_WebVersion.rst>`_

After Finishing With the Guide's Steps ... please do the following Changes.

4.3 KVM
------------------

* your hardware does not support virtualization because it is a virtual machine itselves ::

   apt-get install cpu-checker
   kvm-ok

* If you are using VMWare then you may get a good response. install 

* Edit /etc/nova/nova-compute.conf file again and change 'kvm' to 'qemu' leave the rest as it is::
   
   [DEFAULT]
   libvirt_type=qemu
   
* Now if you try to launch virtual machine instances they will work. 

**Note :** This is for SandBoxing purposes only. Ideal for learning and testing, checking out OpenStack. If you want proper working you must have physical machines working.

10. Configure the internal networks
==============



11. Word Of Advice.
==============

* On any condition do not restart - shutdown your VM's, just Save the machine state.






12. Licensing
============

OpenStack Folsom Install Guide by Bilel Msekni is licensed under a Creative Commons Attribution 3.0 Unported License.

.. image:: http://i.imgur.com/4XWrp.png
To view a copy of this license, visit [ http://creativecommons.org/licenses/by/3.0/deed.en_US ].

13. Contacts
===========

Pranav Salunke: pps.pranav@gmail.com
Bilel Msekni: bilel.msekni@telecom-sudparis.eu

14. Acknowledgment
=================

This work has been supported by:

* CompatibleOne Project (French FUI project) [http://compatibleone.org/]
* Easi-Clouds (ITEA2 project) [http://easi-clouds.eu/]

15. Credits
=================

This work has been based on:

* Emilien Macchi's Folsom guide [https://github.com/EmilienM/openstack-folsom-guide]
* OpenStack Documentation [http://docs.openstack.org/trunk/openstack-compute/install/apt/content/]
* OpenStack Quantum Install [http://docs.openstack.org/trunk/openstack-network/admin/content/ch_install.html]

16. To do
=======

This guide is just a startup. Your suggestions are always welcomed.

