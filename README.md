# FSND Linux Server Configuration Project

## Description

This is a project of the Udacity Full Stack Web Development program, which configures Linux Ubuntu server for hosting web applications. 

## setting

* IP address: 35.156.240.140
* SSH port: 2200
* App URL: http://35.156.240.140/ or http://ec2-35-156-240-140.eu-central-1.compute.amazonaws.com

disclaimer: 

## Amazon Lightsail
You are recommended to create an instance with [Amazon Lightsail Service](https://amazonlightsail.com) to get a publicly accessabible Ubuntu Linux Server.

### Preparation
    
    p.1. Download SSH private key from [here](https://lightsail.aws.amazon.com/ls/webapp/account/keys)
    p.2. Move the key file to the local ssh folder (and rename the file, if preferred)
    p.3. Set the file permission
    ```$ chmod 600 ~/path/(downloaded_key)```
    p.4. Connect to the remote server
    ```$ ssh ubuntu@(your_public_ip_address) -i ~/path/(downloaded_key)```
    
## Configuration

### 0. Make all software up-to-date
    
    0.1. Check Avavilable Package Lists
    ```$ sudo apt-get update```
    
    0.2. Update Installed Packages
    ```$ sudo apt-get upgrade```
    
    0.3. Remove unnecessary Packages
    ```$ sudo apt-get autoremove```
    
### 1. Set the FireWall
    
    1.1 Change the default SSH Port
    ```$ sudo vim /etc/ssh/sshd_config```
    Change the value of `Port`: `22` => `2200`
    ```$ sudo service ssh restart```
    
    1.2. Configure the firewall
     * Default Rules
    ```$ sudo ufw default deny incoming
       $ sudo ufw default allow outgoing
    ```
    * Port Rules (SSH, HTTP and NTP ports)
    ```$ sudo ufw allow ssh
       $ sudo ufw deny 22
       $ sudo ufw allow 2200/tcp
       $ sudo ufw allow 80/tcp
       $ sudo ufw allow 123/udp
    ```
    * Activate configuration
    ```$ sudo ufw enable```
    apply the same configurate in the Lightsail site as well.
    
    
### 2. Create a new user `grader` with sudo privileges

    2.1. Create a new user
    ```$ sudo adduser grader```  

    2.2. Give Sudo Access to the user `grader`
    ```$ sudo visudo```
    add the following line right after the `root` line.
    ```grader ALL=(ALL) ALL ```
    

### 3. Generate SSH key pair 
       
    2.1 Generate encrypted key pair in the local machine
    ```$ ssh-keygen```
    your private key will be saved in `/Users/(your_computer_home)/.ssh/(key_type)`
    
    2.2 Save a public key
    ```/Users/(your_computer_name)/.ssh/(your_chosen_name)```
    your public identification will be saved here.
    
    2.3 Place a public key into the `grader`'s home directory
    ```$ sudo mkdir /home/grader/.ssh
       $ sudo touch /home/grader/.ssh/authorized_keys```
       
    paste the value of the public key
    ```$ sudo vim /home/grader/.ssh/authorized_keys```
    
    give grader ownership of the .ssh directory
    ```$ sudo chown grader:grader /home/grader/.ssh```
    
    change the permissions so that only grader can access the contents
    ```$ sudo chmod 700 /home/grader/.ssh```
    
    change the ownership of the authorized_keys file:
    ```$ sudo chown grader:grader /home/grader/.ssh/authorized_keys```
    
    set the file permission
    ```$ sudo chmod 600 /home/grader/.ssh/authorized_keys```


    
### 3. Force key-based authentication 

    3.1. Disable Password Based Login
    ```$ sudo vim /etc/ssh/sshd_config```
    *`PasswordAuthentication`: `yes` => `no`
    
    3.2. Disable remote login of root user
    ```$ sudo vim /etc/ssh/sshd_config```
    *`PermitRootLogin`: `yes` => `no`
    
    3.3. Enable grader user remote ssh login
    ```$ sudo vim /etc/ssh/sshd_config```
    * add a line `AllowUsers grader`
    
    3.2. Restart SSH
    ```$ sudo service ssh restart```
    

    
### 5. Install Web Server
    
    5.1. Get your server responding to HTTP requests
    ```$ sudo apt-get install apache2```
    
    5.2. Configure Apache to hand-off certain requests to an application handler.
    ```$ sudo apt-get install apache2 libapache2-mod-wsgi python-dev```


### 6. Set up Database Server

    6.1. Install PostgreSQL
    ```$ sudo apt-get install postgresql```
    
    6.2. Connect to psql 
    ```$ sudo su - postgres
       $ psql
    ```
    6.3. Create a new user `catalog` with right to create database
    ```CREATE ROLE catalog WITH LOGIN;
       ALTER ROLE catalog CREATEDB;
    ```
    6.4. Set password for user `catalog`
    ```\password catalog
       \q
       exit
    ```
    6.5. Create a new Linux user `catalog` with a new database
    ```$ sudo adduser catalog
       $ sudo visudo
    ```
    add a line `catalog ALL=(ALL:ALL) ALL` after the `root` line
    create a new database (`catalog` user) `$ createdb catalog`
    
    6.6. switch the database from SQLite to PostgresSQL
    ```engine = create_engine('postgresql://catalog:'your_password'@localhost/catalog')

### 7. Clone a catalog app from github

    7.1. Install Git, if needed
    ```$ sudo apt-get install git```
    
    7.2. Clone the web app to the server's `catalog` folder
    ```$ sudo cd /var/www/catalog
       $ sudo git clone https://github.com/ayrka39/FSDN-4Catalog.git catalog
    ```
    7.3 Modify client_secrets.json file for google authentication
    
### 8. Set up virtual environment

    8.1. setup virtual environment
    * install pip `$ sudo sudo apt-get install python-pip`
    * install virtualenv `$ sudo apt-get install python-virtualenv`
    * create a new one in /var/www/catalog/catalog `$ virtualenv venv`
    * activate it `$ . venv/bin/activate`
        
    8.2. Install packages and dependencies 
    ```$ pip install httplib2 oauth2client sqlalchemy requests psycopg2 flask
       $ sudo apt-get install libpq-dev
       $ deactivate
    ``` 
    
    8.3. Configure virtual host in Apache
    * create a file /etc/apache2/sites-available/catalog.conf
	```
	<VirtualHost *:80>
			ServerName your_public_ip_address
			ServerAdmin your_email_address
			WSGIScriptAlias / /var/www/catalog/catalog.wsgi
			<Directory /var/www/catalog/catalog/>
				Order allow,deny
				Allow from all
				Options -Indexes
			</Directory>
			Alias /static /var/www/catalog/catalog/static
			<Directory /var/www/catalog/catalog/static/>
				Order allow,deny
				Allow from all
				Options -Indexes
			</Directory>
			ErrorLog ${APACHE_LOG_DIR}/error.log
			LogLevel warn
			CustomLog ${APACHE_LOG_DIR}/access.log combined
	</VirtualHost>
	```
	* enable the virtual host
	```$ sudo a2ensite catalog
	   $ sudo service apache2 reload
	```
### 9. Get a flask app work

	9.1. create a .wsgi file in /var/www/catalog directory
	```
	 activate_this = '/var/www/catalog/catalog/venv/bin/activate_this.py'
	 execfile(activate_this, dict(__file__=activate_this))

	 #!/usr/bin/python
	 import sys
	 import logging
	 logging.basicConfig(stream=sys.stderr)
	 sys.path.insert(0,"/var/www/catalog/")

	 from nuevoMexico import app as application
	 application.secret_key = 'your_secret_key'
	```
	9.2. restart Apache
	```$ sudo service apache2 restart```
	

### 10. Populate the database
    10.1 Activate the virtualenv in /var/www/catalog/catalog directory
    ```$ . venv/bin/activate
       $ python database_init.py
       $ deactivate
    ```
    10.2. Restart Apache
    ```$ sudo service apache2 restart```
    
    10.3. Run an app in the browser
    

## References

* Udacity Forum
* digitalocean.com
* askubuntu.com
* linuxacademy.com
* [bencam]’s github(https://github.com/bencam)
