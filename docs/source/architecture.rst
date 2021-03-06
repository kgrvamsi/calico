.. # Copyright (c) Metaswitch Networks 2015. All rights reserved.
   #
   #    Licensed under the Apache License, Version 2.0 (the "License"); you may
   #    not use this file except in compliance with the License. You may obtain
   #    a copy of the License at
   #
   #         http://www.apache.org/licenses/LICENSE-2.0
   #
   #    Unless required by applicable law or agreed to in writing, software
   #    distributed under the License is distributed on an "AS IS" BASIS,
   #    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
   #    implied. See the License for the specific language governing
   #    permissions and limitations under the License.

Calico Architecture
===================

This document discusses the various pieces of the Calico etcd-based
architecture, with a focus on what specific role each component plays in the
Calico network. This does not discuss the Calico etcd data model, which also
acts as the primary API into the Calico network: for more on that, see
:doc:`etcd-data-model`.

.. _etcd: https://github.com/coreos/etcd

Components
----------

Calico is made up of the following interdependent components:

- :ref:`calico-felix-component`, the primary Calico agent that runs on each
  machine that hosts endpoints.
- :ref:`calico-orchestrator-plugin`, orchestrator-specific code that tightly
  integrates Calico into that orchestrator.
- :ref:`calico-etcd-component`, the data store.
- :ref:`calico-bgp-component`, a BGP client that distributes routing
  information.
- :ref:`calico-bgp-rr-component`, an optional BGP route reflector for higher
  scale.

The following sections break down each component in more detail.


.. _calico-felix-component:

Felix
-----

Felix is a daemon that runs on every machine that provides endpoints: in most
cases that means on nodes that host containers or VMs. It is responsible for
programming routes and ACLs, and anything else required on the host, in order
to provide the desired connectivity for the endpoints on that host.

Depending on the specific orchestrator environment, Felix is responsible for
some or all of the following tasks:

Interface Management
~~~~~~~~~~~~~~~~~~~~

Felix programs some information about interfaces into the kernel in order to
get the kernel to correctly handle the traffic emitted by that endpoint. In
particular, it may enable `Proxy ARP`_, `Proxy NDP`_, and IP forwarding for
interfaces that it manages.

It also monitors for interfaces to appear and disappear so that it can ensure
that the programming for those interfaces is applied at the appropriate time.

.. _Proxy ARP: http://en.wikipedia.org/wiki/Proxy_ARP
.. _Proxy NDP: http://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol

Route Programming
~~~~~~~~~~~~~~~~~

Felix is responsible for programming routes to the endpoints on its host into
the Linux kernel FIB. This ensures that packets destined for those endpoints
that arrive on at the host are forwarded accordingly.

ACL Programming
~~~~~~~~~~~~~~~

Felix is also responsible for programming ACLs into the Linux kernel. These
ACLs are used to ensure that only valid traffic can be sent between
endpoints, and ensure that endpoints are not capable of circumventing
Calico's security measures. For more on this, see :doc:`security-model`.

State Reporting
~~~~~~~~~~~~~~~

Felix is responsible for providing data about the health of the network. In
particular, it reports errors and problems with configuring its host. This data
is written into etcd, to make it visible to other components and operators of
the network.


.. _calico-orchestrator-plugin:

Orchestrator Plugin
-------------------

Unlike Felix there is no single 'orchestrator plugin': instead, there are
separate plugins for each major cloud orchestration platform (e.g. OpenStack,
Kubernetes). The purpose of these plugins is to bind Calico more tightly into
the orchestrator, allowing users to manage the Calico network just as they'd
manage network tools that were built in to the orchestrator.

A good example of an orchestrator plugin is the Calico Neutron ML2 mechanism
driver. This component integrates with Neutron's ML2 plugin, and allows users
to configure the Calico network by making Neutron API calls. This provides
seamless integration with Neutron.

The orchestrator plugin is repsonsible for the following tasks:

API Translation
~~~~~~~~~~~~~~~

The orchestrator will inevitably have its own set of APIs for managing
networks. The orchestrator plugin's primary job is to translate those APIs into
the Calico etcd data model (see :doc:`etcd-data-model`) to allow Calico to
perform the appropriate network programming.

Some of this translation will be very simple, other bits may be more complex in
order to render a single complex operation (e.g. live migration) into the
series of simpler operations the rest of the Calico network expects.

Feedback
~~~~~~~~

If necessary, the orchestrator plugin will provide feedback from the Calico
network into the orchestrator. Examples include: providing information about
Felix liveness; marking certain endpoints as failed if network setup failed.


.. _calico-etcd-component:

etcd
----

etcd is a distributed key-value store that has a focus on consistency. 
Calico uses etcd to provide the communication between components and as a
consistent data store, which ensures Calico can always build an accurate 
network.

Depending on the orchestrator plug-in, etcd may either be the master data store
or a lightweight mirror of a separate data store.  For example, in an
OpenStack deployment, the OpenStack database is considered the "source of
truth" and etcd is used to mirror information about the network to the other
Calico components.

The etcd component is distributed across the entire deployment. It is divided
into two groups of machines: the core cluster, and the proxies.

For small deployments the core cluster can be an etcd cluster of one node
(which would typically be co-located with the :ref:`calico-orchestrator-plugin`
component). This deployment model is simple but provides no redundancy for
etcd - in the case of etcd failure the :ref:`calico-orchestrator-plugin` would
have to rebuild the database which, as noted for OpenStack, will simply require
that the plug-in resynchonizes state to etcd from the OpenStack database.

In larger deployments the core cluster can be scaled up, as per the 
`etcd admin guide`_.

Additionally, on each machine that hosts either a
:ref:`calico-felix-component` or a :ref:`calico-orchestrator-plugin`, we run
an etcd proxy. This is reduces load on the core cluster and shields nodes from
the specifics of the etcd cluster. In the case where the etcd cluster has a
member on the same machine as a :ref:`calico-orchestrator-plugin`, we can
forgo the proxy on that machine.

etcd is responsible for performing all of the following tasks:

.. _etcd admin guide: https://github.com/coreos/etcd/blob/master/Documentation/admin_guide.md#optimal-cluster-size

Data Storage
~~~~~~~~~~~~

etcd stores the data for the Calico network in a distributed, consistent,
fault-tolerant manner (for cluster sizes of at least 3 etcd nodes). This set of
properties ensures that the Calico network is always in a known-good state,
while allowing for some number of the machines hosting etcd to fail or become unreachable.

This distributed storage of Calico data also improves the ability of the Calico
components to read from the database (which is their most common operation), as
they can distribute their reads around the cluster.

Communication
~~~~~~~~~~~~~

etcd is also used as a communication bus between components. We do this by
having the non-etcd components watch certain points in the keyspice to ensure
that they see any changes that have been made, allowing them to respond to
tohose changes in a timely manner. This allows the act of committing state
to the database to cause that state to be programmed into the network.


.. _calico-bgp-component:

BGP Client (BIRD)
-----------------

Calico deploys a BGP client on every node that also hosts a
:ref:`calico-felix-component`. The role of the BGP client is to read routing
state that :ref:`calico-felix-component` programs into the kernel and
distribute it around the data center.

In Calico, this BGP component is most commonly `BIRD`_, though any BGP client
that can draw routes from the kernel and distribute them is suitable in this
role.

The BGP client is responsible for performing the following tasks:

.. _BIRD: http://bird.network.cz/

Route Distribution
~~~~~~~~~~~~~~~~~~

When :ref:`calico-felix-component` inserts routes into the Linux kernel FIB,
the BGP client will pick them up and distribute them to the other nodes in the
deployment. This ensures that traffic is efficiently routed around the
deployment.


.. _calico-bgp-rr-component:

BGP Route Reflector (BIRD)
--------------------------

For larger deployments, simple BGP can become a limiting factor because it
requires every BGP client to be connected to every other BGP client in a mesh
topology. This requires an increasing number of connections that rapidly
become tricky to maintain, due to the N^2 nature of the increase.

For that reason, in larger deployments Calico will deploy a BGP route
reflector. This component, commonly used in the Internet, acts as a central
point to which the BGP clients connect, preventing them from needing to talk to
every single BGP client in the cluster.

For redundancy, multiple BGP route reflectors can be deployed seamlessly. The
route reflectors are purely involved in the control of the network: no endpoint
data passes through them.

In Calico, this BGP component is also most commonly `BIRD`_, configured as a
route reflector rather than as a standard BGP client.

The BGP route reflector is responsible for the following tasks:

Centralised Route Distribution
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When :ref:`calico-bgp-component` advertises routes from its FIB to the route
reflector, the route reflector advertises those routes out to the other nodes
in the deployment.
