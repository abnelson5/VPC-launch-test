a detailed, step-by-step guide to launch your VPC, create two EC2 instances (one in a public subnet and one in a private subnet), and configure connectivity:

Step 1: Create the VPC and Subnets
1.1 Create a VPC
Open the AWS Management Console.
Navigate to VPC → Your VPCs → Click Create VPC.
Configure:
Name tag: MyVPC
IPv4 CIDR Block: 10.0.0.0/16
Leave other settings as default and click Create.
1.2 Create a Public Subnet
Go to VPC → Subnets → Click Create subnet.
Configure:
VPC ID: Select MyVPC.
Subnet name: PublicSubnet.
Availability Zone: Pick any zone (e.g., us-east-1a).
IPv4 CIDR Block: 10.0.1.0/24.
Click Create subnet.
1.3 Create a Private Subnet
Go to VPC → Subnets → Click Create subnet.
Configure:
VPC ID: Select MyVPC.
Subnet name: PrivateSubnet.
Availability Zone: Use the same zone as the public subnet.
IPv4 CIDR Block: 10.0.2.0/24.
Click Create subnet.
1.4 Enable Auto-Assign Public IP for Public Subnet
Go to VPC → Subnets → Select PublicSubnet.
Click Actions → Edit subnet settings.
Enable Auto-assign public IPv4.
Save changes.
Step 2: Configure Route Tables
2.1 Attach an Internet Gateway
Go to VPC → Internet Gateways → Click Create internet gateway.
Name tag: MyInternetGateway.
Attach the internet gateway to MyVPC:
Select MyInternetGateway, click Actions → Attach to VPC → Choose MyVPC.
2.2 Set Up a Public Route Table
Go to VPC → Route Tables → Click Create route table.
Name tag: PublicRouteTable.
VPC: Select MyVPC.
Edit the route table:
Select PublicRouteTable, click Routes → Edit routes → Add route.
Destination: 0.0.0.0/0
Target: Internet Gateway → Select MyInternetGateway.
Save changes.
Associate the public route table with PublicSubnet:
Select PublicRouteTable → Subnet associations → Edit subnet associations.
Select PublicSubnet and save.
2.3 Set Up a Private Route Table
Go to VPC → Route Tables → Click Create route table.
Name tag: PrivateRouteTable.
VPC: Select MyVPC.
Associate the private route table with PrivateSubnet:
Select PrivateRouteTable → Subnet associations → Edit subnet associations.
Select PrivateSubnet and save.
Step 3: Launch EC2 Instances
3.1 Launch Public EC2
Go to EC2 → Instances → Click Launch instance.
Configure:
Name: PublicEC2.
AMI: Amazon Linux 2 (or your preferred OS).
Instance type: t2.micro (free tier eligible).
Key pair: Create or select an existing key pair.
Network settings:
VPC: MyVPC.
Subnet: PublicSubnet.
Auto-assign public IP: Enabled.
Security group: Create a new one:
Allow SSH: Port 22 → MyIP.
Launch the instance.
3.2 Launch Private EC2
Go to EC2 → Instances → Click Launch instance.
Configure:
Name: PrivateEC2.
AMI: Amazon Linux 2 (or your preferred OS).
Instance type: t2.micro (free tier eligible).
Key pair: Use the same key pair as PublicEC2.
Network settings:
VPC: MyVPC.
Subnet: PrivateSubnet.
Auto-assign public IP: Disabled.
Security group: Create a new one:
Allow SSH: Port 22 → 10.0.1.0/24 (public subnet CIDR).
Launch the instance.
Step 4: Configure Security Groups
4.1 PublicEC2 Security Group
Inbound Rules:
SSH: Port 22 → Your Public IP.
Allow traffic from private subnet: All Traffic → 10.0.2.0/24.
4.2 PrivateEC2 Security Group
Inbound Rules:
SSH: Port 22 → 10.0.1.0/24 (public subnet CIDR).
Step 5: Test Connectivity
5.1 Connect to Public EC2
From your local machine, SSH into the public EC2 instance:
bash
Copy code
ssh -i /path/to/key.pem ec2-user@<Public EC2 Public IP>
5.2 SSH from Public EC2 to Private EC2
Use the private IP address of PrivateEC2:
bash
Copy code
ssh -i /path/to/key.pem ec2-user@<Private EC2 Private IP>



# Provider Configuration
provider "aws" {
  region = "us-east-1" # Change the region if needed
}

# VPC
resource "aws_vpc" "my_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "MyVPC"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "my_igw" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "MyInternetGateway"
  }
}

# Public Subnet
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.my_vpc.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"
  tags = {
    Name = "PublicSubnet"
  }
}

# Private Subnet
resource "aws_subnet" "private_subnet" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1a"
  tags = {
    Name = "PrivateSubnet"
  }
}

# Public Route Table
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "PublicRouteTable"
  }
}

# Route to Internet Gateway
resource "aws_route" "public_route" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.my_igw.id
}

# Associate Public Route Table with Public Subnet
resource "aws_route_table_association" "public_rta" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

# Private Route Table
resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "PrivateRouteTable"
  }
}

# Associate Private Route Table with Private Subnet
resource "aws_route_table_association" "private_rta" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.private_rt.id
}

# Security Group for Public EC2
resource "aws_security_group" "public_sg" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "PublicSecurityGroup"
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with your IP in production
  }

  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["10.0.2.0/24"] # Private Subnet CIDR
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group for Private EC2
resource "aws_security_group" "private_sg" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "PrivateSecurityGroup"
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.1.0/24"] # Public Subnet CIDR
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Key Pair
resource "aws_key_pair" "my_key" {
  key_name   = "my-key"
  public_key = file("~/.ssh/id_rsa.pub") # Path to your public key
}

# Public EC2 Instance
resource "aws_instance" "public_ec2" {
  ami           = "ami-0c02fb55956c7d316" # Amazon Linux 2 AMI ID for us-east-1
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet.id
  key_name      = aws_key_pair.my_key.key_name
  security_groups = [
    aws_security_group.public_sg.name
  ]

  tags = {
    Name = "PublicEC2"
  }
}

# Private EC2 Instance
resource "aws_instance" "private_ec2" {
  ami           = "ami-0c02fb55956c7d316" # Amazon Linux 2 AMI ID for us-east-1
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.private_subnet.id
  key_name      = aws_key_pair.my_key.key_name
  security_groups = [
    aws_security_group.private_sg.name
  ]

  tags = {
    Name = "PrivateEC2"
  }
}
