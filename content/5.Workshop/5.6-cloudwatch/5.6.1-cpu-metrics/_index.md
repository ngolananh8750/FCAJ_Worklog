---
title: "Monitor EC2 CPU Metrics"
date: 2026-10-31
weight: 1
chapter: false
pre: "<b>5.6.1. </b>"
---

## Objective

Member 4 uses Amazon CloudWatch to view **CPUUtilization** for **workshop-web-01**, apply consistent graph settings, and record a baseline before testing the alarm in Section 5.6.2.

This section only examines existing EC2 metrics. It does not install CloudWatch Agent or generate CPU load.

## Prerequisites

- Complete the prerequisites in [5.6. Monitoring and Alerts with CloudWatch](../).
- The workshop-web-01 EC2 instance is running in **Singapore — ap-southeast-1**.
- The operator can view EC2 and read CloudWatch metrics, including required actions such as `ec2:DescribeInstances`, `cloudwatch:ListMetrics`, `cloudwatch:GetMetricData`, and `cloudwatch:GetMetricStatistics`.
- Use **Basic monitoring** as agreed for the workshop: CPU data is available at 5-minute intervals. Do not enable Detailed monitoring solely for this exercise.

EC2 automatically publishes standard CPU metrics to CloudWatch. Viewing this metric does not require PutMetricData permissions on the EC2 IAM Role, additional inbound ports, or an installed agent. Agent log-writing permissions are addressed separately in Section 5.6.3.

## Step 1: Identify the correct EC2 instance

1. Sign in to the AWS Management Console using the assigned monitoring account.
2. Select **Asia Pacific (Singapore) — ap-southeast-1**.
3. Find and open **EC2**.
4. Choose **Instances** in the left navigation pane.
5. Find **workshop-web-01** and select its row.
6. Confirm **Instance state = Running** and record the **Instance ID**, beginning with `i-...`.
7. If multiple instances share the Name tag, compare their VPC, subnet, and the ID provided by Member 2. Do not identify the instance by name alone.

![](/FCAJ_Worklog/images/55/ec2m1.png)


## Step 2: Preview CPU in EC2

1. Keep the instance selected and open the **Monitoring** tab in the details pane.
2. Find the **CPU utilization** or **CPUUtilization** graph.
3. Select **1 hour**, or **3 hours** for a wider view if the instance has been running longer.
4. Refresh and observe the data. Newly launched instances need time for data to appear; an empty graph does not mean CPU usage is zero.
5. Keep Basic monitoring. If the Console offers Detailed monitoring, do not enable it for this section. If it was already enabled, coordinate with the EC2 operator before changing the setting.

![](/FCAJ_Worklog/images/55/ec2m2.png)


## Step 3: Select the metric in CloudWatch

1. Open **CloudWatch** in a new tab and keep **ap-southeast-1** selected.
2. In the left navigation pane, open **Metrics → All metrics**.
3. In the Browse area, select **EC2 (AWS/EC2)**.
4. Choose **Per-Instance Metrics**.
5. Paste the **actual Instance ID** from Step 1 into the search field and press Enter if needed.
6. Find the row with **Metric name = CPUUtilization** and the correct **InstanceId**.
7. Select its checkbox to graph the metric. Deselect other metrics if any are selected.

Console labels may change slightly. Always verify **AWS/EC2**, **CPUUtilization**, and **InstanceId**. Avoid selecting an aggregate graph by InstanceType or ImageId.

## Step 4: Configure the graph

1. Select **Graphed metrics**.
2. Set **Statistic = Average** on the CPUUtilization row.
3. Set **Period = 5 minutes (300 seconds)**. If Auto is available, explicitly select 5 minutes.
4. Choose the **Line** visualization if another type is active.
5. Set the time range above the graph to **1 hour** or **3 hours**.
6. Check the displayed time zone. UTC or your local time zone is acceptable, but record it with the results. Vietnam uses UTC+7.
7. Choose Refresh. If you enable automatic refresh every minute, remember that this does not turn Basic monitoring into one-minute metric collection.

| Setting | Required value |
| --- | --- |
| Namespace | AWS/EC2 |
| Metric | CPUUtilization |
| Dimension | InstanceId = actual ID of workshop-web-01 |
| Statistic | Average |
| Period | 5 minutes |
| Unit | Percent (%) |
| Time range | 1 hour or 3 hours, including recent data |

![](/FCAJ_Worklog/images/55/ec2m3.png)


## Step 5: Read the data and record a baseline

1. Hover over an existing data point on the graph.
2. Read its timestamp and CPU percentage.
3. Record at least **three consecutive data points** if available. If there are not enough yet, continue observing after updates. Do not invent measurements.
4. While observing, you may sign in to the website and upload/download a small sample document as normal user activity. Record the time if you perform these actions.
5. A single file operation does not have to cause a noticeable CPU increase. A small application and a 5-minute average can make changes difficult to see. A controlled load test belongs in Section 5.6.2.

| Timestamp and time zone | CPU Average (%) | System activity |
| --- | --- | --- |
| Enter actual data | Enter actual data | For example: no website actions yet |
| Enter actual data | Enter actual data | Record actual activity |
| Enter actual data | Enter actual data | Record actual activity |

**Average** represents the average over the evaluation interval, not an individual process's instantaneous CPU usage. With Basic monitoring, selecting a 1-minute Period does not create additional one-minute CPU samples. Operating system tools such as top may report different values because of measurement methods and sampling intervals.

Low CPU usage does not prove that the website, S3 connectivity, or authentication is working correctly. Combine metrics with functional tests and logs in subsequent sections.



## Step 6: Verify and hand over

- Confirm that the graph uses the correct Instance ID, Average, and 5-minute period.
- Save the four actual screenshots and the CPU observation table.
- Record the Region, Instance ID, observation time range, and time zone.
- Keep these metric details for creating **workshop-web-01-cpu-high** in the next section. A separate dashboard is not required to complete this exercise.

## Troubleshooting

| Symptom | Check and resolution |
| --- | --- |
| Instance or metric not found | Check the account, Region, Instance ID, and active filters |
| Empty graph immediately after launch | Allow time for data delivery, refresh, and widen the time range; do not assume CPU = 0 |
| Only old data appears | Verify that the instance is still Running and the range includes the present |
| Multiple CPUUtilization rows | Select the correct InstanceId dimension and deselect other instances |
| AccessDenied | Record the denied action and ask the administrator for appropriate read access, not full administrator permissions |
| Data timestamps do not match actions | Check UTC versus UTC+7 and compare using the same time zone |
| Minute-by-minute refresh shows little change | Basic monitoring still publishes CPU at 5-minute intervals |
| CPU stays low after upload/download | This can be normal for small files; it is not yet a load or alarm test |

## Completion criteria

A CPUUtilization graph for the correct workshop-web-01 and actual baseline observations.

Continue: [5.6.2. Create and Test a CPU Alarm](../5.6.2-cpu-alarm/).

## References

- [View available metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/viewing_metrics_with_cloudwatch.html)
- [EC2 metrics and dimensions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/viewing_metrics_with_cloudwatch.html)
- [Basic and detailed monitoring](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/manage-detailed-monitoring.html)
