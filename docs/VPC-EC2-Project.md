# VPC/EC2 Project COMPLETE

# Project Overview

- Objective: Architect a highly available, secure, and scalable AWS cloud environment using industry best practices, focusing on automated scaling, network isolation, and security hardening. Set up a VPC with public and private subnets, deploy EC2 instances, and ensure internet access for public instances, while keeping private instances isolated.
- Outcome: Successfully created a VPC with public/private subnets, set up Auto Scaling Group for EC2 instances, and demonstrated secure cloud networking by connecting via SSH and implementing security best practices

# Architecture Design

![VPC_EC2-Complete.jpeg]([screenshots/VPC_EC2-Complete.jpeg])

The architecture above follows a multi-tier AWS networking model with:

- Public Subnets: Host the Bastion Host and Load Balancer for external access.
- Private Subnets: Restricted for backend EC2 instances and internal resources.
- Application Load Balancer: Routes incoming traffic dynamically to EC2 instances in the Auto Scaling Group.
- Auto Scaling Group: Automatically adjusts EC2 capacity based on demand and CPU utilization.
- VPC Endpoint (S3 Gateway): Ensures private, secure access to S3 without internet exposure.

# Resources and Components

- VPC: 10.0.0.0/16 with default tenancy
    - Subnets:
        - Public Subnet 1: 10.0.0.0/24
        - Public Subnet 2: 10.0.1.0/24
        - Private Subnet 1: 10.0.16.0/24
        - Private Subnet 2: 10.0.32.0/24
    - 2 Availability zones to ensure high availability:
        - us-east-1a
        - us-east-1b
    - Internet Gateway: to allow the public subnets internet access
    - NAT Gateway: to allow private subnet’s instances to connect to the internet
    - VPC Endpoint: To lockdown VPC resources to S3 for lower latency and secure connection. Also avoids the need for internet access and enhances security by keeping traffic within AWS’s private network
- Security groups:
    - BastionHost-SG: SSH on port 22 with source (my IP to ensure its isolated) and used to SSH into private instances
    - Private-SG: uses the BastionHost-SG SSH on port 22 to keep the instances inaccessible from the internet
    - ApplicationLoadBalancer-SG: HTTP on port 80 (0.0.0.0/0) to allow DNS traffic anywhere
    - AutoScalingGroup-SG: HTTP on port 80 with the source as the ALB-SG to enable scalability and lockdown HTTP access only from the ALB DNS itself
- EC2 Instances:
    - Bastion Host: used to SSH into private instances
    - Private Instances for backend control and for testing
    - Public ASG instances connected to the Application Load Balancer for traffic distribution
- Userdata script for the Auto Scaling Group launch Template
    - #!/bin/bash
    - #to update, download http, start and enable httpd, and to add a text file in /var/www/html/index.html 
    - dnf update -y
    - dnf install httpd -y
    - systemctl start httpd
    - systemctl enable httpd
    - echo “Hello world from $(hostname -f)” > /var/www/html/index.html
- Elastic Load Balancer:
    - Application load balancer to distribute HTTP/HTTPS traffic
- IAM Role:
    - Only EC2-S3-ReadOnlyAccess to ensure least privilege and to access the S3 buckets from the EC2 CLI. The role adheres to the principle of least privilege granting only the minimum access possible to only read from S3.

# Security and Best Practices

- 4 security groups to lockdown each critical areas
    - Application Load Balancer security group only allowing HTTP on port 80 from anywhere (0.0.0.0/0)
    - Public EC2 Instances with Auto Scaling Group to only allow HTTP access from the Application Load Balancer
    - Bastion Host with SSH on port 22 to only connect to the private instances
    - Private instances to only allow SSH from the bastion host and ultimately any other connections for internal backend control within the VPC
- IAM Role:
    - Private EC2 instance role with read only permission for internal S3 access using the VPC gateway endpoint for cost efficiency and low latency and isolated from the internet

# Steps Taken

- VPC Setup:
    - Created a VPC named Project-VPC with an IPv4 CIDR block of 10.0.0.0/16 and a default tenancy.
    - Added a public subnet (project-public-Subnet-1) from the Project VPC and included a IPv4 CIDR block of 10.0.0.0/24 (256 IPs) in the US East (N. Virginia) / us-east-1a availability zone for public-facing resources.
    - Added another public subnet with the same configuration but changed the name to Project-Public-Subnet-2 and the CIDR block to 10.0.1.0/24 and US East (N. Virginia) / us-east-1b availability zone to ensure redundancy.
    - Added a private subnet (Project-Private-Subnet-1) from the the Project VPC and included a IPv4 CIDR block of 10.0.0.0/24 (256 IPs) in the US East (N. Virginia) / us-east-1a availability zone for private resources.
    - Added another private subnet with the same configuration but changed the name to Project-Private-Subnet-2 and the CIDR block to 10.0.1.0/24 and US East (N. Virginia) / us-east-1b availability zone. This ensures fault tolerance.
    - Edited the public subnets and enabled auto-assign public IPv4 address. This will ensure that launched EC2 instances in the public subnets will automatically receive an IP address
    - Created a public route table (Project-Public-RT) from the VPC project and associated the route tables with the public subnets. This route table will have the internet gateway for internet connection.
    - Created a private route table (Project-Private-RT) from the VPC project and associated the route tables with the private route tables. This route table will have NAT gateway.
    - Created an internet gateway (Project-Internet-Gateway) and attached it to the Project-VPC. This will allow public access to the internet.
    - From the public route table (Project-Public-RT), I added a new destination (0.0.0.0/0) specifying anywhere in the internet and the target as my Project-Internet-Gateway that was created prior.
    - Created a NAT gateway (Project-NAT-Gateway) on Project-Public-Subnet-1 with the connectivity type of Public and allocated an Elastic IP. This gateway gives my private instances connectivity over the internet. This also have the private instances on lockdown, inaccessible from the internet but the private instances can now access the internet.
    - Edited the private route table to add a new destination on (0.0.0.0/0) as the target to Project-NAT-Gateway
- Security groups:
    - Created a security group (BastionHost-SG) for Bastion Host on Project-VPC. This security group ensures that the bastion host is only allowed access to SSH on port 22 with all traffic for the outbound rules, and ONLY on my IP address, this ensures security.
    - Created a security group (Private-SG) for private instances on Project-VPC. The inbound rule only allows SSH access on port 22 using the BastionHost-SG to add scalability.
    - Since the BastionHost-SG only allows access with SSH on port 22 with my IP, this could be a problem for my private instances as they will get blocked SSH access. To fix this, I edited the BastionHost-SG to allow inbound traffic SSH on port 22 as my Private-SG security group.
    - Created a security group (ApplicationLoadBalancer-SG) for Application Load Balancer. This security group will only allow DNS traffic on port 80 from anywhere (0.0.0.0/0) this will ensure that the public instance’s IP address can’t be accessed.
    - Created a security group (AutoScalingGroup-SG) for Auto Scaling Group. This will allow the public instances on port 80 only from the ApplicationLoadBalancer-SG to restrict access to only the load balancer.
- EC2 Instance:
    - BastionHost:
        - Created a public instance (BastionHost) with a T3.micro instance type on Project-VPC and Project-Public-Subnet-1 (us-east-1a). Auto-assign public IP is enabled and attached the BastionHost-SG security group that was created prior. This instance will be used as a jump box to connect to private instances.
        - Created an RSA keypair (keypair1) this key is encrypted and will be used to SSH into instances for added security.
    - Private-Instance:
        - Created a private instance (Private-Instance) with t3.micro instance type on Project-VPC and Project-Private-Subnet-1 (us-east-1a). This instance is used for internal backend control. Currently used the RSA keypair1 and have auto-assign public set to disabled as default. Attached the Private-SG security group.
    - Auto Scaling Group:
        - Created an auto scaling group (Project-Auto-Scaling-Group)
            - Created the launch template (Project-Launch-Template), it was unable to select the Project-VPC and multiple subnets so I left it blank. The security group is tied with the VPC so it should fix this nonetheless.
            - Used the same keypair1 and attached the AutoScalingGroup-SG
            - On the launch template user data I plugged in this script:
            - #!/bin/bash
            - #to update, download http, start and enable httpd, and to add a text file in /var/www/html/index.html
            - dnf update -y
            - dnf install httpd -y
            - systemctl start httpd
            - systemctl enable httpd
            - echo “Hello world from $(hostname -f)” > /var/www/html/index.html
            - Attached the Auto Scaling Group Launch Template. Proceeded to step 2 and on the Network section, selected the Project-VPC and 2 public subnets (Project-Public-Subnet 1 and 2) with balanced effort as availability zone distribution.
            - Created a load balancer (Project-ALB) with internet-facing from Project-Public-Subnet-1 and 2. This load balancer will distribute instances within these 2 availability zones to ensure high availability.
            - Created a target group (Project-ALB-TG) on port 80 to target the public instances for the application load balancer.
            - Turned on Elastic Load Balancer health checks to determine whether the instances are healthy or not. Useful for monitoring.
            - Set the desired capacity to 3, min capacity to 1 and max desired capacity to 5.
            - Set target tracking policy of average CPU utilization of target value to 70. This ensures that when instances CPU utilization reaches above the 70% threshold, it will automatically scale.
            - Instance maintenance policy is set to no policy and additional capacity settings to default.
    - IAM Role:
        - Created a new IAM role with trusted entity type as AWS service and the use case as EC2.
        - Added AmazonS3ReadOnlyAccess and the role name as EC2-VPC-Endpoint-S3ReadOnlyAccess.
        - On the Private-Instance, right clicked to select security and modify the IAM role to add EC2-VPC-Endpoint-S3ReadOnlyAccess.
    - VPC Endpoint:
        - Created a VPC endpoint (S3-VPC-Enpoint) as type AWS services. Under services, selected S3 gateway endpoint with the Project-Private-RT route table on Project-VPC. There’s only 2 of these available only to S3 and DynamoDB. This endpoint is cost optimized than VPC interface endpoint.
        - VPC endpoint allows internal interactions within the VPC without accessing the internet. This implements secure control and low latency.

# Testing and Validation

- BastionHost connectivity:
    - Successfully connected to the public instance (BastionHost) via SSH using my Windows Powershell.
    - Pinged [google.com](http://google.com) to test internet gateway is working by accessing the internet. BastionHost is a public instance so connectivity with internet is a success.
    - Created a file and restricted access using sudo chmod 400 <filename> to ensure that only the root can access it.
- Private-Instance connectivity:
    - Successfully SSH into private instance (Private-Instance) using the keypair1 after modifying permissions to chmod 400.
    - Pinged [google](http://googlec.om).com to check if NAT gateway connectivity is working, and it is. Successfully SSH and pinged for internet access.
- Application Load Balancer:
    - Successfully isolated DNS using the Application-Load-Balancer-SG. The Security group specified to allow connections only from HTTP on port 80. The AutoScalingGroup-SG only allows inbound traffic from HTTP on port 80 making the public IP address inaccessible. This ensures tight security.
    - Application load balancer is running smoothly, refreshed a few times and traffic are being distributed evenly through the DNS.
- VPC endpoint:
    - Successfully listed the S3 buckets from the private instance (Private-Instance) with the IAM role attached, this confirmed access to private resources via the VPC endpoint.

# Challenges

- For the BastionHost-SG, its only allowed to my IP so if I connect using my private instances, it wont be able to enter through. After modifying the BastionHost-SG to allow SSH traffic from Private-SG, I tested the SSH connectivity from the Bastion Host to the private instances, which resolved the issue.
- The userdata script did not work as the quotation marks were incorrectly used. When running the script from launch, the server was unable to write to /var/www/html/index.html because instead of straight quotation marks “ I ended up using a a curly one. This prevented the script from running. To fix this, I have updated the launch template from the AWS console. Then, I initiated an instance refresh from the ASG to apply the updated script across all instances.

# Future Improvements

- Convert the entire manual infrastructure using Terraform or CloudFormation to reduce deployment time by 70% and ensure consistent infrastructure as code (IaC).
- Adding centralized monitoring and logging using Cloudwatch alarms to monitor key metrics like CPU utilization which will give better visibility into the health of EC2 instances and ensure early detection of any issues.
