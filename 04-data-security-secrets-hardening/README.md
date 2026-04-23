# AWS: Data Security & Secrets Hardening (S3 · KMS · Macie · Secrets Manager)

## Overview

Hardened data security across the AWS storage layer: implemented SSE and attribute-based access control on S3/DynamoDB, automated sensitive data discovery using Amazon Macie with KMS-encrypted findings, and eliminated hardcoded credentials by migrating secrets to AWS Secrets Manager with Lambda-driven automated rotation.

## Projects Covered

| # | Project | Key Services |
|---|---------|-------------|
| 1 | [Data Encryption — KMS, S3 SSE, DynamoDB, Encryption SDK](#project-1-data-encryption--kms-s3-sse-dynamodb--aws-encryption-sdk) | KMS, S3, DynamoDB, EC2 |
| 2 | [S3 Bucket Policy Hardening — HTTPS & VPC Endpoint Enforcement](#project-2-s3-bucket-policy-hardening--https--vpc-endpoint-enforcement) | S3, KMS, VPC, EC2 |
| 3 | [Sensitive Data Discovery — Amazon Macie & KMS](#project-3-sensitive-data-discovery--amazon-macie--kms) | Macie, S3, KMS |
| 4 | [Secrets Management & Rotation — AWS Secrets Manager & Lambda](#project-4-secrets-management--rotation--aws-secrets-manager--lambda) | Secrets Manager, Lambda, EC2, CloudWatch |

---

## Project 1: Data Encryption — KMS, S3 SSE, DynamoDB & AWS Encryption SDK

### Objectives

- Create a KMS Key for encryption management
- Enable S3 Server-Side Encryption (SSE-KMS)
- Configure Attribute-Based Access Control (ABAC)
- Use the DynamoDB Encryption Client to encrypt database data
- Encrypt data programmatically with the AWS Encryption SDK

### Services Used

- AWS Key Management Service (KMS)
- Amazon S3
- Amazon DynamoDB
- Amazon EC2
- AWS Encryption SDK
- DynamoDB Encryption Client (Python)

The setup includes:
- An S3 bucket storing encrypted medical image objects
- An EC2 instance (`Image Viewer EC2 Instance`) with an IAM role (`viewerInstanceProfileForMedicalImages`)
- Two AWS KMS keys: `ImageAccessKey` and `DynamoDB_Direct_CMP_Key`
- Three DynamoDB tables demonstrating different encryption models
- AWS Encryption SDK applied to local file encryption

---

### Task 1: Create a KMS Key for Encryption Management

Navigate to **KMS → Create key** and configure:

| Setting | Value |
|---------|-------|
| Key type | Symmetric |
| Key material origin | KMS |
| Regionality | Single-Region key |
| Alias | `ImageAccessKey` |
| Description | `This key is used to encrypt and decrypt` |
| Tag key | `ImageType` |
| Tag value | `MRI` |

After creation:
- Copy the **ARN** from the General configuration section.
- Enable **Automatic key rotation** under the Key material and rotations tab.

> **Task complete:** KMS key `ImageAccessKey` created, tagged, and configured with automatic rotation.

---

### Task 2: Enable S3 Server-Side Encryption

Navigate to **S3 → [your bucket] → Properties → Default encryption → Edit**:

| Setting | Value |
|---------|-------|
| Encryption type | SSE-KMS |
| AWS KMS key | Enter KMS key ARN (from Task 1) |
| Bucket Key | Enabled |

> **Note:** This setting applies to new objects only. Existing objects retain their prior encryption state.

> **Task complete:** S3 bucket configured with SSE-KMS encryption using `ImageAccessKey`.

---

### Task 3: Add New Objects to the Secured Bucket

Connect to the EC2 instance via **AWS Systems Manager Fleet Manager (RDP)**.

1. Launch the **Image Transfer** tool from the desktop shortcut.
2. Transfer XRay images for a new patient into the S3 bucket.
3. Verify the uploaded objects in S3:
   - Navigate to the object → **Properties → Server-side encryption settings**
   - Confirm: `Encryption type: SSE-KMS`, `Bucket Key: Enabled`, and the correct KMS key ARN.

> **Note:** Encrypted and unencrypted objects can coexist in the same bucket. Default bucket encryption governs new objects only.

> **Task complete:** XRay images uploaded and encrypted with `ImageAccessKey`.

---

### Task 4: Configure Attribute-Based Access Control (ABAC)

The IAM role `viewerInstanceProfileForMedicalImages` contains a **Deny** condition on `kms:Decrypt` when the KMS key tag `ImageType` does not equal `XRAY`:

```json
{
  "Condition": {
    "StringNotEqualsIgnoreCase": {
      "aws:ResourceTag/ImageType": "XRAY"
    }
  },
  "Action": ["kms:Decrypt"],
  "Resource": "arn:aws:kms:us-west-2:668122******:key/*",
  "Effect": "Deny",
  "Sid": "KMSDecryptDeny"
}
```

To grant access, update the KMS key tag without touching IAM policies or role assignments:

Navigate to **KMS → ImageAccessKey → Tags → Edit**:
- Change tag value from `MRI` → `XRAY`

> **Note:** This is ABAC in action — access was granted purely by updating a resource tag, with no IAM policy changes required.

> **Task complete:** Medical technicians can now decrypt and view XRAY images.

---

### Task 5: Use the DynamoDB Encryption Client

Three DynamoDB tables demonstrate different encryption strategies using the `SalesAnalytics.py` script (run in PyCharm via Fleet Manager RDP).

#### Table 1 — DynamoDB Server-Side Encryption (AWS-owned key)

Standard `put_item` calls with no client-side encryption. DynamoDB automatically encrypts at rest using an AWS-owned key.

```python
dynamodb_resource.Table('Table1').put_item(Item={ ... })
```

**Result:** 1,033 European Office Supply records inserted. All attributes are readable (SSE auto-decrypts on read).

#### Table 2 — DynamoDB Encryption Client with KMS Direct CMP

Create a second KMS key (`DynamoDB_Direct_CMP_Key`), then use the `AwsKmsCryptographicMaterialsProvider`:

```python
spotlight_lab_direct_kms_cmp = AwsKmsCryptographicMaterialsProvider('KMS_KEY_ARN')
encrypted_table_access = EncryptedTable(
    table=spotlightLabTable2,
    materials_provider=spotlight_lab_direct_kms_cmp
)
encrypted_table_access.put_item(Item={ ... })
```

**Result:** 93 North American records inserted. Only the partition key and sort key are readable in the console. All other attributes remain encrypted. Two additional DynamoDB Encryption SDK attributes (`amzn-ddb-map-desc`, `amzn-ddb-map-sig`) are added automatically.

#### Table 3 — DynamoDB Encryption Client with Wrapped Materials Provider (on-premises keys)

Uses locally-generated AES wrapping and HmacSHA512 signing keys to simulate an on-premises key management system:

```python
private_wrapping_key = JceNameLocalDelegatedKey(
    key=get_random_bytes(32), algorithm='AES',
    key_type=EncryptionKeyType.SYMMETRIC, key_encoding=KeyEncodingType.RAW
)
private_signing_key = JceNameLocalDelegatedKey(
    key=get_random_bytes(32), algorithm='HmacSHA512',
    key_type=EncryptionKeyType.SYMMETRIC, key_encoding=KeyEncodingType.RAW
)
spotlight_lab_wrapped_cmp = WrappedCryptographicMaterialsProvider(
    wrapping_key=private_wrapping_key,
    unwrapping_key=private_wrapping_key,
    signing_key=private_signing_key
)
encrypted_table_access = EncryptedTable(
    table=dynamodb_resource.Table('Table3'),
    materials_provider=spotlight_lab_wrapped_cmp
)
```

**Result:** 1,345 low-revenue Vegetable records inserted. Primary key attributes are readable; all others are encrypted until decrypted via the on-prem keys.

> **Task complete:** Three DynamoDB tables populated using three distinct encryption strategies.

---

### Task 6 (Sub-task 9): Encrypt Local Data with the AWS Encryption SDK

Create a third KMS key (`AWS_Encryption_SDK_Key`), then use the AWS Encryption SDK to encrypt local files:

```python
client = aws_encryption_sdk.EncryptionSDKClient(
    commitment_policy=CommitmentPolicy.REQUIRE_ENCRYPT_REQUIRE_DECRYPT
)
master_key_provider = aws_encryption_sdk.StrictAwsKmsMasterKeyProvider(
    key_ids=['KMS_KEY_ARN']
)
ciphertext, encryptor_header = client.encrypt(
    source=recipe_text,
    key_provider=master_key_provider
)
```

**Result:** All files in `C:\Personal\Secret Family Recipes\` are encrypted and saved as `[ENCRYPTED]-<filename>` binary files.

> **Warning:** If the KMS key is deleted, the encrypted files cannot be recovered.

> **Task complete:** Local data encrypted programmatically using the AWS Encryption SDK and KMS.

---

## Project 2: S3 Bucket Policy Hardening — HTTPS & VPC Endpoint Enforcement

### Objectives

- Configure the bucket policy to enforce HTTPS connections only
- Configure the bucket policy to restrict access through the VPC gateway endpoint only
- Configure the bucket policy to enforce an accepted encryption method and key on uploads
- Test all requirements using the AWS CLI

### Services Used

- Amazon S3
- AWS KMS (Green Key, Red Key)
- Amazon VPC (Gateway Endpoint)
- Amazon EC2 (Public Instance, Private Instance)
- AWS Systems Manager (Session Manager)

### Architecture Overview

- An S3 bucket accessible from two EC2 instances
- A VPC with one **public subnet** (internet access via Internet Gateway) and one **private subnet** (S3 access via VPC Gateway Endpoint)
- An EC2 instance in each subnet with an IAM role granting `s3:ListObjects`, `s3:PutObject`, `s3:GetObject`
- Two KMS keys: `Green Key` and `Red Key`

---

### Task 1: Test Connectivity and Upload a Test Object

Connect to **Public-Instance** via Session Manager and set environment variables:

```bash
lab_bucket=mybucket-48523xxxx
lab_region=us-west-2
```

Upload a test object:

```bash
aws --endpoint-url https://s3.$lab_region.amazonaws.com s3api put-object \
  --bucket $lab_bucket \
  --body object01.txt \
  --key object01.txt
```

Expected output confirms SSE-KMS encryption on the upload:

```json
{
  "SSEKMSKeyId": "arn:aws:kms:us-west-2:111122223333:key/...",
  "ETag": "\"f9a29703c427f46f94fd76e5baf2222f\"",
  "ServerSideEncryption": "aws:kms"
}
```

Connect to **Private-Instance** and set variables:

```bash
lab_bucket=INSERT_LAB_BUCKET_NAME
lab_region=INSERT_LAB_REGION_CODE
kms_green_key_id=INSERT_KMS_GREEN_KEY_ID
kms_red_key_id=INSERT_KMS_RED_KEY_ID
```

List objects from Private-Instance to confirm VPC endpoint connectivity:

```bash
aws --endpoint-url https://s3.$lab_region.amazonaws.com s3api list-objects --bucket $lab_bucket
```

> **Task complete:** Connectivity verified from both instances. Bucket has no policy applied yet (all access controlled via IAM).

---

### Task 2: Enforce HTTPS-Only Access

Apply the following bucket policy to deny all HTTP requests:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow SSL Requests Only",
      "Action": "s3:*",
      "Effect": "Deny",
      "Resource": [
        "arn:aws:s3:::YOUR_BUCKET_NAME",
        "arn:aws:s3:::YOUR_BUCKET_NAME/*"
      ],
      "Principal": "*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**Verification:**

| Test | Endpoint | Expected Result |
|------|----------|----------------|
| Public-Instance — HTTPS | `https://s3...` | ✅ Success |
| Public-Instance — HTTP | `http://s3...` | ❌ `AccessDenied` |
| Private-Instance — HTTP | `http://s3...` | ❌ `AccessDenied` |

> **Task complete:** HTTPS enforced on the bucket via bucket policy.

---

### Task 3: Enforce VPC Endpoint-Only Access

Add a second statement to deny `GetObject`, `PutObject`, and `ListBucket` for any request not originating from the VPC gateway endpoint:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSLRequestsOnly",
      "Action": "s3:*",
      "Effect": "Deny",
      "Resource": [
        "arn:aws:s3:::YOUR_BUCKET_NAME",
        "arn:aws:s3:::YOUR_BUCKET_NAME/*"
      ],
      "Principal": "*",
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    },
    {
      "Sid": "Restrict Access only from VPC Endpoint",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Effect": "Deny",
      "Resource": [
        "arn:aws:s3:::YOUR_BUCKET_NAME",
        "arn:aws:s3:::YOUR_BUCKET_NAME/*"
      ],
      "Principal": { "AWS": "INSERT_EC2_ROLE_ARN" },
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "INSERT_VPC_ENDPOINT_ID"
        }
      }
    }
  ]
}
```

> **Caution:** Only use specific actions (`GetObject`, `PutObject`, `ListBucket`) in the VPC endpoint statement. Using a wildcard `*` risks locking yourself out of the bucket entirely.

**Verification:**

| Test | Instance | Expected Result |
|------|----------|----------------|
| `ListObjects` | Public-Instance | ❌ `AccessDenied` |
| `PutObject` | Public-Instance | ❌ `AccessDenied` |
| `GetObject` | Public-Instance | ❌ `AccessDenied` |
| `ListObjects` | Private-Instance | ✅ Success (via VPC endpoint) |
| `PutObject` | Private-Instance | ✅ Success (via VPC endpoint) |

> **Task complete:** S3 access restricted to the VPC gateway endpoint in the private subnet only.

---

## Project 3: Sensitive Data Discovery — Amazon Macie & KMS

### Objectives

- Enable and access Amazon Macie in the AWS account
- Implement encryption for sensitive findings using AWS KMS
- Configure Macie's discovery results repository in Amazon S3
- Create sensitive data discovery jobs in Macie
- Analyze Macie findings in detail

### Services Used

- Amazon Macie
- Amazon S3
- AWS Key Management Service (KMS)

### Architecture Overview

Two pre-configured S3 buckets:

| Bucket | Purpose | Config |
|--------|---------|--------|
| `macie-data-*` | Source data (contains `UserData.csv` with synthetic PII) | Versioning enabled, Block Public Access |
| `macie-data-results-*` | Stores Macie discovery findings | Versioning, encryption, Block Public Access |

> **Note:** The sample data file (`UserData.csv`) contains **synthetic data only** — no real personal information. It includes mock credit card numbers and social security numbers for discovery testing.

---

### Task 1: Enable Amazon Macie

Navigate to **Amazon Macie → Get started → Enable Macie**.

Verify the two pre-configured S3 buckets are present in the S3 console.

> **Task complete:** Macie enabled and source/results buckets confirmed.

---

### Task 2: Create and Configure a KMS Key for Macie

Navigate to **KMS → Create key**:

| Setting | Value |
|---------|-------|
| Key type | Symmetric |
| Key usage | Encrypt and decrypt |
| Alias | `macie-results-key` |
| Description | `Key for encrypting Macie discovery results` |

Grant key administrative permissions to the current user/role. Keep key usage permissions at defaults.

> **Task complete:** KMS key `macie-results-key` created for encrypting Macie findings.

---

### Task 3: Configure Macie Discovery Results Repository

Navigate to **Macie → Settings → Discovery results → Configure now**:

1. Select **Existing bucket** → choose `macie-data-results-*`
2. Under **Advanced**, select `macie-results-key` as the KMS key
3. Click **View policy** → copy the displayed bucket policy → apply it to the results bucket in S3
4. Click **View policy** from the KMS key section → copy the KMS policy statement → append it to the existing `macie-results-key` policy (do **not** overwrite existing statements)
5. Save all changes in both S3, KMS, and Macie

> **Task complete:** Macie configured to store encrypted findings in `macie-data-results-*` using `macie-results-key`.

---

### Task 4: Create and Run a Sensitive Data Discovery Job

Navigate to **Macie → Jobs → Create job**:

| Setting | Value |
|---------|-------|
| S3 bucket | `macie-data-*` (source bucket) |
| Discovery type | One-time job |
| Managed data identifiers | Recommended |
| Job name | `Sensitive data discovery job` |
| Description | `One-time data discovery job` |

Submit the job and wait 10–15 minutes for status to change from **Running** to **Complete**.

> **Task complete:** Macie discovery job created and executed successfully.

---

### Task 5: Review Macie Findings

From the completed job, use **Show results → Show findings**.

**Finding:** `SensitiveData:S3Object/Multiple` — Severity: **High**

| Category | Type | Count |
|----------|------|-------|
| Financial information | Credit card numbers | 30 |
| Personal information | USA Social Security numbers | 30 |
| **Total** | | **60 occurrences** |

Each JSON finding record includes:
- Column name (e.g., `CCN`)
- Column number
- Row number where sensitive data was found

**Downloading findings — three options:**

1. **Download JSON for a specific data type** — from the occurrences window
2. **Download complete finding details** — via the Finding ID at the top of the details panel
3. **Access via S3** — use the `Detailed result location` S3 link in the details panel

> **Task complete:** Macie findings reviewed; 60 sensitive data occurrences identified and analyzed across credit card and SSN categories.

---

## Project 4: Secrets Management & Rotation — AWS Secrets Manager & Lambda

### Objectives

- Create an AWS Secrets Manager secret to store and protect sensitive information
- Programmatically access the secret value via CLI and SDK
- Centrally manage authorization using Resource-Based Policies
- Schedule automated rotation of secrets using AWS Lambda

### Services Used

- AWS Secrets Manager
- Amazon EC2 (Session Manager)
- AWS Lambda
- Amazon CloudWatch

### Task 1: Create a Secret in AWS Secrets Manager

Navigate to **Secrets Manager → Store a new secret**:

| Setting | Value |
|---------|-------|
| Secret type | Other type of secret |
| Key 1 | `username` → your username |
| Key 2 | `password` → your password |
| Encryption key | `aws/secretsmanager` (default) |
| Secret name | `project-secret` |
| Description | `secret created with AWS Secrets Manager` |

> **Note:** The default encryption key (`aws/secretsmanager`) is an AWS-managed KMS key. Secrets Manager handles all key management automatically.

Skip rotation configuration for now. Review and store.

> **Task complete:** Secret `project-secret` created and encrypted.

---

### Task 2: Review the Stored Secret

Navigate to **Secrets Manager → project-secret**:

- Review **Secret name**, **Description**, **Encryption key**
- Under the **Rotation** tab: rotation is currently **disabled**
- Under the **Versions** tab: review modification and retrieval timestamps
- Under **Overview**, click **Retrieve secret value** to view the JSON key-value pairs

> **Task complete:** Secret details explored and structure understood.

---

### Task 3: Programmatically Access the Secret

#### Via AWS CLI (EC2 Session Manager)

Connect to the EC2 instance via **Systems Manager → Session Manager**:

```bash
aws secretsmanager get-secret-value --secret-id project-secret --region us-west-2
```

Expected output:

```json
{
  "Name": "project-secret",
  "VersionId": "63456914-bc83-4103-8e00-5910ae4511fa",
  "SecretString": "{\"username\":\"Muser\",\"password\":\"Pro******\"}",
  "VersionStages": ["AWSCURRENT"],
  "CreatedDate": 1740416113.575,
  "ARN": "arn:aws:secretsmanager:us-west-2:912556106409:secret:project-secret-a1ctwN"
}
```

#### Via AWS SDK (Python / Boto3)

Install Boto3:

```bash
pip3 install boto3
```

Create `secret.py`:

```python
import boto3
from botocore.exceptions import ClientError

def get_secret():
    secret_name = "project-secret"
    region_name = "us-west-2"

    session = boto3.session.Session()
    secrets = session.client(
        service_name='secretsmanager',
        region_name=region_name,
    )

    try:
        get_secret_value_response = secrets.get_secret_value(
            SecretId=secret_name
        )
    except ClientError as e:
        error_code = e.response['Error']['Code']
        if error_code == 'ResourceNotFoundException':
            print(f"The requested secret {secret_name} was not found.")
        elif error_code == 'InvalidRequestException':
            print(f"The request was invalid due to: {e}")
        elif error_code == 'InvalidParameterException':
            print(f"The request had invalid params: {e}")
        elif error_code == 'DecryptionFailure':
            print(f"The requested secret can't be decrypted using the provided KMS key: {e}")
        elif error_code == 'InternalServiceError':
            print(f"An error occurred on the service side: {e}")
        else:
            print(f"An unexpected error occurred: {e}")
    else:
        if 'SecretString' in get_secret_value_response:
            secret = get_secret_value_response['SecretString']
            print(f"Secret retrieved: {secret}")
        elif 'SecretBinary' in get_secret_value_response:
            secret = get_secret_value_response['SecretBinary']
            print(f"Binary secret retrieved: {secret}")
        else:
            print("Secret data not found.")

if __name__ == "__main__":
    get_secret()
```

Run:

```bash
python3 secret.py
```

Expected output:

```
Secret retrieved: {"username":"Muser","password":"Pro********"}
```

> **Task complete:** Secret retrieved via both AWS CLI and Python Boto3 SDK.

---

### Task 4: Configure Lambda Permissions with Resource-Based Policies

Navigate to **Lambda → Create function** named `SecretRotationFunction`.

Add the following rotation logic to the Lambda function:

```python
import boto3
import json

def lambda_handler(event, context):
    client = boto3.client('secretsmanager')
    secret_arn = event['SecretId']

    try:
        get_secret_value_response = client.get_secret_value(SecretId=secret_arn)

        if 'SecretString' in get_secret_value_response:
            secret = json.loads(get_secret_value_response['SecretString'])
        else:
            raise Exception("Secret is in binary format, which is not supported in this example")

        existing_username = secret['username']

        random_password_response = client.get_random_password(
            PasswordLength=16,
            ExcludeCharacters='/"',
            IncludeSpace=False
        )
        new_password = random_password_response['RandomPassword']

        new_secret = {
            'username': existing_username,
            'password': new_password
        }

        client.put_secret_value(
            SecretId=secret_arn,
            SecretString=json.dumps(new_secret),
        )

        return {
            'statusCode': 200,
            'body': json.dumps('Secret rotation successful')
        }

    except Exception as e:
        print(f"Error rotating secret: {str(e)}")
        raise e
```

Grant Secrets Manager permission to invoke the Lambda function:

Navigate to **Lambda → SecretRotationFunction → Configuration → Permissions → Resource-based policy statements → Add permissions**:

| Setting | Value |
|---------|-------|
| Principal type | AWS Service |
| Service | Secrets Manager |
| Statement ID | `secrets` |
| Action | `lambda:InvokeFunction` |

> **Task complete:** Resource-Based Policy configured to allow Secrets Manager to invoke `SecretRotationFunction`.

---

### Task 5: Configure Automatic Secret Rotation

Navigate to **Secrets Manager → project-secret → Rotation tab → Edit rotation**:

| Setting | Value |
|---------|-------|
| Automatic rotation | Enabled |
| Rotation schedule | Every 23 hours |
| Rotation function | `SecretRotationFunction` |

After saving, click **Retrieve secret value** — the password will have been automatically rotated by the Lambda function.

**Verify the rotation via CLI:**

```bash
aws secretsmanager get-secret-value --secret-id project-secret --region us-west-2
```

Expected output shows a new `VersionId` and rotated password:

```json
{
  "Name": "project-secret",
  "VersionId": "2390d349-cd2a-4b55-af13-af48ed3c1b11",
  "SecretString": "{\"username\":\"Muser\",\"password\":\"pO<d********\"}",
  "VersionStages": ["AWSCURRENT"],
  "CreatedDate": 1740422757.933,
  "ARN": "arn:aws:secretsmanager:us-west-2:370508068305:secret:project-secret-g1KrA2"
}
```

Review version history under **Secrets Manager → project-secret → Versions tab**.

> **Task complete:** Automated secret rotation configured and verified. Version history reviewed.

---

### Task 6: Monitor Rotation via CloudWatch Logs

Navigate to **Lambda → SecretRotationFunction → Monitor → View CloudWatch logs**.

In the **Log streams** tab, select the stream with the latest event time to review rotation execution logs.

> **Task complete:** CloudWatch logs reviewed for Lambda rotation events.

---

## Summary of Accomplishments

| Domain | What Was Implemented |
|--------|---------------------|
| **KMS** | Created and tagged symmetric CMKs with automatic rotation; used for S3 SSE, DynamoDB client-side encryption, Macie findings, and Encryption SDK |
| **S3 Hardening** | Enforced HTTPS-only access and VPC endpoint-only access via bucket policies |
| **S3 SSE** | Configured SSE-KMS as the default encryption for new objects |
| **ABAC** | Granted/denied KMS decryption access based on resource tags — no IAM policy changes required |
| **DynamoDB Encryption** | Demonstrated three encryption models: AWS-owned SSE, DynamoDB Encryption Client with KMS CMP, and Wrapped CMP with on-prem keys |
| **AWS Encryption SDK** | Encrypted arbitrary local files programmatically using a KMS key |
| **Amazon Macie** | Automated PII discovery (credit cards, SSNs) in S3 with KMS-encrypted findings stored in a dedicated results bucket |
| **Secrets Manager** | Eliminated hardcoded credentials; implemented programmatic access via CLI and Boto3; automated password rotation with Lambda on a 23-hour schedule |
| **CloudWatch** | Monitored Lambda-driven secret rotation via log streams |

---
