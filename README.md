# 🔐 AWS EC2 Security Hardening Project

> **Full production-grade security hardening of an AWS EC2 Ubuntu 22.04 instance — managed entirely via PowerShell and AWS CLI on Windows.**

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Phase 1 — Environment Setup](#phase-1--environment-setup)
- [Phase 2 — EC2 Instance Launch](#phase-2--ec2-instance-launch)
- [Phase 3 — SSH Hardening](#phase-3--ssh-hardening)
- [Phase 4 — UFW Firewall](#phase-4--ufw-firewall)
- [Phase 5 — Monitoring & Stability](#phase-5--monitoring--stability)
- [Verification Results](#verification-results)
- [Security Checklist](#security-checklist)
- [Technologies Used](#technologies-used)

---

## Project Overview

This project demonstrates end-to-end security hardening of a Linux server on AWS, covering:

- **SSH hardening** — port change, key-only auth, Fail2ban brute-force protection
- **Firewall configuration** — UFW with default-deny, rate limiting, and IP blocking
- **Monitoring** — CloudWatch agent shipping auth/syslog/UFW logs with CPU & status alarms
- **Stability** — swap space, automatic security updates

All provisioning and configuration is driven from **PowerShell 7** on Windows using **AWS CLI v2** and SSH remote commands — no manual console clicking.

---

## Architecture

```
  ┌─────────────────────────────────────────────────────┐
  │              Windows Machine (Local)                │
  │   PowerShell 7+  ──►  AWS CLI v2  ──►  SSH Client  │
  └────────────────────────┬────────────────────────────┘
                           │ HTTPS / SSH (port 2222)
                    ┌──────▼──────┐
                    │   Internet   │
                    └──────┬──────┘
              ┌────────────▼────────────┐
              │      AWS VPC / SG        │
              │   ┌─────────────────┐   │
              │   │  EC2 Ubuntu     │   │
              │   │  t3.micro       │   │
              │   │  SSH :2222      │   │
              │   │  UFW Firewall   │   │
              │   │  Fail2ban       │   │
              │   │  CloudWatch     │   │
              │   └─────────────────┘   │
              └─────────────────────────┘
```

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| PowerShell | 7+ | Main management shell |
| AWS CLI | v2 | Provision AWS resources |
| OpenSSH Client | Built-in | Connect to EC2 |

```powershell
# Install PowerShell 7
winget install --id Microsoft.PowerShell --source winget

# Install AWS CLI v2
Invoke-WebRequest -Uri https://awscli.amazonaws.com/AWSCLIV2.msi -OutFile AWSCLIV2.msi
Start-Process msiexec.exe -ArgumentList '/i AWSCLIV2.msi /quiet' -Wait

# Enable OpenSSH Client (as Administrator)
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

---

## Phase 1 — Environment Setup

Configure AWS credentials:

```powershell
aws configure
# AWS Access Key ID:     <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name:   us-east-1
# Default output format: json

# Verify
aws sts get-caller-identity
```

---

## Phase 2 — EC2 Instance Launch

```powershell
# 1. Create key pair
$keyName = 'hardening-key'
aws ec2 create-key-pair --key-name $keyName --query 'KeyMaterial' --output text `
  | Out-File -Encoding ascii .\$keyName.pem
icacls .\$keyName.pem /inheritance:r /grant:r "$($env:USERNAME):R"

# 2. Create security group (SSH restricted to your IP only)
$myIp = (Invoke-RestMethod https://checkip.amazonaws.com).Trim()
$vpcId = (aws ec2 describe-vpcs --query 'Vpcs[0].VpcId' --output text)
$sgId = (aws ec2 create-security-group --group-name 'hardening-sg' `
  --description 'Hardened EC2 SG' --vpc-id $vpcId --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $sgId `
  --protocol tcp --port 22 --cidr "$myIp/32"

# 3. Launch Ubuntu 22.04 instance
$amiId = (aws ec2 describe-images --owners 099720109477 `
  --filters 'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64*' `
            'Name=state,Values=available' `
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

$instanceId = (aws ec2 run-instances --image-id $amiId --instance-type t3.micro `
  --key-name hardening-key --security-group-ids $sgId `
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=hardened-server}]' `
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $instanceId
$publicIp = (aws ec2 describe-instances --instance-ids $instanceId `
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
```

---

## Phase 3 — SSH Hardening

Key hardening settings applied to `/etc/ssh/sshd_config`:

| Setting | Value | Purpose |
|---------|-------|---------|
| `Port` | `2222` | Non-default port |
| `PermitRootLogin` | `no` | Block root access |
| `PasswordAuthentication` | `no` | Key-only login |
| `LoginGraceTime` | `30` | Limit login window |
| `MaxAuthTries` | `3` | Limit retry attempts |
| `X11Forwarding` | `no` | Disable X11 |
| `LogLevel` | `VERBOSE` | Detailed audit trail |

```powershell
# Push hardened config remotely, update SG port, install Fail2ban
ssh -i .\hardening-key.pem ubuntu@$publicIp 'sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup'

# After applying config, update Security Group
aws ec2 authorize-security-group-ingress --group-id $sgId --protocol tcp --port 2222 --cidr "$myIp/32"
aws ec2 revoke-security-group-ingress  --group-id $sgId --protocol tcp --port 22  --cidr "$myIp/32"
```

**Fail2ban config** (`/etc/fail2ban/jail.local`):
```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port    = 2222
```

---

## Phase 4 — UFW Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH Hardened'
sudo ufw allow 80/tcp   comment 'HTTP'
sudo ufw allow 443/tcp  comment 'HTTPS'
sudo ufw limit 2222/tcp comment 'SSH Rate Limit'
sudo ufw logging on
sudo ufw --force enable
```

**Verified UFW status from the live instance:**

```
Status: active
Logging: on (medium)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
2222/tcp                   LIMIT IN    Anywhere     # SSH Rate Limit
80/tcp                     ALLOW IN    Anywhere     # HTTP
443/tcp                    ALLOW IN    Anywhere     # HTTPS
```

---

## Phase 5 — Monitoring & Stability

### CloudWatch Agent
Collects and ships to AWS CloudWatch:
- `/var/log/auth.log` → log group `/ec2/auth`
- `/var/log/syslog`   → log group `/ec2/syslog`
- `/var/log/ufw.log`  → log group `/ec2/ufw`
- Metrics: CPU, Memory, Disk

### CloudWatch Alarms (verified active)
| Alarm | Metric | Threshold | State |
|-------|--------|-----------|-------|
| High-CPU | CPUUtilization | > 80% for 10 min | ✅ OK |
| Instance-Status-Check | StatusCheckFailed | ≥ 1 for 3 min | ✅ OK |

### Swap Space
```
Swap: 2.0Gi total, 0B used, 2.0Gi free
```

### Automatic Security Updates
```bash
sudo apt install -y unattended-upgrades
# Auto-reboot for security patches scheduled at 03:00 UTC
```

---

## Verification Results

Full audit run from PowerShell — all checks passed:

```
[SSH port changed]:    LISTEN 0  128  0.0.0.0:2222  (sshd,pid=2243)
[Root login disabled]: PermitRootLogin no
[Password auth off]:   PasswordAuthentication no
[UFW enabled]:         Status: active
[Fail2ban running]:    active (1 jail: sshd)
[CW agent running]:    active (running) since 2026-05-08
[Auto-updates on]:     active (running) since 2026-05-08
```

Login banner displayed on every SSH connection:
```
**********************************************
* AUTHORIZED ACCESS ONLY                    *
* All activity is monitored and logged.     *
**********************************************
```

---

## Security Checklist

- [x] SSH default port 22 closed
- [x] SSH running on port 2222
- [x] Root SSH login disabled
- [x] Password authentication disabled (key-only)
- [x] SSH login grace time limited (30s)
- [x] Max auth attempts limited (3)
- [x] Login warning banner configured
- [x] Fail2ban active (1h ban, 3 retries)
- [x] UFW default deny incoming
- [x] UFW rate limiting on SSH
- [x] UFW logging enabled (medium)
- [x] CloudWatch agent shipping logs
- [x] CPU alarm configured
- [x] Status check alarm configured
- [x] Automatic security updates enabled
- [x] 2GB swap space configured
- [x] EC2 IAM role with CloudWatchAgentServerPolicy

---

## Technologies Used

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20CloudWatch%20%7C%20IAM-orange?logo=amazonaws)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20LTS-E95420?logo=ubuntu)
![PowerShell](https://img.shields.io/badge/PowerShell-7%2B-5391FE?logo=powershell)
![UFW](https://img.shields.io/badge/Firewall-UFW-red)
![Fail2ban](https://img.shields.io/badge/IDS-Fail2ban-yellow)

| Layer | Technology |
|-------|-----------|
| Cloud Provider | AWS (EC2, CloudWatch, IAM, VPC) |
| OS | Ubuntu 22.04.5 LTS |
| Management Shell | PowerShell 7 + AWS CLI v2 |
| SSH Daemon | OpenSSH with hardened config |
| Firewall | UFW (Uncomplicated Firewall) |
| Intrusion Detection | Fail2ban |
| Monitoring | Amazon CloudWatch Agent |
| Auto-patching | unattended-upgrades |

---

> **Note:** AWS credentials visible in terminal history should be rotated immediately after use. For production, use IAM roles attached to the instance instead of access keys.


## 👤 Author

**Abdelatef Mohamed**
DevOps Engineer

[![GitHub](https://img.shields.io/badge/GitHub-abdelatif2030-181717?logo=github)](https://github.com/abdelatif2030)
[![Upwork](https://img.shields.io/badge/Upwork-Hire%20Me-6FDA44?logo=upwork)](https://www.upwork.com/freelancers/abdelatif2030)

> Built with PowerShell, AWS CLI, and a passion for secure infrastructure.