

.. _installation on Head Node:

Installation on Head Node
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


Set up PostGRESQL database and Zookeeper
----------------

Update the system

.. code-block:: bash

   dnf update -y

Install PostgreSQL:

.. code-block:: bash

   rpm -Uvh https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   yum install -y postgresql14-server

Initialize PostGreSQL Database

.. code-block:: bash

   /usr/pgsql-14/bin/postgresql-14-setup initdb

Allow local users to access PostgreSQL without requiring a password:

.. code-block:: bash

   cat > /var/lib/pgsql/14/data/pg_hba.conf <<EOF
   # database on headnode
   local   all             all                                     trust
   host    all             all             127.0.0.1/32            trust
   host    all             all             ::1/128                 trust
   # database on dedicated dbnode
   #host   chimera         dcache          192.0.2.123/32          md5
   #host   spacemanager    dcache          192.0.2.123/32          md5
   #host   pinmanager      dcache          192.0.2.123/32          md5
   #host   srm             dcache          192.0.2.123/32          md5
   EOF

Enable and start postgresql-14 service

.. code-block:: bash
   
   systemctl enable postgresql-14 
   systemctl start postgresql-14


Create postgreSQL users and databases

.. code-block:: bash
   
   createuser -U postgres --no-superuser --no-createrole --createdb --no-password dcache
   createdb -U dcache chimera
   createdb -U dcache spacemanager 
   createdb -U dcache pinmanager
   dcache database update

If you want to run dCache without the zookeeper embedded you will need to follow these steps:

Create zookeeper user and give root permissions

.. code-block:: bash
   
   useradd -m zookeeper 
   passwd zookeeper. (new passwd is dcache_datacenter)
   usermod -aG wheel zookeeper

Download and prepare zookeeper: (install zookeeper in /opt)

.. code-block:: bash

   cd /opt
   curl -O https://dlcdn.apache.org/zookeeper/zookeeper-3.8.4/apache-zookeeper-3.8.4-bin.tar.gz
   tar -xvf apache-zookeeper-3.8.4-bin.tar.gz
   rm apache-zookeeper-3.8.4-bin.tar.gz
   sudo chown zookeeper:zookeeper apache-zookeeper-3.8.4-bin -R
   sudo ln -s apache-zookeeper-3.8.4-bin zookeeper
   sudo chown zookeeper:zookeeper zookeeper -h

Configure zookeeper:

.. code-block:: bash

   sudo mkdir -p /data/zookeeper
   sudo chown zookeeper:zookeeper /data/zookeeper
   cd /opt/zookeeper/conf
   vi zookeeper.properties

Write:


.. code-block:: bash

   # measured in milliseconds. It is used to regulate heartbeats, and timeouts
   tickTime=2000
 
   # Amount of time, in ticks, to allow followers to connect and sync to a leader
   initLimit=10
 
   # Amount of time, in ticks, to allow followers to sync with ZooKeeper.
   # If followers fall too far behind a leader, they will be dropped.
   syncLimit=5
 
   # the directory where the snapshot is stored.
   dataDir=/data/zookeeper
 
   # the port at which the clients will connect
   clientPort=2181
 
   # disable the per-ip limit on the number of connections since this is a non-production config
   maxClientCnxns=100
 
   # Disable the adminserver by default to avoid port conflicts.
   # Set the port to something non-conflicting if choosing to enable this
   admin.enableServer=false
   # admin.serverPort=8080
 
   # Cluster hosts setting
   server.1=localhost:2888:3888
   #server.1=zookeeper-server-01:2888:3888
   #server.2=zookeeper-server-02:2888:3888
   #server.3=zookeeper-server-03:2888:3888

Then:

.. code-block:: bash

   cd /data/zookeeper
   vi myid

Write the corresponding server ID:

.. code-block:: bash

   1

Start Zookeeper process:

.. code-block:: bash

   cd /opt/zookeeper/bin
   ./zkServer.sh start /opt/zookeeper/conf/zookeeper.properties
   ./zkCli.sh -server localhost:2181

Last command will open a shell interactive, try to type ‘ls /‘ and see if it works. To exit type ‘quit’


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

Create the corresponding layout file in the standard dCache directory and fill it with your required dCache configuration:

.. code-block:: bash

   touch /etc/dcache/layouts/mylayout.conf
   cat > /etc/dcache/layouts/mylayout.conf <<EOF
   <your configuration>
   EOF

Update the database:

.. code-block:: bash
   
   dcache database update

Now you have to set up the configuration of the authentication handled by ``gplazma``. Now we set up a configuration which enable the autentication via username-password and we give the corresponding permissions to the created users.
First, we create the configuration file for ``gplazma``:

.. code-block:: bash
   
   cat > /etc/dcache/gplazma.conf <<EOF
   auth    optional  htpasswd
   map     optional  multimap
   account  requisite   banfile
   session requisite authzdb
   EOF

Then we create a password file, adding two users (admin and tester):

.. code-block:: bash
   
   touch /etc/dcache/htpasswd
   htpasswd -bm /etc/dcache/htpasswd tester TooManySecrets
   htpasswd -bm /etc/dcache/htpasswd admin dickerelch

Now we assing uids and gids to these users and we tell them to dCache:

.. code-block:: bash

   touch /etc/dcache/multi-mapfile
   cat > /etc/dcache/multi-mapfile <<EOF
   username:tester uid:1000 gid:1000,true
   username:admin uid:0 gid:0,true
   EOF

We also create a banfile:

.. code-block:: bash

   touch /etc/dcache/ban.conf

and we give the authorization to the users for reading/writing the database folders:

.. code-block:: bash

   mkdir -p /etc/grid-security
   touch /etc/grid-security/storage-authzdb
   cat > /etc/grid-security/storage-authzdb <<EOF
   version 2.1

   authorize tester read-write 1000 1000 /home/tester /
   authorize admin read-write 0 0 / /
   EOF
   
Now we can try to access the database typing simply:

.. code-block:: bash

   chimera

this will open the chimera command line.

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

At this point you can upload your authentication settings in dCache and map the user to the username tester:

.. code-block:: bash

   cat > /etc/dcache/gplazma.conf <<EOF
   auth    optional  htpasswd
   auth    optional  x509
   map     optional  multimap
   session requisite authzdb
   EOF

   cat > /etc/dcache/multi-mapfile <<EOF

   "dn:/DC=org/DC=terena/DC=tcs/C=IT/O=Istituto Nazionale di Fisica Nucleare/CN=Davide Miceli miceli@infn.it" username:tester uid:1000 gid:1000,true

   username:tester uid:1000 gid:1000
   username:admin uid:0 gid:0,true

Using VOMS proxy certificates
----------------



