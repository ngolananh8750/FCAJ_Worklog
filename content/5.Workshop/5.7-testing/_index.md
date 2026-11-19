---
title: "System Testing"
date: 2026-10-31
weight: 7
chapter: false
pre: "<b>5.7. </b>"
---

## Overview

After deploying VPC, EC2, S3, and CloudWatch, the team tests the document website end to end. The aim is to verify functionality, confirm that out-of-scope access is blocked, and correlate monitoring data with actual user actions.

This page defines the test plan and evaluation criteria. Detailed procedures, commands, and screenshot locations are provided in **5.7.1** and **5.7.2**. The table below describes expected outcomes, not tests already executed on AWS.

## Objectives

- Verify sign-in, sign-out, sessions, uploads, and downloads over HTTPS.
- Test network access restrictions implemented by Security Groups and Network ACLs.
- Verify EC2 permissions for `portal/`, S3 privacy, and encryption.
- Correlate CPU metrics, alarm states, and application logs in CloudWatch.
- Save evidence identifying resources, timestamps, and outcomes before cleanup.

## Prerequisites

1. Complete Sections 5.3–5.6 and confirm **workshop-web-01** is Running in **ap-southeast-1**.
2. Record the Instance ID, current public IPv4, VPC/subnet IDs, Security Group ID, Network ACL ID, actual bucket name, and **WorkshopEC2AppRole**.
3. Prepare the HTTPS URL, test website credentials, and small TXT/PDF samples without personal data. Keep original files for SHA-256 comparison after download.
4. Prepare the administrator computer with an allowed /32 source and a second network source whose public IPv4 is outside the allowlist. If a second source is unavailable, mark that test **Not tested**.
5. Check **workshop-web-01-cpu-high**, log group **/workshop/webserver**, 7-day retention, and the running agent. Stop any remaining CPU test load.
6. Agree on the test window, configuration owner, and rollback method. Save original settings before temporary changes.

The lab uses the self-signed HTTPS certificate from 5.4.4. Verify the certificate/fingerprint as instructed there. A browser trust warning is not a firewall failure, and a self-signed certificate is not publicly CA-trusted.

## Responsibilities

| Member | Responsibility | Evidence |
| --- | --- | --- |
| 1 | SG, NACL, routes, and network access | Actual rules, test sources, and connectivity results |
| 2 | HTTPS, authentication, sessions, uploads/downloads | Functional outcomes and file comparisons |
| 3 | Private bucket, SSE-S3, and IAM Role | Allow/deny results and bucket/object settings |
| 4 | CPU, alarms, and logs | Graphs, Alarm history, and events matching actions |

## Planned test matrix

| ID | Test case | Expected outcome | Section |
| --- | --- | --- | --- |
| WEB-01 | Open the document page or download URL without signing in | Authentication required; document contents are not returned | 5.7.1 |
| WEB-02 | Submit wrong credentials, then correct credentials | Wrong credentials rejected; correct credentials grant access | 5.7.1 |
| WEB-03 | Upload a valid nonempty TXT/PDF up to 5 MiB | Document stored and listed | 5.7.1 |
| WEB-04 | Upload an invalid or oversized file | Rejected without creating a new object from that attempt | 5.7.1 |
| WEB-05 | Download the uploaded file | Readable file with SHA-256 matching the original | 5.7.1 |
| WEB-06 | Sign out and request the document again | Server requires authentication again | 5.7.1 |
| NET-01 | SSH from allowed and unapproved sources | Allowed source connects; unapproved source is blocked | 5.7.1 |
| NET-02 | Access HTTPS and attempt Internet access to backend port 8000 | HTTPS works; backend is not directly accessible | 5.7.1 |
| NET-03 | Perform a controlled NACL change and rollback | New connections are blocked/restored as the tested rule requires | 5.7.1 |
| S3-01 | EC2 uses its intended instance profile to list/read/write portal/ | In-scope operations succeed | 5.7.2 |
| S3-02 | List outside/ and try deleting a dedicated disposable test object | AccessDenied under the configured permissions | 5.7.2 |
| S3-03 | Open an ordinary unsigned object URL without authentication | Private document cannot be read | 5.7.2 |
| S3-04 | Inspect bucket/object settings and S3 routing | Block Public Access enabled, SSE-S3, correct endpoint route | 5.7.2 |
| MON-01 | View CPUUtilization by InstanceId | Recent data for the correct EC2, Average/5 minutes | 5.7.2 |
| MON-02 | Review the time-limited CPU test evidence | ALARM and recovery to OK match the graph/history | 5.7.2 |
| MON-03 | Perform new website actions | New logs arrive in the correct stream and match actions | 5.7.2 |

If SNS is used, add a test verifying email delivery matching the alarm time. Otherwise mark email **Not applicable**, not Pass.

## Interpreting results

- Mark **Pass** only when the actual outcome matches the requirement and has evidence. A negative test passes when access is denied for the expected reason.
- `AccessDenied` is permission-test evidence; timeouts, DNS failures, or missing credentials are not substitutes.
- A failed connection alone does not establish correct firewall behavior. Check source addresses, routes, SG/NACL rules, and listening services; a port with no listener may also fail to connect.
- Use new connections for firewall tests, not an existing SSH session. Evaluate SG and NACL separately and change only one factor per test.
- Verify port 80 redirects to the HTTPS URL and record the flow logs from 5.3.6.
- Do not expose SSH to the entire Internet, make the bucket public, or grant AdministratorAccess to make a test succeed.
- S3 deletion tests use only a disposable object created for testing. Excessive permissions may allow deletion; never use real documents.
- Successfully downloading from S3 does not prove use of the Gateway Endpoint. Inspect the endpoint, route table, and subnet association; do not claim stronger path evidence than was collected.

## Execution order

1. Test website functions and record the initial state.
2. Test network controls, rolling back each temporary change and rechecking HTTPS/SSH.
3. Test S3 using the correct EC2 role, then inspect bucket/object security settings.
4. Verify CloudWatch. Evidence from 5.6.2 can be reused if the instance/settings match and no relevant changes occurred; record its date/time. Repeat load testing only when needed to verify changes or resolve an unsuccessful result.
5. Generate new application events to verify that the agent still delivers logs.
6. Consolidate results, fix defects, retest affected cases, and save evidence before Section 5.8.

## Results template

Use one row per case; duplicate the sample rows as needed.

| Test ID | Tester | Timestamp/time zone | Resource/test source | Actual outcome | Status | Screenshot/log reference |
| --- | --- | --- | --- | --- | --- | --- |
| WEB-01 | Enter name | Enter time | URL/Instance ID | Complete after testing | Not tested | Enter path |
| NET-01 | Enter name | Enter time | Source IP/SG ID | Complete after testing | Not tested | Enter path |
| S3-01 | Enter name | Enter time | Role/bucket/prefix | Complete after testing | Not tested | Enter path |
| MON-03 | Enter name | Enter time | Log group/stream | Complete after testing | Not tested | Enter path |

Use **Pass**, **Fail**, **Not tested**, or **Not applicable**. For failures, record the cause, fix, and retest; do not replace failure evidence with a generic success statement.

## Screenshots

Place detailed screenshots in the relevant subsections. This page only needs the consolidated results when complete:

| Area | Evidence to retain |
| --- | --- |
| Website | Authentication, upload/download, SHA-256 comparison, and post-logout access |
| Firewall | Relevant rules, source, before/after results, and restored settings |
| S3 | Actual role, allow/deny outcomes, Block Public Access, and object encryption |
| CloudWatch | Alarm history/graph, group/stream, and new events |

**📸 Optional summary screenshot:** Completed test results after finishing the subsections. Do not prefill Pass before execution.

<!-- INSERT OPTIONAL TEST RESULTS SUMMARY SCREENSHOT HERE. -->

Do not publish passwords, private keys, access keys, cookies, or tokens. Retain the resource names and timestamps needed to correlate results.

## Completion criteria

All required cases pass with evidence; identify any unexecuted cases explicitly. The website is operational, CPU load has stopped, temporary rules are rolled back, and security settings are preserved. This lab test suite does not establish that the system is completely secure.

## Contents

- [5.7.1. Test Login, Uploads, Downloads, and Firewall](./5.7.1-website-and-firewall/)
- [5.7.2. Verify S3 Access Permissions and Monitoring Results](./5.7.2-s3-and-monitoring/)

After completion: [5.8. Resource Cleanup](../5.8-cleanup/).
