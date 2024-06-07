

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

.. code-block:: console

   $ vi /etc/selinux/config

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

.. code-block:: console

   yum install java-11-openjdk-headless httpd-tools nfs-utils wget 

Download dCache
----------------
Download dCache packages from website.

.. code-block:: console

   rpm -ivh https://www.dcache.org/old/downloads/1.9/repo/10.0/dcache-10.0.3-1.noarch.rpm


Set up PostGRESQL database and Zookeeper
----------------

Update the system

.. code-block:: console

   dnf update -y

Install PostgreSQL:

.. code-block:: console

   rpm -Uvh https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   yum install -y postgresql14-server

Initialize PostGreSQL Database

.. code-block:: console

   /usr/pgsql-14/bin/postgresql-14-setup initdb

Allow local users to access PostgreSQL without requiring a password:

.. code-block:: console

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

.. code-block:: console
   
   systemctl enable postgresql-14 
   systemctl start postgresql-14


Create postgreSQL users and databases

.. code-block:: console
   
   createuser -U postgres --no-superuser --no-createrole --createdb --no-password dcache
   createdb -U dcache chimera
   createdb -U dcache spacemanager 
   createdb -U dcache pinmanager
   dcache database update

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

