---
title: "Create a Private Bucket and Verify Encryption"
date: 2026-10-31
weight: 1
chapter: false
pre: "<b>5.5.1. </b>"
---

## Objective

Member 3 creates a private S3 bucket in **Singapore — ap-southeast-1**, enables Block Public Access, and verifies **SSE-S3** default encryption. The bucket will store application documents under portal/.

This section verifies bucket configuration. Actual object encryption is checked after uploading sample documents in 5.5.2; default configuration is not evidence about an object that does not yet exist.

## Prerequisites

- Use an account authorized to create buckets and configure security as described in 5.2.1; object read/write permissions alone do not imply bucket creation permissions.
- Agree on the actual name from 5.2.2: **secureweb-docs-683699-280958772766**.

- If the bucket already exists in your account, verify its Region, owner, and configuration before reusing it. Do not create another bucket merely because permissions prevent you from seeing it.

Never include passwords or secrets in bucket names. In the shared namespace, names must be available throughout the corresponding AWS partition, not just your account. If the console offers namespace choices, this lab uses **General purpose with the shared global namespace** to retain the agreed naming pattern; do not switch to a different generated name without updating the application.

When reusing the bucket, verify **Properties → Bucket Versioning** and enable it if needed. Check the security settings in Step 4.

## Step 1: Enter bucket details

1. Sign in to the AWS Management Console using the S3 operator account.
2. Open **Amazon S3 → Buckets → Create bucket**.
3. Select **Asia Pacific (Singapore) — ap-southeast-1** under **AWS Region**. The S3 console may display multiple Regions; verify the Region in the creation form.
4. Choose **Bucket type: General purpose** if shown; do not select Directory bucket/S3 Express One Zone.
5. Enter the agreed actual bucket name.
6. Do not copy settings from a bucket outside the workshop.

![](/FCAJ_Worklog/images/55/b1.png)



## Step 2: Keep the bucket private

1. Under **Object Ownership**, retain **ACLs disabled (recommended)**, corresponding to **Bucket owner enforced**.
2. Under **Block Public Access settings for this bucket**, keep **Block all public access** selected.
3. Verify all four public-access blocking options are enabled. Do not clear them to make the website work.

The EC2 application accesses S3 through its IAM role, and users download through the backend, so the bucket does not need public access. Disabling ACLs keeps access management policy-based rather than using individual object ACL grants.

Block Public Access does not replace IAM or automatically remove all authenticated or cross-account access. Policies still need appropriate restrictions in 5.5.3.

![](/FCAJ_Worklog/images/55/b2.png)



## Step 3: Choose encryption and lab options

| Property | Value |
| --- | --- |
| Bucket Versioning | Enable |
| Tags | Project=SecureWebWorkshop |
| Default encryption | Server-side encryption with Amazon S3 managed keys (SSE-S3) |
| Object Lock | Do not enable |

Select **SSE-S3**, not SSE-KMS or DSSE-KMS. Do not create a dedicated KMS key. If S3 Bucket Key is not shown with SSE-S3, this is expected; that option relates to SSE-KMS.

Enable Versioning for the recovery exercise in 5.5.4. After first enabling it, wait 15 minutes before object writes or deletes.

![](/FCAJ_Worklog/images/55/b3.png)



## Step 4: Create and recheck the bucket

1. Review the Region, name, Block Public Access, and SSE-S3 settings.
2. Select **Create bucket**.
3. Open the new bucket and record its exact name and bucket ARN from **Properties**.
4. Open **Permissions** and confirm Block Public Access remains enabled and Object Ownership is Bucket owner enforced.
5. Check **Bucket policy**: a new bucket does not yet require an access-granting policy. Do not paste an Allow policy with Principal * to make the application work.
6. Open **Properties → Default encryption** and confirm SSE-S3. Check that **Static website hosting** is Disabled.

If reusing a bucket with an existing policy, review the entire policy with the administrator before editing; do not delete it. HTTPS-only policy controls and EC2 permissions will be handled with access configuration without making the bucket public.

![](/FCAJ_Worklog/images/55/b4.png)



## Step 5: Hand over and understand verification limits

| Item | Expected result |
| --- | --- |
| Name and Region | Actual agreed name, ap-southeast-1 |
| Owner | The group's AWS account |
| Public access | All four Block Public Access options enabled |
| Object Ownership | Bucket owner enforced / ACLs disabled |
| Default encryption | SSE-S3 |
| Hosting | Static website hosting Disabled |
| Handover | Bucket name, ARN in the form arn:aws:s3:::BUCKET_NAME, and Region |

Member 2 compares the bucket name with /etc/document-portal/config.json; do not screenshot or publish the configuration containing the password hash. Do not add broad policies if the application reports AccessDenied: EC2 permissions are completed in 5.5.3.

Do not test object URLs while the bucket is empty. A 403 from a bucket URL alone does not prove that all policies are secure; inspect configuration and test specific objects later. SSE-S3 encrypts stored data; HTTPS remains necessary for transport. Setting default encryption does not re-encrypt every existing object in a reused bucket.

## Common issues

| Symptom | Resolution |
| --- | --- |
| Bucket name already taken | If it is not yours, change the suffix and update the agreed name/application settings |
| Invalid bucket name | Use lowercase letters, digits, and hyphens following the pattern; avoid spaces, uppercase letters, and literal placeholders |
| AccessDenied during creation/settings | Record the denied action; ask the administrator to review bucket creation, public access block, ownership, encryption, and tagging permissions |
| Wrong Region after creation | A bucket's Region cannot be directly changed; coordinate a correctly located bucket and handle the mistaken resource without casually deleting data |
| Bucket Key option missing | This option is unnecessary for SSE-S3 |
| Application still returns 503 | The bucket may be correct while IAM permissions are missing; complete 5.5.3 |

## Completion

The private bucket is created and ownership, public access, and default encryption have been verified.

Continue to [5.5.2. Prepare Sample Documents and Verify S3 Storage](../5.5.2-upload-documents/).
