..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Enhance Power Interface for Soft Power Off and NMI
==================================================

https://blueprints.launchpad.net/ironic/+spec/enhance-power-interface-for-soft-reboot-and-nmi

The proposal presents the work required to enhance the power
interface to support soft reboot, soft power off and diagnostic
interrupt (NMI [1]).


Problem description
===================
There exists a problem in the current PowerInterface base class which
doesn't provide with soft power off and diagnostic interrupt (NMI [1])
capabilities even if ipmitool [2] and most of BMCs support.

Here is a part of ipmitool man page in which describes soft power off and
diagnostic interrupt (NMI [1]).

$ man ipimtool::
 ...
 power

        Performs a chassis control command to view and change the
        power state.

        ...

        diag

               Pulse a diagnostic interrupt (NMI) directly to the
               processor(s).

        soft

               Initiate a soft-shutdown of OS via ACPI. This can be
               done in a number of ways, commonly by simulating an
               overtemperature or by simulating a power button press.
               It is necessary for there to be Operating System
               support for ACPI and some sort of daemon watching for
               events for this soft power to work.

From customer's point of view, both tenant admin and tenant user, the
lack of the soft power off and diagnostic interrupt (NMI [1]) lead the
following inconvenience.

1. Customer cannot safely shutdown or soft power off their instance
   without logging on by Ironic CLI or Ironic REST API in case of
   tenant admin, by Nova CLI or NOVA REST API in case of tenant user.

2. Customer cannot take NMI dump to investigate OS related problem by
   themselves by Ironic CLI or Ironic REST API in case of tenant
   admin, by Nova CLI or NOVA REST API in case of tenant user.

From deployer's point of view, that is cloud provider, the lack of the
two capabilities leads the following inconvenience.

1. Cloud provider support staff cannot shutdown customer's instance
   safely without logging on for hardware maintenance reason or etc.

2. Cloud provider support staff cannot ask customer to take NMI dump
   as one of investigation materials.


Proposed change
===============
In order to solve the problems described in the previous section,
this spec proposes to enhance the power states and the PowerInterface
base class so that each driver can implement to initiate/cancel soft
reboot, soft power off and inject NMI.

This enhancement enables initiate/cancel the soft reboot, soft power
off and inject NMI through Ironic CLI and REST API for tenant admin and
cloud provider. Also this enhancement enables them through Nova CLI
and REST API for tenant user when Nova's blue print [3] is implemented.

This spec also proposes to implement the enhanced PowerInterface base
class into the IPMIPower concrete class as a reference implementation.

1. add the following new states to ironic.common.states, and CSP
   channel to TaskManager::

    REBOOT_SOFT = 'rebooting soft'
    POWER_OFF_SOFT = 'power off soft'
    INJECT_NMI = 'inject nmi'
    CANCEL_REBOOT_SOFT = 'cancel rebooting soft'
    CANCEL_POWER_OFF_SOFT = 'cancel power off soft'
    CANCEL_INJECT_NMI = 'cancel inject nmi'

    In order to support CANCEL_* task, CSP channel [4] is added
    since it is easier and simpler than other communication mechanisms
    such as semaphore, mutex, pipe, event, signal and etc.

2. add "get_supported_power_states" method and default implementation
   in PowerInterface::

    def get_supported_power_states(self, task):
        """Get a list of the supported power states.

        :param task: A TaskManager instance containing the node to act on.
        :returns: A list with the supported power states defined
                  in :mod:`ironic.common.states`.
        """
        return [states.POWER_ON, states.POWER_OFF, states.REBOOT,]

3. enhance "set_power_state" method in IPMIPower class so that the
   new states can be accepted as "power_state" parameter::

    def set_power_state(self, task, power_state):
        """Set the power state of the task's node.

        The second parameter "power_state" indicates a new power state
        to be changed to.

        :param task: A TaskManager instance containing the node to act on.
        :param power_state: One of power state in :mod:`ironic.common.states`.
        :raises: MissingParameterValue if a required parameter is missing.
        """

    This IPMIPower reference implementation supports REBOOT_SOFT,
    POWER_OFF_SOFT, CANCEL_REBOOT_SOFT, and CANCEL_POWER_OFF_SOFT.

    REBOOT_SOFT is implemented as POWER CYCLE such that REBOOT is
    implemented. This implementation enables generic BMC detect the
    reboot completion as the power state change from ON -> OFF -> ON.

    However this reference implementation supports INJECT_NMI, but not
    CANCEL_INJECT_NMI.

    The reason is that INJECT_NMI cannot be implemented as POWER CYCLE
    to preserve memory foot print, so power state keeps staying in ON
    during injecting NMI. Therefor CANCEL_INJECT_NMI is not supported
    by IPMIPower.

    The following table shows power state value of each state
    variables.
    ``new_state`` is a value of the second parameter of
    set_power_state() function.
    ``power_state`` is a value of node property.
    ``target_power_state`` is a value of node property.

    new_state         | power_state  | target_power_state | power_state
                      | (start state)| (assigned value)   | (end state)
    ------------------+--------------+--------------------+--------------
    SOFT_REBOOT       | POWER_ON     | POWER_OFF_SOFT     | POWER_OFF*1
                      | POWER_OFF*1  | POWER_ON           | POWER_ON
    SOFT_REBOOT       | POWER_OFF    | POWER_ON           | POWER_ON
    POWER_OFF_SOFT    | POWER_ON     | POWER_OFF_SOFT     | POWER_OFF
    POWER_OFF_SOFT    | POWER_OFF    | NONE               | POWER_OFF
    INJECT_NMI        | POWER_ON     | INJECT_NMI         | POWER_ON
    INJECT_NMI        | POWER_OFF    | NONE               | ERROR

    *1) intermediate state of POWER CYCLE
        SOFT_REBOOT is implemented as power cycle such as REBOOT.

    new_state         | power_state  | target_power_state | power_state
                      | (start state)| (current value)    | (end state)
    ------------------+--------------+--------------------+--------------
    CANCEL_SOFT_REBOOT| not ERROR    | POWER_OFF_SOFT     | start state
    CANCEL_SOFT_REBOOT| any state    | not POWER_OFF_SOFT | start state
    CANCEL_POWER_OFF  | not ERROR    | POWER_OFF_SOFT     | POWER_ON
    CANCEL_POWER_OFF  | any state    | not POWER_OFF_SOFT | start state
    CANCEL_INJECT_NMI | not ERROR    | INJECT_NMI         | start state
    CANCEL_INJECT_NMI | any state    | not INJECT_NMI     | start state

4. add "get_supported_power_states" method and implementation in
   IPMIPower::

    def get_supported_power_states(self, task):
        """Get a list of the supported power states.

        :param task: A TaskManager instance containing the node to act on.
           currently not used.
        :returns: A list with the supported power states defined
                  in :mod:`ironic.common.states`.
        """
        return [states.POWER_ON, states.POWER_OFF, states.REBOOT,
                states.REBOOT_SOFT, states.POWER_OFF_SOFT,
                states.CANCEL_REBOOT_SOFT, states.CANCEL_POWER_OFF_SOFT.]


Alternatives
------------
* Both the soft power off and diagnostic interrupt (NMI [1]) could be
  implemented by vendor passthru. However the proposed change is
  better than the vendor passthru, because users of Ironic API or
  Ironic CLI can write script or program uniformly.


Data model impact
-----------------
* The length of node power state columns needs to be extended from the
  current 'String(15)' to 'String(255)' as followings::

   power_state = Column(String(255), nullable=True)
   target_power_state = Column(String(255), nullable=True)
   provision_state = Column(String(255), nullable=True)


State Machine Impact
--------------------
None

REST API impact
---------------
* Add support for REBOOT_SOFT, POWER_OFF_SOFT, INJECT_NMI,
  CANCEL_REBOOT_SOFT, CANCEL_POWER_OFF_SOFT, and CANCEL_INJECT_NMI to
  the following API. This API is async. In order to get the latest status,
  call "GET /v1/nodes/(node_ident)/states" and check the returned
  value of NodeStates.

  PUT /v1/nodes/(node_ident)/states/power::

   Set the power state of the node.
     Normal http response code:
       * 202: Accepted
     Parameters:
       * node_ident (uuid_or_name) – the UUID or logical name of a node.
       * target (unicode) – The desired power state of the node.
     Raises:
       ClientSideError (HTTP 409) if a power operation is already in progress.
     Raises:
       InvalidStateRequested (HTTP 400) if the requested target state is not
       valid/supported or if the node is in a state which should not
       be interrupted, such as CLEANING, DELETING, ZAPPING.

* Add a new "supported_power_states" member to type Node and
  NodeStates, and enhance the following API so that returned table
  contains "supported_power_states" member.

  GET /v1/nodes/(node_ident)::

   Retrieve information about the given node.
     Normal http response code:
       * 200: OK
     Parameters:
       * node_ident (uuid_or_name) – UUID or logical name of a node.
       * fields (list) – Optional, a list with a specified set of
         fields of the resource to be returned.
     Return type:
       Node

  GET /v1/nodes/(node_ident)/states::

   List the states of the node.
     Normal http response code:
       * 200: OK
     Parameters:
       * node_ident (uuid_or_name) – the UUID or logical_name of a node.
    Return type:
       NodeStates

       Json example
       {
         "console_enabled": false,
         "last_error": null,
         "power_state": "power on",
         "provision_state": null,
         "provision_updated_at": null,
         "target_power_state": "power off soft",
         "target_provision_state": "active",
         "supported_power_states": [
             "power on",
             "power off",
             "reboot",
             "reboot soft",
             "power off soft",
             "cancel reboot soft",
             "cancel power off soft",
             "cancel inject nmi"
          ]
        }


Client (CLI) impact
-------------------
* Enhance "ironic node-set-power-state" so that <power-state>
  parameter can accept 'reboot_soft', 'soft_off', 'inject_nmi' [5],
  'cancel_reboot_soft', 'cancel_soft_off', and 'cancel_inject_nmi'.
  This CLI is async. In order to get the latest status,
  call "ironic node-show-states" and check the returned value.::

   usage: ironic node-set-power-state <node> <power-state>

   Power a node on/off/reboot, power graceful off/reboot,
   inject NMI to a node, cancel graceful off/reboot,
   or cancel inject NMI.

   Positional arguments

   <node>

       Name or UUID of the node.

   <power-state>

       'on', 'off', 'reboot', 'soft_reboot', 'soft_off',
       'inject_nmi' [5], 'cancel_soft_reboot', 'cancel_soft_off' or
       'cancel_inject_nmi'.


* Enhance "ironic node-show" and "ironic node-show-states" so as to
  return "supported_power_states" member in the table format.::

   example of "ironic node-show-states"

   +------------------------+-------------------------------------+
   | Property               | Value                               |
   +------------------------+-------------------------------------+
   | target_power_state     | power off soft                      |
   | target_provision_state | None                                |
   | last_error             | None                                |
   | console_enabled        | False                               |
   | provision_updated_at   | 2015-08-01T00:00:00+00:00           |
   | power_state            | power on                            |
   | provision_state        | active                              |
   | supported_power_states | ["power on", "power off", "reboot", |
   |                        |   "reboot soft", "power off soft",  |
   |                        |    "cancel reboot soft",            |
   |                        |   "cancel power off soft",          |
   |                        |    "cancel inject nmi"]             |
   +------------------------+-------------------------------------+


RPC API impact
--------------
None

Driver API impact
-----------------
PowerInterface base is enhanced by adding a new method,
get_supported_power_states() which returns a list of supported power
states.

Nova driver impact
------------------
The default behavior of "nova reboot" command to a virtual machine
instance such as KVM is soft reboot.
And "nova reboot" command has a option '--hard' to indicate hard reboot.

However the default behavior of "nova reboot" to an Ironic instance
is hard reboot, and --hard option is meaningless to the Ironic instance.

Therefor Ironic Nova driver needs to be update to unify the behavior
between virtual machine instance and bare-metal instance.

This problem is reported as a bug [6]. How to fix this problem will be
specified in the bug report [6].

Security impact
---------------
None

Other end user impact
---------------------
* End user who has admin privilege such as tenant admin has to make
  sure the following:

 * set properties/capabilities='{"power_soft": "true"}' to a node if
   it is capable of soft reboot and soft power off.
   If the key "power_soft" doesn't exist, or a value of the key
   "power_soft" is set to other than "true", it is not capable of soft
   reboot and soft power off.

 * set properties/capabilities='{"inject_nmi": "true"}' to a node if
   it is capable of inject NMI.
   If the key "inject_nmi" doesn't exist, or a value of the key
   "inject_nmi" is set to other than "ture", it is not capable of
   inject NMI.

 * deploy or set up ACPI [7] controllable instance to the node. How to
   make the instance ACPI [7] controllable is described in
   "Dependencies" section.

* End user who doesn't have admin privilege such as tenant user
  requires the following to use this features:

 * In case of Ironic with Nova, use Nova CLI or Nova API which
   implemented Nova's blue print [3], or ask end user who has admin
   privilege such as tenant admin to enable this features.

 * In case of Ironic standalone, ask end user who has admin privilege
   such as tenant admin to enable this features.
   (Note: However how to control admin privilege is solely up to each
   tenant in case of standalone mode, tenant could assign admin
   privilege to all if this procedure is inconvenient.)


Scalability impact
------------------
None

Performance Impact
------------------
None

Other deployer impact
---------------------
* Deployer, cloud provider, needs to set up ACPI [7] capable bare
  metal servers in cloud environment.

* change the default timeout value (sec) if necessary::

   CONF.conductor.power_off_soft_retry_timeout = 600

   CONF.conductor.power_inject_nmi_retry_timeout = 600

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
* Re-factor TaskManager and implement cancel-able task by adding CSP
  channel [4] so that cancel task such as CANCEL_REBOOT_SOFT,
  CANCEL_POWER_OFF_SOFT, and CANCEL_INJECT_NMI can cancel long running
  task such as REBOOT_SOFT, POWER_OFF_SOFT and INJECT_NMI respectively.

* Enhance PowerInterface class to support soft power off and
  diagnostic interrupt (NMI [1]) as described "Proposed change".

* Enhance Ironic API as described in "REST API impact".

* Enhance Ironic CLI as described in "Client (CLI) impact".

* Create a DB migration script to extend power state column length as
  described in "Data model impact"

* Implement the enhanced PowerInterface class into the concrete class
  IPMIPower.
  Implementing vendor's power concrete class is up to each vendor.

* Coordinate the work with Nova NMI support "Inject NMI to an
  instance" [3] if necessary.

Dependencies
============
* Ironic conductor depends on ipmitool [2].

* Ironic managed node depends on ACPI [7]. In case of Linux system,
  acpid [8] has to be installed. In case of Windows system, local
  security policy has to be set as described in "Shutdown: Allow
  system to be shut down without having to log on" [9].

Testing
=======
* Unit Tests.

* Each vendor plans Third Party CI Tests if implemented.

Upgrades and Backwards Compatibility
====================================
None (Forwards Compatibility is out of scope)

Documentation Impact
====================
* None (CLI and REST API reference manuals are generated automatically
  from source code)

References
==========
[1] http://en.wikipedia.org/wiki/Non-maskable_interrupt

[2] http://linux.die.net/man/1/ipmitool

[3] https://review.openstack.org/#/c/187176/

[4] https://en.wikipedia.org/wiki/Communicating_sequential_processes

[5] http://linux.die.net/man/1/virsh

[6] https://bugs.launchpad.net/nova/+bug/1485416

[7] http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface

[8] http://linux.die.net/man/8/acpid

[9] https://technet.microsoft.com/en-us/library/jj852274%28v=ws.10%29.aspx
