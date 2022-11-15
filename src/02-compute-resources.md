# Compute resources

## Network resources

### Creating subnets

In the Management Console, go to `VPC` > `Subnets`, and click `Create subnet`.
Select a VPC ID. This will display the Associated VPC CIDRs.
Name the subnet `sigstore-the-hard-way` and choose an IPv4 CIDR block corresponding to the VPC IPv4 CIDR with a subnet mask `/24` (for example, `172.31.50.0/24` if the VPC IPv4 CIDR is `172.31.0.0/16`).

### Security groups

Create a security group that will allow you to have control over the inbound and outbond traffic in your instances.
In the Management Console, go to the `Network & Security` section in the EC2 navigation panel and select `Security Groups`.
In the top-right corner, select `Create security group`.

Choose a security group name, a description and eventually specify a VPC.

Create the following inbound traffic rules:
```
- Type: Custom TCP
- Protocol: TCP
- Port range: 22
- Source: Anywhere-IPv4 (0.0.0.0/0)
```
```
- Type: Custom TCP
- Protocol: TCP
- Port range: 80
- Source: Anywhere-IPv4 (0.0.0.0/0)
```
```
- Type: Custom TCP
- Protocol: TCP
- Port range: 443
- Source: Anywhere-IPv4 (0.0.0.0/0)
```
```
- Type: All ICMP - IPv4
- Protocol: ICMP
- Port range: All
- Source: Anywhere-IPv4 (0.0.0.0/0)
```
```
- Type: All TCP
- Protocol: TCP
- Port range: 0 - 65535
- Source: 172.31.50.0/24
```
```
- Type: All ICMP - IPv4
- Protocol: ICMP
- Port range: All
- Source: 172.31.50.0/24
```
```
- Type: All UDP
- Protocol: UDP
- Port range: 0 - 65535
- Source: 172.31.50.0/24
```

## Compute resources

In this part, we will create four EC2 instances from Ubuntu Amazon Machine Images.
In the Management Console, go to `EC2` > `Instances` > `Launch an instance`.
Select the `Ubuntu` AMI in the `Application and OS Images (Amazon Machine Image)` section with a `64-bit (x86)` archutecture.
Choose a `t3.small` instance type. This type of machine has a capacity of 2 vCPU and 2 GiB memory, and is available for 0.0228 USD an hour (Linux pricing).

Select the key pair you created in the first section in `Key pair (login)` and the `sigstore-the-hard-way` firewall created in the last section in `Network settings`.
Configure a storage of 200 GiB general purpose SSD (gp2). In the top-right, select a number of 4 instances and click `Launch instances`.