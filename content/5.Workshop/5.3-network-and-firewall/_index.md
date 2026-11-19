---
title: "Network Setup and Firewall Configuration"
date: 2026-10-29
weight: 3
chapter: false
pre: "<b>5.3. </b>"
---

## Overview

In this section, the team builds the network environment for the **website with login functionality for uploading, storing, and downloading documents**. Amazon VPC provides a private network on AWS; the public subnet will host EC2 in Section 5.4. An internet gateway and route table provide internet connectivity, while a security group and network ACL control traffic.

**Member 1** leads this work and hands over network information to the other members. The team shares one VPC using the settings agreed upon in Section 5.2.2.

In this workshop, “Firewall” refers to traffic control using **security groups and network ACLs**; no additional AWS Network Firewall service is deployed. This is an overview and navigation page; Console procedures are provided in Sections 5.3.1–5.3.5.

## Objectives

- Create a VPC and one public subnet for the web server.
- Attach an internet gateway to the VPC, create a route table, and associate it with the subnet.
- Prepare an EC2 security group that restricts SSH and permits the required web access.
- Configure a network ACL to demonstrate an additional subnet-level control.
- Create an S3 gateway VPC endpoint so that EC2 accesses S3 in the same Region through the endpoint route.
- Record resource IDs and configurations for deployment and testing in later sections.

## Prerequisites

1. Complete Section 5.2 and sign in with the network IAM user that has permissions to create and configure resources.
2. Select **Asia Pacific (Singapore) — ap-southeast-1**.
3. Prepare the administrator computer's public IP for SSH, expressed as **IP/32**.
4. Check the handover table to avoid creating duplicate resources. If suitable resources already exist, verify their configuration and reuse them.
5. Agree on the planned bucket name with Member 3. The bucket and data access permissions are deployed in Section 5.5; there is no need to create a bucket just to start the networking section.

For missing-permission errors, record the action and error message so that the administrator can adjust the policy. Being able to open the VPC page does not mean that VPC creation is permitted.

## Deployment Settings

| Component | Name or value |
| --- | --- |
| Region | ap-southeast-1 |
| VPC | vpc-secure-web — 10.0.0.0/16 |
| Public subnet | subnet-web-public — 10.0.0.0/24 |
| Availability Zone | ap-southeast-1a if available; record the actual AZ |
| DNS resolution and DNS hostnames | Enabled |
| Internet gateway | igw-secure-web |
| Route table | rtb-web-public |
| Internet route | 0.0.0.0/0 → igw-secure-web |
| Security group | sg-secure-web |
| Network ACL | nacl-web-public |
| S3 gateway VPC endpoint | s3-gateway-endpoint |
| Endpoint service | com.amazonaws.ap-southeast-1.s3 |
| Shared tags | Project=SecureWebWorkshop |

Resource names do not replace AWS-assigned IDs. When configuring associations, always verify the correct VPC ID, Subnet ID, and Route Table ID.

## Connectivity and Security Principles

| Component | Application in this workshop |
| --- | --- |
| Public subnet | Has a route to an internet gateway. EC2 also needs a public IPv4 address, appropriate network rules, and a running service to accept internet connections over IPv4. |
| Security group | Associated with the EC2 network interface when launching the instance in Section 5.4; supports allow rules only and tracks connection state automatically. |
| Network ACL | Associated with the subnet; supports allow/deny rules, evaluates rule numbers in order, and requires explicit return-traffic rules because it is stateless. |
| S3 gateway VPC endpoint | Associated with the public subnet's route table; does not grant S3 read/write access by itself or replace an IAM role or bucket policy. |

**Planned traffic requirements:**

- **SSH — TCP 22:** only from the administrator IP /32. For multiple administrators, use separate /32 sources.
- **HTTPS — TCP 443:** allow internet users to access the website; HTTPS is configured in Section 5.4.4.
- **HTTP — TCP 80:** test Nginx and redirect to HTTPS.
- **Outbound traffic:** support software installation, S3 access, and CloudWatch log delivery; specific rules are provided in the configuration sections.
- **Return traffic:** account for it in the network ACL, including the required ephemeral ports.

Do not open SSH to 0.0.0.0/0. Do not copy security group rules directly into a network ACL while ignoring return traffic. For a custom network ACL, prepare the necessary rules before associating it with the subnet to avoid losing connectivity.

S3 is a service outside the subnet. In this architecture, traffic to S3 in the same Region uses the gateway endpoint; internet traffic and CloudWatch connections continue to use the internet gateway according to the allowed configuration. Do not create a NAT gateway or interface endpoint within this section's scope.

## Implementation Sequence

| Section | Task | Handover output |
| --- | --- | --- |
| 5.3.1 | Create a VPC and public subnet | VPC ID, Subnet ID, CIDRs, and AZ |
| 5.3.2 | Create an internet gateway and route table | Internet Gateway ID, Route Table ID, internet route, and subnet association |
| 5.3.3 | Configure a security group | Security Group ID and agreed rules |
| 5.3.4 | Configure a network ACL | Network ACL ID, rules for both directions, and subnet association |
| 5.3.5 | Create an S3 gateway VPC endpoint | VPC Endpoint ID and associated route table |

Follow this order. EC2 is created in Section 5.4; testing S3 reads and writes requires the bucket and IAM role in Section 5.5. Before EC2 or the bucket exists, record network configuration checks only; do not claim that the website or document access is already working.

## Expected Outcomes

- The VPC and subnet have the correct CIDRs, Region, and DNS settings.
- The internet gateway is attached to the VPC; the route table has an internet route and is associated with the correct subnet.
- The security group is ready for Member 2 to attach to EC2.
- The network ACL has appropriate rules and is associated with the correct subnet.
- The S3 gateway VPC endpoint has Available status and is associated with the correct route table.
- The handover table contains actual IDs, and the information has been shared with Members 2, 3, and 4.



Detailed procedure and result screenshots belong in each child section. Full connectivity and allow/deny testing takes place in Section 5.7 after the dependent resources are available.

## Contents

1. [5.3.1. Create a VPC and Public Subnet](./5.3.1-vpc-and-subnet/)
2. [5.3.2. Create an Internet Gateway and Route Table](./5.3.2-gateway-and-route-table/)
3. [5.3.3. Configure a Security Group](./5.3.3-security-group/)
4. [5.3.4. Configure a Network ACL](./5.3.4-network-acl/)
5. [5.3.5. Create an S3 Gateway VPC Endpoint](./5.3.5-s3-endpoint/)
- [5.3.6. Flow Logs and ports 80/81](./5.3.6-flow-logs/)

## References

- [Network controls with security groups and network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html)
- [Add internet access to a subnet](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-igw.html)
- [S3 gateway VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
