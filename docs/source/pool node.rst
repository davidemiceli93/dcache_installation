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

Create dCache configuration
----------------

For details please have a look here: https://www.dcache.org/manuals/Book-10.0/install.shtml#creating-a-minimal-dcache-configuration

Update che dCache configuration file adding the layout to be used:

.. code-block:: bash

   touch /etc/dcache/dcache.conf
   cat > /etc/dcache/dcache.conf <<EOF
   dcache.layout = mylayout 
   dcache.systemd.strict=false
   EOF

Create the corresponding layout file in the standard dCache directory and fill it with your required dCache configuration. Since this is the pool node what you need is simply to connect the ``zookeeper`` server running in the head node and the ``DoorDomain``:

.. code-block:: bash

   touch /etc/dcache/layouts/mylayout.conf
   cat > /etc/dcache/layouts/mylayout.conf <<EOF
   dcache.enable.space-reservation = false
   dcache.zookeeper.connection = 192.168.71.129:2181
   ...
   <your configuration>
   EOF

Now you can create the pools. dCache will automatically update the layout file adding the pool domain information.

.. code-block:: bash

   dcache pool create /netapp/vol1_D25/pool pool1 d25-poolsDomain 
   dcache pool create /netapp/vol2_D25/pool pool2 d25-poolsDomain
   dcache pool create /netapp/vol3_D25/pool pool3 d25-poolsDomain
   dcache pool create /netapp/vol4_D25/pool pool4 d25-poolsDomain
   dcache pool create /netapp/vol5_D25/pool pool5 d25-poolsDomain


