---
title: "Collect and Verify Application Logs"
date: 2026-10-31
weight: 3
chapter: false
pre: "<b>5.6.3. </b>"
---

## Objective

Member 4 works with Member 2 to send website logs to **CloudWatch Logs**, verify sign-in/upload/download events, and set **7-day** retention. Continue using **workshop-web-01**, **ap-southeast-1**, and **WorkshopEC2AppRole**.

The document-portal service writes `/var/log/document-portal/application.log`. CloudWatch Agent reads it and sends events to `/workshop/webserver`, stream `{instance_id}/application`.

## Prerequisites

- Complete Sections 5.4.3–5.5.3. HTTPS and S3 operations work, and the CPU load from 5.6.2 has stopped.
- Use Amazon Linux 2023, `document-portal.service`, and user `portal` as previously configured. Member 2 has SSH and sudo access.
- The Console operator needs log-group creation, retention management, and log-reading permissions. An authorized IAM operator creates/attaches the EC2 role policy. Member 4's read-only permissions do not automatically allow deployment.
- EC2 has DNS, outbound TCP 443, and an Internet route through its IGW/public IPv4. The S3 Gateway Endpoint does not carry CloudWatch Logs traffic. No additional inbound port is needed.
- This configuration targets the lab instance without another agent configuration. If one exists, back it up and merge the log collection settings; do not unintentionally replace another workload's configuration with fetch-config.

## Step 1: Create the log group and set retention

1. Open **CloudWatch** in **Singapore — ap-southeast-1**.
2. Select **Logs → Log groups → Create log group**.
3. Enter `/workshop/webserver`, select **Standard** log class, and use CloudWatch Logs default encryption for this lab without selecting a customer managed KMS key.
4. Set **Retention = 1 week (7 days)** if available during creation. Otherwise create the group, open **Edit retention setting**, select 7 days, and save.
5. If the group already exists, verify ownership/settings before reusing it. Reducing retention can expire older events; apply this only to the agreed lab logs.
6. Verify the Region, group name, and saved retention.



## Step 2: Grant EC2 log-writing permissions

1. An authorized operator opens **IAM → Policies → Create policy → JSON**.
2. Paste the policy below and replace **ACCOUNT_ID** in both ARNs with the team's 12-digit AWS Account ID.
3. Review and create it as **WS-CloudWatch-PortalLogs**.
4. Open **IAM → Roles → WorkshopEC2AppRole → Add permissions → Attach policies** and attach the policy.
5. Under EC2 → workshop-web-01 → Security, verify this role is attached. Preserve the S3 permissions from 5.5.3 and the EC2 trust policy.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteWorkshopStreams",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:ap-southeast-1:ACCOUNT_ID:log-group:/workshop/webserver:log-stream:*"
    },
    {
      "Sid": "DescribeWorkshopStreams",
      "Effect": "Allow",
      "Action": "logs:DescribeLogStreams",
      "Resource": "arn:aws:logs:ap-southeast-1:ACCOUNT_ID:log-group:/workshop/webserver:*"
    }
  ]
}
```

The policy permits stream creation/event writing only within the specified group. The operator creates the group first; the agent receives no group-creation, log-deletion, or retention-change permissions. The agent configuration below omits `retention_in_days` because retention is managed in the Console. Do not store access keys or run aws configure on EC2.



## Step 3: Install tools and redirect application output to a file

SSH to EC2 as in Section 5.4.1. Run on EC2:

```bash
sudo dnf install -y amazon-cloudwatch-agent logrotate
rpm -q amazon-cloudwatch-agent logrotate
```

Inspect the existing service and journal:

```bash
sudo systemctl cat document-portal
sudo journalctl -u document-portal -n 30 --no-pager
```

The sample service currently uses StandardOutput/StandardError=journal. Create a dedicated drop-in to redirect both to a file. If `20-file-logging.conf` exists, back it up and compare before replacing it. Do not edit the base service file.

**Restarting clears in-memory login sessions. Notify the team and sign in again afterwards.**

```bash
sudo install -d -o root -g portal -m 0750 /var/log/document-portal
sudo touch /var/log/document-portal/application.log
sudo chown root:portal /var/log/document-portal/application.log
sudo chmod 0640 /var/log/document-portal/application.log
sudo install -d -m 0755 /etc/systemd/system/document-portal.service.d
sudo tee /etc/systemd/system/document-portal.service.d/20-file-logging.conf > /dev/null <<'CONF'
[Service]
StandardOutput=append:/var/log/document-portal/application.log
StandardError=append:/var/log/document-portal/application.log
CONF
sudo systemctl daemon-reload
sudo systemctl restart document-portal
sudo systemctl status document-portal --no-pager
sudo tail -n 20 /var/log/document-portal/application.log
```

Systemd opens the output file and passes its descriptor to the process. Preserve User=portal, ProtectSystem=strict, and other service protections; do not use chmod 777. New application messages go to the file, while service lifecycle messages remain available with journalctl. Historical logs are not automatically copied into CloudWatch.

If the service is not active, troubleshoot its journal before continuing. To undo this change, remove only the newly created `20-file-logging.conf` drop-in, run daemon-reload, and restart the service. Do not remove other drop-ins.

## Step 4: Configure local log rotation

Create the following file; back up and compare any existing file with this name first:

```bash
sudo tee /etc/logrotate.d/document-portal > /dev/null <<'CONF'
/var/log/document-portal/application.log {
    daily
    rotate 7
    maxsize 10M
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
    su root root
}
CONF
sudo logrotate -d /etc/logrotate.d/document-portal
sudo systemctl enable --now logrotate.timer
sudo systemctl status logrotate.timer --no-pager
```

`logrotate -d` only validates; it does not perform rotation. maxsize is checked when logrotate runs, not continuously. daily/rotate 7 retains up to seven rotated copies, which differs from CloudWatch's 7-day retention.

This low-volume lab uses copytruncate so the service can continue writing to its open file without restarting and clearing sessions. A small copy/truncate race can lose events; this is not a lossless audit solution. Do not force rotation while collecting test evidence, and monitor local disk usage.


## Step 5: Configure and start CloudWatch Agent

Create the configuration:

```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/workshop-logs.json > /dev/null <<'JSON'
{
  "agent": {
    "region": "ap-southeast-1",
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/document-portal/application.log",
            "log_group_name": "/workshop/webserver",
            "log_stream_name": "{instance_id}/application"
          }
        ]
      }
    },
    "force_flush_interval": 5
  }
}
JSON
sudo python3 -m json.tool /opt/aws/amazon-cloudwatch-agent/etc/workshop-logs.json > /dev/null
```

Collect only application.log, not a wildcard covering all /var/log. No metrics or traces section is configured, so this configuration does not explicitly collect additional custom metrics/traces. The agent runs as root in the lab to read the restricted file; IAM still limits writes to the specified group. Do not add static credentials to the configuration or agent environment.

Start the agent and enable it at boot:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl   -a fetch-config -m ec2   -c file:/opt/aws/amazon-cloudwatch-agent/etc/workshop-logs.json -s
sudo systemctl enable amazon-cloudwatch-agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
sudo systemctl status amazon-cloudwatch-agent --no-pager
```

Expected: agent status **running** and service **active**. Active alone does not establish delivery; verify end-to-end in the next steps. Fix validation/permission errors before proceeding.



## Step 6: Generate real application events

After starting the agent, open the HTTPS website and perform these actions in order:

| Action | Expected sample application event |
| --- | --- |
| Submit one wrong password through a valid form | login_failed |
| Sign in correctly | login_success |
| Upload a small sample TXT/PDF | upload_success bytes=... |
| Download that document | download_success bytes=... |
| Sign out | logout |

Do not repeatedly submit wrong passwords because login attempts are rate-limited. Use documents without personal information. Record each action's time and time zone, then check EC2:

```bash
date -u
sudo tail -n 30 /var/log/document-portal/application.log
```

Find new events matching your actions. If the file has no events, fix application logging first. Do not manually write login_success lines as evidence. There is no need to break S3 access or change security permissions to produce storage_error.

Do not log passwords, password hashes, cookies, tokens, access keys, or document contents. Review logs before publishing screenshots; do not enable SDK debug that includes request bodies.



## Step 7: Correlate events in CloudWatch Logs

1. Open **CloudWatch → Logs → Log groups → /workshop/webserver**.
2. Open **INSTANCE_ID/application**, replacing INSTANCE_ID with the actual instance ID. Use the stream search if needed.
3. Select the last 30 minutes or a range covering your test, then Refresh.
4. Allow time for delivery/display; events need not appear instantly.
5. Search for `login_failed`, `login_success`, `upload_success`, `download_success`, and `logout` in turn. Clear the filter before searching for the next event.
6. Compare messages and times with the EC2 file. Without timestamp_format, the CloudWatch event timestamp may reflect agent read time; the timestamp inside the message is the application's logging time. Record time zones and do not require exact equality.
7. Recheck 7-day retention. Perform another sign-in/sign-out and verify that new events continue to arrive.



## Troubleshooting

| Symptom | Resolution |
| --- | --- |
| Agent active but no stream | Check the pre-created group, Region, instance profile, CreateLogStream/PutLogEvents permissions, and agent error log |
| AccessDenied | Check ACCOUNT_ID, ARNs, actual EC2 role, SCP/permissions boundary, and required actions; do not attach AdministratorAccess |
| ResourceNotFoundException | Create /workshop/webserver in ap-southeast-1 before sending logs |
| No new local events | Check the drop-in, service restart, website actions, and application errors |
| Local events exist but are not sent | Check file_path, read permissions, applied configuration, credentials, and networking |
| Logs endpoint timeout | Check DNS, outbound 443, IGW/public IPv4 route, and return-path NACL rules; the S3 endpoint does not serve Logs |
| Retention is Never expire | An operator with PutRetentionPolicy must set it in the Console; this configuration does not delegate retention to the agent |
| SELinux denies reads | Inspect audit logs and apply appropriate policy/contexts; do not disable SELinux globally |
| Agent cannot parse JSON | Fix syntax, run json.tool, and apply fetch-config again |

Diagnostic commands:

```bash
sudo tail -n 50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
sudo tail -n 50 /opt/aws/amazon-cloudwatch-agent/logs/configuration-validation.log
sudo journalctl -u document-portal -n 30 --no-pager
```

Do not screenshot application configuration containing password hashes or credentials. CloudWatch Logs charges depend on ingestion/storage; retain only necessary logs. CloudWatch retention does not clean local files; logrotate handles the EC2 side.

## Completion criteria

New application events arrive from EC2 in the correct group/stream, retention is 7 days, write permissions are scoped, and local rotation is configured. 

Continue: [5.7. System Testing](../../5.7-testing/).

## References

- [CloudWatch Agent configuration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [CloudWatch Logs IAM policies](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/iam-identity-based-access-control-cwl.html)
- [CloudWatch Agent troubleshooting](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/troubleshooting-CloudWatch-Agent.html)
