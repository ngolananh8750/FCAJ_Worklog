---
title: "Create an S3 Gateway VPC Endpoint"
date: 2026-10-30
weight: 5
chapter: false
pre: "<b>5.3.5. </b>"
---



## Objective

Create **s3-gateway-endpoint**, a **Gateway** endpoint in vpc-secure-web, and associate it with **rtb-web-public** so EC2 can access S3 in the same Region through the endpoint. Member 1 performs this work and hands the details to Members 2 and 3.

Gateway endpoints have no additional usage charge; S3 and other resource charges still apply. Do not create a NAT Gateway or S3 Interface Endpoint for this lab.

## Prerequisites

- Complete [5.3.4. Configure Network ACLs](../5.3.4-network-acl/).
- Region: **Singapore — ap-southeast-1**.
- Verify that subnet-web-public uses rtb-web-public in vpc-secure-web.
- VPC DNS resolution and DNS hostnames were enabled in 5.3.1.
- Obtain permissions to create VPC endpoints, associate route tables, apply tags, and view network resources.
- The document bucket will be created in **ap-southeast-1** in 5.5.1; it does not need to exist before creating the endpoint.

| Property | Value |
| --- | --- |
| Name | s3-gateway-endpoint |
| Service | com.amazonaws.ap-southeast-1.s3 |
| Type | Gateway |
| VPC | vpc-secure-web |
| IP address type | IPv4 |
| Route table | rtb-web-public |
| Initial policy | Full access |
| Tags | Project=SecureWebWorkshop |

## Step 1: Select the S3 Gateway service

1. Open **VPC → Endpoints → Create endpoint**.
2. Enter Name tag **s3-gateway-endpoint**.
3. Select service category **AWS services**.
4. Filter **Type = Gateway**, then select **com.amazonaws.ap-southeast-1.s3**.
5. Confirm the Type is **Gateway**. Do not select Interface or the s3express service.

![](/FCAJ_Worklog/images/005/ep1.png)


## Step 2: Select the VPC and route table

1. Select **VPC: vpc-secure-web**, verifying its actual VPC ID.
2. If **IP address type** is shown, select **IPv4**; if a DNS record IP type option is shown, select IPv4 to match the workshop network.
3. Under **Route tables**, select **rtb-web-public** using its verified Route Table ID.
4. Confirm this table is associated with subnet-web-public, rather than selecting the default main route table.

Gateway endpoints use route table associations; they do not require a separate endpoint security group as interface endpoints do. If the page requests endpoint subnets/AZs and a security group, recheck the Type.

![](/FCAJ_Worklog/images/005/ep2.png)


## Step 3: Select a policy and create the endpoint

1. Under **Policy**, select **Full access** for this initial setup.
2. Add the Project, Environment, and Owner tags from the prerequisites table.
3. Verify the service, Type, VPC, and route table, then select **Create endpoint**.
4. Record its `vpce-...` Endpoint ID and wait for **Available**, refreshing the list if necessary.

Full access is the endpoint's access policy; it **does not grant IAM permissions or make a bucket public**. This initial infrastructure configuration avoids accidentally blocking buckets containing installation packages or agents. Application permissions must still be restricted through the IAM role and bucket policies in section 5.5.

Before restricting the endpoint policy to only the document bucket later, check installation dependencies. Do not paste policies containing unresolved example bucket names or ARNs. Do not add an aws:SourceVpce-only bucket restriction at this stage either, as it can block administrative console access outside the endpoint.

![](/FCAJ_Worklog/images/005/ep3.png)


![](/FCAJ_Worklog/images/005/ep4.png)


## Step 4: Verify the automatically added S3 route

Open **VPC → Route tables → rtb-web-public → Routes** and check these three routes:

| Destination | Target | Purpose |
| --- | --- | --- |
| 10.0.0.0/16 | local | Internal VPC traffic |
| 0.0.0.0/0 | igw-secure-web ID | Internet and other service access |
| pl-... for S3 in the current Region | s3-gateway-endpoint ID | S3 access through the Gateway Endpoint |

AWS adds the prefix list route when the route table is associated with the endpoint. Do not manually add a 0.0.0.0/0 route to the endpoint or remove the Internet Gateway route. In this configuration, S3 destinations matching the prefix list use the endpoint; CloudWatch and other destinations retain the existing Internet path.

If the S3 route is missing, open the endpoint's **Route tables** tab and check the association. Use **Actions → Manage route tables** to select rtb-web-public and save. Do not directly edit the AWS-managed endpoint route.

![](/FCAJ_Worklog/images/005/ep5.png)


## Step 5: Check firewall settings and hand over

- sg-secure-web retains outbound access for required traffic as configured in 5.3.3.
- nacl-web-public permits outbound TCP 443 and inbound TCP 1024-65535 for S3 requests and responses, as configured in 5.3.4.
- Keep the bucket in the same Region. The application uses the S3 SDK/CLI with Region ap-southeast-1 and HTTPS; do not use the Endpoint ID as a URL.
- Record the actual Endpoint ID, VPC ID, Route Table ID, prefix list ID, and Region in the handover table.

NACLs do not support prefix lists as sources/destinations. The current rules allow the required IPv4 CIDRs; any later restriction to S3 address ranges must use actual CIDRs and be maintained as AWS changes them.

Do not test uploads/downloads yet because EC2, the bucket, and the IAM role are not ready. Perform data access tests in **5.5.3** and **5.7.2**. Available confirms endpoint readiness, not application S3 permissions. A successful download from a laptop does not prove that EC2 traffic uses the endpoint.

## Common issues

| Symptom | Resolution |
| --- | --- |
| Only Interface type appears | Filter for Gateway and com.amazonaws.ap-southeast-1.s3 |
| rtb-web-public is missing | Verify the VPC/Region and Route Table ID |
| No pl-... route | Check the endpoint association and the subnet's actual route table |
| Existing endpoint/route for the same service | Inspect and reuse the correct existing endpoint instead of duplicating it |
| S3 calls time out later | Check DNS, routes, Region, outbound 443, and NACL inbound ephemeral ports |
| S3 returns AccessDenied | Check IAM role, bucket policy, and endpoint policy; do not make the bucket public |
| AccessDenied when creating the endpoint | Ask the administrator to review the denied action, creation/association permissions, and policy conditions |



Continue to [5.4. Deploy a Web Server on EC2](../../5.4-ec2-web-server/).

## References

- [Gateway endpoints for Amazon S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
- [Gateway endpoints: routing and security](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)
