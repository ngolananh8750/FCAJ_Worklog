---
title: "Monitoring and Alerts with CloudWatch"
date: 2026-10-31
weight: 6
chapter: false
pre: "<b>5.6. </b>"
---

## Overview

After deploying the website and verifying S3 access, the team uses **Amazon CloudWatch** to monitor the Web Server, detect CPU threshold breaches, and inspect application logs when errors occur.

In this section, **Member 4** manages monitoring and coordinates with Member 2 to configure log collection on EC2. When permissions on **WorkshopEC2AppRole** must change, coordinate with the IAM administrator and preserve the S3 permissions configured in Section 5.5.3.

This page introduces Section 5.6. Detailed Console procedures, installation commands, and tests are provided in the three subsections below.

## Objectives

- View CPU metrics for the correct EC2 instance, **workshop-web-01**, in **Singapore — ap-southeast-1**.
- Create a CPU alarm and verify state changes using actual metric data.
- Collect application logs in CloudWatch Logs and correlate them with test actions on the website.
- Save monitoring evidence for system testing in Section 5.7.

CloudWatch provides visibility; it does not replace Security Groups, Network ACLs, authentication, or S3 access permissions. Low CPU usage alone does not prove that the website is healthy.

## Scope and agreed configuration

| Component | Workshop configuration |
| --- | --- |
| Region | ap-southeast-1 |
| EC2 | workshop-web-01 — record the actual Instance ID |
| Namespace / Metric | AWS/EC2 / CPUUtilization |
| Dimension | InstanceId of workshop-web-01 |
| EC2 monitoring | Basic monitoring; CPU data at 5-minute intervals |
| Alarm name | workshop-web-01-cpu-high |
| Planned alarm settings | Average, 5-minute period, CPU > 80%, 2/2 datapoints |
| Missing data treatment | missing; missing data is not evidence of a healthy system |
| Log group | /workshop/webserver |
| Log retention | 7 days for the lab |
| Log shipping tool | Amazon CloudWatch Agent on EC2 |
| Email notifications | Optional, through the workshop-alerts SNS topic |

Basic monitoring is sufficient for this CPU monitoring exercise. The EC2 CPUUtilization metric does not require CloudWatch Agent. The agent is used in Section 5.6.3 to ship logs. Do not enable Detailed monitoring or collect additional custom metrics unless needed.

A 2/2 datapoints configuration requires two breaching datapoints within two evaluation periods. With a 5-minute period, testing must allow time for data delivery and evaluation; do not expect an alarm immediately after starting a load test. This threshold is a lab choice, not a universal standard for applications.

## Prerequisites

1. Complete [5.5.3. Grant IAM Role Permissions and Verify EC2 Read/Write Access to S3](../5.5-s3-data-security/5.5.3-ec2-access/).
2. Verify that EC2 is running, record its Instance ID, and select the correct Region in the Console.
3. Confirm that the website is accessible over HTTPS and that sign-in, sample uploads, and downloads work.
4. Prepare permissions for Member 4 to view metrics, create/update alarms, and view logs. The log group operator needs permissions to create the group and set retention; read-only log permissions are insufficient for deployment.
5. When configuring the agent, grant the EC2 IAM Role permission to write logs within the workshop log group. Console user permissions and EC2 permissions are separate; do not store access keys on the server.
6. Verify DNS and outbound HTTPS connectivity from EC2 to CloudWatch Logs. The S3 Gateway Endpoint serves S3 only; logs use the existing Internet route through the Internet Gateway and the instance's public IPv4 address.

If permissions are missing, record the denied action and ask the administrator to grant the required scope. Do not attach AdministratorAccess simply to bypass an error.

## Implementation outline

### 5.6.1. Monitor EC2 CPU Metrics

Open CloudWatch Metrics, select CPUUtilization by InstanceId, set Statistic to Average and Period to 5 minutes. Observe the baseline before generating load. Check the Region, time range, and Instance ID if the graph has no data.

### 5.6.2. Create and Test a CPU Alarm

Create **workshop-web-01-cpu-high**, configure the agreed threshold, and examine **OK**, **ALARM**, and **INSUFFICIENT_DATA** states. Run a time-limited load test, observe CPU and Alarm history, stop the load, and verify recovery.

The t3.micro instance uses CPU credits. In Standard mode, depleted credits can limit performance and prevent the test from reaching the planned threshold. Do not run an unlimited load or switch to Unlimited solely to trigger the alarm. Section 5.6.2 will provide the test procedure and troubleshooting steps.

The alarm can be verified directly in the Console. For email notifications, additionally configure SNS and confirm the subscription. An ALARM state in the Console does not prove that email was delivered; verify delivery separately. Do not configure automatic EC2 stop, terminate, or reboot actions for this lab.

### 5.6.3. Collect and Verify Application Logs

Prepare the log group, retention, EC2 permissions, and CloudWatch Agent. The application currently writes logs to the systemd journal, so configure file output or a journal-to-file mechanism before the agent reads them. Installing the agent alone does not automatically collect application logs.

Next, perform test actions such as successful sign-in, failed sign-in, or document operations. Correlate events actually recorded by the application with CloudWatch logs. Consider collection successful only when new events match the test timestamps.

Do not log passwords, password hashes, cookies, session tokens, access keys, or document contents. When using file output, apply appropriate file permissions and log rotation to prevent the EC2 disk from filling up.



## Cost control

Keep Basic monitoring, create only the required alarm, collect files relevant to testing, and set retention to 7 days. CloudWatch Logs, alarms, SNS, and additional features may incur charges depending on usage and account eligibility; do not assume this entire section is free. Remove unused monitoring resources in Section 5.8 after saving the evidence.

## Expected outcomes

- A CPU graph for the correct server.
- A tested alarm and evidence of recovery after the load stops.
- New application logs in CloudWatch with the correct retention and no secrets.
- Recorded alarm name, log group, Instance ID, and test timestamps for handover.

## Contents

- [5.6.1. Monitor EC2 CPU Metrics](./5.6.1-cpu-metrics/)
- [5.6.2. Create and Test a CPU Alarm](./5.6.2-cpu-alarm/)
- [5.6.3. Collect and Verify Application Logs](./5.6.3-application-logs/)
- [5.6.4. Dashboard and SNS notifications](./5.6.4-dashboard-and-sns/)

## References

- [EC2 basic and detailed monitoring](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/manage-detailed-monitoring.html)
- [CloudWatch alarm evaluation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html)
- [CloudWatch Agent configuration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [Log groups, streams, and retention](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)
