

# Creating VPC through terraform


## Features
- Easy to customise and use.
- Each subnet CIDR block created through automation.
- Using tfvars file to access and modify variables.
- Project name is appended to the resources that are creating.

## Prerequisites for this project

![](https://i.ibb.co/swzJJrn/vpc.png)
- Need AWS CLI access or IAM user access with attached policies for the creation of VPC.
- Terraform need to be installed in your system.
- Knowledge to the working principles of each AWS services especially VPC, EC2 and IP Subnetting.

## Installation

If you need to download terraform , then click here [Terraform](https://www.terraform.io/downloads.html) .


Lets create a file for declaring the variables.This is used to declare the variable and the values are passing through the terrafrom.tfvars file.

### Create a variables.tf file
```sh
variable "region" {}
variable "access_key" {}
variable "secret_key" {}
variable "vpc_cidr" {}
variable "project" {}
```
### Create a provider.tf file 
```sh
provider "aws" {
  region     = var.region
  access_key = var.access_key
  secret_key = var.secret_key
}
```
### Create a terraform.tfvars
By default terraform.tfvars will load the variables to the the reources.
You can modify accordingly as per your requirements.

```sh
region = "put-your-region-here"
access_key = "put-your-access_key-here"
secret_key = "put-your-secret_key-here"
vpc_cidr = "X.X.X.X/16"
project = "name-of-your-project"
```
Go to the directory that you wish to save your tfstate files.Then Initialize the working directory containing Terraform configuration files using below command.
```sh
terraform init
```
#### Lets start creating main.tf file with the details below.


> To create VPC
```sh
resource "aws_vpc" "main" {

   cidr_block = var.vpc_cidr
   
   instance_tenancy = "default"

    enable_dns_support = true

        enable_dns_hostnames = true

    tags = {
        Name = var.project
    }

}
```
> To Gather All Subnet Name
```sh
data "aws_availability_zones" "available" {
  state = "available"
}
```

> To create InterGateWay For VPC
```sh
resource "aws_internet_gateway" "igw" {

  vpc_id = aws_vpc.main.id

  tags = {
    Name = var.project
  }
}
```
Here in this infrastructre we shall create 3 public and 3 private subnets in the region.This sample was meant for regions having 6 availability zone. I have used "us-east-1". Choose your region and modify according to the availability of the AZ. Also we have already provided the CIDR block in our terraform.tfvars you dont need to calculate the subnets, here we use terraform to automate the subnetting in /19.

> Creating public1, public2, public3 Subnet using count 
```sh
resource "aws_subnet" "public" {

  count = 3
 
  vpc_id     = aws_vpc.main.id

  cidr_block = cidrsubnet(var.vpc_cidr, 3, count.index)

  availability_zone = data.aws_availability_zones.subnet.names[count.index]

  map_public_ip_on_launch = true


tags = {
    Name = "${var.project}-public${count.index+1}"

  }
}
```
> Creating private1, private2, private3 Subnet using count 
```sh

resource "aws_subnet" "private" {

  count = 3
 
  vpc_id     = aws_vpc.main.id

  cidr_block = cidrsubnet(var.vpc_cidr, 3, "${count.index+3}")

  availability_zone = data.aws_availability_zones.subnet.names[count.index]

  map_public_ip_on_launch = false


tags = {
    Name = "${var.project}-private${count.index+1}"

  }
}
```
> Creating  Elastic IP For Nat Gateway
```sh
resource "aws_eip" "eip" {
  vpc      = true
  tags     = {
    Name = "${var.project}-eip"
  }
}
```
> Attaching Elastic IP to NAT gateway
```sh
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.eip.id
  subnet_id     = aws_subnet.public1.id
  tags = {
    Name = "${var.project}-nat"
  }
}
```
>  Creating Public Route Table
```sh
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

   route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

   tags = {
    Name = "${var.project}-public"
  }
}
```
>  Creating Private Route Table
```sh
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }

   tags = {
    Name = "${var.project}-private"
  }
}
```
> Creating Public Route Table Association
```sh
resource "aws_route_table_association" "public" {

  count =3 
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id

  depends_on = [
    aws_subnet.public ]
}
```
> Creating Private Route Table Association
```sh
resource "aws_route_table_association" "private" {

  count =3 
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id

  depends_on = [
    aws_subnet.private ]
}
````
### Create an output.tf for getting  terrafrom output.
```sh
output "aws_eip" {
value = aws_eip.eip.public_ip
}
output "aws_vpc" {
value = aws_vpc.main.id
}
output "aws_internet_gateway" {
value = aws_internet_gateway.igw.id
}
output "aws_nat_gateway" {
value = aws_nat_gateway.nat.id
}
output "aws_route_table_public" {
value = aws_route_table.public.id
}
output "aws_route_table_private" {
value = aws_route_table.private.id
}
```
#### Lets validate the terraform files using
```sh
terraform validate
```
#### Lets plan the architecture and verify once again.
```sh
terraform plan
```
#### Lets apply the above architecture to the AWS.
```sh
terraform apply
```
