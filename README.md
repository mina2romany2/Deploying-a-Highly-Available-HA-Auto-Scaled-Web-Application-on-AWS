# Deploying a Highly Available, Auto-Scaled, Load-Balanced Web Application on AWS

A hands-on AWS project demonstrating a production-style, highly available web application architecture built on a custom VPC — spanning two Availability Zones with redundant NAT Gateways, private EC2 instances, an Application Load Balancer, and an Auto Scaling Group.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [NAT Gateway Design](#nat-gateway-design)
- [IAM and AWS Systems Manager](#iam-and-aws-systems-manager)
- [EC2 Web/Application Servers](#ec2-webapplication-servers)
- [Security Groups](#security-groups)
- [AWS Systems Manager Validation](#aws-systems-manager-validation)
- [Application Load Balancer](#application-load-balancer)
- [Target Group](#target-group)
- [Auto Scaling Group](#auto-scaling-group)
- [Auto Scaling and High Availability Validation](#auto-scaling-and-high-availability-validation)
- [High Availability](#high-availability)
- [Security Considerations](#security-considerations)
- [Testing Checklist](#testing-checklist)
- [Recommended Evidence Screenshots](#recommended-evidence-screenshots)
- [Key AWS Concepts Demonstrated](#key-aws-concepts-demonstrated)
- [Architecture Decisions](#architecture-decisions)
- [Cost Considerations](#cost-considerations)
- [Conclusion](#conclusion)

## Architecture Overview

<img width="975" height="434" alt="image" src="https://github.com/user-attachments/assets/49ac71c1-d367-4e09-9558-f5784b31757b" />


The application infrastructure is distributed across `us-east-1a` and `us-east-1b`, with all compute resources placed in private subnets and only the Application Load Balancer exposed to the internet.

## NAT Gateway Design

Two NAT Gateways provide redundant outbound internet connectivity for the private subnets.

| NAT Gateway | AZ | Public Subnet | Elastic IP |
|---|---|---|---|
| `NAT Gateway A` | `us-east-1a` | `Public_Subnet1` | Assigned |
| `NAT Gateway B` | `us-east-1b` | `Public_Subnet2` | Assigned |

Each private subnet uses the NAT Gateway located in the same Availability Zone.

**Private Route Tables**

`Private_RT_1`
```text
Destination: 0.0.0.0/0
Target: NAT Gateway A
```

`Private_RT_2`
```text
Destination: 0.0.0.0/0
Target: NAT Gateway B
```

This design avoids relying on a single NAT Gateway and provides better AZ-level resilience.

## IAM and AWS Systems Manager

An IAM role named `ec2tossm` was created for the EC2 instances, providing the permissions required for AWS Systems Manager Session Manager.

The EC2 instances use this role so they can be administered without:

- SSH keys
- Public IP addresses
- Direct inbound SSH access

This allows the web servers to remain in private subnets while still being administratively accessible through SSM.

## EC2 Web/Application Servers

Two initial EBS-backed EC2 instances were deployed:

- `Web_Server1`
- `Web_Server2`

**Configuration**
```text
AMI: Amazon Linux 2023
Instance Type: t2.micro
Subnet: Private subnet
Access: AWS Systems Manager Session Manager
Security Group: WebSG
```

The instances were deployed in separate Availability Zones.

**User Data — Web Server 1**
```bash
#!/bin/bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "This is server 1 in AWS Region US-EAST-1 in AZ US-EAST-1A" > /var/www/html/index.html
```

**User Data — Web Server 2**
```bash
#!/bin/bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "This is server 2 in AWS Region US-EAST-1 in AZ US-EAST-1B" > /var/www/html/index.html
```

Apache HTTP Server is installed and configured to start automatically on boot.

## Security Groups

Two security groups were created.

### ALBSG

Associated with the Application Load Balancer.

**Inbound**
```text
HTTP (80) from 0.0.0.0/0
```
Allows internet users to reach the application through the ALB.

**Outbound**

Outbound traffic is allowed as required for the load balancer to communicate with the application tier.

### WebSG

Associated with the EC2 web/application instances.

During initial validation, HTTP access was open for testing. After the ALB was validated, the security group was hardened:

```text
HTTP (80)
Source: ALBSG
```

EC2 instances now accept HTTP traffic only from the ALB — a security-group-to-security-group trust relationship that follows the principle of least privilege.

## AWS Systems Manager Validation

After launching the instances, SSM Session Manager was used to connect to each private EC2 instance.

Check the Apache service:
```bash
sudo systemctl status httpd
```
Expected: active/running.

Validate application connectivity:
```bash
curl http://<INSTANCE_PRIVATE_IP>
```
Expected: the response configured in the instance's `index.html`.

## Application Load Balancer

An internet-facing ALB named `WebALB` was deployed across:

- `Public_Subnet1` in `us-east-1a`
- `Public_Subnet2` in `us-east-1b`

**Listener**
```text
HTTP :80
```

The ALB uses `ALBSG` as its security group.

## Target Group

A target group named `WebTG` was created.

```text
Protocol: HTTP
Port: 80
Health Check Protocol: HTTP
```

The initial EC2 instances were registered as targets. Only healthy targets receive traffic from the load balancer.

## Auto Scaling Group

A **Launch Template** standardizes new instance configuration:

```text
AMI: Amazon Linux 2023
Instance Type: t2.micro
Security Group: WebSG
Subnet Placement: Private subnets
```

**Launch Template User Data**
```bash
#!/bin/bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "This is an app server in AWS Region US-EAST-1" > /var/www/html/index.html
```

**ASG Configuration** — name: `ASG`

```text
Minimum: 2
Desired: 4
Maximum: 6
```

The ASG uses private subnets in both AZs, and the existing `WebTG` target group was attached with ALB health checks enabled.

## Auto Scaling and High Availability Validation

1. New instances launched according to desired capacity and were distributed across both AZs.
2. The original manually created EC2 instances were terminated.
3. The ASG maintained required capacity and registered replacement instances with the target group.
4. Once instances passed target group health checks, the application was tested through the ALB DNS hostname.

**Expected response**
```text
This is an app server in AWS Region US-EAST-1
```

## High Availability

| Layer | How it contributes |
|---|---|
| Multi-AZ VPC Design | Infrastructure spans `us-east-1a` and `us-east-1b` |
| Multiple Application Instances | Several EC2 instances run behind the ALB |
| Application Load Balancer | Distributes requests across healthy instances |
| Target Group Health Checks | Detects and removes unhealthy instances from traffic |
| Auto Scaling Group | Maintains desired capacity, replaces failed instances |
| Multiple NAT Gateways | Each AZ has its own NAT Gateway — no single point of dependency |

## Security Considerations

A layered security model:

- **Internet layer** — only the ALB is exposed, via HTTP port 80.
- **Application layer** — EC2 instances sit in private subnets with no public IPs.
- **Security group layer** — `WebSG` allows HTTP only from `ALBSG`.
- **Administrative access** — SSM Session Manager is used instead of exposing SSH port 22.

```
Internet → ALBSG → WebSG → Private EC2 Instances
```

This prevents direct internet traffic from reaching the application instances.

## Testing Checklist

- [x] VPC created with `10.0.0.0/16`
- [x] DNS hostnames enabled
- [x] Two public subnets created
- [x] Two private subnets created
- [x] Internet Gateway attached
- [x] Public route table configured
- [x] Two NAT Gateways created
- [x] Private route tables configured
- [x] IAM role created for SSM
- [x] EC2 instances launched in private subnets
- [x] Apache installed and running
- [x] EC2 instances accessed through SSM
- [x] Private IP connectivity tested
- [x] Target Group created
- [x] ALB created across two Availability Zones
- [x] ALB health checks validated
- [x] Launch Template created
- [x] Auto Scaling Group created
- [x] ALB Target Group integrated with ASG
- [x] Desired capacity configured to 4
- [x] Minimum capacity configured to 2
- [x] Maximum capacity configured to 6
- [x] Original EC2 instances terminated after ASG deployment
- [x] Replacement instances registered with the Target Group
- [x] Targets reached healthy status
- [x] Application tested through the ALB DNS name
- [x] WebSG restricted to traffic from ALBSG

## Recommended Evidence Screenshots

1. VPC details showing the custom CIDR
2. Subnets showing all four subnets and their AZs
3. Public route table showing the Internet Gateway route
4. Private route tables showing NAT Gateway routes
5. NAT Gateways and their AZs
6. EC2 instances showing private subnet placement
7. IAM role `ec2tossm`
8. SSM Session Manager connection
9. Apache service status
10. Target Group showing healthy targets
11. ALB configuration and listeners
12. Auto Scaling Group showing min/desired/max capacity
13. EC2 instances created by the ASG across both AZs
14. Browser showing the application through the ALB DNS hostname
15. Final WebSG and ALBSG security-group rules

> **Note:** Apply the final, locked-down security-group configuration only after validating ALB-to-instance communication — the initial direct-instance testing path and the final architecture use different traffic paths.

## Key AWS Concepts Demonstrated

This project demonstrates practical understanding of several AWS Solutions Architect Associate concepts:

- Amazon VPC & CIDR addressing
- Public and private subnets
- Availability Zones
- Internet Gateway & NAT Gateway
- Route Tables
- Amazon EC2 & EBS-backed instances
- IAM Roles
- AWS Systems Manager
- Application Load Balancer
- Target Groups & Health Checks
- Launch Templates & Auto Scaling Groups
- High Availability, Fault Tolerance, Elasticity
- Security Groups
- Defense in depth & least-privilege network access

## Architecture Decisions

**Why two Availability Zones?**
Reduces the impact of a single-AZ failure and lets the load balancer keep routing requests to healthy resources in another AZ.

**Why private subnets for EC2?**
The application instances don't need to accept direct internet traffic, so keeping them private reduces the attack surface and forces external traffic through the ALB.

**Why an Application Load Balancer?**
Provides Layer 7 HTTP load balancing, health checks, and traffic distribution across multiple instances.

**Why an Auto Scaling Group?**
Maintains required application capacity and launches replacement instances when existing ones become unhealthy or are terminated.

**Why two NAT Gateways?**
One NAT Gateway per AZ avoids making outbound connectivity dependent on a single AZ.

**Why SSM instead of SSH?**
Allows administrative access to private EC2 instances without exposing SSH port 22 or assigning public IP addresses.

## Cost Considerations

This architecture is highly available, but HA introduces additional cost, mainly from:

- Two NAT Gateways
- Multiple EC2 instances
- Application Load Balancer
- EBS storage
- Elastic IP addresses associated with NAT Gateways

For production, use AWS Cost Explorer and the AWS Pricing Calculator to estimate and monitor costs, and remove unused resources after testing to avoid unnecessary charges.

## Conclusion

This project demonstrates the design and deployment of a highly available and scalable web application on AWS using a custom VPC. It follows AWS best practices by separating public and private resources, distributing infrastructure across Availability Zones, using an Application Load Balancer for traffic distribution, using an Auto Scaling Group for elasticity and instance replacement, restricting application access through security groups, and using AWS Systems Manager instead of exposed SSH access.

The result is a resilient AWS architecture that practically demonstrates the networking, compute, load balancing, scaling, security, and high-availability concepts covered by the AWS Solutions Architect Associate certification.

---

**Author:** Mina Romany Nasr
**Project:** Deploying a Highly Available, Auto-Scaled, Load-Balanced Web Application on AWS
