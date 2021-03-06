.. _`The Durability Service`:

######################
The Durability Service
######################

*This section provides a description the most important concepts and mechanisms of
the current durability service implementation, starting with a description of the
purpose of the service. After that all its concepts and mechanisms are described.*

The exact fulfilment of the durability responsibilities is determined by the
configuration of the Durability Service.
There are detailed descriptions of all of the available configuration
parameters and their purpose in the :ref:`Configuration <Configuration>`
section.


.. _`Durability Service Purpose`:

Durability Service Purpose
**************************

Vortex OpenSplice will make sure data is delivered to all ‘compatible’ subscribers
that are available at the time the data is published using the ‘communication paths’
that are implicitly created by the middleware based on the interest of applications
that participate in the domain. However, subscribers that are created after the data
has been published (called late-joiners) may also be interested in the data that was
published before they were created (called historical data). To facilitate this use
case, DDS provides a concept called durability in the form of a Quality of Service
(*DurabilityQosPolicy*).

The ``DurabilityQosPolicy`` prescribes how published data needs to be maintained by
the DDS middleware and comes in four flavours:

*VOLATILE*
  Data does not need to be maintained for late-joiners (*default*).
*TRANSIENT_LOCAL*
  Data needs to be maintained for as long as the DataWriter is active.
*TRANSIENT*
  Data needs to be maintained for as long as the middleware is
  running on at least one of the nodes.
*PERSISTENT*
  Data needs to outlive system downtime. This implies that it
  must be kept somewhere on permanent storage in order to be able to make
  it available again for subscribers after the middleware is restarted.

In Vortex OpenSplice, the realisation of the non-volatile properties is the
responsibility of the durability service. Maintenance and provision of historical data
could in theory be done by a single durability service in the domain, but for
fault-tolerance and efficiency one durability service is usually running on
every computing node. These durability services are on the one hand responsible for
maintaining the set of historical data and on the other hand responsible for providing
historical data to late-joining subscribers. The configurations of the different
services drive the behaviour on where and when specific data will be maintained
and how it will be provided to late-joiners.

.. _`Durability Service Concepts`:

Durability Service Concepts
***************************

The following subsections describe the concepts that drive the implementation of
the OpenSplice Durability Service.

.. _`Role and Scope`:

Role and Scope
==============

Each OpenSplice node can be configured with a so-called role. A role is a logical
name and different nodes can be configured with the same role. The role itself does
not impose anything, but multiple OpenSplice services use the role as a mechanism
to distinguish behaviour between nodes with the equal and different roles.
The durability service allows configuring a so-called scope, which is an expression
that is matches against roles of other nodes. By using a scope, the durability service
can be instructed to apply different behaviour with respect to merging of historical
data sets (see `Merge policy`_) to and from nodes that have equal or different roles.

Please refer to the  :ref:`Configuration <Configuration>` section for
detailed descriptions of:

+  ``//OpenSplice/Domain/Role``
+  ``//OpenSplice/DurabilityService/NameSpaces/Policy/Merge[@scope]``


.. _`Name-spaces`:

Name-spaces
===========

A sample published in DDS for a specific topic and instance is bound to one logical
partition. This means that in case a publisher is associated with multiple partitions, a
separate sample for each of the associated partitions is created. Even though they are
syntactically equal, they have different semantics (consider for instance the situation
where you have a sample in the ‘simulation’ partition *versus* one in the ‘real world’
partition).

Because applications might impose semantic relationships between instances
published in different partitions, a mechanism is required to express this relationship
and ensure consistency between partitions. For example, an application might
expect a specific instance in partition *Y* to be available when it reads a specific
instance from partition *X*.

This implies that the data in both partitions need to be maintained as one single set.
For persistent data, this dependency implies that the durability services in a domain
needs to make sure that this data set is re-published from one single persistent store
instead of combining data coming from multiple stores on disk. To express this
semantic relation between instances in different partitions to the durability service,
the user can configure so-called ‘name-spaces’ in the durability configuration file.

Each name-space is formed by a collection of partitions and all instances in such a
collection are always handled as an atomic data-set by the durability service. In
other words, the data is guaranteed to be stored and reinserted as a whole.

This atomicity also implies that a name-space is a system-wide concept, meaning
that different durability services need to agree on its definition, *i.e.* which
partitions belong to one name-space. This doesn’t mean that each durability service
needs to know all name-spaces, as long as the name-spaces one does know don’t conflict
with one of the others in the domain. Name-spaces that are completely disjoint can
co-exist (their intersection is an empty set); name-spaces conflict when they
intersect. For example: name-spaces {p1, q} and {p2, r} can co-exist, but
name-spaces {s, t} and {s, u} cannot.

Furthermore it is important to know that there is a set of configurable policies for
name-spaces, allowing durability services throughout the domain to take different
responsibilities for each name-space with respect to maintaining and providing of
data that belongs to the name-space. The durability name-spaces define the mapping
between logical partitions and the responsibilities that a specific durability service
needs to play. In the default configuration file there is only one name-space by
default (holding all partitions).

Next to the capability of associating a semantic relationship for data in one
name-space, the need to differentiate the responsibilities of a particular durability
service for a specific data-set is the second purpose of a name-space. Even though
there may not be any relation between instances in different partitions, the choice of
grouping specific partitions in different name-spaces can still be perfectly valid. The
need for availability of non-volatile data under specific conditions (fault-tolerance)
on the one hand *versus* requirements on performance (memory usage, network
bandwidth, CPU usage, *etc.*) on the other hand may force the user to split up the
maintaining of the non-volatile data-set over multiple durability services in the
domain. Illustrative of this balance between fault-tolerance and performance is the
example of maintaining all data in all durability services, which is maximally
fault-tolerant, but also requires the most resources. The name-spaces concept allows
the user to divide the total set of non-volatile data over multiple name-spaces and
assign different responsibilities to different durability-services in the form of
so-called name-space policies.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/NameSpaces/NameSpace``


.. _`Name-space policies`:

Name-space policies
===================

This section describes the policies that can be configured per name-space giving the
user full control over the fault-tolerance versus performance aspect on a per
name-space level.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/NameSpaces/Policy``


.. _`Alignment policy`:

Alignment policy
----------------

The durability services in a domain are on the one hand responsible for maintaining the
set of historical data between services and on the other hand responsible for
providing historical data to late-joining applications. The configurations of the
different services drive the behaviour on where and when specific data will be kept
and how it will be provided to late-joiners. The optimal configuration is driven by
fault-tolerance on the one hand and resource usage (like CPU usage, network
bandwidth, disk space and memory usage) on the other hand. One mechanism to
control the behaviour of a specific durability service is the usage of alignment
policies that can be configured in the durability configuration file. This
configuration option allows a user to specify if and when data for a specific
name-space (see the section about `Name-spaces`_) will be maintained by the durability
service and whether or not it is allowed to act as an aligner for other durability
services when they require (part of) the information.

The alignment responsibility of a durability service is therefore configurable by
means of two different configuration options being the aligner and alignee
responsibilities of the service:

**Aligner policy**

*TRUE*
  The durability service will align others if needed.
*FALSE*
  The durability service will not align others.

**Alignee policy**

*INITIAL*
  Data will be retrieved immediately when the data is available and
  continuously maintained from that point forward.
*LAZY*
  Data will be retrieved on first arising interest on the local node and
  continuously maintained from that point forward.
*ON_REQUEST*
  Data will be retrieved only when requested by a subscriber, but
  not maintained. Therefore each request will lead to a new alignment action.

Please refer to the  :ref:`Configuration <Configuration>` section for
detailed descriptions of:

+  ``//OpenSplice/DurabilityService/NameSpaces/Policy[@aligner]``
+  ``//OpenSplice/DurabilityService/NameSpaces/Policy[@alignee]``



.. _`Durability policy`:

Durability policy
-----------------

The durability service is capable of maintaining (part of) the set of non-volatile data
in a domain. Normally this results in the outcome that data which is written as
volatile is not stored, data written as transient is stored in memory and data that is
written as persistent is stored in memory and on disk. However, there are use cases
where the durability service is required to ‘weaken’ the DurabilityQosPolicy
associated with the data, for instance by storing persistent data only in memory as if
it were transient. Reasons for this are performance impact (CPU load, disk I/O) or
simply because no permanent storage (in the form of some hard-disk) is available on
a node. Be aware that it is not possible to ‘strengthen’ the durability of the data
(Persistent > Transient > Volatile).

The durability service has the following options
for maintaining a set of historical data:

*PERSISTENT*
  Store persistent data on permanent storage, keep transient data in
  memory, and don’t maintain volatile data.
*TRANSIENT*
  Keep both persistent and transient data in memory, and don’t
  maintain volatile data.
*VOLATILE*
  Don’t maintain persistent, transient, or volatile data.

This configuration option is called the ‘durability policy’.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/NameSpaces/Policy[@durability]``


.. _`Delayed alignment policy`:

Delayed alignment policy
------------------------

The durability service has a mechanism in place to make sure that when multiple
services with a persistent dataset exist, only one set (typically the one with the
newest state) will be injected in the system (see `Persistent data injection`_).
This mechanism will, during the startup of the durability service, negotiate with
other services which one has the best set (see `Master selection`_).
After negotiation the ‘best’ persistent set (which can be empty) is restored
and aligned to all durability services.

Once persistent data has been re-published in the domain by a durability service for
a specific name-space, other durability services in that domain cannot decide to
re-publish their own set for that name-space from disk any longer. Applications may
already have started their processing based on the already-published set, and
re-publishing another set of data may confuse the business logic inside applications.
Other durability services will therefore back-up their own set of data and align and
store the set that is already available in the domain.

It is important to realise that an empty set of data is also considered a set.
This means that once a durability service in the domain decides that there is no data
(and has triggered applications that the set is complete), other late-joining
durability services will not re-publish any persistent data that they potentially
have available.

Some systems however do require re-publishing persistent data from disk if the
already re-published set is empty and no data has been written for the corresponding
name-space. The durability service can be instructed to still re-publish data from
disk in this case by means of an additional policy in the configuration called
‘delayed alignment’. This Boolean policy instructs a late-joining durability service
whether or not to re-publish persistent data for a name-space that has been marked
complete already in the domain, but for which no data exists and no DataWriters
have been created. Whatever setting is chosen, it should be consistent between *all*
durability services in a domain to ensure proper behaviour on the system level.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/NameSpaces/Policy[@delayedAlignment]``


.. _`Merge policy`:

Merge policy
------------

A *`split-brain syndrome’* can be described as the situation in which two different
nodes (possibly) have a different perception of (part of) the set of historical data.
This split-brain occurs when two nodes or two sets of nodes (*i.e.* two systems) that
are participating in the same DDS domain have been running separately for some
time and suddenly get connected to each other. This syndrome also arises when
nodes re-connect after being disconnected for some time. Applications on these
nodes may have been publishing information for the same topic in the same
partition without this information reaching the other party. Therefore their
perception of the set of data will be different.

In many cases, after this has occurred the exchange of information is no longer
allowed, because there is no guarantee that data between the connected systems doesn’t
conflict. For example, consider a fault-tolerant (distributed) global id service:
this service will provide globally-unique ids, but this will be guaranteed
*if and only if* there is no disruption of communication between all services. In such
a case a disruption must be considered permanent and a reconnection must be avoided
at any cost.

Some new environments demand supporting the possibility to (re)connect two
separate systems though. One can think of *ad-hoc* networks where nodes
dynamically connect when they are near each other and disconnect again when
they’re out of range, but also systems where temporal loss of network connections is
normal. Another use case is the deployment of Vortex OpenSplice in a hierarchical
network, where higher-level ‘branch’ nodes need to combine different historical
data sets from multiple ‘leaves’ into its own data set. In these new environments
there is the same strong need for the availability of data for ‘late-joining’
applications (non-volatile data) as in any other system.

For these kinds of environments the durability service has additional functionality to
support the alignment of historical data when two nodes get connected. Of course,
the basic use case of a newly-started node joining an existing system is supported,
but in contradiction to that situation there is no universal truth in determining who
has the best (or the right) information when two already running nodes (re)connect.
When this situation occurs, the durability service provides the following
possibilities to handle the situation:

*IGNORE*
  Ignore the situation and take no action at all. This means new
  knowledge is not actively built up. Durability is passive and will only build up
  knowledge that is ‘implicitly’ received from that point forward (simply by
  receiving updates that are published by applications from that point forward and
  delivered using the normal publish-subscribe mechanism).
*DELETE*
  Dispose and delete all historical data. This means existing data is
  disposed and deleted and other data is not actively aligned. Durability is passive
  and will only maintain data that is ‘implicitly’ received from that point forward.
*MERGE*
  Merge the historical data with the data set that is available on the
  connecting node.
*REPLACE*
  Dispose and replace all historical data by the data set that is available
  on the connecting node. Because all data is disposed first, a side effect is that
  instances present both before and after the merge operation transition through
  ``NOT_ALIVE_DISPOSED`` and end up as *NEW* instances, with corresponding
  changes to the instance generation counters.
*CATCHUP*
  Updates the historical data to match the historical data on the remote
  node by disposing those instances available in the local set but not in the remote
  set, and adding and updating all other instances. The resulting data set is the
  same as that for the *REPLACE* policy, but without the side effects. In particular,
  the instance state of instances that are both present on the local node and remote
  node and for which no updates have been done will remain unchanged.

|caution|

  Note that *REPLACE* and *CATCHUP* result in the same data set, but the
  instance states of the data may differ.

From this point forward this set of options will be referred to as *‘merge policies’*.

Like the networking service, the durability service also allows configuration of a
so-called scope to give the user full control over what merge policy should be
selected based on the role of the re-connecting node. The scope is a logical
expression and every time nodes get physically connected, they match the role of
the other party against the configured scope to see whether communication is
allowed and if so, whether a merge action is required.

As part of the merge policy configuration, one can also configure a scope. This
scope is matched against the role of remote durability services to determine what
merge policy to apply. Because of this scope, the merge behaviour for
(re-)connections can be configured on a *per role* basis. It might for instance be
necessary to merge data when re-connecting to a node with the same role, whereas
(re-)connecting to a node with a different role requires no action.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/NameSpaces/Policy/Merge``


.. _`Prevent aligning equal data sets`:

Prevent aligning equal data sets
--------------------------------

As explained in previous sections, temporary disconnections can cause durability
services to get out-of-sync, meaning that their data sets may diverge. To recover
from such situations merge policies have been defined (see `Merge policy`_)
where a user can specify how to combine divergent data sets
when they become reconnected. Many of these situations involve the transfer of
data sets from one durability service to the other. This may generate a considerable
amount of traffic for large data sets.

If the data sets do not get out-of-sync during disconnection it is not necessary to
transfer data sets from one durability service to the other. Users can specify whether
to compare data sets before alignment using the ``equalityCheck`` attribute.
When this check is enabled, hashes of the data sets are calculated and compared;
when they are equal, no data will be aligned. This may save valuable bandwidth
during alignment. If the hashes are different then the complete data sets will be
aligned.

Comparing data sets does not come for free as it requires hash calculations over data
sets. For large sets this overhead may become significant; for that reason is not
recommended to enable this feature for frequently-changing data sets. Doing so will
impose the penalty of having to calculate hashes when the hashes are likely to differ
and the data sets need to be aligned anyway.

Comparison of data sets using hashes is currently only supported for operational
nodes that diverge; no support is provided during initial startup.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/NameSpaces/Policy[@equalityCheck]``


.. _`Dynamic name-spaces`:

Dynamic name-spaces
-------------------

As specified in the previous sections, a set of policies can be configured for a (set
of) given name-space(s). One may not know the complete set of name-spaces for the
entire domain though, especially when new nodes dynamically join the domain.
However, in case of maximum fault-tolerance, one may still have the need to define
behaviour for a durability service by means of a set of policies for name-spaces that
have not been configured on the current node.

Every name-space in the domain is identified by a logical name. To allow a
durability service to fulfil a specific role for any name-space, each policy needs be
configured with a name-space expression that is matched against the name of
name-spaces in the domain. If the policy matches a name-space, it will be applied
by the durability service, independently of whether or not the name-space itself is
configured on the node where this durability service runs. This concept is referred to
as *‘dynamic name-spaces’*.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/NameSpaces/Policy[@nameSpace]``



.. _`Master/slave`:

Master/slave
------------

Each durability service that is responsible for maintaining data in a namespace must
maintain the complete set for that namespace. It can achieve this by either
requesting data from a durability service that indicates it has a complete set or, if
none is available, request all data from all services for that namespace and combine
this into a single complete set. This is the only way to ensure all available data will
be obtained. In a system where all nodes are started at the same time, none of the
durability services will have the complete set, because applications on some nodes
may already have started to publish data. In the worst case every service that starts
then needs to ask every other service for its data. This concept is not very scalable
and also leads to a lot of unnecessary network traffic, because multiple nodes may
(partly) have the same data. Besides that, start-up times of such a system will
grow exponentially when adding new nodes. Therefore the so-called ‘master’
concept has been introduced.

Durability services will determine one ‘master’ for every name-space per
configured role amongst themselves. Once the master has been selected, this master
is the one that will obtain all historical data first (this also includes re-publishing its
persistent data from disk) and all others wait for that process to complete before
asking the master for the complete set of data. The advantage of this approach is that
only the master (potentially) needs to ask all other durability services for their data
and all others only need to ask just the master service for its complete set of data
after that.

Additionally, a durability service is capable of combining alignment requests
coming from multiple remote durability services and will align them all at the same
time using the internal multicast capabilities. The combination of the master concept
and the capability of aligning multiple durability services at the same time make the
alignment process very scalable and prevent the start-up times from growing when
the number of nodes in the system grows. The timing of the durability protocol can
be tweaked by means of configuration in order to increase chances of combining
alignment requests. This is particularly useful in environments where multiple
nodes or the entire system is usually started at the same time and a considerable
amount of non-volatile data needs to be aligned.



.. _`Mechanisms`:

Mechanisms
**********

.. _`Interaction with other durability services`:

Interaction with other durability services
==========================================

To be able to obtain or provide historical data, the durability service needs to
communicate with other durability services in the domain. These other durability
services that participate in the same domain are called *‘fellows’*. The durability
service uses regular DDS to communicate with its fellows. This means all
information exchange between different durability services is done with via
standard DataWriters and DataReaders (without relying on non-volatile data
properties of course).

Depending on the configured policies, DDS communication is used to determine
and monitor the topology, exchange information about available historical data and
alignment of actual data with fellow durability services.


.. _`Interaction with other OpenSplice services`:

Interaction with other OpenSplice services
==========================================

In order to communicate with fellow durability services through regular DDS
DataWriters and DataReaders, the durability service relies on the availability of a
network service. This can be either the interoperable DDSI or the real-time
networking service. It can even be a combination of multiple networking services in
more complex environments. As networking services are pluggable like the
durability service itself, they are separate processes or threads that perform tasks
asynchronously next to the tasks that the durability service is performing. Some
configuration is required to instruct the durability service to synchronise its
activities with the configured networking service(s). The durability service aligns
data separately per partition-topic combination. Before it can start alignment for a
specific partition-topic combination it needs to be sure that the networking
service(s) have detected the partition-topic combination and ensure that data
published from that point forward is delivered from *c.q.* sent over the network. The
durability service needs to be configured to instruct it which networking service(s)
need to be attached to a partition-topic combination before starting alignment. This
principle is called *‘wait-for-attachment’*.

Furthermore, the durability service is responsible to announce its liveliness
periodically with the splice-daemon. This allows the splice-daemon to take
corrective measures in case the durability service becomes unresponsive. The
durability service has a separate so-called *`watch-dog’* thread to perform this task.
The configuration file allows configuring the scheduling class and priority of this
watch-dog thread.

Finally, the durability service is also responsible to monitor the splice-daemon. In
case the splice-daemon itself fails to update its lease or initiates regular termination,0
the durability service will terminate automatically as well.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/Network``


.. _`Interaction with applications`:

Interaction with applications
=============================

The durability service is responsible for providing historical data to
late-joining subscribers.

Applications can use the DCPS API call ``wait_for_historical_data`` on a DataReader
to synchronise on the availability of the complete set of historical data.
Depending on whether the historical data is already available locally, data can be
delivered immediately after the DataReader has been created or must be aligned from
another durability service in the domain first. Once all historical data is delivered
to the newly-created DataReader, the durability service will trigger the DataReader
unblocking the ``wait_for_historical_data`` performed by the application. If the
application does not need to block until the complete set of historical data is
available before it starts processing, there is no need to call
``wait_for_historical_data``. It should be noted that in such a case historical
data still is delivered by the durability service when it becomes available.


.. _`Parallel alignment`:

Parallel alignment
==================

When a durability service is started and joins an already running domain, it usually
obtains historical data from one or more already running durability services. In case
multiple durability services are started around the same time, each one of them
needs to obtain a set of historical data from the already running domain. The set of
data that needs to be obtained by the various durability services is often the same or
at least has a large overlap. Instead of aligning each newly joining durability service
separately, aligning all of them at the same time is very beneficial, especially if the
set of historical data is quite big. By using the built-in multi-cast and broadcast
capabilities of DDS, a durability service is able to align as many other durability
services as desired in one go. This ability reduces the CPU, memory and bandwidth
usage of the durability service and makes the alignment scale also in situations
where many durability services are started around the same time and a large set of
historical data exists. The concept of aligning multiple durability service at the same
time is referred to as *‘parallel alignment’*.

To allow this mechanism to work, durability services in a domain determine a
master durability service for each name-space. Every durability service elects the
same master for a given name-space based on a set of rules that will be explained
later on in this document. When a durability service needs to be aligned, it will
always send its request for alignment to its selected master. This results in only one
durability service being asked for alignment by any other durability service in the
domain for a specific name-space, but also allows the master to combine similar
requests for historical data. To be able to combine alignment requests from different
sources, a master will wait a period of time after receiving a request and before
answering a request. This period of time is called the *‘request-combine period’*.

The actual amount of time that defines the ‘request-combine period’ for the
durability service is configurable. Increasing the amount of time will increase the
likelihood of parallel alignment, but will also increase the amount of time before it
will start aligning the remote durability service and in case only one request comes
in within the configured period, this is non-optimal behaviour. The optimal
configuration for the request-combine period therefore depends heavily on the
anticipated behaviour of the system and optimal behaviour may be different in every
use case.

In some systems, all nodes are started simultaneously, but from that point forward
new nodes start or stop sporadically. In such systems, different configuration with
respect to the request-combine period is desired when comparing the start-up and
operational phases. That is why the configuration of this period is split into different
settings: one during the start-up phase and one during the operational phase.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/Network/Alignment/RequestCombinePeriod``


.. _`Tracing`:

Tracing
=======

Configuring durability services throughout a domain and finding out what exactly
happens during the lifecycle of the service can prove difficult.

OpenSplice developers sometimes have a need to get more detailed durability
specific state information than is available in the regular OpenSplice info and error
logs to be able to analyse what is happening. To allow retrieval of more internal
information about the service for (off-line) analysis to improve performance or
analyse potential issues, the service can be configured to trace its activities to a
specific output file on disk.

By default, this tracing is turned off for performance reasons, but it can be enabled
by configuring it in the XML configuration file.

The durability service supports various tracing verbosity levels. In general can be
stated that the more verbose level is configured (*FINEST* being the most verbose),
the more detailed the information in the tracing file will be.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/Tracing``


.. _`Lifecycle`:

Lifecycle
*********

During its lifecycle, the durability service performs all kinds of activities to be able
to live up to the requirements imposed by the DDS specification with respect to
non-volatile properties of published data. This section describes the various
activities that a durability service performs to be able to maintain non-volatile data
and provide it to late-joiners during its lifecycle.


.. _`Determine connectivity`:

Determine connectivity
======================

Each durability service constantly needs to have knowledge on all other durability
services that participate in the domain to determine the logical topology and changes
in that topology (*i.e.* detect connecting, disconnecting and re-connecting nodes).
This allows the durability service for instance to determine where non-volatile data
potentially is available and whether a remote service will still respond to requests
that have been sent to it reliably.

To determine connectivity, each durability service sends out a heartbeat periodically
(every configurable amount of time) and checks whether incoming heartbeats have
expired. When a heartbeat from a fellow expires, the durability service considers
that fellow disconnected and expects no more answers from it. This means a new
aligner will be selected for any outstanding alignment requests for the disconnected
fellow. When a heartbeat from a newly (re)joining fellow is received, the durability
service will assess whether that fellow is compatible and if so, start exchanging
information.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/Network/Heartbeat``


.. _`Determine compatibility`:

Determine compatibility
=======================

When a durability service detects a remote durability service in the domain it is
participating in, it will determine whether that service has a compatible
configuration before it will decide to start communicating with it. The reason not to
start communicating with the newly discovered durability service would be a
mismatch in configured name-spaces. As explained in the section about the
`Name-spaces`_ concept, having different name-spaces is not an issue as long as they do
not overlap. In case an overlap is detected, no communication will take place
between the two ‘incompatible’ durability services. Such an incompatibility in your
system is considered a mis-configuration and is reported as such in the OpenSplice
error log.

Once the durability service determines name-spaces are compatible with the ones of
all discovered other durability services, it will continue with selection of a master
for every name-space, which is the next phase in its lifecycle.


.. _`Master selection`:

Master selection
================

For each Namespace and Role combination there shall be at most one Master Durability
Service. The Master Durability Service coordinates single source re-publishing of
persistent data and to allow parallel alignment after system start-up,
and to coordinate recovery of a split brain syndrome after connecting nodes having
selected a different Master indicating that more than one state of the data may exist.

Therefore after system start-up as well as after any topology change (i.e. late joining
nodes or leaving master node) a master selection process will take place for each
affected Namespace/Role combination.

To control the master selection process a masterPriority attribute can be used.

Each Durability Service will have a configured masterPriority attribute per namespace
which is an integer value between 0 and 255 and which specifies the eagerness of the
Durability Service to become Master for that namespace.
The values 0 and 255 have a special meaning.
Value 0 is used to indicate that the Durability Service will never become Master.
The value 255 is used to indicate that the Durability Service will not use priorities
but instead uses the legacy selection algorithm.
If not configured the default is 255.

During the master selection process each Durability service will exchange for each
namespace the masterPriority and quality. The namespace quality is the timestamp of the
latest update of the persistent data set stored on disk and only plays a role in master
selection initially when no master has been chosen before and persistent data has not
been injected yet.

Each Durability Service will determine the Master based upon the highest non zero
masterPriority and in case of multiple masters further select based on namespace
quality (but only if persistent data has not been injected before) and again in case of
multiple masters select the highest system id. The local system id is an arbitrary
value which unique identifies a durability service.
After selection each Durability Service will communicate their determined master and on
agreement effectuate the selection, on disagreement which may occur if some Durability
Services had a temporary different view of the system this process of master selection
will restart until all Durability Services have the same view of the system and have
made the same selection.
If no durability services exists having a masterPriority greater than zero then no
master will be selected.

Summarizing, the precedence rules for master selection are (from high to low):

1. The namespace masterPriority
2. The namespace quality, if no data has been injected before.
3. The Durability Service system id, which is unique for each durability service.

Please refer to the  :ref:`Configuration <Configuration>` section for
a detailed description of:

+  ``//OpenSplice/DurabilityService/Network/InitialDiscoveryPeriod``


.. _`Persistent data injection`:

Persistent data injection
=========================

As persistent data needs to outlive system downtime, this data needs to be
re-published in DDS once a domain is started.

If only one node is started, the durability service on that node can simply
re-publish the persistent data from its disk. However, if multiple nodes are
started at the same time, things become more difficult. Each one of them may
have a different set available on permanent storage due to the fact that
durability services have been stopped at a different moment in time.
Therefore only one of them should be allowed to re-publish its data, to prevent
inconsistencies and duplication of data.

The steps below describe how a durability service currently determines whether or
not to inject its data during start-up:

1. *Determine validity of own persistent data* —
   During this step the durability service determines whether its persistent
   store has initially been completely filled with all persistent data in the
   domain in the last run. If the service was shut down in the last run
   during initial alignment of the persistent data, the set of data will be
   incomplete and the service will restore its back-up of a full set of (older)
   data if that is available from a run before that. This is done because it
   is considered better to re-publish an older but complete set of data instead
   of a part of a newer set.

2. *Determine quality of own persistent data* —
   If persistence has been configured, the durability service will inspect the
   quality of its persistent data on start-up. The quality is determined on a
   *per-name-space* level by looking at the time-stamps of the persistent data
   on disk. The latest time-stamp of the data on disk is used as the quality
   of the name-space. This information is useful when multiple nodes are started
   at the same time. Since there can only be one source per name-space that is
   allowed to actually inject the data from disk into DDS, this mechanism allows
   the durability services to select the source that has the latest data, because
   this is generally considered the best data. If this is not true then an
   intervention is required. The data on the node must be replaced by the
   correct data either by a supervisory (human or system management application)
   replacing the data files or starting the nodes in the desired sequence
   so that data is replaced by alignment.

3. *Determine topology* —
   During this step, the durability service determines whether there are other
   durability services in the domain and what their state is.
   If this service is the only one, it will select itself as the ‘best’ source
   for the persistent data.

4. *Determine master* —
   During this step the durability service will determine who
   will inject persistent data or who has injected persistent data already.
   The one that will or already has injected persistent data is called the
   *‘master’*. This process is done on a per name-space level
   (see previous section).

   a) *Find existing master* –
      In case the durability service joins an already-running domain,
      the master has already been determined and this one has already
      injected the persistent data from its disk or is doing it right now.
      In this case, the durability service will set its current set of
      persistent data aside and will align data from the already existing
      master node. If there is no master yet, persistent data has not
      been injected yet.

   b) *Determine new master* –
      If the master has not been determined yet, the durability service
      determines the master for itself based on who has the best quality
      of persistent data. In case there is more than one service with the
      ‘best’ quality, the one with the highest system id (unique number) is
      selected. Furthermore, a durability service that is marked as not
      being an aligner for a name-space cannot become master for
      that name-space.

5. *Inject persistent data* —
   During this final step the durability service injects its persistent data
   from disk into the running domain. This is *only* done when the service has
   determined that it is the master. In any other situation the durability
   service backs up its current persistent store and fills a new store with
   the data it aligns from the master durability service in the domain, or
   postpones alignment until a master becomes available in the domain.

|caution|

  It is strongly discouraged to re-inject persistent data from a persistent
  store in a running system after persistent data has been published.
  Behaviour of re-injecting persistent stores in a running system is not
  specified and may be changed over time.


.. _`Discover historical data`:

Discover historical data
========================

During this phase, the durability service finds out what historical data is available in
the domain that matches any of the locally configured name-spaces. All necessary
topic definitions and partition information are retrieved during this phase. This step
is performed before the historical data is actually aligned from others. The process
of discovering historical data continues during the entire lifecycle of the service and
is based on the reporting of locally-created partition-topic combinations by each
durability service to all others in the domain.


.. _`Align historical data`:

Align historical data
=====================

Once all topic and partition information for all configured name-spaces are known,
the initial alignment of historical data takes place. Depending on the configuration
of the service, data is obtained either immediately after discovering it or only once
local interest in the data arises. The process of aligning historical data continues
during the entire lifecycle of the durability service.


.. _`Provide historical data`:

Provide historical data
=======================

Once (a part of) the historical data is available in the durability service, it is
able to provide historical data to local DataReaders as well as other durability
services.

Providing of historical data to local DataReaders is performed automatically as soon
as the data is available. This may be immediately after the DataReader is created (in
case historical data is already available in the local durability service at that time) or
immediately after it has been aligned from a remote durability service.

Providing of historical data to other durability services is done only on request by
these services. In case the durability service has been configured to act as an aligner
for others, it will respond to requests for historical data that are received. The set of
locally available data that matches the request will be sent to the durability service
that requested it.


.. _`Merge historical data`:

Merge historical data
=====================

When a durability service discovers a remote durability service and detects that
neither that service nor the service itself is in start-up phase, it concludes that they
have been running separately for a while (or the entire time) and both may have a
different (but potentially complete) set of historical data. When this situation occurs,
the configured merge-policies will determine what actions are performed to recover
from this situation. The process of merging historical data will be performed every
time two separately running systems get (re-)connected.

.. _`Threads description`:

Threads description
*******************

This section contains a short description of each durability thread. When applicable,
relevant configuration parameters are mentioned.

ospl_durability
===============
This is the main durability service thread. It starts most of the other threads, e.g.
the listener threads that are used to receive the durability protocol messages,
and it initiates initial alignment when necessary. This thread is responsible for
periodically updating the service-lease so that the splice-daemon is aware the service
is still alive. It also periodically (every 10 seconds) checks the state of all other
service threads to detect if deadlock has occurred. If deadlock has occurred the service
will indicate which thread didn't make progress and action can be taken (e.g. the service
refrains from updating its service-lease, causing the splice daemon to execute a failure
action). Most of the time this thread is asleep.

conflictResolver
================
This thread is responsible for resolving conflicts. If a conflict has been detected and
stored in the conflictQueue, the conflictResolver thread takes the conflict, checks
whether the conflict still exists, and if so, starts the procedure to resolve the conflict
(i.e., start to determine a new master, send out sample requests, etc).

statusThread
============
This thread is responsible for the sending status messages to other durability services.
These messages are periodically sent between durability services to inform each other
about their state (.e.,g INITIALIZING or TERMINATING).

Configuration parameters:

+  ``//OpenSplice/DurabilityService/Watchdog/Scheduling``

d_adminActionQueue
==================
The durability service maintains a queue to schedule timed action. The d_adminActionQueue
periodically checks (every 100 ms) if an action is scheduled.
Example actions are: electing a new master, detection of new local groups and deleting
historical data.

Configuration parameters:

+  ``//OpenSplice/DurabilityService/Network/Heartbeat/Scheduling``

AdminEventDispatcher
====================
Communication between the splice-daemon and durability service is managed by events. The
AdminEventDispatcher thread listens and acts upon these events. For example, the creation
of a new topic is noticed by the splice-daemon, which generates an event for the
durability service, which schedules an action on to request historical data for the topic.

groupCreationThread
===================
The groupCreationThread is responsible for the creation of groups that exist in other
federations. When a durability service receives a newGroup message from another
federation, it must create the group locally in order to acquire data for it.
Creation of a group may fail in case a topic is not yet known. The thread will retry with
a 10ms interval.

sampleRequestHandler
====================
This thread is responsible for the handling of sampleRequests. When a durability
service receives a d_sampleRequest message (see the sampleRequestListener thread) it
will not immediately answer the request, but wait some time until the time to combine
requests has been expired (see
//OpenSplice/DurabilityService/Network/Alignment/RequestCombinePeriod). When this time
has expired the sampleRequestHandler will answer the request by collecting the requested
data and sending the data as d_sampleChain messages to the requestor.

Configuration parameters:

+  ``//OpenSplice/DurabilityService/Network/Alignment/AlignerScheduling``

resendQueue
===========
This thread is responsible for injection of message in the group after it has been rejected
before. When a durability service has received historical data from another fellow,
historical data is injected in the group (see d_sampleChain). Injection of historical data
can be rejected, e.g., when a resource limits are being used. When this happens, a new attempt
to inject the data is scheduled overusing the resendQueue thread. This thread will try to
deliver the data 1s later.

Configuration parameters:

+  ``//OpenSplice/DurabilityService/Network/Alignment/AligneeScheduling``

masterMonitor
=============
The masterMonitor is the thread that handles the selection of a new master. This thread is
invoked when the conflict resolver detects that a master conflict has occurred. The
masterMonitor is responsible for collecting master proposals from other fellows and sending
out proposals to other fellows.

groupLocalListenerActionQueue
=============================
This thread is used to handle historical data requests from specific readers, and to handle
delayed alignment (see //OpenSplice/DurabilityService/NameSpaces/Policy[@delayedAlignment])

d_groupsRequest
===============
The d_groupsRequest thread is responsible for processing incoming d_groupsRequest messages
from other fellows. When a durability service receives a message from a fellow, the durability
service will send information about its groups to the requestor by means of d_newGroup messages.
This thread collects group information, packs it in d_newGroup messages and send them to the
requestor. This thread will only do something when a d_groupsRequest has been received from a
fellow. Most of the time it will sleep.

d_nameSpaces
============
This thread is responsible for processing incoming d_nameSpaces messages from other fellows.
Durability services send each other their namespaces state so that they can detect potential
conflicts. The d_nameSpaces thread processes and administrates every incoming d_nameSpace. When
a conflict is detected, the conflict is scheduled which may cause the conflictResolver thread
to kick in.

d_nameSpacesRequest
===================
The d_nameSpacesRequest thread is responsible for processing incoming d_nameSpacesRequest
messages from other fellows. A durability service can request the namespaces form a fellow by
sending a d_nameSpacesRequest message to the fellow. Whenever a durability service receives a
d_nameSpacesRequest messages it will respond by sending its set of namespaces to the fellow.
The thread handles incoming d_nameSpacesRequest messages. As a side effect new fellows can be
discovered if a nameSpacesRequest is received from an unknown fellow.

d_status
========
The d_status thread is responsible for processing incoming d_status messages from other fellows.
Durability services periodically send each other status information (see the statusThread).
NOTE: in earlier versions missing d_status messages could lead to the conclusion that a fellows
has been removed. In recent versions this mechanism has been replaced so that the durability
service slaves itself to the liveness of remote federations based on heartbeats
(see thread dcpsHeartbeatListener). Effectivily, the d_status message is not used anymore to
verify liveliness of remote federations, it is only used to transfer the durability state of
a remote federation.

d_newGroup
==========
The d_newGroup thread is responsible for handling incoming d_newGroup messages from other fellows.
Durability services inform each other about groups in the namespaces. They do that by sending
d_newGroup messages to each other (see also thread d_groupsRequest). The d_newGroup thread is
responsible for handling incoming groups.

d_sampleChain
=============
The d_sampleChain thread handles incoming d_sampleChain messages from other fellows. When a
durability service answers an d_sampleRequest, it must collect the requested data and send it
to the requestor. The collected data is packed in d_sampleChain messages. The d_sampleChain thread
handles incoming d_sampleChain messages and applies the configured merge policy for the data.
For example, in case of a MERGE it injects all the received data in the local group and delivers
the data to the available readers.

Configuration parameters:

+  ``//OpenSplice/DurabilityService/Network/Alignment/AligneeScheduling``

d_sampleRequest
===============
The d_sampleRequest thread is responsible for handling incoming d_sampleRequest messages from
other fellows. A durability service can request historical data from a fellow by sending a
d_sampleRequest message. The d_sampleRequest thread is used to process d_sampleRequest messages.
Because d_sampleRequest messages are not handled immediately, they are stored in a list and handled
later on (see thread sampleRequestHandler).

d_deleteData
============
The d_deleteData thread is responsible for handling incoming d_deleteData messages from other fellows.
An application can call delete_historical_data(). This causes all historical data up till now to be
deleted. To propagate deletion of historical data to all available durability services in the system,
durability services send each other a d_deleteData message. The d_deleteData thread handles incoming
d_deleteData messages and takes care that the relevant data is deleted. This thread will only be active
after delete_historical_data() is called.

dcpsHeartbeatListener
=====================
The dcpsHeartbeatListener is responsible for the liveliness detection of remote federations. This
thread listens to DCPSHeartbeat messages that are sent by federation. It is used to detect new
federations or federations that disconnect. This thread will only do something when there is a change
in federation topology. Most of the time it will be asleep.

d_capability
============
The thread is responsible for processing d_cability messages from other fellows. As soon as a durability
service detects a fellow it will send its list of capabilities to the fellow. The fellow can use this
list to find what functionality is supported by the durability service. Similarly, the durability
service can receive capabilities from the fellow. This thread is used to process the capabilities sent by
a fellow. This thread will only do something when a fellow is detected.                                                                                                                                                                                                                                                                                                                                                         |                                                                    |

remoteReader
============
The remoteReader thread is responsible for the detection of remote readers on other federations. The DDSI
service performs discovery and reader-writing matching. This is an asynchronous mechanism. When a
durability service (say A) receives a request from a fellow durability service (say B) and DDSI is used
as networking service, then A cannot be sure that DDSI has already detected the reader on B that should
receive the answer to the request. To ensure that durability services will only answer if all relevant
remote readers have been detected, the remoteReader thread keeps track of the readers that have been
discovered by ddsi. Only when all relevant readers have been discovered durability services are allowed
to answer requests. This prevents DDSI from dropping messages destined for readers that have not been
discovered yet.

persistentDataListener
======================
The persistentDataListenerThread is responsible for persisting durable data. When a durability service
retrieves persistent data, the data is stored in a queue. The persistentDataListener thread retrieves the
data from the queue and stores it in the persistent store. For large data sets persisting the data can take
quite some time, depending mostly on the performance of the disk.

Note this thread is only created when persistency is enabled
(//OpenSplice/DurabilityService/Persistent/StoreDirectory has a value set):

Configuration parameters:

+  ``//OpenSplice/DurabilityService/Persistent/Scheduling``

historicalDataRequestHandler
============================
This thread is responsible for handling incoming historicalDataRequest messages from durability clients. In
case an application does not have the resources to run a durability service but still wants to acquire
historical data it can configure a client. The client sends HistoricalDataRequest messages to the durability
service. These messages are handled by the historicalDataRequestHandler thread.

Note this thread is only created when client durability is enabled
(//OpenSplice/DurabilityService/ClientDurability element exists)

durabilityStateListener
=======================
This thread is responsible for handling incoming durabilityStateRequest messages from durability clients.

Note this thread is only created when client durability is enabled
(//OpenSplice/DurabilityService/ClientDurability element exists)

.. EoF


.. |caution| image:: ./images/icon-caution.*
            :height: 6mm
.. |info|   image:: ./images/icon-info.*
            :height: 6mm
.. |windows| image:: ./images/icon-windows.*
            :height: 6mm
.. |unix| image:: ./images/icon-unix.*
            :height: 6mm
.. |linux| image:: ./images/icon-linux.*
            :height: 6mm
.. |c| image:: ./images/icon-c.*
            :height: 6mm
.. |cpp| image:: ./images/icon-cpp.*
            :height: 6mm
.. |csharp| image:: ./images/icon-csharp.*
            :height: 6mm
.. |java| image:: ./images/icon-java.*
            :height: 6mm


