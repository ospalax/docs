.. _mon:
.. _imudppushg:

====================
Monitoring
====================

This section provides an overview of the OpenNebula monitoring subsystem. The monitoring subsystem gathers information relative to the Hosts and the Virtual Machines, such as the Host status, basic performance indicators, as well as Virtual Machine status and capacity consumption. This information is collected by executing a set of static probes provided by OpenNebula. The output of these probes is sent to OpenNebula using a push mechanism.

Overview
==================

Each host periodically sends monitoring data via network to the Frontend which collects it and processes it in a dedicated module. This distributed monitoring system resembles the architecture of dedicated monitoring systems, using a lightweight communication protocol, and a push model.

OpenNebula starts a ``onemonitord`` daemon running in the Front-end that listens for network connections on port 4124. In the first monitoring cycle the OpenNebula connects to the host using ``ssh`` and starts a daemon that will execute the probe scripts and sends the collected data to the onemonitord daemon in the Frontend every specific amount of seconds (configurable in the monitord.conf file). This way the monitoring subsystem doesn't need to make new ssh connections to receive data.

|image0|

If the agent stops in a specific Host, OpenNebula will detect that no monitorization data is received from that hosts and will restart the probe with SSH.

Requirements
============

* The firewall of the Frontend (if enabled) must allow TCP and UDP packages incoming from the hosts on port 4124.

OpenNebula Configuration
========================

Enabling the Drivers
--------------------

To enable this monitoring system ``/etc/one/oned.conf`` must be configured with the following snippets:

.. code::

    #*******************************************************************************
    # Monitor Daemon
    #*******************************************************************************
    #  Monitor daemon, specific monitor drivers can be added in the monitord
    #  configuration file (monitord.conf)
    #
    #   name       : OpenNebula name for the daemon
    #
    #   executable : path of the information driver executable, can be an
    #                absolute path or relative to $ONE_LOCATION/lib/mads (or
    #                /usr/lib/one/mads/ if OpenNebula was installed in /)
    #
    #   arguments  : for the monitor daemon
    #     -c : configuration file (monitord.conf by default)
    #
    #   threads    : number of threads used to process messages from monitor daemon
    #*******************************************************************************
    IM_MAD = [
        NAME       = "monitord",
        EXECUTABLE = "onemonitord",
        ARGUMENTS  = "-c monitord.conf",
        THREADS    = 8 ]

Monitor Daemon Configuration
----------------------------

The configuration of the Monitor Daemon is in monitord.conf file. Important configuration
values are:

.. code::

    #*******************************************************************************
    # UDP Listener
    #*******************************************************************************
    # Reads messages from monitor agent
    #
    #   addresss       : network address to bind the UDP listener to.
    #   monitor_address: agents will send updates to this monitor address
    #
    #   port    : listening port
    #   threads : number of processing threads
    #   pubkey  : Absolute path to public key (agents). Empty for no encryption.
    #   prikey  : Absolute path to private key (monitord). Empty for no encryption.
    #*******************************************************************************
    UDP_LISTENER = [
        ADDRESS         = "0.0.0.0",
        MONITOR_ADDRESS = 127.0.0.1,
        PORT    = 4124,
        THREADS = 8,
        PUBKEY  = "",
        PRIKEY  = ""
    ]

    #*******************************************************************************
    # Probes Configuration
    #*******************************************************************************
    # PORBES_PERIOD: Time in seconds to execute each probe category
    #   beacon_host: heartbeat for the host
    #   system_host:  host static/configuration information
    #   monitor_host: host variable information
    #   state_vm:    VM status (ie. running, error, stopped...)
    #   monitor_vm:   VM resource usage metrics
    #   sync_vm_state: When monitor probes have been stopped more than sync_vm_state
    #                  seconds, send a complete VM report.
    #*******************************************************************************
    PROBES_PERIOD = [
        BEACON_HOST    = 5,
        SYSTEM_HOST    = 600,
        MONITOR_HOST   = 120,
        STATE_VM       = 5,
        MONITOR_VM     = 90,
        SYNC_STATE_VM  = 180
    ]

For each hypervisor there must be configuration of Monitor Driver/Agent (todo choose correct word)

**KVM**:

.. note:: every setting is also aplicable to LXD

.. code::

    #*******************************************************************************
    # Information Driver Configuration
    #*******************************************************************************
    # You can add more monitor drivers with different configurations but make
    # sure it has different names.
    #
    #   name      : name for this monitor driver
    #
    #   executable: path of the monitor driver executable, can be an
    #               absolute path or relative to $ONE_LOCATION/lib/mads (or
    #               /usr/lib/one/mads/ if OpenNebula was installed in /)
    #
    #   arguments : for the driver executable, usually a probe configuration file,
    #               can be an absolute path or relative to $ONE_LOCATION/etc (or
    #               /etc/one/ if OpenNebula was installed in /)
    #
    #   threads   : How many threads should be used to process messages
    #               0 process the message in main loop
    #-------------------------------------------------------------------------------
    #  KVM UDP-push Information Driver Manager Configuration
    #    -r number of retries when monitoring a host
    #    -t number of threads, i.e. number of hosts monitored at the same time
    #    -w Timeout in seconds to execute external commands (default unlimited)
    #-------------------------------------------------------------------------------
    IM_MAD = [
        NAME          = "kvm",
        SUNSTONE_NAME = "KVM",
        EXECUTABLE    = "one_im_ssh",
        ARGUMENTS     = "-r 3 -t 15 -w 90 kvm",
        THREADS       = 0
    ]


.. _monitoring_troubleshooting:

Troubleshooting
===============

Healthy Monitoring System
-------------------------

Default location for monitoring log file is ``/var/log/one/monitor.log``
Every (approximately) ``monitoring_push_cycle`` of seconds OpenNebula is receiving the monitoring data of every Virtual Machine and of a host like such:

.. code::

    Sun Mar 15 22:12:15 2020 [Z0][HMM][I]: Successfully monitored VM: 0
    Sun Mar 15 22:13:10 2020 [Z0][HMM][I]: Successfully monitored host: 0
    Sun Mar 15 22:13:45 2020 [Z0][HMM][I]: Successfully monitored VM: 2
    Sun Mar 15 22:15:10 2020 [Z0][HMM][I]: Successfully monitored host: 1

However, if in ``monitor.log`` a host is being monitored **actively** periodically (every ``MONITORING_INTERVAL_HOST`` seconds) then the monitorization is **not** working correctly:

.. code::

    Sun Mar 15 22:31:55 2020 [Z0][HMM][D]: Monitoring host localhost(0)
    Sun Mar 15 22:31:59 2020 [Z0][HMM][D]: Start monitor success, host: 0
    Sun Mar 15 22:35:10 2020 [Z0][HMM][D]: Monitoring host localhost(0)
    Sun Mar 15 22:35:19 2020 [Z0][HMM][D]: Start monitor success, host: 0

If this is the case it's probably because Monitor Daemon doesn't receive any data from probes, could be caused by wrong UDP settings.

Monitoring Probes
-----------------

For the troubleshooting of errors produced during the execution of the monitoring probes, please refer to the :ref:`troubleshooting <monitoring_troubleshooting>` section.

Tuning & Extending
==================

Adjust Monitoring Interval Times
--------------------------------

In order to tune your OpenNebula installation with appropriate values of the monitoring parameters you need to adjust the monitoring intervals in ``PROBES_PERIOD`` section.

If the system is not working healthily it could be due to the database throughput. If the number of virtual machines and hosts is too large and the monitoring periods too low, OpenNebula will not be able to write that amount of data to the database.

Driver Files
------------

The probes are specialized programs that obtain the monitor metrics. Probes are defined for each hypervisor, and are located at ``/var/lib/one/remotes/im/kvm-probes.d`` for KVM.

You can easily write your own probes or modify existing ones, please see the :ref:`Information Manager Drivers <devel-im>` guide. Remember to synchronize the monitor probes in the hosts using ``onehost sync`` as described in the :ref:`Managing Hosts <host_guide_sync>` guide.

.. |image0| image:: /images/collector.png
