

.. _installation on Head Node:

Installation on Head Node
======================

The entire procedure was performed logged in as user ``root``.

We choose the following configuration: 
.. toctree::
   OS: Almalinux9
   DCache version: 10.0
   PostgreSQL v.14
   zookeeper v 3.8 (installation NOT embedded to dCache)

First Checks
----------------
Once logged in you should Disable selinux on all nodes by opening the /etc/selinux/config file and set the SELINUX mod to disabled.

.. prompt:: bash $

   vi /etc/selinux/config

.. code-block:: console

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

.. prompt:: bash $

   yum install java-11-openjdk-headless httpd-tools nfs-utils wget 

Download dCache
----------------
Download dCache packages from website.

.. prompt:: bash $

   rpm -ivh https://www.dcache.org/old/downloads/1.9/repo/10.0/dcache-10.0.3-1.noarch.rpm


Set up PostGRESQL database and Zookeeper
----------------

Update the system

.. prompt:: bash $

   dnf update -y

Install PostgreSQL:

.. prompt:: bash $

   rpm -Uvh https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   yum install -y postgresql14-server

Initialize PostGreSQL Database

.. prompt:: bash $

   /usr/pgsql-14/bin/postgresql-14-setup initdb

Allow local users to access PostgreSQL without requiring a password:

.. prompt:: bash $

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

.. prompt:: bash $
   
   systemctl enable postgresql-14 
   systemctl start postgresql-14


Create postgreSQL users and databases

.. prompt:: bash $
   
   createuser -U postgres --no-superuser --no-createrole --createdb --no-password dcache
   createdb -U dcache chimera
   createdb -U dcache spacemanager 
   createdb -U dcache pinmanager
   dcache database update

If you want to run dCache without the zookeeper embedded you will need to follow these steps:

Create zookeeper user and give root permissions

.. prompt:: bash $
   
   useradd -m zookeeper 
   passwd zookeeper. (new passwd is dcache_datacenter)
   usermod -aG wheel zookeeper

Download and prepare zookeeper: (install zookeeper in /opt)

.. prompt:: bash $

   cd /opt
   curl -O https://dlcdn.apache.org/zookeeper/zookeeper-3.8.4/apache-zookeeper-3.8.4-bin.tar.gz
   tar -xvf apache-zookeeper-3.8.4-bin.tar.gz
   rm apache-zookeeper-3.8.4-bin.tar.gz
   sudo chown zookeeper:zookeeper apache-zookeeper-3.8.4-bin -R
   sudo ln -s apache-zookeeper-3.8.4-bin zookeeper
   sudo chown zookeeper:zookeeper zookeeper -h

Configure zookeeper:

.. prompt:: bash $

   sudo mkdir -p /data/zookeeper
   sudo chown zookeeper:zookeeper /data/zookeeper
   cd /opt/zookeeper/conf
   vi zookeeper.properties

Write:


.. prompt:: bash $

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

.. prompt:: bash $

   cd /data/zookeeper
   vi myid

Write the corresponding server ID:

.. prompt:: bash $
   1

Start Zookeeper process:

.. prompt:: bash $

   cd /opt/zookeeper/bin
   ./zkServer.sh start /opt/zookeeper/conf/zookeeper.properties
   ./zkCli.sh -server localhost:2181

Last command will open a shell interactive, try to type ‘ls /‘ and see if it works. To exit type ‘quit’


Create dCache configuration
----------------



Authentication and generation of x509 certificates
----------------

To use Lumache, first install it using pip:

.. code-block:: console

   (.venv) $ pip install lumache

Creating recipes
----------------

To retrieve a list of random ingredients,
you can use the ``lumache.get_random_ingredients()`` function:

.. autofunction:: lumache.get_random_ingredients

The ``kind`` parameter should be either ``"meat"``, ``"fish"``,
or ``"veggies"``. Otherwise, :py:func:`lumache.get_random_ingredients`
will raise an exception.

.. autoexception:: lumache.InvalidKindError

For example:

>>> import lumache
>>> lumache.get_random_ingredients()
['shells', 'gorgonzola', 'parsley']

