# Three-tier-Architecture-On-AWS
 Deploying A Complex, Production Level, Three-tier Architecture On AWS

<img width="800" alt="image" src="https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/a27904d6-2e88-4b8a-af0e-d31d0ea0c901">




A Practical Guide To Deploying A Complex, Production Level, Three-tier Architecture On AWS

Multi-tiered architectures have become the most popular way of designing and building applications in the cloud. This is primarily because they offer high availability, replication, flexibility, security and many other numerous benefits. This is as opposed to single-tier architectures which typically involve packaging all the requisite components of a software application into a single server.

The most popular multi-tier design pattern is a three-tier architecture. The three-tier architecture consists of the following layers:

Presentation Layer: This is the outermost layer of the application and provides an interface for interacting with the user. It also provides a secure communication channel with the other tiers.
Logic Layer: This is where information is processed and application logic is executed. It processes input from the presentation layers, processes it and also communicates with the data layer when necessary. it is also known as the application layer or middleware.
Data Layer: This is where the database management system sits, thus providing it with a secure, isolated environment for storing and managing application information.
In this guide, we are going to design a very fault-tolerant, highly scalable Flask application on AWS using the three-tier architecture design pattern.

Architectural Design
The Presentation layer will comprise of; Cloudfront, Elastic Load Balancer, Internet Gateway, NAT Gateway and two Bastion Hosts.
The Application (Logic) Layer will consist of EC2 instances based on EBS volumes, provisioned through an Auto Scaling group.
The Data Layer will be made up of a PostgreSQL Database with a Read Replica. The EC2 instances provisioned in the application layer will be connected to an Elastic Filesystem for data storage. So technically, the EFS is also residing in this layer.

Bonus:

AWS Route 53 will be used to provide a domain name for the application.
We will integrate ClodFront with the Application Load Balancer to provide worldwide accelerated content delivery and reduce latency.
Terraform will be used as the Infrastructure-as-code (IAC) tool to automate this whole process from end to end.
To automatically create this architecture from end to end, click on this Link to get the Terraform code.

This is a first part of a three-series project.
In the second project, we will implement DevOps on AWS via CodePipeline, CodeBuild, and CodeDeploy.
The third project introduces a Reporting Layer into the architecture.
This part focuses solely on designing and deploying a three-tier architecture for a Flask application on AWS.


Step 1: Create The Environment.
Firstly we will create a VPC. A virtual private cloud is a secure, isolated, networking environment hosted in a public Cloud. A VPC is more or less a virtual data center in the cloud. This is where most of our application resources will be hosted.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/6002c71f-8aa6-4cda-9d9b-e20b60055a0f)

My VPC has a name tag of project-x. This means that all other resources created within this VPC will have the prefix of "project-x".
You can also notice that I am assigning a CIDR block of 10.0.0.0/16 to my VPC. This is a block of private IP addresses from which my VPC resources will get their local addresses.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/92809970-8be7-4ac1-a9d6-c272638eb15c)

Notice that I am in the us-east-1 region (North Virginia).
Our VPC will be provisioned in three availability zones. 6 subnets will be created in total. Three public and three private ones. The public subnets are public soley because an internet gateway will be attached to them.
Also notice that I am creating a NAT gateway. This NAT gateway will be used by the EC2 instances hosting our application to connect to the internet (E.g: For Updates).

Step 2: Create A Security Group
A security group acts like a firewall and determines how traffic will enter or exit instances.
Since our web servers will be hosted in the logic-tier, it is important that we restrict the type of traffic entering them.

The name of our security group will be "project-x-logic-tier-sg"
The security group must allow inbound traffic on port 5000 since that is the port on which Flask listens on.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/53685108-5211-4cff-85ff-6f80be0b305b)

Step 3: Create A Launch Template
A Launch Template is a way of specifying important configuration details such that we can then launch instances using the details in the template. This template will be used by an auto scaling group to create instances in our Application tier.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/522e08de-6aaa-4b6e-a0be-a69f6ba45dad)

I assigned a name and description to my template. Notice also that i duly tagged the environment as "prod".

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/e797d787-993f-449a-a05e-4063a479f1a5)

I am using a "t3.xlarge" instance type.
Be sure to select the "project-x-logic-tier-sg" security group.
Leave every other detting as is.
Under "Advanced details", I am going to paste in the following under user data:


#!/bin/bash

# Mount EFS
fsname=fs-093de1afae7166759.efs.us-east-1.amazonaws.com # You must change this value to represent your EFS DNS name.
mkdir /efs
mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport $fsname:/ /efs

# install and set up Flask
apt update
apt upgrade -y
apt install python3-flask mysql-client mysql-server python3-pip python3-venv -y
apt install sox ffmpeg libcairo2 libcairo2-dev -y
apt install python3-dev default-libmysqlclient-dev build-essential -y



Step 4: Create an Autoscaling Group, Elastic Load balancer And Target Group
The Load balancer is the entry point to the application.
The Application Load Balancer, residing in the presentation layer, will route traffic through the AutoScaling Group to logic-tier instances residing in the logic layer.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/d418845c-3645-437b-a95c-0741f557b7b4)

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/0e326061-0def-42ea-b7d1-34608fb2fd59)

I am naming this Autoscaling group "project-x-asg"
Click on next. Select the "project-x-vpc" we created earlier. Also make sure that you select only the private subnets from the three availability zones. This is very crucial since our VPC will be launched in the logic tier. The subnets must not be public.


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/35ee8fab-15fd-4194-8203-8f255b02b7e6)


Click on next.
Under "Configure advanced options", select "Attach to a new load balancer". Also select "Create a target group".
On the next page, we'll configure our ASG to have a minium capacity of 2, a desired capacity of 4 and a maximum capacity of 6. We'll also set up scaling based on target tracking metrics to scale based on average CPU Utilization.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/bb4b838b-9dce-400e-8831-d2206b4e2ee8)

You can decide to add tags. I'll add a tag name of "Environment" with a value of "Prod". this will be needed during the CodeDeploy stage.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/d0bf9b39-637d-4369-ae08-76db503da50c)


Create your ASG.

Step 5: Attach An Elastic Filesystem
AWS EFS is a fully managed, highly scalable shared storage solution in the cloud. It is NFS compatible.
This Elastic filesystem will provide shared storage for all our application tier servers. Since it provides storage, EFS sits in the data layer of the three tier architecture.

First Create a new security group. This security group should allow only inbound NFS traffic from the security group of our logic-tier instances. You can get a detailed guide on how to create that Here
**
Go to the EFS console. Click on "Create filesystem*" and then click on **customize*.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/a5c33a8e-08ef-4d71-8666-3253b9474c30)

Assign a name to your EFS. Leave every other setting as default. Click on Next.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/5e527588-322f-41ad-870e-e9e3bd4b0b88)

On the Network access page, select your propject-x VPC. Select the EFS security group you have created. Click on next.

Under Filesystem Policy, leave everything as default.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/82a47961-db76-4108-862e-7064c99a0260)


Skip to Review and then Create your filesystem.
Now we must update our user data to look like this:



#!/bin/bash

# Use Google's DNS
echo "nameserver 8.8.8.8" >> /etc/resolv.conf

# Force apt to use IPV4
apt-get -o Acquire::ForceIPv4=true update

# Change hostname
echo "project-x-app-server" > /etc/hostname

# Install efs-utils
apt-get install awscli -y
mkdir /efs
sudo apt-get -y install git binutils
git clone https://github.com/aws/efs-utils
cd /efs-utils
./build-deb.sh
apt-get -y install ./build/amazon-efs-utils*deb

# Mount EFS
fsname=$(aws efs describe-file-systems --region us-east-1 --creation-token project-x --output table |grep FileSystemId |awk '{print $(NF-1)}')
mount -t efs $fsname /efs

# Get DB credentials
DB=$(aws rds describe-db-instances --db-instance-identifier --region us-east-1 database-1 --output table |grep DBName |awk '{print $(NF-1)}')
HOST=$( aws rds describe-db-instances --db-instance-identifier --region us-east-1 database-1 --output table |grep Address |awk '{print $(NF-1)}')
ARN=$(aws secretsmanager list-secrets --region us-east-1 --filters "Key=tag-value, Values=project-x-rds-mysqldb-instance" --output table |grep ARN |awk '{print $(NF-1)}')
USER=$(aws secretsmanager get-secret-value --region us-east-1 --secret-id $ARN --output table |grep -w SecretString |awk '{print $3}' |cut -d: -f2 |sed 's/password//' |tr -d '",')
PRE_PASSWORD=$(aws secretsmanager get-secret-value --region us-east-1 --secret-id $ARN --output table |grep -w SecretString |awk '{print $3}' |cut -d: -f3 |tr -d '"')
PASSWORD=${PRE_PASSWORD%?}

# install and set up Flask
apt-get update -y && apt-get upgrade -y 
apt-get install python3-flask mysql-client mysql-server python3-pip python3-venv -y 
apt-get install sox ffmpeg libcairo2 libcairo2-dev -y 
apt-get install python3-dev default-libmysqlclient-dev build-essential -y 

# Clone the app
cd /
git clone https://github.com/Kelvinskell/terra-tier.git
cd /terra-tier

# Populate App with environmental variables
echo "MYSQL_ROOT_PASSWORD=$PASSWORD" > .env
cd /terra-tier/application
echo "MYSQL_DB=$DB" > .env
echo "MYSQL_HOST=$HOST" >> .env
echo "MYSQL_USER=$USER" >> .env
echo "DATABASE_PASSWORD=$PASSWORD" >> .env
echo "MYSQL_ROOT_PASSWORD=$PASSWORD" >> .env
echo "SECRET_KEY=08dae760c2488d8a0dca1bfb" >> .env # FLASK EXTENSION KEY. NOT NECESSARILY A "SECRET".
echo "API_KEY=f39307bb61fb31ea2c458479762b9acc" >> .env 
# YOU TYPICALLY DON'T ADD SECRETS SUCH AS API KEYS AS PART OF SOURCE CONTROL IN PLAIN TEXT.
# THIS IS BEIGN ADDED HERE SO THAT YOU CAN EASILY REPLICATE THIS INFRASTRUCTURE WITHOUT ANY HASSLES.
# YOU CAN REPLACE IT WITH YOUR OWN MEDIASTACK API KEY.

# Setup virtual environment
cd /terra-tier
python3 -m venv venv
source venv/bin/activate

# Run Flask Application
pip install -r requirements.txt
export FLASK_APP=run.py
export FLASK_ENV=production
flask run -h 0.0.0.




Here is where you find your filesystem's DNS name

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/79ce8dc6-e205-4e1b-b5c5-a765a1848c07)

Make sure to change the value of the "fsname" variable to represent your own filesystem's DNS name.

Step 6: Create A Bastion Host
A Bastion Host is a special server used to manage access to servers sitting in an internal network or other private AWS resources from an external network. The bastion host sits in the public network and provides limited access to administrators to log in to servers sitting in an isolated network. It is also commonly referred to as a Jump Box or Jump Server.

From the bastion host, we'll be able to gain SSH access into our application layer servers, for administrative purposes.

Here is a good resource on how to create your bastion host and connect to your logic layer servers through it.
Security Caution: Connecting to bastion hosts using ssh key-pairs is no longer the recommended practice. Instead use AWS Systems Manager for a more secure connection and tunneling through the bastion host to your private instances.

Step 7: Create A Database
The database solidly sits in the data layer, together with the Elastic filesystem.
Our Flask application will need to connect to a relational database. We will be using MYSQL for this architecure.
MYSQL is a widely relational database management system (DBMS) which is free and open source. MYSQL is renowned for its ability to support large production workloads, hence its suitability for our project.

We will however be utilizing Amazon RDS for MYSQL. It is a fully managed database solution in the cloud.

On the RDS Console, click on Databases and then clcik on "Create database".


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/97cb84d4-4e9b-497a-92bd-a947c58a61a5)


I am selecting a Production template and choosing Multi-AZ DB-instance.


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/8bf6923c-59f2-4a6e-9b22-217f7a85a787)


Select your master username and DB Instance Class. We'll be using AWS SEcrets manager for managing our DB credentials.


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/4bd408a5-681a-436d-95db-0bdee01fe4e6)


Under "Connectivity", select the project-x-vpc. Click on "Create a new security group".


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/0f11ae17-56a5-4a4a-b365-12725731b387)


Create a security group that only allows incoming traffic on port 3306 from security group of the application layer servers. Select the security group.

Under "Database Authentication", choose "password".

Go down to "Advanced Options", Under "database name", choose "newsreadb". (You must choose this exact name for the Flask App to be able to connect to the database).

Click on Create database.

Now our architecture is almost ready !!!


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/c0aae298-5986-4c18-97a2-19a75fe6de23)



To connect to the database, our application will execute API calls to AWS Secrets Manager in order to get the DB credentials. Therefore, we must create an IAM role that allows it to perform that. We will also modify the launch template to attach the role as instance profile for our application layer servers.

Step 8: Create An IAM Role And Modify Launch Template
Go to the IAM Dashboard and click on Roles.
Click on Create Role. Under UseCase, select EC2 and click Next.
Search for secrets and select the permission that pops up.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/5c302c67-e782-46a0-8063-a624cd7430a0)

Click on next and click on Create Role.

Next, we'll need to modify our launch template to include this role so that our logic tier servers can assume it to communicate with AWS Secrets Manager.

Go to the EC2 Console, select launch templates, Actions and click on "Modify launch template"

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/caa5a921-b32e-4bcd-910a-8065c80db0e2)

Go to "Advanced details", Click on the box under IAM instance profile and then select your newly created role. Click on Create.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/e739bde4-01f3-4487-ad28-c2ac386a05f9)

Finally, be sure to set the new version as the default versioN

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/b6e9e91b-ea62-4765-abb8-a3f5b1a1d6e4)


AND YOU're DONE !!!!!!!
Our three-tier application should now be up and running.

Step 9: Accessing Your Application
If you have followed through with all these steps, Congrats.
Now to access your application, you have to visit the EC2 console to get the domain name of your load balancer.

Go to the EC2 Console and click on Load Balancers.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/4346c4a7-7cc7-4c8c-aeb7-e1884af225f6)

Copy the DNS Name

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/e815fb78-2b71-4c2d-bc60-6642ab379dea)


Now paste this into a web browser.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/758aae38-9730-4efa-9518-fe7f2fae1436)


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/e27a611e-07bb-43e0-bc34-80961dc45a8f)


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/6cebf1d8-a36c-41f0-80b9-7ee619d718ac)




Now Onto The Bonus Section.
Firstly, Here ( https://github.com/envyvalery/3-tier-Automation-IAC-) is terraform code that automates this process from end to end.

Now we will provide a domain name for our application using AWS Route53. Finally, we will integrate our Application Load Balancer with CloudFront for accelerated content delivery to users. This is a necessary step if you were to run this architecture in a production environment.

Step 1: Create A Domain Name

An active domain name is required to follow through with the remaining part of this project.
You can either purchase a domain directly from Route53 or you can use other providers such as GoDaddy or Namecheap.

Step 2: Create A Hosted Zone
A hosted zone is a container for records, which define how you want to route traffic to your domain.
To create a public hosted zone, go to the Route53 Console, Click on Hosted Zones and populate the fields with your values.



![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/193bf8f4-bcbe-4e28-9cb6-a9be74019502)

Step 3: Create An SSL Certificate
An SSL/TLS certificate is a digital object that allows systems to verify the identity & subsequently establish an encrypted network connection to another system using the Secure Sockets Layer/Transport Layer Security (SSL/TLS) protocol.

AWS Certificate Manager is a service that provisons and manages public and private SSL/TLS certificates.

Go to the Certificate Manager Console and click on "Request Certificate".
Enter your fully qualified domain name and click on Request. It normally takes some time for AWS to approve this request, especially if you purchased a domain name outside of AWS. So you have to exercise a bit of patience here

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/dd7fb1ba-7cc8-4231-9f19-7156f9bad4da)

You will need this certificate while provisioning your CloudFront distribution In a production environment.
However, I won't be using HTTPS here due to logistical reasons. So this step isn't necessary to follow through with the project.

Step 4: Create A CloudFront Distribution And Integrate With Your ALB
AWS CloudFront is a Contnet Delivery Network that provides your data globally to your users with very low latency. It does this by routing users request through the nearest edge location.
We'll integrate CloudFront with our ALB for accelerated dynamic contnet delivery.
N.B: In a production environment, you'd want to implement SSL termination on the ALB. But since we're not using SSL Certificates in this project, you won't need to do this to follow through.

Go to the CloudFront Console, Click on "Create Distribution".
Select your Application Load Balancer as the Origin Domain


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/eb48e2db-782b-4346-955d-8fd6a7a9eb36)

Choose HTTP Only. Leave other fields as default.


![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/24d7426c-93ec-49c4-8971-edfff58aecd4)

Select "CachingDisabled" as your caching policy. Origin request policy should be AllViewer.

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/f3159500-3f04-4476-9ea1-a14b789396e8)


Click on "Create Distribution"

Step 5 (Final Step): Copy The Distribution Domain Name And Paste In A Web Browser

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/9c3365dd-3602-4137-9d01-cfb175446069)

Aaaaaand... Here we go!!!!

![image](https://github.com/envyvalery/Three-tier-Architecture-On-AWS/assets/140969141/ab577ac4-a70e-444c-861f-387770d9b912)


Wrap Up
Three-tier architectures are a well established pattern of software application deployment. They make the application architecture fully resilient, fault tolerant and well secure.

After going through this project, you should be well equipped to design and deploy similar architectures on AWS.
Remember to check out the second part of this project series which focuses on CI/CD.

Happy Clouding!!!
























