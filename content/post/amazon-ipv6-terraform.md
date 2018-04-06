---
title: "Setting up IPv6 on AWS with Terraform"
date: 2017-09-06T23:10:08+02:00
categories:
tags:
- aws
- ipv6
- terraform
keywords:
- tech
#thumbnailImage: //example.com/image.jpg
---

In January 2017, [Amazon rolled out support for IPv6](https://aws.amazon.com/blogs/aws/aws-ipv6-update-global-support-spanning-15-regions-multiple-aws-services/) in a large number of regions. This means that you can now use IPv6 when you deploy your servers on ec2.

<!--more-->

 For me, the major advantage with IPv6-support in ec2 is that it is easier to communicate between ec2-instances in different regions, since each ec2-instance now has a public IPv6 address that can be reached from all other ec2-instances if you allow it.

IPv6 is not enabled on ec2 by default. Amazon has [documented](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-migrate-ipv6.html) how to set up IPv6 and it requires setting up a number of different resources to make it work. [Terraform](https://github.com/hashicorp/terraform) has support for IPv6 on Amazon since at least version 0.10, but I haven’t found any tutorial documenting how to put all the pieces together. This document shows a complete example of how to setup an ec2-instance in an Amazon region and enable it to communicate with the outside world via IPv6.

# Preparations

First, we set up a provider and a public key:

```
provider "aws" {
    alias = "eu-central-1"
    region = "eu-central-1"
}

resource "aws_key_pair" "deployer" {
    provider = "aws.eu-central-1"
    key_name   = "mattias"
    public_key = "ssh-rsa your-public-key-here me@example.com"
}
```

Since I want to provision to several aws-regions eventually, I define an alias to for aws that points to aws for a specific region. I can then create all other resources with the provider aws.eu-central-1 to place them in the correct region. Note that I have chosen to name all resources after the region that I create them in.

# Create VPC

Next step is to setup a VPC and a associate a subnet with it:

```
resource "aws_vpc" "eu-central-1" {
    provider = "aws.eu-central-1"
    enable_dns_support = true
    enable_dns_hostnames = true
    assign_generated_ipv6_cidr_block = true
    cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "eu-central-1" {
    provider = "aws.eu-central-1"
    vpc_id = "${aws_vpc.eu-central-1.id}"
    cidr_block = "${cidrsubnet(aws_vpc.eu-central-1.cidr_block, 4, 1)}"
    map_public_ip_on_launch = true

    ipv6_cidr_block = "${cidrsubnet(aws_vpc.eu-central-1.ipv6_cidr_block, 8, 1)}"
    assign_ipv6_address_on_creation = true
}
```

For the VPC, we ask Amazon to assign an IPv6 address block to our VPC. For the subnet, we then generate a cidr_block that contains a subset of these addresses for our subnet. [Amazon always assigns a /56 subnet to a vpc](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html#vpc-sizing-ipv6). The `cidrsubnet(prefix, 8, 1)` creates a /64 subnet by appending a byte with the value 1 to that prefix.

# Internet gateway

Next step is to create a gateway between our subnet and the internet. Since we want to let traffic flow between our ec2-instances and the outside world, we create an Internet gateway, setup routing rules to forward everything and then associate the routing-table to the subnet.

```
resource "aws_internet_gateway" "eu-central-1" {
    provider = "aws.eu-central-1"
    vpc_id = "${aws_vpc.eu-central-1.id}"
}

resource "aws_default_route_table" "eu-central-1" {
    provider = "aws.eu-central-1"
    default_route_table_id = "${aws_vpc.eu-central-1.default_route_table_id}"

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = "${aws_internet_gateway.eu-central-1.id}"
    }

    route {
        ipv6_cidr_block = "::/0"
        gateway_id = "${aws_internet_gateway.eu-central-1.id}"
    }
}

resource "aws_route_table_association" "eu-central-1" {
    provider = "aws.eu-central-1"
    subnet_id      = "${aws_subnet.eu-central-1.id}"
    route_table_id = "${aws_default_route_table.eu-central-1.id}"
}
```

# Security group

By default, Amazon assigns a security group to our VPC. To allow traffic to and from our instances, we have to open up the security group to allow IPv6 traffic.

```
resource "aws_security_group" "eu-central-1" {
    provider = "aws.eu-central-1"
    name = "terraform-example-instance"
    vpc_id = "${aws_vpc.eu-central-1.id}"
    ingress {
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        from_port = 0
        to_port = 0
        protocol = "-1"
        ipv6_cidr_blocks = ["::/0"]
    }

    egress {
      from_port = 0
      to_port = 0
      protocol = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }

    egress {
      from_port = 0
      to_port = 0
      protocol = "-1"
      ipv6_cidr_blocks = ["::/0"]
    }
}
```

These rules allow all IPv6 traffic to and from our instances and only ssh over IPv4 to the instances. You might want to have stricter firewall rules for IPv6, but that is a topic for another tutorial…

# Create instance

Finally, we setup an ec2-instance that uses the resources that we created:

```
resource "aws_instance" "eu-central-1" {
    provider = "aws.eu-central-1"
    ami = "ami-657bd20a" # Amazon linux
    key_name = "mattias"
    instance_type = "t2.micro"
    subnet_id = "${aws_subnet.eu-central-1.id}"
    ipv6_address_count = 1
    vpc_security_group_ids = ["${aws_security_group.eu-central-1.id}"]
    tags {
        Name = "my-ipv6-test"
    }
    depends_on = ["aws_internet_gateway.eu-central-1"]
}
```

The instance uses Amazon linux. Amazon linux has IPv6 enabled by default. If you want to use another AMI, make sure that you [configure it to get an IPv6 address via DHCP](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-migrate-ipv6.html#ipv6-dhcpv6-ubuntu).

# Print results

When you run your terraform file, you want to know the IP address of the created ec2-instance. You can print it with an output statement:

```
output "eu-central-1 public IPv4" {
  value = "${aws_instance.eu-central-1.public_ip}"
}

output "eu-central-1 IPv6" {
  value = ["${aws_instance.eu-central-1.ipv6_addresses}"]
}
```

Note that the ec2-instance can have several IPv6 addresses, but since we only asked for one address with `ipv6_address_count=1` when we created the instance, it will only have one.

# Testing

Now that you have a complete terraform file, run it with

```
terraform apply
```

It will run and print the assigned IPv6-address and the public IPv4. You can login to the instance as the ec2-user using the public IPv4 address and check that IPv6 works with ping6:

```
[ec2-user@ip-10-0-19-164 ~]$ ping6 -n ping.sunet.se
PING ping.sunet.se(2001:6b0:7::18) 56 data bytes
64 bytes from 2001:6b0:7::18: icmp_seq=1 ttl=56 time=23.8 ms
64 bytes from 2001:6b0:7::18: icmp_seq=2 ttl=56 time=23.0 ms
64 bytes from 2001:6b0:7::18: icmp_seq=3 ttl=56 time=23.1 ms
64 bytes from 2001:6b0:7::18: icmp_seq=4 ttl=56 time=23.1 ms
^C
```

To check that your ec2-instance can be reached from outside via IPv6 if you don’t have access to another system with IPv6, you can use the [Online ping IPv6](https://www.subnetonline.com/pages/ipv6-network-tools/online-ipv6-ping.php) tool.

# TL;DR

This is the complete terraform file for your copy/paste pleasure:

```
provider "aws" {
    alias = "eu-central-1"
    region = "eu-central-1"
}

resource "aws_key_pair" "deployer" {
    provider = "aws.eu-central-1"
    key_name   = "mattias"
    public_key = "ssh-rsa your-public-key-here me@example.com"
}

resource "aws_vpc" "eu-central-1" {
    provider = "aws.eu-central-1"
    enable_dns_support = true
    enable_dns_hostnames = true
    assign_generated_ipv6_cidr_block = true
    cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "eu-central-1" {
    provider = "aws.eu-central-1"
    vpc_id = "${aws_vpc.eu-central-1.id}"
    cidr_block = "${cidrsubnet(aws_vpc.eu-central-1.cidr_block, 4, 1)}"
    map_public_ip_on_launch = true

    ipv6_cidr_block = "${cidrsubnet(aws_vpc.eu-central-1.ipv6_cidr_block, 8, 1)}"
    assign_ipv6_address_on_creation = true
}

resource "aws_internet_gateway" "eu-central-1" {
    provider = "aws.eu-central-1"
    vpc_id = "${aws_vpc.eu-central-1.id}"
}

resource "aws_default_route_table" "eu-central-1" {
    provider = "aws.eu-central-1"
    default_route_table_id = "${aws_vpc.eu-central-1.default_route_table_id}"

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = "${aws_internet_gateway.eu-central-1.id}"
    }

    route {
        ipv6_cidr_block = "::/0"
        gateway_id = "${aws_internet_gateway.eu-central-1.id}"
    }
}

resource "aws_route_table_association" "eu-central-1" {
    provider = "aws.eu-central-1"
    subnet_id      = "${aws_subnet.eu-central-1.id}"
    route_table_id = "${aws_default_route_table.eu-central-1.id}"
}

resource "aws_security_group" "eu-central-1" {
    provider = "aws.eu-central-1"
    name = "terraform-example-instance"
    vpc_id = "${aws_vpc.eu-central-1.id}"
    ingress {
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        from_port = 0
        to_port = 0
        protocol = "-1"
        ipv6_cidr_blocks = ["::/0"]
    }

    egress {
      from_port = 0
      to_port = 0
      protocol = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }

    egress {
      from_port = 0
      to_port = 0
      protocol = "-1"
      ipv6_cidr_blocks = ["::/0"]
    }
}

resource "aws_instance" "eu-central-1" {
    provider = "aws.eu-central-1"
    ami = "ami-657bd20a" # Amazon linux
    key_name = "mattias"
    instance_type = "t2.micro"
    subnet_id = "${aws_subnet.eu-central-1.id}"
    ipv6_address_count = 1
    vpc_security_group_ids = ["${aws_security_group.eu-central-1.id}"]
    tags {
        Name = "my-ipv6-test"
    }
    depends_on = ["aws_internet_gateway.eu-central-1"]
}

output "eu-central-1 public IPv4" {
  value = "${aws_instance.eu-central-1.public_ip}"
}

output "eu-central-1 IPv6" {
  value = ["${aws_instance.eu-central-1.ipv6_addresses}"]
}
```
