==========================================================
  How to load a volume group after a system reboot?
==========================================================

:Version: 1.0
:Keywords: OpenStack, Cinder, Ubuntu Server 12.04 LTE (64 bits).
:Status: Stable

Authors
==========

Copyright (C) Bilel Msekni <bilel.msekni@telecom-sudparis.eu>

This work is licensed under the Creative Commons Attribution-ShareAlike 3.0 Unported License.
 
To view a copy of this license, visit [ http://creativecommons.org/licenses/by-sa/3.0/ ].

Contributors
==========

* Marco Consonni [marco_consonni@hp.com]
* Dennis E Miyoshi [dennis.miyoshi@hp.com]

**Wana contribute ? Read the idea, send your contribution and get your name listed ;)**

0. What is this about?
==============

The Cinder service uses a volume group usually named cinder-volumes to provide storage for VMs. Unfortunatly, this volume group gets lost after a system reboot which causes failure when there is a volume provisioning.
There are multiple ways to fix this, two of them are described below.

1. Using rc.local
=================

* To automatically load the volume group after a system reboot, proceed like the following::

   sudo su
   nano /etc/rc.local

* Add the following line to the rc.local file **before the exit 0 line**::
   
   losetup /dev/loop2 %Your_path_to_cinder_volumes%

* Exit and save::

**That's it**, try it out by rebooting your system and then run the vgdisplay command.

2. Using upstart
================

* Create an upstart script to create the loopback device as soon as the mount is available::

    sudo vim /etc/init/losetup.conf

* Add the following lines::

    description     "Set up loop devices"

    start on mounted MOUNTPOINT=/
    task
    exec /sbin/losetup /dev/loop0 %Your_path_to_cinder_volume%

* Make sure that MOUNTPOINT in the above configuration is set to the mount point your loopback file is on,
  and you substitute the Your_path_to_cinder_volume variable.

3. I have a better idea!
====================

You have a better idea ? Share it with us and get it named after you :)  


