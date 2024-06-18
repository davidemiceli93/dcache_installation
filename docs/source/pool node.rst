.. _installation on Pool Node:

Installation on Pool Node
======================

The entire procedure was performed logged in as user ``root``.

We choose the following configuration: 

.. code-block:: bash

   OS: Almalinux9
   DCache version: 10.0
   PostgreSQL v.14
   zookeeper v 3.8 (installation NOT embedded to dCache)

First Checks
----------------
Once logged in you should Disable selinux on all nodes by opening the /etc/selinux/config file and set the SELINUX mod to disabled.

.. code-block:: bash

   vi /etc/selinux/config

.. code-block:: bash

   #
   #    grubby --update-kernel ALL --remove-args selinux
   #
   SELINUX=disabled
   # SELINUXTYPE= can take one of these three values:
   #     targeted - Targeted processes are protected,
   #     minimum - Modification of targeted policy. Only selected processes are protected.
   #     mls - Multi Level Security protection.
   SELINUXTYPE=targeted

Download prerequisites: 

.. code-block:: bash 

   yum install java-11-openjdk-headless httpd-tools nfs-utils wget 

Download dCache
----------------
Download dCache packages from website.

.. code-block:: bash

   rpm -ivh https://www.dcache.org/old/downloads/1.9/repo/10.0/dcache-10.0.3-1.noarch.rpm


