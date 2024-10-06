# Deploying a WordPress Application with RDS in a Private Network on AWS

This guide outlines the process for deploying a WordPress application on AWS, leveraging Amazon RDS for database management within a secure private network architecture.

## Architecture Overview

- **VPC**: A Virtual Private Cloud to host your resources.
- **Subnets**: Segregation into public and private subnets.
- **Internet Gateway (IGW)**: Provides internet access for public subnets.
- **NAT Gateway**: Allows private subnets to access the internet securely.
- **EC2 Instance**: Hosts the WordPress application.
- **RDS**: Manages the MySQL database.
- **Application Load Balancer (ALB)**: Distributes incoming traffic to EC2 instances.

---

## Step-by-Step Deployment Guide

### Step 1: Create a VPC

![vpc](https://github.com/user-attachments/assets/20e8f374-07b9-4046-b512-740d0522e14e)


1. **Access the VPC Dashboard**:
   - Log into the AWS Management Console.
   - Go to the **VPC** service.

2. **Create a VPC**:
   - Click "Your VPCs" and select "Create VPC."
   - **Name**: `WordPress-VPC`
   - **IPv4 CIDR block**: `171.2.0.0/24`
   - Click "Create."

### Step 2: Create Subnets

![subnets](https://github.com/user-attachments/assets/a53b4313-7555-4922-9d78-45484ad05fe0)


1. **Create Public Subnets**:
   - Click on "Subnets" and then "Create subnet."
   - **Name**: `Public-Subnet-1` and `Public-Subnet-2`
   - **VPC**: Select `WordPress-VPC`
   - **IPv4 CIDR block**: `171.2.0.0/26` and `171.2.0.64/26`
   - Click "Create."

2. **Create Private Subnets**:
   - Repeat the above for private subnets.
   - **Name**: `Private-App-Subnet-1`, `Private-App-Subnet-2`, `Private-DB-Subnet-1`, and `Private-DB-Subnet-2`
   - **IPv4 CIDR block**: `171.2.0.128/26`, `171.2.0.192/26` (app subnets), and `171.2.0.208/28`, `171.2.0.224/28` (DB subnets)
   - Click "Create."

### Step 3: Create Route Tables

![Route tables](https://github.com/user-attachments/assets/a74e8600-0f32-41d3-af1c-b4e1aeb4eeb8)

1. **Public Route Table**:
   - Click "Route Tables" and "Create route table."
   - **Name**: `Public-RT`
   - **VPC**: Select `WordPress-VPC`
   - Click "Create."

2. **Private Route Tables**:
   - Create two additional route tables:
   - **Names**: `Private-App-RT` and `Private-DB-RT`
   - Click "Create" for each.

### Step 4: Associate Route Tables

1. **Public Subnets**:
   - Select `Public-RT`, go to the **Subnet Associations** tab.
   - Associate with `Public-Subnet-1` and `Public-Subnet-2`.

2. **Private Subnets**:
   - Associate `Private-App-RT` with `Private-App-Subnet-1` and `Private-App-Subnet-2`.
   - Associate `Private-DB-RT` with `Private-DB-Subnet-1` and `Private-DB-Subnet-2`.

### Step 5: Create an Internet Gateway

![Internet gateway](https://github.com/user-attachments/assets/525fa6ef-7a13-47c6-98f1-e15aadbb7b09)


1. **Create and Attach**:
   - Navigate to "Internet Gateways" and click "Create internet gateway."
   - **Name**: `WordPress-IGW`
   - Attach it to `WordPress-VPC`.

### Step 6: Create a NAT Gateway

![Nat gateways](https://github.com/user-attachments/assets/01bbde12-e722-46b1-9aae-4a4a00add0aa)


1. **Allocate Elastic IP**:
   - Go to the EC2 Dashboard, select "Elastic IPs," and allocate a new address.

2. **Create NAT Gateway**:
   - Click "NAT Gateways," create a new one in `Public-Subnet-1` using the allocated Elastic IP.

### Step 7: Add Routes

1. **Public Route Table**:
   - Edit `Public-RT` to route `0.0.0.0/0` to `WordPress-IGW`.

2. **Private Route Tables**:
   - For `Private-App-RT` and `Private-DB-RT`, route `0.0.0.0/0` to `WordPress-NAT-GW`.

### Step 8: Create a Bastion Host

![Bastion host](https://github.com/user-attachments/assets/2fcd6765-d205-4c53-b4ad-62cb05d7b12f)


1. **Launch EC2 Instance**:
   - Go to EC2 and click "Launch Instance."
   - Select an Amazon Linux AMI and instance type `t2.micro`.
   - Set the network to `Public-Subnet-1`, configure storage, and select a key pair.
   - Click "Launch."

![App ec2 instance](https://github.com/user-attachments/assets/120cf317-6dde-44c5-808e-f61fcfcf732c)


2. **Create an APP EC2 Instance**:
   - Go to EC2 and click "Launch Instance."
   - Select an Amazon Linux AMI and instance type `t2.micro`.
   - Set the network to `Private-App-Subnet-1`, configure storage, and select a key pair.
   - Click "Launch."

### Step 9: Create a Subnet Group for RDS

![subnet group](https://github.com/user-attachments/assets/3ac49e7c-53b7-4629-abab-7ab4d9702be0)


1. **Access RDS Dashboard**:
   - Click on **Subnet groups** under **Network & Security**.

2. **Create Subnet Group**:
   - Click "Create DB Subnet Group."
   - **Name**: `webpress-db -subnetgroups`
   - **Description**: Add a relevant description.
   - **VPC**: Choose `WordPress-VPC`.
   - **Add Subnets**: Select the two DB private subnets.

### Step 10: Set Up RDS Instance

![db instance](https://github.com/user-attachments/assets/b144202a-0d46-4dc3-b6d8-4f461b77ff4f)


1. **Create Database**:
   - In RDS, click "Databases" and "Create database."

2. **Select Engine**:
   - Choose MySQL, then "Standard Create."

3. **Configure DB**:
   - **DB Identifier**: `wordpress-db`
   - **Master Username**: `admin`
   - **Password**: Create a secure password.

4. **Instance Class and Connectivity**:
   - Set instance class (e.g., `db.t4g.micro`).
   - Choose `WordPress-VPC`, select the subnet group, and ensure "Public Accessibility" is **No**.
   - create and go to DB security groups change the inbound rule to App-server SG

5. **Review and Create**:
   - Review your settings and click "Create database."

### Step 11: Launch EC2 Instance for WordPress

1. **Launch Instance**:
   - Go back to EC2, click "Launch Instance," and select an Amazon Linux AMI.
   - Set the network to `Private-App-Subnet-1`.
   - Complete the instance creation process.

### Step 12: Configure Security Groups

1. **For EC2**:
   - Create a security group allowing HTTP (80) and HTTPS (443) from the ALB.
   - Allow SSH (22) only from the bastion host's IP.

2. **For RDS**:
   - Create a security group allowing MySQL (3306) from the EC2 instance’s security group.

### Step 13: Install and Configure WordPress

1. **SSH into EC2**:
   - Copy the .pem keyfile from local host to Bastion host
    ```bash
    scp -i ./keypair-new.pem  ./keypair-new.pem ubuntu@54.12.84.90:/home/ec2user
    ```
     
   - Connect via SSH through your bastion host:
     ```bash
     ssh -i /path/to/your-key.pem ec2-user@<bastion-public-ip>
     ssh -i /path/to/your-key.pem ec2-user@<private-ec2-ip>
     ```

3. **Install Software**:
   ```bash
   sudo yum update -y
   sudo yum install -y httpd php php-mysqlnd
   ```

4. **Start Apache**:
   ```bash
   sudo systemctl start httpd
   sudo systemctl enable httpd
   ```

5. **Download and Configure WordPress**:
   ```bash
   wget https://wordpress.org/latest.tar.gz
   tar -xzf latest.tar.gz
   sudo mv wordpress/* /var/www/html/
   cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
   nano /var/www/html/wp-config.php
   ```
   - Update the database details:
     ```php
     define('DB_NAME', 'wordpress-db');
     define('DB_USER', 'admin');
     define('DB_PASSWORD', 'yourpassword');
     define('DB_HOST', 'your-rds-endpoint');
     ```

6. **Set Permissions**:
   ```bash
   sudo chown -R apache:apache /var/www/html/*
   ```

7. **Official AWS Installation Documents**:
   - link `http://<your-ALB-DNS-name>](https://aws.amazon.com/tutorials/deploy-wordpress-with-amazon-rds/module-four/` 

### Step 14: Create an Application Load Balancer

![LB-Targetgroup](https://github.com/user-attachments/assets/faa54601-bf7b-41da-985b-7611ddf33d69)


1. **Create Load Balancer**:
   - Go to "Load Balancers" and select "Create Load Balancer."
   - Choose **Application Load Balancer**, set a name, and configure it as "internet-facing" with port 80.

2. **Configure Security Settings**:
   - Set up security groups to allow HTTP traffic.
  
     ![LB-Targetgroup](https://github.com/user-attachments/assets/b12ba18b-03be-48ca-8404-997810359ca8)


3. **Configure Target Group**:
   - Name it `Webpress-TG` and register your EC2 instance.

4. **Review and Create**:
   - Finalize your load balancer setup.

5. **Link to the WordPress Website**
   - link `Wordpress-ALB-2100940295.us-east-1.elb.amazonaws.com`


### Step 15: Security Groups

1. **App-Webpress-SG Inbound rules**:
   - HTTP - Webpress-ALB-SG
   - SSH - Bastion host

2. **Wordpress-DB-SG Inbound rules**:
   - MYSQL/Aurora - App-Webpress-SG

3. **Webpress-ALB-SG Inbound rules**:
   - HTTP - 0.0.0.0/0

4. **Bastion-host-SG**:
   - SSH - 0.0.0.0/0
     
## Conclusion

![wordpress-site](https://github.com/user-attachments/assets/e2ffd30c-8f1e-4abc-ba7b-75bfda13f15b)

You have successfully deployed a WordPress application on AWS, utilizing RDS in a secure private network architecture. This setup enhances security and scalability, allowing you to focus on your

![wordpress-db initiation](https://github.com/user-attachments/assets/f583006b-a8bc-4d5b-ac4e-dd6ce8718b79)

 WordPress site’s growth and development.
