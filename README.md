 # Define the provider
provider "aws" {
  region = "us-east-1"  # Change to your desired region
}

# Create a VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = {
    Name = "MyVPC"
  }
}

# Create a public subnet
resource "aws_subnet" "public_subnet_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"  # Change to your desired AZ
  map_public_ip_on_launch = true
  tags = {
    Name = "PublicSubnet1"
  }
}

# Create another public subnet
resource "aws_subnet" "public_subnet_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"  # Change to your desired AZ
  map_public_ip_on_launch = true
  tags = {
    Name = "PublicSubnet2"
  }
}

# Create an Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "MyInternetGateway"
  }
}

# Create a route table
resource "aws_route_table" "main" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "MyRouteTable"
  }
}

# Associate the route table with the subnets
resource "aws_route_table_association" "assoc_subnet_1" {
  subnet_id      = aws_subnet.public_subnet_1.id
  route_table_id = aws_route_table.main.id
}

resource "aws_route_table_association" "assoc_subnet_2" {
  subnet_id      = aws_subnet.public_subnet_2.id
  route_table_id = aws_route_table.main.id
}

# Create a security group with inbound rules for HTTP (8080) and SSH (22)
resource "aws_security_group" "allow_http_ssh" {
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "AllowHTTPandSSH"
  }
}

# Launch an EC2 instance for Jenkins and Trivy
resource "aws_instance" "jenkins_trivy" {
  ami           = "ami-02c21308fed24a8ab"  # Amazon Linux 2 AMI (change if needed)
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet_1.id
  security_groups = [aws_security_group.allow_http_ssh.name]

  tags = {
    Name = "JenkinsAndTrivy"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y java-1.8.0-openjdk
              wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat/jenkins.repo
              rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key
              yum install -y jenkins
              systemctl start jenkins
              systemctl enable jenkins
              yum install -y git
              git clone https://github.com/aquasecurity/trivy.git
              cd trivy
              cp trivy /usr/local/bin/
              EOF
}

# Launch the first EC2 instance for Tomcat
resource "aws_instance" "tomcat_1" {
  ami           = "ami-02c21308fed24a8ab"  # Amazon Linux 2 AMI (change if needed)
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet_2.id
  security_groups = [aws_security_group.allow_http_ssh.name]

  tags = {
    Name = "TomcatInstance1"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y java-1.8.0-openjdk
              wget http://apache.mirrors.tds.net/tomcat/tomcat-9/v9.0.93/bin/apache-tomcat-9.0.93.tar.gz
              tar xvf apache-tomcat-9.0.93.tar.gz
              mv apache-tomcat-9.0.93 /opt/tomcat
              /opt/tomcat/bin/startup.sh
              EOF
}

# Launch the second EC2 instance for Tomcat
resource "aws_instance" "tomcat_2" {
  ami           = "ami-02c21308fed24a8ab"  # Amazon Linux 2 AMI (change if needed)
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet_2.id
  security_groups = [aws_security_group.allow_http_ssh.name]

  tags = {
    Name = "TomcatInstance2"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y java-1.8.0-openjdk
              wget http://apache.mirrors.tds.net/tomcat/tomcat-9/v9.0.93/bin/apache-tomcat-9.0.93.tar.gz
              tar xvf apache-tomcat-9.0.93.tar.gz
              mv apache-tomcat-9.0.93 /opt/tomcat
              /opt/tomcat/bin/startup.sh
              EOF
}
