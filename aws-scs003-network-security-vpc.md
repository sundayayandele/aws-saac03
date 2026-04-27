# AWS SCS-C03 Study Guide: Network Security – VPC

---

## 1. Security Groups vs Network ACLs

### Core Distinction

| Feature | Security Groups (SG) | Network ACLs (NACL) |
|---|---|---|
| **State** | **Stateful** | **Stateless** |
| **Applies to** | EC2 instances / ENIs | Subnet level |
| **Rule types** | Allow only | Allow AND Deny |
| **Inbound/Outbound** | Tracked — return traffic auto-allowed | Must explicitly allow both directions |
| **Rule evaluation** | All rules evaluated together | Rules evaluated in **number order** (lowest first) |
| **Default (new VPC)** | No inbound, all outbound allowed | All inbound/outbound ALLOWED |
| **Rule count** | Up to 60 inbound + 60 outbound per SG | Up to 20 rules (request increase for more) |
| **Association** | One instance can have multiple SGs | One NACL per subnet; one subnet per NACL |

---

### Stateful vs Stateless Explained

**Stateful (Security Groups):**
- If you allow inbound traffic on port 443, the **response is automatically allowed** out — no outbound rule needed for the reply.
- Tracks connection state.

**Stateless (Network ACLs):**
- Each packet is evaluated independently in both directions.
- Must add rules for **both** the request (inbound) AND the response (outbound ephemeral ports: 1024–65535).
- Forgetting the ephemeral port outbound rule is a classic exam trap.

---

### NACL Rule Evaluation

Rules are processed in **ascending numerical order**. First matching rule wins — processing stops.

```
Rule #  | Type       | Protocol | Port  | Source       | Action
--------|------------|----------|-------|--------------|--------
100     | HTTP       | TCP      | 80    | 0.0.0.0/0    | ALLOW
200     | HTTPS      | TCP      | 443   | 0.0.0.0/0    | ALLOW
300     | Custom TCP | TCP      | 22    | 10.0.0.0/8   | ALLOW
*       | All Traffic| All      | All   | 0.0.0.0/0    | DENY   ← implicit deny
```

> **Exam Tip:** To **block a specific IP**, add a DENY rule with a **lower number** than the ALLOW rules. Security Groups cannot do this — use NACLs for explicit deny.

---

### When to Use Each

| Scenario | Use |
|---|---|
| Allow traffic to specific ports on an EC2 instance | Security Group |
| Block a known malicious IP range | Network ACL |
| Allow traffic between instances in the same SG | Security Group (self-referencing) |
| Subnet-level firewall controlling all subnet traffic | Network ACL |
| Fine-grained per-instance control | Security Group |

---

## 2. VPC Endpoints

VPC Endpoints allow AWS service traffic to **stay within the AWS network** — no internet gateway, NAT device, VPN, or AWS Direct Connect needed. Eliminates exposure to the public internet.

---

### Gateway Endpoints

| Property | Detail |
|---|---|
| **Services** | **S3** and **DynamoDB** only |
| **Mechanism** | Route table entry — traffic routed through AWS backbone |
| **Cost** | **Free** |
| **Scope** | Region-level (not AZ-specific) |
| **Security** | Endpoint policies control what resources can be accessed |
| **DNS** | No DNS change required |

**Use case:** EC2 in private subnet needs to access S3 without NAT Gateway.

---

### Interface Endpoints (AWS PrivateLink)

| Property | Detail |
|---|---|
| **Services** | All other AWS services (EC2 API, KMS, SSM, SNS, SQS, etc.) |
| **Mechanism** | Elastic Network Interface (ENI) with private IP in your subnet |
| **Cost** | Hourly + data processing charges |
| **Scope** | AZ-specific (deploy in each AZ for HA) |
| **Security** | Security Groups + Endpoint policies |
| **DNS** | Private DNS enabled by default (service DNS resolves to private IP) |

---

### Gateway vs Interface Comparison

| | Gateway Endpoint | Interface Endpoint |
|---|---|---|
| **Services** | S3, DynamoDB | Everything else |
| **Implementation** | Route table | ENI (private IP) |
| **Cost** | Free | Paid |
| **High Availability** | Automatic | Manual (per AZ) |
| **On-premises access** | ❌ No | ✅ Yes (via VPN/DX) |
| **Cross-region** | ❌ No | ❌ No |

> **Exam Tip:** If the scenario involves on-premises servers accessing an AWS service privately (via Direct Connect or VPN), it must be an **Interface Endpoint** — Gateway Endpoints are not accessible from outside the VPC.

---

### Endpoint Policies

- JSON-based resource policies attached to the endpoint
- Can restrict **which principals** can access **which resources** through the endpoint
- Example: Restrict S3 Gateway Endpoint to only allow access to a specific bucket

```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-secure-bucket/*"
}
```

---

## 3. AWS Network Firewall & WAF

### AWS Network Firewall

A managed, stateful network firewall for **VPC-level** protection (Layer 3/4 + Layer 7 for some protocols).

| Feature | Detail |
|---|---|
| **Placement** | Deployed into a dedicated subnet in each AZ |
| **Inspection** | Stateful + stateless packet inspection |
| **Rules** | IP/port filtering, domain name filtering, Suricata IDS/IPS rules |
| **Integration** | Works with AWS Firewall Manager for multi-account |
| **Use case** | Filter outbound traffic, block C2 callback domains, IDS/IPS |
| **Cost** | Hourly per endpoint + data processing |

**Key capabilities:**
- **Stateless rules** — fast, packet-by-packet matching
- **Stateful rules** — track connection state, protocol detection
- **Domain list rules** — allow/deny traffic to specific FQDNs (e.g., block `*.badactor.com`)
- **Suricata-compatible IPS** — custom intrusion prevention rules

---

### AWS WAF (Web Application Firewall)

Layer 7 (HTTP/HTTPS) protection for web applications. Operates at the **application layer**.

| Feature | Detail |
|---|---|
| **Applies to** | CloudFront, ALB, API Gateway, AppSync, Cognito |
| **Rules** | IP sets, geo-match, rate limiting, SQL injection, XSS, custom regex |
| **Managed Rules** | Pre-built rule groups from AWS and Marketplace (e.g., OWASP Top 10) |
| **Cost** | Per WebACL + per rule + per request |
| **Logging** | Sends logs to S3, CloudWatch, Kinesis Firehose |

**Common WAF Rules:**
- AWS Managed Rules — AWSManagedRulesCommonRuleSet (OWASP)
- Rate-based rules — block IPs exceeding X requests/5 min
- Geographic match — block traffic from specific countries
- SQL injection / XSS protection

---

### Network Firewall vs WAF

| | AWS Network Firewall | AWS WAF |
|---|---|---|
| **OSI Layer** | Layer 3/4 (+ L7 for domains) | Layer 7 (HTTP/HTTPS) |
| **Placement** | Inside VPC subnet | CloudFront / ALB / API GW edge |
| **Traffic** | Any TCP/UDP | HTTP/HTTPS only |
| **DDoS** | Basic filtering | Rate-based rules (partial) |
| **Use case** | East-west + north-south VPC traffic | Web app OWASP protection |

> **Exam Tip:** WAF protects **web applications** (SQL injection, XSS, bot control). Network Firewall protects **network traffic** at the VPC level (IDS/IPS, domain filtering, any protocol).

---

## 4. AWS Shield

### Shield Standard (Free, Automatic)

- **Automatically enabled** for all AWS accounts at no cost
- Protects against **common Layer 3/4 DDoS attacks**: SYN floods, UDP reflection, DNS amplification
- Integrated with CloudFront and Route 53
- No configuration required

---

### Shield Advanced (Paid)

| Feature | Detail |
|---|---|
| **Cost** | $3,000/month per organization + data transfer fees |
| **Protected resources** | EC2 EIPs, ELB, CloudFront, Global Accelerator, Route 53 |
| **DDoS Response Team (DRT)** | 24/7 access to AWS DDoS experts |
| **Cost protection** | AWS credits for scaling costs during attacks |
| **Advanced detection** | Near real-time attack visibility, event summaries |
| **WAF integration** | Automatic WAF rule creation during attacks |
| **Health-based detection** | Uses Route 53 health checks to improve detection |

---

### Shield Standard vs Advanced

| | Shield Standard | Shield Advanced |
|---|---|---|
| **Cost** | Free | $3,000/month |
| **L3/L4 protection** | ✅ Basic | ✅ Enhanced |
| **L7 protection** | ❌ No | ✅ With WAF |
| **DRT access** | ❌ No | ✅ 24/7 |
| **Cost protection** | ❌ No | ✅ Yes |
| **Real-time metrics** | ❌ No | ✅ Yes |
| **Automatic response** | ❌ No | ✅ Yes |

> **Exam Tip:** If scenario mentions "DDoS protection", "DRT team", "cost protection from DDoS scaling", or "advanced attack visibility" → **Shield Advanced**. Standard covers only basic volumetric attacks automatically.

---

## 5. Defense-in-Depth Architecture

```
Internet
    │
    ▼
[Route 53 + Shield Standard (free, auto)]
    │
    ▼
[CloudFront + WAF WebACL + Shield Advanced]
    │  ← Layer 7: SQL injection, XSS, geo-block, rate limit
    ▼
[ALB + Security Group (allow 443 from CloudFront only)]
    │
    ▼
[VPC — AWS Network Firewall subnet]
    │  ← Layer 3/4/7: IDS/IPS, domain filtering
    ▼
[Private subnet — EC2 with Security Group]
    │
    ▼
[S3 via Gateway Endpoint / KMS via Interface Endpoint]
    (traffic never leaves AWS backbone)
```

---

## Quick Reference: Decision Tree

```
Network security question?
│
├─ Need to BLOCK a specific IP or IP range?
│   └─ Network ACL (DENY rule, lower number than ALLOW)
│
├─ Per-instance port filtering?
│   └─ Security Group
│
├─ Keep traffic off the public internet to S3 or DynamoDB?
│   └─ Gateway Endpoint (free)
│
├─ Keep traffic off the internet to other AWS services?
│   └─ Interface Endpoint (PrivateLink)
│
├─ On-premises → AWS service privately?
│   └─ Interface Endpoint (via Direct Connect / VPN)
│
├─ Protect web app from OWASP / SQL injection / XSS?
│   └─ WAF (attach to CloudFront / ALB / API GW)
│
├─ VPC-level IDS/IPS or block outbound domains?
│   └─ AWS Network Firewall
│
└─ DDoS protection?
    ├─ Basic (automatic, free) ─────── Shield Standard
    └─ Advanced + DRT + cost cover ─── Shield Advanced
```

---

## Exam-Critical Facts Summary

| # | Fact |
|---|---|
| 1 | Security Groups are **stateful** — return traffic automatically allowed |
| 2 | NACLs are **stateless** — must allow both inbound AND outbound (incl. ephemeral ports 1024–65535) |
| 3 | NACL rules evaluated in **ascending number order** — first match wins |
| 4 | To **block an IP**, use NACL (SGs cannot deny, only allow) |
| 5 | Gateway Endpoints: **S3 and DynamoDB only**, free, route-table based |
| 6 | Interface Endpoints: all other AWS services, paid, ENI-based |
| 7 | Interface Endpoints support **on-premises access** via Direct Connect/VPN; Gateway do not |
| 8 | WAF operates at **Layer 7** (HTTP/HTTPS); attach to CloudFront, ALB, API GW |
| 9 | AWS Network Firewall operates at **Layer 3/4** inside VPC; supports Suricata IPS rules |
| 10 | Shield Standard is **free and automatic** for all accounts |
| 11 | Shield Advanced costs **$3,000/month** and includes DRT access + cost protection |
| 12 | Shield Advanced + WAF = protection against both L3/4 and L7 DDoS |
