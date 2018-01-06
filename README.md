# Udacity-Linux-Server-Configuration
This is the final project for "Full Stack Web Developer Nanodegree" on Udacity.
In this project, a Linux virtual machine needs to be configurated to support the Item Catalog website.
You can visit http://35.154.168.187/ for the website deployed.
### Note:We need to change the url's and redirect uri's in "client_secrets.json" file. So, make the changes accordingly and download the new file.(Ex: You need to change the port from 0.0.0.0:8000 to http://35.154.168.187/catalog etc)

# Instructions for ssh access into the instance
1. Download Private Key from the SSH keys section in the Account section on Amazon Lightsail.
2. Move the private key file into the folder ~/.ssh (where ~ is your environment's home directory). So if you downloaded the file to the Downloads folder, just execute the following command in your terminal. `mv ~/Downloads/Key.pem  ~/.ssh/`
3. Open your terminal and type in `chmod 400 ~/.ssh/Key.pem`
4. In your terminal, type in `ssh -i ~/.ssh/Key.pem ubuntu@35.154.168.187`

# Create a new user named grader
1. `sudo adduser grader`
2. `sudo touch /etc/sudoers.d/grader`
3. `sudo vi /etc/sudoers.d/grader`, type in `grader ALL=(ALL:ALL) NOPASSWD:ALL`, save(`ctrl+x` then `shift+y` and then `enter` and quit.

# Setup Key based authentication
1. Generate keys on local machine using`ssh-keygen` ; then save the private key in ~/.ssh on local machine.(Open another terminal and generate keys)
2. Deploy public key on developement enviroment
On you virtual machine:
`$ su - grader`
`$ mkdir .ssh`
`$ touch .ssh/authorized_keys`
`$ nano .ssh/authorized_keys`
Copy the public key (one with the extension .pub) generated on your local machine to this file and save.
`$ chmod 700 .ssh`
`$ chmod 644 .ssh/authorized_keys`
3. reload ssh using `sudo service ssh restart`
4. now you can use ssh to login with the new user you created
 `ssh -i ~/.ssh/"privateKeyFilename" grader@35.154.168.187`

# Update all currently installed packages
`$ sudo apt-get update`
`$ sudo apt-get upgrade`

# Change the ssh port from 22 to 2200
1. Use `sudo nano /etc/ssh/sshd_config` and then change Port 22 to Port 2200 , save & quit.
2. reload ssh using `sudo service ssh restart`
### Note: Remember to add and save port 2200 with Application as Custom and Protocol as TCP in the Networking section of your instance on Amazon Lightsail.

# Configure Firewall to only allow incoming connections for SSH (port 2200)
HTTP (port 80), and NTP (port 123)
1. Check current UFW status `sudo ufw status` This will show that UFW is Inactive.
2. Set-up ufw with the following commands:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 2200/tcp (Note: Changed SSH above to port 2200)
sudo ufw allow www
sudo ufw allow 123/udp
sudo ufw enable
```
3. Check UFW status after updates `sudo ufw status` This will show that UFW is now active with the settings below:
```
To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere
2200/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
123/udp                    ALLOW       Anywhere
22 (v6)                    ALLOW       Anywhere (v6)
2200/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
123/udp (v6)               ALLOW       Anywhere (v6)
```
4. To disable port 22 use the command `sudo ufw deny 22`
5. Confirm that root can SSH and login from local computer, `ssh -i ~/.ssh/"PrivateKeyFileName" -p 2200 grader@35.154.168.187` 
If yes, hooray we can proceed. If not, repeat the steps above since you are locked out of the server.
# Configuring local time-zone to UTC
1. Configure the time zone `sudo dpkg-reconfigure tzdata`
2. Press `None of the Above` and then select `UTC`
# Install and configure Apache to serve a Python mod_wsgi application
1. Install Apache `sudo apt-get install apache2`
Confirm successful installation by visiting http://35.154.168.187/. It should say "It Works" and display other Apache information on the page. 
2. Install Python mod_wsgi `sudo apt-get install libapache2-mod-wsgi` 
3. Install and Configure Demo WSGI app `sudo nano /etc/apache2/sites-enabled/000-default.conf`
4. At the end of the <VirtualHost *:80> block, right before the closing add this line: 
`WSGIScriptAlias / /var/www/html/myapp.wsgi`
5. Restart Apache `sudo service apache2 restart`
NOTE: After restart the Home page will return a 404 error which we will fix by configuring Apache to serve WSGI application
#  Configure Apache to serve basic WSGI application to confirm installation of Apache and mod_wsgi
1. Create the file /var/www/html/myapp.wsgi as 
`sudo nano /var/www/html/myapp.wsgi`
2. Within this file, write the following application
```
def application(environ, start_response):     
	status = '200 OK'     
	output = 'Hello World!'      
	response_headers = [('Content-type', 'text/plain'), ('Content-Length', str(len(output)))] 
	start_response(status, response_headers)      
	return [output] 
```
3. Refresh the page and the text in the script above will be displayed
# Install and configure postgresql
1. Install PostgreSQL 
`sudo apt-get install postgresql postgresql-contrib`
2. Check that remote connections are not allowed 
`sudo less /etc/postgresql/9.5/main/pg_hba.conf` 
3. By default, remote connections to the database are disabled for security reasons when installing PostgreSQL from the Ubuntu repositories.
4. Basic server set-up 
`sudo -u postgres psql postgres`
5. Set-up a password for user postgres 
`\password postgres` and enter a password
# Create a new database user named with limited permissions to the database
1. Connect to database as the user postgres `sudo su - postgres`
2. Type `psql` to generate PostgreSQL prompt
3. Create a new user 
`CREATE USER catalog WITH PASSWORD 'password';`
4. Confirm that the user was created 
`\du`
# Limit permissions to new database user
1. Run `\du` to see what permissions the user catalog has
2. To see possible user roles, type: `\h CREATE ROLE`
3. Update permissions for catalog user: 
`ALTER ROLE catalog WITH LOGIN;`
`ALTER USER catalog CREATEDB;`
4. Create the database 
`CREATE DATABASE catalog WITH OWNER catalog;`
5. Login to the database 
`\c catalog`
6. Revoke all rights `REVOKE ALL ON SCHEMA public FROM public;`
7. Grant only access to the catalog role 
`GRANT ALL ON SCHEMA public TO catalog;`
8. Exit out of PostgreSQL and the postgres user 
`\q`, then `exit`
9. Restart postgresql 
`sudo service postgresql restart`
# Installing git
1. Install Git as 
`sudo apt-get install git`
2. Edit Git Configuration
`git config --global user.name "Your Name"`
`git config --global user.email youremail@domain.com`
3. Confirm by running `git config --list`
# Clone Item Catalog project to the AWS instance
1. Create a folder inside the /var/www folder called "catalog" and cd into this folder, see commands below. Remember this is a Python Flask app and not just html.
```
cd /var/www 
sudo mkdir catalog 
cd catalog 
```
2. Clone repo for Udacity Project(Item-Catalog): `sudo git clone  https://github.com/Sidharth1998/Item_Catalog.git catalog` 
3. The project is now at /var/www/catalog/catalog
# Installing Flask and creating Virtual Environment for Item Catalog app
```
sudo apt-get install python-pip
sudo pip install virtualenv
```
1. Give the following command (where venv is the name you would like to give your temporary environment): `sudo virtualenv venv`
2. Now, install Flask in that environment by activating the virtual environment with the following command 
`source venv/bin/activate`
3. Give this command to install Flask inside 
`sudo pip install Flask`
4. Run the following command to test if the installation is successful and the app is running 
`sudo python __init__.py` renamed project.py to init.py 
5. It should display “Running on http://localhost:5000/” or "Running on http://127.0.0.1:5000/". 
If you see this message, you have successfully configured the app.
6. To deactivate the environment, give the following command 
`deactivate`
7. Configure and Enable the new Virtual Host `sudo nano /etc/apache2/sites-available/catalog.conf`
8. Add file contents for VirtualHost configuration: 
`sudo nano /etc/apache2/sites-available/catalog.conf`
```
<VirtualHost *:80>
		ServerName mywebsite.com
		ServerAdmin admin@mywebsite.com
		ServerAlias "YourPublicIPAddress"
		WSGIScriptAlias / /var/www/catalog/flaskapp.wsgi
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
</VirtualHost>
```
9. Enable the virtual host with the following command 
`sudo a2ensite catalog`
`service apache2 reload`
10. Create the .wsgi file
`cd /var/www/catalog`
`sudo nano flaskapp.wsgi`
11. Restart Apache `sudo service apache2 restart`
# Additional packages for the App
1. Go to directory for the catalog app
`cd /var/www/catalog/catalog`
```
sudo apt-get install python-pip
sudo pip install virtualenv
sudo virtualenv venv
Activate the environment 
source venv /bin/activate
```
2. Install App dependencies for Flask and Database
```
sudo pip install Flask
sudo pip install sqlalchemy
sudo pip install Flask-SQLAlchemy
sudo pip install psycopg2
sudo apt-get install python-psycopg2 
sudo pip install flask-seasurf
sudo pip install oauth2client
sudo pip install httplib2
sudo pip install requests
```
3. Configure Virtual Host in Apache `sudo nano /etc/apache2/sites-available/catalog.conf`
4. Enable this Virtual Host `sudo a2ensite catalog` Prompted to run: service apache2 reload to activate the new configuration
5. Configure the WSGI file
```
cd /var/www/catalog
sudo nano catalog.wsgi
```
6. Add this to the file:
```
#!/usr/bin/python 
import sys 
import logging 

logging.basicConfig(stream=sys.stderr) 
sys.path.insert(0,"/var/www/catalog/")  

from catalog import app as application 
application.secret_key = 'super_secret_key'
```
7. Restart Apache `sudo service apache2 restart`
8. Modify the database calls in the catalog app to use PostgreSQL vs. SQLite 
Edit these files: database_setup.py, project.py(or "init.py"), and item.py 
Remove: engine = create_engine(‘sqlite:///catalogsusers.db’, ) 
Add: engine = create_engine('postgresql://catalog:password@localhost/catalog')
9. Use the full path to client_secrets.json(change to "/var/www/catalog/catalog/client_secrets.json") in the project.py(or "init.py") file
# To display our Catalog website
1. Add IP (without the http://) to 'hosts' file 
`sudo nano /etc/hosts`
2. Remove default.conf and catalog.conf from being enabled (extra step since issues with seeing site on AWS)
`sudo a2dissite 000-default.conf`
`sudo a2dissite catalog.conf`
3. Check what sites are enabled 
`ls -alh /etc/apache2/sites-enabled/`
4. Enable the catalog.conf using `sudo a2ensite catalog`
5. Restart Apache 
`sudo service apache2 restart`
6. Restart app 
`sudo python __init__.py`
# Refrences
1. Udacity's FSND Forum
2. https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
3. https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
