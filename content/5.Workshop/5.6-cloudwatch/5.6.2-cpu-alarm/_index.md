---
title: "Create and Test a CPU Alarm"
date: 2026-10-31
weight: 2
chapter: false
pre: "<b>5.6.2. </b>"
---



## Objective

Member 4 creates **workshop-web-01-cpu-high** and works with Member 2 to generate time-limited load on the lab EC2 instance. Verify **OK → ALARM → OK** using actual CPU data and save the graph and Alarm history.

## Prerequisites

- Complete [5.6.1. Monitor EC2 CPU Metrics](../5.6.1-cpu-metrics/), with recent CPU data for workshop-web-01 in ap-southeast-1.
- Member 4 has metric read access and `cloudwatch:PutMetricAlarm`, `cloudwatch:DescribeAlarms`, and `cloudwatch:DescribeAlarmHistory` within the assigned scope. EC2 does not need CloudWatch permissions merely to create an alarm through the Console.
- Member 2 has SSH and sudo access on Amazon Linux 2023.
- Agree on a test window: high CPU may slow the website. Do not test during demonstrations or important data processing.
- Keep **Basic monitoring** and **Standard** CPU credit mode. Alarms may incur charges depending on usage and account eligibility.

## Step 1: Select the alarm metric

1. Open **CloudWatch** in **Singapore — ap-southeast-1**.
2. Choose **Alarms → All alarms → Create alarm**.
3. Choose **Select metric → EC2 → Per-Instance Metrics**.
4. Find the actual Instance ID of **workshop-web-01**.
5. Select **CPUUtilization** with the correct InstanceId, then choose **Select metric**.
6. Set **Statistic = Average**, **Period = 5 minutes**.

If this alarm name already exists, check its Instance ID and owner before editing. Do not overwrite another team's alarm.

![](/FCAJ_Worklog/images/55/m1.png)

## Step 2: Set the condition

| Field | Value |
| --- | --- |
| Threshold type | Static |
| Whenever CPUUtilization is | Greater |
| Threshold | 80 |
| Datapoints to alarm | 2 out of 2 |
| Missing data treatment | Treat missing data as missing |

1. Enter **CPU > 80%**.
2. Expand **Additional configuration** and set **2/2 datapoints**.
3. Select **missing** for missing data treatment, not not breaching.
4. Choose **Next**.

With complete data, two CPU averages from 5-minute periods must breach the condition to satisfy 2/2. A brief spike is insufficient. State changes also depend on period boundaries, data delivery, and alarm evaluation; they are not immediate after starting the command.

![](/FCAJ_Worklog/images/55/m2.png)

## Step 3: Configure actions and create the alarm

1. Under **Configure actions**, skip notifications only for a new alarm if SNS is not ready. Retain existing lab alarm actions.
2. Do not add EC2 Stop, Terminate, Reboot, Recover, or Auto Scaling actions.
3. Choose **Next** and enter **workshop-web-01-cpu-high**.
4. Description: `CPU Average above 80 percent for 2 periods of 5 minutes on workshop web server`.
5. Choose **Next**, review the Instance ID and all settings, then **Create alarm**.
6. Open the alarm's **Details** and **History**. Wait for **OK** with baseline data below the threshold before testing.

| State | Meaning |
| --- | --- |
| OK | Evaluated data does not satisfy the alarm condition |
| ALARM | The alarm condition is satisfied |
| INSUFFICIENT_DATA | Insufficient data to determine the state |

INSUFFICIENT_DATA immediately after creation is not necessarily an error. If it persists, check the metric, Instance ID, Region, and recent data as in Section 5.6.1.

![](/FCAJ_Worklog/images/55/m3.png)

### Optional: Receive email through SNS

Follow this branch if the team wants email notifications and the operator has appropriate SNS permissions:

1. In SNS in the same Region, create or reuse the team's **Standard** topic named `workshop-alerts`.
2. Open the topic → **Create subscription** → Protocol **Email**, enter the team member's notification address, and create the subscription.
3. The recipient opens the AWS email and selects **Confirm subscription**. Verify the subscription is no longer Pending confirmation.
4. Edit the alarm to add an **In alarm** notification to the topic. Optionally add an **OK** notification for recovery. Save before testing.
5. Ensure alarm actions are enabled and the topic policy allows CloudWatch to publish. For AccessDenied, ask the administrator to grant the required permissions; do not make the topic public.

If topic/subscription creation is not permitted, ask an authorized operator to prepare it or complete the Console-only branch first. An ALARM state does not prove email delivery.



## Step 4: Check CPU credits before generating load

1. In CloudWatch Metrics → EC2 → Per-Instance Metrics, select **CPUCreditBalance** for the same Instance ID.
2. Inspect the latest point with a 5-minute period; Average is suitable for observation.
3. Verify that the instance uses **Standard** credit mode in EC2. Do not switch to Unlimited just to trigger the alarm.
4. For a two-vCPU test lasting up to 15 minutes, approximately **30 CPU credits** is a conservative preparation target, not an AWS requirement or a guarantee of reaching the threshold.

If credits are too low, postpone the test and allow idle time to accumulate credits. Do not repeatedly run load tests. When Standard-mode credits run out, CPU performance is restricted near baseline and may not exceed 80%.

## Step 5: Generate automatically limited load

From your computer, connect as in Section 5.4.1, replacing PUBLIC_IPV4:

```bash
ssh -i "YOUR_KEY_PAIR.pem" ec2-user@PUBLIC_IPV4
```

On EC2, check CPU count and time:

```bash
nproc
date -u
```

The command uses the number of vCPUs returned by `nproc`.

Run **once**:

```bash
sudo systemd-run --unit=workshop-cpu-test \
  --property=RuntimeMaxSec=900 \
  --property=KillMode=control-group \
  /bin/bash -c 'for ((i=0; i<$(nproc); i++)); do yes > /dev/null & done; wait'
```

The yes processes consume CPU without writing output to disk. systemd stops the service after a maximum of 900 seconds (15 minutes); KillMode stops its children too. The command returns immediately. Do not create another unit to overlap tests.

Check:

```bash
sudo systemctl status workshop-cpu-test --no-pager
top
```

Press `q` to exit top; **q does not stop the load**. On a two-vCPU instance, each process may show close to 100% in top, whereas CloudWatch reports instance-level usage using a different calculation.

Stop as soon as ALARM evidence is captured or the website becomes too slow:

```bash
sudo systemctl stop workshop-cpu-test
```

The systemd time limit continues to work if SSH disconnects. If the command reports that the unit already exists, inspect its state instead of starting a duplicate. For an old stopped unit in a failed state, use `sudo systemctl reset-failed workshop-cpu-test` and only retry after identifying the cause.


## Step 6: Observe ALARM and recovery

1. Return to CloudWatch → Alarms → **workshop-web-01-cpu-high**.
2. Select **1 hour** and refresh periodically. Do not reduce the period or change the threshold just to obtain an ALARM screenshot.
3. Check that CPU Average exceeds 80% for the required 2/2 datapoints. A 15-minute run allows at least two complete periods if load is sustained, but does not guarantee success when credits or data delivery are insufficient.
4. When the state becomes **ALARM**, inspect **History** and the state-change reason. Record timestamps, time zone, and evaluated values, then stop the load.
5. If SNS is configured, check the inbox and spam folder. Record email success only when a matching notification is received.



Confirm that the load has stopped:

```bash
sudo systemctl is-active workshop-cpu-test
```

The result must no longer be active. The unit may be failed because RuntimeMaxSec was reached, or it may have been unloaded. Inspect its journal if needed:

```bash
sudo journalctl -u workshop-cpu-test -n 30 --no-pager
```

Wait for lower CPU data to arrive and be evaluated. When the state returns to **OK**, capture History. Do not stop EC2 to create missing data instead of testing recovery. Confirm that website sign-in and document operations still work.



## Troubleshooting

| Symptom | Resolution |
| --- | --- |
| Alarm remains OK | Check actual CPU, Instance ID, Average, 5 minutes, >80 and 2/2; one high point is insufficient |
| CPU remains low while yes runs | Check CPUCreditBalance, Standard mode, and both load processes; do not force success with Unlimited |
| Alarm remains ALARM after stopping | Allow new data to arrive, inspect History and CPU, and check for other workloads |
| Persistent INSUFFICIENT_DATA | Check EC2 Running state, metric, Region, time range, and missing data |
| No email | Check subscription, topic, action, publish permissions, and spam; staying in ALARM does not generate continuous emails |
| Unit failed after 15 minutes | This may be the intentional timeout; inspect the journal and confirm load stopped |
| AccessDenied when creating the alarm | Check PutMetricAlarm on the exact name/ARN and read permissions; do not grant AdministratorAccess |

If ALARM is not reached within 15 minutes, stop, save the actual outcome, and investigate. Do not report success. Do not use SetAlarmState as proof that CPU crossed the threshold: manually changing state does not verify the real metric pipeline.

## Results and handover

Record the Instance ID, alarm name, settings, load start/stop times, ALARM/OK timestamps, and SNS outcome if applicable. Keep the alarm for Section 5.7 and remove it during Section 5.8 cleanup. 

Continue: [5.6.3. Collect and Verify Application Logs](../5.6.3-application-logs/).

## References

- [Create a CPU usage alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/US_AlarmAtThresholdEC2.html)
- [Alarm evaluation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html)
- [CPU credits and baseline](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-credits-baseline-concepts.html)
- [SNS notifications for alarm changes](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Notify_Users_Alarm_Changes.html)
