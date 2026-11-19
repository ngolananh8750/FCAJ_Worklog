---
title: "Configure Network ACLs"
date: 2026-10-30
weight: 4
chapter: false
pre: "<b>5.3.4. </b>"
---



## Objective

Create **nacl-web-public** for **subnet-web-public**, adding subnet-level traffic filtering for the document portal. Member 1 performs this section.

Network ACLs (NACLs) are stateless: requests and responses must both be permitted. Rules are evaluated from the lowest number upward, stopping at the first match. NACLs support ALLOW and DENY; the final `*` rule denies unmatched traffic.

## Prerequisites

- Complete [5.3.3. Configure Security Groups](../5.3.3-security-group/).
- Region: **Singapore — ap-southeast-1**.
- VPC: **vpc-secure-web — 10.0.0.0/16**; subnet: **subnet-web-public — 10.0.0.0/24**.
- Obtain permissions to create NACLs, manage rules, change subnet associations, and apply tags.
- Record the subnet's current NACL ID under **VPC → Subnets → subnet-web-public → Network ACL** for rollback.
- Prepare the administrator's public IPv4/32, matching the SSH source in sg-secure-web.

**Associate the new NACL only after configuring both directions.** A new custom NACL blocks all traffic; associating it immediately may disconnect resources in the subnet.

## Step 1: Create the network ACL

1. Open **VPC → Network ACLs → Create network ACL**.
2. Enter **Name: nacl-web-public** and select the VPC ID of **vpc-secure-web**.
3. Add tags: **Project=SecureWebWorkshop**.
4. Select **Create network ACL** and record its `acl-...` ID.
5. Do not associate a subnet yet. If the NACL already exists, verify its ID/VPC and inspect it before reuse.

![](/FCAJ_Worklog/images/005/nacl1.png)


## Step 2: Configure inbound rules

Select **nacl-web-public → Inbound rules → Edit inbound rules → Add new rule**, enter each row, and select **Save changes**.

| Rule number | Type | Protocol | Port range | Source | Allow/Deny |
| --- | --- | --- | --- | --- | --- |
| 100 | SSH | TCP | 22 | ADMIN_PUBLIC_IP/32 | ALLOW |
| 120 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 110| HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 130 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | All | 0.0.0.0/0 | DENY |

Replace **ADMIN_PUBLIC_IP/32** with the SSH computer's actual Internet address followed by `/32`. For additional administrators, add separate SSH rules numbered 101, 102, etc., each with its corresponding /32 source. Do not open port 22 to the entire Internet.

Rule 110 allows port 80 for the initial Nginx page and the HTTPS redirect in 5.4.4.

Rule 130 permits responses to ephemeral ports when EC2 initiates outgoing connections, such as S3/CloudWatch calls. The lab uses 1024-65535 to accommodate different operating systems. A NACL cannot determine whether a packet is a response; this rule also permits new TCP packets to that range. **sg-secure-web must still block new inbound connections to backend ports.**

Retain the `*` rule; do not add an All traffic ALLOW rule ahead of these entries. This workshop does not require IPv6 rules.

![](/FCAJ_Worklog/images/005/nacl2.png)


## Step 3: Configure outbound rules

Select **Outbound rules → Edit outbound rules → Add new rule**, enter the table below, then **Save changes**.

| Rule number | Type | Protocol | Port range | Destination | Allow/Deny |
| --- | --- | --- | --- | --- | --- |
| 100 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 110 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | All | 0.0.0.0/0 | DENY |

Outbound 80/443 supports package downloads and HTTP/HTTPS service calls; outbound 80 does not make the website accept HTTP connections. Rule 110 permits SSH/HTTPS responses to clients' ephemeral ports. Outbound port 22 is not a substitute for SSH response traffic.


![](/FCAJ_Worklog/images/005/nacl3.png)


## Step 4: Associate the public subnet

1. Recheck both rule tables and the recorded previous NACL ID.
2. Open **nacl-web-public → Subnet associations → Edit subnet associations** (the button may appear as **Edit**).
3. Select **subnet-web-public**, verifying its Subnet ID and CIDR 10.0.0.0/24.
4. Select **Save**. Change only the workshop subnet association.
5. Open **Subnets → subnet-web-public → Network ACL** and confirm the ID of nacl-web-public.

A subnet uses one NACL at a time; the new association replaces the previous one. Do not modify the default NACL shared by other subnets.

![](/FCAJ_Worklog/images/005/nacl4.png)


## Step 5: Verify and hand over

- Confirm the Region, VPC, subnet, and NACL ID.
- SSH permits only administrator /32 sources; HTTPS permits Internet users.
- TCP 1024-65535 exists in both directions; no unexpected broad ALLOW or lower-numbered DENY rules exist.
- Record the new NACL ID, previous rollback ID, and SSH sources in the handover table.
- Update both the NACL and security group when administrator IP addresses change.

This is configuration verification; do not test connections before EC2 exists. After deployment in section 5.4, perform allow/deny tests in 5.7.1. To demonstrate DENY later, temporarily add inbound rule **90, HTTPS/TCP 443, test computer IP/32, DENY**, open a new HTTPS connection, and record the result. Then remove rule 90 and confirm restored access. Do not add this test rule during the current configuration step.

## Common issues and rollback

| Symptom | Check/action |
| --- | --- |
| Connectivity lost after association | Check both directions; for quick rollback, open Subnets → Network ACL → Edit network ACL association, select the previous NACL ID, and save |
| SSH/HTTPS times out despite correct SG rules | Check lower-numbered DENY rules, /32 sources, and ephemeral response ports |
| EC2 cannot download packages or call S3/CloudWatch | Check outbound 80/443, inbound ephemeral ports, routes, and DNS; S3 AccessDenied requires IAM/policy checks |
| Subnet missing from association list | Verify both resources' VPC and Region |
| Cannot create/edit a rule | Check duplicate rule numbers and IAM permissions; send the denied action to the administrator |



Continue to [5.3.5. Create an S3 Gateway VPC Endpoint](../5.3.5-s3-endpoint/).

## References

- [Create a network ACL](https://docs.aws.amazon.com/vpc/latest/userguide/create-network-acl.html)
- [Custom network ACLs and ephemeral ports](https://docs.aws.amazon.com/vpc/latest/userguide/custom-network-acl.html)
- [Control subnet traffic with network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
