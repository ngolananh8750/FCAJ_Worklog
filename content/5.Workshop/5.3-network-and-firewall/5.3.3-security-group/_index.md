---
title: "Configure Security Groups"
date: 2026-10-30
weight: 3
chapter: false
pre: "<b>5.3.3. </b>"
---


## Objective

Create **sg-secure-web** in **vpc-secure-web** to control connections to the document storage web server. Member 1 configures it in the AWS Management Console, then hands the Security Group ID to Member 2 for EC2 deployment in section 5.4.1.

A security group applies to associated resources, rather than directly to a subnet. Security groups are stateful: response traffic for an allowed connection does not require a separate rule in the opposite direction. Therefore, do not open ephemeral ports in the security group merely to receive responses; Network ACL requirements are covered separately in 5.3.4.

## Prerequisites

- Complete [5.3.2. Create an Internet Gateway and Route Table](../5.3.2-gateway-and-route-table/).
- Select **Singapore — ap-southeast-1** and verify the VPC ID of **vpc-secure-web**.
- Use an account authorized to create security groups, manage inbound/outbound rules, and apply tags as described in 5.2.1.
- Identify the public IPv4 address of the administrator's SSH connection. Do not use a LAN address such as 192.168.x.x.
- If sg-secure-web already exists, verify its VPC ID and rules before reusing it; do not create a duplicate.

| Property | Value |
| --- | --- |
| Security group name | sg-secure-web |
| Description | Security group for workshop document web server |
| VPC | vpc-secure-web — 10.0.0.0/16 |
| Name tag | sg-secure-web |
| Other tags | Project=SecureWebWorkshop |

## Step 1: Define access rules

Use the following inbound configuration for the workshop website:

| Type | Protocol | Port range | Source | Purpose |
| --- | --- | --- | --- | --- |
| SSH | TCP | 22 | ADMIN_PUBLIC_IP/32 | Administer EC2 from an authorized computer |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Allow users to access the website over HTTPS |
| HTTP | TCP | 80 | 0.0.0.0/0 | Initial Nginx page, then HTTPS redirect |

**ADMIN_PUBLIC_IP/32 is a placeholder: replace it with the actual public IPv4 address followed by /32.** Do not enter the placeholder literally in the console. Administrators connecting from different Internet addresses require separate SSH /32 rules.

The HTTPS rule permits connections to the server; the application must still enforce login and document access checks. Opening port 443 does not install a certificate or enable HTTPS automatically.

Allow 80 for Nginx testing and HTTPS redirection. Restrict SSH to the administrator IPv4 /32 and leave 81 and 8000 outside the allowed ports.

## Step 2: Create the security group

1. Open **VPC → Security groups → Create security group**.
2. Enter the name and Description from the prerequisites table.
3. Under **VPC**, select **vpc-secure-web** using its verified ID; remove the default VPC selection if necessary.
4. Add the Name, Project, Environment, and Owner tags.
5. Initially leave inbound rules empty and retain the default outbound rule; select **Create security group**. Add inbound rules immediately in the next step.
6. Record the Security Group ID, which starts with `sg-`.

![](/FCAJ_Worklog/images/005/sg1.png)

## Step 3: Configure inbound rules

1. Select **sg-secure-web → Inbound rules → Edit inbound rules**.
2. Select **Add rule**, Type **SSH**, TCP **22**.
3. Select Source **My IP** if you are using the same computer and network path that will be used for SSH. Verify the populated /32 CIDR. When configuring access for another member, select **Custom** and enter that member's public IPv4/32.
4. Enter Description: `SSH from administrator public IP`.
5. Add Type **HTTPS**, TCP **443**, Source **Anywhere-IPv4 — 0.0.0.0/0**; Description: `HTTPS document portal`.
6. Confirm there are no broad SSH rules or unnecessary rules, then select **Save rules**.
7. Review the Inbound rules tab after saving.

If the browser and SSH use different VPNs or proxies, My IP might not match the SSH source address. Verify the Internet address used by the SSH connection. When the administrator's IP changes, update the /32 rule and remove any old address that is no longer used.

**📸 Screenshot 02 — Capture before saving:** Edit inbound rules showing SSH from a /32 source and HTTPS from 0.0.0.0/0.  
**Filename:** 02-inbound-rules-edit.png.

![](/FCAJ_Worklog/images/005/sg2.png)

![](/FCAJ_Worklog/images/005/sg4.png)


## Step 4: Verify outbound rules

This workshop retains the default IPv4 outbound rule so EC2 can download installation packages and connect to S3, CloudWatch, and other required services:

| Type | Protocol | Port range | Destination |
| --- | --- | --- | --- |
| All traffic | All | All | 0.0.0.0/0 |

1. Open the **Outbound rules** tab of sg-secure-web.
2. Check the rule against the table. Retain it if already correct.
3. If missing, select **Edit outbound rules → Add rule**, enter the values above, and select **Save rules**.

This is a simplified lab choice and does not tightly restrict outgoing traffic. Allowing outbound traffic does not open additional inbound ports or grant S3 read/write permissions. Data permissions are controlled by the IAM role and bucket policy. The S3 Gateway Endpoint is created in 5.3.5; security groups do not replace routing configuration.

![](/FCAJ_Worklog/images/005/sg3.png)


## Step 5: Verify and hand over

| Item | Expected result |
| --- | --- |
| Name and VPC | sg-secure-web belongs to vpc-secure-web |
| SSH | TCP 22 only from administrators' public IPv4/32 addresses |
| HTTPS | TCP 443 from 0.0.0.0/0 |
| HTTP | TCP 80 |
| Backend ports | No inbound Internet access |
| Outbound | All traffic to 0.0.0.0/0 for the lab scope |
| Handover information | Region, VPC ID, Security Group ID, and agreed SSH sources |

Record the actual IDs in the handover table. In **5.4.1**, Member 2 selects **Select existing security group → sg-secure-web** when launching **workshop-web-01** in subnet-web-public. Do not let the launch wizard create a different group with broad SSH access. If multiple security groups are attached, their allow rules are combined; another group may unintentionally broaden access.

Do not test SSH or HTTPS before EC2 and the web service exist. After deployment, section 5.7 covers SSH from an authorized source, SSH from an unauthorized source, and HTTPS access. When troubleshooting, also check the public IPv4 address, routes, NACL, and listening services.

## Common issues

| Symptom | Resolution |
| --- | --- |
| vpc-secure-web is not listed | Check the Region and permissions to view VPCs |
| Security group name already exists | Verify its ID/VPC and reuse the correct workshop group |
| My IP differs from the administrator's IP | Check VPN/proxy routing and enter the SSH connection's correct public IPv4/32 |
| SSH stops working after changing networks | Update the /32 source; do not allow SSH from the entire Internet |
| Duplicate rule error | Keep the existing correct rule instead of adding a duplicate |
| AccessDenied when creating or editing | Record the denied action and ask the administrator to review policies from 5.2.1 |
| An unexpected port remains accessible later | Check every security group attached to the EC2 network interface |



Continue to [5.3.4. Configure Network ACLs](../5.3.4-network-acl/).

## References

- [Create a security group](https://docs.aws.amazon.com/vpc/latest/userguide/creating-security-groups.html)
- [Configure security group rules](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-security-group-rules.html)
- [Security group rules](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)
- [Security group basics](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
