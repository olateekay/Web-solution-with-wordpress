# Web Solution With WordPress

*Step 1 — Prepare a Web Server*

Launch an EC2 instance that will serve as “Web Server”. 

Create 3 volumes in the same AZ as your Web Server EC2, each of 10 GiB.

![alt text](image1.jpg)

![alt text](image2.jpg)

![alt text](image3.jpg)

Open up the Linux terminal to begin configuration
Use lsblk command to inspect what block devices are attached to the server. Notice names of your newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly created block devices there - their names will likely be xvdf, xvdh, xvdg.

![alt text](image4.jpg)

Use `df -h` command to see all mounts and free space on your server

Use `gdisk` utility to create a single partition on each of the 3 disks

![alt text](image5.jpg)

![alt text](image6.jpg)

![alt text](image7.jpg)

Install `lvm2 `package using `sudo yum install lvm2`. Run sudo lvmdiskscan command to check for available partitions.

![alt text](image8.jpg)


Use `pvcreate` utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM.

NB: you can create for the 3 discs in one command;

`sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`

![alt text](image9.jpg)

Running `sudo pvs` shows you the physical volumes created

![alt text](image10.jpg)

Use `vgcreate` utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg

`sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`

![alt text](image11.jpg)

Verify that your VG has been created successfully by running `sudo vgs`

![alt text](image12.jpg)


Use l`vcreate` utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.

`sudo lvcreate -n apps-lv -L 14G webdata-vg`


![alt text](image13.jpg)


`sudo lvcreate -n logs-lv -L 14G webdata-vg`


![alt text](image14.jpg)

Verify that your Logical Volume has been created successfully by running `sudo lvs`

![alt text](image15.jpg)

Verify the entire setup

`sudo vgdisplay -v ` #*view complete setup - VG, PV, and LV*

`sudo lsblk `

![alt text](image16.jpg)

![alt text](image17.jpg)

Use mkfs.ext4 to format the logical volumes with `ext4` filesystem

`sudo mkfs -t ext4 /dev/webdata-vg/apps-lv`

`sudo mkfs -t ext4 /dev/webdata-vg/logs-lv`


![alt text](image18.jpg)


Create `/var/www/html` directory to store website files

`sudo mkdir -p /var/www/html`

![alt text](image19.jpg)

Create `/home/recovery/logs` to store backup of log data


`sudo mkdir -p /home/recovery/logs`



Mount `/var/www/html` on apps-lv logical volume


`sudo mount /dev/webdata-vg/apps-lv /var/www/html/
`

![alt text](image20.jpg)

Use `rsync utility `to backup all the files in the log directory `/var/log` into `/home/recovery/logs` (This is required before mounting the file system)

`sudo rsync -av /var/log/. /home/recovery/logs/`

![alt text](image21.jpg)

Mount `/var/log on logs-lv `logical volume. (Note that all the existing data on /var/log will be deleted. That is why step 15 above is very important)

`sudo mount /dev/webdata-vg/logs-lv /var/log`

![alt text](image22.jpg)

Restore log files back into `/var/log` directory

`sudo rsync -av /home/recovery/logs/log/. /var/log
`

![alt text](image24.jpg)

Update `/etc/fstab `file so that the mount configuration will persist after restart of the server.

The UUID of the device will be used to update the `/etc/fstab` file;

`sudo blkid`

![alt text](image24.jpg)

`sudo vi /etc/fstab`

Update `/etc/fstab` in this format using your own UUID and remember to remove the leading and ending quotes.

![alt text](image26.jpg)

Test the configuration and reload the daemon

`sudo mount -a`

`sudo systemctl daemon-reload`

Verify your setup by running `df -h`, output must look like this:

![alt text](image28.jpg)


# Step 2 — Prepare the Database Server

Launch a second RedHat EC2 instance that will have a role - ‘DB Server’ Repeat the same steps as for the Web Server, but instead of `apps-lv` create `db-lv` and mount it to `/db` directory instead of `/var/www/html/`.


![alt text](image29.jpg)

![alt text](image30.jpg)

![alt text](image31.jpg)

![alt text](image32.jpg)

![alt text](image33.jpg)

![alt text](image34.jpg)

![alt text](image36.jpg)

# Step 3 — Install Wordpress on your Web Server EC2

Update the repository

`sudo yum -y update`

Install wget, Apache and it’s dependencies

`sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`

Start Apache

`sudo systemctl enable httpd`


`sudo systemctl start httpd`


To install PHP and it’s dependencies

```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1

```

![alt text](image37.jpg)

Restart Apache

`sudo systemctl restart httpd`

Download wordpress and copy wordpress to `var/www/html`

```
mkdir wordpress
cd   wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html/
```

![alt text](image38.jpg)

# Configure SELinux Policies

```
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1

```

![alt text](image39.jpg)


# Step 4 — Install MySQL on your DB Server EC2

```
sudo yum update
sudo yum install mysql-server
```

Verify that the service is up and running by using `sudo systemctl status mysqld`, if it is not running, restart the service and enable it so it will be running even after reboot:

```
sudo systemctl restart mysqld
sudo systemctl enable mysqld

```

![alt text](image40.jpg)

# Step 5 — Configure DB to work with WordPress

```
sudo mysql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```



# Step 6 — Configure WordPress to connect to remote database.


Hint: Do not forget to open MySQL port 3306 on DB Server EC2. For extra security, you shall allow access to the DB server ONLY from your Web Server’s IP address, so in the Inbound Rule configuration specify source as /32




Install MySQL client and test that you can connect from your Web Server to your DB server by using `mysql-client`

```
sudo yum install mysql
sudo mysql -u admin -p -h <DB-Server-Private-IP-address>
```

Verify if you can successfully execute S`HOW DATABASES;` command and see a list of existing databases.


![alt text](image42.jpg)

Try to access from your browser the link to your WordPress `http://<Web-Server-Public-IP-Address>`

![alt text](image43.jpg)

![alt text](image44.jpg)