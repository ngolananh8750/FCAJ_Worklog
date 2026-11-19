---
title: "Week 6 Worklog"
weight: 6
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.6 </b>"
---



### Objectives for Week 6:

* Implement HTTPS traffic encryption for Web Server using free SSL/TLS certificates.
* Automate HTTPS redirection and certificate renewal using Certbot (Let's Encrypt).

### Tasks to be executed this week:

| Day | Task | Start Date | Completion Date | Reference |
| :--- | :--- | :--- | :--- | :--- |
| Mon | Study SSL/TLS protocols and data transmission encryption mechanisms | | | |
| Tue | Install Certbot and `python3-certbot-nginx` plugin on the EC2 instance | | | |
| Wed | Provision free SSL/TLS certificate from Let's Encrypt for the domain name | | | |
| Thu | Configure Nginx automatic HTTP to HTTPS (301) redirect and set up auto-renewal schedule (`certbot renew`) | | | |

### Results achieved in Week 6:

* Encrypted $100\%$ of Web Server traffic via secure HTTPS protocol.
* Automated periodic SSL/TLS certificate renewal process.