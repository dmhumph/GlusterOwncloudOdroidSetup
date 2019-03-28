# Gluster on Odroid HC2 with Ubuntu for OwnCloud on Odroid C2
How to setup Odroid HC2 Ubuntu and use as Gluster servers.  Then how to configure Odroid C2 Owncloud server as gluster client.

## Hardware needed

2 or more Odroid HC2:
https://ameridroid.com/collections/single-board-computer/products/odroid-hc2

1 Odroid C2:
https://ameridroid.com/collections/single-board-computer/products/odroid-c2

Make sure to get the necessary power supplies and either MicroSD or other storage medium for flashing the OS to boot from on the SBC.

## Install Gluster Servers

Ubuntu download for Odroid HC2:
https://odroid.in/ubuntu_18.04lts/

Flash the above image to flash media.

Insert flash media into flash port and add power to HC2 to start the boot process.
ssh to odroid after boot up.  Might take two tries (most of the time).
ssh root@<host>
password: odroid

Change password and hostname.  Reboot.
```
passwd
hostnamectl set-hostname <hostname>
reboot
```

Update and install necessary packages:
```
apt update
apt upgrade -y
apt-get -y install thin-provisioning-tools
apt-get -y install lvm2
apt-get -y install xfsprogs
reboot
```

Add gluster ppa and install gluster:
```
add-apt-repository ppa:gluster/glusterfs-4.1 && sudo apt update
apt install glusterfs-server -y
```

Check to make sure gluster is now running:
```
systemctl status glusterd
```

Create a Trusted Storage Pool between gluster nodes.  These commands should only be ran from one host and in this case it is ran from gfs01:
```
gluster peer probe gfs02
```
Now check the storage pool status:
```
gluster peer status
gluster pool list
```

Partition the /dev/sda device:
```
fdisk /dev/sda
n
p
1
Hit ENTER
Hit ENTER
w
```

Format the partition:
```
mkfs.ext4 /dev/sda1
```

Make new directory:
```
mkdir -p /glusterfs/distributed
```

Mount volume to folder:
```
mount /dev/sda1 /glusterfs/distributed/
```

Make sure to mount on reboot:
vi /etc/fstab
```
/dev/sda1  /glusterfs/distributed  ext4  defaults  0  0
```

Create distributed glisterfs volume:

```
gluster volume create vol01 replica 2 transport tcp \
> gfs01.humphries.home:/glusterfs/distributed \
> gfs02.humphries.home:/glusterfs/distributed \
> force
```
Start the gluster volume:
```
gluster volume start vol01
```

Check the volume information:
```
gluster volume info vol01
```

## Install Gluster Client Host and OwnCloud

Use the following Ubuntu image of Odroid C2:
https://odroid.in/ubuntu_18.04lts/ubuntu-18.04-3.16-minimal-odroid-c2-20180626.img.xz

Flash the above image to flash media.

Insert flash media into flash port and add power to HC2 to start the boot process.
ssh to odroid after boot up.  Might take two tries (most of the time).
ssh root@<host>
password: odroid

Change password and hostname.  Reboot.
```
passwd
hostnamectl set-hostname <hostname>
reboot
```

Update and install necessary packages:
```
apt update
apt upgrade -y
apt-get -y install thin-provisioning-tools
apt-get -y install lvm2
apt-get -y install xfsprogs
apt-get -y install software-properties-common
reboot
```

Add gluster ppa and install gluster:
```
add-apt-repository ppa:gluster/glusterfs-4.1 && sudo apt update
```

Install Gluster client:
```
apt install glusterfs-client -y
```

Create directory to mount gluster volume to on the client:
```
mkdir -p /mnt/glusterfs
```

Mount the gluster volume to the client directory:
```
mount -t glusterfs gfs01.humphries.home:/vol01 /mnt/glusterfs
```

Make sure to auto mount the gluster volume on restarts:
vi /etc/fstab
```
gfs01.humphries.home:/vol01 /mnt/glusterfs glusterfs defaults,_netdev 0 0
```

Install Apache:
```
apt install apache2 -y
```

Disable directory listing on Apache:
```
a2dismod autoindex
```

Enable Apache modules for Owncloud to work properly:
```
a2enmod rewrite
a2enmod headers
a2enmod env
a2enmod dir
a2enmod mime
```

Restart Apache:
```
systemctl restart apache2
```

Install MariaDB server:
```
apt-get install mariadb-server mariadb-client -y
```

Secure MariaDB:
```
mysql_secure_installation
```

Login to MariaDB server with root and password you just created in securing the installation:
```
mysql -u root -p

Create OwnCloud database, user, and password and replace <password> below:
CREATE DATABASE owncloud;
CREATE USER 'oc_user'@'localhost' IDENTIFIED BY 'PASSWORD';
GRANT ALL ON owncloud.* TO 'oc_user'@'localhost' IDENTIFIED BY 'PASSWORD' WITH GRANT OPTION; 
FLUSH PRIVILEGES;
EXIT;
```

Install PHP:
```
add-apt-repository ppa:ondrej/php
apt update
apt install php7.1 -y
```

Install PHP related modules:
```
apt-get install php7.1-cli php7.1-common php7.1-mbstring php7.1-gd php7.1-intl php7.1-xml php7.1-mysql php7.1-zip php7.1-curl php7.1-xmlrpc -y
```

Adjust PHP default settings for OwnCloud:
```
vi /etc/php/7.1/apache2/php.ini
```

Find the following settings and change their values to the following:
```
memory_limit - 256M
upload_max_filesize = 10G
max_file_uploads = 100
```

Restart Apache:
```
systemctl restart apache2
```
