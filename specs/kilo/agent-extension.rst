..
 This work is licensed under a Creative Commons Attribution 4.0 International
 License.

 http://creativecommons.org/licenses/by/4.0/

===================
Agent Extension API
===================

Neutron API allows users to manage the agents in Neutron using its 'agent'
extension.  This document explains the MidoNet implmentation of this
extension so that it can be used to provide the health status of the
Midolman agents, as well as all the useful information the Midolman agent
was able to gather on that host.


Problem Description
===================

Since the Neutron API does not provide information about the physical hosts
in which the MidoNet agents are running, we have very little visibility
into the physical layer of the cloud deployment.  The use cases for this
API are:

* Operator wants to see the health of the MidoNet agents on the hosts using
  REST API.  Without it, debugging is cumbersome and time-consuming.
* Operator wants to see the basic networking information on the hosts, such
  as the network interfaces, mtu, status, MAC and IP addresses, for debugging.

Note that Nova API would be another good place to display information about the
compute hosts, but to avoid maintaining a custom Nova code, it has been
decided to implement this feature as a Neutron API extension.

Even though Neutron provides an 'agents' extension API, the MidoNet plugin
does not implement it for its agents.


Proposed Change
===============

The 'agent' extension API has several fields that are useful to provide the
health state of the Midolman agents and the information of the hosts.

 
The 'agent' extension API has 'alive' field to indicate the health of the
Midolman agents.  
REST API is implemented as a Neutron vendor extension API to provide
the following read-only information:

* MidoNet agent health on the hosts
* Metadata of the midolman agents
* Network interfaces, mtu, status, MAC addresses and IP addresses assigned.

MidoNet cluster provides the host state through its read-only API, and
Neutron Host REST API fetches this information on each request.


Data Model Impact
-----------------

Since the host information is fetched from the cluster on each request, the
cluster holds all the host-related data, and nothing needs to be stored in
the Neutron DB.


REST API Impact
---------------

**Host**

+----------+-----------+-------+---------+-----------+------------------------+
|Attribute |Type       |Access |Default  |Validation/|Description             |
|Name      |           |       |Value    |Conversion |                        |
+==========+===========+=======+=========+===========+========================+
|id        |string     |RO,    |generated|N/A        |identity                |
|          |(UUID)     |admin  |         |           |                        |
+----------+-----------+-------+---------+-----------+------------------------+
|name      |string     |RO,    |''       |N/A        |human-readable name     |
|          |           |admin  |         |           |                        |
+----------+-----------+-------+---------+-----------+------------------------+
|alive     |bool       |RO,    |False    |N/A        |MidoNet agent health    |
|          |           |admin  |         |           |                        |
+----------+-----------+-------+---------+-----------+------------------------+
|interfaces|array      |RO,    |[]       |N/A        |network interfaces on   |
|          |(Interface)|admin  |         |           |the host                |
+----------+-----------+-------+---------+-----------+------------------------+


**Interface**

+----------+--------+-------+---------+------------+--------------------------+
|Attribute |Type    |Access |Default  |Validation/ |Description               |
|Name      |        |       |Value    |Conversion  |                          |
+==========+========+=======+=========+============+==========================+
|name      |string  |RO,    |''       |N/A         |interface name            |
|          |        |admin  |         |            |                          |
+----------+--------+-------+---------+------------+--------------------------+
|type      |string  |RO,    |'UNKNOWN'|'PHYSICAL', |interface type            |
|          |        |admin  |         |'VIRTUAL',  |                          |
|          |        |       |         |'TUNNEL',   |                          |
|          |        |       |         |'UNKNOWN'   |                          |
+----------+--------+-------+---------+------------+--------------------------+
|endpoint  |string  |RO,    |'UNKNOWN'|'DATAPATH', |detailed type             |
|          |        |admin  |         |'PHYSICAL', |                          |
|          |        |       |         |'LOCALHOST',|                          |
|          |        |       |         |'TUNTAP',   |                          |
|          |        |       |         |'UNKNOWN',  |                          |
+----------+--------+-------+---------+------------+--------------------------+
|mac       |string  |RO,    |''       |N/A         |MAC address               |
|          |        |admin  |         |            |                          |
+----------+--------+-------+---------+------------+--------------------------+
|mtu       |int     |RO,    |0        |N/A         |MTU size                  |
|          |        |admin  |         |            |                          |
+----------+--------+-------+---------+------------+--------------------------+
|status    |int     |RO,    |0        |0 (unknown) |interface status          |
|          |        |admin  |         |1 (up),     |                          |
|          |        |       |         |2 (carrier) |                          |
+----------+--------+-------+---------+------------+--------------------------+
|addresses |array   |RO,    |[]       |N/A         |IP addresses, including   |
|          |(string)|admin  |         |            |IPv4 and IPv6             |
+----------+--------+-------+---------+------------+--------------------------+

mac, mtu and addresses have default values as specified above, but they should
almost never have these values.  If they have these default values set, that
means that the data could not be retrieved and are unknown at the time of
the request.

addresses array contains a mix of IPv4 and IPv6 addresses.  It is left up to
the client ot differentiate them.


Security Impact
---------------

Physical host information becomes visible only to admins.


Other End User Impact
---------------------

The Host API will be available from neutronclient.

The following command is available to view the hosts

::

    neutron host-list [-h] [-P SIZE] [--sort-key FIELD]
                      [--sort-dir {asc, desc}]

-h, --help::
    show the help message

-P SIZE, --page-size SIZE::
    Specify retrieve unit of each request

--sort-key FIELD::
    Sorts the list by the specified fields

--sort-dir {asc,desc}::
    Sorts the list in the specified direction


Dependencies
============

The cluster refactoring that will expose the host information through its API
is required [1].


Testing
=======

Unit tests will be provided.


Documentation Impact
====================

API should be included in the REST API specification.


References
==========

[1] https://review.gerrithub.io/#/c/13816/

