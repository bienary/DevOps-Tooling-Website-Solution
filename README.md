# **DevOps-Tooling-Website-Solution**
## **🚀 Project Overview**

> The DevOps Tooling Website Solution project is designed to develop a centralized, web-based platform 🌐 that provides seamless access to a suite of DevOps tools 🛠️. This solution aims to streamline and automate key processes involved in software development 💻, testing, deployment, and monitoring, enabling teams to work more efficiently and collaboratively 🤝.

> By integrating essential DevOps functionalities into a user-friendly website, the platform will support continuous integration and delivery (CI/CD) 🔄, infrastructure management, and real-time monitoring. This will help reduce manual effort, minimize errors ❌, and accelerate software release cycles, ultimately improving overall project delivery and system reliability ✅.

> The project focuses on scalability and adaptability 🔧 to support multiple projects and evolving organizational needs, fostering a culture of agility and continuous improvement 🌟.

## DevOps Tools Overview

- Jenkins 🧩 – A free and open-source automation server used to create and manage CI/CD pipelines, helping teams build, test, and deploy code efficiently.

- Kubernetes ☸️ – An open-source container orchestration platform for automating the deployment, scaling, and management of containerized applications.

- JFrog Artifactory 🏗️ – A universal repository manager that supports all major packaging formats, CI servers, and build tools, serving as a single source of truth for binary artifacts.

- Rancher 🐳 – An open-source container management platform that simplifies the deployment and operation of Docker and Kubernetes clusters in production environments.

- Grafana 📊 – A powerful, open-source analytics and visualization platform that enables interactive dashboards and monitoring across multiple data sources.

- Prometheus ⏱️ – A leading open-source monitoring and alerting toolkit featuring a time-series database, flexible query language, and modern alerting capabilities.

- Kibana 📈 – A free and open user interface for visualizing Elasticsearch data and interacting with the broader Elastic Stack, offering real-time insights and dashboards.

## 🛠️ Project Architecture

> In this project, you will implement a robust solution that includes the following core components and technologies:

☁️ Infrastructure: Amazon Web Services (AWS) – A scalable and reliable cloud platform for hosting and managing your application and infrastructure.

🖥️ Web Server OS: Red Hat Enterprise Linux 8 (RHEL 8) – A stable and secure enterprise-grade Linux distribution for hosting the PHP web application.

🗄️ Database Server: Ubuntu 24.04 with MySQL – A modern Linux OS combined with a powerful open-source relational database system to manage application data.

💾 Storage Server: RHEL 8 with NFS Server – A dedicated storage solution using Network File System (NFS) to enable shared access across environments.

💻 Programming Language: PHP – A widely-used server-side scripting language suited for web development and backend logic.

🔗 Code Repository: GitHub – A cloud-based platform for version control and collaborative development using Git.

<img width="832" height="513" alt="image" src="https://github.com/user-attachments/assets/562a5651-4a4e-4188-a05d-5332f121df3a" />

---

## 🛠️ Step 1 – Set Up the NFS Server
- Launch an EC2 instance with Red Hat Enterprise Linux Operating System.

<img width="1151" height="183" alt="Screenshot From 2025-11-13 07-03-01" src="https://github.com/user-attachments/assets/ff4bf7a6-11ff-469d-9978-dd5526d05d92" />


- Configure Logical Volume Management on the Server

- Instead of formatting the disks as `ext4` you will have to format them as `xfs`
- Ensure there are 3 Logical Volumes. `lv-opt` `lv-apps`, and `lv-logs`
- Create mount points on /mnt directory for the logical volumes as follows:

> Mount lv-apps on /mnt/apps - To be used by webservers

> Mount lv-logs on /mnt/logs - To be used by webserver logs

> Mount lv-opt on /mnt/opt - To be used by Jenkins server in the next project

### **Create and Mount Volumes**

> Create 3 EBS volumes in the same Availability Zone as the NFS Server EC2, each of 10GB, and attach them one by one to the NFS Server.

<img width="1155" height="223" alt="Screenshot From 2025-11-13 07-19-47" src="https://github.com/user-attachments/assets/ada52454-cdcb-4f4e-851b-ef85bbc6c5bd" />


### **Server Configuration**

> Log in to the Linux terminal to begin configuration:
```
ssh -i <Your-private-key.pem> ec2-user@<EC2-Public-IP-address>
```

- Run `lsblk` to view the block devices connected to the server.

<img width="1019" height="291" alt="image" src="https://github.com/user-attachments/assets/00d22f01-c036-4bd6-ba7b-5f1c180b7a61" />

### **Partition the Disks**

- We’ll use the `fdisk` command to create a single partition on each of the three available disks.

```
sudo fdisk /dev/nvme1n1

sudo fdisk /dev/nvme2n1

sudo fdisk /dev/nvme3n1
```

> NOTE:

> g → to create a new GPT partition table

> n → add a new partition
Press Enter to accept defaults for partition number, first sector, and last sector.

> w → write the changes to disk and exit.

<img width="1313" height="729" alt="image" src="https://github.com/user-attachments/assets/6c19ab9b-bb2b-4e1f-b46d-5bd255c521b2" />


### **Install the Logical Volume Manager (LVM2) package**
```
sudo yum install lvm2 -y
```

### **Create LVM Physical Volumes**

- Run the `pvcreate` command to initialize each of the three disks as Physical Volumes (PVs) for LVM use.
```
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```
```
sudo pvs
```

### **Set Up the Volume Group**
- Run `vgcreate` to add the three PVs to a new Volume Group called `webdata-vg`.

```
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```
```
sudo vgs
```

### **Logical Volume Creation**
- Run `lvcreate` to create three LVM logical volumes: `lv-apps`, `lv-logs`, and `lv-opt`.

```
sudo lvcreate -n lv-apps -L 9G webdata-vg

sudo lvcreate -n lv-logs -L 9G webdata-vg

sudo lvcreate -n lv-opt -L 9G webdata-vg
```
```
sudo lvs
```

### **Format Logical Volumes with XFS**
- Use the `mkfs.xfs` command to format the logical volumes with the XFS filesystem.

```
sudo mkfs.xfs /dev/webdata-vg/lv-apps

sudo mkfs.xfs /dev/webdata-vg/lv-logs

sudo mkfs.xfs /dev/webdata-vg/lv-opt
```

### **Create mount points on the /mnt directory**
```
sudo mkdir -p /mnt/{apps,logs,opt}

sudo mount /dev/webdata-vg/lv-apps /mnt/apps

sudo mount /dev/webdata-vg/lv-logs /mnt/logs

sudo mount /dev/webdata-vg/lv-opt /mnt/opt
```

### **Install NFS server, then configure it to start on reboot:**
```
sudo yum update -y

sudo yum install nfs-utils -y

sudo systemctl start nfs-server.service

sudo systemctl enable nfs-server.service

sudo systemctl status nfs-server.service
```

- **Export the mounts for webservers' `subnet cidr` to connect as clients.**
> set up permission that will allow our Web servers to read, write and execute files on NFS:
```
sudo chown -R nobody: /mnt/apps /mnt/logs /mnt/opt

sudo chmod -R 777 /mnt/apps /mnt/logs /mnt/opt

sudo systemctl restart nfs-server.service
```

### **Configure access to NFS for clients within the same subnet (example of Subnet CIDR - 172.31.32.0/20 ):**

> To check your subnet CIDR: Open your EC2 details in the AWS web console, and locate the 'Networking' tab and open the Subnet link.
```
sudo vi /etc/exports
sudo exportfs -arv
```

> For NFS Server to be accessible from the client, set the following ports: TCP 111, UDP 111, UDP 2049, and TCP 2049. Set the Web Server subnet CIDR as the source in the security group.

---

## **Step 2: Database Server Configuration**
- Create an Ubuntu EC2 Instance for the DB Server

> In the Security Group:

> Allow SSH (port 22) for your IP (for management).

> Allow MySQL/Aurora (port 3306) and use the Subnet CIDR as the source (for secure DB access).

- Update & Upgrade Ubuntu Packages
```
sudo apt update && sudo apt upgrade -y
```

### **Install MySQL Server**
```
sudo apt install mysql-server -y
```

### **Run MySQL secure installation script:**
```
sudo mysql_secure_installation
```

### **Create a user, and grant privileges:**
```
sudo mysql

CREATE DATABASE tooling;

CREATE USER 'webaccess'@'DB-private IP' IDENTIFIED BY 'StrongPassword!';

GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'DB-private IP';

FLUSH PRIVILEGES;

SHOW DATABASES;

EXIT;
```

### **Configure MySQL Bind Address and Restart the Service**
```
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```
> Locate the line bind-address = 127.0.0.1 and change it to:

> bind-address = 0.0.0.0

```
sudo systemctl restart mysql
sudo systemctl status mysql
```

---

### **Step 3: Prepare the Web Servers**
> We need to make sure that our Web Servers can serve the same content from shared storage solutions, in our case - NFS Server and MySQL database.

- We will do the following in our next steps:

> Configure NFS client (this step must be done on all three servers)

> Deploy a Tooling application to our Web Servers into a shared NFS folder

> Configure the Web Servers to work with a single MySQL database

- Launch a New EC2 Instance with RHEL Operating System

- Install NFS Client
```
sudo yum install nfs-utils nfs4-acl-tools -y
```

- Mount /var/www/ and Target the NFS Server's Export for apps
```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```
```
sudo vi /etc/fstab
```

- Add the following line:

```
<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
```

### **Install Remi's Repository, Apache and PHP**

```
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

setsebool -P httpd_execmem 1
```

> Repeat the Configuration (Steps 1–5) for Each of the Two Additional Web Servers.

### **Step 4: Verify Apache Files and Directories**

- Verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. If you see the same files - it means NFS is mounted correctly.

> Create a test file on `WebServer 1` and check if the same file is accessible from on `WebServer 2`.

- Mount Log Folder to NFS Server

> On the Web Server, locate the Apache log path and mount it to the corresponding log export on the NFS server. Ensure the mount is automatically re-established after reboot.

```
sudo mkdir -p /var/log/httpd

sudo mount -t nfs -o rw,nosuid <NFS-Private-IP>:/mnt/logs /var/log/httpd
```
```
sudo vi /etc/fstab
```

- Add the following line:

```
<NFS-Private-IP>:/mnt/logs /var/log/httpd nfs defaults 0 0
```

### **Fork the Tooling Source Code**

> Fork the tooling source code from StegHub Github Account to your Github account.

### **Deploy the tooling website's code to the Webserver**

> Ensure that the html folder from the repository is deployed to /var/www/html

> Install Git and clone the tooling repository:

- Adjust Permissions:

```
sudo chown -R apache:apache /var/www/html

sudo chmod -R 755 /var/www/html

sudo setenforce 0
```

- Change the line:

```
sudo vi /etc/sysconfig/selinux
```

> In the configuration file, find the line SELINUX=enforcing and modify it to SELINUX=disabled.

> Restart:

```
sudo systemctl restart httpd
```

### **Update the website's configuration to connect to the database**

- Run this on each of the web servers:

```
sudo vi /var/www/html/functions.php
```

- Locate the following line and replace it with your own details.

```
$db = mysqli_connect('DB_Private-IP', 'DB_Username', 'DB_Password', 'tooling');
```

- Run the `tooling-db.sql` script to initialize the database.

```
sudo mysql -h <databse-private-ip> -u <db-username> -p tooling < ~/tooling/tooling-db.sql
```

- Connect to the Database Server from your Web Server instance.

```
sudo mysql -h <databse-private-ip> -u <db-username> -p
```

### **Create a New Admin User in MySQL**

> Create in MySQL a new admin user with username: myuser and password: password:

```
sudo mysql -u <db-username> -p -h <db-Private-IP>

USE tooling;

INSERT INTO users (id, username, password, email, user_type, status)

VALUES (2, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');

EXIT;
```

> Open your browser and navigate to http://<Web-Server-Public-IP>/index.php, replacing <Web-Server-Public-IP> with the public IP addresses of Web Server 1, Web Server 2, and Web Server 3.

### **Congratulations!**

You have just set up a `LAMP web application` architecture for a DevOps team, using `Remote Database` and `NFS servers` to ensure high availability and centralized data management.
