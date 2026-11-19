---
title: "Project Proposal - Secure Web Server on AWS"
weight: 2
date: "` Sys.Date()`"
chapter: false
pre : "<b> 2. </b>"
---



## Tiered Security Architecture and Cost-Optimized Operation on AWS

---

### 1. Executive Summary

This proposal presents the architectural design and deployment plan for a **production-ready, enterprise-grade secure Web Server** on Amazon Web Services (AWS). Rather than deploying an isolated EC2 instance in a default network—which introduces significant security risks and potential uncontrolled costs—this solution adopts a secure network partitioning model, isolating the compute environment and automating operational management.

The entire infrastructure is built on a dedicated **AWS VPC** with clear Public and Private Subnet isolation. The **Amazon EC2** virtual server runs Linux (Ubuntu 22.04 LTS / Amazon Linux 2023) hosting an **Nginx Web Server**, with $100\%$ end-to-end traffic encryption powered by free **SSL/TLS (HTTPS)** certificates from Let's Encrypt. The system adheres strictly to the Principle of Least Privilege via **IAM Roles**, secures network ports using **Security Groups**, and automates source code deployments through a **GitHub Actions CI/CD Pipeline**. The solution is engineered to operate fully within the **AWS Free Tier** limit ($\sim 0\text{ USD/month}$).

---

### 2. Problem Statement

#### Current Issues
* **Security Risks & Unrestricted Public Ports:** Default EC2 setups frequently expose administrative ports such as SSH (Port 22) and HTTP (Port 80) to the entire Internet (`0.0.0.0/0`), creating high exposure to automated brute-force attacks and vulnerability exploitation.
* **Unencrypted Network Traffic (Plaintext HTTP):** Operating under unencrypted HTTP leaves data in transit vulnerable to eavesdropping and Man-in-the-Middle (MITM) attacks.
* **Uncontrolled Cloud Spending:** New cloud administrators can easily exceed Free Tier boundaries due to misconfigurations (e.g., provisioning NAT Gateways, unattached Elastic IPs, or exceeding EBS storage limits).
* **Manual Management & Operational Overhead:** Manually SSH-ing into servers to pull code or install dependencies causes unnecessary downtime and introduces human error.

#### Proposed Solution
The system is architected around two core principles: **Defense in Depth** and **Cost Optimization by Design**.

The infrastructure is organized into 4 independent operational flows:
1. **Flow C (GitOps Deployment Loop):** Automated CI/CD via GitHub Actions to build, test, and deploy code to Nginx via SSH/SSM without user interruption.
2. **Flow A (User Access & SSL/TLS Routing):** Routes user traffic from the Internet through CloudFront/WAF to Nginx, automatically enforcing HTTP $\rightarrow$ HTTPS redirection and TLS termination.
3. **Flow S (Security & Access Boundary):** Strictly controls server administration. Locks Port 22 or restricts it to a static management IP, with a recommendation to use AWS Systems Manager (SSM) Session Manager for keyless, portless access.
4. **Flow M (Monitoring & Cost Control Loop):** Exports system logs to CloudWatch and triggers automated email alerts via AWS Budgets when spending reaches $\$0.80\text{ USD}$.

#### Key Benefits & Return on Investment (ROI)
* **Optimal Operating Cost ($\sim 0\text{ USD/month}$):** Fully leverages AWS Free Tier allowances (750 EC2 hours, 30 GB EBS, 100 GB Data Out). Eliminates high-cost services like permanent NAT Gateways or Application Load Balancers.
* **Enterprise-Grade Security:** Enforces $100\%$ HTTPS encryption, dynamic port security, and eliminates hardcoded credentials using IAM Instance Profiles.
* **Operational Automation:** Reduces code deployment time by over $90\%$ using a CI/CD Pipeline.

---

### 3. Solution Architecture

#### Architecture Blueprint

```text
                                +-----------------------------------------------------------------------------------+
                                |                                  AWS Cloud VPC                                    |
                                |                                                                                   |
[ Internet User ] ------------->|===> [ CloudFront / AWS WAF ]                                                      |
                                |            |                                                                      |
                                |            v                                                                      |
                                |   [ Internet Gateway ]                                                            |
                                |            |                                                                      |
                                |            v                                                                      |
                                |   +---------------------------------------------------------------------------+   |
                                |   | Public Subnet (10.0.1.0/24)                                               |   |
                                |   |                                                                           |   |
                                |   |   +-------------------------------------------------------------------+   |   |
                                |   |   | Security Group (Inbound: 80, 443, 22-MyIP)                         |   |   |
                                |   |   |                                                                   |   |   |
                                |   |   |   [ EC2 Instance: t2.micro / t3.micro ]                           |   |   |
                                |   |   |   +-----------------------------------------------------------+   |   |   |
                                |   |   |   | Nginx Web Server + Certbot (SSL/TLS)                      |   |   |   |
                                |   |   |   +-----------------------------------------------------------+   |   |   |
                                |   |   +-------------------------------------------------------------------+   |   |
                                |   +---------------------------------------------------------------------------+   |
                                |                                    |                                              |
                                |                                IAM Role                                           |
                                |                                    |                                              |
                                |            +-----------------------+-----------------------+                      |
                                |            v                                               v                      |
                                |   [ AWS CloudWatch Logs ]                         [ AWS Budgets Alarm ]           |
                                +-----------------------------------------------------------------------------------+
```

---

#### Detailed Operational Flows

1. **Flow C — GitOps Deployment Loop (Automated Deployment)**
   * **C1 - C2:** Developer pushes new web code to the GitHub Repository.
   * **C3:** GitHub Actions triggers the workflow to perform linting and build checks.
   * **C4:** Pipeline securely syncs files to `/var/www/html` on the EC2 instance and reloads Nginx smoothly.

2. **Flow A — User Access & SSL/TLS Routing (Routing & Encryption)**
   * **A1:** User sends HTTP (Port 80) or HTTPS (Port 443) requests to the domain name.
   * **A2:** Nginx intercepts the request and issues a 301 Permanent Redirect from HTTP to HTTPS.
   * **A3:** SSL/TLS certificate (issued via Certbot / Let's Encrypt) executes the TLS handshake for encrypted data transmission.
   * **A4:** Nginx serves the static web content back to the user client browser.

3. **Flow S — Security & Identity Management (Access Control)**
   * **S1:** Security Group acts as a stateful firewall, accepting HTTP/HTTPS from `0.0.0.0/0` while restricting SSH (Port 22) exclusively to `My IP`.
   * **S2:** EC2 is attached to an **IAM Role (Instance Profile)** granting log export permissions to CloudWatch without storing Access Keys on disk.
   * **S3 (Advanced):** SSH access can be entirely replaced by **AWS Systems Manager (SSM) Session Manager**, allowing Port 22 to be closed completely.

4. **Flow M — Monitoring & Budget Control (Monitoring & Cost Management)**
   * **M1:** CloudWatch Agent collects CPU, RAM, Disk Utilization metrics, and Nginx Access Logs.
   * **M2:** AWS Budgets continuously tracks account expenditure and sends an immediate email alert if forecasted spend exceeds $\$0.80\text{ USD}$.

---

#### AWS Services Utilized

* **AWS VPC & Networking:** Dedicated Virtual Private Cloud, Subnets, Route Tables, and Internet Gateway.
* **Amazon EC2:** Virtual compute instance hosting Nginx (`t2.micro` or `t3.micro`).
* **AWS Security Group:** Virtual firewall managing stateful inbound/outbound rules.
* **AWS IAM:** Access management and secure role delegation for EC2 instances.
* **Amazon CloudWatch:** Centralized metric monitoring and log management.
* **AWS Budgets:** Automated spend tracking and threshold alerting.
* **Amazon CloudFront & AWS WAF (Optional):** Global Content Delivery Network (CDN) and Web Application Firewall for edge acceleration and DDoS mitigation.

---

### 4. Technical Implementation

#### Implementation Stages

1. **Stage 1: Network Infrastructure & Security Design (Week 1)**
   * Provision custom VPC (`10.0.0.0/16`), Public Subnet (`10.0.1.0/24`), and attach Internet Gateway.
   * Configure Route Table directing default internet traffic (`0.0.0.0/0`) via Internet Gateway.
   * Create Security Group enforcing Least Privilege rules (HTTP 80, HTTPS 443, SSH 22 restricted to `My IP`).

2. **Stage 2: Compute Provisioning & IAM Access (Week 2)**
   * Launch Free Tier eligible EC2 instance (`t2.micro` / `t3.micro`, Ubuntu 22.04 LTS).
   * Generate SSH Key Pair (`.pem`) and restrict file permissions (`chmod 400`).
   * Create IAM Role with `CloudWatchAgentServerPolicy` and attach to EC2.

3. **Stage 3: Web Server Installation & HTTPS Setup (Week 3)**
   * Connect via SSH, install and optimize Nginx:
     ```bash
     sudo apt update && sudo apt install nginx -y
     sudo systemctl enable --now nginx
     ```
   * Point Domain A Record to the EC2 Public IP address.
   * Install Certbot and request a free Let's Encrypt SSL/TLS certificate:
     ```bash
     sudo apt install certbot python3-certbot-nginx -y
     sudo certbot --nginx -d your-domain.com
     ```

4. **Stage 4: Automated CI/CD Deployment Pipeline (Week 4)**
   * Set up GitHub Repository containing web application source code.
   * Create GitHub Actions workflow to auto-deploy code updates to `/var/www/html` upon merges into `main`.

5. **Stage 5: Security Hardening, Audit & Cost Governance (Weeks 5 - 6)**
   * Configure Nginx Security Headers (`X-Frame-Options`, `X-Content-Type-Options`, `CSP`).
   * Configure AWS Budget alarm at a $\$1.00\text{ USD}$ threshold.
   * Perform security evaluation (Port Scans, SSL Labs Audit) and finalize documentation.

---

#### Technical & Security Requirements

* **100% Traffic Encryption:** Enforce HTTPS (TLS 1.2/1.3) across all user connections.
* **Server Hardening:** Disable Nginx signature display (`server_tokens off;`), block access to hidden files (`.git`, `.env`).
* **Secure Key Management:** Never commit Private Keys (`.pem`) or AWS Access Keys to repository control.
* **Automated SSL Renewal:** Schedule cron job executing `certbot renew --dry-run` to maintain continuous certificate validity.

---

### 5. Implementation Roadmap & Milestones

```text
+-----------------------------------------------------------------------------------+
| Weeks 1 - 2: Network Infrastructure & Security Design                             |
|   - Establish isolated VPC & Route Table                                          |
|   - Configure Security Groups adhering to Least Privilege                         |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| Weeks 3 - 4: EC2 Deployment, Nginx Web Server & HTTPS Encryption                  |
|   - Launch EC2 Instance (Ubuntu 22.04 LTS) attached to IAM Role                   |
|   - Install Nginx, map Domain, and issue Let's Encrypt SSL/TLS Certificate        |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| Weeks 5 - 6: CI/CD Automation, Cost Governance & Report Packaging                 |
|   - Automate deployment via GitHub Actions Pipeline                               |
|   - Configure AWS Budgets & CloudWatch Alarms                                     |
|   - Execute security auditing (SSL Labs Grade A+) & finalize Internship Report    |
+-----------------------------------------------------------------------------------+
```

---

### 6. Estimated Budget

By avoiding expensive standalone resources (such as NAT Gateways or Application Load Balancers) and staying within the **AWS Free Tier** limits, the estimated monthly operating cost is:

| AWS Service / Resource | Configuration / Scale | Estimated Monthly Cost |
| :--- | :--- | :---: |
| **Amazon EC2** | 1 x `t2.micro` / `t3.micro` (750 Hours/Month) | **$0.00 USD** (Free Tier) |
| **Amazon EBS Storage** | 30 GB GP3 Storage | **$0.00 USD** (Free Tier) |
| **AWS Data Transfer** | < 100 GB Data Out to Internet / Month | **$0.00 USD** (Free Tier) |
| **Amazon CloudWatch** | Basic Metrics & Log Collection (< 5 GB) | **$0.00 USD** (Free Tier) |
| **SSL/TLS Certificate** | Certbot / Let's Encrypt | **$0.00 USD** (Open Source) |
| **AWS Budgets** | 2 Spending Alarms | **$0.00 USD** (Free) |
| **Total Estimated Cost** | **Secure Web Server Infrastructure on AWS** | **~ $0.00 USD / Month** |

> **Cost Optimization Key Highlights:**
> 1. No **NAT Gateway** required (saves $\sim \$32\text{ USD/month}$).
> 2. No permanent **Application Load Balancer** required (saves $\sim \$20\text{ USD/month}$).
> 3. No idle **Elastic IP** holding charges.

---

### 7. Risk Analysis & Mitigation Strategies

| Potential Risk | Impact | Likelihood | Mitigation Strategy |
| :--- | :---: | :---: | :--- |
| **SSH Brute-Force Attacks (Port 22)** | High | High | Restrict Port 22 inbound rules to `My IP` strictly, or use **AWS SSM Session Manager** to close Port 22 completely. |
| **Unexpected AWS Spending** | Medium | Medium | Set up **AWS Budgets** to alert via email at $\$0.80\text{ USD}$. Strictly avoid provisioning non-Free Tier resources. |
| **SSL Certificate Expiration** | Medium | Low | Configure a cron job to automatically check and run `certbot renew`. |
| **Credential Leakage in Code Repository** | Critical | Low | Add `.pem` and `.env` files to `.gitignore`. Use GitHub Secrets to store deployment credentials. |

---

### 8. Expected Outcomes

* **Standardized & Secure Infrastructure:** Successfully built Web Server meeting Cloud Security Best Practices, achieving **Grade A+** on SSL Labs Audits.
* **Complete Cost Optimization:** Proven production-level architecture operating at minimal cost ($\sim 0\text{ USD/month}$) within the AWS Free Tier.
* **Reusable Architectural Blueprint:** Fully packaged documentation and deployment guides that can be reused for future web application deployments on AWS.