==========================================================
  Modify the iptables manager
==========================================================

:Version: 1.0
:Keywords: OpenStack, Quantum, l3_agent, Ubuntu Server 12.04 LTE (64 bits).
:Status: Stable

Authors
==========

Copyright (C) Bilel Msekni <bilel.msekni@telecom-sudparis.eu>

This work is licensed under the Creative Commons Attribution-ShareAlike 3.0 Unported License.
 
To view a copy of this license, visit [ http://creativecommons.org/licenses/by-sa/3.0/ ].

Contributors
==========

**Wana contribute ? Read the idea, send your contribution and get your name listed ;)**

0. What is this about?
==============

When you aim at using a single router for all of your tenants, you have to set router_id attribute in the l3_agent or once you do so, you trigger a malfunction in the quantum l3_agent. This problem has been fixed but the packaging is not yet ready so you have to do the work manually

1. How to fix it?
====================

* Dive inside OpenStack code at cd /usr/lib/python2.7/dist-packages/quantum/agent/linux/iptables_manager.py::

   sudo nano /usr/lib/python2.7/dist-packages/quantum/agent/linux/iptables_manager.py

* Get to the line 272 (use Ctrl+c to check your position)::
   
   # Replace this line : s = [('/sbin/iptables', self.ipv4)] with s = [('iptables', self.ipv4)]

* Exit, save and restart the l3_agent::

   service quantum-l3-agent restart 

**That's it**, if you want to learn more about this bug go to: https://review.openstack.org/#/c/14811/1/quantum/agent/linux/iptables_manager.py

2. I have a better idea!
====================

You have a better idea ? Share it with us and get it named after you :)  


