---
title: Backing up and restoring Bookstack
image: bookstack.png
img_path: /images/
date: 2023-09-02
categories: [homelab,how-to]
tags: [bookstack,restore,backup]
pin: false
comments: true
---

## Backing up

cd /var/www/bookstack
sudo mysqldump -u root bookstack > bookstack.backup.sql
tar -czvf bookstack-files-backup.tar.gz .env public/uploads storage/uploads bookstack.backup.sql

Transfer file to new bookstack server

## Restoring

wget https://raw.githubusercontent.com/BookStackApp/devops/main/scripts/installation-ubuntu-22.04.sh
chmod a+x installation-ubuntu-22.04.sh
nano installation-ubuntu-22.04.sh
comment out line php artisan migrate --no-interaction --force
sudo ./installation-ubuntu-22.04.sh
tar -xvzf bookstack-files-backup.tar.gz
cp -r public/uploads/* /var/www/bookstack/public/uploads
cp -r storage/uploads/* /var/www/bookstack/storage/uploads
nano .env
copy the APP_KEY
nano /var/www/bookstack/.env
replace new APP_KEY with old one
optionally add darkmode or allow iframe hosts to .env
APP_DEFAULT_DARK_MODE=true
ALLOWED_IFRAME_HOSTS="https://dashboard.sysblob.com"
update APP_URL field to new address
mysql -u bookstack -p bookstack < bookstack.backup.sql
php artisan migrate

php artisan bookstack:update-url http://old.address.com https://new.address.com

# Set the bookstack folders and files to be owned by the user barry and have the group www-data
sudo chown -R barry:www-data /var/www/bookstack

# Set all bookstack files and folders to be readable, writeable & executable by the user (barry) and
# readable & executable by the group and everyone else
sudo chmod -R 755 /var/www/bookstack

# For the listed directories, grant the group (www-data) write-access
sudo chmod -R 775 /var/www/bookstack/storage /var/www/bookstack/bootstrap/cache /var/www/bookstack/public/uploads

# Limit the .env file to only be readable by the user and group, and only writable by the user.
sudo chmod 640 /var/www/bookstack/.env
