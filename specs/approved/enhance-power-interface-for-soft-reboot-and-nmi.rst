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
following inconveniences.

1. Customer cannot safely shutdown or soft power off their instance
   without logging on, by Ironic CLI or Ironic REST API in case of
   tenant admin, by Nova CLI or NOVA REST API in case of tenant user.

2. Customer cannot take NMI dump to investigate OS related problem by
   themselves, by Ironic CLI or Ironic REST API in case of tenant
   admin, by Nova CLI or NOVA REST API in case of tenant user.

From deployer's point of view, that is cloud provider, the lack of the
two capabilities leads the following inconveniences.

1. Cloud provider support staff cannot shutdown customer's instance
   safely without logging on for hardware maintenance reason or etc.

2. Cloud provider support staff cannot ask customer to take NMI dump
   as one of investigation materials.


Proposed change
===============
In order to solve the problems described in the previous section,
this spec proposes to enhance the power states and the PowerInterface
base class so that each driver can implement to initiate soft reboot,
soft power off and inject NMI.

And this enhancement enables the soft reboot, soft power off and
inject NMI through Ironic CLI and REST API for tenant admin and cloud
provider. Also this enhancement enables them through Nova CLI and REST
API for tenant user when Nova's blue print [3] is implemented.

This spec also proposes to implement the enhanced PowerInterface base
class into the IPMIPower concrete class as a reference implementation.

1. add the following new states to ironic.common.states::

    REBOOT_SOFT = 'soft reboot'
    POWER_OFF_SOFT = 'soft power off'
    INJECT_NMI = 'inject nmi'

2. add "get_supported_power_states" method and default implementation
   in PowerInterface::

    def get_supported_power_states(self, task):
        """Get a list of the supported power states.

        :param task: A TaskManager instance containing the node to act on.
        :returns: A list with the supported power states defined
                  in :mod:`ironic.common.states`.
        """
        return [states.POWER_ON, states.POWER_OFF, states.REBOOT]

        Note: WakeOnLanPower driver supports only states.POWER_ON.

3. enhance "set_power_state" method in IPMIPower class so that the
   new states can be accepted as "power_state" parameter::

    def set_power_state(self, task, power_state):
        """Set the power state of the task's node.

        The second parameter "power_state" indicates a new power state
        to be changed to.
        """

    This IPMIPower reference implementation supports SOFT_REBOOT,
    SOFT_POWER_OFF and INJECT_NMI.

    SOFT_REBOOT is implemented as POWER CYCLE such that REBOOT is
    implemented. This implementation enables generic BMC detect the
    reboot completion as the power state change from ON -> OFF -> ON.

    The following table shows power state value of each state
    variables.
    ``new_state`` is a value of the second parameter of
    set_power_state() function.
    ``power_state`` is a value of node property.
    ``target_power_state`` is a value of node property.

    new_state         | power_state  | target_power_state | power_state
                      | (start state)| (assigned value)   | (end state)
    ------------------+--------------+--------------------+--------------
    SOFT_REBOOT       | POWER_ON     | SOFT_POWER_OFF     | POWER_OFF*1
                      | POWER_OFF*1  | POWER_ON           | POWER_ON
    SOFT_REBOOT       | POWER_OFF    | POWER_ON           | POWER_ON
    SOFT_POWER_OFF    | POWER_ON     | SOFT_POWER_OFF     | POWER_OFF
    SOFT_POWER_OFF    | POWER_OFF    | NONE               | POWER_OFF
    INJECT_NMI        | POWER_ON     | INJECT_NMI         | POWER_ON
    INJECT_NMI        | POWER_OFF    | NONE               | POWER_OFF

    *1) intermediate state of POWER CYCLE
        SOFT_REBOOT is implemented as power cycle such as REBOOT.

    In case that timeout occurred during soft reboot, soft power off
    or inject nmi, the end state becomes Error.

    new_state         | power_state  | target_power_state | power_state
                      | (start state)| (assigned value)   | (end state)
    ------------------+--------------+--------------------+--------------
    SOFT_REBOOT       | POWER_ON     | SOFT_POWER_OFF     | Error
    SOFT_POWER_OFF    | POWER_ON     | SOFT_POWER_OFF     | Error
    INJECT_NMI        | POWER_ON     | INJECT_NMI         | Error

    The timeout can be configured in the Ironic configuration file,
    typically /etc/ironic/ironic.conf, as follows.

    [conductor]
    # timeout (in seconds) of soft power off operation (integer value)
    soft_power_off_timeout = 600

    # timeout (in seconds) of inject nmi operation (integer value)
    inject_nmi_timeout = 600


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
                states.SOFT_REBOOT, states.SOFT_POWER_OFF_SOFT,
                states.INJECT_NMI]
        if node's properties/capabilities='{"soft_power": "true"}' and
        properties/capabilities='{"inject_nmi": "true"}'.
        otherwise exclude states.SOFT_REBOOT, states.SOFT_POWER_OFF_SOFT,
        and/or states.INJECT_NMI from the returned value.

Alternatives
------------
* Both the soft power off and diagnostic interrupt (NMI [1]) could be
  implemented by vendor passthru. However the proposed change is
  better than the vendor passthru, because users of Ironic API or
  Ironic CLI can write script or program uniformly.


Data model impact
-----------------
None

State Machine Impact
--------------------
None

REST API impact
---------------
* Add support of SOFT_REBOOT, SOFT_POWER_OFF and INJECT_NMI to the
  target parameter of following API::

   PUT /v1/nodes/(node_ident)/states/power

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
         "target_power_state": "power off soft",
         "target_provision_state": "active",
         "supported_power_states": [
             "power on",
             "power off",
             "reboot",
             "soft reboot",
             "soft power off",
             "inject nmi"
          ]
        }

   Consequently Ironic CLI "ironic node-show" and "ironic node-show-states" 
   return "supported_power_states" member in the table format.

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
   |                        |   "reboot soft", "soft power off",  |
   |                        |   "inject nmi"]                     |
   +------------------------+-------------------------------------+

Client (CLI) impact
-------------------
* Enhance "ironic node-set-power-state" so that <power-state>
  parameter can accept 'soft_reboot', 'soft_off' and 'inject_nmi' [5].
  This CLI is async. In order to get the latest status,
  call "ironic node-show-states" and check the returned value.::

   usage: ironic node-set-power-state <node> <power-state>

   Power a node on/off/reboot, power graceful off/reboot,
   inject NMI to a node.

   Positional arguments

   <node>

       Name or UUID of the node.

   <power-state>

       'on', 'off', 'reboot', 'soft_reboot', 'soft_off', inject_nmi' [5] 

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
specified in the bug report [6] as well as nova blueprint [10] and
spec [11].

Security impact
---------------
None

Other end user impact
---------------------
* End user who has admin privilege such as tenant admin has to make
  sure the following:

 * set properties/capabilities='{"soft_power": "true"}' to a node if 
   it is capable of soft reboot and soft power off.
   If the key "soft_power" doesn't exist, or a value of the key
   "soft_power" is set to other than "true", it is not capable of soft
   reboot and soft power off.

 * set properties/capabilities='{"inject_nmi": "true"}' to a node if
   it is capable of inject NMI.
   If the key "inject_nmi" doesn't exist, or a value of the key
   "inject_nmi" is set to other than "true", it is not capable of
   inject NMI.

 * deploy or set up ACPI [7] controllable instance to the node. How to
   make the instance ACPI [7] controllable is described in
   "Dependencies" section.

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

* change the default timeout value (sec) in the Ironic configuration
  file, typically /etc/ironic/ironic.conf if necessary::

  [conductor]
  # timeout (in seconds) of soft power off operation (integer value)
  soft_power_off_timeout = 600

  # timeout (in seconds) of inject nmi operation (integer value)
  inject_nmi_timeout = 600

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
* Enhance PowerInterface class to support soft power off and
  inject nmi [1] as described "Proposed change".

* Enhance Ironic API as described in "REST API impact".

* Enhance Ironic CLI as described in "Client (CLI) impact".

* Implement the enhanced PowerInterface class into the concrete class
  IPMIPower.
  Implementing vendor's power concrete class is up to each vendor.

* Coordinate the work with Nova NMI support "Inject NMI to an
  instance" [3] if necessary.

Dependencies
============
* IPMIPower driver depends on ipmitool [2].

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

* Note
  There's a backwards compatibility issue with the behavior of "nova
  reboot --soft", but that is discussed in the Nova blueprint [10] and
  spec [11] for the Ironic driver changes.

Documentation Impact
====================
* The deployer doc needs to be updated.
  (CLI and REST API reference manuals are generated automatically
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

[10] https://blueprints.launchpad.net/nova/+spec/soft-reboot-poweroff

[11] https://review.openstack.org/#/c/229282/
