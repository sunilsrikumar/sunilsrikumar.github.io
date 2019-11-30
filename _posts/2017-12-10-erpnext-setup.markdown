---
layout: post
title:  "ERPNext manual installation on ubuntu 16.04 LTS."
date:   2016-03-29 09:05:33
categories: ERPNext
description: "How to install ERPNext on ubuntu 16.04 LTS"
keywords: "Nescode technology, Nescode, ERPNext, Frappe Framework, ERPNext on Ubuntu 16.04 LTD"
tags: ERPNext
comments: true
---

### Overview:
ERPNext is an open source business application that helps you streamline process and operation. In this article we will be installing ERPNext on Ubuntu 16.04 LTS using best practice.

**Step1:** Configure a cloud server with minimum 1GB RAM. Configuration steps depends upon the cloud server provider. I am assuming that creating a server and configuration will be well documented in respective vendor. <br>

**Step2:** Configure server user account

```
adduser username
gpasswd -a username sudo
```

***Step3:*** Copy ssh key from local machine

```
pbcopy < ~/.ssh/id_rsa.pub
su - nescode
mkdir .ssh
chmod 700 .ssh
vi .ssh/authorized_key
```

Paste the ssh key which you copied in step 3 first line.

```
chmod 600 .ssh/authorized_keys
exit
/etc/ssh/sshd_config
```

Search for a line `PermitRootLogin yes` and change to to `PermitRootLogin no`

```
service ssh restart
ssh username@SERVER_IP_ADDRESS
```

We can see if the system has `swap` file configuration by typing:

```
sudo -s
sudo swapon --show
```

***Create a Swap File***

Now that we know our available hard drive space, we can go about creating a swap file within our filesystem. We will create a file of the swap size that we want called swapfile in our root (/) directory.

The best way of creating a swap file is with the fallocate program. This command creates a file of a preallocated size instantly.

Since the server in our example has 1GB of RAM, we will create a 1 Gigabyte file in this guide. Adjust this to meet the needs of your own server:

```
sudo fallocate -l 1G /swapfile
```

We can verify that the correct amount of space was reserved by typing:

```
ls -lh /swapfile
```

***Enabling the Swap File***

Now that we have a file of the correct size available, we need to actually turn this into swap space.

First, we need to lock down the permissions of the file so that only the users with root privileges can read the contents. This prevents normal users from being able to access the file, which would have significant security implications.

Make the file only accessible to root by typing:

```
sudo chmod 600 /swapfile
```

Verify the permissions change by typing:

```
ls -lh /swapfile
```

We can now mark the file as swap space by typing:

```
sudo mkswap /swapfile
```

After marking the file, we can enable the swap file, allowing our system to start utilizing it:

```
sudo swapon /swapfile
```

We can verify that the swap is available by typing:

```
sudo swapon --show
```

We can check the output of the free utility again to corroborate our findings:

```
free -h
```

***Make the Swap File Permanent***

Our recent changes have enabled the swap file for the current session. However, if we reboot, the server will not retain the swap settings automatically. We can change this by adding the swap file to our /etc/fstab file.

Back up the /etc/fstab file in case anything goes wrong:

```
sudo cp /etc/fstab /etc/fstab.bak
```

You can add the swap file information to the end of your /etc/fstab file by typing:

```
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```


***Adjusting the Swappiness Property***

The swappiness parameter configures how often your system swaps data out of RAM to the swap space. This is a value between 0 and 100 that represents a percentage.

We can see the current swappiness value by typing:

```
cat /proc/sys/vm/swappiness
```

For instance, to set the swappiness to 10, we could type:

```
sudo sysctl vm.swappiness=10
```

This setting will persist until the next reboot. We can set this value automatically at restart by adding the line to our /etc/sysctl.conf file:

```
sudo vi /etc/sysctl.conf
```

At the bottom, you can add:

```
/etc/sysctl.conf
vm.swappiness=10
```

***Adjusting the Cache Pressure Setting***

Another related value that you might want to modify is the `vfs_cache_pressure`. This setting configures how much the system will choose to cache inode and dentry information over other data.

Basically, this is access data about the filesystem. This is generally very costly to look up and very frequently requested, so it's an excellent thing for your system to cache. You can see the current value by querying the proc filesystem again:

```
cat /proc/sys/vm/vfs_cache_pressure
```

Again, this is only valid for our current session. We can change that by adding it to our configuration file like we did with our swappiness setting:

```
sudo vi /etc/sysctl.conf
```

### Applicable only if existing ERPNext installation failed

***Step 1:*** Remove frappe folder (from /home) , .bench (from /tmp), mysql & mariadb completely from server.

To remove any trace of mariadb installed through `apt-get`:
(Note: Backup your db if required)

```
sudo service mysql stop
sudo apt-get --purge remove "mysql*"
sudo rm -rf /etc/mysql/
```

and it is all gone. Including databases and any configuration file.

***Step 2:*** Setup the system apt repo and install mariadb (10+ as required by ERPNext)

```
sudo apt-get -y install python-minimal
sudo apt-get -y install python-pip
sudo apt-get -y install software-properties-common
sudo apt-get install build-essential python-setuptools
sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
sudo add-apt-repository 'deb [arch=amd64,i386,ppc64el] http://mirror.jmu.edu/pub/mariadb/repo/10.1/ubuntu xenial main'

sudo apt update -y
sudo apt install -y mariadb-server
```

***Step 3:*** Secure your mariadb installation

```
sudo /usr/bin/mysql_secure_installation
```

During the interactive process, answer questions one by one as follows:

```
Enter current password for root (enter for none):
Set root password? [Y/n]: Y
New password:
Re-enter new password:
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]: Y
Reload privilege tables now? [Y/n]: Y
```

Note: Be sure to replace with your own MariaDB root password.

***Step 4:*** Install nodejs (Due to legecy nodejs dependencies on Ubuntu 16.04, ERPNext setup trigger a compilation error)

```
sudo apt-get -y update
sudo apt-get -y install nodejs
sudo apt-get -y install npm
sudo apt-get -y update && sudo apt-get -y install nodejs-legacy
```

### Redis server installation

```
sudo apt-get -y install build-essential tcl
```

Download and Extract the Source Code
Since we won't need to keep the source code that we'll compile long term (we can always re-download it), we will build in the /tmp directory. Let's move there now:

```
cd /tmp
curl -O http://download.redis.io/redis-stable.tar.gz
tar xzvf redis-stable.tar.gz
cd redis-stable
```

Build and Install Redis
Now, we can compile the Redis binaries by typing:

```
make
```

After the binaries are compiled, run the test suite to make sure everything was built correctly. You can do this by typing:

```
make test
```

This will typically take a few minutes to run. Once it is complete, you can install the binaries onto the system by typing:

```
sudo make install
```

### Configure Redis

Now that Redis is installed, we can begin to configure it. To start off, we need to create a configuration directory. We will use the conventional `/etc/redis` directory, which can be created by typing:

```
sudo mkdir /etc/redis
```

Now, copy over the sample Redis configuration file included in the Redis source archive:

```
sudo cp /tmp/redis-stable/redis.conf /etc/redis
```

Next, we can open the file to adjust a few items in the configuration:

```
sudo vi /etc/redis/redis.conf
```

In the file, find the supervised directive. Currently, this is set to no. Since we are running an operating system that uses the systemd init system, we can change this to systemd:

```
/etc/redis/redis.conf
```

***Note***

If you run Redis from upstart or systemd, Redis can interact with your
supervision tree. Options:
   supervised no      - no supervision interaction
   supervised upstart - signal upstart by putting Redis into SIGSTOP mode
   supervised systemd - signal systemd by writing READY=1 to $NOTIFY_SOCKET
   supervised auto    - detect upstart or systemd method based on
                        UPSTART_JOB or NOTIFY_SOCKET environment variables
 Note: these supervision methods only signal "process is ready."
       They do not enable continuous liveness pings back to your supervisor.
supervised systemd

Next, find the dir directory. This option specifies the directory that Redis will use to dump persistent data. We need to pick a location that Redis will have write permission and that isn't viewable by normal users.

We will use the `/var/lib/redis` directory for this, which we will create in a moment:

```
/etc/redis/redis.conf
```

The working directory.
The DB will be written inside this directory, with the filename specified
above using the 'dbfilename' configuration directive.

The Append Only File will also be created inside this directory.

Note that you must specify a directory here, not a file name.

```
dir /var/lib/redis
```

Save and close the file when you are finished.


###Create a Redis systemd Unit File

Next, we can create a systemd unit file so that the init system can manage the Redis process.

Create and open the `/etc/systemd/system/redis.service` file to get started:

```
sudo vi /etc/systemd/system/redis.service
```

Inside, we can begin the `[Unit]` section by adding a description and defining a requirement that networking be available before starting this service:

```
/etc/systemd/system/redis.service
```
```
[Unit]
Description=Redis In-Memory Data Store
After=network.target
```

In the `[Service]` section, we need to specify the service's behavior. For security purposes, we should not run our service as root. We should use a dedicated user and group, which we will call redis for simplicity. We will create these momentarily.

To start the service, we just need to call the redis-server binary, pointed at our configuration. To stop it, we can use the Redis shutdown command, which can be executed with the redis-cli binary. Also, since we want Redis to recover from failures when possible, we will set the Restart directive to "always":

```
/etc/systemd/system/redis.service
```
```
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always
```

Finally, in the [Install] section, we can define the systemd target that the service should attach to if enabled (configured to start at boot):

```
/etc/systemd/system/redis.service
```
```
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always

[Install]
WantedBy=multi-user.target
```

Create the Redis User, Group and Directories
Now, we just have to create the user, group, and directory that we referenced in the previous two files. Begin by creating the redis user and group. This can be done in a single command by typing:

```
sudo adduser --system --group --no-create-home redis
```

Now, we can create the `/var/lib/redis` directory by typing:

```
sudo mkdir /var/lib/redis
```

We should give the redis user and group ownership over this directory:

```
sudo chown redis:redis /var/lib/redis
```

Adjust the permissions so that regular users cannot access this location:

```
sudo chmod 770 /var/lib/redis
```

###Start the Redis Service

Start up the systemd service by typing:

```
sudo systemctl start redis
```

###Check that the service had no errors by running:

```
sudo systemctl status redis
```

You should see something that looks like this:

```
Output
● redis.service - Redis Server
   Loaded: loaded (/etc/systemd/system/redis.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2016-05-11 14:38:08 EDT; 1min 43s ago
  Process: 3115 ExecStop=/usr/local/bin/redis-cli shutdown (code=exited, status=0/SUCCESS)
 Main PID: 3124 (redis-server)
    Tasks: 3 (limit: 512)
   Memory: 864.0K
      CPU: 179ms
   CGroup: /system.slice/redis.service
           └─3124 /usr/local/bin/redis-server 127.0.0.1:6379       
```

###Enable Redis to Start at Boot

If all of your tests worked, and you would like to start Redis automatically when your server boots, you can enable the systemd service. To do so, type:

```
sudo systemctl enable redis
```
###Install `wkhtmltopdf`

```
sudo apt-get -y install wkhtmltopdf
```

(Note: this should install the latest version with QT)

***Step 5:*** Follow manual installation as `non-root` user
(Note: This will install ERPNext which can be checked by http://ipadress:8000

```
git clone https://github.com/frappe/bench bench-repo
sudo pip install -e bench-repo
```
--------
For .config/git/attributes': Permission denied

cd ~/
ls -al
<Noticed .config was owned by root, unlike everything else in $HOME>
sudo chown -R $(whoami) .config
-------

For Python3

git clone https://github.com/frappe/bench
pip3 install -e ./bench
bench init --frappe-branch master frappe-bench --python python3



***Note:*** In case of `unsupported locale` setting error, run `export LC_ALL=C` from another terminal tab

```
bench init frappe-bench && cd frappe-bench
bench new-site erp.domain.com
```

site1.local sometime doesn't work because of mariadb issue. check guide below

```
sudo vi /etc/mysql/my.cnf
```

###More workaround for an error

```
[mysqld]
innodb-file-format=barracuda
innodb-file-per-table=1
innodb-large-prefix=1
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

***And restart mysql***

```
sudo /etc/init.d/mysql start
```

```
bench get-app --branch master erpnext

bench --site erp.domain.com install-app erpnext
Step 6: Setup production
sudo apt-get -y install supervisor
sudo apt-get -y install nginx
```

###For production setup

```
sudo bench setup production {{ username_goes_here }}
```

Creation of your site - site1.local failed because MariaDB is not properly
configured to use the Barracuda storage engine.
Please add the settings below to MariaDB's my.cnf, restart MariaDB
sudo systemctl restart mariadb.service
then run `bench new-site site1.local` again.

```
[mysqld]
innodb-file-format=barracuda
innodb-file-per-table=1
innodb-large-prefix=1
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

###Multi site setup based on dns

```
bench config dns_multitenant on
bench new-site site2.local
bench setup nginx
sudo service nginx reload
sudo supervisorctl reload
```

Remove mariadb databae

mysqldump -u {user} -p {database} > /home/$USER/Documents/backup.sql
To remove any trace of mariadb installed through apt-get:

sudo service mysql stop
sudo apt-get --purge remove "mysql*"
sudo rm -rf /etc/mysql/

and it is all gone. Including databases and any configuration file.

To check if anything named mysql is gone do a

sudo updatedb
and a

locate mysql
---------

HTTPS Installation

$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository universe
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install certbot python-certbot-nginx

sudo certbot --nginx certonly
sudo certbot renew --dry-run
