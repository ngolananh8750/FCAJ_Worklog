---
title: "Grant IAM Role Permissions and Verify EC2 Read/Write Access to S3"
date: 2026-10-31
weight: 3
chapter: false
pre: "<b>5.5.3. </b>"
---

## Objective

Allow the application on **workshop-web-01** to use **WorkshopEC2AppRole** to list, read, and write documents under **portal/** in the correct bucket. Test successful operations and denied out-of-scope actions, then verify HTTPS uploads/downloads.

Member 3 coordinates with Member 2 and an authorized IAM operator. Do not assume the S3 IAM user can create/edit roles. These instructions are for your group's account; no AWS actions have been performed on your behalf.

## Step 1: Verify role and bucket

Complete 5.5.1–5.5.2 with a private bucket and sample documents. EC2 has the 5.4.3 application and 5.4.4 HTTPS configuration. The Region is **ap-southeast-1**.

1. Open EC2 → Instances → workshop-web-01 → Security and record the attached IAM role.
2. Open IAM → Roles → **WorkshopEC2AppRole** → Trust relationships.
3. Verify EC2 service trust against the example below. Do not casually replace trust on a role used by other systems.
4. If missing, the administrator creates IAM → Roles → Create role → AWS service → EC2, names it WorkshopEC2AppRole, then attaches the policy in step 2.

Role and instance profile names can differ. The prepared profile is WorkshopEC2AppRole; a console-created profile may share the role name. Verify it contains the correct role rather than duplicating it because names differ.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```


## Step 2: Create and attach the least-privilege policy

An authorized IAM operator opens **IAM → Policies → Create policy → JSON**. Paste the JSON below and verify the ARNs against the lab bucket. Select Next, name it **WS-S3-Portal-ReadWrite**, review, and Create policy.

Open **Roles → WorkshopEC2AppRole → Add permissions → Attach policies**, find the new policy, and attach it. If a policy with that name already exists, inspect it before reuse.

ListBucket applies to the bucket ARN with a prefix condition; GetObject/PutObject apply to portal/* object ARNs. DeleteObject, ListAllMyBuckets, and KMS permissions are unnecessary for this SSE-S3 application.

**This Allow policy is not a permissions boundary.** Review all other role and bucket policies: broader S3 grants elsewhere can expand access. The administrator should remove unnecessary S3 permissions from the workshop role after review while retaining required CloudWatch permissions. Do not detach shared policies used by unrelated systems.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListPortalOnly",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::secureweb-docs-683699-280958772766",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "portal/",
            "portal/*"
          ]
        }
      }
    },
    {
      "Sid": "ReadWritePortalObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::secureweb-docs-683699-280958772766/portal/*"
    }
  ]
}
```



## Step 3: Require HTTPS at the bucket

An authorized bucket-policy operator opens **S3 → bucket → Permissions → Bucket policy → Edit**. For a bucket with no policy, use the JSON below and replace secureweb-docs-683699-280958772766 in both ARNs. Otherwise merge the Deny statement into the existing Statement list without overwriting other conditions.

This Deny rejects non-TLS requests from principals that are not AWS services; it does not grant public access. Keep Block Public Access enabled. In this same-account model, step 2's IAM policy normally grants access without an Allow Principal * bucket statement.

Do not require aws:SourceVpce here. The endpoint already routes S3 traffic; requiring every request to use it is a separate change that could block administrative console access.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::secureweb-docs-683699-280958772766",
        "arn:aws:s3:::secureweb-docs-683699-280958772766/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false",
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```



## Step 4: Verify the EC2 instance profile

If the correct role was attached in 5.4.1, retain it. If missing/incorrect, an operator with EC2 permissions and iam:PassRole selects **EC2 → instance → Actions → Security → Modify IAM role**, chooses the profile containing WorkshopEC2AppRole, and saves.

Recheck Security. Allow a short period for IAM changes to propagate; do not create access keys to work around propagation. Do not run aws configure with IAM user credentials on EC2.

The application's Boto3 must obtain credentials from the instance profile. Check that the service does not set AWS_ACCESS_KEY_ID/AWS_SECRET_ACCESS_KEY or an alternative profile; never print secret values. Keep IMDSv2 enabled as configured in 5.4.1.



## Step 5: Test as the actual service account

Use the supplied **ec2-s3-check.py**. On your local computer, put it beside the key pair and transfer it:

```bash
scp -i "YOUR_KEY_PAIR.pem" ec2-s3-check.py ec2-user@PUBLIC_IPV4:~/
```

Replace PUBLIC_IPV4, then run on EC2:

```bash
sudo install -m 644 ~/ec2-s3-check.py /opt/document-portal/ec2-s3-check.py
```

Run the following command as portal in a clean environment. The script reads the bucket from configuration without printing password hashes or credentials. It requires credentials.method to be iam-role and the Caller ARN to contain WorkshopEC2AppRole; otherwise it stops.

The script creates **one new disposable TXT object** with a randomized portal/ key, reads it back, compares SHA-256, and checks AES256. It then attempts to list outside/ and delete **only its newly created test object**, expecting AccessDenied. Never substitute a real document for the deletion test: if access is too broad, deletion may succeed and the script reports FAIL.

STS uses the existing Internet HTTPS route, not the S3 Gateway Endpoint. Investigate STS timeouts as networking issues before concluding that IAM is incorrect.

```bash
sudo -u portal env -i \
  PATH=/usr/bin:/bin HOME=/nonexistent \
  AWS_SHARED_CREDENTIALS_FILE=/dev/null AWS_CONFIG_FILE=/dev/null \
  /opt/document-portal/.venv/bin/python /opt/document-portal/ec2-s3-check.py
```



## Step 6: Verify the website and finish

1. Confirm the actual bucket name in /etc/document-portal/config.json. If editing is necessary, use sudoedit and retain root:portal ownership and mode 640; do not screenshot file contents.
2. After configuration changes, run `sudo systemctl restart document-portal`; in-memory sessions are cleared, so log in again.
3. Open HTTPS as described in 5.4.4 and log in with the website account.
4. Confirm huong-dan.txt/tai-lieu.pdf from 5.5.2 appear.
5. Upload another small TXT; verify its new object is under portal/ with SSE-S3.
6. Download through the website and compare SHA-256 with the original as in 5.5.2.
7. Log out and reopen a download URL: it must redirect to login.

Console success does not replace EC2 testing. Successful EC2 PUT/GET alone does not prove the endpoint path either; verify pl-... → vpce-... in rtb-web-public against 5.3.5. Do not describe it as captured endpoint traffic evidence.



## Troubleshooting and handover

- **Unexpectedly allowed:** stop and review every permission grant. Attaching a minimal policy does not reduce existing broader access.
- **AccessDenied on an allowed operation:** verify bucket name, ARNs, portal/ prefix, actual role, permissions boundary, SCPs, and bucket/endpoint explicit denies.
- **No credentials/not iam-role:** check the profile, IMDSv2, and credential overrides. Do not print metadata credentials.
- **Timeout:** check DNS, routes, NACLs, and outbound 443. The clean test environment does not inherit custom proxies.
- **Website 503 despite script PASS:** verify service configuration, role, and credential overrides.

Count a denial as PASS only for **AccessDenied** on the intended operation; timeout/NoSuchBucket is not permission-boundary evidence. These are representative tests, not exhaustive access verification.

Record policy/role names and ARNs, bucket, test key, and results. The test object remains when deletion is correctly denied; an authorized administrator removes it in 5.8. Do not grant the application DeleteObject just to clean up the test.



Continue to [5.6. CloudWatch Monitoring and Alerts](../../5.6-cloudwatch/).
