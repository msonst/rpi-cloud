$Download & write rpi image
# Assuming: 2017-04-10-raspbian-jessie-lite

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
sudo apt-get -y install apache2 php5 php5-gd php-xml-parser php5-intl php5-sqlite php5-mysql smbclient curl libcurl3 php5-curl
sudo apt-get -y install mysql-server mysql-client php-apc
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
## 0) Single drive

# Configure fstab for automounting drives on startup
sudo su -c "echo '/dev/sda1 /data/virtual ext3 defaults 1 2' >> /etc/fstab"
sudo mount -a

######################################
## 1) Joined filesystems via mhddfs
       
## Partition & Format
# Assuming sdb as second drive
sudo fdisk /dev/sdb # n, p, enter, enter, enter, w
sudo mke2fs -j /dev/sdb1

# Create mountpoints
sudo mkdir -p /data/data1
sudo mkdir -p /data/data2

# Install mhddfs and configure fstab for automounting drives on startup
sudo apt-get -y install mhddfs
sudo su -c "echo '/dev/sda1 /data/data1 ext3 defaults 1 2' >> /etc/fstab"
sudo su -c "echo '/dev/sdb1 /data/data2 ext3 defaults 1 2' >> /etc/fstab"
sudo su -c "echo 'mhddfs#/data/data1,/data/data2 /data/virtual fuse defaults,allow_other 0 0' >> /etc/fstab"
sudo mount -a

######################################
## 2) Via mdadm (Untested)
##   (min 3 drives)

## Partition & Format
# Assuming sdb and sdc as drives in addition to sda
sudo fdisk /dev/sdb # n, p, enter, enter, enter, w
sudo fdisk /dev/sdc # n, p, enter, enter, enter, w
sudo mke2fs -j /dev/sdb1
sudo mke2fs -j /dev/sdc1

# Create MD RAID 5 
mdadm --create /data/virtual --level=5 --raid-devices=3 /data/sda1 /data/sdb1 /data/sdc1

# In the event that a drive fails do the following
mdadm /dev/md1 --fail /dev/sda1

# Format the new drive
mdadm --add /dev/md1 /dev/sda1

################################################################################
## Permissions

sudo usermod -aG www-data <USER> 
sudo chown www-data:www-data /data/virtual
sudo chmod 777 /data/virtual

################################################################################
## Hardening

################################################################################
## Router Setup 
# - port forwarding to owncloud pi
# - Setup ddns

################################################################################
## Setup Owncloud
# - Set data dir during setup to /data/virtual
# - Set DB to MySQL

################################################################################
# END
################################################################################

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


