# **Devops Tooling Website Solution**

This project entails an implementation of a Devops Tooling Website, that will consist of the various tools which devops engineers use within their infrastrustures. This project will make use of the following components:

- Infrastructure: AWS
- Webserver Linux: Red Hat Enterprise Linux 8
- Database Server: Ubuntu 20.04 + MySQL
- Storage Server: Red Hat Enterprise Linux 8 + NFS Server
- Programming Language: PHP

The infrastructure as shown below will consist of three webservers, one database server and an nfs server. NFS stands for Network File System, it is a distributed file system protocol that allows a client access files over a network. This system architecture allows the webservers and nfs server to be mapped/synchronized over an ip network i.e whatever files get stored in the the webservers will be the same on the nfs servers, this allows for the webservers to be *stateless*, which means they can be terminated or removed but that will not affect the integrity of the data that has been stored on the nfs server. All of the webserver will also share a common database.


![1](https://user-images.githubusercontent.com/47898882/128235739-5667183a-9cc6-41a4-af9a-3c5b9b1c9cc9.JPG)

# **Launch EC2 instance for the NFS Server (Partitioning, Configuring with LVM & Mounting)**

- Create a linux virtual servers and configure one to act as a `nfs-server`

- Create Volumes for the `nfs-server` by going to the volumes section in your AWS console and creating a volume of size 15GB. The caveat is that they must be set to the same AVZ(Availabilty Zone), this is very important.

- After creation, attach the volume to your `nfs-server` instance, then we can begin configuring.

- You can check the newly created volume on the terminal using `lsblk`. We have to partition our newly created volume `xvdf`, this will be made possible by a utility called `gdisk`. GDISK is used in partitioning GPT(GUID Partition Table) type disks and that is the type of disk our ec2 instance has, in the case where it is an MBR(Master Boot Record) disk, it will be configured with another utility known as `fdisk`

![2](https://user-images.githubusercontent.com/47898882/128239276-5ede7c31-0538-4d17-84de-37adc7bce7af.JPG)

- To partition this volume, use the command below

```
$ sudo gdisk /dev/xvdf
```

- When the prompt is loaded and it asks for a command, type `n` to create the partition. The most important option we have to update is what type of partition we want this disk to be. By default it is configured to be a Linux filesystem disk, but we will have to configure it to a Linux LVM disk by using hex code `8E00`, this is gdisks' hex code for Linux LVM. To save your configuration type `w` and this will write your configuration onto those volumes.

- To make the conversion of our partitions into logical volumes, the logical volume manager package has to be installed.

```
$ sudo yum install lvm2
```

- After the volumes have been partitioned successfully, mark it as a physical volume using the `pvcreate` utility.

```
$ sudo pvcreate /dev/xvdf1
```
- Confirm that it has been marked as physical voulume using the `sudo pvs` command

![3](https://user-images.githubusercontent.com/47898882/128239586-3d82742d-5563-4f4a-9679-c10cf512df2e.JPG)

- Use the `vgcreate` utility to the add the volume to a volume group

```
$ sudo vgcreate webdata-vg /dev/xvdf1
```

- Verify that it has been added to the volume group by using the `sudo vgs` command

![4](https://user-images.githubusercontent.com/47898882/128239953-fbe02e25-71a1-4eaf-93a5-0d8c37692b15.JPG)

- Make the volume group you created, a logical volume by using the command below. Divide the logical volumes into three and make lv-apps store data concerning the website, while lv-logs stores data for logs, and lv-opt.

```
$ sudo lvcreate -n lv-apps -L 5G webdata-vg
$ sudo lvcreate -n lv-logs -L 5G webdata-vg
$ sudo lvcreate -n lv-opt -l 100%FREE webdata-vg
```

- As before, verify the logical volumes have been created using `sudo lvs` command.

![55](https://user-images.githubusercontent.com/47898882/128240659-2d6b9e9b-738c-4191-bbbd-586f66573693.JPG)

- We have successfully configured the partition into a logical volumes. To check your setup use `lsblk`

- Create a filesystem for the logical volumes using `xfs` filesystem.

```
$ sudo mkfs -t xfs /dev/webdata-vg/lv-apps
$ sudo mkfs -t xfs /dev/webdata-vg/lv-logs
$ sudo mkfs -t xfs /dev/webdata-vg/lv-opt
```
- Create a directory called /mnt/apps to store our web files, also create another directory called /mnt/logs, this directory will store the log files, and /mnt/opt to be used in a later project

```
$ mkdir /mnt/apps
$ mkdir /mnt/logs
$ mkdir /mnt/opt
```

- Mount /mnt/apps on the `lv-apps` logical volume, mount /mnt/logs on `lv-logs`, also mount /mnt/opt on `lv-opt`

```
$ sudo mount /dev/webdata-vg/lv-apps /mnt/apps
$ sudo mount /dev/webdata-vg/lv-apps /mnt/apps
$ sudo mount /dev/webdata-vg/lv-apps /mnt/apps
```

- To make sure that the mount configuration just performed is persisted, we have to update the '/etc/fstab' file. But before the update is made, let us copy the block `ids` for our logical volumes using the `blkid` command

![6](https://user-images.githubusercontent.com/47898882/128244980-ea0a1769-1e17-4e84-bb75-bd5e281ec45c.JPG)

- Open the /etc/fstab/ with the vim editor and make your changes. After editing your file should look like below:

```
$ sudo vi /etc/fstab
```

![7](https://user-images.githubusercontent.com/47898882/128244989-089059df-854d-470a-ab31-0de926d93e9f.JPG)

- Run `mount -a` to test that the update was successful and restart your `daemon-reload` also

```
$ mount -a
$ sudo systemctl daemon-reload
```

- Setup permissions to allow user, group and anyone to read, write and execute the files within the /mnt/apps, /mnt/logs, /mnt/opt directories.

```
$ sudo chmod -R 777 /mnt/apps
$ sudo chmod -R 777 /mnt/logs
$ sudo chmod -R 777 /mnt/opt
```

- Change ownership within these directories into *nobody*, by default it will be set to *root*. This is very important because the webservers will be reading, writing to these directories. If it is set to root, the webservers will not be able to read nor write files into these directories

```
$ sudo chown -R nobody: /mnt/apps
$ sudo chown -R nobody: /mnt/logs
$ sudo chown -R nobody: /mnt/opt
```

- Install the nfs server, enable the service and chec the status to be sure it is active and enabled.

```
$ sudo yum install nfs-utils -y
$ sudo systemctl start nfs-server.service
$ sudo systemctl enable nfs-server.service
$ sudo systemctl status nfs-server.service
```
- Get the *subnet_cidr* address of the nfs server, this is the subnet all of the clients/webservers will connect from.

*Note: This a security flaw, in production it is advised to configure the webservers on different subnets so as to improve the level of security.*

![9](https://user-images.githubusercontent.com/47898882/128246655-5e1cc65b-7cd6-49d3-a2a2-d1bd582e0c4a.JPG)

- Enable access to the nfs server for clients that are within the same subnet. In order to do this, open the /etc/exports file and paste in the command below:

![8](https://user-images.githubusercontent.com/47898882/128246534-300c32fe-7272-45e4-b104-43fd170e898b.JPG)

- To perform the export, run the command below:

```
$ sudo exportfs -arv
```

- Configure Inbound connection to allow traffic from the following ports: TCP 111, UDP 111, UDP 2049.

# **Install MySQL on the EC2 Database Server**

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
mysql> CREATE DATABASE tooling;
mysql> CREATE USER `webaccess`@`<Nfs-Server-Subnet-Cidr>` IDENTIFIED BY 'password';
mysql> GRANT ALL ON tooling.* TO 'webaccess'@'<Nfs-Server-Subnet-Cidr>';
mysql> FLUSH PRIVILEGES;
mysql> SHOW DATABASES;
mysql> exit
```

# **Launch EC2 Instances for Web Servers**
- Install nfs client package

```
$ sudo yum install nfs-utils nfs4-acl-tools -y
```

- Install apache

```
$ sudo yum install httpd -y
```

- Create the /var/www directory and mount the nfs exports for the apps i.e /mnt/apps

```
$ sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```

- Mount /var/log on the nfs exports for logs i.e /mnt/logs. Before performing this action, all of the log files from the /var/log directory have to be copied and stored in another directory(/home/logs). The log files are very important for the functions of the virtual machine. In an instance where this isn't done, all of the log files will be deleted.

```
$ sudo rsync -av /var/log/. /home/logs
```

- Perform the mount

```
$ sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/logs /var/log
```

- Copy the files back into the /var/log directory

```
$ sudo rsync -av /home/logs/. /var/log
```

- Persist the configuration, by adding the following in the /etc/fstab file.

```
172.31.17.143:/mnt/apps /var/www nfs defaults 0 0
172.31.17.143:/mnt/logs /var/log nfs defaults 0 0
```

![10](https://user-images.githubusercontent.com/47898882/128255321-94192f00-044f-496e-9156-77826aea4613.JPG)

- Verify the nfs server was properly mounted on the webserver. To do this create a file in the /var/www directory of the ebserver and check /mnt/apps of the nfs to find out if the same file can be found. If it is found, then the mount was successful.

- Fork this [repo](https://github.com/darey-io/tooling). This is where all the files to be served will be gotten from.

- Install git and clone repo

```
$ sudo yum install git -y
$ git clone https://github.com/darey-io//tooling
```

- The folder *tooling* will be cloned to your root directory. Copy the html folder in that folder into the html folder in the /var/www/html directory.

```
$ sudo rsync -av html/. /var/www/html
```

- Install MySQL on the webserver in order to be able to communicate with the database server

```
$ sudo yum install mysql-server
```

- Populate the database with the users table. Within the tooling folder, an sql script called tooling-db.sql will be found. To perform this action, use the command below.

```
$ sudo mysql -u <database-ipaddress> -h -p tooling<tooling-db.sql
```

- Install php and all it's dependencies

```
$ sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
$ sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
$ sudo yum module list php
$ sudo yum module reset php
$ sudo yum module enable php:remi-7.4
$ sudo yum install php php-opcache php-gd php-curl php-mysqlnd
$ sudo systemctl start php-fpm
$ sudo systemctl enable php-fpm
$ sudo setsebool -P httpd_execmem 1
$ sudo setsebool -P httpd_can_network_connect=1
$ sudo setsebool -P httpd_can_network_connect_db=1
```

- Enable & start the php-fpm service.

- You can test your setup, but before you do so, if you check the status of your httpd service it'll be inactive. This happens due to the fact that SELinux is enabled. Security Enhanced Linux (selinux) is an extra layer of security enabled by default on Redhat and CentOS linux distributions. At this point SELinux has blocked port 80 which is meant to be used to access the website over the web. In other words, ports need to be added to a context or it will appear that they are blocked, even though they have been opened in the firewall. To combat this issue, use the *semange* command. The command semanage is used to view and change selinux configuration settings.

```
$ sudo semanage port -a -t http_port_t -p tcp 80
```

- Another alternative is to run `sudo setenforce 0` and set SELinux to disabled in the /etc/sysconfig/selinux directory. Due to the fact that SELinux adds an extra layer of security. I'll advise to go with the first option

- Restart apache and php-fpm services

```
$ sudo systemctl restart httpd
$ sudo systemctl restart php-fpm
```

- Copy the public ip addrsee of your instance and test your setup in your web browser.

![11](https://user-images.githubusercontent.com/47898882/128258621-3663f225-1f93-4bc3-806e-32934e456719.JPG)

![12](https://user-images.githubusercontent.com/47898882/128258630-e2ae21ba-cf6b-4756-a59c-67b0c115fe1b.JPG)

Congratulations!!. We have succesfully created our whole setup and deployed the devops tooling website on AWS.
