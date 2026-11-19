---
title: "Prepare Sample Documents and Verify S3 Storage"
date: 2026-10-31
weight: 2
chapter: false
pre: "<b>5.5.2. </b>"
---

## Objective

Member 3 prepares sample documents, uploads them under **portal/** in the private bucket, verifies object encryption, and downloads them for comparison. This section uses the **S3 console with the data administrator account**; EC2 permissions and website access are tested separately in 5.5.3.

## Prerequisites

- Complete [5.5.1. Create a Private Bucket and Verify Encryption](../5.5.1-private-bucket/).
- Use the correct group bucket in **ap-southeast-1**, retaining Block Public Access, disabled ACLs, and SSE-S3 default encryption.
- The operator can list the bucket, upload/read objects, and inspect encryption settings within the lab scope. Record console permission errors so the administrator can grant the appropriate actions.
- Create a local `s3-sample-documents` directory for originals and later comparison.

## Step 1: Prepare sample documents

Prepare two small files without personal or confidential information:

| File | Creation method | Requirement |
| --- | --- | --- |
| huong-dan.txt | Notepad/VS Code, saved as UTF-8 | Nonempty, with a real .txt extension |
| tai-lieu.pdf | Word or an editor → Export/Save as PDF | Valid PDF that opens correctly |

You can use this TXT content:

```text
Tai lieu thuc hanh AWS Secure Web Workshop.
Chuc nang: dang nhap, tai len va tai xuong tai lieu.
Du lieu nay chi dung de kiem thu, khong chua thong tin ca nhan.
```

The PDF can contain one page describing the workshop, VPC/EC2/S3/CloudWatch services, and test objectives. **Do not rename a .txt extension to .pdf** to create a PDF.

Each file must be at most **5 MiB = 5 × 1024 × 1024 bytes**, matching the 5.4.3 application; files below 100 KB are sufficient for testing. The S3 console may accept larger files, but the application will still reject oversized files. Use ASCII names as shown, lowercase extensions, and no nested folders under portal/ for compatibility with the sample code.

On Windows, enable **View → Show → File name extensions** in File Explorer to avoid huong-dan.txt.txt. Record each file's size and keep the originals unchanged.




## Step 2: Create the portal/ prefix

1. Open **Amazon S3 → Buckets → your group bucket → Objects**.
2. Select **Create folder**, enter **portal**, retain suitable default encryption, and select **Create folder**.
3. Open **portal/** before uploading. If it already exists, open it directly; do not create portal/portal/.

S3 stores objects by key; folders are the console's representation of prefixes. Create folder may create a zero-byte marker object with the key portal/. It is not a sample document, and the application does not list it as a downloadable file.




## Step 3: Upload files to S3

1. Inside **portal/**, select **Upload → Add files**.
2. Choose huong-dan.txt and tai-lieu.pdf from your computer.
3. Confirm Destination is the correct bucket with **portal/**, not the bucket root.
4. Under Properties, retain **Storage class: S3 Standard**.
5. If Server-side encryption is shown, use bucket default encryption (SSE-S3); do not select KMS or customer-provided keys.
6. Do not add public ACLs or Make public options. Review the list and select **Upload**.
7. Wait until both files succeed, select **Close**, and return to Objects.

If an object already has the same key, stop and inspect it before uploading again. The lab bucket is unversioned, so uploading the same key may replace existing data. Rename the sample file before upload if both copies must be retained.




## Step 4: Verify objects and encryption

1. Under portal/, open **huong-dan.txt → Properties**.
2. Record **portal/huong-dan.txt**, size, Last modified, and Storage class.
3. Find **Server-side encryption settings** and confirm **SSE-S3 / Amazon S3 managed keys**.
4. Repeat for tai-lieu.pdf; its key must be **portal/tai-lieu.pdf**.
5. Recheck bucket Permissions if needed: Block Public Access remains enabled and ACLs disabled.

Bucket default encryption and actual object encryption are separate evidence. If an object shows KMS, review upload choices and object history; do not grant broad KMS permissions to bypass the configuration discrepancy.




## Step 5: Download and compare with originals

1. Select each object in the console, choose **Download**, and save it in a separate directory such as `s3-download-check`.
2. Open the downloaded TXT and PDF to confirm readability and matching content.
3. Compare SHA-256 hashes of originals and downloads. Run commands on **your local computer**, adjusting paths as necessary.

**Windows PowerShell:**

```powershell
Get-FileHash -Algorithm SHA256 -Path ".\s3-sample-documents\huong-dan.txt", ".\s3-download-check\huong-dan.txt"
Get-FileHash -Algorithm SHA256 -Path ".\s3-sample-documents\tai-lieu.pdf", ".\s3-download-check\tai-lieu.pdf"
```

**Linux:**

```bash
sha256sum s3-sample-documents/huong-dan.txt s3-download-check/huong-dan.txt
sha256sum s3-sample-documents/tai-lieu.pdf s3-download-check/tai-lieu.pdf
```

On macOS, replace `sha256sum` with `shasum -a 256`. Both hashes for each document must match. Do not treat ETag as SHA-256: an ETag is not always the content checksum you might expect.

Console downloads use the signed-in user's permissions and may use presigned URLs; success does not mean an object is public. Do not share signed download URLs or screenshot signature parameters.




## Step 6: Test anonymous access and hand over

From object details, copy the **ordinary Object URL**, not an Open/Download or presigned URL. For a simple filename, it has this form:

```text
https://BUCKET_NAME.s3.ap-southeast-1.amazonaws.com/portal/huong-dan.txt
```

Replace BUCKET_NAME with the actual name. Open it in an incognito window without AWS credentials. Expect **AccessDenied / HTTP 403**, not document contents. If it is publicly readable, stop and inspect Block Public Access and policies; do not count it as success.

A 403 is one unsigned-access test, not proof that every policy is secure. Verify the object exists and the path is correct, so a wrong-key 403 is not mistaken for evidence of protecting the intended file.




Hand over bucket name, Region, both object keys, sizes, and SHA-256 hashes. Keep originals for Member 2's website download comparison. Do not grant public access to make documents appear in the application; configure the IAM role in 5.5.3.

## Common issues

| Symptom | Resolution |
| --- | --- |
| Files appear at bucket root or portal/portal/ | Check Destination; upload to the correct prefix after confirming no overwrite, and remove misplaced copies only when authorized |
| Upload/download AccessDenied | Check IAM user, bucket policy, prefix permissions, and explicit denies; do not disable Block Public Access |
| Website does not list a file | Check ASCII naming, lowercase .pdf/.txt extensions, keys directly under portal/, and EC2 permissions |
| Hashes differ | Verify the correct object/version and directory and that originals were not modified; download again for comparison |
| Console accepts a file >5 MiB | The console does not enforce application limits; use a suitably small sample |

## Completion

Both sample documents are stored under portal/, use SSE-S3, download with matching SHA-256, and cannot be read anonymously. 
Continue to [5.5.3. Grant IAM Role Permissions and Verify EC2 Read/Write Access to S3](../5.5.3-ec2-access/).
