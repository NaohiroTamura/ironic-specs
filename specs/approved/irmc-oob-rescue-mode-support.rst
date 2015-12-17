..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
iRMC OOB rescue mode support
============================

https://bugs.launchpad.net/ironic/+bug/1526470

The proposal presents the work required to add support for OOB (Out Of
Band) rescue feature for FUJITSU PRIMERGY iRMC, integrated Remote
Management Controller, Drivers in Ironic.

Problem description
===================
iRMC Virtual Media driver supports OOB boot and deploy [1].

However if rescue mode is booted from rescue network using Pluggable
network providers [2], IB (In Band) rescue mode provides customer with
NOT the same network configuration as costumer's production environment.
This would cause another problem when customer tried to investigate
their problem in the production environment.

Proposed change
===============
In order solve the problem described in the previous section, it is
better to support not only IB rescue mode [3], but also OOB rescue mode.

Therefor this spec proposes the following changes to support OOB
rescue mode for iRMC drivers, `iscsi_irmc` and `agent_irmc`.

1. Implement `IRMCRescue` class by inheriting `base.RescueInterface`.

2. Add `irmc_rescue_iso` into the `driver_info` as
   REQUIRED_PROPERTIES, and `validate` method in `IRMCRescue`
   validates it.

Alternatives
------------
* IB (In Bound) rescue mode [3] can be used via network boot such as PXE.

Data model impact
-----------------
None

State Machine Impact
--------------------
None

REST API impact
---------------
None

Client (CLI) impact
-------------------
None

RPC API impact
--------------
None.

Driver API impact
-----------------
None.

Nova driver impact
------------------
None.

Security impact
---------------
None.

Other end user impact
---------------------
None.

Scalability impact
------------------
None.

Performance Impact
------------------
None.

Other deployer impact
---------------------
* The following driver_info field is required.

  * `irmc_rescue_iso`: rescue ISO image which is either a file name
    relative to remote_image_share_root, Glance UUID, Glance URL or
    Image Service URL.

Developer impact
----------------
None.

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
* Implement IRMCRescue class by inheriting base.RescueInterface as
  described "Proposed change".

Dependencies
============
This spec depends on the spec `Implement Rescue Mode` [3].

Testing
=======
* Unit Tests.

* FUJITSU provides Third Party CI Tests.

Upgrades and Backwards Compatibility
====================================
None.

Documentation Impact
====================
* The deployer doc needs to be updated.

References
==========
[1] https://specs.openstack.org/openstack/ironic-specs/specs/liberty-implemented/irmc-virtualmedia-deploy-driver.html

[2] https://specs.openstack.org/openstack/ironic-specs/specs/approved/network-provider.html

[3] `Implement rescue mode <https://review.openstack.org/#/c/171878/>`_
