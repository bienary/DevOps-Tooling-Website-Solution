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
- Configure Logical Volume Management on the Server
- Instead of formatting the disks as `ext4` you will have to format them as `xfs`
- Ensure there are 3 Logical Volumes. `lv-opt` `lv-apps`, and `lv-logs`
- Create mount points on /mnt directory for the logical volumes as follows:

> Mount lv-apps on /mnt/apps - To be used by webservers

> Mount lv-logs on /mnt/logs - To be used by webserver logs

> Mount lv-opt on /mnt/opt - To be used by Jenkins server in the next project

### **Create and Mount Volumes**

> Create 3 EBS volumes in the same Availability Zone as the NFS Server EC2, each of 10GB, and attach them one by one to the NFS Server.

### **Server Configuration**

> Log in to the Linux terminal to begin configuration:
```
ssh -i <Your-private-key.pem> ec2-user@<EC2-Public-IP-address>
```
- Run `lsblk` to view the block devices connected to the server.

### **Partition the Disks**

- We’ll use the `fdisk` command to create a single partition on each of the three available disks.

```
sudo gdisk /dev/nvme1n1

sudo gdisk /dev/nvme2n1

sudo gdisk /dev/nvme3n1
```

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

