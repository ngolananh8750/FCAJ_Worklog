---
title: "Prepare the AWS Account and Access Permissions"
weight: 1
chapter: false
pre: "<b>5.2.1. </b>"
---

Prepare one AWS account for the workshop. The administrator creates the four IAM users below and lets each user switch to their assigned role. Team members use MFA to sign in to the AWS Console.

| IAM user | IAM role | Required lab permissions |
| --- | --- | --- |
| `vpc` | `WorkshopNetworkRole` | Create and manage VPCs, subnets, route tables, internet gateways, security groups, network ACLs, S3 gateway endpoints, and VPC Flow Logs. |
| `ec2` | `WorkshopComputeRole` | Create and manage lab EC2 instances, EBS volumes, and key pairs. View and select existing network resources. Attach the instance profile for the approved application role. |
| `s3` | `WorkshopStorageRole` | Create and manage the private lab bucket, its bucket policy, encryption, Versioning, Lifecycle, and sample objects. |
| `cloudwatch` | `WorkshopMonitoringRole` | Read EC2 metrics. Manage lab log groups, retention, dashboards, alarms, and SNS topics. |

**AWS service roles**

| IAM role | Required permissions |
| --- | --- |
| `WorkshopEC2AppRole` | EC2 uses this role to list, read, and upload S3 documents under `portal/` and send application logs to CloudWatch Logs. |
| `WorkshopFlowLogsDeliveryRole` | VPC Flow Logs uses this role to write to the workshop log group. |

Grant each role only the permissions needed for its tasks. Limit `iam:PassRole` for the compute and network roles to their respective service roles. Keep the S3 bucket private and use the IAM role for EC2 credentials.

Continue to [5.2.2. Choose the Region, Resource Names, and Demo Scope](../5.2.2-project-settings/).
