Deploying Python Web App to Linux server
========================================
This file explains the steps to setup ubuntu server and run a python application using flask framework with postgresdb.

Using Amazaon Web Services (lightsail), I have created ubuntu instance (18.04 LTS - Bionic) as below:
https://lightsail.aws.amazon.com/ls/webapp/create/instance?region=us-east-1
click create instance:
Platoform: Linux/Unix
Blueprint: OS only + Ubuntu 18.04 LTS
Instance identity: fsnd-linux

I have created a static IP and linked it to the instance created.

now, from the shell, I logged in to the server using ubuntu user with private key I have downloaded from Amazon as below.
ssh -i LightsailDefaultKey-us-east-1.pem ubuntu@52.205.222.51 -p 22 

Update all currently installed packages:
=======================================
	sudo apt-get update
	sudo apt-get upgrade

Configure the local timezone to UTC:
====================================
	sudo dpkg-reconfigure tzdata
select None of the above 
then select UTC

Crete bionic user:
==================
- Create the bionic user
	sudo adduser bionic
	sudo ls /etc/sudoers.d
- To give the user the root privillegs. I created new file under the sudoers.d directory
	sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/bionic
- Edit the file and change the user name
	sudo nano /etc/sudoers.d/bionic
- Using ssh-keygen, I have generated the bionic.pub file
	as c/Users/.ssh/bionic

	cd /home/bionic
	sudo mkdir .ssh
	cd .ssh
	sudo touch authorized_keys
	sudo nano authorized_keys
I have copied the contents of the pub file gnerated from the ssh-keygen to the authorized_keys file 
	sudo chmod 700 .ssh
	sudo chmod 644 authorized_keys

ssh -i bionic bionic@52.205.222.51 -p 2200
pass phrase: udacitygrader

Create user grader:
==================
Same stpes used to create the bionic user

ssh -i grader grader@52.205.222.51 -p 2200

Prevent Root Login using SSH:
============================
sudo nano /etc/ssh/sshd_config

Locate the following line:
PermitRootLogin yes
Modify the line as follows:
PermitRootLogin no
AllowUsers bionic grader

Lastly restart the ssh service
sudo service ssh restart

change the port for ssh to 2200
================================

edit the file sshd_config and locate the line port 20
sudo nano /etc/ssh/sshd_config
uncommented the line and change the port from 22 to 2200

Configuring the Uncomplicated Fire Wall (UFW)
============================================

sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 2200
sudo ufw allow 80
sudo ufw allow 123

sudo ufw enable

sudo ufw status to make sure of all the changes to the firewall took place

Postgres DB:
============
to install the dataabse, run the command 
 sudo apt install postgresql postgresql-contrib

switch to Postgresdb using:
	sudo -i -u postgres
type 
	psql
Now, create a new database user with least privilleges.
 	createuser catalog -D -S -R -e
 	alter user catalog with encrypted password 'catalogdb'
 	Createdb catalog
 - Grant privilleges to catalogdb to catalog user
 	GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog

 	login to the catalog:
 	psql -U catalog


Deloying catalog Porject to the server:
=======================================
- Create a folder catalog under the /var/www directory
cd /var/www
sudo mkdir catalog

cd catalog
- create a nother directory under catalog
	cd catalog
	sudo mkdir catalog
- connect to the remote github repository to pull the project files

Git:
====

- Install git
	sudo apt -y install git
- connect to the remote rpository and pull the code
	git remote origin https://github.com/fatima-awwami/catalog.git 
	git pull origin master

Apache2 server:
===============
- Install the apache server
	sudo apt-get apache2
- Starting the server:
	sudo /etc/init.d/apache2 start

Virtualenironment:
=================
1- Install pyhton
	sudo apt-get python3
2- Install virtual environment for Python3
	sudo apt-get install python3-venv

3- Create a virtual enviroment. This has the benefit of installing the dependecies for each application in it own env.
	sudo python3 -m venv Virtenv

4- activate the environment
	source Virtenv/bin/activate
5- install Apache WSGI module for python3
	sudo python3 -m pip install libapache2-mod-wsgi-py3


Installing Dependencies:
===========================
after activating the environment install all the dependecies.
- sudo python3 -m pip install sqlalchemy
- sudo python3 -m pip install flask
- sudo python3 -m pip install psycopg2-binary
- sudo python3 -m pip install Pillow
- sudo -H python3 -m pip install  oauth2client

After installing all the required programs and pakcages:

- Create a wsgi file under the /var/www/catalog as below
	cd /var/www/catalog
	sudo touch flaskapp.wsgi
	sudo nano flaskapp.wsgi
- Paste the below code
	import sys
	import logging
	logging.basicConfig(stream=sys.stderr)
	sys.path.insert(0,"/var/www/catalog/catalog")
	from application import app as application
	application.secret_key = 'FX6G1BEBLBO90WP61SS0TMHPDOG9F9FM'

- Create apache config file which will interact with the above file when the app is requested as an http
	sudo touch /etc/apache2/sites-available/FlaskApp.conf
	sudo nano /etc/apache2/sites-available/FlaskApp.conf

	<VirtualHost *:80>
	        ServerName catalog.chickenkiller.com
	        WSGIScriptAlias / /var/www/catalog/flaskapp.wsgi
	        WSGIPassAuthorization On
	        <Directory /var/www/catalog/catalog/>
	                Order allow,deny
	                Allow from all
	        </Directory>
	        Alias /static /var/www/catalog/catalog/static/
	        <Directory /var/www/catalog/catalog/static/>
	                Order allow,deny
	                Allow from all
	        </Directory>
	        ErrorLog ${APACHE_LOG_DIR}/error.log
	        # e.g. LogLevel debug
	        LogLevel debug
	        CustomLog ${APACHE_LOG_DIR}/access.log combined
	</VirtualHost>


	Below changes in the application files were made to run:
1- changing the database from sqlite to Postgresdb
effected files:
	application.py
	seeder.py
	catalogdb_setup.py

Code change:
Only one required change that is the engine creation as below

POSTGRES = {
    'user': 'catalog',
    'pw': 'catalogdb',
    'db': 'catalog',
    'host': 'localhost',
    'port': '5432',
}


engine = create_engine('postgresql://%(user)s:\
%(pw)s@%(host)s:%(port)s/%(db)s' % POSTGRES)

2- Change the path to static and Client_secret.josn files to be absolute

Efected files:
	application.py

	UPLOAD_FOLDER = '/var/www/catalog/catalog/static/image'
	APP_PATH = '/var/www/catalog/catalog/'

3- remove the debug statment:
Effected files:
	application.py

    app.debug = True
4- change the run command as below:
Effected files:
	application.py

	app.run(host='0.0.0.0', port=5000)
	to 
	app.run()
5- In order to save or delete from the image folder under the static directory, I change the permission for the entire directory as below:
	sudo chmod -R 757 /var/www

6- Google Oauth configuration:
Aside from installing the library to run it, I have made some changes to the Oauth console.
First:
I have configured a DNS as a subdomain under chickenkiller.com (catalog.chickenkiller.com) which points to Ubuntu instance static IP: 52.205.222.51
Website: http://freedns.afraid.org
The same was used in the Oauth settings:
Authorized redirect URIs: http://catalog.chickenkiller.com/oauth2callback
Authorized JavaScript origins: http://catalog.chickenkiller.com
I have downloaded the new JSON client secret and used the same in the server.

7- Login html template:
I have changed the new reidrect location to be as below:
	window.location.href = "http://catalog.chickenkiller.com/"

8- to prevent access to the .git folder:
I have created .htaccess under the .git directory and  added the below code
	Order allow,deny
	Deny from all


Steps taken to run the app:
===========================

I have navigated to the project categry.
	cd /var/www/catalog/catalog
- I created the database tables
	python3 catalogdb_setup.py
- I inserted the data
	python3 seeder.py
- Lastly, I ran the application file to make sure the fie runs error free
	python3 application.py
Now, in the browser navigate to:
http://catalog.chickenkiller.com/
