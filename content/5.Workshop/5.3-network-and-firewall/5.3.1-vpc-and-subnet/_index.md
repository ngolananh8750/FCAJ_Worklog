---
title: "Create a VPC and Public Subnet"
date: 2026-10-29
weight: 1
chapter: false
pre: "<b>5.3.1. </b>"
---



## Objectives

Create the **vpc-secure-web** VPC and **subnet-web-public** subnet to prepare for the document website's EC2 deployment. Member 1 performs these steps in the AWS Management Console.



## Prerequisites

- The network IAM user has permission to create VPCs and subnets, add tags, and modify DNS attributes.
- Region: **Asia Pacific (Singapore) — ap-southeast-1**.
- If correctly configured resources already exist, verify and reuse them rather than creating duplicates.
- Prepare an image folder to save numbered evidence screenshots as described below.

| Setting | Value |
| --- | --- |
| VPC Name | vpc-secure-web |
| VPC IPv4 CIDR | 10.0.0.0/16 |
| Tenancy | Default |
| IPv6 | Not assigned in this lab |
| Subnet Name | subnet-web-public |
| Subnet IPv4 CIDR | 10.0.0.0/24 |
| Availability Zone | ap-southeast-1a if available |
| DNS resolution / DNS hostnames | Enable both |
| Tags | Project=SecureWebWorkshop |

These CIDRs and names follow Section 5.2.2. If the team has agreed on different settings, consistently use the actual values and update this table before proceeding.

## Step 1: Open the VPC Console

1. Sign in to the AWS Management Console.
2. Search for **VPC** and open the service.
3. Check the Region at the top and select **Singapore — ap-southeast-1**.
4. Choose **Your VPCs** in the left menu.

![vpc1](/FCAJ_Worklog/images/005/vpc1.png)


## Step 2: Create the VPC

1. Choose **Create VPC**.
2. Under **Resources to create**, select **VPC only**.
3. Enter **Name tag: vpc-secure-web**.
4. Choose manual IPv4 CIDR input and enter **10.0.0.0/16**.
5. Select **No IPv6 CIDR block** and keep **Tenancy: Default**.
6. Add the Project, Environment, and Owner tags from the prerequisites table; retain the Name tag.
7. Review the settings before creation.

Use **VPC only** to configure each component separately according to the workshop contents; this guide does not use the automated VPC and more workflow.

![vpc2](/FCAJ_Worklog/images/005/vpc2.png)


8. Choose **Create VPC**.
9. Open the created VPC and verify its **Available** status and CIDR.
10. Copy the actual **VPC ID** into the handover table.

![vpc3](/FCAJ_Worklog/images/005/vpc3.png)


## Step 3: Enable VPC DNS Settings

1. In **Your VPCs**, select the correct **vpc-secure-web**.
2. Choose **Actions → Edit VPC settings**.
3. Enable **Enable DNS resolution** and **Enable DNS hostnames**.
4. Choose **Save changes**.
5. Check Details again to confirm both attributes are enabled.

![vpc4](/FCAJ_Worklog/images/005/vpc4.png)

## Step 4: Create the Subnet

1. Choose **Subnets → Create subnet**.
2. For **VPC ID**, select the newly created **vpc-secure-web**; verify its ID against Step 2.
3. Enter the following values:

| Field | Value |
| --- | --- |
| Subnet name | subnet-web-public |
| Availability Zone | ap-southeast-1a |
| IPv4 VPC CIDR block, if shown | 10.0.0.0/16 |
| IPv4 subnet CIDR block | 10.0.0.0/24 |

4. If the proposed AZ is unavailable, select another AZ in Singapore and record it.
5. Add the subnet tags Project=SecureWebWorkshop.
6. Create only **one subnet** in this step.

![](/FCAJ_Worklog/images/005/subnet1.png)


7. Choose **Create subnet**.
8. Open the new subnet and verify its **Available** status.
9. Record its **Subnet ID**, **VPC ID**, **AZ**, and **CIDR** in the handover table.

![](/FCAJ_Worklog/images/005/subnet2.png)


## Step 5: Verify and Hand Over

Check the following table and enter actual IDs; do not use example strings as IDs.

| Item | Expected result / value to record |
| --- | --- |
| Region | ap-southeast-1 |
| VPC | vpc-secure-web, Available status |
| VPC ID | Enter the actual ID |
| VPC CIDR | 10.0.0.0/16 |
| DNS resolution | Enabled |
| DNS hostnames | Enabled |
| Subnet | subnet-web-public, Available status |
| Subnet ID | Enter the actual ID |
| Subnet CIDR | 10.0.0.0/24 |
| AZ | Enter the selected AZ |
| Subnet belongs to the correct VPC | VPC ID matches vpc-secure-web |
| Internet connectivity | Not configured in this section |

**You do not need to enable subnet-wide public IPv4 auto-assignment at this stage.** Following Section 5.2.2, Member 2 will set **Auto-assign public IP: Enable** when launching EC2 in Section 5.4.1. Creating the subnet does not assign a public IPv4 address to a server because no EC2 instance exists yet.

A new VPC includes default components such as a main route table, default security group, and default network ACL. Do not delete them in this step. The workshop's separate route table, security group, and network ACL are handled in later sections.

## Common Issues

| Issue | Resolution |
| --- | --- |
| The new VPC is not visible | Check the AWS account, Region, and list filters |
| AccessDenied during creation or tagging | Send the action and error message to the administrator; check creation/tagging permissions and policy conditions |
| Invalid subnet CIDR | Verify that 10.0.0.0/24 is within VPC 10.0.0.0/16 |
| Overlapping subnet CIDR | Check existing subnets; do not create another if the team already has a correctly configured subnet |
| AZ ap-southeast-1a is unavailable | Select another available AZ in the same Region and update the handover table |
| The subnet does not have internet access | This is expected; continue with internet gateway and route table configuration in 5.3.2 |

.

Continue to: [5.3.2. Create an Internet Gateway and Route Table](../5.3.2-gateway-and-route-table/).

## References

- [Create a VPC](https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html)
- [Create a subnet](https://docs.aws.amazon.com/vpc/latest/userguide/create-subnets.html)
- [View and update DNS attributes for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns-updating.html)
