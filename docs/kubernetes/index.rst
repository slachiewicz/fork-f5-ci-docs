F5 Kubernetes Container Integration
===================================

Overview
--------

The F5 `Kubernetes`_ Container Integration consists of the `F5 Kubernetes BIG-IP Controller </products/connectors/k8s-bigip-ctlr/latest>`_ and the `F5 Application Service Proxy </products/asp/latest>`_ (ASP).

The |kctlr-long| configures a BIG-IP to expose applications in a `Kubernetes cluster`_ as BIG-IP virtual servers, serving North-South traffic.

The |asp| provides load balancing and telemetry for containerized applications, serving East-West traffic.

.. image:: /_static/media/kubernetes_solution.png
    :scale: 50 %
    :alt: F5 Container Solution for Kubernetes


General Prerequisites
---------------------

This documentation set assumes that you:

- already have a `Kubernetes cluster`_ running;
- are familiar with the `Kubernetes dashboard`_ and `kubectl`_ ;
- already have a BIG-IP :term:`device` licensed and provisioned for your requirements; [#bigipcaveat]_ and
- are familiar with BIG-IP Local Traffic Manager (LTM) concepts and ``tmsh`` commands. [#bigipcaveat]_

.. [#bigipcaveat] Not required for the |asp| and ASP controllers (|aspk|, |aspm|).


|asp|
-----

The |asp| (ASP) provides container-to-container load balancing, traffic visibility, and inline programmability for applications. Its light form factor allows for rapid deployment in datacenters and across cloud services. The ASP integrates with container environment management and orchestration systems and enables application delivery service automation. The ASP's `Kubernetes`_ integration, |aspk|, builds on Kubernetes' existing `network proxy <https://kubernetes.io/docs/admin/kube-proxy/>`_ functionality.

.. seealso:: `ASP product documentation </products/asp/latest/index.html>`_


|aspk-long|
-----------

The |aspk-long| -- |aspk| -- deploys the |asp|. It replaces the standard Kubernetes network proxy, or `kube-proxy`_. Like the |kctlr-long|, |aspk-long| watches the Kubernetes API; when it discovers Services containing the :ref:`ASP annotation <k8s-service-annotate>`, it launches a pod running the |aspk|, with the configurations specified in the annotation.

The ASP's ``bind_port`` and ``shared-listen`` `configuration options <tbd>`_ allow you to configure a single, shared ingress socket on ASP instances. The standard definitions for these options in Kubernetes, which are read by |aspk|, are ``bindport: 10000`` and ``shared-listen: true``.

The |asp| collects traffic statistics for the Services it load balances; these stats are either logged locally or sent to an external analytics application. You can set the location and type of the analytics application in the `stats </products/asp/latest/index.html#stats>`_ section of the :ref:`Service annotation <k8s-service-annotate>`.

.. todo:: add "Export ASP Stats to an analytics provider"


|kctlr-long|
------------

The |kctlr-long| is a docker container that runs on a `Kubernetes Pod`_. To launch the |kctlr| application in Kubernetes, just :ref:`create a Deployment <install-kctlr>`.

Once the |kctlr| pod is running, it watches the `Kubernetes API <https://kubernetes.io/docs/api/>`_ for special Kubernetes "F5 Resource" `ConfigMap`_ s. These ConfigMaps contain an F5 Resource JSON blob that tells |kctlr|:

- what `Kubernetes Service`_ we want it to manage, and
- how we want to configure the BIG-IP for that specific Service.

When the |kctlr| discovers new or updated :ref:`F5 Resource ConfigMaps <kctlr-create-vs>`, it dynamically applies the configurations to the BIG-IP.

.. important::

    * The |kctlr-long| cannot manage objects in the BIG-IP's ``/Common`` :term:`partition`.
    * Each |kctlr-long| deployment monitors one (1) Kubernetes `namespace`_ and manages objects in its assigned BIG-IP :term:`partition`. *If you create more than one (1)* :ref:`k8s-bigip-ctlr deployment <k8s-bigip-ctlr-deployment>`, *each must manage a different BIG-IP partition.*
    * Each F5 Resource defines a virtual server on the BIG-IP for one (1) port associated with one (1) `Service`_. *Create a separate* :ref:`F5 Resource ConfigMap <kctlr-create-vs>` *for each Service port you wish to expose to the BIG-IP.*

You can use the |kctlr-long| to :ref:`manage BIG-IP objects <kctlr-manage-bigip-objects>` directly, or :ref:`deploy iApps <kctlr-deploy-iapps>`.

Key Kubernetes Concepts
-----------------------

.. _k8s-f5-resources:

F5 Resource Properties
``````````````````````

The |kctlr-long| uses special 'F5 Resources' to identify what objects it should create on the BIG-IP. An F5 resource is defined as a JSON blob in a Kubernetes `ConfigMap`_.

The :ref:`F5 Resource JSON blob <f5-resource-blob>` must contain the following properties.

+---------------------+-------------------------------------------------------+
| Property            | Description                                           |
+=====================+=======================================================+
| f5type              | a ``label`` property defining the type of resource    |
|                     | to create on the BIG-IP;                              |
|                     |                                                       |
|                     | e.g., ``f5type: virtual-server``                      |
+---------------------+-------------------------------------------------------+
| schema              | identifies the schema |kctlr| uses to interpret the   |
|                     | encoded data                                          |
+---------------------+-------------------------------------------------------+
| data                | a JSON blob                                           |
|                     |                                                       |
| - frontend          | - a subset of ``data``; defines the virtualServer     |
|                     |   object                                              |
| - backend           | - a subset of ``data``; identifies the                |
|                     |   `Kubernetes Service`_ to proxy                      |
+---------------------+-------------------------------------------------------+

The frontend property defines how to expose a Service on the BIG-IP.
You can define the frontend using the standard `k8s-bigip-ctlr virtualServer parameters </products/connectors/k8s-bigip-ctlr/index.html#virtualserver>`_ or the `k8s-bigip-ctlr iApp parameters </products/connectors/k8s-bigip-ctlr/index.html#iapps>`_.

The frontend iApp configuration parameters include a set of customizable ``iappVariables`` parameters. These parameters must be custom-defined to correspond to fields in the iApp template you want to launch. In addition, you'll need to define the `iApp Pool Member Table </products/connectors/k8s-bigip-ctlr/index.html#iapp-pool-member-table>`_ that the iApp creates on the BIG-IP.

The backend property identifies the `Kubernetes Service`_ that makes up the server pool. You can also define health monitors for the virtual server and pool(s) in this section.


Using BIG-IP as an Edge Load Balancer in OpenShift Origin
---------------------------------------------------------

Red Hat's `OpenShift Origin`_ is a containerized application platform with a native Kubernetes integration. The |kctlr-long| enables use of a BIG-IP as an edge load balancer, proxying traffic from outside networks to pods inside an OpenShift cluster. OpenShift Origin uses a pod network defined by the `OpenShift SDN`_ .

There are a few additional prerequisites for working with OpenShift Origin clusters that do not apply to basic Kubernetes:

#. The |kctlr-long| needs an `OpenShift user account`_ with permission to access nodes, endpoints, services, and configmaps.
#. You'll need to use the `OpenShift Origin CLI`_, in addition to ``kubectl``, to execute OpenShift-specific commands.
#. To :ref:`integrate your BIG-IP into an OpenShift cluster <bigip-openshift-setup>`, you'll need to :ref:`assign an OpenShift overlay address to the BIG-IP <k8s-openshift-assign-ip>`.

Once you've added the BIG-IP to the OpenShift overlay network, it will have access to all pods in the cluster. You can then use the |kctlr| the same as you would in Kubernetes.

Monitors and Node Health
------------------------

When the |kctlr-long| runs with ``pool-member-type`` set to ``nodeport`` -- the default setting -- the |kctlr| will not be aware if a node is taken down. This means that all pool members on that node would remain active even if the node itself is unavailable. When using ``nodeport`` mode, it's important to configure a health monitor so the node is marked as unhealthy if it is rebooting or otherwise unavailable.

When the |kctlr-long| runs with ``pool-member-type`` set to ``cluster`` -- which integrates the BIG-IP into the cluster network -- the |kctlr| watches the NodeList in the Kubernetes API server; FDB entries are created/updated according to that list.


Related
-------

- `k8s-bigip-ctlr </products/connectors/k8s-bigip-ctlr/latest/>`_
- `asp </products/asp/latest>`_



.. _OpenShift Origin: https://www.openshift.org/
.. _OpenShift user account: https://docs.openshift.org/1.2/admin_guide/manage_users.html
.. _OpenShift Origin CLI: https://docs.openshift.org/1.2/cli_reference/index.html
.. _OpenShift SDN: https://docs.openshift.org/latest/architecture/additional_concepts/sdn.html


