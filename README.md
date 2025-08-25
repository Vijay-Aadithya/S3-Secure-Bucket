# S3 Secure Bucket Baseline (SSE-KMS, TLS-Only, Versioning, CloudTrail Data Events, S3 Inventory)

A hands-on project that hardens an S3 bucket for application/data storage:

- Private bucket with **Object Ownership = Bucket owner enforced** (ACLs off)  
- **Default encryption** with your **KMS CMK (SSE-KMS)** + **S3 Bucket Keys**  
- **Bucket policy** that enforces **TLS only** and **the exact KMS key**  
- **Versioning** + **Lifecycle** hygiene (expire old versions, abort incomplete multipart uploads)  
- **CloudTrail data events** for S3 reads/writes (encrypted)  
- **S3 Inventory** to a separate destination bucket  
- **CLI tests** that prove denies/allow  
- **Teardown & cost notes**

> ⚠️ **Redaction**: When publishing screenshots, mask account IDs and **KMS Key IDs/ARNs** (e.g., `arn:aws:kms:eu-west-2:****:key/****`). Never publish access keys.

---

## 0) Prerequisites

- Region used here: **eu-west-2** (London)  
- Admin user with **MFA**  
  ![MFA assigned](screenshots/Screenshot%20(150).png)
- (Optional) Zero-spend **budget guardrail**  
  ![Budget created](screenshots/Screenshot%20(151).png)
- (Optional) AWS CLI v2 configured:  
  ```bash
  aws configure   # region eu-west-2, output json


## 1) Create a customer-managed **KMS key**

- Navigate to **KMS → Customer managed keys → Create key**  
- Choose **Symmetric**  
- Set alias: `alias/s3-secure-data`  
- Add your admin user as **Key Admin** and **Key User**  
- Create the key  

![KMS key created](screenshots/Screenshot%20(158).png)

⚠️ Copy the **Key ARN** — needed for bucket policy. Redact it in screenshots.


## 2. Create the secure S3 bucket

- Console → **S3 → Create bucket** (Region: `eu-west-2`)
- **Block public access:** ON (all four)
- **Object ownership:** Bucket owner enforced (ACLs disabled) *(verify under Permissions after create)*
- **Bucket versioning:** **Enabled**
- **Default encryption:** **SSE-KMS** → select `alias/s3-secure-data` → **Enable S3 Bucket Keys**

![Block public & Versioning](screenshots/Screenshot%20(160).png)  
![Default SSE-KMS at create](screenshots/Screenshot%20(159).png)

---

## 3. Enforce TLS-only and the exact KMS key (bucket policy)

- S3 → **your bucket → Permissions → Bucket policy → Edit**
- Paste the policy below, replacing placeholders (`YOUR_BUCKET`, `REGION`, `ACCOUNT_ID`, `KMS_KEY_ID`).
- **Save changes**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::YOUR_BUCKET",
        "arn:aws:s3:::YOUR_BUCKET/*"
      ],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    },
    {
      "Sid": "DenyMissingEncryptionHeader",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET/*",
      "Condition": { "Null": { "s3:x-amz-server-side-encryption": "true" } }
    },
    {
      "Sid": "DenyNotUsingKMS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET/*",
      "Condition": {
        "StringNotEquals": { "s3:x-amz-server-side-encryption": "aws:kms" }
      }
    },
    {
      "Sid": "DenyWrongKMSKey",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption-aws-kms-key-id":
          "arn:aws:kms:REGION:ACCOUNT_ID:key/KMS_KEY_ID"
        }
      }
    }
  ]
}
```
**Screenshots:**  
- ![Policy editor](screenshots/Screenshot%20(161).png)  
- ![Policy saved](screenshots/Screenshot%20(163).png)

---

## 4. Configure lifecycle rules

Create two rules under **Management → Lifecycle rules**:

- **Rule:** `expire-noncurrent-30days`  
  - Action: Permanently delete noncurrent versions  
  - Days after noncurrent: **30**

- **Rule:** `abort-multipart-7d`  
  - Action: Abort incomplete multipart uploads  
  - Days after initiation: **7**

**Screenshots:**  
![Expire noncurrent rule wizard](screenshots/Screenshot%20(164).png)  
![Expire rule details](screenshots/Screenshot%20(165).png)  
![Abort multipart rule](screenshots/Screenshot%20(166).png)  
![Lifecycle rules list](screenshots/Screenshot%20(167).png)

---

## 5. Enable CloudTrail data events

### 5.1. Logs bucket policy
If using a dedicated logs bucket, ensure the bucket policy allows CloudTrail to write:

![Logs bucket policy](screenshots/Screenshot%20(170).png)

### 5.2. Create the trail
- **CloudTrail → Create trail** (multi-region if desired)  
- Use existing logs bucket or create a new one  
- Enable **SSE-KMS encryption** for log files  
- Enable **log file validation**

**Screenshots:**  
![Trail – storage & naming](screenshots/Screenshot%20(171).png)  
![Trail – encryption](screenshots/Screenshot%20(172).png)

### 5.3. Add S3 data events
- Add data event → **S3 → Select your secure bucket → Read & Write**

![Add S3 data events](screenshots/Screenshot%20(173).png)

### 5.4. Verify logging
- Go to **Trails list** and check status = **Logging**

**Screenshots:**  
![Trail logging](screenshots/Screenshot%20(174).png)  
![Trail logging (alt view)](screenshots/Screenshot%20(169).png)

---

## 6. Configure S3 Inventory

### 6.1. Destination bucket
Create a private destination bucket for reports (Block public access ON, ACLs off).

![Inventory destination bucket](screenshots/Screenshot%20(175).png)

### 6.2. Inventory configuration
On your secure bucket → **Management → Inventory → Create**:  
- **Name:** `encryption-and-replication`  
- **Destination:** your inventory bucket  
- **Frequency:** Daily  
- **Format:** CSV or Parquet  
- **Versions:** Current (or Current + Previous)  
- **Optional fields:** Encryption status, Replication status, Storage class, ETag, Object Lock, etc.  
- **Report encryption:** SSE-S3 or SSE-KMS

**Screenshots:**  
![Inventory configuration details](screenshots/Screenshot%20(176).png)  
![Inventory optional fields](screenshots/Screenshot%20(177).png)

**Reports appear at:**  

---

## 7. CLI Tests (Evidence)

> Replace `YOUR_BUCKET` and `YOUR_KEY_ARN` in the commands below.

### 7.1. Deny: missing SSE header
```bash
aws s3api put-object --bucket YOUR_BUCKET --key no-sse.txt --body no-sse.txt
```
### 7.2. Deny: wrong SSE (AES256)
```bash
aws s3api put-object --bucket YOUR_BUCKET --key sse-s3.txt --body sse-s3.txt --server-side-encryption AES256
```
### 7.3. Allow: correct SSE-KMS
```bash
aws s3api put-object --bucket YOUR_BUCKET --key ok.txt --body ok.txt \ --server-side-encryption aws:kms --ssekms-key-id YOUR_KEY_ARN
```
### 7.4. Versioning proof (v1, v2, list versions)
```bash
echo v1> vdemo.txt
aws s3api put-object --bucket YOUR_BUCKET --key version-demo.txt --body vdemo.txt \
  --server-side-encryption aws:kms --ssekms-key-id YOUR_KEY_ARN

echo v2> vdemo.txt
aws s3api put-object --bucket YOUR_BUCKET --key version-demo.txt --body vdemo.txt \
  --server-side-encryption aws:kms --ssekms-key-id YOUR_KEY_ARN

aws s3api list-object-versions --bucket YOUR_BUCKET --prefix version-demo.txt
```
### 7.5. ACLs blocked (public-read)
```bash
aws s3api put-object-acl --bucket YOUR_BUCKET --key ok.txt --acl public-read
```
