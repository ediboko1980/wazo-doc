.. _contact_and_presence_sharing:

****************************
Contact and Presence Sharing
****************************

Wazo allow the administrator to share presence and statuses between multiple
installations. For example, an enterprise could have a Wazo in each office and
still be able to search, contact and view the statuses of colleagues in other
offices.

This page will describe the steps that are required to configure such use case.

.. figure:: images/presence_sharing_diagram.png


Prerequisite
============

#. All Wazo that you interconnect should be on the same version
#. All ports necessary for communication should be open :ref:`network_ports`

.. warning::

   If you are cloning a virtual machine or copy the database, the UUID of the
   two Wazo will be the same, you must regenerate them in the *infos* table of
   the *asterisk* database and restart all services. You must also remove all
   consul data that included the old UUID.

.. warning::

   Telephony will be interrupted during the configuration period.

.. warning::

   The configuration must be applied to each Wazo you want to interconnect. For
   example, if 6 different Wazo are to be connected, the configuration for all
   other Wazo should be added. This does not apply to the message bus which can
   use a ring policy, each Wazo talking to its two neighbours.

.. warning::

   You should use your firewall to restrict access to the HTTP ports of consul
   and xivo-ctid, because they don't have any authentication mechanism enabled.

.. note::

   In an architecture with a lot of Wazo, we recommend that you centralize some
   services, such as xivo-dird, to make your life easier. Don't forget
   redundancy. This applies also to RabbitMQ and Consul. In this case, the
   configuration will have to be done entirely manually in YAML config files.


For this procedure, the following name and IP addresses will be used:

* Wazo 1: 192.168.1.124
* Wazo 2: 192.168.1.125


Configure service discovery on all wazo-services
================================================

The default configuration for each services in wazo uses the IP address of the "eth0"
interface as it's advertised IP address. To advertise a different address you will have
to override the default configuration.

To do se you have to create a file in the conf.d directory of all services.

First, create a configuration file that will be used for all services.

.. code-block:: yaml

   service_discovery:
     advertise_address: 192.168.1.124

or

.. code-block:: yaml

   service_discovery:
     advertise_address: auto
     advertise_address_interface: "<interface of the address to advertise. ex: enp0s3>"


Save that file and create a symlink such that each service will be able to used that file.

.. code-block:: sh

  for dir in /etc/{xivo,wazo}-*/conf.d; do ln -sf /root/config/service_discovery.yml $dir/service_discovery.yml; done


.. _create_ws_user:

Add a Web Service User
======================

The first thing to do is to create a new web service access to be able to search users and get
there presences using the following ACL.

* ctid-ng.lines.*.presences.read
* ctid-ng.users.*.presences.read
* confd.users.read

This can be done in :menuselection:`Configuration --> Management --> Web Services Access`

.. figure:: images/create_user_ws.png
.. figure:: images/create_user_ws_acl.png


Configuring the directories
===========================

Add New Directory Sources for Remote Wazo
-----------------------------------------

For each remote Wazo a new directory has to be created in
:menuselection:`Configuration --> Management --> Directories`

.. note:: We recommend doing a working configuration without certificate
          verification first. Once you get it working, enable certificate
          verification.

.. figure:: images/list_directory.png
.. figure:: images/create_directory.png


Add a Directory Definition for Each New Directories
---------------------------------------------------

To add a new directory definition, go to :menuselection:`Services --> CTI Server
--> Directories --> Definitions`

.. figure:: images/list_definition.png

In each directory definition, add the fields to match the configured *Display filters*

.. figure:: images/create_definition.png


Add the New Definitions to Your Dird Profiles
---------------------------------------------

At the moment of this writing xivo-dird profiles are mapped directly to the
user's profile. For each internal context where you want to be able to see
user's from other Wazo, add the new directory definitions in
:menuselection:`Services --> CTI Server --> Directories --> Direct directories`.

.. figure:: images/list_direct_directories.png
.. figure:: images/create_direct_directories.png


Restart xivo-dird
-----------------

To apply the new directory configuration, you can either restart from:

* :menuselection:`Services --> IPBX`
* on the command line *service xivo-dird restart*


Check that the Configuration is Working
---------------------------------------

At this point, you should be able to search for users on other Wazo from the
:ref:`people-xlet`.


Configuring RabbitMQ
====================

Create a RabbitMQ user
----------------------

.. code-block:: sh

    rabbitmqctl add_user xivo xivo
    rabbitmqctl set_user_tags xivo administrator
    rabbitmqctl set_permissions -p / xivo ".*" ".*" ".*"
    rabbitmq-plugins enable rabbitmq_federation


Restart RabbitMQ
----------------

.. code-block:: sh

    systemctl restart rabbitmq-server


Setup Message Federation
------------------------

.. code-block:: sh

    rabbitmqctl set_parameter federation-upstream xivo-dev-2 '{"uri":"amqp://xivo:xivo@192.168.1.125","max-hops":1}'  # remote IP address
    rabbitmqctl set_policy federate-xivo 'xivo' '{"federation-upstream-set":"all"}' --priority 1 --apply-to exchanges


Check That Service Discovery is Working
---------------------------------------

.. code-block:: sh

    apt-get install consul-cli

.. code-block:: sh

    consul-cli agent services --ssl --ssl-verify=false

The output should include a service names *xivo-ctid* with an address that is
reachable from other XiVO.

.. code-block:: javascript

    {"consul": {"ID": "consul",
                "Service": "consul",
                "Tags": [],
                "Port": 8300,
                "Address": ""},
     "e546a652-e290-47e2-8519-ec3642daa6e6": {"ID": "e546a652-e290-47e2-8519-ec3642daa6e6",
                                              "Service": "xivo-ctid",
                                              "Tags": ["xivo-ctid",
                                                       "607796fc-24e2-4e26-8009-cbb48a205512"],
                                              "Port": 9495,
                                              "Address": "192.168.1.124"}}


Configure Ctid-ng
=================

Add a configuration file on ctid-ng conf.d directory named discovery.yml with your configuration.

The `service_id` and `service_key` are the ones you defined :ref:`earlier <create_ws_user>` in the web interface.

.. code-block:: yaml

    remote_credentials:
      xivo-2:
        xivo_uuid: 1cc7fbf2-5f13-4898-9869-986990cb9b0a
        service_id: remote-directory
        service_key: supersecret

To get the xivo_uuid information on your second Wazo, use the command:

.. code-block:: sh

    echo $XIVO_UUID


Restart xivo-ctid-ng
--------------------

.. code-block:: sh

    systemctl restart xivo-ctid-ng


Configure Consul
================

Stop Wazo
---------

.. code-block:: sh

    wazo-service stop
    systemctl stop consul


Remove All Consul Data
----------------------

.. code-block:: sh

    rm -rf /var/lib/consul/raft/
    rm -rf /var/lib/consul/serf/
    rm -rf /var/lib/consul/services/
    rm -rf /var/lib/consul/tmp/
    rm -rf /var/lib/consul/checks/


Configure Consul to be Reachable from Other Wazo
------------------------------------------------

Add a new configuration file `/etc/consul/xivo/interconnection.json` with the
following content where `advertise_addr` is reachable from other Wazo.

.. code-block:: javascript

    {
    "client_addr": "0.0.0.0",
    "bind_addr": "0.0.0.0",
    "advertise_addr": "192.168.1.124"  // The IP address of this Wazo, reachable from outside
    }


Check that the Configuration is Valid
-------------------------------------

.. code-block:: sh

    consul configtest --config-dir /etc/consul/xivo/

No output means that the configuration is valid.


Start Consul
------------

.. code-block:: sh

    systemctl start consul


Start Wazo
----------

.. code-block:: sh

    wazo-service start


Join the Consul Cluster
-----------------------

Join another member of the Consul cluster. Only one join is required as members
will be propagated.

.. code-block:: sh

    consul join -wan 192.168.1.125


Check that Consul Sees other Consul
-----------------------------------

List other members of the cluster with the following command

.. code-block:: sh

    consul members -wan

Check consul logs for problems

.. code-block:: sh

    consul monitor


Check That Everything is Working
================================

There is no further configuration needed, you should now be able to connect your
Wazo Client and search contacts from the People Xlet. When looking up contacts
of another Wazo, you should see their phone status, their user availability, and
agent status dynamically.


Troubleshooting
===============

Chances are that everything won't work the first time, here are some interesting
commands to help you debug the problem.

.. code-block:: sh

    tail -f /var/log/xivo-dird.log
    tail -f /var/log/xivo-ctid-ng.log
    tail -f /var/log/xivo-confd.log
    consul monitor
    consul members -wan
    consul-cli agent services --ssl --ssl-verify=false
    rabbitmqctl eval 'rabbit_federation_status:status().'


What's next?
============

One you get this part working, check out :ref:`phonebook_sharing`.
