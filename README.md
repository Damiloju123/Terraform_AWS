# Terraform_AWS
Created an EC2 Instance, deployed on a custom VPC &amp; Subnet and assigned a Public IP Address, SSH for connection to make changes automatically and  setup a web server hosted on Ubuntu


Step 1: Create a key pair within AW. This is for connectivity from Terraform to AWS Console

![A](https://user-images.githubusercontent.com/52894481/189686307-af15620e-dc97-4456-ad69-ecba1e9265b7.JPG)


Step 2:  Create vpc

resource "aws_vpc" "prod-vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
      Name = "production"
  }
}


Step 3: Create internet gateway

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.prod-vpc.id

}

Step 4: Create custom route table

resource "aws_route_table" "prod-route-table" {
  vpc_id = aws_vpc.prod-vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  route {
    ipv6_cidr_block        = "::/0"
    gateway_id             = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "Prod"
  }
}

Step 5: Create a subnet

resource "aws_subnet" "subnet-1" {
    vpc_id     = aws_vpc.prod-vpc.id
    cidr_block = "10.0.1.0/24"
    availability_zone = "us-east-1a"
    tags = {
         Name = "prod-subnet"
    }
}

Step 6:  Associate a subnet with Route Table

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.subnet-1.id
  route_table_id = aws_route_table.prod-route-table.id
}

Step 7: Create Security Group to allow port 22, 80, 443

resource "aws_security_group" "allow_web" {
  name        = "allow_web_traffic"
  description = "Allow web traffic"
  vpc_id      = aws_vpc.prod-vpc.id

  ingress {
    description      = "HTTPS"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  ingress {
    description      = "HTTP"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
    ingress {
    description      = "SSH"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "allow_web"
  }
}

Step 8:  Create a network Interface with an ip in the subnet that was created in step 4

resource "aws_network_interface" "web-server-nic" {
  subnet_id       = aws_subnet.subnet-1.id
  private_ips     = ["10.0.1.50"]
  security_groups = [aws_security_group.allow_web.id]

}

Step 9:  Assign an elastic IP to the network interface created in step 7

resource "aws_eip" "one" {
  vpc                       = true
  network_interface         = aws_network_interface.web-server-nic.id
  associate_with_private_ip = "10.0.1.50"
  depends_on                = [aws_internet_gateway.gw]
}

Step 10: Create an Ubuntu Server and install/enable apache2

resource "aws_instance" "web-server-instance" {
    ami           = "ami-052efd3df9dad4825"
    instance_type = "t2.micro"
    availability_zone = "us-east-1a"
    key_name = "main-key"

    network_interface {
      device_index = 0
        network_interface_id = aws_network_interface.web-server-nic.id
    }
    
    user_data = <<-EOF
                #! /bin/bash
                sudo apt update -y
                sudo apt install apache2 -y
                sudo systemctl start apache2
                sudo systemctl enable apache2
                sudo bash -c 'echo first web server > /var/www/html/index.html'
                EOF
    tags = {
        Name = "web-server"
    }
}

Step 11: Run Terraform Plan command 

Step 12: Run Terraform Apply Command.

Step 13: Confirm Status on AWS Console

![b](https://user-images.githubusercontent.com/52894481/189687058-c04403ed-3c08-4197-a0ac-abade59e3fac.JPG)

Step 14: Browse to website

![c](https://user-images.githubusercontent.com/52894481/189687234-b603ed2a-ef4e-4aa8-8578-58d841f7a9c5.JPG)
