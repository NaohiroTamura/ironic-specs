..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Bare metal node N+1 redundancy
==============================

https://bugs.launchpad.net/ironic/+bug/1526234

This proposes high availability/reliability feature to support Bare
Metal Node N+1 Redundancy based on `[RFE] Add volume connection
information into ironic db` [1] and `[RFE] Enhance Power Interface
for Soft Power Off and Inject NMI` [2].


Problem description
===================
When a bare metal node is failed due to hardware problem or is likely
to be failed due to a sign of hardware failure, this function allows
to switch over the node safely to another bare metal node in short
time if the failed node is booted from a share file system such as FC,
FCoE, iSCSI, NFS and etc.


Proposed change
===============
Ironic CLI and API are enhanced to support Ironic CLI new sub-command
`node-switchover`.

`ironic node-switchover <source node> [<destination node>]`

The work flow of the `node switchover` sub-command is as follows:

#. if the `source node` is not capable to boot from a Cinder volume,
   then return an error (HTTP 400).

#. if the optional parameter `destination node` is specified, check if
   the `source node` and `destination node` are identical bare metal
   hardware. If the result is not identical, then return an error
   (HTTP 400)

#. if the optional parameter `destination node` is NOT specified,
   search bare metal nodes in ``enroll state`` or ``manageable
   state`` and pick up an identical bare metal hardware with the
   `source node`. If the result is `not found`, then return an error
   (HTTP 400)

#. shutdown the `source node` by `soft power off` if the node
   supports, otherwise by `power off`. And then change the state of
   `source node` to ``maintenance mode``.

#. move all of node information including ``volume_connectors`` and
   ``volume_targets`` from `source node` to `destination node`.

#. boot the `destination node`.

Alternatives
------------
None


Data model impact
-----------------
None


State Machine Impact
--------------------
None


REST API impact
---------------
Add REST API end point to trigger the switching over of the node
asynchronously.

* PUT /v1/nodes/(node_ident)/switchover::
  {"target_node": uuid_or_name}

  Request parameters
  * node_ident (uuid_or_name) – UUID or logical name of a source node.
  * target_node (uuid_or_name) – Optional. UUID or logical name of a
  destination node.

  Normal response codes 202 Accepted
  Error response codes
  * 400 Bad Request if source node doesn't support boot from Cinder
  volume or suitable destination node is found.
  * 406 Not Acceptable if the API version specified is not supported
  * 409 Conflict if the src node is locked

Client (CLI) impact
-------------------
Add `node-switchover` sub-command to trigger the switching over of the
node asynchronously.

* ironic node-switchover <src id> [--destination <dst id>]::

  * <src id> - Name or UUID of the source node
  * --destination <id> - Optional. Name or UUID of the destination node


RPC API impact
--------------
None


Driver API impact
-----------------
Driver must support `[RFE] Add volume connection information into
ironic db` [1].


Nova driver impact
------------------
Nova driver is enhanced to support `nova evacuate` for Ironic server
instance. Nova driver enhancement is specified in Nova spec [3].


Security impact
---------------
None


Other end user impact
---------------------
End user who doesn't administrator privilege must use `nova evacuate` [3]
 instead of `ironic switchover`.


Scalability impact
------------------
None


Performance Impact
------------------
None


Other deployer impact
---------------------
Deployer has to maintain the number of spare bare metal hardware
according to estimated failure rate.


Developer impact
----------------
None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Naohiro Tamura (naohirot)

Other contributors:
  None


Work Items
----------
* implement Ironic REST API endpoint which is described in the section
  ``REST API impact``.

* implement Ironic CLI sub-command which is described in the section
  ``Client (CLI) impact``.

Dependencies
============
This proposed function depends on the other specs [1] and [2].

Testing
=======
* Unit Tests.

* Third Party CI Tests


Upgrades and Backwards Compatibility
====================================
None.

Documentation Impact
====================
* The deployer doc needs to be updated.
  (CLI and REST API reference manuals are generated automatically
  from source code)


References
==========
[1] `[RFE] Add volume connection information into ironic db <https://bugs.launchpad.net/ironic/+bug/1526231>`_

[2] `[RFE] Enhance Power Interface for Soft Power Off and Inject NMI <https://bugs.launchpad.net/ironic/+bug/1526226>`_

[3] [TBD] nova driver enhancement for `nova evacuate` to support
    Ironic server instance.
