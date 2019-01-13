# Project:  Linux Server Configuration

Linux server configuration, part of the Udacity [Full Stack Web Developer
Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004).

## Description
Set up and secure a Linux server on a virtual machine, install and configure a web and database server to host the catalog application created in Project 5.

## Virtual Server
- Server IP: 34.205.129.202
- Application URL: [http://34.205.129.202.xip.io/](http://34.205.129.202.xip.io/)
- SSH server access port: 2200
- SSH login username: grader

# Setup/Configuration
## Create a new Ubuntu Linux server instance on Amazon Lightsail 

- Login into [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/resources) using an Amazon Web Services account.
- After you login, click `Create instance`. 
- Choose `Linux/Unix` platform, `OS Only` and  `Ubuntu 18.04 LTS`.
- Choose instance plan.
- Keep the default name provided by AWS or rename your instance.
- Click the `Create` button to create the instance.
- Wait for the instance to start up.

## Access the virtual server
- Download the default private key from Amazon Lightsail.
- Copy the private key to `~/.ssh/aws_key`
- Connect to the virtual server using a terminal: `ssh -i ~/.ssh/aws_key  ubuntu@34.205.129.202`

## Update the virtual server
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade

sudu reboot
```
## Create the grader account
``` 
sudo adduser grader 
``` 
use `grader` for the password and choose defaults.

## Give grader sudo access
```
sudo visudo
```
add the following entry
```
grader  ALL=(ALL:ALL) ALL
```
after the entry
```
root    ALL=(ALL:ALL) ALL
```
Save and exit using CTRL+X and confirm with Y

## Create Public/Private key for grader
```
sudo su grader
cd ~
mkdir /home/grader/.ssh
chmod 700 /home/grader/.ssh
ssh-keygen -f grader -N ''
cp grader.pub /home/grader/.ssh/authorized_keys
chmod 644 /home/grader/.ssh/authorized_keys

sudo service ssh restart
```
Copy the content of the file called grader. You can display the content of the file using ` cat /home/grader/grader`


On your local machine
```
nano ~/.ssh/grader_key
```

Paste the conents of grader from the virtual server into this file. 

Save and exit using CTRL+X and confirm with Y

```
chmod 600 ~/.ssh/grader_key
```
Using the terminal connect to the virtual server as grader using ` ssh -i ~/.ssh/grader_key grader@34.205.129.202`

## Disable root login
```
sudo nano /etc/ssh/sshd_config
```
Change `PermitRootLogin without-password` to `PermitRootLogin no` 

Update `PasswordAuthentication` to `PasswordAuthentication no`

Save and exit using CTRL+X and confirm with Y
```
sudo service ssh restart
```

## UTC Timezone
Check the timezone with the date command. This will display the current timezone after the time. If it's not UTC change it using:
```
sudo timedatectl set-timezone UTC
```

## Change the SSH Port from 22 to 2200
Edit the /etc/ssh/sshd_config file with nano
```
sudo nano /etc/ssh/sshd_config
```
Change the port number from 22 to 2200

Save and exit using CTRL+X and confirm with Y.

Restart SSH
```
sudo service ssh restart
```
Close the current port 22 connection: `exit`

On the networking tab of your Amazon Lightsail instance:

- Remove the SHH Option with port 22
- Add custom TCP with port 2200
- Add Custom UDP with port 123

Now connect using 
```
ssh -i ~/.ssh/grader_key -p 2200 grader@34.205.129.202
```

## Configure Firewall (UFW)
Configure the default firewall to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
```
sudo ufw default deny incoming  
sudo ufw default allow outgoing 

sudo ufw allow 2200/tcp         
sudo ufw allow www              
sudo ufw allow 123/udp          

sudo ufw deny 22                
```
Verify firewall rules `sudo ufw status`. You should see the following:
```
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

Enable the firewall `sudo ufw enable`. You should see the following
```
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

## Install Apache
```
sudo apt-get install apache2
```
Allow python applications to run from Apache 
```
sudo apt-get install libapache2-mod-wsgi
sudo service apache2 restart
```
you should see the default apache page at `http://34.205.129.202/`

## Install Python
```
sudo apt install python-pip
```
Install required packages
```
sudo pip install --upgrade Flask SQLAlchemy httplib2 oauth2client requests psycopg2 psycopg2-binary
```

## Install PostgreSQL
```
sudo apt-get install postgresql
```
The installation created a user called `postgres`. Switch to this user
```
sudo su - postgres
```
Open psql, create the catalog database, user, role and grant privileges
```
psql
CREATE DATABASE catalog;
CREATE USER catalog_user;
ALTER ROLE catalog_user WITH PASSWORD 'catalog';
GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog_user;
\q
```

## Deploy Project Source Code
```
sudo mkdir /var/www/catalog/catalog
sudo chown -R grader:grader /var/www/catalog

cd /var/www/catalog/catalog

git clone https://github.com/mwarr3n/fswd-prj-5.git /var/www/catalog/catalog

mv /var/www/catalog/catalog/application.py /var/www/catalog/catalog/__init__.py
```

In your code files replace
```
engine = create_engine('sqlite:///catalog.db')
```
with 
```
engine = create_engine('postgresql://catalog_user:catalog@localhost/catalog')
```

## Create the Apache config file
```
sudo nano /etc/apache2/sites-available/CatalogApp.conf
```
Add the following to `CatalogApp.conf`
```
<VirtualHost *:80>
   ServerName 34.205.129.202 
   ServerAdmin youremail@domain.com
   WSGIScriptAlias / /var/www/catalog/catalogapp.wsgi
   <Directory /var/www/catalog/catalog/>
       Require all granted
   </Directory>
   Alias /static /var/www/catalog/catalog/static
   <Directory /var/www/catalog/catalog/static/>
       Require all granted
   </Directory>
   ErrorLog ${APACHE_LOG_DIR}/error.log
   LogLevel warn
   CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Save and exit using CTRL+X and confirm with Y

Enable the virtual host:
```
sudo a2ensite CatalogApp

sudo service apache2 restart
```

## Create the WSGI file
```
nano /var/www/catalog/catalogapp.wsgi
```
Add the following to `catalogapp.wsgi`
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/")

from application import app as application
```
Save and exit using CTRL+X and confirm with Y

Restart Apache
```
sudo service apache2 restart
```
You should be able to run the application at `http://34.205.129.202.xip.io/`.

Note: If you see the default Apache site then run the following commands:
```
sudo a2dissite 000-default.conf
sudo service apache2 restart
```

## Disable Directory Listing
```
sudo nano /etc/apache2/apache2.conf
```
Remove `Indexes` from the following 
```
<Directory /var/www/>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```
Save and exit using CTRL+X and confirm with Y
```
sudo service apache2 restart
```

## Google OAuth
Go to [Google Cloud Plateform](https://console.cloud.google.com/).

Click `APIs & services` on left menu.

Click Credentials.

Under `OAuth consent screen` add `xip.io` to Authorized domains.

Under OAuth Client ID add the following to authorized JavaScript origins.
- `http://34.205.129.202.xip.io`

Under OAuth Client ID add the following to authorized redirect URIs:
- `http://34.205.129.202.xip.io/callback`
- `https://34.205.129.202.xip.io/callback`

Download the client ID file. Replace client_secretes.json with the new downloaded client file.


## Resources
[Digital Ocean](https://www.digitalocean.com/community/)

[StackOverflow](https://stackoverflow.com/)

[Python](https://www.python.org/)

[Flask](http://flask.pocoo.org) 

[SQLAlchemy](http://www.sqlalchemy.org)

[Google OAuth2](https://developers.google.com/identity/protocols/OAuth2)
