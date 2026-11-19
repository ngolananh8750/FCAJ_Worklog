---
title: "Week 7 Worklog"
weight: 7
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.7 </b>"
---



### Objectives for Week 7:

* Manage server access permissions using IAM Roles (Instance Profile).
* Enhance security by managing EC2 via AWS SSM Session Manager without opening Port 22.

### Tasks to be executed this week:

| Day | Task | Start Date | Completion Date | Reference |
| :--- | :--- | :--- | :--- | :--- |
| Mon | Create IAM Role with `AmazonSSMManagedInstanceCore` policy and attach it to EC2 Instance Profile | | | |
| Wed | Verify EC2 session connection via SSM Session Manager and completely close Port 22 on Security Group | | | |

### Results achieved in Week 7:

* Completely blocked public SSH port (Port 22), eliminating brute-force attack risks.
* Managed server securely from AWS Console or AWS CLI via secure channel.