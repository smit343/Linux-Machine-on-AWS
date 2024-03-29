# wecloud_project1
# Bash shell script that will create AWS infrastructure as per the requirement
# Team members: Shivam Tayal and Smitkumar Patel
# Public URL: https://github.com/shiv-1991-shiv/wecloud_project1.git
#Architectural diagram![image](https://github.com/shiv-1991-shiv/wecloud_project1/assets/141082654/01a39de3-91ee-429d-8120-79ed531b9bf6)
# Instructions to run the script:
# 1. Download the file
# 2. Using the Linux command line, change the permissions of the file linux_server_on_aws.sh to 755, using chmod 755 filename
# 3. Execute the script using ./linux_server_on_aws.sh

#!/bin/bash

REGION="us-east-1"
SUBNET_PUBLIC_AZ="us-east-1a"

#Create VPC
VPC_ID=$(aws ec2 create-vpc \
--cidr-block 10.0.0.0/16 \
--tag-specification 'ResourceType=vpc,Tags=[{Key=Name,Value=vpc_project}, {Key=project,Value=wecloud}]' \
--region $REGION \
--output text \
--query 'Vpc.VpcId')

echo "VPC $VPC_ID is created."

#Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \a
--tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=igw-project}, {Key=project,Value=wecloud}]' \
--output text \
--query 'InternetGateway.InternetGatewayId')

echo "IGW $IGW_ID is created."

#Attach Internet Gateway to VPC
aws ec2 attach-internet-gateway \
--internet-gateway-id $IGW_ID \
--vpc-id $VPC_ID

echo "IGW $IGW_ID is attached to VPC $VPC_ID."

#Create public subnet
SUBNET_ID=$(aws ec2 create-subnet \
--vpc-id $VPC_ID \
--cidr-block 10.0.0.0/24 \
--tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet_project}, {Key=project,Value=wecloud}]' \
--availability-zone $SUBNET_PUBLIC_AZ \
--region $REGION \
--output text \
--query 'Subnet.SubnetId')

echo "Subnet $SUBNET_ID is created successfully."

#Enable subnet to auto-assign public IP
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_ID --map-public-ip-on-launch
echo "Public IP auto-assign is enabled."

#Create route table
RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --output text --query 'RouteTable.RouteTableId' \
--tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt_project}, {Key=project,Value=wecloud}]')

echo "Route Table $RT_ID is created."

#Create route to internet gateway
aws ec2 create-route --route-table-id $RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

echo "Route to the IGW $IGW_ID is created."

#Associate the subnet with the route table
aws ec2 associate-route-table --subnet-id $SUBNET_ID --route-table-id $RT_ID

echo "Subnet $SUBNET_ID is associated with route table $RT_ID."

#Create security group
SG_ID=$(aws ec2 create-security-group \
    --group-name project_sg \
    --description "SG to allow SSH Access" \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg_project}, {Key=project,Value=wecloud}]' \
    --vpc-id $VPC_ID \
    --output text \
    --query 'GroupId')

echo "Security group $SG_ID created successfully."


#Enable the security group to allow SSH and ICMP access
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol icmp --port -1 --source-group $SG_ID
echo "Security group $SG_ID is authorized for SSH."

#Create key-pair
aws ec2 create-key-pair \
--key-name project_key \
--key-type rsa \
--query 'KeyMaterial' \
--output text \
> project_key.pem 

echo "Key-pair 'project_key.pem' is created successfully." 

#Create EC2 Instance Master-Node-1
MASTER_NODE=$(aws ec2 run-instances \
    --image-id ami-0aa2b7722dc1b5612 \
    --count 1 \
    --instance-type t2.small \
    --key-name project_key \
    --subnet-id $SUBNET_ID \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=master-node-01}, {Key=project,Value=wecloud}]' \
    --user-data file://userdata.sh \
    --security-group-ids $SG_ID \
    --output text \
    --query 'Instances[0].InstanceId')

echo "Instance Master Node $MASTER_NODE is created successfully."

#Create EC2 Instance Worker-Node-1
WORKER_NODE1=$(aws ec2 run-instances \
    --image-id ami-0aa2b7722dc1b5612 \
    --count 1 \
    --instance-type t2.micro \
    --key-name project_key \
    --subnet-id $SUBNET_ID \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-node-01}, {Key=project,Value=wecloud}]' \
    --user-data file://userdata.sh \
    --security-group-ids $SG_ID \
    --output text \
    --query 'Instances[0].InstanceId')

echo "Instance Worker Node 1 $WORKER_NODE1 is created."

#Create EC2 Instance Worker-Node-2
WORKER_NODE2=$(aws ec2 run-instances \
    --image-id ami-0aa2b7722dc1b5612 \
    --count 1 \
    --instance-type t2.micro \
    --key-name project_key \
    --subnet-id $SUBNET_ID \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-node-02}, {Key=project,Value=wecloud}]' \
    --user-data file://userdata.sh \
    --security-group-ids $SG_ID \
    --output text \
    --query 'Instances[0].InstanceId')

echo "Instance Worker Node 2 $WORKER_NODE2 is created."
