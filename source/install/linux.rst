===============================================================
Installing Lizmap Web Client on Linux Debian or Ubuntu
===============================================================

Generic Server Configuration
===============================================================

This documentation provides an example for configuring a server with the Debian or Ubuntu Server distribution. We assume you have base systrem installed and updated.

.. warning:: This page does not describe how to secure your Apache server.

Configure Locales
--------------------------------------------------------------

For simplicity, it is interesting to configure the server with UTF-8 default encoding.

.. code-block:: bash

   # configure locales
   locale-gen fr_FR.UTF-8 #replace fr with your language
   dpkg-reconfigure locales
   # define your timezone [useful for logs]
   dpkg-reconfigure tzdata
   apt-get install ntp ntpdate

.. note:: It is also necessary configure the other software so that they are using this default encoding if this is not the case.

Installing necessary packages
--------------------------------------------------------------

.. warning:: Lizmap web client is based on Jelix 1.6. You must install at least the **5.4** version of PHP. The **dom**, **simplexml**, **pcre**, **session**, **tokenizer** and **spl** extensions are required (they are generally turned on in a standard PHP 5.4 installation)

.. code-block:: bash

   sudo su # only necessary if you are not logged in as root
   apt-get update # update package lists
   apt-get install xauth htop curl apache2 libapache2-mod-fcgid libapache2-mod-php5 php5-gd php5-sqlite php5-curl python-simplejson python-software-properties

.. todo:: Check whether the following packages should be installed: php5 php5-cgi

.. todo:: Check this: still necessary?
   a2dismod php5
   a2enmod actions
   a2enmod fcgid
   a2enmod ssl
   a2enmod rewrite
   a2enmod headers
   a2enmod deflate
   service apache2 restart # restart Apache server

Then go to the *www* default Apache directory (modify as needed).

php5 configuration
-----------------------

.. todo:: Check this: still necessary?

In this example, we use Apache mpm-worker. So we must manually configure the activation of php5.

.. code-block:: bash

   cat > /etc/apache2/conf.d/php.conf << EOF
   <Directory /usr/share>
     AddHandler fcgid-script .php
     FCGIWrapper /usr/lib/cgi-bin/php5 .php
     Options ExecCGI FollowSymlinks Indexes
   </Directory>

   <Files ~ (\.php)>
     AddHandler fcgid-script .php
     FCGIWrapper /usr/lib/cgi-bin/php5 .php
     Options +ExecCGI
     allow from all
   </Files>
   EOF

.. note:: In later versions of apache, the config for ``php-cgi`` is in ``/etc/apache2/conf-available/php.conf``. Copy the text above, then::

   a2enconf php

mpm-worker configuration
-----------------------------

.. todo:: Check this: still necessary?

We modify the Apache configuration file to adapt the options to mpm_worker server configuration.

.. code-block:: bash

   # configuring worker
   nano /etc/apache2/apache2.conf # aller au worker et mettre par exemple
   <IfModule mpm_worker_module>
     StartServers       4
     MinSpareThreads    25
     MaxSpareThreads    100
     ThreadLimit          64
     ThreadsPerChild      25
     MaxClients        150
     MaxRequestsPerChild   0
   </IfModule>

mod_fcgid configuration
---------------------------

QGIS Server runs fcgi mode. We must therefore configure the Apache mod_fcgid to suit to the server capabilities.

.. code-block:: bash

   # Open the mod_fcgid configuration file
   nano /etc/apache2/mods-enabled/fcgid.conf
   # Paste the following content and adapt it
   <IfModule mod_fcgid.c>
     AddHandler    fcgid-script .fcgi
     FcgidConnectTimeout 300
     FcgidIOTimeout 300
     FcgidMaxProcessesPerClass 50
     FcgidMinProcessesPerClass 20
     FcgidMaxRequestsPerProcess 500
     IdleTimeout   300
     BusyTimeout   300
   </IfModule>

Setting the compression
-------------------------------

.. code-block:: bash

   nano /etc/apache2/conf.d/mod_deflate.conf # y ajouter
   <Location />
           # Insert filter
           SetOutputFilter DEFLATE
           # Netscape 4.x encounters some problems ...
           BrowserMatch ^Mozilla/4 gzip-only-text/html
           # Netscape 4.06-4.08 encounter even more problems
           BrowserMatch ^Mozilla/4\.0[678] no-gzip
           # MSIE pretends it is Netscape, but all is well
           BrowserMatch \bMSIE !no-gzip !gzip-only-text/html
           # Do not compress images
           SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png)$ no-gzip dont-vary
           # Ensure that proxy servers deliver the right content
           Header append Vary User-Agent env=!dont-vary
   </Location>

Enable geolocation
-------------------

The automatic geolocation provided by Lizmap relies on Google services. To enable it, your webGIS must be placed under a secure protocol, like HTTPS.See https://sites.google.com/a/chromium.org/dev/Home/chromium-security/deprecating-powerful-features-on-insecure-origins for more details.

Restart Apache
------------------

You must restart the Apache server to validate the configuration.

.. code-block:: bash

   service apache2 restart

Create directories for data
============================================

.. code-block:: bash

   mkdir /home/data
   mkdir /home/data/cache/
  # optional
   mkdir /home/data/ftp
   mkdir /home/data/ftp/template/
   mkdir /home/data/ftp/template/qgis

Spatial DBMS: PostgreSQL
============================================

.. note:: This section is optional

PostgreSQL and PostGIS can be very useful to manage spatial data centralized manner on the server.

Install
-------------

.. code-block:: bash

   # Install packages
   apt-get install postgresql postgresql-contrib postgis pgtune php5-pgsql

   # A cluster is created in order to specify the storage directory
   mkdir /home/data/postgresql
   service postgresql stop
   pg_dropcluster --stop 9.5 main
   chown postgres:postgres /home/data/postgresql
   pg_createcluster 9.5 main -d /home/data/postgresql --locale fr_FR.UTF8 -p 5678 --start

   # Creating a "superuser" user
   su - postgres
   createuser myuser --superuser
   # Modifying passwords
   psql
   ALTER USER postgres WITH ENCRYPTED PASSWORD '************';
   ALTER USER myuser WITH ENCRYPTED PASSWORD '************';
   \q
   exit

Adapting the PostgreSQL configuration
----------------------------------------------

We will use pgtune, an utility program that can automatically generate a PostgreSQL configuration file adapted to the properties of the server (memory, processors, etc.)

.. code-block:: bash

   # PostgreSQL Tuning with pgtune
   pgtune -i /etc/postgresql/9.5/main/postgresql.conf -o /etc/postgresql/9.5/main/postgresql.conf.pgtune --type Web
   cp /etc/postgresql/9.5/main/postgresql.conf /etc/postgresql/9.5/main/postgresql.conf.backup
   cp /etc/postgresql/9.5/main/postgresql.conf.pgtune /etc/postgresql/9.5/main/postgresql.conf
   nano /etc/postgresql/9.5/main/postgresql.conf
   # Restart to check any problems
   service postgresql restart
   # If error messages, increase the linux kernel configuration variables
   echo "kernel.shmall = 4294967296" >> /etc/sysctl.conf # to increas shred buffer param in kernel
   echo "kernel.shmmax = 4294967296" >> /etc/sysctl.conf
   echo 4294967296 > /proc/sys/kernel/shmall
   echo 4294967296 > /proc/sys/kernel/shmmax
   sysctl -a | sort | grep shm
   # Restart PostgreSQL
   service postgresql restart

FTP Server: pure-ftpd
=======================

.. note:: This section is optional

Install
---------------

.. code-block:: bash

   apt-get install pure-ftpd pure-ftpd-common

Configure
---------------

.. code-block:: bash

   # Creating an empty shell for users
   ln /bin/false /bin/ftponly
   # Configuring FTP server
   echo "/bin/ftponly" >> /etc/shells
   # Each user is locked in his home
   echo "yes" > /etc/pure-ftpd/conf/ChrootEveryone
   # Allow to use secure FTP over SSL
   echo "1" > /etc/pure-ftpd/conf/TLS
   # Configure the properties of directories and files created by users
   echo "133 022" > /etc/pure-ftpd/conf/Umask
   # The port range for passive mode (opening outwards)
   echo "5400 5600" > /etc/pure-ftpd/conf/PassivePortRange
   # Creating an SSL certificate for FTP
   openssl req -x509 -nodes -newkey rsa:1024 -keyout /etc/ssl/private/pure-ftpd.pem -out /etc/ssl/private/pure-ftpd.pem
   chmod 400 /etc/ssl/private/pure-ftpd.pem
   # Restart FTP server
   service pure-ftpd restart

Creating a user account
--------------------------------

.. code-block:: bash

   # Creating a user accountr
   MYUSER=demo
   useradd -g client -d /home/data/ftp/$MYUSER -s /bin/ftponly -m $MYUSER -k /home/data/ftp/template/
   passwd $MYUSER
   # Fix the user's FTP root
   chmod a-w /home/data/ftp/$MYUSER
   # Creating empty directories that will be the future Lizmap Web Client directories
   mkdir /home/data/ftp/$MYUSER/qgis/rep1 && chown $MYUSER:client /home/data/ftp/$MYUSER/qgis/rep1
   mkdir /home/data/ftp/$MYUSER/qgis/rep2 && chown $MYUSER:client /home/data/ftp/$MYUSER/qgis/rep2
   mkdir /home/data/ftp/$MYUSER/qgis/rep3 && chown $MYUSER:client /home/data/ftp/$MYUSER/qgis/rep3
   mkdir /home/data/ftp/$MYUSER/qgis/rep4 && chown $MYUSER:client /home/data/ftp/$MYUSER/qgis/rep4
   mkdir /home/data/ftp/$MYUSER/qgis/rep5 && chown $MYUSER:client /home/data/ftp/$MYUSER/qgis/rep5
   # Create a directory to store the cached server
   mkdir /home/data/cache/$MYUSER
   chmod 700 /home/data/cache/$MYUSER -R
   chown www-data:www-data /home/data/cache/$MYUSER -R

Map server: QGIS Server
====================================

.. note:: Details for the installation may differ for specific versions of the operating system. Please refer to http://qgis.org/en/site/forusers/download.html for up to date documentation.

Install
---------------

.. code-block:: bash

   # Add the repository UbuntuGis
   cat /etc/apt/sources.list.d/debian-gis.list
   deb http://qgis.org/debian trusty main
   deb-src http://qgis.org/debian trusty main

   # Add keys
   sudo gpg --recv-key DD45F6C3
   sudo gpg --export --armor DD45F6C3 | sudo apt-key add -

   # Update package list
   sudo apt-get update

   # Install QGIS Server
   sudo apt-get install qgis-server python-qgis

.. note:: See http://docs.qgis.org/testing/en/docs/user_manual/working_with_ogc/ogc_server_support.html for more information on QGIS Server.

Retrieve and install Lizmap Web Client
--------------------------------------------------------------

.. code-block:: bash

   cd /var/www/

With ZIP file
~~~~~~~~~~~~~~~~~~~~~~~~

Retrieve the latest available stable version from https://github.com/3liz/lizmap-web-client/releases/

.. code-block:: bash

   cd /var/www/
   # Options
   MYAPP=lizmap-web-client
   VERSION=2.12.4
   # Archive recovery with wget
   wget https://github.com/3liz/$MYAPP/archive/$VERSION.zip
   # Unzip archive
   unzip $VERSION.zip
   # virtual link for http://localhost/lm
   ln -s /var/www/$MYAPP-$VERSION/lizmap/www/ /var/www/html/lm
   # Remove archive
   rm $VERSION.zip

Development version with git
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. warning:: The development version is always changing, and bugs can occur. Do not use it in production.

* The first time

.. code-block:: bash

   apt-get install git
   cd /var/www/
   MYAPP=lizmap-web-client
   VERSION=master
   # Clone the master branch
   git clone https://github.com/3liz/lizmap-web-client.git $MYAPP-$VERSION
   # Go into the git repository
   cd $MYAPP-$VERSION
   # Create a personal branch for your changes
   git checkout -b mybranch

* To update your branch from the master repository

.. code-block:: bash

   cd /var/www/$MYAPP-$VERSION
   # Check that you are on the branch: mybranch
   git checkout mybranch

   # If you have any changes, make a commit
   git status
   git commit -am "Your commit message"

   # Save your configuration files!
   cp lizmap/var/jauth.db /tmp/jauth.db && cp lizmap/var/logs.db /tmp/logs.db && cp lizmap/var/config/lizmapConfig.ini.php /tmp/lizmapConfig.ini.php

   # Update your master branch
   git checkout master && git fetch origin && git merge origin/master
   # Apply to your branch, marge and manage potential conflicts
   git checkout mybranch && git merge master
   # Apply rights
   chown :www-data temp/ lizmap/var/ lizmap/www lizmap/install/qgis/edition/ -R
   chmod 775 temp/ lizmap/var/ lizmap/www lizmap/install/qgis/edition/ -R

.. note:: It is always good to make a backup before updating.

Give the appropriate rights to directories and files
--------------------------------------------------------------

.. code-block:: bash

   cd /var/www/$MYAPP-$VERSION
   chown :www-data temp/ lizmap/var/ lizmap/www lizmap/install/qgis/edition/ -R
   chmod 775 temp/ lizmap/var/ lizmap/www lizmap/install/qgis/edition/ -R

First test
--------------------------------------------------------------

Go to the Lizmap Web Client home to see if the installation was performed correctly: http://localhost/lm

.. note:: Replace ``localhost`` with the address or IP number of your server.

Lizmap is accessible, without further configurations, also as WMS and WFS server from a browser:

http://localhost/lm/index.php/lizmap/service/?repository=montpellier&project=montpellier&VERSION=1.3.0&SERVICE=WMS&REQUEST=GetCapabilities

http://localhost/lm/index.php/lizmap/service/?repository=montpellier&project=montpellier&SERVICE=WFS&REQUEST=GetCapabilities

and from QGIS:

http://localhost/lm/index.php/lizmap/service/?repository=montpellier&project=montpellier&VERSION=1.3.0&

http://localhost/lm/index.php/lizmap/service/?repository=montpellier&project=montpellier&

.. note:: Access to the WMS and WFS servers can be limited by assigning privileges to specific repositories, see the administration section.

Editing tool: Configure the server with the database support
=============================================================================

.. note:: This section is optional

PostgreSQL
------------------------------

For the editing of PostGIS layers in Web Client Lizmap operate, install PostgreSQL support for PHP.

.. code-block:: bash

   sudo apt-get install php5-pgsql
   sudo service apache2 restart

.. note:: For editing, we strongly recommend using a PostgreSQL database. This greatly simplifies installation and retrieval of data entered by users.

Spatialite
------------------------------

Enable Spatialite extension
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To use editing on layers spatiatlite,you have to add the spatialite extension in PHP. You can follow these instructions to do so:
http://www.gaia-gis.it/spatialite-2.4.0-4/splite-php.html

Lizmap Web Client tests whether the spatialite support is enabled in PHP. If it is not, then spatialities layers will not be used in the editing tool. You can always use PostgreSQL data for editing.

Give the appropriate rights to the directory containing Spatialite databases
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

So that Lizmap Web Client can modify the data contained in databases Spatialite, we must ensure that **the Apache user (www-data) has well write access to the directory containing each Spatialite file**

For example, if a directory contains a QGIS project, which uses a Spatialite database placed in a **db** directory at the same level as the QGIS project:

.. code-block:: bash

   /path/to/a/lizmap_directory
   |--- mon_projet.qgs
   |--- bdd
      |--- my_spatialite_file.sqlite

So you have to give the rights in this way:

.. code-block:: bash

   chown :www-data /path/to/a/lizmap_directory -R
   chmod 775 /path/to/a/lizmap_directory -R

.. note:: so if you want to install Lizmap to provide access to multiple map publishers, you should tell them to always create a **db** directory at the same level as the QGIS projects in the Lizmap Web Client directory. This will facilitate the admin work that just have to change the rights of this unique directory.

Version upgrade
===============================================================

Preliminary backup
--------------------------------------------------------------

Before update, make a backup of the configuration data: *lizmap/var/config/lizmapConfig.ini.php*, *lizmap/var/jauth.db* and the log file (from 2.8) *lizmap/var/logs.db*

.. code-block:: bash

   MYAPP=lizmap-web-client
   OLDVERSION=2.8.1 # replace by the version number of your current lizmap installation
   # if you installation is 2.1.0 or less, use an empty OLDVERSION instead :
   # OLDVERSION=
   cp /var/www/$MYAPP-$OLDVERSION/lizmap/var/jauth.db /tmp/jauth.db # user database
   cp /var/www/$MYAPP-$OLDVERSION/lizmap/var/config/lizmapConfig.ini.php /tmp/lizmapConfig.ini.php # text configuration file with services and repositories
   cp /var/www/$MYAPP-$OLDVERSION/lizmap/var/logs.db /tmp/logs.db # lizmap logs

Then do a typical installation of the new version (see above), which will create a new folder in the directory */var/www/*

Copy the files saved in the folder of the new version
-----------------------------------------------------------------------

.. code-block:: bash

   $VERSION=2.10.3
   cp /tmp/jauth.db /var/www/$MYAPP-$VERSION/lizmap/var/jauth.db
   cp /tmp/lizmapConfig.ini.php /var/www/$MYAPP-$VERSION/lizmap/var/config/lizmapConfig.ini.php
   cp /tmp/logs.db /var/www/$MYAPP-$VERSION/lizmap/var/logs.db

.. note:: In some versions, it is also necessary to update the database that stores the rights. See the following for more details.

From version 2.3 or lower to version 2.4 or upper
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Jelix framework (tool with which Lizmap Web Client is built) has been updated. It is necessary to change the rights management SQLite database:

.. code-block:: bash

   cd /var/www/$MYAPP-$VERSION/
   sqlite3 lizmap/var/jauth.db < lizmap/install/sql/upgrade_jacl2db_1.3_1.4.sql

From version 2.6 or lower to version 2.7
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Support for annotations and management of related rights was added to Lizmap Web Client. It is necessary to change the rights management SQLite database to upgrade it:

.. code-block:: bash

   cd /var/www/$MYAPP-$VERSION/
   sqlite3 lizmap/var/jauth.db < lizmap/install/sql/upgrade_jacl2db_lizmap_from_2.0_and_above_to_2.5.sql


From version 2.7.*  to version 2.8
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The editing tool replaced the annotation tool and fields to describe each Lizmap Web Client user has been added. It is necessary to upgrade the rights management SQLite database:

.. code-block:: bash

   cd /var/www/$MYAPP-$VERSION/
   sqlite3 lizmap/var/jauth.db < lizmap/install/sql/upgrade_jacl2db_2.7_2.8.sql

From version 2.8.*  to version 2.9
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The functionality of layers data filtering based on the connected user requires the addition of rights related to the user data base:

.. code-block:: bash

   cd /var/www/$MYAPP-$VERSION/
   sqlite3 lizmap/var/jauth.db < lizmap/install/sql/upgrade_jacl2db_2.8_2.9.sql

From version 2.9.*  to version 2.10
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The functionality of layers data filtering based on the connected user requires the addition of rights related to the user data base:

.. code-block:: bash

   cd /var/www/$MYAPP-$VERSION/
   sqlite3 lizmap/var/jauth.db < lizmap/install/sql/upgrade_jacl2db_2.9_2.10.sql

Delete Jelix temporary files
--------------------------------------------------------------

.. code-block:: bash

   rm -rf /var/www/$MYAPP-$VERSION/temp/lizmap/*

Redefine the rights to the application files
-------------------------------------------------------

.. code-block:: bash

   cd /var/www/$MYAPP-$VERSION
   chown :www-data temp/ lizmap/var/ lizmap/www lizmap/install/qgis/edition/ -R
   chmod 775 temp/ lizmap/var/ lizmap/www lizmap/install/qgis/edition/ -R
