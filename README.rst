==========================================================
  OpenStack Folsom Install Guide
==========================================================

:Version: 1.0
:Source: https://github.com/mseknibilel/OpenStack-Folsom-Install-guide
:Keywords: OpenStack, Folsom, Quantum, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM.

Developers
==========

Copyright (C) Bilel Msekni <bilel.msekni@telecom-sudparis.eu>

Redistribution of this software is permitted under the terms of the **LGPL** License

Table of Contents
=================

::

  0. What is it?
  1. Requirements
  2. Getting Ready
  3. Keystone 
  4. Glance
  5. KVM
  6. OpenVSwitch
  7. Quantum
  8. Nova
  9. Cinder
  10. Horizon
  11. Start your first VM


0. What is it?
==============

OpenStack Folsom Install Guide is an easy and tested way to create your own OpenStack plateform. 

version 1.0

10 Oct 2012

status: Still an ongoing work


1. Requirements
====================

:Node Role: NICs.
:Controller Node: eth0 (157.159.100.232), eth1 (157.159.100.234).
:Compute Node: eth0 (157.159.100.250), eth1 (157.159.100.252).

2. Getting Ready
===============

2.1. Adding the Offical Folsom repositories
-----------------

* Go to the sudo mode and stay in it until the end of this guide::

   sudo su

* Add the OpenStack Folsom repositories to your ubuntu repositories::

   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main >> /etc/apt/sources.list.d/folsom.list
   apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 5EDB1B62EC4926EA

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

2.2. MySQL & RabbitMQ
------------

* Install MySQL::

   apt-get install mysql-server python-mysqldb

* Configure mysql to accept all incoming requests::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* Install RabbitMQ::

   apt-get install rabbitmq-server 

2.3. Node synchronization
------------------

* Install other services::

   apt-get install ntp

* Configure the NTP server to synchronize between your compute nodes and the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
   service ntp restart  

2.4. Others
-------------------
* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   nano /etc/sysctl.conf
   #Uncomment net.ipv4.ip\_forward=1

________________________________________________________________________________________________________________________

4. Keystone
=====================================================================

This is how we install OpenStack's identity service:

* Start by the keystone packages::

   apt-get install keystone python-keystone python-keystoneclient

* Create a new MySQL database for keystone::

   mysql -u root -p
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   quit;

* Adapt the connection attribute in the /etc/keystone/keystone.conf to the new database::

   connection = mysql://keystoneUser:keystonePass@157.159.100.232/keystone

* Restart the identity service then synchronize the database::

   service keystone restart
   keystone-manage db_sync

* Fill up the keystone database using the two scripts available.Beware that you MUST modify the HOST_IP variable before executing the scripts::

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
   export OS_AUTH_URL="http://157.159.100.232:5000/v2.0/"
   # Load it:
   source creds

* To test Keystone, we use a simple curl request::

   apt-get install curl openssl
   curl http://157.159.100.232:35357/v2.0/endpoints -H 'x-auth-token: ADMIN'

________________________________________________________________________________________________________________________

________________________________________________________________________________________________________________________

4. Glance
=====================================================================

* After installing Keystone, we continue with installing image storage service a.k.a Glance::

   apt-get install glance python-glance python-glanceclient

* Create a new MySQL database for Glance::

   mysql -u root -p
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
   quit;

* Update /etc/glance/glance-api-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 157.159.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update the /etc/glance/glance-registry-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 157.159.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update /etc/glance/glance-api.conf with::

   sql_connection = mysql://glanceUser:glancePass@157.159.100.232/glance

* And::

   [paste_deploy]
   flavor = keystone

* Update the /etc/glance/glance-registry.conf with::

   sql_connection = mysql://glanceUser:glancePass@157.159.100.232/glance

* And::

   [paste_deploy]
   flavor = keystone

* Restart the glance-api and glance-registry services::

   service glance-api restart; service glance-registry restart

* Synchronize the glance database::

   glance-manage db_sync

* Restart the services again to take into account the new modifications::

   service glance-registry restart; service glance-api restart

* To test Glance's well installation, we upload a new image to the store. Start by downloading an ubuntu cloud image to your node and then uploading it to Glance::

   mkdir images
   cd images
   wget http://uec-images.ubuntu.com/releases/precise/release/ubuntu-12.04-server-cloudimg-amd64.tar.gz
   tar xzvf ubuntu-12.04-server-cloudimg-amd64.tar.gz
   glance add name="Ubuntu" is_public=true container_format=ovf disk_format=qcow2 < precise-server-cloudimg-amd64.img

* Now list the images to see what you have just uploaded::

   glance image-list

________________________________________________________________________________________________________________________

________________________________________________________________________________________________________________________

* Create Resources of a kind::

   curl -X POST -d@post_resources.json -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{resource}/

* Create a Resource with a custom URL path::

   curl -X PUT -d@post_custom_resource.json -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{resource}/{user_id}/{my_custom_resource_id}

* Get a Resource::

   curl -X GET -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{resource}/{user-id}/{resource-id}

* Full Update a Resource::

   curl -X PUT -d@full_update_resource.json -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{resource}/{user-id}/{resource-id}

* Partial Update a Resource::

   curl -X POST -d@partial_update_resource.json -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{resource}/{user-id}/{resource-id}

* Delete a Resource::

   curl -X DELETE -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{resource}/{user-id}/{resource-id}

________________________________________________________________________________________________________________________

________________________________________________________________________________________________________________________

* Create Links of a kind::

   curl -X POST -d@post_links.json -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{link}/

* Create a Link with a custom resource path::

   curl -X PUT -d@post_custom_resource.json -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{my_custom_link_path}

* Get a Link::

   curl -X GET -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{link}/{user-id}/{link-id}

* Full update a Link::

   curl -X PUT -d@full_update_link.json -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{link}/{user-id}/{link-id}

* Patial update a Link::

   curl -X POST -d@partial_update_link.json -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{link}/{user-id}/{link-id}

* Delete a link::

   curl -X DELETE -H 'content-type: application/occi+json' -H 'accept: application/occi+json' --user user_1:pass -v http://localhost:8090/{link}/{user-id}/{link-id}

________________________________________________________________________________________________________________________

5. For developers
=================

If you want export the use of your service through OCCI, two parts should be developped:

#. the definition of the kind, action, and mixin with the list of attributes
#. implementation of the specific service backend (CRUD operations)


6. Licensing
============

::

  Copyright (C) 2011 Houssem Medhioub - Institut Mines-Telecom

  This library is free software: you can redistribute it and/or modify
  it under the terms of the GNU Lesser General Public License as
  published by the Free Software Foundation, either version 3 of
  the License, or (at your option) any later version.

  This library is distributed in the hope that it will be useful,
  but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  GNU Lesser General Public License for more details.

  You should have received a copy of the GNU Lesser General Public License
  along with this library. If not, see <http://www.gnu.org/licenses/>.

7. Contacts
===========

Houssem Medhioub: houssem.medhioub@it-sudparis.eu

Bilel Msekni: bilel.msekni@telecom-sudparis.eu

Djamal Zeghlache: djamal.zeghlache@it-sudparis.eu

8. Acknowledgment
=================
This work has been supported by:

* SAIL project (IST 7th Framework Programme Integrated Project) [http://sail-project.eu/]
* CompatibleOne Project (French FUI project) [http://compatibleone.org/]


9. Todo
=======
This release of pyocni is experimental.

Some of pyocni's needs might be:

*

10. json files to execute the HowTo use examples (available under client/request_examples folder)
=======================================================================

* post_actions.json::

   {
       "actions": [
           {
               "term": "stop",
               "scheme": "http://schemas.ogf.org/occi/infrastructure/compute/action#",
               "title": "Stop Compute instance",
               "attributes": {
                   "method": {
                       "mutable": true,
                       "required": false,
                       "type": "string",
                       "pattern": "graceful|acpioff|poweroff",
                       "default": "poweroff"
                   }
               }
           }
       ]
   }

* get_actions.json::

   {
       "actions": [
           {
               "term": "stop",
               "scheme": "http://schemas.ogf.org/occi/infrastructure/compute/action#",
               "title": "Stop Compute instance",
               "attributes": {
                   "method": {
                       "mutable": true,
                       "required": false,
                       "type": "string",
                       "pattern": "graceful|acpioff|poweroff",
                       "default": "poweroff"
                   }
               }
           }
       ]
   }

* put_actions.json::

    {
        "actions": [
            {
                "attributes": {
                    "method": {
                        "default": "poweroff",
                        "mutable": true,
                        "required": false,
                        "type": "string",
                        "pattern": "graceful|acpioff|poweroff"
                    }
                },
                "term": "start",
                "scheme": "http://schemas.ogf.org/occi/infrastructure/compute/action#",
                "title": "start Compute instance"
            }
        ]
    }

* delete_actions.json::

    {
        "actions": [
            {
                "term": "stop",
                "scheme": "http://schemas.ogf.org/occi/infrastructure/compute/action#"
            }
        ]
    }

* post_mixins.json::

   {
       "mixins": [
           {
               "term": "medium",
               "scheme": "http://example.com/template/resource#",
               "title": "Medium VM",
               "related": [
                   "http://schemas.ogf.org/occi/infrastructure#resource_tpl"
               ],
               "attributes": {
                   "occi": {
                       "compute": {
                           "speed": {
                               "type": "number",
                               "default": 2.8
                           }
                       }
                   }
               },
               "location": "/template/resource/medium/"
           }
       ]
   }

* get_mixins.json::

   {
       "mixins": [
           {
               "term": "medium",
               "scheme": "http://example.com/template/resource#",
               "title": "Medium VM",
               "related": [
                   "http://schemas.ogf.org/occi/infrastructure#resource_tpl"
               ],
               "attributes": {
                   "occi": {
                       "compute": {
                           "speed": {
                               "type": "number",
                               "default": 2.8
                           }
                       }
                   }
               },
               "location": "/template/resource/medium/"
           }
       ]
   }

* put_mixins.json::

    {
        "mixins": [
            {
                "term": "medium",
                "scheme": "http://example.com/template/resource#",
                "title": "Large VM",
                "related": [
                    "http://schemas.ogf.org/occi/infrastructure#resource_tpl"
                ],
                "attributes": {
                    "occi": {
                        "compute": {
                            "speed": {
                                "type": "number",
                                "default": 3
                            }
                        }
                    }
                },
                "location": "/template/resource/medium/"
            }
        ]
    }

* delete_mixins.json::

   {
       "mixins": [
           {
               "term": "medium",
               "scheme": "http://example.com/template/resource#"
           }
       ]
   }

* post_categories.json::

    {
        "actions": [
            {
                "term": "start",
                "scheme": "http://schemas.ogf.org/occi/infrastructure/compute/action#",
                "title": "Stop Compute instance",
                "attributes": {
                    "method": {
                        "mutable": true,
                        "required": false,
                        "type": "string",
                        "pattern": "graceful|acpioff|poweroff",
                        "default": "poweroff"
                    }
                }
            }
        ],
        "kinds": [
            {
                "term": "storage",
                "scheme": "http://schemas.ogf.org/occi/infrastructure#",
                "title": "Compute Resource",
                "attributes": {
                    "occi": {
                        "compute": {
                            "hostname": {
                                "mutable": true,
                                "required": false,
                                "type": "string",
                                "pattern": "(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\\\\-]*[a-zA-Z0-9])\\\\.)*",
                                "minimum": "1",
                                "maximum": "255"
                            },
                            "state": {
                                "mutable": false,
                                "required": false,
                                "type": "string",
                                "pattern": "inactive|active|suspended|failed",
                                "default": "inactive"
                            }
                        }
                    }
                },
                "actions": [
                    "http://schemas.ogf.org/occi/infrastructure/compute/action#start"
                ],
                "location": "/storage/"
            }
        ],
        "mixins": [
            {
                "term": "resource_tpl",
                "scheme": "http://schemas.ogf.org/occi/infrastructure#",
                "title": "Medium VM",
                "related": [],
                "attributes": {
                    "occi": {
                        "compute": {
                            "speed": {
                                "type": "number",
                                "default": 2.8
                            }
                        }
                    }
                },
                "location": "/template/resource/resource_tpl/"
            }
        ]
    }


* filter_categories.json::

    {
        "actions": [
            {
                "term": "start",
                "scheme": "http://schemas.ogf.org/occi/infrastructure/compute/action#",
                "title": "Stop Compute instance",
                "attributes": {
                    "method": {
                        "mutable": true,
                        "required": false,
                        "type": "string",
                        "pattern": "graceful|acpioff|poweroff",
                        "default": "poweroff"
                    }
                }
            }
        ],
        "kinds": [
            {
                "term": "storage",
                "scheme": "http://schemas.ogf.org/occi/infrastructure#",
                "title": "Compute Resource",
                "attributes": {
                    "occi": {
                        "compute": {
                            "hostname": {
                                "mutable": true,
                                "required": false,
                                "type": "string",
                                "pattern": "(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\\\\-]*[a-zA-Z0-9])\\\\.)*",
                                "minimum": "1",
                                "maximum": "255"
                            },
                            "state": {
                                "mutable": false,
                                "required": false,
                                "type": "string",
                                "pattern": "inactive|active|suspended|failed",
                                "default": "inactive"
                            }
                        }
                    }
                },
                "actions": [
                    "http://schemas.ogf.org/occi/infrastructure/compute/action#start"
                ],
                "location": "/storage/"
            }
        ],
        "mixins": [
            {
                "term": "resource_tpl",
                "scheme": "http://schemas.ogf.org/occi/infrastructure#",
                "title": "Medium VM",
                "related": [],
                "attributes": {
                    "occi": {
                        "compute": {
                            "speed": {
                                "type": "number",
                                "default": 2.8
                            }
                        }
                    }
                },
                "location": "/template/resource/resource_tpl/"
            }
        ]
    }

* put_categories.json::

    {
        "actions": [
            {
                "term": "start",
                "scheme": "http://schemas.ogf.org/occi/infrastructure/compute/action#",
                "title": "Stop Compute instance",
                "attributes": {
                    "method": {
                        "mutable": true,
                        "required": false,
                        "type": "string",
                        "pattern": "graceful|acpioff|poweroff",
                        "default": "poweroff"
                    }
                }
            }
        ],
        "kinds": [
            {
                "term": "storage",
                "scheme": "http://schemas.ogf.org/occi/infrastructure#",
                "title": "Compute Resource",
                "attributes": {
                    "occi": {
                        "compute": {
                            "hostname": {
                                "mutable": true,
                                "required": false,
                                "type": "string",
                                "pattern": "(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\\\\-]*[a-zA-Z0-9])\\\\.)*",
                                "minimum": "1",
                                "maximum": "255"
                            },
                            "state": {
                                "mutable": false,
                                "required": false,
                                "type": "string",
                                "pattern": "inactive|active|suspended|failed",
                                "default": "inactive"
                            }
                        }
                    }
                },
                "actions": [
                    "http://schemas.ogf.org/occi/infrastructure/compute/action#start"
                ],
                "location": "/storage/"
            }
        ],
        "mixins": [
            {
                "term": "resource_tpl",
                "scheme": "http://schemas.ogf.org/occi/infrastructure#",
                "title": "Medium VM",
                "related": [],
                "attributes": {
                    "occi": {
                        "compute": {
                            "speed": {
                                "type": "number",
                                "default": 2.8
                            }
                        }
                    }
                },
                "location": "/template/resource/resource_tpl/"
            }
        ],
        "providers": [
            {
                "Provider": {
                    "local": [
                        "Houssem"
                    ],
                    "remote": [
                        "Bilel"
                    ]
                },
                "OCCI_ID": "http://schemas.ogf.org/occi/core#resource"
            }
        ]
    }

* delete_categories.json::

    {
        "actions": [
            {
                "term": "start",
                "scheme": "http://schemas.ogf.org/occi/infrastructure/compute/action#"
            }
        ],
        "kinds": [
            {
                "term": "storage",
                "scheme": "http://schemas.ogf.org/occi/infrastructure#"
            }
        ],
        "mixins": [
            {
                "term": "resource_tpl",
                "scheme": "http://schemas.ogf.org/occi/infrastructure#"
            }
        ]
    }

* post_kinds.json::

   {
       "kinds": [
           {
               "term": "compute",
               "scheme": "http://schemas.ogf.org/occi/infrastructure#",
               "title": "Compute Resource",
               "related": [
                   "http://schemas.ogf.org/occi/core#resource"
               ],
               "attributes": {
                   "occi": {
                       "compute": {
                           "hostname": {
                               "mutable": true,
                               "required": false,
                               "type": "string",
                               "pattern": "(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\\\\-]*[a-zA-Z0-9])\\\\.)*",
                               "minimum": "1",
                               "maximum": "255"
                           },
                           "state": {
                               "mutable": false,
                               "required": false,
                               "type": "string",
                               "pattern": "inactive|active|suspended|failed",
                               "default": "inactive"
                           }
                       }
                   }
               },
               "actions": [
                   "http://schemas.ogf.org/occi/infrastructure/compute/action#start",
                   "http://schemas.ogf.org/occi/infrastructure/compute/action#stop",
                   "http://schemas.ogf.org/occi/infrastructure/compute/action#restart"
               ],
               "location": "/compute/"
           }
       ]
   }

* get_kinds.json::

   {
       "kinds": [
           {
               "term": "compute",
               "scheme": "http://schemas.ogf.org/occi/infrastructure#",
               "title": "Compute Resource",
               "related": [
                   "http://schemas.ogf.org/occi/core#resource"
               ],
               "attributes": {
                   "occi": {
                       "compute": {
                           "hostname": {
                               "mutable": true,
                               "required": false,
                               "type": "string",
                               "pattern": "(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\\\\-]*[a-zA-Z0-9])\\\\.)*",
                               "minimum": "1",
                               "maximum": "255"
                           },
                           "state": {
                               "mutable": false,
                               "required": false,
                               "type": "string",
                               "pattern": "inactive|active|suspended|failed",
                               "default": "inactive"
                           }
                       }
                   }
               },
               "actions": [
                   "http://schemas.ogf.org/occi/infrastructure/compute/action#start",
                   "http://schemas.ogf.org/occi/infrastructure/compute/action#stop",
                   "http://schemas.ogf.org/occi/infrastructure/compute/action#restart"
               ],
               "location": "/compute/"
           }
       ]
   }

* put_kinds.json::

    {
        "Kinds": [
            {
                "term": "compute",
                "title": "Compute Resource",
                "related": [
                    "http://schemas.ogf.org/occi/core#resource"
                ],
                "actions": [],
                "attributes": {
                    "occi": {
                        "compute": {
                            "state": {
                                "default": "inactive",
                                "mutable": false,
                                "required": false,
                                "type": "string",
                                "pattern": "inactive|active|suspended|failed"
                            },
                            "hostname": {
                                "pattern": "(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\\\\-]*[a-zA-Z0-9])\\\\.)*",
                                "required": false,
                                "maximum": "255",
                                "minimum": "1",
                                "mutable": true,
                                "type": "string"
                            }
                        }
                    }
                },
                "scheme": "http://schemas.ogf.org/occi/infrastructure#",
                "location": "/compute/"
            }
        ]
    }

* put_providers.json::

    {
        "providers": [
            {
                "Provider": {
                    "local": [
                        "Houssem"
                    ],
                    "remote": [
                        "Bilel"
                    ]
                },
                "OCCI_ID": "http://schemas.ogf.org/occi/infrastructure#compute"
            }
        ]
    }


* delete_kinds.json::

    {
        "Kinds": [
            {
                    "term": "compute",
                    "scheme": "http://schemas.ogf.org/occi/infrastructure#"
            }
        ]
    }

* post_resources.json::

   {
       "resources": [
           {
               "kind": "http: //schemas.ogf.org/occi/infrastructure#compute",
               "mixins": [
                   "http: //schemas.opennebula.org/occi/infrastructure#my_mixin",
                   "http: //schemas.other.org/occi#my_mixin"
               ],
               "attributes": {
                   "occi": {
                       "compute": {
                           "speed": 2,
                           "memory": 4,
                           "cores": 2
                       }
                   },
                   "org": {
                       "other": {
                           "occi": {
                               "my_mixin": {
                                   "my_attribute": "my_value"
                               }
                           }
                       }
                   }
               },
               "actions": [
                   {
                       "title": "Start My Server",
                       "href": "/compute/996ad860-2a9a-504f-8861-aeafd0b2ae29?action=start",
                       "category": "http://schemas.ogf.org/occi/infrastructure/compute/action#start"
                   }
               ],
               "id": "996ad860-2a9a-504f-8861-aeafd0b2ae29",
               "title": "Compute resource",
               "summary": "This is a compute resource",
               "links": [
                   {
                       "target": "http://myservice.tld/storage/59e06cf8-f390-5093-af2e-3685be593",
                       "kind": "http: //schemas.ogf.org/occi/infrastructure#storagelink",
                       "attributes": {
                           "occi": {
                               "storagelink": {
                                   "deviceid": "ide: 0: 1"
                               }
                           }
                       },
                       "id": "391ada15-580c-5baa-b16f-eeb35d9b1122",
                       "title": "Mydisk"
                   }
               ]
           }
       ]
   }

* full_update_resource.json::

   {
       "_id": "fb1cff2a-641c-47b2-ab50-0e340bce9cc2",
       "_rev": "2-8d02bacda9bcb93c8f03848191fd64f0"

   }

* post_links.json::

   {
       "links": [
           {
               "kind": "http://schemas.ogf.org/occi/infrastructure#networkinterface",
               "mixins": [
                   "http://schemas.ogf.org/occi/infrastructure/networkinterface#ipnetworkinterface"
               ],
               "attributes": {
                   "occi": {
                       "infrastructure": {
                           "networkinterface": {
                               "interface": "eth0",
                               "mac": "00:80:41:ae:fd:7e",
                               "address": "192.168.0.100",
                               "gateway": "192.168.0.1",
                               "allocation": "dynamic"
                           }
                       }
                   }
               },
               "actions": [
                   {
                       "title": "Disable networkinterface",
                       "href": "/networkinterface/22fe83ae-a20f-54fc-b436-cec85c94c5e8?action=up",
                       "category": "http: //schemas.ogf.org/occi/infrastructure/networkinterface/action#"
                   }
               ],
               "id": "22fe83ae-a20f-54fc-b436-cec85c94c5e8",
               "title": "Mynetworkinterface",
               "target": "http: //myservice.tld/network/b7d55bf4-7057-5113-85c8-141871bf7635",
               "source": "http: //myservice.tld/compute/996ad860-2a9a-504f-8861-aeafd0b2ae29"
           }
       ]
   }

* full_update_link.json::

   {
       "_id": "fb1cff2a-641c-47b2-ab50-0e340bce9cc2",
       "_rev": "2-8d02bacda9bcb93c8f03848191fd64f0"
   }

* DocumentSkeleton::

   {
       "_id": "id value",
       "_rev": "rev value",
       "LastUpdate": "datetime",
       "CreationDate": "datetime",
       "OCCI_Description": {
       },
       "Creator": "creator login",
       "OCCI_ID": "scheme+term",
       #OCCI_Location not available for actions documents
       "OCCI_Location": "path to the document",
       "Type": "Type of the OCCI description",
       #Provider field is available only in kind documents
       "Provider": {
           "remote": [

           ],
           "local": [

           ]
       }
   }

