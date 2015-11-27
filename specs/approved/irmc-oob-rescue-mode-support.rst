..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
iRMC OOB rescue mode support
============================

https://blueprints.launchpad.net/ironic/+spec/irmc-oob-rescue-mode-support

Problem description
===================
iRMC Virtual Media driver supports OOB (Out Of Band) boot and deploy [1].

However if rescue mode booted from rescue network using Pluggable
network providers [2], IB (In Band) rescue mode provides customer with
NOT the same network configuration as costumer's production environment.
This would cause another problem when customer tried to investigate
thier problem in the production environment.

Proposed change
===============
In order solve the problem described in the previsouce secton, it is
better to support not only IB rescue mode, but also OOB rescue mode.

Therefor this spec proposes the followoing changes to support OOB
rescue mode for iRMC drivers, `iscsi_irmc` and `agent_irmc`.

1. Implement `IRMCRescue` class by inheriting `base.RescueInterface`.

2. Add `irmc_rescure_iso` into the `driver_info` as
   REQUIRED_PROPERTIES, and `validate` method in `IRMCRescure`
   validates it.

Alternatives
------------
* IB (In Bound) rescue mode can be used via network boot such as PXE.

Data model impact
-----------------
None

State Machine Impact
--------------------
None

 * Note: RESCUE state and its transitions need to be implemented in
   `common.states` module.

REST API impact
---------------
* Enhance `PUT /v1/nodes/(node_ident)/states/provision` so that the
  target parameter can support "inject nmi" as Json data
  `{"target": "inject nmi"}`.

Client (CLI) impact
-------------------
* Enchance `ironic node-set-provision-state` so that
  `<provision-state>` parameter can support `rescure` state.

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

  * `irmc_rescure_iso`: rescure ISO image which is either a file name
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
None.

Testing
=======
* Unit Tests.

* Each vendor plans Third Party CI Tests if implemented.

Upgrades and Backwards Compatibility
====================================
None.

Documentation Impact
====================
* The deployer doc needs to be updated.

References
==========
[1] http://specs.openstack.org/openstack/ironic-specs/specs/liberty-implemented/irmc-virtualmedia-deploy-driver.html

[2] http://specs.openstack.org/openstack/ironic-specs/specs/approved/network-provider.html
