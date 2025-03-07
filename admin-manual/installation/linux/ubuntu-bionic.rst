.. _installation-linux-ubuntu-bionic:

========================================
Linux - Ubuntu 18.04 LTS (Bionic Beaver)
========================================

Most of the configuration steps described in this
document apply to any modern Linux environment, however some of them will
apply only to Ubuntu and likely to any Ubuntu-based distribution.

This document is based in
`Ubuntu 18.04 LTS (Bionic Beaver) <http://releases.ubuntu.com/bionic/>`_. Once
you have installed it, you should be able to follow the instructions described
below. In particular, we are going to use Ubuntu packages that can be found
under the `repositories <https://help.ubuntu.com/community/Repositories/Ubuntu>`_
*main* and *universe*.

.. IMPORTANT::

   Please make sure you have reviewed the :ref:`requirements
   <installation-requirements>` before proceeding. Also, you may want to
   consider setting up the :ref:`firewall <security-firewall>` before you start
   installing the services described below to avoid exposing them to outside
   access.

.. _linux-ubuntu-bionic-install-dependencies:

Install the dependencies
========================

.. _linux-ubuntu-bionic-dependency-mysql:

MySQL
-----

We strongly recommend using `MySQL <https://www.mysql.com/>`__  5.7 as it's
much better than the previous major release in terms of speed, scalability and
user-friendliness. Also, we've experienced very good results using forks like
Percona Server or MariaDB, so don't be afraid and use them if you want!

.. code-block:: bash

   sudo apt install mysql-server-5.7

During the installation, you will be prompted to set a password for the default
administrator user (root). We strongly recommend that you use a strong password
and please write it down somewhere safe, you are going to need it later.

.. TIP::

   If you are not prompted on installation to create a username and password,
   you can use 'root' as the user and password in later steps. We will create a
   different mysql user in later steps, where you can add a secure password.

.. _linux-ubuntu-bionic-dependency-elasticsearch:

Elasticsearch
-------------

A relatively new search server based on Apache Lucene and developed in Java that
has brought AtoM a lot of advanced features, performance and scalability. This
is probably the biggest change introduced in AtoM 2.x and we are pleased with
the results.

Ubuntu doesn't provide a package but you can download it directly from the
`Elasticsearch site <https://www.elastic.co/downloads/elasticsearch>`_ if you
are unable to download it using the method that follows.

Make sure that `Java <https://www.java.com/en/>`__ is installed. In this
example we are going to use OpenJDK but Oracle's JVM would also work.

.. code-block:: bash

   sudo add-apt-repository ppa:openjdk-r/ppa
   sudo apt-get update
   sudo apt install openjdk-11-jre-headless software-properties-common

After successfully installing Java, proceed to install Elasticsearch. Download
and install the public signing key used in their repository:

.. code-block:: bash

   wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

.. IMPORTANT:: 

   Don't miss the dash ( ``-`` ) at the end of the above command! 

Now add their repository:

.. code-block:: bash

   echo "deb https://artifacts.elastic.co/packages/5.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-5.x.list

Ready to be installed. Run:

.. code-block:: bash

   sudo apt update
   sudo apt install elasticsearch

Start the service and configure it to start when the system is booted.

.. code-block:: bash

   sudo systemctl enable elasticsearch
   sudo systemctl start elasticsearch

.. _linux-ubuntu-bionic-dependency-httpd:

Web server
----------

There are many web servers out there capable of working well with
`PHP <http://php.net/>`__. `Apache <https://httpd.apache.org/>`__ is probably
the most popular and we like it, but we've found that
`Nginx <http://nginx.com/>`__ adapts itself much better to limited resource
environments while it also scales better and more predictably under high loads.
You are welcome to try other solutions, but the following documentation will
focus on Nginx.

.. WARNING::

   The following instructions assume that the Nginx package is creating the
   directory :file:`/usr/share/nginx` and that is the location where we are
   going to place the AtoM sources. However, we have been told this location may
   be different in certain environments (e.g. :file:`/var/www`) or you may opt
   for a different location. If that is the case, please make sure that you
   update the configuration snippets that we share later in this document
   according to your location.

.. _linux-ubuntu-bionic-dependency-httpd-nginx:

Nginx
`````

In Ubuntu, the installation of Nginx is simple:

.. code-block:: bash

   sudo apt install nginx

Nginx deploys a default server (aka VirtualHost, for Apache users) called
**default** and you can find it in :file:`/etc/nginx/sites-available/default`.
In order to install AtoM you could edit the existing server block or add a new
one. We are going to you show you how to do the latter:

.. code-block:: bash

   sudo touch /etc/nginx/sites-available/atom
   sudo ln -sf /etc/nginx/sites-available/atom /etc/nginx/sites-enabled/atom
   sudo rm /etc/nginx/sites-enabled/default

We have now created the configuration file and linked it from sites-enabled/,
which is the directory that Nginx will look for. This means that you could
disable a site by removing its symlink from sites-enabled/ while keeping the
original one under sites-available/, in case that you want to re-use it in the
future. You can do this with the Nginx default server.

The following is a recommended server block for AtoM. Put the following contents
in :file:`/etc/nginx/sites-available/atom`.

.. code-block:: nginx

   upstream atom {
     server unix:/run/php7.2-fpm.atom.sock;
   }

   server {

     listen 80;
     root /usr/share/nginx/atom;

     # http://wiki.nginx.org/HttpCoreModule#server_name
     # _ means catch any, but it's better if you replace this with your server
     # name, e.g. archives.foobar.com
     server_name _;

     client_max_body_size 72M;

     # http://wiki.nginx.org/HttpCoreModule#try_files
     location / {
       try_files $uri /index.php?$args;
     }

     location ~ /\. {
       deny all;
       return 404;
     }

     location ~* (\.yml|\.ini|\.tmpl)$ {
       deny all;
       return 404;
     }

     location ~* /(?:uploads|files)/.*\.php$ {
       deny all;
       return 404;
     }

     location ~* /uploads/r/(.*)/conf/ {

     }

     location ~* ^/uploads/r/(.*)$ {
       include /etc/nginx/fastcgi_params;
       set $index /index.php;
       fastcgi_param SCRIPT_FILENAME $document_root$index;
       fastcgi_param SCRIPT_NAME $index;
       fastcgi_pass atom;
     }

     location ~ ^/private/(.*)$ {
       internal;
       alias /usr/share/nginx/atom/$1;
     }

     location ~ ^/(index|qubit_dev)\.php(/|$) {
       include /etc/nginx/fastcgi_params;
       fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
       fastcgi_split_path_info ^(.+\.php)(/.*)$;
       fastcgi_pass atom;
     }

     location ~* \.php$ {
       deny all;
       return 404;
     }

   }

Now you need to enable and reload Nginx:

.. code-block:: bash

   sudo systemctl enable nginx
   sudo systemctl reload nginx

.. _linux-ubuntu-bionic-dependency-php:

PHP
---

Ubuntu 18.04 bundles PHP 7.2, which is much faster than older releases.

Our favorite way to deploy AtoM is using `PHP-FPM <http://php-fpm.org/>`__, a
process manager that scales better than other solutions like FastCGI. The
following command will install it along with the rest of PHP extensions
:ref:`required <installation-requirements>` by AtoM:

.. code-block:: bash

   sudo apt install php7.2-cli php7.2-curl php7.2-json php7.2-ldap php7.2-mysql php7.2-opcache php7.2-readline php7.2-xml php7.2-fpm php7.2-mbstring php7.2-xsl php7.2-zip php-apcu

If you are using Memcached as cache engine, you will also need to install `php-memcache`:

.. code-block:: bash

   sudo apt-get install php-memcache

Let's add a new PHP pool for AtoM by adding the following contents in a new file
called :file:`/etc/php/7.2/fpm/pool.d/atom.conf`:

.. code-block:: ini

   [atom]

   ; The user running the application
   user = www-data
   group = www-data

   ; Use UNIX sockets if Nginx and PHP-FPM are running in the same machine
   listen = /run/php7.2-fpm.atom.sock
   listen.owner = www-data
   listen.group = www-data
   listen.mode = 0600

   ; The following directives should be tweaked based in your hardware resources
   pm = dynamic
   pm.max_children = 30
   pm.start_servers = 10
   pm.min_spare_servers = 10
   pm.max_spare_servers = 10
   pm.max_requests = 200

   chdir = /

   ; Some defaults for your PHP production environment
   ; A full list here: http://www.php.net/manual/en/ini.list.php
   php_admin_value[expose_php] = off
   php_admin_value[allow_url_fopen] = on
   php_admin_value[memory_limit] = 512M
   php_admin_value[max_execution_time] = 120
   php_admin_value[post_max_size] = 72M
   php_admin_value[upload_max_filesize] = 64M
   php_admin_value[max_file_uploads] = 10
   php_admin_value[cgi.fix_pathinfo] = 0
   php_admin_value[display_errors] = off
   php_admin_value[display_startup_errors] = off
   php_admin_value[html_errors] = off
   php_admin_value[session.use_only_cookies] = 0

   ; APC
   php_admin_value[apc.enabled] = 1
   php_admin_value[apc.shm_size] = 64M
   php_admin_value[apc.num_files_hint] = 5000
   php_admin_value[apc.stat] = 0

   ; Zend OPcache
   php_admin_value[opcache.enable] = 1
   php_admin_value[opcache.memory_consumption] = 192
   php_admin_value[opcache.interned_strings_buffer] = 16
   php_admin_value[opcache.max_accelerated_files] = 4000
   php_admin_value[opcache.validate_timestamps] = 0
   php_admin_value[opcache.fast_shutdown] = 1

   ; This is a good place to define some environment variables, e.g. use
   ; ATOM_DEBUG_IP to define a list of IP addresses with full access to the
   ; debug frontend or ATOM_READ_ONLY if you want AtoM to prevent
   ; authenticated users
   env[ATOM_DEBUG_IP] = "10.10.10.10,127.0.0.1"
   env[ATOM_READ_ONLY] = "off"

The process manager has to be enabled and restarted:

.. code-block:: bash

   sudo systemctl enable php7.2-fpm
   sudo systemctl start php7.2-fpm

If the service fails to start, make sure that the configuration file has been
has been pasted properly. You can also check the syntax by running:

.. code-block:: bash

   sudo php-fpm7.2 --test

If you are not planning to use the default PHP pool (``www``), feel free to
remove it:

.. code-block:: bash

   sudo rm /etc/php/7.2/fpm/pool.d/www.conf
   sudo systemctl restart php7.2-fpm

.. _linux-ubuntu-bionic-gearman:

Gearman job server
------------------

Gearman job server is required by AtoM as of version 2.2.

.. code-block:: bash

   sudo apt install gearman-job-server

We'll configure the job server and the workers after initial installation (see
:ref:`below <linux-ubuntu-bionic-workers>`). Full configuration detalis can be
found here:

* :ref:`installation-asynchronous-jobs`.

.. _linux-ubuntu-bionic-other-packages:

Other packages
--------------
In order to generate PDF finding aids, AtoM requires Apache FOP 2.1 to be
installed. Fortunately, Apache FOP 2.1 can now be installed directly from
Ubuntu packages using the command below.

.. note::

   The command specified below uses the ``--no-install-recommends`` parameter:
   this is intentional and ensures that only dependencies are installed and not
   'recommended' or 'suggested' packages. If ``--no-install-recommends`` is not
   specified, openjdk-8-jre will be installed as a dependency for one of the
   recommended packages. Since openjdk-8-jre-headless was already installed in
   the Elasticsearch installation section above, we want to prevent installing
   the openjdk-8-jre package as well.

.. code-block:: bash

   sudo apt install --no-install-recommends fop libsaxon-java

If you want AtoM to be able to process :term:`digital objects <digital object>`
in formats like JPEG or to extract the text from your PDF documents, there are
certain packages that you need to install. They are not mandatory but if they
are found in the system, AtoM will use them to produce digital object
derivatives from your :term:`master objects <master digital object>`. for
more information on each, see: :ref:`Requirements: other dependencies
<other-dependencies>`. The following will install all the recommended
dependencies at once:

.. code-block:: bash

   sudo apt install imagemagick ghostscript poppler-utils ffmpeg

.. _linux-ubuntu-bionic-install-atom:

Download AtoM
=============

Now that we have installed and configured all dependencies, we are ready to
download and install AtoM itself. The safest way is to install AtoM from the
tarball, which you can find in the
`download section <http://www.accesstomemory.org/download/>`_. However,
experienced users may prefer to check out the code from our `public repository
<https://github.com/artefactual/atom>`__.

The following instructions assume that we are installing AtoM under
:file:`/usr/share/nginx` and that you are using AtoM |release|.

.. _linux-ubuntu-bionic-install-tarball:

Option 1: Download the tarball
------------------------------

.. code-block:: bash

   wget https://storage.accesstomemory.org/releases/atom-2.5.3.tar.gz
   sudo mkdir /usr/share/nginx/atom
   sudo tar xzf atom-2.5.3.tar.gz -C /usr/share/nginx/atom --strip 1

Please note that the tarball may not be available yet if this version is still
in development. In this case, you can try the alternative installation method
explained below.

.. _linux-ubuntu-bionic-checkout-git:

Option 2: Check out the code from our git repository
----------------------------------------------------

Install git:

.. code-block:: bash

   sudo apt install git

.. code-block:: bash

   sudo mkdir /usr/share/nginx/atom
   sudo git clone -b stable/2.5.x http://github.com/artefactual/atom.git /usr/share/nginx/atom
   cd /usr/share/nginx/atom

If you are not interested in downloading all the history from git, you could
also truncate it to a specific number of revisions, e.g.: just one revision

.. code-block:: bash

   git clone --depth 1 http://github.com/artefactual/atom.git /usr/share/nginx/atom

Once you've cloned the code from our git repository, you'll need to compile
the CSS files:

.. code-block:: bash

   sudo apt install nodejs npm make
   sudo npm install -g "less@<2.0.0"
   sudo make -C /usr/share/nginx/atom/plugins/arDominionPlugin

.. _linux-ubuntu-bionic-filesystem-permissions:

Filesystem permissions
======================

By default, Nginx runs as the www-data user. There are a few directories under
AtoM that must be writable by the web server. The easiest way to ensure this is
to update the owner of the AtoM directory and its contents by running:

.. code-block:: bash

   sudo chown -R www-data:www-data /usr/share/nginx/atom

If you are deploying AtoM in a shared environment we recommend you to pay
attention to the permissions assigned to **others**. The following is an
example on how to clear all mode bits for others:

.. code-block:: bash

   sudo chmod o= /usr/share/nginx/atom

.. _linux-ubuntu-bionic-create-database:

Create the database
===================

Assuming that you are running `MySQL <https://www.mysql.com/>`__ in localhost,
please create the database by running the following command using the
password you created :ref:`earlier <linux-ubuntu-bionic-dependency-mysql>`:

.. code-block:: bash

   sudo mysql -h localhost -u root -p -e "CREATE DATABASE atom CHARACTER SET utf8 COLLATE utf8_unicode_ci;"

.. note:: 

   If you do not supply the MySQL root password after the -p, you will be
   prompted for it when you enter the command. If you do supply the password,
   there is no space following -p; in other words, ``-pPASSWORD``. (Replace PASSWORD
   with the password you created.) Remember, supplying the password on the
   command line is less secure as it may be visible to others in the
   .bash_history file.
   
Notice that the database has been called **atom**. Feel free to change its name.

In case your MySQL server is **not** the same as your web server, replace
"localhost" with the address of your MySQL server.

.. warning::

   Plase make sure that you are using an empty database! Don't reuse an old
   database unless it's empty. You can always drop it by using the
   :command:`DROP DATABASE` command and then create it again.

Additionally, it's always a good idea to create a specific MySQL user for AtoM
to keep things safer. This is how you can create an user called ``atom`` with
password ``12345`` and the permissions needed for the database created above.

.. code-block:: bash

   sudo mysql -h localhost -u root -p -e "GRANT ALL ON atom.* TO 'atom'@'localhost' IDENTIFIED BY '12345';"

Note that the ``INDEX``, ``CREATE`` and ``ALTER`` privileges are only necessary
during the installation process or when you are upgrading AtoM to a newer
version. They can be removed from the user once you are finished with the
installation or you can change the user used by AtoM in :ref:`config.php <config-config-php>`.

Finally, let’s configure our MySQL modes. The The MySQL server can operate in
different SQL modes, which affects the SQL syntax MySQL supports and the data
validation checks it performs. We’ll add our preferred mode settings in a new
file.

First, let’s create a new file with our SQL modes. 

Paste the following values in a new file at ``/etc/mysql/conf.d/mysqld.cnf`` 
and save:

.. code-block:: bash

   [mysqld]
   sql_mode=STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
   optimizer_switch='block_nested_loop=off'

Now we’ll restart MySQL:

.. code-block:: bash

   sudo systemctl restart mysql

.. _linux-ubuntu-bionic-run-installer:

Run the web installer
=====================

You should now be ready to run the installer. It's a simple web interface that
changes some internal configuration files according to your environment and adds
the necessary tables and initial data to the database recently created.

Open your browser and type the URL in the address bar. The URL can greatly
change depending on your web server configuration. The URL will usually be
something like http://localhost. AtoM will redirect you to the installer
automatically.

The installation process consists of a number of steps where you will be asked
for configuration details such as the location of your database server. In some 
cases, it may provide default values, such as ``root`` for the database username.
If you have followed this document to the letter (including creating a different 
database user in the database configuration step 
:ref:` above <linux-ubuntu-bionic-create-database>`, this is how you should fill 
the following fields:

* Database name: ``atom``
* Database username: ``atom``
* Database password: ``12345``
* Database host: ``localhost``
* Database port: ``3306``
* Search host: ``localhost``
* Search port: ``9200``
* Search index: ``atom``

Of course, some of these fields will look very different if you are running
AtoM in a distributed way, where your services like MySQL or Elasticsearch are
running in separate machines.

The rest of the fields can be filled as you need:

* Site title
* Site description
* Site base URL
* Username
* E-mail address
* Password

.. TIP::

   You can always change the :term:`site title`, :term:`site description`, and
   :term:`Base URL` later via **Admin > Settings > Site information**. See:
   :ref:`site-information` for more information. The Username, email, and
   password can also be changed by an :term:`administrator` after installation
   via the :term:`user interface` - see: :ref:`edit-user`.

.. _linux-ubuntu-bionic-workers:

Deployment of workers
=====================

Gearman is used in AtoM to add support for asynchronous tasks like
SWORD deposits, managing rights inheritance, and generating finding aids. Check
out the following page for further installation details:
:ref:`installation-asynchronous-jobs`.

.. IMPORTANT::

   You **must** complete the installation instructions found on the Job Scheduler
   page for your AtoM installation to be fully functional! Increasingly in AtoM,
   the job scheduler is used to support long-running tasks, some of which are
   core functionality such as updating the :term:`publication status` of a
   descriptive hierarchy, moving descriptions to a new :term:`parent record`, and
   much more. Please do this now! See:

   * :ref:`installation-asynchronous-jobs`

.. _linux-ubuntu-bionic-configuration-files:

Configure AtoM via the command-line
===================================

There are various settings that can only be configured via the command-line -
for example, the default timezone of the application. Depending on your local
requirements, it may be preferable to configure some of these now. For more
information on these settings see: :ref:`customization-config-files`.

.. _linux-ubuntu-bionic-security-considerations:

Security considerations
=======================

Now that AtoM is installed, please take a moment to read our
:ref:`security section <security>` where we will show you how to
configure the firewall in Ubuntu and back up AtoM.

We strongly encourage our users to configure a firewall because some of the
services configured should not be exposed in the wild, e.g. Elasticsearch was
not designed to be accessible from untrusted networks and it's a common attack
vector.
