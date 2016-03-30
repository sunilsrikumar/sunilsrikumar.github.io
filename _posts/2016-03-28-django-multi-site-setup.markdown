---
layout: post
title:  "Django multi-site setup on ubuntu server."
date:   2016-03-29 09:05:33
categories: Django
description: "How to setup django multisite on ubuntu server with nginx and uwsgi."
keywords: "Nescode technology, Nescode, Django, Django framework, Sunil Kumar, Django wth nginx and uwsgi, django with nginx server, setup django on digital ocean, deploy django on ubuntu server"
tags: Django
comments: true
---

### Overview

Django framework is a leading python framework that can help developer setup a small website to highly concurrent web application. If you would like to test Django locally, it comes with much simplified development server for development but when it comes to production deployment, where you need a powerful and secure web server.

In this blog, I will explain how to install and configure a required packages on Ubuntu to serve django application. For demonstration, I will push local development code to server using github, setup nginx to reverse proxy to uWSGI, setting up domain name and letsencrypt ssl certificate.

### Prerequisites and goals

To complete the guide and successful django setup, you should have a new ubuntu 14.04 virtual machine running with non-root with sudo privileges configured. We will be using a dedicated virtual environment for individual project. This will allow respective project to run and manage requirements successfully. I also assume, you have already configured your local django environment, git and github. You should have an github account if you thinking of releasing your code in open source domain. If your code is in closed source, you should should bitbucket or gitlab. Both are equally good.

Once we have our application ready, we will configure our server to deploy the code, followed with virtual environment, nginx, uWSGI and letsencrypt. Lets start the 5 minute journey of deployment.

### Configure VirutualEnv and VirutualEnvWrapper

We will be installing our django project in their own virutal environment(python version 2.7) to isolate the requirements for each. To do this, lets install required packages of virtual environment.

```python
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install python-pip
sudo apt-get install python-dev
sudo pip install virtualenv virtualenvwrapper
sudo apt-get install git
```
With these installed components, now you can configure shell with the information it needs to work with the virtualenvwrapper script. We will place our virtual environment within home directory called "Env". This is configured through and environment variable called WORKON_HOME. We can add this to our shell initialization script and can source the virtual environmentwrapper script.

```python
echo "export WORKON_HOME=~/Env" >> ~/.bashrc
echo "source /usr/local/bin/virtualenvwrapper.sh" >> ~/.bashrc
source ~/.bashrc
```

This will create a folder named "Env" inside your home directory where virtual environment information will be stored. Now, create virtualenv, name could be anything but as per convention name as project, this will help you refer easily.

```python
mkvirtualenv portalone
```

### Pull code from github or bitbucket

Since virtualenv is ready, pull your code from github or bitbucket and then install required packages within activated virtualenv. I assume, you have freeze your project requirement in requirements.txt file. If not you should freeze first and then update pull git repo:

```python
pip freeze > requirements.txt
```

```python
git clone https://github.com/sunilsrikumar/portalone
cd portalone
pip install -r requirements.txt
```

As per industry standard practice, I assume, you haven't uploaded your database file on git repositories. Now, you can migrate and create superuser as required.

```python
python manage.py migrate
python manage.py createsuperuser
```

This will make your app ready and admin user for use. If you are using PostgreSQL or MySQL, then first you need to install required database package along with necessary settings in settings.py.
Since, we will be configuring Nginx to serve our site, we need to configure a directory which will hold our site's static assets. We will setup django to place these into a directory called static in our project base directory. Add this line to the bottom of the settings.py. You can modify settings.py using vim.

```python
sudo vi /portalone/settings.py
```
Now add this line in bottom

```python
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
```

Exit vim editor

```python
esc # Press esc button
:wq # type this to save and exit from vim
```

Now, collect static files used in project and place in static folder. Static folder will get created automatically when you will run this command. Type yes, when terminal will prompt to do so.

```python
python manage.py collectstatic
```

It's time to test whether project is running inside virtualenv or not. Run below command within virtualenv.

```python
python manage.py runserver 0.0.0.0:8000
```

This will startup the development server on port 8000. Visit your server's domain name or IP address followed by 8000 port in your browser. If this is not working then check whether 8000 port is open in your virtual machine or not. If you are using digital ocean, scaleway, google cloud; its open by default. If you are using Amazon AWS, Microsoft azure, need to open this network port manually.

After testing the development server, stop the server by typing CTRL-C in your terminal. Now, we can move on to setup uWSGI setup. Let's deactivate your virtualenv and install uWSGI globally:

```python
sudo apt-get intall uwsgi
```
Since uWSGI is installed now, we can test this application server by passing it the information for first project.

```python
uwsgi --http :8000 --home /home/user/Env/portalone --chdir /home/user/portalone -w portalone.wsgi
```

Here, we have told uWSGI to use our virtual environment located in our ~/Env directory, to change to our project directory, and to use the wsgi.py file stored within our inner portalone directory to serve the file. For demonstration, we setup to serve HTTP on port 8000. If you go to server domain name or IP address in your browser, followed by :8000, you should see your site again. Now, like before close the server by typing CTRL-C in ther terminal.

### Configure uWSGI file for site

Running uWSGI from the command line is useful for testing, but it's not for production deployment. Instead, we will run uWSGI in "Emperor mode", which allows a master process to manage separate application automatically given a set of configuration files.

Now, we will create a directory which will hold the configuration files. Since, this is a global process, we will create a directory call /etc/uwsgi/sites to serve our configuration files.

```python
sudo mkdir -p /etc/uwsgi/sites
cd /etc/uwsgi/sites
sudo vi portalone.ini
```

Paste this information with requried changes. Wherever, I have used a term "user", you should use your own server user name.

```python
[uwsgi]
project = portalone
base = /home/user

chdir = %(base)/%(project)
home = %(base)/Env/%(project)
module = %(project).wsgi:application

master = true
processes = 5

socket = %(base)/%(project)/%(project).sock
chmod-socket = 664
vacuum = true
```

With this uWSGI configuration file, we have completed our first site. Save and close the vi editor as I explained earlier.

The advantage of setting up uWSGI file with variable is that it makes it incredibly simple to reuse for another site which we are going to setup after this.

### Configure an upstart script for uWSGI

To automate the process, we need to configure an upstart script. We will create an upstart script to automatically start uWSGI at boot. Let's create inside /etc/init directory.

```python
sudo vi /etc/init/uwsgi.conf
```

Paste the following script with required changes

```python
description "uWSGI application server in Emperor mode"

start on runlevel [2345]
stop on runlevel [!2345]

setuid user
setgid www-data

exec /usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
```

Now save and close the vi editor. We will setup nginx as final steps.

### Configure Nginx as a reverse proxy

Since, our uWSGI configured and ready to go, we can configure nginx as our reverse proxy. First install nginx and create a server block for first site.

```python
sudo apt-get install nginx
sudo vi /etc/nginx/sites-available/portalone
```

Paste the server block code with required changes. In server_name, your should use your server domain name or IP address.

```python
server {
    listen 80;
    server_name portalone.com www.portalone.com;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/user/portalone;
    }

    location / {
        include         uwsgi_params;
        uwsgi_pass      unix:/home/user/portalone/portalone.sock;
    }
}
```

This is basic server block which we need. As I mentioned earlier about SSL setup, will configure SSL block in later stage. Now remove the default nginx file and make a symlink

```python
sudo rm -rf /etc/nginx/sites-available/default
sudo rm -rf /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/portalone /etc/nginx/sites-enabled/portalone
```

It's time to test nginx server block and restart nginx

```python
sudo service nginx configtest
sudo service nginx restart
sudo service uwsgi start
```

Now, you should be able to access your site on respective server domain name or IP address.

### Setup another site on same server

It's time to setup another setup on same server. Pull another project in side home directory followed by dedicated virtualenv and required package installation within virtualenv

```python
git clone https://github.com/sunilsrikumar/portaltwo
cd portaltwo
mkvirtualenv portaltwo
pip install -r requirements.txt
```
Add the location of static file in settings.py as suggested for portalone. Run migrate and create superuser if required.

```python
python manage.py migrate
python manage.py createsuperuser
```

Deactivate the virtual environment and create uWSGI configuration for portaltwo. Modify as per portaltwo.

```python
sudo cp /etc/uwsgi/sites/portalone.ini /etc/uwsgi/sites/portaltwo.ini
sudo vi /etc/uwsgi/sites/portaltwo.ini
```

Edit as required

```python
[uwsgi]
project = portaltwo
base = /home/user

chdir = %(base)/%(project)
home = %(base)/Env/%(project)
module = %(project).wsgi:application

master = true
processes = 5

socket = %(base)/%(project)/%(project).sock
chmod-socket = 664
vacuum = true
```

Now, copy portalone nginx server block and modify for portaltwo

```python
sudo cp /etc/nginx/sites-available/portalone /etc/nginx/sites-available/portaltwo
sudo vi /etc/nginx/sites-available/portaltwo
```

Modify portaltwo server block

```python
server {
    listen 80;
    server_name portaltwo.com www.portaltwo.com;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/user/portaltwo;
    }

    location / {
        include         uwsgi_params;
        uwsgi_pass      unix:/home/user/portaltwo/portaltwo.sock;
    }
}
```
Make a symlink for portaltwo followed by nginx restart

```python
sudo ln -s /etc/nginx/sites-available/portaltwo /etc/nginx/sites-enabled/portaltwo
sudo service nginx restart
```

It's time to browse through your server name or IP address. You will see your second site up and runnning.

### Setup letsencrypt SSL Certificate

Before, going with further steps, one things need to be ensure that you should have control on registered domain name that you wish to use the Certificate with. In case you don't have any domain, you may choose Godaddy domain registrar.

Create a A Record with respective domain that points your domain to the public IP address of your server. This is mandatory because letsencrypt will validate your domain ownership before issusing a certificate. Stop nginx before installing certificate.

```python
sudo service nginx stop
```

### Install letsencrypt client

The best way to install letsencrypt is to simply clone it from their official github repositories. Use your admin email id to generate Certificate.

```python
sudo apt-get -y install git bc
sudo git clone https://github.com/letsencrypt/letsencrypt /opt/letsencrypt
cd /opt/letsencrypt
sudo ./letsencrypt-auto certonly --standalone --email admin@example.com
```

This will ask you to enter domain name. Give your domain name with comma and space (ex. example.com, www.example.com). Accept their terms and condition. If everything is alright, you should see a successful message on terminal and your certificate will be stored in your example.com domain directory.

```python
/etc/letsencrypt/live/example.com
sudo ls -l /etc/letsencrypt/live/example.com
```
Create a log directory where uWSGI should store access and error log.

```python
cd ~/home/user
mkdir logs
cd logs
mkdir portalone
cd portalone
touch uwsgi-access.log
touch uwsgi-error.log
```

Edit your site nginx server block with required information.

```python
sudo vi /etc/nginx/sites-available/portalone
```

Your server block should look like this

```python
server {
    listen 443 ssl;
    server_name portalone.com www.portalone.com;
    client_max_body_size 4G;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    access_log /home/user/logs/portalone/uwsgi-access.log;
    error_log /home/user/logs/portalone/uwsgi-error.log;

    # intermediate configuration. tweak to your needs.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;

    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;

    ssl_stapling_verify on;
    add_header Strict-Transport-Security max-age=15768000;
    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;

    location /static/ {
        root /home/user/portalone;
    }

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/user/portalone/portalone.sock;
    }
    location /robots.txt {
            root            /home/user/portalone/;
            access_log      off;
            log_not_found   off;
        }

        location /favicon.ico {
            root            /home/user/portalone/static;
            access_log      off;
            log_not_found   off;
        }

    }

    server {
        listen 80;
        server_name portalone.com www.portalone.com;
        return 301 https://$server_name$request_uri;
    }
```

If there are no type error in your server block, you should be able to restart your nginx successfully.

```python
sudo service nginx restart
```

Browse your domain name in any browser. You will see HTTPS. Thats it. Probably, this is one of the most detailed and complete guide you have followed. Take a break and have a Coffee. Feel free to drop a comment below in case you stuck anywhere. Happy coding.
