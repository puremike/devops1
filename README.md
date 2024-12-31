# AWS Cloud Deployment Guide: Lift and Shift Project

## Project Overview

**Project Name**: AWS Cloud for Web Setup (Lift and Shift)

**Description**: The process of migrating an on-premises environment and workload to the cloud. This guide details the steps taken to deploy an application to AWS.

**Components**:

- **Frontend**: Hosted on a Tomcat EC2 instance.
- **Backend Services**: Includes MySQL, Memcached, and RabbitMQ hosted on separate EC2 instances.

---

## Steps

### 1. Fork the Project from GitHub

- Clone the project source code using the command:
  ```bash
  git clone -b awsliftandshift https://github.com/hkhcoder/vprofile-project.git
  ```
- Understand the project setup for proper deployment to AWS.

---

### 2. Understand Project Parts

- The frontend (Tomcat instance) communicates with backend services (MySQL, Memcached, RabbitMQ).

---

### 3. AWS Security Groups Setup

#### a. Security Group for Tomcat EC2 Instance (vprofile-app-tomcat):

- **Inbound Rules**:
  - SSH: Port 22 (Accessible only from your IP address).

#### b. Security Group for Backend Services (vprofile-backend-srv):

- **Inbound Rules**:
  - MySQL: Port 3306 (Accessible by app01 security group).
  - Memcached: Port 11211 (Accessible by app01 security group).
  - RabbitMQ: Port 5672 (Accessible by app01 security group).
  - SSH: Port 22 (Accessible only from your IP address).
  - All Traffic: Accessible by the backend-service security group itself.

#### c. Security Group for ELB (vprofile-elb):

- **Inbound Rules**:
  - HTTP: Port 80 (Accessible to all IPs).
  - HTTPS: Port 443 (Accessible to all IPs).

---

### 4. Create Key Pair

- Generate an AWS key pair for EC2 instance access.

---

### 5. Launch EC2 Instances

#### a. MySQL Instance (vprofile-db01):

- **User Data Script**:
  ```bash
  #!/bin/bash
  DATABASE_PASS='admin123'
  sudo dnf update -y
  sudo dnf install git zip unzip -y
  sudo dnf install mariadb105-server -y
  sudo systemctl enable --now mariadb
  cd /tmp/
  git clone -b main https://github.com/hkhcoder/vprofile-project.git
  sudo mysqladmin -u root password "$DATABASE_PASS"
  sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
  sudo mysql -u root -p"$DATABASE_PASS" accounts </tmp/vprofile-project/src/main/resources/db_backup.sql
  ```

#### b. Memcached Instance (vprofile-mc01):

- **User Data Script**:
  ```bash
  #!/bin/bash
  sudo dnf install memcached -y
  sudo systemctl enable --now memcached
  sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
  sudo systemctl restart memcached
  ```

#### c. RabbitMQ Instance (vprofile-rmq01):

- **User Data Script**:
  ```bash
  #!/bin/bash
  dnf install -y erlang rabbitmq-server
  systemctl enable --now rabbitmq-server
  sudo rabbitmqctl add_user test test
  sudo rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
  ```

#### d. Tomcat Instance (vprofile-app01):

- **User Data Script**:
  ```bash
  #!/bin/bash
  sudo apt update && sudo apt upgrade -y
  sudo apt install openjdk-17-jdk tomcat10 -y
  ```

---

### 6. Configure Route53 Hosted Zone

- Create private hosted zone and A records. The hosted zone and records can be configured to use a domain register in/out of Amazon:
  - `db01.vprofile.in` → MySQL private IP
  - `mc01.vprofile.in` → Memcached private IP
  - `rmq01.vprofile.in` → RabbitMQ private IP
  - `app01.vprofile.in` → Tomcat private IP
- Verify using:
  ```bash
  ping -c 4 db01.vprofile.in
  ```

---

### 7. Update Application Properties

Edit `src/main/resources/application.properties` to match Route53 records:

```properties
jdbc.url=jdbc:mysql://db01.vprofile.in:3306/accounts
memcached.active.host=mc01.vprofile.in
rabbitmq.address=rmq01.vprofile.in
```

---

### 8. Create S3 Bucket

- Upload the application artifact (`vprofile-v2.war`) to the S3 bucket.

---

### 9. Deploy Application on Tomcat

- SSH into Tomcat instance:
  ```bash
  ssh -i <keypair.pem> ubuntu@<public-ip>
  ```
- Copy the artifact from S3 to Tomcat:
  ```bash
  aws s3 cp s3://<bucket_name>/vprofile-v2.war /tmp/
  cp /tmp/vprofile-v2.war /var/lib/tomcat/webapps/ROOT.war
  systemctl restart tomcat
  ```

---

### 10. Set Up ELB

- Create a Load Balancer (ALB):
  - Protocol: HTTP (80) and HTTPS (443).
  - Target Group: Route traffic to Tomcat instance (port 8080).

---

### 11. Configure Auto Scaling Group (ASG)

- Automatically scale Tomcat instances based on traffic or CPU load.

---

## Conclusion

This guide provides a comprehensive step-by-step process for deploying a web application to AWS using the Lift and Shift approach.
