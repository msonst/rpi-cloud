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
