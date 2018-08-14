# FSND Project: Linux Server Configuration
#### by Omotayo Madein
---

## Description

This is a project for [Udacity Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004). The Linux Server Configuration project focused on the installation of a Linux distribution on a virtual machine and preparing it to host [a web application](https://github.com/tayomadein/fsnd-catalog), to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers. 

## Accessing the Server
>- This server as been stopped

* URL: ~[Home](http://35.180.121.118/)~
* SSH port: changed from 22 to 2200
* IP adress: ~35.180.121.118~

## Configuration
---

### Setting Up [AWS Lightsail Instance](https://lightsail.aws.amazon.com/)

* Log in to Amazon Lightsail via AWS account
* Create and configure an instance with a plain Ubuntu Linux image
* Connect to instance from local machine:
    >* Download default key from AWS Lightsail account page.
    >* Move downloaded file with `.pem` extension to `~/.ssh/`
    >* Give permissions to the file by running `chmod 600 ~/.ssh/file_name.pem` on your terminal.
    >* Run `ssh -i ~/.ssh/file_name.pem [username]@[public-ip-address]`. username and public-ip-address can be gotten from Lightsail's dashboard.

### Securing Your Server

* Update currently installed packages by running `sudo apt-get update` and `sudo apt-get upgrade`
* To change the SSH port from 22 to 2200 run `sudo nano /etc/ssh/sshd_config` look for `PORT 22` and change it to `PORT 2200`. Save and run `sudo service ssh restart`
* Run the following commands to configure the Uncomplicated Firewall:
    ```text
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh
    sudo ufw allow 2200/tcp
    sudo ufw allow 80/http
    sudo ufw allow 123/udp
    sudo ufw deny 22
    sudo ufw enable
    sudo ufw status
    ```
* `sudo ufw status` would produce an output like this:
    ```text
    Status: active

    To                         Action      From
    --                         ------      ----
    2200/tcp                   ALLOW       Anywhere
    80/tcp                     ALLOW       Anywhere
    123/udp                    ALLOW       Anywhere
    22                         DENY        Anywhere
    2200/tcp (v6)              ALLOW       Anywhere (v6)
    80/tcp (v6)                ALLOW       Anywhere (v6)
    123/udp (v6)               ALLOW       Anywhere (v6)
    22 (v6)                    DENY        Anywhere (v6)
    ```
* Update AWS Lightsail firewall by clicking on the networking tab for an instance and updating its rules for ports 80(TCP), 123(UDP), and 2200(TCP)

* You must specify PORT number to access the terminal again 
    ```ssh -i ~/.ssh/file_name.pem -p 2200 [username]@[public-ip-address]```

#### User `grader`

* Run `sudo adduser grader` to create `grader` user and enter password when prompted.
* Give `grader` user sudo permissions by running `sudo visudo` and update file with `grader ALL=(ALL:ALL) ALL` in the appropraite section
* Create an SSH key pair for `grader`:
    >* On local terminal, run `ssh-keygen`
    >* Choose a name to store the key pair `file_name_key` and enter password when prompted.
    >* Login as `grader` by running `sudo su - grader` on your instance terminal
    >* Run `sudo mkdir ~/.ssh` and `sudo touch .ssh/authorized_keys`
    >* Copy the content of `file_name_key.pub` from local machine and paste it in `.ssh/authorized_keys`
    >* Run `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
    >* Open `/etc/ssh/sshd_config` and ensure that `PasswordAuthentication` is set to **no**
    >* Run `sudo service ssh restart`
* You can login via the grader user by running
    ```ssh -i ~/.ssh/file_name_key.pub -p 2200 grader@[public-ip-address]```

### Deployment

#### Configure Environment

* Change your timezone to UTC `sudo timedatectl set-timezone UTC`
* Install Python, Apache, PostgreSQL, git and mod_wsgi
    ```text
    sudo apt-get install apache2
    sudo apt-get install libapache2-mod-wsgi python-dev
    sudo a2enmod wsgi //To enable mond_wsgi
    sudo apt-get install postgresql
    sudo apt-get install python2 // Or python3
    sudo apt-get install git
    ```
* Create a new role for catalog:
    >* Run `sudo -u postgres psql postgres` to access psql using the `postgres` user created during install
    >* Run `CREATE ROLE catalog WITH LOGIN;` and `ALTER ROLE catalog CREATEDB;` to create the `catalog` user and give the user ability to create database
    >* Run `\password catalog` to create a password
    >* Run `\q` to exit psql
* Follow the steps for [`grader`](#user-grader) to create a new user `catalog`
* Login as `catalog` by running `sudo su - catalog` and run `createdb catalog`
* Switch back to `ubuntu` user

#### Update Repo and Clone

* Prepare a release version of the repo and set the release branch as default:
    >* Rename `app.py` to `__init__.py`
    >* In `__init__.py`, change `app.run(host='0.0.0.0', port=5000)` to `app.run()`
    >* Update `__init__.py`, `database_init.py` and `database_setup.py` with `engine = create_engine('postgresql://catalog:DB_PASSWORD@localhost/catalog')`
    >* n
* Our app will be stored in _/var/www_ `cd /var/www`, run `sudo mkdir itemCatalog`
* Change the ownership of `itemCatalog` by running `sudo chown -R www-data:www-data itemCatalog/` and `cd itemCatalog`
* Then clone the [Catalog Project](https://github.com/tayomadein/fsnd-catalog) by running `sudo -u www-data git clone https://github.com/tayomadein/fsnd-catalog.git itemCatalog`
* Update catalog project on Google API Console to accept new public IP address

#### Setup Virtual environment

* Install python-pip `sudo apt-get install python-pip`
* Install virtualenv with `sudo apt-get install python-virtualenv`
* Run `cd /var/www/itemCatalog/itemCatalog` and `sudo virtualenv venv` (_venv_ is the environment name, you can use any name)
* Activate the environment by running `. venv/bin/activate`
* Install required libraries
    ```text
    apt-get install python-psycopg2 python-flask
    apt-get install python-sqlalchemy python-pip //pip and sql alchemy
    pip install --upgrade oauth2client //oauth for login
    pip install requests
    pip install httplib2
    pip install flask-seasurf
    sudo apt-get install libpq-dev
    ```
* Run `python __init__.py` to test that it works
* Run `deactivate` to stop the virtual environment

#### Setup Application

* Configure the virtual host by running `sudo nano /etc/apache2/sites-available/itemCatalog.conf` and add the following and save:
    ```ApacheConf
    <VirtualHost *:80>
                ServerName **PUBLIC IP ADDRESS**
                ServerAdmin myadminemail@domain.com
                WSGIScriptAlias / /var/www/itemCatalog/itemcatalog.wsgi
                <Directory /var/www/itemCatalog/itemCatalog/>
                        Order allow,deny
                        Allow from all
                        Options -Indexes
                </Directory>
                Alias /static /var/www/itemCatalog/itemCatalog/static
                <Directory /var/www/itemCatalog/itemCatalog/static/>
                        Order allow,deny
                        Allow from all
                        Options -Indexes
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```
* Enable the virtual host using `sudo a2ensite itemCatalog`
* Create the `.wsgi` File to serve the app by running `cd /var/www/itemCatalog` and `sudo nano /var/www/itemCatalog/itemCatalog.wsgi` and save with:
    ```py
    activate_this = '/var/www/itemCatalog/itemCatalog/venv/bin/activate_this.py'
    execfile(activate_this, dict(__file__=activate_this))

    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, '/var/www/itemCatalog')

    from itemCatalog import app as application

    application.secret_key = ENTER-YOUR-KEY
    ```
* Restart apache `sudo service apache2 restart`

#### Setup the DB schema and populate

* Activate the virtual environment in the project folder by running `cd /var/www/itemCatalog/itemCatalog` and `. venv/bin/activate`
* Then run `python database_init.py` to populate the database.
* Restart apache `sudo service apache2 restart`

##### Visit the URL using your favorite browser

* Track errors in `/var/log/apache2/error.log`

## Third Party Resources


* [How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* Google and Facebook Authentication API
* Software and Libraries mentioned above.
