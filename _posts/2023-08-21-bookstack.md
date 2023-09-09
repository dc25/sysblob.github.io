---
title: Everything Bookstack
image: bookstack.png
img_path: /images/
date: 2023-08-21
categories: [homelabbing]
tags: [bookstack,documentation,self-hosted]
pin: false
comments: true
---

Bookstack is a self-hosted wiki that makes editing and storing your documentation in an organized and secure fashion fast, efficient, and easy.

Link: [https://www.bookstackapp.com/](https://www.bookstackapp.com/)

Refer to the Table of Contents if you are looking for something specific about Bookstack.

## What is Bookstack?

Bookstack is first and foremost a self-hosted wiki. A place to store all your knowledge and documentation so you don't lose it or forget it. Unlike static website generators, however, Bookstack is a fully fledged app. You create, edit, publish, and admin over, all your content directly from Bookstack's Web GUI. This makes it more like a Word Press designed for documentation.

Bookstack can be setup as a stand alone apache web server, or a docker container. Luckily, Bookstack is an easy installation -- as the creator has made many scripts available to the community. I personally run both of my Bookstack instances (one local, one remote) as stand alones as I gave them their own virtual machines. 

Let's take a look at the interface of Bookstack and see what makes it unique.

![bookstack gui](bookstack-gui.png){: w="840" h="400" }

You'll notice right away the unique way Bookstack displays information. Your documentation is laid out in a hierarchy of Shelfs > Books > Chapters > Pages. You can use as much or as little of this hierarchy as you want to. To give you an idea of my layout I have two shelves, "Applications", and "Commands and Processes". Under My applications shelf for example I have a Book Docker. I do not use Chapters, but I then have several pages on Docker, i.e. "Docker setup", and "Docker logging".

You'll also notice the customization. Bookstack has a settings area which allows for colors and themes. You can upload logos, and if you're really ambitious, you can add CSS overrides or edit the code as you wish. 

![bookstack settings](bookstack-settings.png){: w="840" h="400" }

Bookstack also has built in user management which is nice. You can add users, authenticate them via email, delegate what rights they have and over what pages, and much more. Other features Bookstack supports are Webhooks to send out data on post actions, and a built in audit log if you want to have a bit of user accountability.

![bookstack users](bookstack-users.png){: w="840" h="400" }

Where Bookstack really shines though, is the ease at which you can create content. Just go where you want, click new page, and everything you need to create and publish a new post is right there. From font customization and coloring, code blocks with syntax highlighting, tables, graphs, images, videos, the whole tool kit is there. 

![bookstack content](bookstack-content.png){: w="840" h="400" }

The best part is Bookstack uses a database and a few files which can easily be backed up so you can take your content on the go pretty easily. The developer even included a script to backup and restore which I'll cover.

I highly recommend Bookstack as an open source free project to store your documents or blog.

## Setting up Bookstack

You can install Bookstack via an automated script which makes it easy, or you can use a docker stack. Let's see the script method first.

### Automatic install (Requires fresh Ubuntu 22.04)

To utilize the script we download it, make sure it has the right permissions, edit any lines we need, and run it. Series of commands:

```bash
wget https://raw.githubusercontent.com/BookStackApp/devops/main/scripts/installation-ubuntu-22.04.sh
chmod a+x installation-ubuntu-22.04.sh
nano installation-ubuntu-22.04.sh # edit the script if you're importing an old bookstack
sudo ./bookstack-install.sh
```

> If your plan is to import your old bookstack look to comment out the line which says "php artisan migrate --no-interaction --force"
{: .prompt-info }

### Docker Installation

If you need help setting up Docker see my guide here: [A deep dive on Docker]({% post_url 2023-09-02-docker %})

Make a docker-compose.yml file and paste in this content:

```yaml
---
version: "2"
services:
  bookstack:
    image: lscr.io/linuxserver/bookstack
    container_name: bookstack
    environment:
      - PUID=1000
      - PGID=1000
      - APP_URL=https://bookstack.example.com
      - DB_HOST=bookstack_db
      - DB_PORT=3306
      - DB_USER=bookstack
      - DB_PASS=<yourdbpass>
      - DB_DATABASE=bookstackapp
    volumes:
      - ./bookstack_app_data:/config
    ports:
      - 6875:80
    restart: unless-stopped
    depends_on:
      - bookstack_db
  bookstack_db:
    image: lscr.io/linuxserver/mariadb
    container_name: bookstack_db
    environment:
      - PUID=1000
      - PGID=1000
      - MYSQL_ROOT_PASSWORD=<yourdbpass>
      - TZ=Europe/London
      - MYSQL_DATABASE=bookstackapp
      - MYSQL_USER=bookstack
      - MYSQL_PASSWORD=<yourdbpass>
    volumes:
      - ./bookstack_db_data:/config
    restart: unless-stopped
```
Container images are configured using parameters passed at runtime (such as those above). Binding is when we tell docker we want ports or directories to be represented inside the container directly. For example, -p 8080:80 would expose port 80 from inside the container to be accessible from the host's IP on port 8080 outside the container. This container uses 6875 externally and can be reached at -- http://hostname:6875

PUID and GUID should be set to your user that will be owning the volume directories for Docker. You can check your ids in linux by running command "id username".

If you need help specifying timezone see this link: [Timezone List](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List)

You can then run the command:

```bash
docker compose up -d
```

## Usage and Tips

### Backing up Bookstack

In simplified terms you really only need to backup a couple files as well as export the MySQL database.

```bash
cd /var/www/bookstack
```
```bash
sudo mysqldump -u bookstack bookstack > bookstack.backup.sql # -u bookstack is the name of your database owner found in /var/www/bookstack/.env
```
```bash
tar -czvf bookstack-files-backup.tar.gz .env public/uploads storage/uploads bookstack.backup.sql
```
Now move this file where ever you need.

Bookstack also comes with a great script for backing up your stack. 

Usage

1. Copy the script down to a file (bookstack-backup.sh).
2. Tweak the configuration variables at the top of the script.
3. Make the script executable (chmod +x bookstack-backup.sh).
4. Run the script (./bookstack-backup.sh).

Here is the script:

```bash
#!/bin/bash

# Directory to store backups within
# Should not end with a slash and not be stored within 
# the BookStack directory
BACKUP_ROOT_DIR="$HOME"

# Directory of the BookStack install
# Should not end with a slash.
BOOKSTACK_DIR="/var/www/bookstack"

# Get database options from BookStack .env file
export $(cat "$BOOKSTACK_DIR/.env" | grep ^DB_ | xargs)

# Create an export name and location
DATE=$(date "+%Y-%m-%d_%H-%M-%S")
BACKUP_NAME="bookstack_backup_$DATE"
BACKUP_DIR="$BACKUP_ROOT_DIR/$BACKUP_NAME"
mkdir -p "$BACKUP_DIR"

# Dump database to backup dir using the values
# we got from the BookStack .env file.
mysqldump --single-transaction \
 --no-tablespaces \
 -u "$DB_USERNAME" \
 -p"$DB_PASSWORD" \
 "$DB_DATABASE" > "$BACKUP_DIR/database.sql"

# Copy BookStack files into backup dir
cp "$BOOKSTACK_DIR/.env" "$BACKUP_DIR/.env"
cp -a "$BOOKSTACK_DIR/storage/uploads" "$BACKUP_DIR/storage-uploads"
cp -a "$BOOKSTACK_DIR/public/uploads" "$BACKUP_DIR/public-uploads"

# Create backup archive
tar -zcf "$BACKUP_DIR.tar.gz" \
 -C "$BACKUP_ROOT_DIR" \
 "$BACKUP_NAME"

# Cleanup non-archive directory
rm -rf "$BACKUP_DIR"

echo "Backup complete, archive stored at:"
echo "$BACKUP_DIR.tar.gz"
```

### Restoring Bookstack

This restore process works best on a fresh Bookstack installation and refers to the directories of the standalone setup. 

After moving your tar file over from the backup process unzip it.
```bash
tar -xvzf bookstack-files-backup.tar.gz
```
Move the files into appropriate directories.
```bash
cp -r public/uploads/* /var/www/bookstack/public/uploads
```
```bash
cp -r storage/uploads/* /var/www/bookstack/storage/uploads
```
> It's important the APP_Key value remains the same between old and new bookstack or it can break things.
{: .prompt-info }

Open the old .env file and copy the APP_KEY value and paste it into new Bookstacks .env
```bash
nano .env # and copy key
nano /var/www/bookstack/.env # paste key
```
Finally while you're in the new Bookstack's .env file make sure to specify the new app URL in APP_URL.
```bash
APP_URL=https://sysblob.com
```
Import the old database into your new running Bookstack.
```bash
mysql -u bookstack -p bookstack < bookstack.backup.sql
```
Run the first time database setup
```bash
cd /var/www/bookstack; php artisan migrate --no-interaction --force
```
Finally it's important if you changed the APP_URL field to update the database so it knows.
```bash
php artisan bookstack:update-url http://old.address.com https://new.address.com
```

### Locating your .env file

Your .env file handles a lot of configuration for Bookstack. This is located at /var/www/bookstack/.env

### Bookstack inside an iFrame

By default BookStack will only allow itself to be embedded within iframes on the same domain as you’re hosting on. This is done through a CSP: frame-ancestors header. You can add additional trusted hosts by setting an ALLOWED_IFRAME_HOSTS option in your .env file like the example below:

```yaml
# Adding a single host
ALLOWED_IFRAME_HOSTS="https://example.com"
# Multiple hosts can be separated with a space
ALLOWED_IFRAME_HOSTS="https://a.example.com https://b.example.com"
```

> When this option is used, all cookies will be served with SameSite=None (info) set so that a user session can persist within the iframe.
{: .prompt-info }

### Setting default Dark mode

Open the file at /var/www/bookstack/.env and add in this line:
```yaml
APP_DEFAULT_DARK_MODE=true
```

### Default file permissions
If you're having any issues with file permissions here is the exact quote taken from Bookstack on a typical file setup using the user Barry.

- Set the bookstack folders and files to be owned by the user barry and have the group www-data
```bash
sudo chown -R barry:www-data /var/www/bookstack
```
- Set all bookstack files and folders to be readable, writeable & executable by the user (barry) and readable & executable by the group and everyone else
```bash
sudo chmod -R 755 /var/www/bookstack
```
- For the listed directories, grant the group (www-data) write-access
```bash
sudo chmod -R 775 /var/www/bookstack/storage /var/www/bookstack/bootstrap/cache /var/www/bookstack/public/uploads
```
- Limit the .env file to only be readable by the user and group, and only writable by the user.
```bash
sudo chmod 640 /var/www/bookstack/.env
```