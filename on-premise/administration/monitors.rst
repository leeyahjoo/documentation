.. _monitors:

Monitors
========

The enStratus monitor service is a Java service installed to /services/monitor. It is
composed of two components: assign and control.

.. figure:: ./images/monitorWorker.png
   :height: 300 px
   :width: 450 px
   :scale: 70 %
   :alt: Monitors/Workers Service
   :align: center

   Monitors/Workers Service Connections

Overview
--------

The enStratus monitors service is responsible for maintaining an accurate representation
of cloud state, checking on the completion of jobs initiated by the dispatcher service,
orchestrating server launches and service installations.

The relationship of the enStratus monitors to the rest of the enStratus cloud management
system is shown in the above diagram.

The monitors service does not listen for incoming connections.

The purpose of the monitor system of enStratus is to, well, monitor cloud state and
accommodate changes that happen both inside of the enStratus cloud management system and
discover changes that happen outside of enStratus, for example when a user initiates an
action in the cloud provider console.

Installation
------------

Installation of the enStratus monitor service is best handled by using a configuration
management system such as Chef or Puppet.

Software Requirements
---------------------

The monitor service depends on the Open JDK.

Incoming Connections
--------------------

None.


Outgoing Connections
--------------------

#. Cloud API

   The monitor service connects to the cloud provider API to discover changes, track the
   completion of jobs initiated by the dispatcher, and maintain cloud state.

#. Riak and MySQL

   The monitor service connects to the Riak and MySQL databases for the purposes of pushing
   state information. enStratus monitors also connect directly to the cloud provider API and
   poll to discover changes. 
   
   The polling interval for each enStratus monitor varies depending on the cloud service
   being monitored. Cloud resources that require up-to-the-minute monitoring, for example
   when servers are started, will be polled more rapidly than less critical resources such as
   the creation of a snapshot, or the deletion of a volume.

#. Guest VM

   The monitor service will also poll guest VM running in the cloud for the purpose of
   collecting logs.

Starting Monitor
----------------

To start the monitor services:

.. code-block:: bash

	/etc/init.d/enstratus-monitor start

Stopping Monitor
----------------
To stop the monitor services:

.. code-block:: bash

	/etc/init.d/enstratus-monitor stop

Running Monitors
----------------

In a high demand environment, or in environments where the amount of resources per server
is limited, it is possible to split up the monitors that run on any given machine. For
example, you may want to run high-activity monitors like the server or image monitor on a
separate machine.

The downside of doing this is that it introduces unique server in your deployed enStratus
environment and can add management complexity.

It's also possible to run multiple monitor processes on the same server or on multiple
servers. For example, to separate hosts can be running the server monitor. This can be
done for any of the monitor services.

To adjust what monitors start/stop, simply edit the file
``/services/monitor/resources/monitors.cfg`` and remove any monitors that aren't
appropriate or are perhaps running on another host. For example, if the cloud providers
you are managing with enStratus do not have a concept of block storage devices, you may
want to disable the unnecessary volume and snapshot monitors.

Individual monitor processes are not in and of themselves critical. Not having a monitor
running will result in changes not being picked up and displayed in the enStratus console.
Starting the monitor will cause the monitor to poll and discover any changes that have
occurred. The monitors are stateless, and can be stopped and started at any time.

Logging
-------

Logging for the monitors is done to:

``/services/monitor/logs/assign.log``

``/services/monitor/logs/monitor.log``

Monitoring
----------

Backups
-------

Service
~~~~~~~

The enStratus monitor service files should be backed up before and after any changes, and
once/day during steady-state operations. Backups should be performed on
/services/monitor.

An example of how to backup the monitor service is shown here, in this case excluding the
log directory.

.. code-block:: bash

   #!/bin/bash
   
   TAR=/bin/tar
   GZIP=/bin/gzip
   
   DIR=/var/enstratus/backups
   BASE=monitors
   DA=`date +%Y%m%d-%H%M%S`
   
   FILE=${DIR}/${BASE}-${DA}.tar.gz
   
   find ${DIR} -type f -iname "*.gz" -mtime +2 | xargs rm -f
   
   cd /services/monitor/
   $TAR -czf ${FILE}  --exclude='work/*' --exclude='log/*' . > /dev/null 2>&1
   chmod 700 ${FILE}

Databases
~~~~~~~~~

The enStratus monitor service depends on the provisioning and analytics databases along
with the enStratus dispatcher service. Backups of these database are discussed in the
dispatcher service section.

Configuration Files
-------------------

The enStratus monitors service has 7 configuration files in /services/monitor/

.. hlist::
   :columns: 2

   * resources/log4j.properties
   * resources/cloud.properties
   * resources/dasein-persistence.properties
   * resources/enstratus-km-client.cfg
   * resources/enstratus-provisioning.cfg
   * resources/mq.cfg
   * resources/monitors.cfg

log4j.properties
~~~~~~~~~~~~~~~~

Path:

  ``/services/monitor/resources/log4j.properties``

This file is responsible for controlling the logging behavior for the monitor service.

cloud.properties
~~~~~~~~~~~~~~~~

Path:

  ``/services/monitor/resources/cloud.properties``

The cloud.properties file is used to define the connection points for the monitor service
to connect to the provisioning and analytics MySQL databases.

dasein-persistence.properties
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Path:

  ``/services/monitor/resources/dasein-persistence.properties``

This file defines the connection to the dasein persistence layer of enStratus. It also
specifies the connection point to the Riak database service.

enstratus-km-client.cfg
~~~~~~~~~~~~~~~~~~~~~~~

Path:

  ``/services/monitor/resources/enstratus-km-client.cfg``

This file controls the connection to the KM service by the monitors. 

enstratus-provisioning.cfg
~~~~~~~~~~~~~~~~~~~~~~~~~~

Path:

  ``/services/monitor/resources/enstratus-provisioning.cfg``

This file is a general control point for several items, the most important of which is the
encryption key for encrypting connections to the KM service. This is also where a setting
called SOURCE_CIDR is made, which specifies IP addresses from which enStratus will make
connections to guest VM.

mq.cfg
~~~~~~

Path:

  ``/services/monitor/resources/mq.cfg``

This file controls how the monitor service connects to the mq service.

monitors.cfg
~~~~~~~~~~~~

Path:

  ``/services/monitor/resources/monitors.cfg``

The is file is used to specify which of the enStratus monitors are started during the
start process. This file is read by the assign process.
