---
title: "Create an Internet Gateway and Route Table"
date: 2026-10-29
weight: 2
chapter: false
pre: "<b>5.3.2. </b>"
---


## Objectives

Create **igw-secure-web**, attach it to **vpc-secure-web**, then create **rtb-web-public** with an internet route and associate it with **subnet-web-public**. Member 1 performs these steps in the AWS Management Console.

After this section, the subnet has a direct route to an internet gateway and becomes a public subnet. EC2 still needs a public IPv4 address, appropriate security group and network ACL rules, and a running service for connectivity; this does not yet prove that the website works.

## Prerequisites

- Complete [5.3.1. Create a VPC and Public Subnet](../5.3.1-vpc-and-subnet/).
- Sign in with the network IAM user authorized to create/attach an internet gateway, create route tables, add routes, associate subnets, and tag resources.
- Select **Singapore — ap-southeast-1**.
- Verify the actual VPC ID and Subnet ID from the handover table.
- If resources already exist, verify and reuse them; do not create duplicates or detach a gateway currently in use.

| Component | Value |
| --- | --- |
| VPC | vpc-secure-web — 10.0.0.0/16 |
| Subnet | subnet-web-public — 10.0.0.0/24 |
| Internet gateway | igw-secure-web |
| Route table | rtb-web-public |
| IPv4 internet route | 0.0.0.0/0 → Internet gateway igw-secure-web |
| Tags | Project=SecureWebWorkshop |

## Step 1: Create the Internet Gateway

1. Open **VPC → Internet gateways → Create internet gateway**.
2. Enter **Name tag: igw-secure-web**.
3. Add the shared tags from the table above.
4. Review and choose **Create internet gateway**.
5. Record the Internet Gateway ID. A new gateway not attached to a VPC has **Detached** status; this is not an error.

![](/FCAJ_Worklog/images/005/igw1.png)


## Step 2: Attach the Internet Gateway to the VPC

1. Select **igw-secure-web → Actions → Attach to VPC**.
   ![](/FCAJ_Worklog/images/005/igw2.png)
2. Select **vpc-secure-web**, verifying the VPC ID.
   ![](/FCAJ_Worklog/images/005/igw3.png)
3. Choose **Attach internet gateway**.
4. Verify **Attached** status and the associated VPC.

A VPC can have only one attached internet gateway at a time. If vpc-secure-web is missing from the list, check the Region and any existing attachment; do not detach another system's gateway to proceed.

![](/FCAJ_Worklog/images/005/igw4.png)


## Step 3: Create a Dedicated Route Table

1. In the VPC menu, choose **Route tables → Create route table**.
2. Enter **Name: rtb-web-public**.
3. Select **VPC: vpc-secure-web** using the recorded ID.
4. Add the Project, Environment, and Owner tags from the prerequisites table.
5. Choose **Create route table**.
6. Record the Route Table ID and verify the VPC ID.

Keep the default main route table unchanged. Do not replace the main route table; associate only the workshop subnet with the new table.

![](/FCAJ_Worklog/images/005/rtb1.png)

## Step 4: Add the Internet Route

1. Select **rtb-web-public → Routes → Edit routes → Add route**.
2. Enter **Destination: 0.0.0.0/0**.
3. For **Target**, select **Internet Gateway**, then select the **igw-secure-web** ID.
4. Keep the **10.0.0.0/16 → local** route unchanged.
5. Choose **Save changes** and verify that the new route is **Active**.

| Destination | Target | Expected status |
| --- | --- | --- |
| 10.0.0.0/16 | local | Active |
| 0.0.0.0/0 | Actual igw-secure-web ID | Active |

**0.0.0.0/0 here is a routing destination**, not a rule allowing every port into EC2. Do not confuse it with opening SSH to the entire internet. Do not select a NAT gateway or add an IPv6 route in this IPv4 lab.

![](/FCAJ_Worklog/images/005/rtb2.png)


## Step 5: Associate the Route Table with the Subnet

1. Select **Subnets → subnet-web-public → Route table**.
2. Choose **Edit route table association**.
3. Select the **rtb-web-public** ID and choose **Save**.
4. Recheck the subnet's Route table tab: it must show the new table and internet route.
5. You can also verify under **Route tables → rtb-web-public → Subnet associations**: subnet-web-public should appear under **Explicit subnet associations**.

Adding a route to a table that the subnet does not use will not change the subnet's routing. Do not rely on the table name alone; verify its ID and association.


![](/FCAJ_Worklog/images/005/rtb3.png)


## Step 6: Verify and Hand Over

| Check | Required result |
| --- | --- |
| igw-secure-web | Attached to the correct vpc-secure-web |
| rtb-web-public | Belongs to vpc-secure-web without replacing the main route table |
| Local route | 10.0.0.0/16 → local |
| Default route | 0.0.0.0/0 → igw-secure-web ID, Active |
| Subnet association | subnet-web-public uses rtb-web-public |
| IDs to record | Internet Gateway ID, Route Table ID, VPC ID, Subnet ID |

Save the actual IDs in the handover table and notify Member 2. In Section 5.4.1, EC2 will be launched in this subnet with public IPv4 assignment enabled.

**Do not test the website or ping the internet at this stage if EC2 does not yet exist.** This is a routing configuration check. The S3 gateway VPC endpoint will be added to the route table in Section 5.3.5.

## Common Issues

| Issue | Checks and resolution |
| --- | --- |
| igw-secure-web is missing from Target choices | Check the Region, route table VPC, and whether the gateway is attached to the correct VPC |
| The route shows Blackhole | Check whether the target gateway was deleted or detached; restore the correct attachment/target |
| A 0.0.0.0/0 route already exists | Inspect the existing route; retain it if correct, or edit its target rather than adding a duplicate |
| The subnet still uses the default table | Repeat Step 5 and verify the actual table ID |
| rtb-web-public cannot be selected for the subnet | Verify that the table and subnet belong to the same VPC |
| AccessDenied | Record the action and error; ask the administrator to check route creation, gateway attachment, subnet association permissions, and policy conditions |


Continue to: [5.3.3. Configure a Security Group](../5.3.3-security-group/).

## References

- [Add internet access to a subnet](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-igw.html)
- [Manage subnet route tables](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html)
