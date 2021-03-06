=========
Multinode
=========

Overview
========

In order to drive towards a production-ready OpenStack solution, our
goal is to provide containerized, yet stable `persistent
volumes <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>`_
that Kubernetes can use to schedule applications that require state,
such as MariaDB (Galera). Although we assume that the project should
provide a "batteries included" approach towards persistent storage, we
want to allow operators to define their own solution as well. Examples
of this work will be documented in another section, however evidence of
this is found throughout the project. If you have any questions or
comments, please create an `issue
<https://bugs.launchpad.net/openstack-helm>`_.

.. note::
  Please see the supported application versions outlined in the
  `source variable file <https://github.com/openstack/openstack-helm-infra/blob/master/playbooks/vars.yaml>`_.

Other versions and considerations (such as other CNI SDN providers),
config map data, and value overrides will be included in other
documentation as we explore these options further.

The installation procedures below, will take an administrator from a new
``kubeadm`` installation to Openstack-Helm deployment.

.. warning:: Until the Ubuntu kernel shipped with 16.04 supports CephFS
   subvolume mounts by default the `HWE Kernel
   <../troubleshooting/ubuntu-hwe-kernel.rst>`__ is required to use CephFS.

Kubernetes Preparation
======================

You can use any Kubernetes deployment tool to bring up a working Kubernetes
cluster for use with OpenStack-Helm. For simplicity however we will describe
deployment using the OpenStack-Helm gate scripts to bring up a reference cluster
using KubeADM and Ansible.

OpenStack-Helm Infra KubeADM deployment
---------------------------------------

.. note::
   Throughout this guide the assumption is that the user is:
   ``ubuntu``. Because this user has to execute root level commands
   remotely to other nodes, it is advised to add the following lines
   to ``/etc/suders`` for each node:

   ``root    ALL=(ALL) NOPASSWD: ALL``

   ``ubuntu  ALL=(ALL) NOPASSWD: ALL``

On the master node install the latest versions of Git, CA Certs & Make if necessary

.. literalinclude:: ../../../tools/deployment/developer/common/000-install-packages.sh
    :language: shell
    :lines: 1,17-

On the worker nodes

.. code-block:: shell

    #!/bin/bash
    set -xe
    sudo apt-get update
    sudo apt-get install --no-install-recommends -y git


SSH-Key preparation
-------------------

Create an ssh-key on the master node, and add the public key to each node that
you intend to join the cluster.

.. note::
   1. To generate the key you can use ``ssh-keygen -t rsa``
   2. To copy the ssh key to each node, this can be accomplished with
      the ``ssh-copy-id`` command, for example: *ssh-copy-id
      ubuntu@192.168.122.178*
   3. Copy the key: ``sudo cp ~/.ssh/id_rsa /etc/openstack-helm/deploy-key.pem``
   4. Set correct ownership: ``sudo chown ubuntu
      /etc/openstack-helm/deploy-key.pem``

   Test this by ssh'ing to a node and then executing a command with
   'sudo'. Neither operation should require a password.

Clone the OpenStack-Helm Repos
------------------------------

Once the host has been configured the repos containing the OpenStack-Helm charts
should be cloned onto each node in the cluster:

.. code-block:: shell

    #!/bin/bash
    set -xe

    sudo chown -R ubuntu: /opt
    git clone https://git.openstack.org/openstack/openstack-helm-infra.git /opt/openstack-helm-infra
    git clone https://git.openstack.org/openstack/openstack-helm.git /opt/openstack-helm


Create an inventory file
------------------------

On the master node create an inventory file for the cluster:

.. note::
   node_one, node_two and node_three below are all worker nodes,
   children of the master node that the commands below are executed on.

.. code-block:: shell

    #!/bin/bash
    set -xe
    cat > /opt/openstack-helm-infra/tools/gate/devel/multinode-inventory.yaml <<EOF
    all:
      children:
        primary:
          hosts:
            node_one:
              ansible_port: 22
              ansible_host: $node_one_ip
              ansible_user: ubuntu
              ansible_ssh_private_key_file: /etc/openstack-helm/deploy-key.pem
              ansible_ssh_extra_args: -o StrictHostKeyChecking=no
        nodes:
          hosts:
            node_two:
              ansible_port: 22
              ansible_host: $node_two_ip
              ansible_user: ubuntu
              ansible_ssh_private_key_file: /etc/openstack-helm/deploy-key.pem
              ansible_ssh_extra_args: -o StrictHostKeyChecking=no
            node_three:
              ansible_port: 22
              ansible_host: $node_three_ip
              ansible_user: ubuntu
              ansible_ssh_private_key_file: /etc/openstack-helm/deploy-key.pem
              ansible_ssh_extra_args: -o StrictHostKeyChecking=no
    EOF

Create an environment file
--------------------------

On the master node create an environment file for the cluster:

.. code-block:: shell

    #!/bin/bash
    set -xe
    function net_default_iface {
     sudo ip -4 route list 0/0 | awk '{ print $5; exit }'
    }
    cat > /opt/openstack-helm-infra/tools/gate/devel/multinode-vars.yaml <<EOF
    kubernetes_network_default_device: $(net_default_iface)
    EOF

.. note::
  This installation, by default will use Google DNS servers, 8.8.8.8 or 8.8.4.4
  and updates resolv.conf. These DNS nameserver entries can be changed by
  updating file ``/openstack-helm-infra/tools/images/kubeadm-aio/assets/opt/playbooks/vars.yaml``
  under section ``external_dns_nameservers``. This change must be done on each
  node in your cluster.


Run the playbooks
-----------------

On the master node run the playbooks:

.. code-block:: shell

    #!/bin/bash
    set -xe
    cd /opt/openstack-helm-infra
    make dev-deploy setup-host multinode
    make dev-deploy k8s multinode

Deploy OpenStack-Helm
=====================

.. note::
  The following commands all assume that they are run from the
  ``/opt/openstack-helm`` directory.


Setup Clients on the host and assemble the charts
-------------------------------------------------

The OpenStack clients and Kubernetes RBAC rules, along with assembly of the
charts can be performed by running the following commands:

.. literalinclude:: ../../../tools/deployment/multinode/010-setup-client.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/010-setup-client.sh


Deploy the ingress controller
-----------------------------

.. literalinclude:: ../../../tools/deployment/multinode/020-ingress.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/020-ingress.sh


Deploy Ceph
-----------

.. note::
  The `./tools/deployment/multinode/kube-node-subnet.sh` script requires docker
  to run.

.. literalinclude:: ../../../tools/deployment/multinode/030-ceph.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/030-ceph.sh

Activate the openstack namespace to be able to use Ceph
-------------------------------------------------------

.. literalinclude:: ../../../tools/deployment/multinode/040-ceph-ns-activate.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/040-ceph-ns-activate.sh

Deploy MariaDB
--------------

.. literalinclude:: ../../../tools/deployment/multinode/050-mariadb.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/050-mariadb.sh

Deploy RabbitMQ
---------------

.. literalinclude:: ../../../tools/deployment/multinode/060-rabbitmq.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/060-rabbitmq.sh

Deploy Memcached
----------------

.. literalinclude:: ../../../tools/deployment/multinode/070-memcached.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/070-memcached.sh

Deploy Keystone
---------------

.. literalinclude:: ../../../tools/deployment/multinode/080-keystone.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/080-keystone.sh

Create Ceph endpoints and service account for use with keystone
---------------------------------------------------------------

.. literalinclude:: ../../../tools/deployment/multinode/090-ceph-radosgateway.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/090-ceph-radosgateway.sh

Deploy Glance
-------------

.. literalinclude:: ../../../tools/deployment/multinode/100-glance.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/100-glance.sh

Deploy Cinder
-------------

.. literalinclude:: ../../../tools/deployment/multinode/110-cinder.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/110-cinder.sh

Deploy OpenvSwitch
------------------

.. literalinclude:: ../../../tools/deployment/multinode/120-openvswitch.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/120-openvswitch.sh

Deploy Libvirt
--------------

.. literalinclude:: ../../../tools/deployment/multinode/130-libvirt.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/130-libvirt.sh

Deploy Compute Kit (Nova and Neutron)
-------------------------------------

.. literalinclude:: ../../../tools/deployment/multinode/140-compute-kit.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/140-compute-kit.sh

Deploy Heat
-----------

.. literalinclude:: ../../../tools/deployment/multinode/150-heat.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/150-heat.sh

Deploy Barbican
---------------

.. literalinclude:: ../../../tools/deployment/multinode/160-barbican.sh
    :language: shell
    :lines: 1,17-

Alternatively, this step can be performed by running the script directly:

.. code-block:: shell

  ./tools/deployment/multinode/160-barbican.sh
