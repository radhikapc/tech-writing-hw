# What To Do

Below, you'll find the beginning of an article on setting up MediaWiki with the lighttpd web server on Ubuntu 14.04.  The introduction, prerequisites, and some section headers are included to get you started.  Edit these if you need to, but use them as a guideline for what your tutorial's scope should be.

An outdated version of MediaWiki is available through Ubuntu's repository, but you should tell users how to set up the latest stable version instead.

If you wish to expand the scope of the article, consider adding information about one of these:

* Securing your wiki with SSL
* Storing your wiki data in a remote database
* Creating rewrites to configure pretty URLs

Good luck!

----------------------

### Introduction

MediaWiki is a popular open source wiki platform that can be used for public or internal collaborative content publishing.  MediaWiki is used for many of the most popular wikis on the internet including Wikipedia, the site that the project was originally designed to serve.

In this guide, we will be setting up the latest version of MediaWiki on an Ubuntu 14.04 server.  We will use the `lighttpd` web server to make the actual content available, `php-fpm` to handle dynamic processing, and `mysql` to store our wiki's data.

## Prerequisites

To complete this guide, you should have access to a clean Ubuntu 14.04 server instance.  On this system, you should have a non-root user configured with `sudo` privileges for administrative tasks.  You can learn how to set this up by following our [Ubuntu 14.04 initial server setup guide](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04).

When you are ready to continue, log into your server with your `sudo` user and get started below.

## Install the Server Components

You must first install the server software that supports the MediaWiki. Use `apt-get` to install the `Lighttpd` web server, `php-fpm`modules, and `mysql` server. This combination is an alternative to the LAMP stack that you generally install to get the web server up and running.

### Prerequisites

* The initial server configuration is completed on your Ubuntu 14.04 machine
* User is given the root privileges 

### Install Lighttpd

Lighttpd is a lightweight alternative to the Apache web server. Lighttpd is capable of serving high loads of traffic while consuming less memory as compared to other popular web servers.

To install Lighttpd, open a terminal and run the following:

    $ sudo apt-get update && sudo apt-get install lighttpd

The `apt-get update` command fetch the latest package index files from the source repositories onto your machine. The `apt-get install` command install the latest version of `lighttpd`. The advantage of combining two commands by using `&&` is that you don't have to repeat the `sudo` command.

Direct your browser to "http://localhost" on your local machine or to "http://<serverip>" from any other machines on the network to access the Lighttpd placeholder web page. Note that <serverip> is the IP address of the machine running the lighttpd service.

### Install MySQL

MySQL is a robust database management system used for organizing and retriving data.

To install MySQL, open a terminal and run the following:

    $ sudo apt-get install mysql-server mysql-client

Create and confirm a root password when prompted. You will use this password to connect to the MySQL database server. Alternatively, you can skip this step and set up the root password later from within a MySQL shell.

If you prefer to have a secure MySQL installation,  run the following:

    $ sudo mysql_secure_installation

Follow the on-screen instructions. If you have not created a root password yet, create one.

### Install PHP-FPM

PHP-FPM (FastCGI Process Manager) is an alternative PHP FastCGI implementation designed for virtual hosting. PHP-FPM has additional features, such as accelerated upload support, that are useful for sites with high traffic load.

To install PHP-FPM, open a terminal and run the following:

    $ sudo apt-get install php5-fpm php5-cli php5-mysql -y

This command installs `php-fpm` and the modules `php-cli` and `php-mysql`. The `php-cli` module makes the php5 command available to the terminal for php5 scripting. The `php-mysql` module allows PHP5 scripts to talk to a MySQL database. By default, `php-fpm` service cannot talk to a MySQL database.

You can browse through the list of modules by running the following:

    $ sudo apt-cache search php5

To support images, caching, and localization for international data on the MediaWiki site, three additional PHP modules are necessary: `php5-gd`, `php5-xcache`, and `php5-intl`. Install them.

    $ sudo apt-get install php5-gd php5-xcache php5-intl
  
After you finish installing the new packages, restart the `php-fpm` service.

    $ sudo service php5-fpm restart

## Configure MySQL and Create Credentials for MediaWiki

You need a database and database user exclusively for Mediawiki. For example, create a database named *mediawikidb* and user named *wikiuser*.

   1. Open a MySQL terminal.
      `mysql -u root -p`
   2. Create a MySQL database for MediaWiki.
      `CREATE DATABASE mediawikidb;`
   3. Create a MySQL user for MediaWiki.
      `CREATE USER wikiuser@localhost IDENTIFIED BY 'mediawikipassword';`
   4. Grant permission to the user that you have created in the previous step.
      `GRANT index, create, select, insert, update, delete, alter, lock tables on mediawikidb.* TO`             `wikiuser@localhost;`
   5. Confirm the database and the user are created:
      `SELECT User,Host FROM mysql.user`
     6. Restart MySQL.
      `service mysql restart`


## Configure Lighttpd

Configure your lighttpd instance to provide only the services that you need for your use case. Each user on the MediaWiki can have their own web directory in their home folder if you enable user directories on `lighttpd`. The default document root is `/var/www` on Ubuntu, and the configuration file is `/etc/lighttpd/lighttpd.conf`. Additional configurations are stored in files in the `/etc/lighttpd/conf-available` directory. You can enable these configurations by using the `lighttpd-enable-mod` command which creates a symlink from the `/etc/lighttpd/conf-enabled` directory to the appropriate configuration file in `/etc/lighttpd/conf-available`. You can disable configurations with the `lighttpd-disable-mod` command.

To enable user directories, open a terminal and run the following:

    $ sudo lighttpd-enable-mod userdir

To reload the configuration, run the following:

    $ sudo service lighttpd reload

Now users can place files in a `public_html` folder in their home directory to have them hosted on the web server. 
For example, the user *wikiuser* would place the files in `/home/wikiuser/public_html` and access them at `http://serverip/~wikiuser`.

## Configure PHP-FPM 

PHP-FPM is known to be an efficient backend for handling php requests as compared to fastcgi-php. If both `php-fpm` and `lighttpd` work on the same machine, commucating through native Unix domain sockets is apparently efficient than TCP socket. 

To enable PHP5 in Lighttpd, modify `/etc/php5/fpm/php.ini` and uncomment the line `cgi.fix_pathinfo=1`:

    [...]
    ; cgi.fix_pathinfo provides *real* PATH_INFO/PATH_TRANSLATED support for CGI.  PHP's
    ; previous behaviour was to set PATH_TRANSLATED to SCRIPT_FILENAME, and to not grok
    ; what PATH_INFO is.  For more information on PATH_INFO, see the cgi specs.  Setting
    ; this to 1 will cause PHP CGI to fix its paths to conform to the spec.  A setting
    ; of zero causes PHP to behave as before.  Default is 1.  You should fix your scripts
    ; to use SCRIPT_FILENAME rather than PATH_TRANSLATED.
    ; http://php.net/cgi.fix-pathinfo
    cgi.fix_pathinfo=1
    [...]

To enable native unix domain socket commucation,  edit `/etc/lighttpd/conf-available/10-php-fpm.conf` as follows:

    server.modules+=("mod_fastcgi")
    fastcgi.server = ( ".php" =>
      ( "localhost" =>
         (
         "socket" => "/var/run/php5-fpm.sock"
          )
      )
    )

The `15-fastcgi-php.conf` configuration file must be updated to use `php5-fpm`. 

First, take a back up of the file:

    $ sudo cp /etc/lighttpd/conf-available/15-fastcgi-php.conf 15-fastcgi-php.conf.BAK
  
Open `15-fastcgi-php.conf` in a text editor and change the socket path to `/var/run/php5-fpm.sock`:

     # -*- depends: fastcgi -*-
     # /usr/share/doc/lighttpd/fastcgi.txt.gz
     # http://redmine.lighttpd.net/projects/lighttpd/wiki/Docs:ConfigurationOptions#mod_fastcgi-fastcgi
     
     ## Start an FastCGI server for php (needs the php5-cgi package)
     fastcgi.server += ( ".php" =&gt; 
            ((
                    "socket" =&gt; "/var/run/php5-fpm.sock",
                    "broken-scriptfilename" =&gt; "enable"
            ))
    )

Enable `fastcgi` and `fastcgi-php`:

    $ sudo lighttpd-enable-mod fastcgi
    $ sudo lighttpd-enable-mod fastcgi-php

Reload the web server configuration:

    $ sudo /etc/init.d/lighttpd force-reload 
  
Restart both `lightpd` and `php-fpm` services.

    $ sudo /etc/init.d/php5-fpm restart
    $ sudo /etc/init.d/lighttpd restart

To verify whether `php-fpm` is running, run the following:

    $ status php5-fpm
    $ service php5-fpm status 

If the service is running, you should see a message as follows:

    php5-fpm start/running, process 6754

To test your `php-fpm` installation, create a file, `info.php`, in the `/var/www`directory. It is the document root of the default website.

    $ vi /var/www/info.php

Enter the following:

    <?php
     phpinfo();
    ?>

Open a browser and direct to http://<server IP>/info.php. You will see a page displaying the PHP information.


## Install MediaWiki

Install the MediaWiki package:

    $ sudo apt-get install mediawiki

Optionally, install the add-on packages:

    $  sudo apt-get install imagemagick mediawiki-math php5-gd

Use your favorite browser to complete rest of the MediaWiki configuration:

  1. Open a browser and go to "http://server_ip_address/wiki/index.php".
  2. Click on the link to setup the wiki.
  3. Enter the information and click continue to complete the MediaWiki setup.
  
After the MediaWiki configuration is completed, an administrator account is created. On the top-right side of the Main page, click Login and confirm that you can login to the wiki by using the account’s credentials. 
  

## Conclusion

The Ubuntu 14.04 server is now configured with a `lighttdp` web server, MySQL database, and a latest MediaWiki site. Feel free to share the wiki site within or outside of your organization.


