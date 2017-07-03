For an enclusure see [rpi-drive](https://github.com/msonst/rpi-drive)

```
################################################################################
#  Copyright (c) 2017, Michael Sonst, All Rights Reserved.
# 
#  Licensed under the Apache License, Version 2.0 (the "License");
#  you may not use this file except in compliance with the License.
#  You may obtain a copy of the License at
# 
#  http://www.apache.org/licenses/LICENSE-2.0
# 
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
#  limitations under the License.
################################################################################

# Download & write rpi image
# Assuming: 2017-04-10-raspbian-jessie-lite

# After creating the boot-SD card, copy the SSH file into the root directory
# Mount the created drives in a Linux VM and copy interfaces and wpa_supplicant.conf to
# their correct location
# Replace <WIFI-NAME> and <WIFI-KEY> in wpa_supplicant.conf
# Please note that the configuration in wpa_supplicant.conf is specifically for WPA2-PSK [AES] e.g. on a WNDR3400v3

ssh pi@<IP>

# Set new password
passwd
sudo adduser <USER>
sudo usermod -aG sudo <USER> 

sudo raspi-config
# Interfacing Options > Enable SSH
# Hostname > private-cloud
# Advanced > Expand Filesystem
# Localisation Options > Change Wi-fi Country > US
# Localisation Options > Change Timezone > US > Eastern
# reboot
# login with <USER>

sudo timedatectl set-ntp True
sudo apt-get -y update
sudo apt-get -y dist-upgrade
sudo apt-get -y update 
sudo apt-get -y install rsync

# MYSQL https://samhobbs.co.uk/2013/10/install-owncloud-on-your-raspberry-pi
sudo apt-get -y install apache2 php5 php5-gd php-xml-parser php5-intl php5-sqlite php5-mysql smbclient curl libcurl3 php5-curl mysql-server mysql-client php-apc git
sudo service apache2 restart

## Install Owncloud
wget https://download.owncloud.org/community/owncloud-10.0.2.zip
sudo cp owncloud-10.0.2.zip /var/www/html
cd /var/www/html
sudo unzip -q owncloud-10.0.2.zip
sudo rm owncloud-10.0.2.zip

## Partition & Format
# Assuming sda as drive
sudo fdisk /dev/sda # n, p, enter, enter, enter, w
sudo mke2fs -j /dev/sda1

# Create mountpoints
sudo mkdir -p /data/virtual

################################################################################
## 0) Single drive (for alternative see multidrive.md)

# Configure fstab for automounting drives on startup
sudo su -c "echo '/dev/sda1 /data/virtual ext3 defaults 1 2' >> /etc/fstab"
sudo mount -a

################################################################################
## Permissions

sudo chmod 777 /data/virtual

sudo usermod -aG www-data <USER> 
sudo chown -R www-data:www-data /data/virtual
sudo find /data/virtual -type f -exec chmod 664 {} + -o -type d -exec chmod 775 {} +
sudo find /data/virtual -type d -exec chmod g+rx {} +
sudo find /data/virtual -type f -exec chmod g+r {} +
sudo find /data/virtual -type d -exec chmod g+s {} +

sudo chown -R www-data:www-data /var/www/html
sudo find /var/www/html -type f -exec chmod 664 {} + -o -type d -exec chmod 775 {} +
sudo find /var/www/html -type d -exec chmod g+rx {} +
sudo find /var/www/html -type f -exec chmod g+r {} +
sudo find /var/www/html -type d -exec chmod g+s {} +

################################################################################
## Hardening

sudo rm /var/www/html/index.html

sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod ssl
sudo a2enmod dav_fs
sudo service apache2 restart

sudo mkdir /etc/apache2/ssl
sudo openssl req -newkey rsa:4096 -sha512 -x509 -days 365 -nodes -keyout /etc/apache2/ssl/server.key -out /etc/apache2/ssl/server.crt
sudo ln -s /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-enabled/000-default-ssl.conf

# Alternative (Untested)
# cd ~
# wget https://dl.eff.org/certbot-auto
# chmod a+x ./certbot-auto
# ./certbot-auto

nano /etc/apache2/sites-enabled/000-default-ssl.conf
> SSLCertificateFile      /etc/apache2/ssl/server.crt
> SSLCertificateKeyFile   /etc/apache2/ssl/server.key

nano /etc/apache2/conf-available/security.conf
# Uncomment:
> <Directory />
>    AllowOverride None
>    Order Deny,Allow
>    Deny from all
> </Directory>
#
# Add:
> <Directory /var/www/html/owncloud>
>   Options Indexes FollowSymLinks MultiViews
>   AllowOverride All
>   Order allow,deny
>   Allow from all
>   Satisfy Any
> </Directory>
#
# Set 
>  ServerTokens Prod
>  ServerSignature Off
>  Header edit Set-Cookie ^(.*)$ $1;HttpOnly;Secure
>  Header always append X-Frame-Options SAMEORIGIN
>  Header set X-XSS-Protection "1; mode=block"
>  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
>  Header set X-Content-Type-Options: "nosniff"
# # ISSUE: OC can't find files anymore #>  Header set Content-Security-Policy "default-src 'self';"
>  RewriteEngine On
>  RewriteCond %{THE_REQUEST} !HTTP/1\.1$
>  RewriteRule .* - [F]

nano/etc/apache2/mods-available/ssl.conf
> SSLProtocol -all +TLSv1
> SSLCipherSuite HIGH:!MEDIUM:!aNULL:!MD5:!RC4

nano/etc/apache2/mods-available/userdir.conf
> <LimitExcept GET POST> # OPTIONS>

nano /etc/apache2/apache2.conf
> Timeout 60

sudo service apache2 restart


# On backup node
ssh-copy-id -i ~/.ssh/id_rsa.pub <USER>@<OWNCLOUD_INTERNAL_IP>

nano /boot/config.txt
> dtoverlay=pi3-disable-wifi
> dtoverlay=pi3-disable-bt

# On owncloud node
sudo ifdown wlan0
sudo apt-get install -y wipe
sudo wipe /etc/wpa_supplicant/wpa_supplicant.conf

sudo nano /etc/ssh/sshd_config
> ChallengeResponseAuthentication no
> PasswordAuthentication no
> UsePAM no
> PermitRootLogin no
sudo /etc/init.d/ssh reload

################################################################################
## Router Setup 
# - port forwarding to owncloud pi
# - Setup ddns ==> <DDNS>
# Inbound https
# Outbound https, http

nano /var/www/html/owncloud/config/config.php
> 'trusted_domains' =>
>  array (
>    0 => '<STATICIP>',
>    1 => '<DDNS>',
>  ),

################################################################################
## Setup Owncloud
# - Set data dir during setup to /data/virtual
# - Set DB to MySQL

################################################################################
## Troubleshooting
# WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! 
# > ssh-keygen -R <IP>

#######################
# If you get an error about database drivers, then try installing php5-mysql 
# again – for some reason, it didn’t install properly first time for me, 
# although no errors were shown:
sudo apt-get install php5-mysql
sudo reboot

#######################
# Error: MySQL/MariaDB user 'oc_admin'@'localhost' exists already. Drop this user from MySQL/MariaDB
mysql -u root -p
mysql> DROP USER 'oc_admin'@'localhost';

#######################
# Error: MySQL/MariaDB user 'oc_admin'@'%' already exists	Drop this user from MySQL/MariaDB.
mysql -u root -p
mysql> DROP USER 'oc_admin'@'%';

#######################
## Reset MySQL
mysql -u root -p
mysql> SHOW DATABASES;
mysql> DROP DATABASE IF EXISTS owncloud;
mysql> quit

#######################
# Accidently installed a version 7.x.x e.g. via apt causes the following issues after
# reinstallation version 10:
# The requested URL /owncloud/index.php was not found on this server.
sudo rm /etc/apache2/conf-enabled/owncloud.conf
sudo rm /etc/apache2/conf-available/owncloud.conf
sudo systemctl restart apache2

#######################
## Benchmark system
# IO, Mysql, etc.
#https://www.howtoforge.com/how-to-benchmark-your-system-cpu-file-io-mysql-with-sysbench

#######################
## Benchmark Network
sudo apt-get -y install iperf iptraf
sudo iperf -s
sudo iperf -c <SERVER_IP> -d
sudo iptraf

#######################
## Benchmark HDD Troughput
sudo apt-get install

#######################
## Interesting things to measure
sudo vcgencmd measure_temp  # > temp=52.6'C
sudo vcgencmd get_throttled # > throttled=0x0
```
