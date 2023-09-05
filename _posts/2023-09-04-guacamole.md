---
title: Apache Guacamole for a remote lab
image: guacamole.png
img_path: /images/
date: 2023-09-04
categories: [homelabbing]
tags: [guacamole,remote,ssh,rdp]
pin: false
comments: true
---

Apache Guacamole is a browser based experience for remote SSH and RDP access. In a nut shell you run a self-hosted server which you connect to via a web GUI. From within this site you can add connections to your various networked devices - and they work right there in your browser. Guacamole supports many connection types and encryption protocols so you're sure to find what you need. 

## An Introduction

Let's jump right in and take a look at some of the interface of Guacamole.

![guacamole gui](guacamole-gui.png){: w="840" h="400" }{: .shadow }

Guacamole has a clean interface for quickly getting at your saved connections. It even features a preview mode in each box so you can have an idea of what you're connecting to. Guacamole supports the following protocols:

- Kubernetes
- RDP
- SSH
- Telnet
- VNC

For SSH Guacamole supports username and password based authentication or SSH keys. If you plan on using SSH though, see the note below.

> For SSH key algorithms Guacamole is very picky. You're required to use PEM format. To generate a key compatible with Guacamole try "ssh-keygen -t rsa -b 4096 -m PEM"
{: .prompt-info }

Guacamole allows for User management and has some minimal settings. No distractions here from adding connections and getting going.

![guacamole settings](guacamole-connections.png){: w="840" h="400" }{: .shadow }

When adding an SSH connection Guacamole wants you to specify your key in the OpenSSH format as shown. Guac allows for some terminal customization if you prefer a certain color when you hack away. I think the green on black looks the smoothest as seen below.

![guacamole ssh](guacamole-ssh.png){: w="840" h="400" }{: .shadow }
![guacamole settings](guacamole-terminal.png){: w="840" h="400" }{: .shadow }

Now that we've taken a look at the straight forward settings of Guacamole. Let's go through setting up a Guacamole server.

## Setup

This installation is based off a fresh **Ubuntu 22.04** server.

### Installing Guacd

Guacamole has a lot of dependencies based on what connections you intend to run. Let's install the usual suspects.

```bash
sudo apt install build-essential libcairo2-dev libjpeg-turbo8-dev libpng-dev libtool-bin uuid-dev libavcodec-dev \
libavformat-dev libavutil-dev libswscale-dev freerdp2-dev libpango1.0-dev \
libssh2-1-dev libtelnet-dev libvncserver-dev libwebsockets-dev \
libpulse-dev libssl-dev libvorbis-dev libwebp-dev
```

Next let's download Guacamole to our server.

```bash
wget https://downloads.apache.org/guacamole/1.5.2/source/guacamole-server-1.5.2.tar.gz
```

Extract the file and navigate to its directory.

```bash
tar -xvf guacamole-server-1.5.2.tar.gz
cd guacamole-server-1.5.2
```

Build the installation based off the source files.

```bash
sudo ./configure --with-init-dir=/etc/init.d --enable-allow-freerdp-snapshots
sudo make
sudo make install
```
Update installed library cache and reload systemd.

```bash
sudo ldconfig
sudo systemctl daemon-reload
```
Start Guacd and enable it to start at boot.

```bash
sudo systemctl start guacd
sudo systemctl enable guacd
```
Create a directory to store Guacamole configuration files and extensions. These directories are used in later steps.
```bash
sudo mkdir -p /etc/guacamole/{extensions,lib}
```
### Installing Tomcat

Install Apache Tomcat and modules.

```bash
sudo apt install tomcat9 tomcat9-admin tomcat9-common tomcat9-user
```

Download the Guacamole client.

```bash
wget https://downloads.apache.org/guacamole/1.5.2/binary/guacamole-1.5.2.war
```

Move the client to the Tomcat web directory.

```bash
sudo mv guacamole-1.5.2.war /var/lib/tomcat9/webapps/guacamole.war
```

Restart both Apache Tomcat and Guacd.

```bash
sudo systemctl restart tomcat9 guacd
```

### Setting up a Database

While Apache Guacamole does support basic user authentication via a user-mapping.xml file, it should only be used for testing. For this guide, we will use production-ready database authentication through MySQL/MariaDB.

Install either MySQL or MariaDB on your system. (This guide follows MySQL)

```bash
sudo apt install mysql-server
```
Run the following commands to perform the initial security configuration:
```bash
sudo mysql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'SetRootPasswordHere';
exit
sudo mysql_secure_installation
```
Before populating the database, install the MySQL Connector/J library and Guacamole JDBC authenticator plugin.
Download the [MySQL Connector/J (Java Connector)](https://dev.mysql.com/downloads/connector/j/). For this guide, download the platform independent archived file.
```bash
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.26.tar.gz
```
Extract the tar file and copy it to /etc/guacamole/lib/.

```bash
tar -xf mysql-connector-java-8.0.26.tar.gz
sudo cp mysql-connector-java-8.0.26/mysql-connector-java-8.0.26.jar /etc/guacamole/lib/
```
Download the JDBC auth plugin for Apache Guacamole. This file can be found on [http://guacamole.apache.org/releases/](http://guacamole.apache.org/releases/) by selecting the release version and then locate the “jdbc” file.

```bash
wget https://downloads.apache.org/guacamole/1.5.2/binary/guacamole-auth-jdbc-1.5.2.tar.gz
```
Extract the tar file and copy it to /etc/guacamole/extensions/.

```bash
tar -xf guacamole-auth-jdbc-1.5.2.tar.gz
sudo mv guacamole-auth-jdbc-1.5.2/mysql/guacamole-auth-jdbc-mysql-1.5.2.jar /etc/guacamole/extensions/
```

Log in to mysql as the root user.

```bash
mysql -u root -p
```
The prompt should change again to mysql>.

While in the mysql prompt, change the root password, create a database, and create a new user for that database. When running the below commands, replace any instance of password with a secure password string for the mysql root user and the new user for your database, respectively.

```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE guacamole_db;
CREATE USER 'guacamole_user'@'localhost' IDENTIFIED BY 'password';
GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO 'guacamole_user'@'localhost';
FLUSH PRIVILEGES;
```
Exit the MySQL prompt by typing quit.

Locate the scheme files in the extracted directory for the JDBC plugin.

```bash
cd guacamole-auth-jdbc-1.5.2/mysql/schema
```
Import those sql schema files into the MySQL database.


```bash
cat *.sql | mysql -u root -p guacamole_db
```
Create the properties file for Guacamole.

```bash
sudo nano /etc/guacamole/guacamole.properties
```
Paste in the following configuration settings, replacing [password] with the password of the new guacamole_user that you created for the database.


```text
# MySQL properties
mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacamole_user
mysql-password: [password]
```
Restart all related services.

```bash
sudo systemctl restart tomcat9 guacd mysql
```

### All done
Guacamole should now be accessible at:
```text
[ip]:8080/guacamole
```

![guacamole login](guacamole-login.png){: w="840" h="400" }{: .shadow }

## Connection tips

I've discovered a couple quirks when it comes to setting up Guacamole connections. Here are some tips.

- For Windows RDP connections set the security mode to NLA Authentication
- For both linux and windows connections make sure to check the box to ignore certificate warnings
- For SSH the entry only requires hostname, port 22, your username, and the SSH key in the format seen below.
- I've found Guacamole doesn't seem to do well with DNS so I use IP addresses. This could be my own issues.

![guacamole connections](guacamole-connections.png){: w="840" h="400" }{: .shadow }

