# AWS Three Tier Web Architecture

## Description:
A hands-on walkthrough for building a three-tier web architecture in AWS, manually configuring network, security, application, and database components to create an available and scalable system.

## Audience:
Technical professionals with foundational knowledge of AWS services (VPC, EC2, RDS, S3, ELB) and the AWS Console.

## Pre-requisites:
1. AWS account (create one at [AWS Console](https://aws.amazon.com/console/) if needed)
2. IDE or text editor

## Architecture Overview

![Architecture Diagram](/demos/3TierArch.png)

This architecture features a public-facing Application Load Balancer routing client traffic to Nginx webservers in the web tier. These servers host a React.js website and redirect API calls to an internal load balancer, which forwards traffic to Node.js application servers. The app tier interacts with an Aurora MySQL multi-AZ database. Load balancing, health checks, and autoscaling groups maintain availability at each layer.

## Setup - Part 0

### Download Code from Github
```
git clone https://github.com/AmanKumar-Gupta/Practical-AWS-3-tier-app-deployment.git
```

### S3 Bucket Creation
1. Navigate to S3 in AWS console and create a new bucket
2. Select your region and provide a unique bucket name

### IAM EC2 Instance Role Creation
1. Navigate to IAM dashboard and create an EC2 role
2. Select EC2 as the trusted entity
3. Add these AWS managed policies:
   - AmazonSSMManagedInstanceCore
   - AmazonS3ReadOnlyAccess
4. Name your role and create it

## Networking and Security - Part 1

Building VPC networking components and security groups to protect EC2 instances, Aurora databases, and load balancers.

### VPC and Subnets

#### VPC Creation
1. Navigate to VPC dashboard > Your VPCs
2. Select VPC only and create a VPC with a name and CIDR range

   **NOTE**: Ensure consistent region deployment and choose a CIDR range allowing for at least 6 subnets

   ![VPC](/demos/FillVPCSettings.png)

#### Subnet Creation
1. Navigate to Subnets and click Create subnet
2. Create six subnets across two availability zones (three per zone), each corresponding to one architecture layer

   **NOTE**: Use a naming convention like Public-Web-Subnet-AZ-1, Private-App-Subnet-AZ-1, Private-DB-Subnet-AZ-1

   **NOTE**: Subnet CIDR ranges should be subsets of your VPC CIDR range

   ![subnet](/demos/FillSubnetDetails.png)

### Internet Connectivity

#### Internet Gateway
1. Navigate to Internet Gateway and create one with a name
2. Attach it to your VPC

   ![internetGateway](/demos/AttachIGW1.png)

#### NAT Gateway
1. Navigate to NAT Gateways and create one for each public subnet
2. Provide a name, select a public subnet, and allocate an Elastic IP

   ![NAT](/demos/FillNATGWDetails.png)

3. Repeat for the second public subnet

### Routing Configuration
1. Create a route table for web layer public subnets

   ![](/demos/CreatePublicRT.png)

2. Edit routes in the Routes tab

   ![](/demos/EditRoutes.png)

3. Add a route directing external traffic to the internet gateway

   ![](/demos/AddIGWRoute.png)

4. Edit subnet associations and select web layer public subnets

   ![](/demos/EditSubnetAssociations.png)
   ![](/demos/AssociatePublicSubnets.png)

5. Create two more route tables for app layer private subnets, each routing external traffic to its respective NAT gateway

   ![](/demos/PrivateSubnetRT1.png)
   ![](/demos/PrivateSubnetAssociation.png)

### Security Groups
1. Create a security group for the public load balancer with an inbound rule allowing HTTP from your IP

   ![](/demos/ExternalLBSG.png)

2. Create a security group for web tier instances allowing HTTP from the public load balancer security group and your IP

   ![](/demos/WebTierSG.png)

3. Create a security group for the internal load balancer allowing HTTP from the web instance security group

   ![](/demos/InternalLBSG.png)

4. Create a security group for private instances allowing TCP port 4000 from the internal load balancer security group and your IP

   ![](/demos/PrivateInstanceSG.png)

5. Create a security group for the database allowing MYSQL/Aurora (port 3306) from the private instance security group

   ![](/demos/DBSG.png)

## Database Deployment - Part 2

### Subnet Groups
1. Navigate to RDS dashboard > Subnet groups > Create DB subnet group
2. Name your group, add a description, and select your VPC

   ![](/demos/FillSubnetGroupDetails1.png)

3. Add the database subnets from each availability zone

   ![](/demos/FillSubnetGroupDetails2.png)

### Multi-AZ Database Deployment
1. Navigate to Databases and create a MySQL-Compatible Amazon Aurora database with Standard create

   ![](/demos/DBConfig1.png)

2. Choose Dev/Test template, set username and password (note these for later)

   ![](/demos/DBConfig2.png)

3. Create an Aurora Replica in a different availability zone, select your VPC, subnet group, and disable public access

   ![](/demos/DBConfig3.png)

4. Select the database security group and password authentication

   ![](/demos/DBConfig4.png)

5. Note the writer endpoint for later use

   ![](/demos/DBEndpoint.png)

## App Tier Instance Deployment - Part 3

In this section, you'll create and configure an EC2 instance for the Node.js application tier (running on port 4000) and set up the database schema.

- Tasks:
  - Create App Tier Instance
  - Configure Software Stack
  - Configure Database Schema
  - Test DB connectivity

### App Instance Deployment

1. Navigate to EC2 dashboard > Instances > Launch Instances
2. Select the first Amazon Linux 2 AMI
3. Choose the free tier eligible **T.2 micro** instance type
4. Configure instance details: select the correct Network, subnet (use a private app layer subnet), and IAM role

   ![](/demos/ConfigureInstanceDetails.png)

5. Keep storage defaults. Add a Name tag with value "AppLayer"

   ![](/demos/AddTag.png)

6. Select the previously created security group for private app layer instances, then Review and Launch (ignore port 22 warning)
7. Review configuration, click Launch, proceed without a key pair, and confirm launch

### Connect to Instance

1. Once instance state is running, select it and click Connect > Session Manager > Connect

   **NOTE**: If connection fails, verify NAT gateway routing and IAM permissions

2. Switch to ec2-user:
   ```
   sudo -su ec2-user
   ```

3. Test internet connectivity:
   ```
   ping 8.8.8.8
   ```
   
   **NOTE**: If unsuccessful, check route tables and subnet associations

### Configure Database

1. Install MySQL CLI:
   ```
   sudo yum install mysql -y
   ```

2. Connect to your Aurora RDS:
   ```
   mysql -h CHANGE-TO-YOUR-RDS-ENDPOINT -u CHANGE-TO-USER-NAME -p
   ```
   
   **NOTE**: If connection fails, verify credentials and security groups

3. Create database:
   ```
   CREATE DATABASE webappdb;
   ```
   Verify:
   ```
   SHOW DATABASES;
   ```

4. Create table:
   ```
   USE webappdb;
   CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL AUTO_INCREMENT, amount DECIMAL(10,2), description VARCHAR(100), PRIMARY KEY(id));
   ```
   Verify:
   ```
   SHOW TABLES;
   ```

5. Insert test data:
   ```
   INSERT INTO transactions (amount,description) VALUES ('400','groceries');
   ```
   Verify:
   ```
   SELECT * FROM transactions;
   ```

6. Type `exit` to close MySQL client

### Configure App Instance

1. Update database credentials in `application-code/app-tier/DbConfig.js` with your database writer endpoint, username, password, and "webappdb" as database name

   **NOTE**: This is for simplicity; using AWS Secrets Manager would be best practice

2. Upload the app-tier folder to your S3 bucket

3. Install Node Version Manager:
   ```
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
   source ~/.bashrc
   ```

4. Install Node.js:
   ```
   nvm install 16
   nvm use 16
   ```

5. Install PM2:
   ```
   npm install -g pm2
   ```

6. Download code from S3:
   ```
   cd ~/
   aws s3 cp s3://BUCKET_NAME/app-tier/ app-tier --recursive
   ```

7. Set up the application:
   ```
   cd ~/app-tier
   npm install
   pm2 start index.js
   ```
   
   Verify status:
   ```
   pm2 list
   ```
   
   If status shows "errored", check logs:
   ```
   pm2 logs
   ```

8. Configure PM2 to auto-start on reboot:
   ```
   pm2 startup
   ```
   Copy and run the command provided in the output (not the example)
   ```
   pm2 save
   ```

### Test App Tier

Test health check endpoint:
```
curl http://localhost:4000/health
```
Expected response: `"This is the health check"`

Test database connection:
```
curl http://localhost:4000/transaction
```
Expected response: JSON data with your inserted records

If both tests succeed, your networking, security, database, and app configurations are correct.

## Internal Load Balancing and Auto Scaling

In this section, you'll create an AMI of the configured app tier instance and set up autoscaling with a load balancer for high availability.

- **Objectives:**
  - Create an AMI of the App Tier
  - Create a Launch Template
  - Configure Autoscaling
  - Deploy Internal Load Balancer

### App Tier AMI

1. Navigate to EC2 dashboard > Instances > select app tier instance > Actions > Image and templates > Create Image

   ![](/demos/CreateAMI1.png)

2. Name and describe your image, then click Create image

   ![](/demos/CreateAMI2.png)

### Target Group

1. Navigate to EC2 dashboard > Target Groups > Create Target Group

2. Select Instances as target type, name it, set protocol to HTTP and port to 4000 (Node.js app port), select your VPC, and set health check path to /health

3. Skip registering targets and create the target group

### Internal Load Balancer

1. Navigate to EC2 dashboard > Load Balancers > Create Load Balancer

2. Select Application Load Balancer

3. Name it, select internal type (not public-facing)

   ![](/demos/LBConfig1.png)

   Configure VPC and private subnets

   ![](/demos/LBConfig2.png)

   Select the internal ALB security group, configure HTTP listener on port 80 forwarding to your target group

   ![](/demos/LBConfig3.png)

### Launch Template

1. Navigate to EC2 dashboard > Launch Template > Create Launch Template

2. Name it and select your app tier AMI

   ![](/demos/LaunchTemplateConfig1.png)

   Choose t2.micro instance type, skip key pair and network settings

   ![](/demos/LaunchTemplateConfig2.png)

   Select app tier security group and the same IAM instance profile

   ![](/demos/LaunchTemplateConfig3.png)
   ![](/demos/LaunchTemplateConfig4.png)

### Auto Scaling

1. Navigate to EC2 dashboard > Auto Scaling Groups > Create Auto Scaling group

2. Name it and select your launch template

   ![](/demos/ConfigureASG1.png)

3. Select VPC and private app tier subnets

   ![](/demos/ConfigureASG2.png)

4. Attach to your internal load balancer's target group

   ![](/demos/ConfigureASG3.png)

5. Set desired, minimum, and maximum capacity to 2, then create the Auto Scaling Group

   **NOTE**: Your original app tier instance remains outside the ASG; keep it for troubleshooting

## Web Tier Instance Deployment - Part 5

### Update Config File

1. Edit application-code/nginx.conf line 58: replace [INTERNAL-LOADBALANCER-DNS] with your internal load balancer's DNS

   ![](/demos/ReplaceCode.png)

2. Upload this file and the application-code/web-tier folder to your S3 bucket

### Web Instance Deployment

1. Create an instance following the same process as the app tier, but use a public subnet and auto-assign a public IP

   ![](/demos/WebInstanceCreate1.png)
   ![](/demos/WebInstanceCreate2.png)

### Connect to Instance

1. Connect via Session Manager, switch to ec2-user, and test internet connectivity:
   ```bash
   sudo -su ec2-user
   ping 8.8.8.8
   ```

### Configure Web Instance

1. Install NVM and Node.js:
   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
   source ~/.bashrc
   nvm install 16
   nvm use 16
   ```

2. Download web tier code and build the React app:
   ```bash
   cd ~/
   aws s3 cp s3://BUCKET_NAME/web-tier/ web-tier --recursive
   cd ~/web-tier
   npm install
   npm run build
   ```

3. Install NGINX:
   ```bash
   sudo amazon-linux-extras install nginx1 -y
   ```

4. Configure NGINX:
   ```bash
   cd /etc/nginx
   ls
   sudo rm nginx.conf
   sudo aws s3 cp s3://BUCKET_NAME/nginx.conf .
   sudo service nginx restart
   chmod -R 755 /home/ec2-user
   sudo chkconfig nginx on
   ```

5. Test your website by accessing the instance's public IP in a browser

   ![](/demos/WebPage1.png)
   ![](/demos/WebPage2.png)

## External Load Balancer and Auto Scaling - Part 6

### Web Tier AMI

1. Select your web tier instance > Actions > Image and templates > Create Image
2. Name and describe your image

### Target Group

1. Create a new target group for the web tier
2. Select Instances type, HTTP protocol, port 80 (NGINX port), your VPC, and /health as health check path
3. Skip registering targets

### Internet Facing Load Balancer

1. Create an Application Load Balancer
2. Name it, select internet facing type
3. Configure with VPC and public subnets
4. Select external ALB security group, HTTP listener on port 80 forwarding to web tier target group

### Launch Template

1. Navigate to **Launch Templates** in the EC2 dashboard and click **Create Launch Template**.
2. Name it and select the previously created app tier AMI.

   ![](</demos/LaunchTemplateConfig1%20(1).png>)

3. Choose **t2.micro** as the instance type. Skip **Key Pair** and **Network Settings** (configured in Auto Scaling).
4. Assign the web tier security group and the same IAM instance profile as other EC2 instances.

### Auto Scaling

1. Navigate to **Auto Scaling Groups** in the EC2 dashboard and click **Create Auto Scaling Group**.
2. Name the group, select the Launch Template, and proceed.
3. Set VPC and public subnets for the web tier.
4. Attach the existing web tier load balancer’s target group.
5. Configure **desired, minimum, and maximum capacity** to **2** and create the group.

   Verify that two web tier instances are created. To test, delete one instance and check if a new one replaces it. Visit the load balancer DNS in your browser to validate functionality.

   **NOTE**: The original web tier instance is outside the ASG, so three instances may appear. Keep the original for troubleshooting.

   ![](/demos/FinalLBDNS.png)

### **Congrats! You’ve Implemented a 3-Tier Web Architecture!**

## Clean Up

To avoid charges, delete resources in this order:

1. **Auto Scaling Groups** – Delete web and app tier groups in the EC2 dashboard.
2. **Application Load Balancers** – Remove listeners first, then delete load balancers.
3. **Target Groups** – Delete now that ALB dependencies are removed.
4. **Launch Templates** – Delete templates in the EC2 dashboard.
5. **AMIs** – Deregister AMIs under Images.
6. **EC2 Instances** – Terminate any remaining instances.
7. **Aurora Database** – Disable deletion protection, modify the cluster, and delete instances.
8. **NAT Gateways** – Delete both NAT gateways.
9. **Elastic IPs** – Release all allocated IPs.
10. **Route Tables** – Remove subnet associations, then delete route tables.
11. **Internet Gateway** – Detach from VPC, then delete it.
12. **VPC** – Delete the created VPC, which also removes subnets and security groups.

## License

Licensed under the MIT-0 License. See the LICENSE file.
