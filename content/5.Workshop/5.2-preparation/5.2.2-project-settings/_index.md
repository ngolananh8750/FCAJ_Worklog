---
title: "Choose the Region, Resource Names, and Demo Scope"
weight: 2
chapter: false
pre: "<b>5.2.2. </b>"
---

Select **Asia Pacific (Singapore)** for the workshop. Use the names below when creating resources.

| Resource | Naming convention |
| --- | --- |
| VPC | `vpc-secure-web` |
| Public subnet | `subnet-web-public` |
| Internet Gateway | `igw-secure-web` |
| Route table | `rtb-web-public` |
| Security group | `sg-secure-web` |
| Network ACL | `nacl-web-public` |
| S3 Gateway Endpoint | `s3-gateway-endpoint` |
| VPC flow log | `flow-web-public` |
| EC2 instance | `workshop-web-01` |
| Key pair | `workshop-web-key` |
| S3 bucket | `secureweb-docs-<unique-suffix>` |
| Network log group | `/workshop/vpc/flow-logs` |
| Application log group | `/workshop/webserver` |
| CloudWatch alarm | `workshop-web-01-cpu-high` |
| CloudWatch dashboard | `workshop-web-01` |
| SNS topic | `workshop-alerts` |

For S3, replace `<unique-suffix>` with your own lowercase suffix, for example `secureweb-docs-team07-a4k` if available.

**Project tag:** Apply `Project=SecureWebWorkshop` when creating resources that support tags so the team can identify lab resources.

**Demo:** Prepare a sample TXT or PDF file for the sign-in, upload, and download checks.

Continue to [5.3. Network setup and firewall](../../5.3-network-and-firewall/).
