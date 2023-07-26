provider "aws" {
  region     = "ap-south-1"
  access_key = 
  secret_key = 
}


# going to create vpc

resource "aws_vpc" "myvpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "myvpc"
  }
}
#this is public subnet
resource "aws_subnet" "publicsubnet" {
  vpc_id     = aws_vpc.myvpc.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "public subnet"
  }
}

#this is private subnet

resource "aws_subnet" "privatesubnet" {
  vpc_id     = aws_vpc.myvpc.id
  cidr_block = "10.0.2.0/24"

  tags = {
    Name = "private subnet"
  }
}

#this is security group

resource "aws_security_group" "mysg" {
  name        = "mysg"
  vpc_id      = aws_vpc.myvpc.id

  ingress {
    description      = "TLS from VPC"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

 ingress {
    description      = "TLS from VPC"
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
  }

  tags = {
    Name = "mysg"
  }
}

#internet gatway

resource "aws_internet_gateway" "myigw" {
  vpc_id = aws_vpc.myvpc.id

  tags = {
    Name = "myigw"
  }
}

#elastic ip for nat

resource "aws_eip" "eipfornat" {
  vpc   = true
}

#eip for instance

resource "aws_eip" "eipforinstance" {
  instance = aws_instance.web.id
  vpc   = true
}

# nat gatway

resource "aws_nat_gateway" "mynat" {
  allocation_id = aws_eip.eipfornat.id
  subnet_id     = aws_subnet.publicsubnet.id

  tags = {
    Name = "mynat"
  }
}

#route table 1

resource "aws_route_table" "publicrt" {
  vpc_id = aws_vpc.myvpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.myigw.id
  }

  tags = {
    Name = "public route table"
  }
}

#private route table


resource "aws_route_table" "privatert" {
  vpc_id = aws_vpc.myvpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.mynat.id
  }

  tags = {
    Name = "private route table"
  }
}

#route table association

resource "aws_route_table_association" "myassociation" {
  subnet_id      = aws_subnet.publicsubnet.id
  route_table_id = aws_route_table.publicrt.id
}

resource "aws_route_table_association" "newassociation" {
  subnet_id      = aws_subnet.privatesubnet.id
  route_table_id = aws_route_table.privatert.id
}


resource "aws_key_pair" "project" {
  key_name   = "project"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCRirauAQgKrpgtvgoPGsigH9HP6Ox3TSWVgp4NpxL8PheAh8C43sJzPKw41EnlGyy7JvTntAh4Ia682MBoATD/ZKs3t6pz+8+bT1aepnhKqCdXwujsrwhWtOL9KgVrCF7ZLKvToMeAzmDk/cj4MipoGzspK9uKp93PMgbu61s76boraUQwIpd1uvbvDVAk/TXaxpXYzp243v4GgI2bjkjdSZTp7tINfh8qn3hrOAgD2/Z611iNqKIPAMgLzoMY3SRT2s86PZMgEMUdzR/JNhVAC8FE9XLRK2Lrczk5KaewuubDSXRf1cUdtZ2gGnIRagXlXeucPx6QS1It0ZejlCOAAVgUokxQqbn0L8XDqPP6v2pI+i7t5HOsTfMbzLIH5Vd8I7kzhJt+YB54rc/EHPRSO1a4hOkmNeOFP8j8YxmJtmFXOEfZQj3wfTz+8wl4v/WD8+UnG2K+7tr3m9g7w7JiJwOSZA5wSeA88uvbkXhX5oEuiIXgwdi3hD70dHVi8QM= root@ip-172-31-37-86.ap-south-1.compute.internal"
}


resource "aws_instance" "web" {
  ami           = "ami-008b85aa3ff5c1b02"
  instance_type = "t2.micro"
  subnet_id = aws_subnet.publicsubnet.id
  vpc_security_group_ids = [aws_security_group.mysg.id]
  key_name = "project"

  tags = {
    Name = "web-server"
  }
}

resource "aws_instance" "db" {
  ami           = "ami-008b85aa3ff5c1b02"
  instance_type = "t2.micro"
  subnet_id = aws_subnet.privatesubnet.id
  vpc_security_group_ids = [aws_security_group.mysg.id]
  key_name = "project"

  tags = {
    Name = "db-server"
  }
}
