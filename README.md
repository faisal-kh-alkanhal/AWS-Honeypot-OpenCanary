# 🍯 AWS Honeypot — OpenCanary + CloudWatch

A hands-on cybersecurity project deploying a cloud-based honeypot on AWS using **OpenCanary** for deception services and **AWS CloudWatch** for centralized logging, alerting, and threat analysis, all within the **AWS Free Tier**.

---

## 📌 Project Overview

This project demonstrates the design and deployment of a production-style honeypot infrastructure in AWS. A honeypot is a deliberately exposed decoy system used to detect, log, and study unauthorized access attempts. Because no legitimate user should ever interact with it, every connection is a signal, a port scan, a brute-force attempt, a credential spray.

The goal is to passively collect real-world attacker behavior data (IPs, tools, credential patterns) while practicing core cloud security and infrastructure skills.

---

## 🏗️ Architecture

```
Internet
    │
    ▼
┌─────────────────────────────┐
│                             │
│  AWS VPC                    │
│                             │
│  ┌───────────────────────┐  │
│  │  EC2 t2.small         │  │
│  │  Amazon Linux         │  │
│  │                       │  │
│  │  OpenCanary           │  │
│  │  ├─ Fake SSH :22      │  │
│  │  ├─ Fake HTTP :80     │  │
│  │  ├─ Fake FTP  :21     │  │
│  │  ├─ Fake MySQL :3306  │  │
│  │  └─ Port scan logger  │  │
│  └──────────┬────────────┘  │
└─────────────│───────────────┘
              │
              ▼
     CloudWatch Agent
              │
              ▼
     CloudWatch Logs
     (Log Group: opencanary-honeypot)
              │
        ┌─────┴──────┐
        ▼            ▼
  Metric Filter   Insights
  + Alarm         Queries
        │
        ▼
  SNS Email Alert
```

---

## 🛠️ Technologies Used

| Technology | Role |
|---|---|
| **AWS EC2 (t2.small)** | Host for the honeypot, free tier eligible |
| **AWS VPC** | Isolated network with public subnet, IGW, and routing |
| **AWS Security Groups** | Firewall controlling inbound ports |
| **AWS IAM** | Role-based permissions (no hardcoded credentials) |
| **OpenCanary** | Open-source honeypot, emulates SSH, HTTP, FTP, MySQL |
| **AWS CloudWatch Agent** | Ships OpenCanary logs from EC2 to CloudWatch in real time |
| **AWS CloudWatch Logs** | Centralized log storage and search |
| **AWS CloudWatch Alarms** | Triggers email alerts on brute-force patterns |
| **AWS CloudWatch Insights** | Log analytics and attacker pattern queries |
| **AWS SNS** | Email notification delivery |

---

## 🔐 Security Concepts Demonstrated

- **Deception-based detection**: attracting and identifying threat actors through fake services
- **Principle of least privilege**: IAM role with only the CloudWatch permissions needed, nothing more
- **Network segmentation**: dedicated VPC isolating honeypot traffic from any real workloads
- **Defense in depth**: multiple fake services, port scan detection, and alerting layers
- **Threat intelligence collection**: logging attacker IPs, credentials used, and tools fingerprints
- **Cloud-native SIEM integration**: structured JSON logs queryable in real time via CloudWatch Insights

---
## 🔒 Security Group Configuration

To strictly isolate the honeypot while exposing the necessary deception services to the public internet, a dedicated AWS Security Group was configured with explicit inbound and outbound rules. 

**Inbound Rules (Ingress)**
The inbound configuration allows public traffic only to the specific ports emulated by OpenCanary, ensuring all other potential attack vectors remain tightly closed.

| Protocol | Port Range | Source | Purpose / Emulated Service |
| :--- | :--- | :--- | :--- |
| **TCP** | `22` | `0.0.0.0/0` | Fake SSH service (captures brute-force attempts) |
| **TCP** | `21` | `0.0.0.0/0` | Fake FTP service (captures login attempts) |
| **TCP** | `80` | `0.0.0.0/0` | Fake HTTP service (captures web scanners/crawlers) |
| **TCP** | `3306` | `0.0.0.0/0` | Fake MySQL service (captures database access attempts) |
| **TCP** | `2222` | `My-IP/32` | Secure administrative access to the host EC2 instance |

**Outbound Rules (Egress)**
The outbound rules enforce the principle of least privilege, preventing a compromised instance from being used to launch attacks against external networks.

| Protocol | Port Range | Destination | Purpose |
| :--- | :--- | :--- | :--- |
| **TCP** | `443` | `AWS CloudWatch Endpoints Security Group` | Encrypted transmission of logs via CloudWatch Agent |
---

## 📊 Sample Log Output

Every attacker interaction generates a structured JSON log entry shipped to CloudWatch:

```json
{
    "dst_host": "x.x.x.x", "Instance Private IP"
    "dst_port": 80,
    "local_time": "2026-05-27 21:58:35.173333",
    "local_time_adjusted": "2026-05-27 21:58:35.173357",
    "logdata": {
        "HOSTNAME": "x.x.x.x", "Instance Public IP"
        "PATH": "/index.html",
        "SKIN": "nasLogin",
        "USERAGENT": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/72.0.3626.121 Safari/537.36"
    },
    "logtype": 3000,
    "node_id": "opencanary-aws-honeypot",
    "src_host": "34.105.193.168",
    "src_port": 33280,
    "utc_time": "2026-05-27 21:58:35.173353"
}
```

These logs can then be queried with CloudWatch Insights, for example, finding the top attacking IPs over the past week:

```
fields src_host, logtype
| stats count() as attempts by src_host
| sort attempts desc
| limit 20
```

---

## 📊 Results & Findings

The honeypot remained active for a continuous 24-hour period prior to compiling this report, during which it captured significant background noise, automated scanners, and targeted brute-force attempts from the public internet. 

### Key Metrics
* **Total Logged Events:** `1434`
* **Unique Attacking IPs:** `31`
* **Most Targeted Service:** `SSH (Port 22)`

### Top Attacking Countries
Using CloudWatch Insights to aggregate attacker source IPs, the majority of traffic originated from the following regions:
1. **USA** (`About 70%` of total traffic)
2. **China** (`About 15%` of total traffic)

Notably, most are using cloud providers.

### 🔑 Credential Analysis (Brute-Force Patterns)
The fake SSH and HTTP endpoints captured thousands of automated credential-stuffing attempts. Threat actors heavily favored default administrative credentials, validating the critical real-world need for strong password policies and disabling root logins.

| Emulated Service | Top Usernames Attempted | Top Passwords Attempted |
| :--- | :--- | :--- |
| **SSH (Port 22)** | `root`, `ubuntu`, `admin`, `user`| `123456`, `password`, `admin`, `b` |

## 💰 Cost

**This project runs entirely within the AWS Free Tier.**

---

## ⚠️ Disclaimer

This honeypot is intentionally exposed to the internet to attract real attack traffic. It is designed to be deployed in an **isolated AWS account or VPC used exclusively for this purpose**. Do not deploy it alongside production systems or store any sensitive data on the instance. All emulated services are fake and non-exploitable by design, but treat the instance as untrusted.

---

*Built as a personal cybersecurity lab project to practice cloud security architecture, threat detection, and AWS infrastructure.*
