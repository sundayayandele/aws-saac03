# AWS SCS-C03 Study Guide: Encryption & Key Management

---

## 1. KMS (AWS Key Management Service)

### Overview
AWS KMS is a managed service for creating and controlling cryptographic keys used to protect your data. It integrates with most AWS services and provides a centralized key management solution.

---

### Symmetric vs Asymmetric Keys

| Feature | Symmetric Keys | Asymmetric Keys |
|---|---|---|
| **Algorithm** | AES-256-GCM | RSA, ECC (ECDSA, ECDH) |
| **Key material** | Single secret key | Public/Private key pair |
| **Operations** | Encrypt & Decrypt (same key) | Encrypt with public, Decrypt with private (RSA) OR Sign with private, Verify with public |
| **Usage in AWS** | Default for most AWS service integrations (S3, EBS, RDS) | Cross-account/cross-region signing, code signing, external clients |
| **API call** | `Encrypt`, `Decrypt`, `GenerateDataKey` | `Encrypt` (public key), `Decrypt` (private key), `Sign`, `Verify` |
| **Performance** | Faster | Slower |
| **Key never leaves KMS?** | ✅ Yes | ⚠️ Public key can be downloaded; private key never leaves |

> **Exam Tip:** KMS symmetric keys **never leave KMS unencrypted**. AWS services use **envelope encryption** — KMS generates a Data Encryption Key (DEK), encrypts your data with the DEK, then KMS encrypts the DEK itself.

---

### Key Types in KMS

| Type | Who Creates It | Who Controls It | Rotation |
|---|---|---|---|
| **AWS Managed Key** | AWS | AWS | Automatic (every 3 years) |
| **Customer Managed Key (CMK)** | Customer | Customer | Optional (annual, manual, or on-demand) |
| **AWS Owned Key** | AWS | AWS (shared across accounts) | AWS managed |

---

### Automatic Key Rotation

- Applies only to **symmetric Customer Managed Keys (CMKs)**
- When enabled: KMS rotates key material **every year** (365 days)
- **Old key material is retained** — data encrypted with old versions can still be decrypted
- The **Key ID and ARN do NOT change** after rotation
- **Not supported for:** Asymmetric keys, HMAC keys, keys with imported material, AWS managed keys (they rotate on their own schedule), Custom Key Store (CloudHSM) keys
- **On-demand rotation** is available (independent of scheduled rotation)

> **Exam Tip:** Rotation changes the *backing key material* only — the KMS Key ID/ARN stays the same, so no application changes are needed.

---

## 2. Encryption at Rest

### S3 Encryption Options

| Method | Full Name | Key Management | Who Controls the Key |
|---|---|---|---|
| **SSE-S3** | Server-Side Encryption with S3 Managed Keys | S3 manages keys automatically | AWS |
| **SSE-KMS** | Server-Side Encryption with KMS Keys | KMS (AWS or CMK) | Customer (via KMS) |
| **DSSE-KMS** | Dual-Layer SSE with KMS Keys | Two independent KMS encryption layers | Customer |
| **SSE-C** | Server-Side Encryption with Customer-Provided Keys | Customer provides key per request | Customer (not stored by AWS) |
| **CSE** | Client-Side Encryption | Encrypted before upload | Customer |

#### Key Differences for the Exam
- **SSE-S3**: Simplest, no cost, least control. Uses AES-256.
- **SSE-KMS**: Audit trail via CloudTrail, fine-grained access with key policies. Extra KMS API call costs apply.
- **DSSE-KMS**: Strongest S3 encryption — two independent encryption layers. Required for some compliance workloads (e.g., CNSSI 1253).
- **SSE-C**: AWS encrypts/decrypts but you supply the key — key is NOT stored by AWS.

#### Enforcing Encryption on S3
- Use **S3 Bucket Policy** to deny `s3:PutObject` unless `x-amz-server-side-encryption` header is present.
- Use **S3 Default Encryption** to automatically apply SSE-S3 or SSE-KMS to new objects.

---

### EBS Encryption

- Uses **AES-256** with AWS KMS
- Encrypts: data at rest, data moving between instance and volume, snapshots, and volumes created from those snapshots
- Enable at the **account level** (encrypt by default) or per-volume
- **Snapshots of encrypted volumes are encrypted**; snapshots of unencrypted volumes are unencrypted
- To encrypt an existing unencrypted volume: **snapshot → copy snapshot (enable encryption) → create volume from encrypted snapshot**

---

### RDS Encryption

- Enabled at **DB instance creation** (cannot be added later)
- Uses KMS CMK; encrypts storage, automated backups, read replicas, and snapshots
- To encrypt an unencrypted DB: **snapshot → copy with encryption enabled → restore**
- **Read replicas** of encrypted primary DBs are automatically encrypted with the same key
- Supports **SSL/TLS** for in-transit encryption (separate from at-rest)

---

### DynamoDB Encryption

- **Encryption at rest enabled by default** for all tables (since 2018)
- Three key options:
  - **DynamoDB owned key** (DEFAULT): Free, AWS managed, no CloudTrail visibility
  - **AWS managed key** (`aws/dynamodb`): Free, KMS-visible in CloudTrail
  - **Customer Managed Key (CMK)**: Full control, audit trail, additional KMS costs
- Encrypts: primary data, secondary indexes, global tables, streams, backups

---

## 3. Encryption in Transit

### TLS Everywhere
- Use **TLS 1.2 minimum** (TLS 1.3 preferred) for all data in transit
- AWS services expose HTTPS endpoints by default
- Common enforcement points:
  - **S3**: Bucket policy denying non-HTTPS (`aws:SecureTransport: false`)
  - **RDS/Aurora**: Require SSL via parameter group (`rds.force_ssl = 1`)
  - **API Gateway**: HTTPS only by default
  - **ALB/NLB**: HTTPS listeners with security policies

### AWS Certificate Manager (ACM)

| Feature | Detail |
|---|---|
| **Free public certs** | ✅ No charge for SSL/TLS certs used with AWS services |
| **Auto-renewal** | ✅ ACM auto-renews before expiry |
| **Supported services** | ALB, NLB, CloudFront, API Gateway, Elastic Beanstalk |
| **Private CA** | ACM Private CA (paid) for internal/private certificates |
| **Cannot export** | Public certs from ACM **cannot be exported** (private key stays in ACM) |
| **DNS / Email validation** | Two methods to prove domain ownership |

> **Exam Tip:** ACM certificates are **regional**. CloudFront requires certs in **us-east-1 (N. Virginia)**.

---

## 4. CloudHSM

### What Is It?
AWS CloudHSM provides **dedicated, single-tenant Hardware Security Modules (HSMs)** in the AWS cloud. Unlike KMS (shared infrastructure), CloudHSM gives you exclusive control over a physical HSM device.

### KMS vs CloudHSM Comparison

| Feature | AWS KMS | CloudHSM |
|---|---|---|
| **Hardware** | Shared (multi-tenant) | Dedicated (single-tenant) |
| **Management** | AWS manages | Customer manages (AWS provides hardware) |
| **Key storage** | KMS-managed | Customer-controlled HSM |
| **FIPS 140-2 Level** | Level 2 | **Level 3** |
| **Integration** | Native AWS service integration | Manual (PKCS#11, JCE, CNG) |
| **Cost** | Per API call | Hourly per HSM (~$1.45/hr) |
| **High Availability** | Built-in | Must create HSM cluster (2+ HSMs) |
| **Use case** | General encryption, most AWS workloads | Compliance-heavy (FIPS L3), custom crypto, SSL offload |

### Key CloudHSM Use Cases
1. **FIPS 140-2 Level 3 compliance** — required by some government/financial regulations
2. **Custom key management** — you own and manage the keys; AWS has NO access
3. **Oracle TDE (Transparent Data Encryption)** — Oracle DB encryption with your HSM
4. **SSL/TLS offloading** on web servers
5. **Digital signing** with full key custody

> **Exam Tip:** If the question mentions **"dedicated HSM"**, **"FIPS 140-2 Level 3"**, **"customer-managed keys with no AWS access"**, or **"compliance requiring exclusive key control"** → the answer is **CloudHSM**.

---

## Quick Reference: Decision Tree

```
Need to encrypt data in AWS?
│
├─ At rest in S3?
│   ├─ No key management needed → SSE-S3
│   ├─ Audit trail + access control → SSE-KMS
│   └─ Max security / dual layer → DSSE-KMS
│
├─ At rest in EBS/RDS/DynamoDB?
│   └─ Enable KMS encryption (CMK for audit + control)
│
├─ In transit?
│   ├─ Public-facing HTTPS → ACM (free certs)
│   └─ Enforce TLS → Bucket policies / security groups / parameter groups
│
└─ Key management?
    ├─ Standard workloads → AWS KMS (CMK)
    ├─ Symmetric, fast, AWS-integrated → Symmetric CMK
    ├─ Cross-region signing / external clients → Asymmetric CMK
    └─ FIPS L3 / dedicated hardware / compliance → CloudHSM
```

---

## Exam-Critical Facts Summary

| # | Fact |
|---|---|
| 1 | KMS symmetric keys **never leave KMS** — envelope encryption is used |
| 2 | Key rotation changes **key material only** — Key ID/ARN stays the same |
| 3 | Asymmetric keys in KMS: public key can be downloaded; private key stays in KMS |
| 4 | SSE-C: AWS encrypts, but does NOT store your key |
| 5 | DSSE-KMS = two independent encryption layers (strongest S3 option) |
| 6 | RDS/EBS encryption must be enabled at creation; cannot be added to existing resources |
| 7 | ACM certs are **free** for use with AWS services, but **cannot be exported** |
| 8 | CloudFront ACM certs must be in **us-east-1** |
| 9 | CloudHSM = FIPS 140-2 Level 3; KMS = Level 2 |
| 10 | CloudHSM: AWS has **zero access** to your keys |
| 11 | DynamoDB encrypts at rest **by default** |
| 12 | S3 bucket policy: deny `aws:SecureTransport: false` to enforce HTTPS |
