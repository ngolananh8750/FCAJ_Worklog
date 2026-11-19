---
title: "Resource Cleanup"
date: 2026-11-02
weight: 8
chapter: false
pre: "<b>5.8. </b>"
---

## Objective

Remove workshop-only resources after testing to avoid leaving resources that continue to incur charges. Preserve the report, source code, and evidence outside the environment being deleted.

**This is a guide for the team to execute on AWS, not confirmation that resources have been deleted.** EC2 termination, bucket emptying, and log-group deletion can permanently remove data. Operate only on verified lab resource IDs that are no longer shared or needed.

## Preparation and responsibilities

1. Complete [5.7. System Testing](../5.7-testing/). Save results and screenshots locally or elsewhere outside the lab bucket/EC2 instance.
2. Download required documents, source code, and report logs. Do not publish private keys, passwords/hashes, cookies, or access keys.
3. Select the correct AWS account and **Singapore — ap-southeast-1**. IAM is global; still verify the account and resource users when cleaning it up.
4. Agree that the lab is finished and record actual resource IDs and operators. Retain shared resources with a documented reason; do not delete based on Name tags alone.
5. Operators need the relevant deletion permissions. For AccessDenied, use an authorized administrator rather than granting broad administrator access or deletion rights to the application role.

| Owner | Resources |
| --- | --- |
| Member 2 | EC2, EBS, key pair, separately allocated IPs if any |
| Member 3 | S3 objects and bucket |
| Member 4 | Alarms, log groups, optional SNS |
| Member 1 | Endpoint, subnet, route table, NACL, SG, IGW, VPC |
| IAM administrator | Lab-only role, instance profile, and policies |

Recommended sequence: preserve evidence → stop writers → delete alarms → terminate EC2/check leftovers → delete S3/logs/SNS → remove networking → clean IAM → review resources and costs.

## Step 1: Stop the application and log shipping

After preserving final evidence, Member 2 connects to EC2 and runs:

```bash
sudo systemctl stop document-portal
sudo systemctl stop nginx
sudo systemctl stop amazon-cloudwatch-agent
```

If the CPU test is still active, stop it separately:

```bash
sudo systemctl stop workshop-cpu-test
```

If the test unit is absent or already stopped, record that and continue. The website becomes unavailable at this step. There is no need to uninstall Python/Nginx or remove individual files before terminating the instance. If SSH is unavailable but all required data has been saved, terminate the verified instance in Step 3 to stop writers. Do not delete the bucket/log group while writers remain active.

## Step 2: Delete CloudWatch alarms

1. Open **CloudWatch → Alarms → All alarms**.
2. Find **workshop-web-01-cpu-high**, verify its metric Instance ID, and save History if necessary.
3. Select the alarm → **Actions → Delete**, then confirm.
4. Refresh and verify it is no longer listed.
5. Review any additional workshop-only alarms individually. Do not delete shared alarms.

Deleting an alarm does not delete EC2 or historical CPU datapoints. Manually deleting EC2 metrics is not required for cleanup.



## Step 3: Terminate EC2 and check EBS/IP/key pairs

1. Open **EC2 → Instances** and select **workshop-web-01** by the recorded Instance ID.
2. Before deletion, open **Storage**, record each Volume ID, and inspect **Delete on termination**. The lab configures root-volume deletion, but verify actual settings.
3. Choose **Instance state → Terminate (delete) instance**, verify the name/ID, confirm, and wait for **Terminated**.
4. If termination protection is enabled, disable it only on the confirmed lab instance and retry.
5. Open **Elastic Block Store → Volumes** and search the recorded IDs. Delete retained volumes only after verifying that their data is no longer needed and they are not used elsewhere: select an **Available** volume → **Actions → Delete volume**.
6. If snapshots/AMIs were created outside the core guide, inventory them separately. Deregister only unused lab AMIs and delete snapshots no longer needed by other AMIs/backups. Missing tags alone are not grounds for deletion.
7. The automatically assigned public IPv4 is released on termination. The lab does not require an Elastic IP. If one was separately allocated, open **Elastic IP addresses**, verify its Allocation ID, disassociate if needed, and **Release Elastic IP addresses** when unused. Do not confuse auto-assigned IPs with Elastic IPs.
8. Under **Network & Security → Key Pairs**, delete **workshop-web-key** only if no other instance uses it. Delete the local private key only when no longer needed; Console key-pair deletion does not revoke keys already installed on other machines.

**Stop is not a substitute for Terminate** for final cleanup. Retained EBS volumes/snapshots or other resources can still incur charges. A Terminated instance may remain visible in the Console for some time.



## Step 4: Empty and delete the S3 bucket

- Use **Empty** with version deletion permissions on the versioned bucket. Check **Show versions** for remaining versions and delete markers, then delete the bucket. The 30-day lifecycle rule does not empty it on demand.

Member 3 or an authorized data-deletion operator performs this step. Do not grant DeleteObject to WorkshopEC2AppRole just for cleanup.

1. Open **S3 → General purpose buckets**, identify the actual workshop bucket, and verify account, Region, and backups.
2. Ensure EC2/the application no longer writes to it.
3. Select the bucket → **Empty**. Review the scope, enter the exact confirmation requested by the Console, and proceed.
4. Wait for completion and inspect any errors. Samples, uploaded documents, and `portal/ec2-check-...txt` objects are included in this lab bucket's contents.
5. If Versioning was ever enabled/suspended, remove all versions and delete markers. Console Empty handles versions; ordinary deletion of current objects alone may not empty the bucket.
6. If multipart uploads were used outside the lab, inspect unfinished uploads and abort them with authorized S3 tooling before completing cleanup. They are not ordinary listed objects.
7. Return to the bucket list → select the bucket → **Delete**, enter its name as requested, and confirm.
8. Refresh and verify it is no longer listed.

If Object Lock/retention/legal holds block deletion, do not bypass them. These are outside the lab design and require the data owner's handling. For AccessDenied, review permissions/bucket policy/SCP; keep Block Public Access enabled. A deleted bucket no longer reserves its name for you; remove references from any remaining application/configuration used elsewhere.



## Step 5: Delete log groups and optional SNS

- Delete the subnet flow log `flow-web-public` **before** deleting `/workshop/vpc/flow-logs`. Wait for delivery to finish and retain records needed for the report.

- Delete dashboard `workshop-web-01`. After deleting the alarm, remove the subscription and SNS topic `workshop-alerts` if exclusive to the lab.



1. Confirm the agent is stopped or EC2 is Terminated to prevent continued shipping/resource recreation.
2. Open **CloudWatch → Logs → Log groups** and select **/workshop/webserver**.
3. After saving required events, choose **Actions → Delete log group(s)**, confirm, and verify removal. This removes its streams/events; 7-day retention does not replace final group deletion.
4. Inventory and delete any additional workshop-only dashboards/log groups. Do not assume VPC Flow Logs exist if the team never deployed them.
5. If using SNS, open **SNS → Topics → workshop-alerts**, verify its ARN/subscriptions, and ensure it serves no other workload. Delete confirmed lab email subscriptions where supported, then delete the topic and recheck the list. Pending-confirmation subscriptions are handled differently in the Console; do not recreate the topic merely to remove a lingering entry.
6. If SNS was not deployed, record **Not applicable**.



## Step 6: Delete the S3 Gateway Endpoint

1. Open **VPC → Endpoints** and identify **s3-gateway-endpoint** using the recorded Endpoint/VPC IDs.
2. Choose **Actions → Delete VPC endpoints** and confirm.
3. Wait for completion and verify the managed `pl-... → vpce-...` route is removed from the associated route table.
4. Do not delete the AWS-managed S3 prefix list or manually edit the managed route instead of deleting the endpoint.



## Step 7: Remove networking and delete the VPC

Proceed only after EC2 and the endpoint no longer use the network. Verify **VPC ID** and do not select the default VPC.

1. **Network interfaces:** inspect interfaces in vpc-secure-web. The primary EC2 interface is normally deleted after termination; allow time. For remaining service-owned interfaces, identify and remove the owning lab resource first. Do not force-delete unknown interfaces.
2. **Subnets:** select **subnet-web-public**, verify no remaining users, then **Actions → Delete subnet** and confirm.
3. **Route tables:** select **rtb-web-public**. Delete it when it has no subnet associations and is not the main table. If it is main, let VPC deletion handle it rather than deleting another table by mistake.
4. **Network ACLs:** delete **nacl-web-public** after all subnet associations are gone. Do not manually delete the default NACL.
5. **Security groups:** delete **sg-secure-web** after interface attachments and references are removed. Investigate referencing groups and remove only lab-owned references. Do not manually delete the default SG.
6. **Internet gateways:** select **igw-secure-web → Actions → Detach from VPC**, verify vpc-secure-web, then **Delete internet gateway**.
7. **Your VPCs:** select **vpc-secure-web → Actions → Delete VPC**. Review listed dependencies. If an out-of-scope resource appears, stop and investigate. Confirm the correct VPC.
8. Refresh and search by VPC ID to verify removal.

For DependencyViolation, identify the blocking interface, association, or owning service. Do not bulk-delete account resources. The VPC's default SG/NACL/main route table are handled during VPC deletion; the account's default VPC is out of scope.


## Step 8: Remove lab-only IAM resources

- The IAM administrator removes the flow logs delivery role when no flow log references it. Retire only the member IAM users and roles that were dedicated to the lab and are no longer in use.

The IAM administrator performs this after EC2 termination. IAM is global, but the account must still match.

1. Check whether **WorkshopEC2AppRole** or its instance profile is used by another EC2 instance, launch template, or system. If so, retain it with a reason.
2. Record the actual profile name: it may be **WorkshopEC2AppRole** or the same as the role when created through the Console. Do not guess.
3. For lab-only resources, detach policies and delete the role via **IAM → Roles → role → Delete**. Check its instance profile: the Console may remove a same-name profile, while a differently named profile needs separate role removal/profile deletion.
4. If CLI handling is required, the administrator uses their own authenticated CLI environment, not access keys installed on EC2. Replace the placeholder below. Run remove-role only if the role is still in the profile:

```bash
aws iam remove-role-from-instance-profile \
  --instance-profile-name ACTUAL_PROFILE_NAME \
  --role-name WorkshopEC2AppRole
aws iam delete-instance-profile \
  --instance-profile-name ACTUAL_PROFILE_NAME
```

5. Under **IAM → Policies**, delete customer-managed **WS-S3-Portal-ReadWrite** and **WS-CloudWatch-PortalLogs** only when no other identity uses them. Do not delete AWS-managed policies.
6. The four team IAM users are not automatically deleted in this step. Retain shared/ongoing learning accounts and remove only unnecessary lab access. Handle account deletion separately only when the team confirms the accounts were created exclusively for this workshop.



## Step 9: Review remaining resources and costs

1. Recheck the deployment Region for EC2, EBS volumes/snapshots, optional Elastic IPs, S3, CloudWatch alarms/log groups, SNS, and VPC endpoints.
2. If resources were accidentally deployed in other Regions, check those too. Review IAM at account level.
3. If the team added NAT Gateways, load balancers, interface endpoints, or other resources outside the guide, inventory them separately and verify ownership/dependencies before deletion.
4. An authorized billing operator reviews **Billing and Cost Management → Bills** or Cost Explorer. Billing data is delayed; unchanged amounts immediately after cleanup do not prove deletion failed or resources remain running.
5. Do not claim a zero bill. Previously incurred charges remain payable and retained resources may continue to incur charges.


## References

- [Terminate EC2 instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html)
- [Empty an S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/empty-bucket.html)
- [Delete a VPC](https://docs.aws.amazon.com/vpc/latest/userguide/delete-vpc.html)
