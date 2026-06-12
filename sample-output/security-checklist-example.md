# AWS Data Protection & Encryption Security Policy Checklist
### For: Small Startup Web Application (EC2 + S3 + RDS)

---

## 🔴 IMMEDIATE ACTIONS
*Quick wins implementable within days*

---

### IAM Identity & Access Management

- [ ] **Audit root account usage** — Verify the AWS root account has no active access keys (`IAM Console → Security Credentials`). **PASS:** Zero root access keys exist. **FAIL:** Any root access keys are present → delete immediately.

- [ ] **Enable MFA on root account** — Attach a hardware MFA or virtual MFA device to the root account. **PASS:** MFA status shows "Enabled" in IAM Security Credentials. **FAIL:** MFA status shows "Not enabled."

- [ ] **Enable MFA for all IAM console users** — Run `aws iam generate-credential-report` and verify `mfa_active = true` for every human user. **PASS:** All users show `true`. **FAIL:** Any user shows `false` → enforce MFA immediately or disable the account.

- [ ] **Delete or deactivate unused IAM users** — Review the credential report for `password_last_used` and `access_key_last_used` fields. **PASS:** No accounts unused for 90+ days remain active. **FAIL:** Stale accounts exist → disable or delete.

- [ ] **Remove inline policies from IAM users** — Verify no IAM users have inline policies attached directly. **PASS:** All permissions are managed via groups or roles. **FAIL:** Inline policies exist → migrate to managed policies.

---

### EC2 Instance Role (Least Privilege for S3 Access)

- [ ] **Create a dedicated EC2 IAM Role** — Create an IAM role with the trust policy below and attach it to the EC2 instance via Instance Profile. **PASS:** EC2 instance shows an attached IAM role in the console. **FAIL:** No role attached, or credentials are hardcoded in application code.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-SPECIFIC-BUCKET-NAME",
        "arn:aws:s3:::YOUR-SPECIFIC-BUCKET-NAME/*"
      ]
    }
  ]
}
```

> ⚠️ **Critical:** Replace `YOUR-SPECIFIC-BUCKET-NAME` with your actual bucket name. The first ARN covers `ListBucket`; the second covers object-level operations.

- [ ] **Verify EC2 role has NO wildcard S3 permissions** — Check the attached policy contains no `"Resource": "*"` for S3 actions. **PASS:** Resource is scoped to the specific bucket ARN only. **FAIL:** Wildcard resource exists → remediate immediately.

- [ ] **Confirm no hardcoded AWS credentials in application code** — Search the codebase and environment files: `grep -r "AKIA" .` (access key prefix) and `grep -r "aws_secret" .`. **PASS:** Zero matches found. **FAIL:** Credentials found → rotate keys immediately, use IAM role instead.

---

### S3 Bucket Security

- [ ] **Block all public access on the S3 bucket** — Navigate to `S3 → Bucket → Permissions → Block Public Access` and enable all four settings. **PASS:** All four toggles show "On." **FAIL:** Any toggle is "Off" → enable unless explicitly required with documented justification.

- [ ] **Enable S3 default encryption** — Set bucket default encryption to `SSE-S3` (AES-256) or `SSE-KMS`. **PASS:** Encryption shows "Enabled" with chosen algorithm. **FAIL:** Encryption shows "Disabled."

```bash
# CLI command to verify
aws s3api get-bucket-encryption --bucket YOUR-BUCKET-NAME
```

- [ ] **Remove any bucket policies granting public read/write** — Run `aws s3api get-bucket-policy --bucket YOUR-BUCKET-NAME` and verify no `"Principal": "*"` statements exist without explicit `Condition` restrictions. **PASS:** No open principal statements. **FAIL:** Public principal found → remove immediately.

---

### RDS Database Credentials

- [ ] **Store database credentials in AWS Secrets Manager** — Create a secret in Secrets Manager for RDS credentials and update the Node.js application to retrieve credentials via the SDK at runtime. **PASS:** Application retrieves credentials via `secretsmanager:GetSecretValue` API call. **FAIL:** Credentials exist in `.env` files, config files, or source code.

```javascript
// Node.js example — retrieve secret at runtime
const { SecretsManagerClient, GetSecretValueCommand } = require("@aws-sdk/client-secrets-manager");

const client = new SecretsManagerClient({ region: "us-east-1" });
const response = await client.send(
  new GetSecretValueCommand({ SecretId: "prod/myapp/rds-credentials" })
);
const { username, password } = JSON.parse(response.SecretString);
```

- [ ] **Grant EC2 role permission to read only the specific secret** — Add the following policy to the EC2 IAM role. **PASS:** Role can retrieve the specific secret and nothing else. **FAIL:** Role has `secretsmanager:*` or access to all secrets.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:REGION:ACCOUNT-ID:secret:prod/myapp/rds-credentials-*"
    }
  ]
}
```

---

## 🟡 SHORT-TERM ACTIONS
*Implementable within 1–4 weeks*

---

### IAM Policy Hardening

- [ ] **Implement IAM Permission Boundaries for developer roles** — Create a permission boundary policy that caps maximum permissions for developer IAM roles, preventing privilege escalation. **PASS:** All developer roles have a `PermissionsBoundary` ARN attached. **FAIL:** No boundaries set.

- [ ] **Create IAM groups aligned to job functions** — Define groups such as `Developers`, `ReadOnlyAuditors`, and `DBAdmins` with appropriate managed policies. Assign users to groups rather than attaching policies directly. **PASS:** All users belong to at least one group; no direct user-level policy attachments. **FAIL:** Users have individually attached policies.

- [ ] **Enforce IAM password policy** — Configure the account password policy with the following minimum settings via `IAM → Account Settings`:
  - Minimum length: **14 characters**
  - Require uppercase, lowercase, numbers, and symbols
  - Password expiration: **90 days**
  - Prevent password reuse: **last 12 passwords**

  **PASS:** All criteria met. **FAIL:** Any criterion not configured.

- [ ] **Run AWS IAM Access Analyzer** — Enable IAM Access Analyzer in each region and review all findings for external access to S3, IAM roles, and KMS keys. **PASS:** Zero unresolved "External Access" findings. **FAIL:** Open findings exist → archive with justification or remediate.

```bash
aws accessanalyzer create-analyzer \
  --analyzer-name "startup-analyzer" \
  --type ACCOUNT
```

- [ ] **Apply an explicit deny policy for dangerous IAM actions** — Attach a Service Control Policy (SCP) or IAM boundary that denies the following high-risk actions for non-admin roles:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDangerousActions",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:AttachUserPolicy",
        "iam:CreateAccessKey",
        "iam:DeleteBucketPolicy",
        "s3:DeleteBucket",
        "rds:DeleteDBInstance"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### RDS Security Hardening

- [ ] **Enable RDS encryption at rest** — Verify the RDS instance has encryption enabled (must be set at creation time). **PASS:** `StorageEncrypted: true` in `aws rds describe-db-instances`. **FAIL:** `false` → snapshot the database, restore to a new encrypted instance, and decommission the unencrypted one.

- [ ] **Enable RDS encryption in transit** — Enforce SSL/TLS connections by setting the `rds.force_ssl` parameter to `1` in the RDS Parameter Group. **PASS:** Parameter shows `1` and application connects with SSL. **FAIL:** Parameter is `0` or not set.

- [ ] **Place RDS in a private subnet** — Verify the RDS instance is in a private subnet with no route to an Internet Gateway. **PASS:** `PubliclyAccessible: false` in RDS settings and subnet has no IGW route. **FAIL:** RDS is publicly accessible → modify instance to disable public accessibility.

- [ ] **Restrict RDS Security Group to EC2 only** — Configure the RDS Security Group inbound rule to allow traffic only from the EC2 instance's Security Group ID on port 5432 (PostgreSQL) or 3306 (MySQL). **PASS:** Inbound rule source is the EC2 Security Group ID. **FAIL:** Source is `0.0.0.0/0` or a broad CIDR range.

- [ ] **Enable automated RDS backups** — Set backup retention period to a minimum of **7 days**. **PASS:** `BackupRetentionPeriod >= 7`. **FAIL:** Value is `0` (disabled) or less than 7.

---

### S3 Advanced Security

- [ ] **Enable S3 Versioning** — Enable versioning on the bucket to protect against accidental deletion and ransomware. **PASS:** Versioning status shows "Enabled." **FAIL:** "Suspended" or "Disabled."

- [ ] **Configure S3 Object Lock or MFA Delete** — For sensitive documents, enable MFA Delete or Object Lock in COMPLIANCE mode to prevent deletion without MFA. **PASS:** Feature is enabled and tested. **FAIL:** Neither protection is in place.

- [ ] **Enable S3 server access logging** — Configure access logging to a separate logging bucket to capture all requests. **PASS:** Logging shows a target bucket and prefix. **FAIL:** Logging is disabled.

- [ ] **Create a restrictive S3 bucket policy enforcing HTTPS only** — Add the following bucket policy to deny all non-HTTPS requests:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonHTTPS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

---

### Encryption Key Management

- [ ] **Create a Customer Managed KMS Key (CMK) for sensitive data** — Create a KMS key for encrypting S3 objects and RDS storage. **PASS:** CMK exists with a defined key policy. **FAIL:** Using only AWS-managed keys with no custom key policy control.

- [ ] **Define a restrictive KMS key policy** — Ensure the KMS key policy grants `kms:Decrypt` and `kms:GenerateDataKey` only to the EC2 role and RDS service, not to all IAM principals:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2RoleAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-ID:role/EC2-App-Role"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowKeyAdministration",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-ID:role/SecurityAdminRole"
      },
      "Action": [
        "kms:Create*",
        "kms:Describe*",
        "kms:Enable*",
        "kms:List*",
        "kms:Put*",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 🟢 LONG-TERM STRATEGIC ACTIONS
*1–3 months*

---

### Monitoring & Threat Detection

- [ ] **Enable AWS CloudTrail in all regions** — Create a multi-region trail logging all management events to an S3 bucket with log file validation enabled. **PASS:** Trail shows "Logging: Yes," multi-region enabled, and log file validation enabled. **FAIL:** Trail is single-region, disabled, or lacks validation.

- [ ] **Enable Amazon GuardDuty** — Activate GuardDuty in all active regions to detect threats such as credential compromise, unusual API calls, and S3 data exfiltration. **PASS:** GuardDuty status shows "Enabled" with no unreviewed High/Critical findings. **FAIL:** GuardDuty is disabled.

- [ ] **Enable AWS Security Hub** — Activate Security Hub with the `AWS Foundational Security Best Practices` standard enabled. **PASS:** Security Hub score above 80% for your account. **FAIL:** Disabled or score below 60%.

- [ ] **Create CloudWatch Alarms for critical security events** — Configure alarms for the following CloudTrail metric filters:
  - Root account login
  - IAM policy changes
  - S3 bucket policy changes
  - Failed console login attempts (>5 in 5 minutes)
  - RDS security group changes

  **PASS:** All five alarms exist and route to an SNS topic with verified email subscription. **FAIL:** Any alarm is missing.

- [ ] **Implement VPC Flow Logs** — Enable VPC Flow Logs for the VPC hosting EC2 and RDS, publishing to CloudWatch Logs. **PASS:** Flow logs show "Active" status. **FAIL:** Flow logs are disabled.

---

### Network Security

- [ ] **Implement AWS WAF on the application endpoint** — Attach AWS WAF to the Application Load Balancer (or CloudFront) with the `AWS Managed Rules Common Rule Set` enabled. **PASS:** WAF is associated and logging is enabled. **FAIL:** No WAF protection on public-facing endpoints.

- [ ] **Restrict EC2 Security Group inbound rules** — Verify the EC2 Security Group allows inbound traffic only on ports 80/443 from `0.0.0.0/0` and port 22 (SSH) only from specific team IP addresses or a VPN CIDR. **PASS:** No unrestricted SSH (port 22 from `0.0.0.0/0`). **FAIL:** SSH open to the world → restrict immediately.

- [ ] **Use AWS Systems Manager Session Manager instead of SSH** — Disable SSH access entirely and use SSM Session Manager for EC2 access. **PASS:** Security Group has no inbound port 22 rule; SSM agent is running on EC2. **FAIL:** SSH is the primary access method.

---

### Secrets & Credential Lifecycle

- [ ] **Enable automatic rotation for Secrets Manager secrets** — Configure automatic rotation for the RDS credentials secret with a rotation interval of 30 days using the AWS-provided Lambda rotation function. **PASS:** Secret shows "Rotation: Enabled, every 30 days." **FAIL:** Rotation is disabled.

- [ ] **Audit and rotate all existing IAM access keys** — Identify all access keys older than 90 days via the credential report and rotate them. **PASS:** No active access keys older than 90 days. **FAIL:** Stale keys exist.

- [ ] **Implement AWS Organizations with SCPs** — If the startup grows to multiple AWS accounts, implement AWS Organizations with Service Control Policies to enforce guardrails at the organization level (e.g., deny disabling CloudTrail, deny leaving the organization). **PASS:** SCPs are attached to all non-management OUs. **FAIL:** Single account with no organizational guardrails.

---

### Data Classification & Lifecycle

- [ ] **Enable Amazon Macie on the S3 bucket** — Activate Macie to automatically discover and classify sensitive data (PII, credentials) stored in S3. **PASS:** Macie is enabled and has completed at least one discovery job on the target bucket. **FAIL:** Macie is disabled.

- [ ] **Define and implement S3 lifecycle policies** — Create lifecycle rules to transition objects to `S3-IA` after 30 days and `Glacier` after 90 days for cost and data management. **PASS:** Lifecycle rules are configured and verified. **FAIL:** No lifecycle rules exist.

---

## 📋 COMPLIANCE AND AUDIT CHECKS
*Ongoing verification against industry standards*

---

### CIS AWS Foundations Benchmark v1.5

| Check | CIS Control | Verification Method |
|---|---|---|
| Root account MFA enabled | 1.5 | IAM Console → Security Credentials |
| No root account access keys | 1.4 | Credential report: `root_account_access_key_active = false` |
| Password policy meets requirements | 1.8–1.11 | `aws iam get-account-password-policy` |
| CloudTrail enabled in all regions | 3.1 | `aws cloudtrail describe-trails` |
| S3 bucket logging enabled | 3.6 | `aws s3api get-bucket-logging` |
| VPC flow logging enabled | 3.9 | `aws ec2 describe-flow-logs` |
| No security groups allow 0.0.0.0/0 on port 22 | 5.2 | `aws ec2 describe-security-groups` |

### SOC 2 Type II Relevant Controls

- [ ] **Verify access reviews are conducted quarterly** — Document that IAM user access, role permissions, and S3 bucket policies are reviewed every 90 days. **PASS:** Review records exist with dates and reviewer names. **FAIL:** No documented review process.

- [ ] **Verify encryption is applied to all data at rest** — Confirm S3 default encryption and RDS `StorageEncrypted: true`. **PASS:** Both services show encryption enabled. **FAIL:** Either service lacks encryption.

- [ ] **Verify encryption in transit for all services** — Confirm HTTPS-only S3 bucket policy, RDS SSL enforcement, and ALB HTTPS listener. **PASS:** All three enforced. **FAIL:** Any plaintext path exists.

### GDPR / Data Privacy (if handling EU user data)

- [ ] **Verify S3 bucket region is compliant with data residency requirements** — Confirm the bucket is in an approved AWS region. **PASS:** Region matches data residency policy. **FAIL:** Data stored in non-compliant region.

- [ ] **Verify Macie PII findings are reviewed and remediated** — Review Macie findings monthly. **PASS:** All High severity findings resolved within 30 days. **FAIL:** Unresolved findings older than 30 days.

---

## 📁 POLICY DOCUMENTATION REQUIREMENTS
*Records and evidence to maintain*

---

### Required Documents

| Document | Description | Review Frequency |
|---|---|---|
| **IAM Policy Register** | Inventory of all IAM roles, policies, and their business justification | Quarterly |
| **Data Classification Policy** | Defines sensitivity levels for S3 objects and RDS data | Annually |
| **Encryption Standards Document** | Documents encryption algorithms, key rotation schedules, and CMK ownership | Annually |
| **Access Review Records** | Signed-off quarterly reviews of all IAM users, roles, and S3 bucket policies | Quarterly |
| **Incident Response Runbook** | Step-by-step procedures for credential compromise, S3 data exposure, and RDS breach | Annually + after incidents |
| **Change Management Log** | Record of all IAM policy changes with approver, date, and justification | Ongoing (CloudTrail serves as technical log) |
| **Secrets Rotation Log** | Evidence that Secrets Manager rotation is functioning (rotation dates, success/failure) | Monthly |
| **Security Training Records** | Evidence that all team members with AWS Console access completed security awareness training | Annually |

---

### Evidence Collection for Audits

- [ ] **Export and archive monthly IAM credential reports** — `aws iam generate-credential-report && aws iam get-credential-report` → store output in a secure, versioned location.

- [ ] **Retain CloudTrail logs for minimum 1 year** — Configure S3 lifecycle policy on the CloudTrail bucket to retain logs for 365 days minimum, with Glacier archival after 90 days.

- [ ] **Document all exceptions to this policy** — Maintain a risk register for any checklist items that cannot be implemented, including business justification, risk owner, and compensating controls.

- [ ] **Capture Security Hub compliance score monthly** — Screenshot or export the Security Hub findings summary monthly and store in the compliance evidence folder.

---

## ⚠️ COMMON MISCONFIGURATIONS TO AVOID

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TOP MISCONFIGURATIONS                            │
├─────────────────────────────────────────────────────────────────────┤
│ 1. Using "Resource": "*" in EC2 role S3 policies                   │
│    → Always scope to specific bucket ARNs                           │
│                                                                     │
│ 2. Storing AWS credentials in EC2 user data or .env files          │
│    → Always use IAM Instance Profiles                               │
│                                                                     │
│ 3. RDS with PubliclyAccessible: true                               │
│    → Always place RDS in private subnets                            │
│                                                                     │
│ 4. S3 bucket with ACLs enabled (legacy)                            │
│    → Disable ACLs, use bucket policies exclusively                  │
│                                                                     │
│ 5. Sharing IAM users between team members                          │
│    → One IAM user (or SSO identity) per person, always             │
│                                                                     │
│ 6. No MFA on accounts with console access                          │
│    → Enforce MFA via IAM condition key                              │
│                                                                     │
│ 7. KMS key policy with "Principal": "*"                            │
│    → Always specify exact role/user ARNs                            │
│                                                                     │
│ 8. CloudTrail disabled or single-region only                       │
│    → Multi-region trail is mandatory                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

*Last reviewed against: CIS AWS Foundations Benchmark v1.5 | AWS Well-Architected Security Pillar | SOC 2 CC6 Controls*
*Recommended review cycle: Quarterly or after any significant infrastructure change*
