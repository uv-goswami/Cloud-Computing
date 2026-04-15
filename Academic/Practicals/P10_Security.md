# P10 — Cloud Security
**Track: Academic | Practical 10 of 10**

## Objective
Implement least-privilege IAM, S3 encryption, restrictive security groups, CloudTrail audit.

## Steps

### Part A: Least-Privilege IAM

```json
// Policy: S3 read-only on ONE specific bucket
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }]
}
```

Create IAM Role for EC2 (not user with access keys):
IAM → Roles → AWS Service → EC2 → Attach policy above

### Part B: S3 Encryption
S3 → Bucket → Properties → Default encryption → SSE-KMS → Create KMS key

### Part C: Restrictive Security Group
```
Inbound:  HTTPS 443 from 0.0.0.0/0   (public web)
Inbound:  SSH   22  from YOUR_IP/32  (only you)
Outbound: HTTPS 443 to 0.0.0.0/0    (outbound API calls)
Outbound: DNS   53  to 0.0.0.0/0    (name resolution)
Outbound: PG    5432 to 10.0.2.0/24  (DB in private subnet only)
```

### Part D: Enable CloudTrail
CloudTrail → Create Trail → All regions → S3 bucket for logs → Enable log file validation

## Viva Questions
1. **IAM User vs IAM Role?** User = permanent identity for a person. Role = temporary identity for services. Use roles on EC2 — never access keys.
2. **IAM evaluation logic?** Default deny → explicit allow → explicit deny wins (cannot be overridden).
3. **Why never store access keys on EC2?** If instance compromised, attacker gets permanent credentials. Roles give temporary auto-rotating credentials.
4. **What does CloudTrail log?** Every API call: who, when, from where, what action, success/fail. Stored as JSON in S3.
5. **SSE-KMS over SSE-S3?** Use KMS when: compliance requires decrypt audit trail, need to revoke access, need separate key policies per data type.
