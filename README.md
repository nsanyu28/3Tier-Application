# Deploy a Three-Tier Web Application on AWS

> From zero to a live, auto-scaling AWS application: a secure three-tier architecture with a public web tier, private application tier, and protected database tier distributed across two Availability Zones.

**Author:** Sanyukta Nirmal  
**LinkedIn:** [linkedin.com/in/sanyukta-nirmal-aws-sd](https://www.linkedin.com/in/sanyukta-nirmal-aws-sd)  
**Source Code:** [3Tier-Application](https://github.com/nsanyu28/3Tier-Application)

---

## Table of Contents

- [Introduction](#introduction)
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [AWS Services Used](#aws-services-used)
- [Network Design](#network-design)
- [Security Groups](#security-groups)
- [Project Implementation](#project-implementation)
  - [1. Create the VPC and Subnets](#1-create-the-vpc-and-subnets)
  - [2. Create Internet Gateway, NAT Gateways, and Route Tables](#2-create-internet-gateway-nat-gateways-and-route-tables)
  - [3. Create Security Groups](#3-create-security-groups)
  - [4. Create the Database Subnet Group and RDS](#4-create-the-database-subnet-group-and-rds)
  - [5. Create the IAM Role](#5-create-the-iam-role)
  - [6. Create an S3 Bucket and Upload the Source Code](#6-create-an-s3-bucket-and-upload-the-source-code)
  - [7. Create and Configure the Test App Server](#7-create-and-configure-the-test-app-server)
  - [8. Create an AMI of the App Server](#8-create-an-ami-of-the-app-server)
  - [9. Create the App Tier Target Group, Launch Template, and Internal Load Balancer](#9-create-the-app-tier-target-group-launch-template-and-internal-load-balancer)
  - [10. Create and Configure the Test Web Server](#10-create-and-configure-the-test-web-server)
  - [11. Create an AMI of the Web Server](#11-create-an-ami-of-the-web-server)
  - [12. Create the Web Tier Target Group, Launch Template, and External Load Balancer](#12-create-the-web-tier-target-group-launch-template-and-external-load-balancer)
  - [13. Create Auto Scaling Groups](#13-create-auto-scaling-groups)
  - [14. Access and Verify the Application](#14-access-and-verify-the-application)
  - [15. Clean Up the Infrastructure](#15-clean-up-the-infrastructure)
- [Quick Troubleshooting](#quick-troubleshooting)
- [Notes](#notes)

---

## Introduction

This project demonstrates how to deploy a three-tier web application on Amazon Web Services (AWS). The architecture separates the application into three layers:

| Tier | Layer | Responsibility |
|---|---|---|
| **Web Tier** | Presentation / Frontend | Displays the user interface and collects user input. |
| **App Tier** | Application / Backend | Processes requests, runs application logic, and communicates with the database. |
| **Database Tier** | Data Storage | Stores and manages application data. |

A request travels from the user through the external load balancer to the web tier, then through the internal load balancer to the app tier. The app tier communicates with the database and returns the result through the same path.

## Project Overview

The project builds a highly available, secured, scalable, and fault-tolerant three-tier application inside a custom VPC. The resources are distributed across two Availability Zones in the AWS Mumbai region (`ap-south-1`).

Key features:
- Separate public web and private application/database layers.
- External and internal Application Load Balancers.
- Auto Scaling Groups for the web and app servers.
- Amazon Aurora (MySQL Compatible) for database storage.
- Amazon S3 for application source-code storage.
- IAM role and AWS Systems Manager Session Manager for instance access.

## Architecture

**Request flow:**

```text
User
  |
  v
External Application Load Balancer (HTTP : 80)
  |
  v
Web Tier (Nginx / frontend servers)
  |
  v
Internal Application Load Balancer (HTTP : 80)
  |
  v
App Tier (Node.js servers : 4000)
  |
  v
Aurora MySQL Database (: 3306)
```

The architecture uses:
- **Public subnets** for the web tier so users can reach the application.
- **Private subnets** for the app tier and database tier.
- **Two Availability Zones** to distribute resources.
- **S3** to store the application code used when configuring the EC2 instances.

## AWS Services Used

| # | AWS Service / Component | Purpose |
|---|---|---|
| 1 | Amazon EC2 | Hosts the web and application servers. |
| 2 | Amazon S3 | Stores the application source code and configuration files. |
| 3 | Amazon VPC | Provides the isolated network environment. |
| 4 | Auto Scaling Groups | Maintains the configured number of web and app instances. |
| 5 | Application Load Balancer | Distributes traffic to the web and app tiers. |
| 6 | AWS IAM | Controls permissions for EC2 instances. |
| 7 | Amazon RDS (Aurora MySQL Compatible) | Provides managed relational database storage. |
| 8 | Amazon Machine Image (AMI) | Saves the configured app and web server images. |
| 9 | Launch Templates | Defines the configuration used to launch instances. |
| 10 | Target Groups | Registers instances for load-balancer routing and health checks. |
| 11 | Internet Gateway | Provides internet connectivity for public subnets. |
| 12 | NAT Gateway | Allows private-subnet instances to access the internet for outbound downloads. |
| 13 | Route Tables | Controls network traffic routing. |
| 14 | Security Groups | Restricts traffic between the architecture layers. |

## Network Design

**VPC CIDR:** `10.0.0.0/16`  
**AWS Region:** `ap-south-1` (Mumbai)

### Subnet Configuration

| Subnet Name | Type | Availability Zone | CIDR Block |
|---|---|---|---|
| Web-Tier-Subnet-1-Public | Public | ap-south-1a | `10.0.1.0/24` |
| App-Tier-Subnet-1-Private | Private | ap-south-1a | `10.0.2.0/24` |
| DB-Tier-Subnet-1-Private | Private | ap-south-1a | `10.0.3.0/24` |
| Web-Tier-Subnet-2-Public | Public | ap-south-1b | `10.0.4.0/24` |
| App-Tier-Subnet-2-Private | Private | ap-south-1b | `10.0.5.0/24` |
| DB-Tier-Subnet-2-Private | Private | ap-south-1b | `10.0.6.0/24` |

### Route Table Configuration

| Route Table | Subnet Association | Default Route (`0.0.0.0/0`) |
|---|---|---|
| Web-Tier-RT | Both public web subnets | Internet Gateway |
| App-Tier-RT-1 | App-Tier-Subnet-1-Private | NAT-GW-1 |
| App-Tier-RT-2 | App-Tier-Subnet-2-Private | NAT-GW-2 |

## Security Groups

The security groups restrict inbound traffic so each tier accepts traffic from the tier in front of it.

| Security Group | Protocol / Port | Source |
|---|---|---|
| External-Load-Balancer-SG | HTTP (80) | `0.0.0.0/0` |
| Web-Tier-SG | HTTP (80) | External-Load-Balancer-SG |
| Internal-Load-Balancer-SG | HTTP (80) | Web-Tier-SG |
| App-Tier-SG | Custom TCP (4000) | Internal-Load-Balancer-SG |
| DB-Tier-SG | MySQL (3306) | App-Tier-SG |

---

## Project Implementation

### 1. Create the VPC and Subnets

1. Open **VPC → Your VPCs → Create VPC**.
2. Choose **VPC only**.
3. Set the name to `Three-tier-project-VPC`.
4. Set the IPv4 CIDR block to `10.0.0.0/16`.
5. Create the six subnets listed in the [Network Design](#network-design) table, using the specified Availability Zones and CIDR blocks.
6. Enable **Auto-assign public IPv4 address** for both public web subnets.

**Checkpoint:** The VPC and all six subnets should be available. Both public web subnets should have automatic public IPv4 assignment enabled.

### 2. Create Internet Gateway, NAT Gateways, and Route Tables

**Internet Gateway**
1. Open **VPC → Internet Gateways → Create internet gateway**.
2. Create an internet gateway, for example `Internet-GW-Three-tier-app`.
3. Attach it to `Three-tier-project-VPC`.

**NAT Gateways**
1. Open **VPC → NAT Gateways → Create NAT gateway**.
2. Create `NAT-GW-1` in `Web-Tier-Subnet-1-Public` and allocate an Elastic IP.
3. Create `NAT-GW-2` in `Web-Tier-Subnet-2-Public` and allocate an Elastic IP.

**Route Tables**
1. Create `Web-Tier-RT`, `App-Tier-RT-1`, and `App-Tier-RT-2` in the custom VPC.
2. Add the default routes and associate the subnets as shown in the [Route Table Configuration](#route-table-configuration) table.
3. Check the VPC **Resource map** to confirm the routes and subnet associations.

**Checkpoint:** Public web subnets route through the Internet Gateway. Each private app subnet routes through its corresponding NAT Gateway.

### 3. Create Security Groups

1. Open **VPC → Security Groups → Create security group**.
2. Create the five security groups listed in the [Security Groups](#security-groups) table.
3. Add the corresponding inbound rules and select the preceding tier's security group as the source wherever specified.

**Checkpoint:** All five security groups exist in the custom VPC with the intended inbound rules.

### 4. Create the Database Subnet Group and RDS

**DB Subnet Group**
1. Open **RDS → Subnet groups → Create DB subnet group**.
2. Provide a name such as `DB-Subnet-Group-Three-Tier-Application` and select the custom VPC.
3. Select Availability Zones `ap-south-1a` and `ap-south-1b`.
4. Add the DB subnets `10.0.3.0/24` and `10.0.6.0/24`.

**Database**
1. Open **RDS → Databases → Create database**.
2. Choose **Standard create** and **Aurora (MySQL Compatible)**.
3. Select the **Production** template.
4. Set a DB cluster identifier (the guide uses `database-1`).
5. Set a master username (the guide uses `admin`) and create your own strong password.
6. Choose the instance configuration shown in the project guide (`db.t3.medium`).
7. Select your custom VPC and the DB subnet group.
8. Set **Public access** to **No**.
9. Select `DB-Tier-SG` and create the database.

**Important:** Keep your database username and password secure. Do not publish credentials in this repository.

**Checkpoint:** Wait until the database status is **Available**.

### 5. Create the IAM Role

1. Open **IAM → Roles → Create role**.
2. Select **AWS service** as the trusted entity and **EC2** as the use case.
3. Attach these permission policies:
   - `AmazonS3ReadOnlyAccess`
   - `AmazonSSMManagedInstanceCore`
4. Name the role `IAM-role-three-tier-app` and create it.

This role allows EC2 instances to read application files from S3 and connect through Systems Manager Session Manager.

### 6. Create an S3 Bucket and Upload the Source Code

1. Open **S3 → Create bucket** and choose a globally unique bucket name in your project region.
2. Clone the project repository:

   ```bash
   git clone https://github.com/nsanyu28/3Tier-Application.git
   ```

3. Upload the `application-code/` folder to your S3 bucket.
4. In `application-code/app-tier/DbConfig.js`, configure the database connection values before uploading the updated file:

   ```js
   module.exports = Object.freeze({
     DB_HOST: "<RDS Writer Instance Endpoint>",
     DB_USER: "admin",
     DB_PWD: "<YOUR_DB_PASSWORD>",
     DB_DATABASE: "webappdb",
   });
   ```

5. Replace the existing `DbConfig.js` in the S3 bucket with the updated configuration.

**Security note:** Use your own database password and do not commit credentials to GitHub or expose them in screenshots.

### 7. Create and Configure the Test App Server

**Launch the EC2 instance**
1. Open **EC2 → Instances → Launch instances**.
2. Name the instance `Test-APP-Server`.
3. Select **Amazon Linux 2023** and instance type `t2.micro`.
4. Select your custom VPC and `App-Tier-Subnet-1-Private`.
5. Select `App-Tier-SG`.
6. Under advanced details, select the `IAM-role-three-tier-app` instance profile.
7. Launch the instance.

**Connect**
- Select the instance and choose **Connect → Session Manager → Connect**. The IAM role enables Session Manager access without requiring an SSH key in this setup.

**Configure the app server**

Switch to the EC2 user:

```bash
sudo -su ec2-user
```

Install the MySQL client:

```bash
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install mysql-community-client -y
```

Connect to Aurora using the writer endpoint:

```bash
mysql -h <RDS-Endpoint> -u admin -p
```

Create the application database and table, then insert the sample record:

```sql
CREATE DATABASE webappdb;
USE webappdb;

CREATE TABLE IF NOT EXISTS transactions(
  id INT NOT NULL AUTO_INCREMENT,
  amount DECIMAL(10,2),
  description VARCHAR(100),
  PRIMARY KEY(id)
);

INSERT INTO transactions (amount, description)
VALUES ('400', 'groceries');

SELECT * FROM transactions;
exit
```

Copy the app-tier source code from S3. Replace the bucket placeholder with your bucket name:

```bash
cd /home/ec2-user
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/app-tier app-tier --recursive
cd app-tier
sudo chown -R ec2-user:ec2-user /home/ec2-user/app-tier
sudo chmod -R 755 /home/ec2-user/app-tier
```

Install Node.js using NVM and install the application dependencies:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16

npm install -g pm2
npm install
npm audit fix
```

Start the backend and configure it to restart after a reboot:

```bash
pm2 start index.js
pm2 logs
```

After confirming the app is listening on port `4000`, exit the log view with `Ctrl+C`, then run:

```bash
pm2 startup
# Run the sudo env command printed by pm2 startup.
pm2 save

curl http://localhost:4000/health
```

**Checkpoint:** The health endpoint should return a healthy response.

### 8. Create an AMI of the App Server

1. Open **EC2 → Instances** and select `Test-APP-Server`.
2. Choose **Actions → Image and templates → Create image**.
3. Name the image `APP-server-image`.
4. Wait until the AMI status is **Available**.

### 9. Create the App Tier Target Group, Launch Template, and Internal Load Balancer

**Target Group**
1. Open **EC2 → Target Groups → Create target group**.
2. Choose **Instances** as the target type.
3. Set protocol to `HTTP` and port to `4000`.
4. Select the custom VPC.
5. Set the health check path to `/health`.
6. Name the target group `App-Server-TG`.

**Launch Template**
1. Open **EC2 → Launch Templates → Create launch template**.
2. Name it `App-Server-Launch-template`.
3. Select `APP-server-image` under My AMIs.
4. Choose instance type `t2.micro`.
5. Select `App-Tier-SG` and the `IAM-role-three-tier-app` instance profile.
6. Create the launch template.

**Internal Application Load Balancer**
1. Open **EC2 → Load Balancers → Create load balancer → Application Load Balancer**.
2. Set the scheme to **Internal**.
3. Select the custom VPC and both private app subnets:
   - `App-Tier-Subnet-1-Private`
   - `App-Tier-Subnet-2-Private`
4. Select `Internal-Load-Balancer-SG`.
5. Configure an HTTP listener on port `80` forwarding to `App-Server-TG`.
6. Create the load balancer and note its DNS name for the web server configuration.

### 10. Create and Configure the Test Web Server

**Update Nginx configuration**

Before launching the web server, update `nginx.conf` with the DNS name of the internal load balancer:

```nginx
location /api/ {
    proxy_pass http://<Internal-Load-Balancer-DNS-Name>:80/;
}
```

Upload the updated `nginx.conf` to the S3 bucket, replacing the previous file.

**Launch and configure the web server**
1. Launch an EC2 instance named `Test-Web-Server`.
2. Select **Amazon Linux 2023**, your custom VPC, and `Web-Tier-Subnet-1-Public`.
3. Select `Web-Tier-SG` and the `IAM-role-three-tier-app` instance profile.
4. Connect to the instance and copy the web-tier files from S3:

```bash
sudo -su ec2-user
cd /home/ec2-user

sudo aws s3 cp s3://<S3_BUCKET_NAME>/application-code/web-tier web-tier --recursive
sudo chown -R ec2-user:ec2-user /home/ec2-user
sudo chmod -R 755 /home/ec2-user
```

Install Node.js and build the frontend:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16

cd /home/ec2-user/web-tier
npm install
npm run build
```

Install and configure Nginx:

```bash
sudo yum install nginx -y
cd /etc/nginx
sudo mv nginx.conf nginx-backup.conf
sudo aws s3 cp s3://<S3_BUCKET_NAME>/application-code/nginx.conf .
sudo chmod -R 755 /home/ec2-user
sudo service nginx restart
sudo chkconfig nginx on
```

### 11. Create an AMI of the Web Server

1. Open **EC2 → Instances** and select `Test-Web-Server`.
2. Choose **Actions → Image and templates → Create image**.
3. Name the image `Web-Server-Image`.
4. Wait until the AMI status is **Available**.

### 12. Create the Web Tier Target Group, Launch Template, and External Load Balancer

**Target Group**
1. Create a target group in the custom VPC.
2. Set protocol to `HTTP` and port to `80`.
3. Name it `Web-Server-TG`.

**Launch Template**
1. Create a launch template using `Web-Server-Image`.
2. Select `Web-Tier-SG`.
3. Select the `IAM-role-three-tier-app` instance profile.

**External Application Load Balancer**
1. Create an Application Load Balancer with scheme **Internet-facing**.
2. Select the custom VPC and both public web subnets.
3. Select `External-Load-Balancer-SG`.
4. Configure an HTTP listener on port `80` forwarding to `Web-Server-TG`.
5. Create the load balancer.

### 13. Create Auto Scaling Groups

**App Tier Auto Scaling Group**
1. Open **EC2 → Auto Scaling Groups → Create Auto Scaling group**.
2. Name it `App-Tier-ASG` and select `App-Server-Launch-template`.
3. Select the custom VPC and both private app subnets.
4. Attach the existing `App-Server-TG` target group.
5. Set desired capacity to `2`, minimum capacity to `2`, and maximum capacity to `2`.
6. Create the Auto Scaling group.

**Web Tier Auto Scaling Group**
1. Create another Auto Scaling group using the web-tier launch template.
2. Select both public web subnets.
3. Attach `Web-Server-TG`.
4. Set desired capacity to `2` (the project guide uses a desired capacity of 2).
5. Create the Auto Scaling group.

**Checkpoint:** Two app instances and two web instances should be launched and become healthy in their respective target groups. After confirming that the Auto Scaling instances are healthy, the two test instances can be terminated.

### 14. Access and Verify the Application

1. Open **EC2 → Load Balancers**.
2. Select the external (internet-facing) load balancer.
3. Copy its DNS name.
4. Open the DNS name in a browser. The project guide uses HTTP; if the page does not load, try an Incognito window and confirm the address starts with `http`, not `https`.
5. Open the **DB DEMO** section and verify that the sample transaction is displayed.
6. The demo allows you to add and delete items, which helps verify the database connection.

**Checkpoint:** The application home page loads through the external load balancer and the DB DEMO page displays data from Aurora.

### 15. Clean Up the Infrastructure

Some AWS resources used in this project incur ongoing charges, including NAT Gateways, load balancers, and the database. Delete the resources when you have finished testing.

Delete resources in the following order to avoid dependency errors:

1. Auto Scaling groups.
2. Load balancers (delete listeners first, then the load balancers).
3. Target groups.
4. Launch templates.
5. AMIs.
6. EBS snapshots.
7. EC2 instances.
8. Aurora database (disable delete protection first; when deleting, do not create a final snapshot if you do not need one).
9. NAT gateways.
10. Elastic IPs (release them).
11. Route tables (remove subnet associations first).
12. Internet gateways (detach, then delete).
13. VPC.
14. S3 bucket (empty it first).
15. IAM role.

---

## Quick Troubleshooting

| Problem | Likely Fix |
|---|---|
| Session Manager will not connect | Attach the IAM role to the instance and check that the private subnet has a NAT route. |
| `aws s3 cp` fails | Check the bucket name and confirm that the instance role has S3 read access. |
| App cannot reach the database | Check `DB_HOST`, username, and password in `DbConfig.js`. Confirm `DB-Tier-SG` allows traffic from `App-Tier-SG` on port `3306`. |
| Load balancer shows unhealthy targets | Check the target-group port, `/health` health-check path, and security-group rules. |
| Website shows `502` or `504` | Check that the internal load balancer DNS name in `nginx.conf` is correct. |
| Site does not load in the browser | Try an Incognito window and confirm that the external load balancer allows HTTP traffic on port `80`. |

## Notes

- **Region:** The project guide uses the Mumbai region (`ap-south-1`). Keep all resources in the same region and use the specified Availability Zones consistently.
- **Database credentials:** Use your own strong password. Do not commit passwords or other secrets to source control.
- **Application URL:** The application is accessed through the external load balancer's DNS name. The guide notes that a custom domain can be configured using Amazon Route 53.
- **Cost awareness:** NAT Gateways, load balancers, Aurora, and other AWS resources may incur charges while running. Follow Step 15 and verify that billable resources have been deleted when the project is complete.
- **Resource names:** Names shown in this README follow the project guide. If you choose different names, keep references consistent across the VPC, security groups, target groups, load balancers, and Auto Scaling groups.

---

**Created by Sanyukta Nirmal**  
[LinkedIn](https://linkedin.com/in/sanyukta-nirmal-aws-sd) | [GitHub Source Code](https://github.com/nsanyu28/3Tier-Application)
