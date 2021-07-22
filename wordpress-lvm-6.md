# **Deploy a full scale web solution using wordpress and configure the linux storage subsystem using LVM (Logical Volume Manager)**

This projects entails deploying a wordpress CMS to the cloud using AWS EC2 instances, which will be partitioned, configured into logical volumes using the logical volume manager, and mounted on the Linux servers.

## **Launch EC2 instances (Partitioning, Configuring with LVM & Mounting)**

- Create two linux virtual servers and configure one to act as a `web-server` and the other as a `database-server`

- Create Volumes for the `web-server` by going to the volumes section in your AWS console and creating three volumes of size 10GB each. The caveat is that they must be set to the same AVZ(Availabilty Zone), this is very important.

- After creation, attach those volumes to your `web-server` instance, then we can begin configuring.

- You can check the newly created volumes on the terminal using `lsblk`. We have to partition our newly created volumes `xvdf`, `xvdg`, `xvdh`, this will be made possible by a utility called `gdisk`. GDISK is used in partitioning GPT(GUID Partition Table) type disks and that is the type of disk our ec2 instance has, in the case where it is an MBR(Master Boot Record) disk, it will be configured with another utility known as `fdisk`

![11](https://user-images.githubusercontent.com/47898882/126381242-8bd82a81-acb3-4f0c-82b3-7f50188d4370.JPG)

- To partition these volumes, use the command below

```
$ sudo gdisk /dev/xvdf
$ sudo gdisk /dev/xvdg
$ sudo gdisk /dev/xvdh
```
- When the prompt is loaded and it asks for a command, type `n` to create the partition. The most important option we have to update is what type of partition we want this disk to be. By default it is configured to be a Linux filesystem disk, but we will have to configure it to a Linux LVM disk by using hex code `8E00`, this is gdisks' hex code for Linux LVM. To save your configuration type `w` and this will write your configuration onto those volumes.

- To make the conversion of our partitions into logical volumes, the logical volume manager package has to be installed.

```
$ sudo yum install lvm2
```

- After all 3 volumes have been partitioned successfully, mark them as physical volumes using the `pvcreate` utility.

```
$ sudo pvcreate /dev/xvdf1
$ sudo pvcreate /dev/xvdg1
$ sudo pvcreate /dev/xvdh1
```
- Confirm they have been marked as physical voulume using the `sudo pvs` command

![11](https://user-images.githubusercontent.com/47898882/126381242-8bd82a81-acb3-4f0c-82b3-7f50188d4370.JPG)

- Use the `vgcreate` utility to add them all to the same volume group

```
$ sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
```
- Verify they have all been added to the same volume group by using the `sudo vgs` command

![22](https://user-images.githubusercontent.com/47898882/126381249-ba1b4dc1-a229-4c6a-a2ee-47930813703b.JPG)

- Make the volume group you created, a logical volume by using the command below. Divide the logical volumes into half and make apps-lv store data concerning the website, while logs-lv stores data for logs

```
$ sudo lvcreate -n apps-lv -L 14G webdata-vg
$ sudo lvcreate -n logs-lv -L 14G webdata-vg
```

- As before, verify the logical volumes have been created using `sudo lvs` command.

![33](https://user-images.githubusercontent.com/47898882/126381252-c1ba7f02-4d16-4ce3-aee0-1426be967d61.JPG)

- We have successfully configured our partitions into logical volumes. To check your setup use `lsblk`

![55](https://user-images.githubusercontent.com/47898882/126381232-b8ebb6ec-464c-4bde-9bb9-7d970475ce91.JPG)

- Create a filesystem for the logical volumes using `ext4` filesystem.

```
$ sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
$ sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```

- Create a directory called /var/www/html to store our web files, also create another directory called /home/recovery/logs, this directory will store the log files in the /var/log directory temporarily until we mount it on the logical volume we created for our logs i.e (/dev/web-data-vg/log-lv).

```
$ mkdir -p /var/www/html
$ mkdir -p /home/recovery/logs
```
- Mount /var/www/html on the `apps-lv` logical volume 

```
$ sudo mount /dev/webdata-vg/apps-lv /var/www/html
```

- Transfer the log files in the /var/log directory into the /home/recovery/logs directory temporarily. 

*Note: If we just mount in the /var/log directory without transferring the files/folder out, all of the content will be deleted, and these files/folders are very important for our setup to function*.

```
$ sudo rsync -av /var/log/. /home/recovery/logs/
```

- Mount /var/log on the `logs-lv` logical volume

```
$ sudo dev/webdata-vg/logs-ls /var/log
```

- Restore the files/folders back into the /var/log directory

```
$ sudo rsync -av /home/recovery/logs/log/. /var/log
```
- To make sure that the mount configuration just performed is persisted, we have to update the '/etc/fstab' file. But before the update is made, let us copy the block `ids` for our logical volumes using the `blkid` command

![44](https://user-images.githubusercontent.com/47898882/126381255-580ad66a-4b1f-4ab1-a6fd-1d6916c16181.JPG)


- Open the /etc/fstab/ with the vim editor and make your changes. After editing your file should look like below:

```
$ sudo vi /etc/fstab
```
![66](https://user-images.githubusercontent.com/47898882/126395500-2779383c-1d68-4cb5-bdf5-1b7c1e658ae8.JPG)

- Run `mount -a` to test that the update was successful and restart your `daemon-reload` also

```
$ mount -a
$ sudo systemctl daemon-reload
```

- Repeat all of the steps above for the `database-server`, the only difference is that there'll be only one logical volume `db-lv` and it will be mounted on the /db directory to store database related files. The size of the logical volume can be between 1-29GB, remember the size of our attached volumes are 10gb each. 

*Note: This logical volume will be storing database related content, so it is advisable to use a large chunk of the logical volume size i.e >20GB. This should vary based on the requirements for the application.*

## **Install Wordpress on the EC2 Web Server**
- Update you redhat repo and install these packages

```
$ sudo yum -y update  
$ sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```

- Restart the httpd service
```
$ sudo systemctl enable httpd
$ sudo systemctl start httpd
```

- Install all of php dependencies, most of this installations are specific to the redhat o.s

```
$ sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
$ sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
$ sudo yum module list php
$ sudo yum module reset php
$ sudo yum module enable php:remi-7.4
$ sudo yum install php php-opcache php-gd php-curl php-mysqlnd
$ sudo systemctl start php-fpm
$ sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```
- Create A folder called `wordpress` in the home directory.

```
$ mkdir wordpress && cd wordpress
```
- Download the wordpress and then copy it to the /var/www/html folder. The wordpress folder will be a gzip file, unzip using the 'tar xzvf' command.

```
$ sudo wget http://wordpress.org/latest.tar.gz
$ sudo tar xzvf latest.tar.gz
$ sudo rm -rf latest.tar.gz
$ cp wordpress/wp-config-sample.php wordpress/wp-config.php
$ cp -R wordpress /var/www/html/
```
- Update the wp-config.php file with the database name, database user, and the ip address.

- Configure selinux policies using this commands:

```
$ sudo chown -R apache:apache /var/www/html/wordpress
$ sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
$ sudo setsebool -P httpd_can_network_connect=1
$ sudo setsebool -P httpd_can_network_connect_db=1
```

## **Install MySQL on the EC2 Database Server**

- Install MySQL on the database server

```
$ sudo yum update
$ sudo yum install mysql-server
```
- Check the status of the mysqld service to confirm it is active and enabled

```
$ sudo systemctl mysqld
```

- Create a Database and assign privileges a user. Also configure the user's ip address to be the webserver's ip, as the webserver sending requests and receiving responses from the database-server

```
mysql> sudo mysql
mysql> CREATE DATABASE wordpress;
mysql> CREATE USER `user`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'password';
mysql>GRANT ALL ON wordpress.* TO 'user'@'<Web-Server-Private-IP-Address>';
mysql> FLUSH PRIVILEGES;
mysql> SHOW DATABASES;
mysql> exit
```
- Allow inbound connections on the AWS console from port 3306, and let the source allow traffic from the web-server's ip address.

![77](https://user-images.githubusercontent.com/47898882/126400818-3f7fc586-ae26-4f41-8556-024bb7b4f91b.JPG)

- Install MySQL client on the web-server instance to test that it can connect with the database-server instance.

```
$ sudo yum install mysql-client
```

- Use the command below to to connect

```
$ sudo mysql -u admin -p -h <DB-Server-Private-IP-address>
```

![88](https://user-images.githubusercontent.com/47898882/126403142-b64fffcd-1150-4f9e-8da4-b143c54107ac.JPG)

- Change permissions for the wordpress folder in the /var/www/html directory, in order for apache to be able the serve the website

```
$ sudo chown apache:apache wordpress
```

- Test your setup with the public ip/dns of your web server instance http://Web-Server-Public-IP-Address/wordpress/


![33](https://user-images.githubusercontent.com/47898882/126402263-809dc592-9220-4034-b934-0d47855e9012.JPG)

![32](https://user-images.githubusercontent.com/47898882/126402254-e8fb2e41-5925-4134-97b6-b6a2a4fb0723.JPG)

**This is how to deploy a full scale web solution using wordpress and configure the linux storage subsystem using LVM (Logical Volume Manager)**

# **BLOCKERS**
There was error connecting to database when i tried opening the site on the browser due to the fact that I failed to update the wp-config.php file with the database name, user and host address in the /var/www/html/wordpress directory. It was fixed when i noticed the error and corrected it. 







