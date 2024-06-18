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


Authentication and generation of x509 certificates
----------------

Now we will add to dCache the possiblity to authenticate using x509 certificates. In order to do that you will need the ``host`` certificate and key, the ``CA`` certificate and your own ``user`` certificate and key.

In this example the ``host`` certificate and key was given in the ``sms`` Frascati machine, the ``user`` certificate and key are the one issued by INFN and the ``CA`` certificate can be found in the website of the corresponding CA.

In order to make dCache recognize the host you should put the host key and certificate in the path ``/etc/grid-security`` and you have to change the permission settings:

.. code-block:: bash

   chown 644 hostcert.pem
   chown 400 hostkey.pem

Then, once you ahve the CA certificate you have to add it in the list of trusted CA which means, if your certificate is ``cacert.crt`` you need to:

.. code-block:: bash

   cp cacert.crt /etc/pki/ca-trust/source/anchors/
   update-ca-trust

Then, go in ``/etc/yum.repos.d`` and create the file ``EGI-trustanchors.repo`` with the following content:

.. code-block:: bash
   
   cat > /etc/yum.repos.d/EGI-trustanchors.repo <<EOF
   [EGI-trustanchors]
   name=EGI-trustanchors
   baseurl=http://repository.egi.eu/sw/production/cas/1/current/
   gpgkey=http://repository.egi.eu/sw/production/cas/1/GPG-KEY-EUGridPMA-RPM-3
   gpgcheck=1
   enabled=1
   EOF

Then you populate the certificates folder in ``/etc/grid-security`` with:

.. code-block:: bash

   yum install ca-policy-egi-core

Then:

.. code-block:: bash

   yum update
   sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
   sudo yum clean all 
   sudo yum makecache
   yum install fetch-crl
   fetch-crl

The command ``fetch-crl`` should be repeated at least once per day.

At this point you can upload your layout configuration file adding the following lines to the file:

.. code-block:: bash

   [doorsDomain-d25]
   [doorsDomain-d25/webdav]
    webdav.authn.protocol = https


Using VOMS proxy certificates
----------------

