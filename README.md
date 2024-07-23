# multitier-webapp-manual-setup
Deploying a Multi-tier Java Web Application  for manual provisioning
### VM SETUP
1. Clone source code.
2. Cd into the repository.
3. Switch to the main branch.
4. cd into multitier-webapp-manual-setup

- Bring up vm’s

```
vagrant up
```
### Vagrant Status

Output
```
Aungs-MacBook-Pro:multitier-webapp-manual-setup aungkohtet$ vagrant status
Note: For hostname management, consider installing the vagrant-hostmanager plugin
You can install it with: vagrant plugin install vagrant-hostmanager
Current machine states:

db01                      running (virtualbox)
mc01                      running (virtualbox)
rmq01                     running (virtualbox)
app01                     running (virtualbox)
web01                     running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

### PROVISIONING

Services
1. Nginx            => Webservice
2. Tomcat           => Application Service
3. RabbitMQ         => Broker /Queuing Agent
4. Memcache         => DB Caching
5. ElasticSearch    => Indexing/Search service
6. MySQL            => SQL Database


### Setup should be done in below mentioned order

MySQL       (Database SVC) 
Memcache    (DB Caching SVC) 
RabbitMQ    (Broker/Queue SVC) 
Tomcat      (Application SVC) 
Nginx       (Web SVC)

## 1. MYSQL Setup

Login to the db vm
```bash
$ vagrant ssh db01
```

Verify Hosts entry, if entries missing update the it with IP and hostnames
```
 cat /etc/hosts

 ```

Update OS with latest patches

```
 yum update -y 
 ```

 Set Repository
 ```
 yum install epel-release -y 
 ```

 Install Maria DB Package
 ```
 yum install git mariadb-server -y 
 ```
 Starting & enabling mariadb-server
 ```
 systemctl start mariadb
 systemctl enable mariadb

 ```

### RUN mysql secure installation script.
NOTE: Set db root password, I will be using admin123 as password

Set DB name and users.
```
mysql -u root -padmin123
```

```
MariaDB [(none)]> create database accounts;
Query OK, 1 row affected (0.001 sec)

MariaDB [(none)]> grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123';
Query OK, 0 rows affected (0.002 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> exit;
Bye
[root@db01 ~]# 
```

### Adding Source code & Initialize Database

```
[root@db01 ~]# git clone -b main https://github.com/hkhcoder/vprofile-project.git
Cloning into 'vprofile-project'...
remote: Enumerating objects: 516, done.
remote: Total 516 (delta 0), reused 0 (delta 0), pack-reused 516
Receiving objects: 100% (516/516), 7.69 MiB | 13.87 MiB/s, done.
Resolving deltas: 100% (201/201), done.
[root@db01 ~]# cd vprofile-project/
[root@db01 vprofile-project]# mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
[root@db01 vprofile-project]# mysql -u root -padmin123 accounts
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 5
Server version: 10.5.22-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [accounts]> show tables;
+--------------------+
| Tables_in_accounts |
+--------------------+
| role               |
| user               |
| user_role          |
+--------------------+
3 rows in set (0.001 sec)

MariaDB [accounts]> exit;
Bye
[root@db01 vprofile-project]# 
```


### Restart mariadb-server
```
systemctl restart mariadb
```
### Starting the firewall and allowing the mariadb to access from port no. 3306

```
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --get-active-zones
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
systemctl restart mariadb
```

- Output

```
[root@db01 vprofile-project]# systemctl restart mariadb
[root@db01 vprofile-project]# systemctl start firewalld
[root@db01 vprofile-project]# systemctl enable firewalld
Created symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service → /usr/lib/systemd/system/firewalld.service.
Created symlink /etc/systemd/system/multi-user.target.wants/firewalld.service → /usr/lib/systemd/system/firewalld.service.
[root@db01 vprofile-project]# firewall-cmd --get-active-zones
public
  interfaces: enp0s3 enp0s8
[root@db01 vprofile-project]# firewall-cmd --zone=public --add-port=3306/tcp --permanent
success
[root@db01 vprofile-project]# firewall-cmd --reload
success
[root@db01 vprofile-project]# systemctl restart mariadb
[root@db01 vprofile-project]# 
```

## 2. MEMCACHE SETUP

### Login to the Memcache vm
```
vagrant ssh mc01
```
### Verify Hosts entry, if entries missing update the it with IP and hostnames

```
cat /etc/hosts
```

### Update OS with latest patches

```
yum update -y
```

### Install, start & enable memcache on port 11211

```
sudo dnf install epel-release -y
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached
```
### Starting the firewall and allowing the port 11211 to access memcache

```
firewall-cmd --add-port=11211/tcp
firewall-cmd --runtime-to-permanent
firewall-cmd --add-port=11111/udp
firewall-cmd --runtime-to-permanent
sudo memcached -p 11211 -U 11111 -u memcached -d
```
## 3. RABBITMQ SETUP

### Login to the RabbitMQ vm
```
vagrant ssh rmq01
```
### Verify Hosts entry, if entries missing update the it with IP and hostnames
```
cat /etc/hosts
```
### Update OS with latest patches
```
yum update -y
```
### Set EPEL Repository
```
yum install epel-release -y
```
### Install Dependencies

```
sudo yum install wget -y
cd /tmp/
dnf -y install centos-release-rabbitmq-38
dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
systemctl enable --now rabbitmq-server
```

### Setup access to user test and make it admin

```
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo systemctl restart rabbitmq-server
```
### Starting the firewall and allowing the port 5672 to access rabbitmq

```
firewall-cmd --add-port=5672/tcp
firewall-cmd --runtime-to-permanent
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo systemctl status rabbitmq-server
```


## 4. TOMCAT SETUP

### Login to the tomcat vm
```
vagrant ssh app01
```

### Verify Hosts entry, if entries missing update the it with IP and hostnames
```
cat /etc/hosts
```
### Update OS with latest patches

```
yum update -y
```

### Set Repository
```
yum install epel-release -y
```
### Install Dependencies
```
dnf -y install java-11-openjdk java-11-openjdk-devel
dnf install git maven wget -y
```
### Change dir to /tmp
```
cd /tmp/
```
### Download & Tomcat Package
```
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.75/bin/apache-tomcat-9.0.75.tar.gz
```
```
tar xzvf apache-tomcat-9.0.75.tar.gz
```
### Add tomcat user
```
useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat

```
### Copy data to tomcat home dir
```
cp -r /tmp/apache-tomcat-9.0.75/* /usr/local/tomcat/
```
### Make tomcat user owner of tomcat home dir
```
chown -R tomcat.tomcat /usr/local/tomcat
```
### Setup systemctl command for tomcat
- Create tomcat service file
```
vi /etc/systemd/system/tomcat.service
```
### Update the file with below content
```
[Unit]
Description=Tomcat
After=network.target
[Service]
User=tomcat
WorkingDirectory=/usr/local/tomcat
Environment=JRE_HOME=/usr/lib/jvm/jre
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINE_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/catalina.sh run
ExecStop=/usr/local/tomcat/bin/shutdown.sh
SyslogIdentifier=tomcat-%i
[Install]
WantedBy=multi-user.target
```

### Reload systemd files
```
systemctl daemon-reload
```
### Start & Enable service
```
systemctl start tomcat
systemctl enable tomcat
```
### Enabling the firewall and allowing port 8080 to access the tomcat

```
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --get-active-zones
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --reload
```

## CODE BUILD & DEPLOY (app01)

### Download Source code
```
git clone -b main https://github.com/hkhcoder/vprofile-project.git
```

### Update configuration
```
cd vprofile-project
vim src/main/resources/application.properties
Update file with backend server details
```
### Build code
### Run below command inside the repository (vprofile-project)
```
mvn install
```
### Deploy artifact
```
systemctl stop tomcat
```

```
rm -rf /usr/local/tomcat/webapps/ROOT*
cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
systemctl start tomcat
chown tomcat.tomcat /usr/local/tomcat/webapps -R
systemctl restart tomcat
```


## 5. NGINX SETUP

### Login to the Nginx vm
```
vagrant ssh web01
sudo -i
```
### Verify Hosts entry, if entries missing update the it with IP and hostnames
```
cat /etc/hosts
```
### Update OS with latest patches
```
apt update
apt upgrade
```
### Install nginx
```
apt install nginx -y
```

### Create Nginx conf file
```
vim /etc/nginx/sites-available/vproapp
```

### Update with below content
```
upstream vproapp {
    server app01:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://vproapp;
    }
}
```
### Remove default nginx conf
```
rm -rf /etc/nginx/sites-enabled/default
```
### Create link to activate website
```
 ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
 ```


### Restart Nginx
```
systemctl restart nginx
```
