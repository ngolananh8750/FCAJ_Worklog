---
title: "Record VPC Flow Logs and Check Ports 80 and 81"
weight: 6
chapter: false
pre: "<b>5.3.6. </b>"
---

## Configure subnet flow logging

1. Member 4 creates `/workshop/vpc/flow-logs` in Singapore if needed. Set seven-day retention for the lab.
2. The administrator checks that `WorkshopFlowLogsDeliveryRole` trusts `vpc-flow-logs.amazonaws.com`, can write to CloudWatch Logs, and that `WorkshopNetworkRole` can pass that specific delivery role.
3. Member 1 opens **VPC → Subnets → subnet-web-public → Flow logs**. Verify `subnet-03d4ed311fc7d0e28`. Reuse `flow-web-public` if present. To create it, select **All**, **1 minute**, **Send to CloudWatch Logs**, and the correct log group and delivery role. Choose the default record format for the query below.
4. Confirm Active status. Generate traffic and wait for delivery. Check delivery status if records are missing.

<!-- 📸 Insert subnet, flow log, and log group screenshots here. -->

## Compare allowed and rejected traffic

After Nginx starts, run these commands from your computer in PowerShell, using the current public IPv4 address:

```powershell
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 80
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 81
```

On Linux/macOS, use `curl -I --connect-timeout 5 http://PUBLIC_IPV4/` and `curl -I --connect-timeout 5 http://PUBLIC_IPV4:81/`. The lab recorded **ACCEPT on 80** and **REJECT on 81**. After enabling HTTPS, port 80 returns a redirect. Keep port 81 outside the allowed rules.

In **CloudWatch → Logs Insights**, select the log group and time window. Use this query with the default record format:

```text
fields @timestamp, @message
| parse @message "* * * * * * * * * * * * * *" as version, account, eni, src, dst, sport, dport, proto, packets, bytes, started, ended, action, status
| filter dport in ["80", "81"] and proto = "6"
| display @timestamp, eni, src, dst, dport, action, status
| sort @timestamp desc
```

Match the ENI, source IP, and test time. Flow Logs show whether traffic was accepted or rejected. Inspect the SG and NACL to identify the rule. One minute is the maximum aggregation interval. Delivery can take longer.

<!-- 📸 Insert ACCEPT 80 and REJECT 81 records here. -->

[Create a Flow Log for CloudWatch Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-cwl-create-flow-log.html)
