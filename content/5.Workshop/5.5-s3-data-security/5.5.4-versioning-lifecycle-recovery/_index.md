---
title: "Manage Lifecycle and Recover Document Versions"
weight: 4
chapter: false
pre: "<b>5.5.4. </b>"
---

## Create two versions of one object

1. Sign in as `s3` and switch to `WorkshopStorageRole`. Open the lab bucket and verify **Properties → Bucket Versioning → Enabled**.
2. Upload two sample TXT files named `recovery-test.txt` containing `version 1` and `version 2` in that order. Upload each separately to `portal/`.
3. Turn on **Show versions** and filter for `portal/recovery-test.txt`. Record both version IDs, download the older version, and check its contents.



## Recover the older version

Download version 1 by version ID and upload it back to the same key. S3 creates a new current version containing `version 1`. Download the current file through the website, verify its contents, and capture the result. Use the storage role for version recovery. The application reads and writes current objects in `portal/` according to 5.5.3.


## Configure lab lifecycle

Under **Management → Create lifecycle rule**, name the rule `portal-expire-noncurrent`, filter for `portal/`, choose **Permanently delete noncurrent versions of objects**, and set **30 days** after a version becomes noncurrent. Retain current versions and capture the review. The rule also covers existing eligible versions and runs asynchronously. Use a disposable file for the recovery demonstration.



[Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/versioning-workflows.html) · [Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)
