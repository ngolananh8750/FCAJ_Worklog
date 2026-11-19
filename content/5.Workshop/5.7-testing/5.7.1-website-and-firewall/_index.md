---
title: "Test Login, Uploads, Downloads, and Firewall"
date: 2026-10-31
weight: 1
chapter: false
pre: "<b>5.7.1. </b>"
---

## Objective

Execute **WEB-01–WEB-06** and **NET-01–NET-03** from Section 5.7. Member 2 tests the website; Member 1 tests networking and rolls back every temporary change. Record actual results rather than pre-marking Pass.

## Prerequisites

- Complete 5.3–5.6. workshop-web-01 runs in ap-southeast-1, Nginx serves HTTPS on 443, and the backend listens only on 127.0.0.1:8000.
- Prepare `https://PUBLIC_IPV4`, website credentials, and nonempty TXT/PDF samples no larger than **5 MiB = 5,242,880 bytes**. Replace PUBLIC_IPV4 with the current address.
- Verify the self-signed certificate as in 5.4.4 before signing in. Do not hide certificate problems by disabling TLS verification.
- Prepare an administrator computer with an allowed public IPv4 /32 and a second network source outside the allowlist. Two computers on the same Wi-Fi often share a public IP and are not distinct test sources.
- Record the Instance ID, all attached interface Security Groups, NACL, and subnet association. Save original rule screenshots.
- The NACL operator must have a signed-in AWS Console and rollback permissions. Do not rely solely on SSH for network recovery. Test only the team's lab resources, not an out-of-scope shared subnet.

## Step 1: Verify authentication is required — WEB-01

1. Open a private/incognito window without an authenticated session.
2. Open the HTTPS website. Expect a sign-in page without the document list.
3. Copy the download link for an existing sample document from a separate signed-in session and paste it into the unauthenticated window. Do not copy cookies or tokens.
4. Expect redirection to sign-in without document contents. If no document exists yet, complete Step 3 and return to this case.

**📸 Screenshot 01:** Authentication required before accessing documents. `01-auth-required.png`.

<!-- INSERT SCREENSHOT 01 -->

## Step 2: Test sign-in — WEB-02

1. Submit the correct username with a wrong password **once** through the normal form.
2. Verify rejection and no document access.
3. Submit correct credentials and verify access to the document list.
4. Do not capture passwords or cookies. If rate-limited, wait and retry; rate limiting is not evidence of incorrect-password handling.

**📸 Screenshot 02:** Rejected and successful sign-in, using two images if needed. `02-login-results.png`.

<!-- INSERT SCREENSHOT 02 -->

## Step 3: Test valid uploads — WEB-03

1. Prepare UTF-8 `kiem-thu.txt` and a genuine `kiem-thu.pdf`, each below the limit. Renaming TXT to PDF does not create a PDF.
2. Record original sizes and SHA-256 hashes. In **PowerShell on your computer**, from the sample directory:

```powershell
Get-Item .\kiem-thu.txt, .\kiem-thu.pdf | Select-Object Name, Length
Get-FileHash .\kiem-thu.txt -Algorithm SHA256
Get-FileHash .\kiem-thu.pdf -Algorithm SHA256
```

3. While signed in, select and upload each file.
4. Confirm success and new list entries. The application adds a random filename prefix; record the **actual names** and download URLs.
5. Ask Member 3 to correlate the new objects under `portal/` in the correct bucket if end-to-end storage confirmation is needed.

**📸 Screenshot 03:** Document list with the new uploads. `03-valid-uploads.png`.

<!-- INSERT SCREENSHOT 03 -->

## Step 4: Test rejected files — WEB-04

Create these files in a dedicated local test directory using PowerShell. If a filename already exists, choose a different name rather than overwriting real data.

```powershell
Set-Content -Path .\khong-hop-le.exe -Value 'Harmless test text' -Encoding ascii
Set-Content -Path .\gia-pdf.pdf -Value 'This is not a PDF' -Encoding ascii
[System.IO.File]::WriteAllText((Join-Path (Get-Location) 'rong.txt'), '')
[System.IO.File]::WriteAllText((Join-Path (Get-Location) 'qua-lon.txt'), ('A' * 5242881))
Get-Item .\khong-hop-le.exe, .\gia-pdf.pdf, .\rong.txt, .\qua-lon.txt | Select-Object Name, Length
```

The .exe contains harmless text; do not execute it. `qua-lon.txt` contains ASCII text and exceeds 5 MiB by one byte.

1. Upload each file while signed in.
2. Expect rejection of the unsupported extension, fake PDF, empty file, and oversized file.
3. If the file picker only shows TXT/PDF, select All files. If the browser blocks submission, record only client-side validation; this does not establish server-side validation.
4. Check the error or HTTP status in Developer Tools → Network if needed. Do not capture headers containing cookies.
5. Correlate the website list and new S3 objects with test timestamps to confirm that the rejected attempts created no objects.

The application file limit is 5 MiB; Nginx's request limit is 6 MiB as previously configured. A 413 for a larger request can originate from Nginx. Do not attribute every rejection to the same layer.

**📸 Screenshot 04:** Rejection messages and test file sizes; multiple images may be used. `04-invalid-uploads.png`.

<!-- INSERT SCREENSHOT 04 -->

## Step 5: Download and compare — WEB-05

1. Download the TXT and PDF uploaded in Step 3.
2. Save them separately from the originals. Open them and verify readability.
3. Run Get-FileHash with the **actual downloaded paths**, comparing each with its original. Replace the placeholder in this example:

```powershell
Get-FileHash 'C:\ACTUAL_PATH\DOWNLOADED_FILE.txt' -Algorithm SHA256
```

4. Pass when both hash pairs match and the files open. Random filename prefixes may change names, but contents must remain identical.

**📸 Screenshot 05:** Matching original/downloaded SHA-256 hashes. `05-download-hash.png`.

<!-- INSERT SCREENSHOT 05 -->

## Step 6: Sign out and request documents again — WEB-06

1. Save a document download URL while signed in, then select **Log out**.
2. Paste the URL into the same browser's address bar and issue a new request.
3. Reopen the document list. Both requests must require authentication again.
4. Back may show a cached page; reload or issue a new request to test the server. Previously downloaded local files do not disappear on logout.
5. Also check the download URL in an unauthenticated private window to finish WEB-01 if still pending.

**📸 Screenshot 06:** A new post-logout document request requires sign-in. `06-logout-protection.png`.

<!-- INSERT SCREENSHOT 06 -->

## Step 7: Test SSH from allowed and blocked sources — NET-01

On your computer, use PowerShell and replace the IP:

```powershell
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 22
```

1. From the allowed /32, expect `TcpTestSucceeded : True`; confirm SSH login with the existing key as in 5.4.1.
2. From the unapproved second network, repeat the command and expect False/timeout. No private key needs to be copied to another computer just to test TCP.
3. Record both actual public source IPs, timestamps, and relevant SG/NACL rules.
4. If both fail, investigate the running instance, address, routes, NACL, and sshd before claiming a successful allow/deny test.

SSH `Permission denied (publickey)` indicates that SSH was reached but key authentication failed; it is not proof of TCP filtering. Without a second source, mark **Not tested**. This case tests combined network controls; do not attribute blocking solely to SG if the NACL also blocks that source.

**📸 Screenshot 07:** Both source results and /32 rules. `07-ssh-allow-deny.png`.

<!-- INSERT SCREENSHOT 07 -->

## Step 8: Test HTTPS and the backend — NET-02

On your computer:

```powershell
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 443
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 8000
```

Expect True for 443 with a working HTTPS website and False for 8000. On EC2, inspect listening sockets:

```bash
sudo ss -lntp
```

Verify Gunicorn binds only to **127.0.0.1:8000**, Nginx listens on 443, and none of the attached SGs opens 8000 to the Internet. Do not change the bind address or expose the backend to manufacture test evidence. Failure on 8000 plus socket/rule inspection establishes that the backend is not public; it does not isolate SG effectiveness.

Run `curl.exe -I http://PUBLIC_IPV4/` on Windows or `curl -I http://PUBLIC_IPV4/` on Linux/macOS. Expect 301 and a Location pointing to HTTPS. Check port 81 against 5.3.6.

**📸 Screenshot 08:** 443/8000 results, backend socket, and relevant SGs. `08-https-backend.png`.

<!-- INSERT SCREENSHOT 08 -->

## Step 9: Perform a controlled NACL test — NET-03

Temporarily block **HTTPS from only the test computer's IP**, preserving SSH and outbound rules. Proceed only with Console access and rollback permissions available.

1. Identify the current test computer's public IPv4, represented by **TEST_PUBLIC_IP/32**. Replace this placeholder with the actual CIDR. Avoid VPN/proxy differences between browser and PowerShell sources.
2. Before changing anything, test TCP 443 again from this computer and record True.
3. Open **VPC → Network ACLs**, select **nacl-web-public** by ID, and verify **Subnet associations** includes the EC2 instance's subnet-web-public. Save original inbound/outbound settings.
4. Open **Inbound rules → Edit inbound rules → Add new rule**. If 90 is unused and precedes the applicable HTTPS allow rule, add:

| Field | Test value |
| --- | --- |
| Rule number | 90 |
| Type / Protocol / Port | HTTPS / TCP / 443 |
| Source | Actual TEST_PUBLIC_IP/32 |
| Allow/Deny | DENY |

5. If 90 is occupied, do not overwrite it. Choose an unused number below the applicable 443 allow rule. Ensure no earlier matching allow bypasses the test deny. If uncertain, stop and have the network owner review it.
6. Save and start a **new TCP connection** with Test-NetConnection to 443. Expect False/timeout. Do not rely solely on a cached browser page or existing connection.
7. **Roll back immediately after recording the result:** delete only the temporary deny rule and save. Do not alter the default * rule, original rules, or subnet association.
8. Retest 443 and expect True; open the website and sign in. Confirm SSH still works.

If the deny is ineffective, check the source IP, rule order, and NACL association. Do not expand the deny to 0.0.0.0/0. Even when the test fails, remove the temporary rule before further troubleshooting.

**📸 Screenshot 09:** Temporary deny rule and blocked 443 result. `09-nacl-deny.png`.

<!-- INSERT SCREENSHOT 09 -->

**📸 Screenshot 10:** Temporary rule removed, restored 443, and working website. `10-nacl-restored.png`.

<!-- INSERT SCREENSHOT 10 -->

## Results

| ID | Required outcome | Actual status |
| --- | --- | --- |
| WEB-01 | Unauthenticated users cannot read documents | Not tested |
| WEB-02 | Wrong credentials rejected; correct sign-in succeeds | Not tested |
| WEB-03 | Valid TXT/PDF uploads succeed | Not tested |
| WEB-04 | Invalid uploads rejected without objects being created | Not tested |
| WEB-05 | Downloads match original SHA-256 | Not tested |
| WEB-06 | New post-logout requests require sign-in | Not tested |
| NET-01 | SSH allow/deny matches source restrictions | Not tested |
| NET-02 | HTTPS works; backend is not public | Not tested |
| NET-03 | NACL blocks and connectivity recovers after rollback | Not tested |

Enter actual Pass/Fail results with tester, timestamp, and evidence references using Section 5.7's template. There are **10 screenshot locations**, each of which may contain multiple images. Distinguish TLS, application, SSH authentication, and network errors; do not broaden permissions to bypass causes.

Completion requires evidence for the required cases, rollback of temporary rules, and a working website. Delete only locally created test files after retaining evidence; disposable S3 objects will be handled during cleanup.

Continue: [5.7.2. Verify S3 Access Permissions and Monitoring Results](../5.7.2-s3-and-monitoring/).
