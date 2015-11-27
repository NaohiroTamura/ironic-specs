..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Task control functions for long running tasks
=============================================

https://blueprints.launchpad.net/ironic/+spec/task-control-functions-for-long-running-tasks

This proposal has been separated from the spec `Enhance Power
Interface for Soft Power Off and NMI` [1], and mainly specifies a
mechanism of supporting `abort` operation for `soft reboot`, `soft
power off` and `inject nmi`.

However the basic mechanism which this specification proposes can be
generalized in concurrent programming to support not only `abort` but
also `cancel`, `rollback`, `get status or current progress`,
`suspend`, `resume` and alike which requires inter thread
communication to control a background long running task.

Problem description
===================
The power control operation such as `soft reboot`, `soft power off`
and `inject nmi` would take a long time or could be hanged.

However currently there is no way to abort those tasks.

Proposed change
===============
In order to enable to abort those tasks, this spec proposes to
introduce ABORT target power state as a transition state.

The following changes are required:

1. add the following new states to ironic.common.states::

    ABORT_SOFT_REBOOT = 'abort soft rebooting'
    ABORT_SOFT_POWER_OFF = 'abort soft power off'
    ABORT_INJECT_NMI = 'abort inject nmi'

2. update the default implementation of "get_supported_power_states" method
   in PowerInterface to support only `ABORT_SOFT_REBOOT` and
   `ABORT_SOFT_POWER_OFF` but not `ABORT_INJECT_NMI`::

    def get_supported_power_states(self, task):
        """Get a list of the supported power states.

        :param task: A TaskManager instance containing the node to act on.
        :returns: A list of the supported power states defined
                  in :mod:`ironic.common.states`.
        """
        return [states.POWER_ON, states.POWER_OFF, states.REBOOT,
                states.ABORT_SOFT_REBOOT, states.ABORT_SOFT_POWER_OFF]

        Note: `states.ABORT_INJECT_NMI` is excluded because Ironic
              standard driver, IPMIPower, cannot support it.

3. enhance "set_power_state" method in concrete class so that the
   new states can be accepted as `ABORT_*` parameter::

    def set_power_state(self, task, power_state):
        """Set the power state of the task's node.

        The second parameter "power_state" indicates a new power state
        to be changed to.
        """

    The following table shows power state value of each state
    variables.
    ``new_state`` is a value of the second parameter of
    set_power_state() function.
    ``power_state`` is a value of node property.
    ``target_power_state`` is a value of node property.

    new_state         | power_state  | target_power_state | power_state
                      | (start state)| (current value)    | (end state)
    ------------------+--------------+--------------------+--------------
    ABORT_SOFT_REBOOT | not ERROR    | POWER_OFF_SOFT     | ERROR
    ABORT_SOFT_REBOOT | any state    | not POWER_OFF_SOFT | start state
    ABORT_POWER_OFF   | not ERROR    | POWER_OFF_SOFT     | ERROR
    ABORT_POWER_OFF   | any state    | not POWER_OFF_SOFT | start state
    ABORT_INJECT_NMI  | not ERROR    | INJECT_NMI         | ERROR
    ABORT_INJECT_NMI  | any state    | not INJECT_NMI     | start state

    Note: This IPMIPower reference implementation supports
          SOFT_POWER_OFF and INJECT_NMI, but not states.ABORT_INJECT_NMI.
          However IRMCPower can supports all of three states.ABORT_*.

Alternatives
------------
None.

Data model impact
-----------------
None

State Machine Impact
--------------------
None

REST API impact
---------------
* Add support of ABORT_SOFT_REBOOT, ABORT_SOFT_POWER_OFF and
  ABORT_INJECT_NMI to the target parameter of following API::

   PUT /v1/nodes/(node_ident)/states/power

   The target parameter supports the following Json data respectively.

   {"target": "abort soft rebooting"}
   {"target": "abort soft power off"}
   {"target": "abort inject nmi"}

* Add a new "supported_power_states" member to the return type Node
  and NodeStates, and enhance the following APIs::

   GET /v1/nodes/(node_ident)

   GET /v1/nodes/(node_ident)/states

   Json example of the returned type NodeStates
       {
         "console_enabled": false,
         "last_error": null,
         "power_state": "power on",
         "provision_state": null,
         "provision_updated_at": null,
         "target_power_state": "soft power off",
         "target_provision_state": "active",
         "supported_power_states": [
             "power on",
             "power off",
             "rebooting",
             "soft rebooting",
             "soft power off",
             "inject nmi",
             "abort soft rebooting",
             "abort soft power off",
             "abort inject nmi"
          ]
        }

   Consequently Ironic CLI "ironic node-show" and "ironic node-show-states"
   return "supported_power_states" member in the table format.

   example of "ironic node-show-states"

   +------------------------+----------------------------------------+
   | Property               | Value                                  |
   +------------------------+----------------------------------------+
   | target_power_state     | soft power off                         |
   | target_provision_state | None                                   |
   | last_error             | None                                   |
   | console_enabled        | False                                  |
   | provision_updated_at   | 2015-08-01T00:00:00+00:00              |
   | power_state            | power on                               |
   | provision_state        | active                                 |
   | supported_power_states | ["power on", "power off", "rebooting", |
   |                        |   "soft rebooting", "soft power off",  |
   |                        |   "inject nmi", "abort soft rebooting",|
   |                        |   "abort soft power off",              |
   |                        |   "abort inject nmi"]                  |
   +------------------------+----------------------------------------+

Client (CLI) impact
-------------------
* Enhance "ironic node-set-power-state" so that <power-state>
  parameter can accept 'abort_soft_reboot', 'abort_soft_off' and
  'abort_inject_nmi'. This CLI is async. In order to get the latest
  status, call "ironic node-show-states" and check the returned
  value.::

   usage: ironic node-set-power-state <node> <power-state>

   Power a node on/off/reboot, power graceful off/reboot,
   inject NMI to a node.

   Positional arguments

   <node>

       Name or UUID of the node.

   <power-state>

       'on', 'off', 'reboot', 'soft_reboot', 'soft_off', 'inject_nmi',
       'abort_soft_reboot', 'abort_soft_off', 'abort_inject_nmi',

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
None.

Developer impact
----------------
* Each driver developer needs to follow this interface to implement
  this proposed feature.

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
* Enhance PowerInterface class to support abort sort reboot, abort
  soft power off and abort inject nmi as described "Proposed change".

* Enhance Ironic API as described in "REST API impact".

* Enhance Ironic CLI as described in "Client (CLI) impact".

* Implement the enhanced PowerInterface class into the concrete class
  IPMIPower.
  Implementing vendor's power concrete class is up to each vendor.

Dependencies
============
This spec is solely depends on the spec `Enhance Power Interface for
Soft Power Off and Inject NMI` [1].

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
[1] https://blueprints.launchpad.net/ironic/+spec/enhance-power-interface-for-soft-reboot-and-nmi
