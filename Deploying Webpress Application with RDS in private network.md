# Deploying a WordPress Application with RDS in a Private Network on AWS

This guide provides a comprehensive overview of how to set up a WordPress application on AWS, using Amazon RDS for the database in a secure, private network architecture.

## Architecture Overview

- **VPC**: A Virtual Private Cloud for networking.
- **Subnets**: Public and private subnets for resource segmentation.
- **Internet Gateway (IGW)**: Allows internet access for the public subnet.
- **NAT Gateway**: Enables the private subnet to access the internet securely.
- **EC2 Instance**: Hosts the WordPress application.
- **RDS**: Manages the MySQL database securely.
- **Application Load Balancer (ALB)**: Distributes incoming traffic to the EC2 instances.

---

## Step-by-Step Deployment Guide

### Step 1: Create a VPC

1. **Access the VPC Dashboard**:
   - Log in to the AWS Management Console.
   - Navigate to the **VPC** service.

2. **Create a VPC**:
   - Click on "Your VPCs" and then "Create VPC."
   - **Name**: `WordPress-VPC`
   - **IPv4 CIDR block**: `171.2.0.0/24`
   - Click "Create."

### Step 2: Create Subnets

1. **Create 2 Public Subnets**:
   - Go to "Subnets" and click "Create subnet."
   - **Name**: `Public-WordPress-Subnet-1` and `Public-WordPress-Subnet-2`
   - **VPC**: Select `WordPress-VPC`
   - **IPv4 CIDR block**: `171.2.0.64/26` and `171.2.0.0/26`
   - Click "Create."

2. **Create 4 Private Subnets**:
   - Repeat the above steps.
   - **Name**: `App-Private-WordPress-Subnet-1`, `App-Private-WordPress-Subnet-2`, `DB-Private-WordPress-Subnet-1`, and `DB-Private-WordPress-Subnet-2`
   - **IPv4 CIDR block**: `171.2.0.128/26`, `171.2.0.192/26` (App private subnets) and `171.2.0.208/28`, `171.2.0.224/28` (DB private subnets)
   - Click "Create."

### Step 3: Create Route Tables

1. **Public Route Table**:
   - Click on "Route Tables" and select "Create route table."
   - **Name**: `WordPress-Public-RT`
   - **VPC**: Select `WordPress-VPC`
   - Click "Create."

2. **Private App Route Table**:
   - Create another route table.
   - **Name**: `WordPress-App-Private-RT`
   - Click "Create."

3. **Private DB Route Table**:
   - Create another route table.
   - **Name**: `WordPress-DB-Private-RT`
   - Click "Create."

### Step 4: Route Table Subnet Associations

1. **Associate Public Subnets**:
   - Select `WordPress-Public-RT`, click on the **Subnet Associations** tab.
   - Click "Edit subnet associations," select `Public-WordPress-Subnet-1` and `Public-WordPress-Subnet-2`, and save.

2. **Associate App Private Subnets**:
   - Select `WordPress-App-Private-RT`, click on the **Subnet Associations** tab.
   - Select `App-Private-WordPress-Subnet-1` and `App-Private-WordPress-Subnet-2` and save.

3. **Associate DB Private Subnets**:
   - Select `WordPress-DB-Private-RT`, click on the **Subnet Associations** tab.
   - Select `DB-Private-WordPress-Subnet-1` and `DB-Private-WordPress-Subnet-2` and save.

### Step 5: Create an Internet Gateway

1. **Create Internet Gateway**:
   - Navigate to "Internet Gateways" and click "Create internet gateway."
   - **Name**: `WordPress-IGW`
   - Click "Create."

2. **Attach to VPC**:
   - Select the internet gateway, click "Actions," then "Attach to VPC."
   - Choose `WordPress-VPC` and click "Attach."

### Step 6: Create a NAT Gateway

1. **Allocate Elastic IP**:
   - Go to the EC2 Dashboard, click "Elastic IPs," and allocate a new address.

2. **Create NAT Gateway**:
   - Go to "NAT Gateways" and click "Create NAT gateway."
   - **Name**: `WordPress-NAT-GW`
   - **Subnet**: Select `Public-WordPress-Subnet-1`
   - **Elastic IP**: Choose the allocated Elastic IP.
   - Click "Create NAT gateway."

### Step 7: Add Routes for IGW and NAT

1. **Public Route Table**:
   - Select `WordPress-Public-RT`, go to the **Routes** tab.
   - Click "Edit routes," then "Add route":
     - **Destination**: `0.0.0.0/0`
     - **Target**: Select `WordPress-IGW`
   - Click "Save routes."

2. **Private App Route Table**:
   - Select `WordPress-App-Private-RT`, go to the **Routes** tab.
   - Click "Edit routes," then "Add route":
     - **Destination**: `0.0.0.0/0`
     - **Target**: Select `WordPress-NAT-GW`
   - Click "Save routes."

3. **Private DB Route Table**:
   - Select `WordPress-DB-Private-RT`, go to the **Routes** tab.
   - Click "Edit routes," then "Add route":
     - **Destination**: `0.0.0.0/0`
     - **Target**: Select `WordPress-NAT-GW`
   - Click "Save routes."

### Step 8: Create a Bastion Host

1. **Launch EC2 Instance**:
   - Go to the EC2 Dashboard and click "Launch Instance."
   - **AMI**: Choose an Amazon Linux AMI.
   - **Instance Type**: Choose `t2.micro` (or as needed).
   - **Network Settings**: Choose `Public-WordPress-Subnet-1`.
   - **Storage**: Use default settings.
   - **Key Pair**: Create or select an existing key pair.
   - Click "Launch."

### Step 9: Set Up RDS

1. **Navigate to RDS Dashboard**:
   - Click on "Databases" and then "Create database."

2. **Select Database Engine**:
   - Choose MySQL.

3. **Choose a Use Case**:
   - Select "Standard Create."

4. **Configure Database Settings**:
   - **DB Instance Identifier**: `wordpress-db`
   - **Master Username**: `admin`
   - **Password**: Create a secure password.

5. **DB Instance Class**:
   - Choose `db.t4g.micro` (or as needed).

6. **Storage**:
   - Use default settings; adjust as needed.

7. **Connectivity**:
   - Choose the VPC (`WordPress-VPC`).
   - **Subnet group**: Use private subnets.
   - **Public accessibility**: No (for security).
   - **VPC Security Groups**: Create a new group allowing traffic from the private subnet.

8. **Additional Configuration**:
   - Configure backup and monitoring options as needed.

9. **Create Database**:
   - Click "Create database."

### Step 10: Launch EC2 Instance for WordPress

1. **Launch EC2 Instance**:
   - Go to the EC2 Dashboard and click "Launch Instance."
   - **AMI**: Choose an Amazon Linux AMI.
   - **Instance Type**: Choose `t2.micro` (or as needed).
   - **Network Settings**: Choose `App-Private-WordPress-Subnet-1`.
   - **Storage**: Use default settings.
   - **Key Pair**: Create or select an existing key pair.
   - Click "Launch."

### Step 11: Configure Security Groups

1. **Create Security Group for EC2**:
   - Allow inbound traffic on port 80 (HTTP) and port 443 (HTTPS) from the ALB.
   - Allow inbound traffic on port 22 (SSH) from the bastion host's IP.

2. **Create Security Group for RDS**:
   - Allow inbound traffic on port 3306 (MySQL) from the private EC2 instance's security group.

### Step 12: Install and Configure WordPress

1. **SSH into Your EC2 Instance**:
   - Use your bastion host to connect to the private EC2 instance:
     ```bash
     ssh -i /path/to/your-key.pem ec2-user@<private-ec2-public-ip>
     ```

2. **Install Apache, PHP, and MySQL Extensions**:
   ```bash
   sudo yum update -y
   sudo yum install -y httpd php php-mysqlnd
   ```

3. **Start Apache**:


   ```bash
   sudo systemctl start httpd
   sudo systemctl enable httpd
   ```

4. **Download WordPress**:
   ```bash
   wget https://wordpress.org/latest.tar.gz
   tar -xzf latest.tar.gz
   sudo mv wordpress/* /var/www/html/
   ```

5. **Configure WordPress**:
   - Copy the sample configuration file:
     ```bash
     cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
     ```
   - Edit the `wp-config.php` file:
     ```bash
     nano /var/www/html/wp-config.php
     ```
   - Update the following lines:
     ```php
     define('DB_NAME', 'WordPress-DB');
     define('DB_USER', 'admin');
     define('DB_PASSWORD', 'yourpassword');
     define('DB_HOST', 'your-rds-endpoint');
     ```

6. **Set Permissions**:
   ```bash
   sudo chown -R apache:apache /var/www/html/*
   ```

7. **Complete WordPress Installation**:
   - Navigate to `http://<your-ALB-DNS-name>` to complete the WordPress installation through the web interface.

### Step 13: Create and Configure an Application Load Balancer

1. **Navigate to Load Balancers**:
   - Click on "Load Balancers" and then "Create Load Balancer."

2. **Choose Application Load Balancer**:
   - **Name**: `WordPress-ALB`
   - **Scheme**: Choose "internet-facing."
   - **Listeners**: Add a listener on port 80.
   - **Availability Zones**: Select the public subnets.

3. **Configure Security Settings**:
   - Skip for HTTP; configure HTTPS later if needed.

4. **Configure Security Groups**:
   - Create a new security group or select an existing one allowing HTTP traffic.

5. **Configure Target Group**:
   - **Name**: `WordPress-Target-Group`
   - **Target Type**: Instance.
   - **Health checks**: Use HTTP and set the path to `/`.

6. **Register Targets**:
   - Add the EC2 instance (WordPress) to the target group.

7. **Review and Create**:
   - Click "Create" to finalize the load balancer.

### Step 14: Additional Configurations (Optional)

1. **Set Up HTTPS**:
   - Consider using AWS Certificate Manager (ACM) to provision an SSL certificate.
   - Attach the certificate to your ALB for HTTPS traffic.

2. **Enable Backups**:
   - Set up regular backups for your RDS instance.

3. **Monitoring and Logging**:
   - Use Amazon CloudWatch to monitor your EC2 and RDS instances.
   - Configure logging for Apache to track access and errors.

## Conclusion

You've successfully deployed a WordPress application on AWS using RDS in a private network. This architecture provides enhanced security and scalability. With the ability to manage your infrastructure effectively, you can now focus on developing and growing your WordPress site.

