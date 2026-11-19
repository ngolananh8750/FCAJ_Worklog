---
title: "Create a Dashboard and Send Alerts Through SNS"
weight: 4
chapter: false
pre: "<b>5.6.4. </b>"
---

1. Sign in as `cloudwatch` and switch to `WorkshopMonitoringRole` in Singapore. Open **CloudWatch → Dashboards → Create dashboard** and name it `workshop-web-01`.
2. Add a **Line** widget for `AWS/EC2 → CPUUtilization` using instance `i-0c233adcff2a5a919`, Average and a five-minute period. Add an **Alarm status** widget for `workshop-web-01-cpu-high` and save.
3. Under **SNS → Topics**, create the **Standard** topic `workshop-alerts` if needed. Subscribe a demo email address using **Email** and confirm through the email link. Verify Confirmed status.
4. Open **CloudWatch → All alarms → workshop-web-01-cpu-high → Edit**. Add an **In alarm** notification to the topic and save. Keep Average > 80%, five minutes, and 2/2. Check CloudWatch publish permissions if the topic uses a custom policy.
5. Run the bounded load test in 5.6.2. Compare timestamps in the dashboard, alarm history, and email. If the alarm was already in ALARM before SNS was attached, wait for OK before testing another transition.


[CloudWatch and SNS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Notify_Users_Alarm_Changes.html)
