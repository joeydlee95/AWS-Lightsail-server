# Lightsail Amazon Server

## Introduction
public IP: 52.34.112.151
Server Alias Name: ec2-52-25-0-41.us-west-2.compute.amazonaws.com

## Getting Started With Amazon Lightsail
1. Go to https://amazonlightsail.com/ and you can click the "Get started for free" where you get asked to log in.
2. Create a Ubuntu instance (OS only).
3. Choose a plan (choosing the lowest plan with get you free-tier access for a month).
4. Create a hostname.
5. You now have an instance running (you might need to wait a bit for it to be operational).

## Initial server updates
1. Click on your instance and click "Connect using SSH". (logged in as ubuntu)
2. You will be logged in as ubuntu.
3. Update available packages: `$ sudo apt-get update`
4. Install updated list: `$ sudo apt-get upgrade`

## Create a new user
1. Create a new user named grader: `$ sudo adduser grader`
2. Choose whatever password you want (We will be disabling this feature). Then, you will be asked for addition information which you can just press enter (optional).
3. We must now give this user sudo power. Use the following commands:
  - `$ sudo cp /etc/sudoers.d/ubuntu /etc/sudoers.d/grader`
  - `$ sudo vi /etc/sudoers.d/grader`
    Replace `ubuntu` with `grader`.

## As grader, force Key-Based Authentication
Source: https://askubuntu.com/questions/53553/how-do-i-retrieve-the-public-key-from-a-ssh-private-key
        http://www.hypexr.org/linux_scp_help.php
1. As ubuntu, use `$ sudo su - grader` to switch the user to grader.
2. Go to https://lightsail.aws.amazon.com/ls/webapp/account/keys and create a new key pair.
3. In your local machine, you want to use `$ ssh-keygen -y -f {file location of downloaded private key} > {public key file}`
  - Copy that file into the user grader home directory `scp {public key file} grader@{ip_address}:/home/grader/` (you may need to use your password that you made for grader)
4. As grader, in the home directory `$ mkdir .ssh`
  - `$ cd .ssh`
  - `$ touch authenticated_keys`
  - `$ mv ../{public key file} authorized_keys`
  - you may need to use sudo for these and if you do you must change the ownership of the files (use chown) to grader.
  - Then, set file rights, in home directory: (you might need to change these permissions as well if you get an error)
  `$ chmod 700 .ssh`
  `$ chmod 644 .ssh/authorized_keys`
5. Disable password based login
  - `$ sudo vi /etc/ssh/sshd_config` and change `PasswordAuthentication yes` to `PasswordAuthentication no`
  - restart the ssh service: `$ sudo service ssh restart`
6. Now you must use key pair to login. In local machine, in the directory with the .pem file you can login to the lightsail server:
  `ssh -i {.pem file} grader@{ip_address}`

## Disabling ubuntu user
Source: https://mediatemple.net/community/products/dv/204643810/how-do-i-disable-ssh-login-for-the-root-user
1. Change `PermitRootLogin yes` to `PermitRootLogin no` with
  `$ sudo vi /etc/ssh/sshd_config`
  `$ sudo service ssh restart`

## Enabling firewall
1. Change port # in config file to listen to 2200.
  `$ sudo vi /etc/ssh/sshd_config` - Change the `PORT 22` to `PORT 2200`
  `$ sudo service ssh restart`
1. Allow listening in port 2200: `$ sudo ufw allow 2200/tcp` 
2. Allow http in port 80: `sudo ufw allow 80/tcp`
3. Allow ntp in port 123: `sudo ufw allow 123/udp`
4. Enable firewall: `sudo ufw enable`
5. Check the status: `sudo ufw status`
6. Change the settings in Networking tab Firewall in the Lightsail AWS website for HTTP, custom udp port: 123, custom tcp port:2200.
7. Now, the firewall is enabled and you must ssh with `-p 2200`.

## Getting apache2, mod_wsgi, and git
1. To get apache2: `$ sudo apt-get install apache2`
2. To get mod_wsgi: `$ sudo apt-get install libapache2-mod-wsgi`
3. To get git: `$ sudo apt-get install git`
  - setting your name, email:
    `$ git config --global user.name "YOUR NAME"`
    `$ git config --global user.email "YOUR EMAIL ADDRESS"`
4. Enable mod_wsgi:
  `sudo a2enmod wsgi`

## Setting up Flask Application
Source: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
1. Go into apache2 directory and create a catalog directory
   `$ cd /var/www`
   `$ mkdir catalog`
   `$ cd catalog`
   `$ git clone https://github.com/joeydlee95/One-Stop-Playlists catalog`
   `$ cd catalog`
   `$ touch __init__.py`
2. Add to __init__.py:
  `from flask import Flask
   app = Flask(__name__)
   @app.route("/")
   def hello():
     return "Hello, I love Digital Ocean!"
   if __name__ == "__main__":
     app.run()`
3. Get Flask and set up virtual env
  '$ sudo apt-get install python-pip'
  - You should also run the command to update pip
  '$ sudo pip install virtualenv'
4. In the catalog/catalog directory: create the environment in the directory 
  '$ sudo virtualenv venv'
5. Activate the virtual environment:
  '$ source venv/bin/activate'
6. Install Flask:
  'sudo pip install Flask'
7. The app should work with:
  'sudo python __init__.py'
8. Deactivate the env with:
  'deactivate'

## Setting up the config file for apache
1. You must change the /etc/apache2/sites-available/catalog.conf with:
`<VirtualHost *:80>
    ServerName 52.34.112.151
    ServerAlias ec2-52-34-112-151.us-west-2.compute.amazonaws.com
    ServerAdmin admin@52.34.112.151
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
      Order allow,deny
      Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
      Order allow,deny
      Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>`
2. Enable virtual host:
  `sudo a2ensite catalog`

## Setting up wsgi file
1. Create the wsgi file in /var/www/catalog/ directory as catalog.wsgi:
`import sys
 import logging
 logging.basicConfig(stream=sys.stderr)
 sys.path.insert(0,"/var/www/catalog/")
 from catalog import app as application`
2. Restart Apache: `sudo service apache2 restart`

## Installing needed modules (in /catalog/catalog/ directory)
1. Go inside `$ source venv/bin/activate`
2. Use pip to install all modules:
  `$ pip install httplib2` - For httplib2 module
  `$ pip install requests` - For requests module
  `$ pip install --upgrade oauth2client` - To use oauth authentication
  `$ pip install sqlalchemy` - To use the python sqlalchemy
  `$ sudo apt-get install python-psycopg2` - To use python Postgresql psycopg

## Setting up database
Source: https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
1. Install PostgreSQL
  `$ sudo apt-get install postgresql postgresql-contrib`
2. Change the `engine` lines in database_setup.py and application.py to:
  `engine = create_engine('postgresql://catalog:{Password}@localhost/catalog');`
3. Rename application.py
  `$ mv application.py __init__.py`
4. Create a user for psql
  `$ sudo adduser catalog {Password}`
5. Change user to postgres:
  `$ sudo su - postgres`
6. Connect to psql:
  `$ psql`
7. Setup up the postgreSQL database:
  `# CREATE USER catalog WITH PASSWORD '{Password}';`
  `# ALTER USER catalog CREATEDB;`
  `# CREATE DATABASE catalog WITH OWNER catalog;`
  `# \c catalog`
  `# REVOKE ALL ON SCHEMA public FROM public;`
  `# GRANT ALL ON SCHEMA public TO catalog;` 
  `# \q`
  `$ exit`
8. Run the database_setup.py:
  `python database_setup.py`


## Setting up OAuth
1. Use http://www.hcidata.info/host2ip.cgi to get your host name
2. Google oauth:
  - https://console.developers.google.com/project
  Go to your application credentials tab and add your public ip and hostname to Authorized JS origins
  - Edit client_secrets.json accordingly.
3. Facebook oauth:
  - https://developers.facebook.com/apps/
  Go to your application and the settings tab
  - Add your public ip to the site URL

## Running the application
1. You may need to edit the file open functions in __init__.py.
2. Move the app.secret_key outside of the if statement.
3. Restart Apache: `sudo service apache2 restart`
4. Run application: open the browser to 52.34.112.151
5. To check for errors: sudo tail -10 /var/log/apache/error.log



## Errors
1. Currently the google sign in is not working properly. In console, after being able to choose a username you get a "500 Internal Error" with traces to the ajax call to google oauth. In /var/www/catalog/ typing "sudo tail -20 /var/log/apache2/error.log" you get a InvalidClientSecretsError: ('Error opening file', 'client_secrets.json', 'No such file or directory', 2). For now ignore google sign in and just sign in with Facebook.