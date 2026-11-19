---
title: "Launch EC2 and Connect Using SSH"
date: 2026-10-30
weight: 1
chapter: false
pre: "<b>5.4.1. </b>"
---



## Objective

Member 2 launches **workshop-web-01** in the prepared network and connects using SSH from a local computer. This section covers instance creation and verification only; web server installation follows in 5.4.2.

## Prerequisites

- Complete section 5.3 and verify the IDs of vpc-secure-web, subnet-web-public, and sg-secure-web.
- Region: **Singapore — ap-southeast-1**.
- The IAM user needs permissions to create EC2 instances, key pairs, EBS volumes, tags, and use the instance profile; limit iam:PassRole to the approved role.
- The administrator has prepared **WorkshopEC2AppRole** and **WorkshopEC2AppRole**. The actual profile name may differ when created through the console; confirm that it contains the correct role.
- Install an OpenSSH client locally: Terminal on Linux/macOS or PowerShell on Windows. Check with `ssh -V`.
- The SSH computer's public IPv4/32 is allowed in both the security group and NACL. Update it after changing networks/VPNs.

## Step 1: Choose the name, AMI, and instance type

Open **EC2 → Instances → Launch instances** and configure:

| Field | Lab value |
| --- | --- |
| Name | workshop-web-01 |
| AMI | Standard Amazon Linux 2023 provided by Amazon |
| Architecture | 64-bit (x86) / x86_64 |
| Instance type | t3.micro |
| Number of instances | 1 |
| Tags | Project=SecureWebWorkshop |

This is the implementation choice for subsequent sections, using the **ec2-user** operating system account and **dnf** package manager. Do not choose a Marketplace or Arm AMI for t3.micro. Record the actual AMI ID, which varies by Region and version.

Check costs and account benefits before launch; a Free tier label does not mean the entire lab is free. If t3.micro is unavailable to your account, agree on an alternative x86_64 instance type with the administrator before continuing.

![](/FCAJ_Worklog/images/54/ec21.png)


## Step 2: Create or select a key pair

1. Under **Key pair (login)**, select **Create new key pair** if no valid key is available.
2. Enter **workshop-web-key**, select **RSA**, and choose **.pem** format.
3. Select **Create key pair** and save the downloaded file in a private directory on the SSH computer.
4. Confirm workshop-web-key is selected in the launch form.

Reuse an existing key pair only if you possess its matching private key. AWS does not allow downloading an existing key pair's private key again. Do not proceed without a key pair for this SSH method. Never upload the .pem file to Git, the workshop website, or screenshots.

![](/FCAJ_Worklog/images/54/ec23.png)


## Step 3: Select networking and the security group

Under **Network settings → Edit**, configure:

| Field | Value |
| --- | --- |
| VPC | vpc-secure-web |
| Subnet | subnet-web-public |
| Auto-assign public IP | Enable |
| Firewall | Select existing security group |
| Security group | sg-secure-web |

Verify actual IDs, not just names. Do not create a new security group or allow SSH from 0.0.0.0/0. This step does not require an Elastic IP.

![](/FCAJ_Worklog/images/54/ec22.png)


## Step 4: Configure storage and security

1. Under **Configure storage**, use one **8 GiB gp3** root volume if supported by the selected AMI; never reduce it below the AMI minimum.
2. Expand the volume's advanced settings, enable **Encrypted**, use the default **aws/ebs** key for the lab, and retain **Delete on termination** so the root volume is removed when terminating the instance.
3. Open **Advanced details → IAM instance profile** and select the profile containing **WorkshopEC2AppRole**.
4. Keep the metadata endpoint Enabled; select **Metadata version: V2 only (token required)** to require IMDSv2.
5. Set **Credit specification** to **Standard** to avoid Unlimited surplus CPU credit charges. Performance may fall to baseline when credits run out.
6. Leave detailed CloudWatch monitoring disabled at this stage and **User data** empty. Do not enter passwords, private keys, or access keys in User data.

If the instance profile is missing, ask the administrator to check the profile, role, and listing/PassRole permissions. Do not substitute an arbitrary broadly privileged role. Specific S3 permissions are completed in 5.5.3.

![](/FCAJ_Worklog/images/54/ec24.png)


![](/FCAJ_Worklog/images/54/ec26.png)

## Step 5: Launch and verify the instance

1. Review all settings, confirm Number of instances = **1**, and select **Launch instance**.
2. Select **View all instances** and open the newly created workshop-web-01.
3. Wait for **Instance state: Running** and all displayed status checks to pass.
4. Record the Instance ID, public IPv4, private IPv4, Availability Zone, and AMI ID.
5. Verify sg-secure-web and the IAM role under **Security**, and the correct VPC/subnet under **Networking**.

If no public IPv4 appears, check the launch configuration. Do not use the private IPv4 for direct SSH from the Internet. Do not test the website yet because no web service has been installed.

![](/FCAJ_Worklog/images/54/ec27.png)


## Step 6: Connect using SSH from your computer

Run these commands **on your local computer**, from the directory containing YOUR_KEY_PAIR.pem. Replace **PUBLIC_IPV4** with the instance's actual public IPv4 address.

**Linux/macOS:**

```bash
chmod 400 YOUR_KEY_PAIR.pem
ssh -i "YOUR_KEY_PAIR.pem" ec2-user@PUBLIC_IPV4
```

**Windows PowerShell with OpenSSH:**

```powershell
ssh -i ".\YOUR_KEY_PAIR.pem" ec2-user@PUBLIC_IPV4
```

If Windows reports overly permissive private key access, open the .pem file's Properties → Security → Advanced, disable inheritance, and retain only the required read access for your Windows account. Do not grant access to Everyone/Users. Then reconnect.

On the first connection, SSH asks you to confirm the host fingerprint. Compare it with the **instance** fingerprint from EC2 → Actions → Monitor and troubleshoot → Get system log, looking for SSH HOST KEY FINGERPRINTS where available. Do not confuse it with the key pair fingerprint. Enter `yes` only after verifying the instance; ask the administrator to assist if the fingerprint cannot be obtained.

After logging in, run these read-only commands:

```bash
whoami
cat /etc/os-release
hostname
```

Expected results: user **ec2-user** and **Amazon Linux 2023**. Do not print metadata credentials for screenshots. Enter `exit` to end the SSH session.

![](/FCAJ_Worklog/images/54/ec28.png)

## Common issues

| Symptom | Check/action |
| --- | --- |
| Connection timed out | Verify public IPv4, IGW route, SSH /32 in SG/NACL, and NACL outbound ephemeral ports; check whether the local network blocks TCP 22 |
| Permission denied (publickey) | Verify ec2-user, the matching private key for the launch key pair, and the target instance |
| Unprotected private key file | Fix local file permissions; use chmod 400 on Linux/macOS |
| Host key changed | Check for changed IP or recreated instance; verify its fingerprint before updating known_hosts, never disable host key checks |
| AccessDenied during launch | Record the denied action; check EC2/EBS/tag/PassRole permissions and policy conditions |
| Instance immediately stops/terminates | Check State transition reason, EBS encryption permissions, and AMI/instance configuration |
| Browser EC2 Instance Connect fails | This guide uses direct SSH from your computer; browser connections have different network sources, so do not open SSH globally to fix them |

## Results and handover

Record Instance ID, AMI ID, public/private IPv4, VPC/Subnet IDs, Security Group ID, role/profile, and key pair name. Never document the private key. Add screenshots at the **7 marked locations**; some locations may need more than one image.

When pausing the lab, Stop the instance to reduce running time; EBS remains and may incur charges. An automatically assigned public IPv4 can change after Stop/Start: recheck before SSH. Clean up all resources in 5.8.

Continue to [5.4.2. Install the Web Server](../5.4.2-install-web-server/).

## References

- [Launch an EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-instance-wizard.html)
- [Instance configuration parameters](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-launch-parameters.html)
- [SSH connection prerequisites](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs-general.html)
- [Connect using an SSH client](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-ssh.html)
- [Configure CPU credit mode](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-how-to.html)
- [Configure IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-IMDS-new-instances.html)
