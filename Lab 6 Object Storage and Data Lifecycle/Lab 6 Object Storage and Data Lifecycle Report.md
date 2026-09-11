# Lab 5:Monitoring, Logging and Incident Detection

**Course:** Cloud Computing Security Essentials  
**Lab:** 5 Monitoring, Logging and Incident Detection  
**Date:** 3 September 2026

## 1. Introduction and objectives

This lab examined object-storage security throughout the data lifecycle using an Amazon S3-compatible LocalStack environment. The work classified hospital data, reproduced a public-bucket breach, applied public-access guardrails and least-privilege policies, compared IAM and bucket-policy decisions, enforced default SSE-KMS encryption, and tested time-limited sharing.

The second session addressed data state and disposal. It demonstrated versioning, delete markers, object-level data remanence, lifecycle retention, and cryptographic erasure through KMS key disablement and scheduled deletion. Together, these controls support confidentiality, integrity, controlled retention, and provable disposal of sensitive cloud data.

## 2. Lab environment and setup

LocalStack was started with IAM enforcement enabled. The AWS CLI was configured to use the local endpoint and the LocalStack account ID `000000000000` was used in the bucket-policy ARNs.

```bash
docker run -d --name localstack -p 4566:4566 \
  -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
  -e ENFORCE_IAM=1 localstack/localstack-pro:latest

export EP='--endpoint-url=http://localhost:4566'
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
aws $EP sts get-caller-identity
```

Evidence: *[Insert link to LocalStack startup and `sts get-caller-identity` screenshot.]*

## 3. Session A — Object storage and exposure control

### Task 1: Data classification and storage

The patient-records bucket contained three objects. Object keys use a flat namespace: `public/`, `internal/`, and `confidential/` are key prefixes rather than real folders. This is security-relevant because access policies must scope the prefix precisely.

| Classification | Who may read it | Impact if leaked | Control applied |
| --- | --- | --- | --- |
| `public` | Anyone authorised to view public hospital information | Low; may cause confusion if altered but is not sensitive | Object tag `classification=public`; deliberately scoped distribution policy when needed |
| `internal` | Hospital staff and approved internal systems | Moderate; operational information could assist social engineering or disrupt operations | Least-privilege bucket policy permitting the account only on `internal/*` |
| `confidential` | Authorised clinical personnel and tightly controlled services only | High; disclosure of patient data can cause privacy harm and regulatory non-compliance | Object tag `classification=confidential`, explicit bucket-policy denial, SSE-KMS, versioning, lifecycle retention |

The following commands created and checked the objects and the confidential-data tag:

```bash
aws $EP s3api list-objects-v2 --bucket $BUCKET \
  --query 'Contents[].[Key,Size]' --output table
aws $EP s3api get-object-tagging --bucket $BUCKET \
  --key confidential/record.txt
```

Evidence: *[Insert Task 1 list-objects and confidential-object tagging screenshots.]*

### Task 2: Public-bucket breach

A deliberately unsafe bucket policy allowed `s3:GetObject` to `"Principal": "*"` on `arn:aws:s3:::$BUCKET/*`. An unauthenticated `curl` request to `confidential/record.txt` returned HTTP 200 and the patient record. This reproduced a cloud-data exposure caused by authorisation configuration alone; no exploit was required.

```bash
curl -s -o leaked.txt -w 'HTTP %{http_code}\n' \
  http://localhost:4566/$BUCKET/confidential/record.txt
cat leaked.txt
```

Evidence: *[Insert Task 2 screenshot showing HTTP 200 and the leaked record.]*

### Task 3: Block Public Access and least privilege

The unsafe policy was removed and all four Block Public Access settings were enabled: `BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, and `RestrictPublicBuckets`. On AWS, `BlockPublicPolicy=true` rejects a bucket policy that would make the bucket public; `RestrictPublicBuckets=true` also prevents public/cross-account access granted by a public policy. The ACL flags provide parallel protection against public ACLs.

```bash
aws $EP s3api delete-bucket-policy --bucket $BUCKET
aws $EP s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws $EP s3api get-public-access-block --bucket $BUCKET
```

LocalStack may store this configuration without fully enforcing it. If the attempted re-application of the public policy or anonymous request still succeeded locally, that is a LocalStack limitation rather than the expected AWS result. The captured configuration remains evidence that the correct AWS guardrail was applied.

The replacement policy granted only the LocalStack account root principal access to `internal/*`, not the entire bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AccountReadInternalOnly",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::000000000000:root"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/internal/*"
  }]
}
```

Evidence: *[Insert Task 3 Block Public Access output and anonymous-read re-test screenshot.]*

### Task 4: Identity policy versus resource policy

`DataAnalyst` received an identity-based IAM policy allowing `s3:GetObject` and `s3:ListBucket` on all resources. The bucket resource policy then allowed the analyst to read `internal/*` but explicitly denied `s3:*` on `confidential/*`.

| Request | IAM policy | Bucket policy | Expected decision | Reason |
| --- | --- | --- | --- | --- |
| Read `internal/roster.txt` | Allow | Allow `AllowAnalystInternal` | Allowed | Both policies permit the action; no explicit deny matches. |
| Read `confidential/record.txt` | Allow | Explicit deny `DenyAnalystConfidential` | Denied | An explicit resource-policy deny overrides every allow. |

The evaluation order is default deny, then any matching explicit deny, then matching allow. If LocalStack did not enforce the denial even with `ENFORCE_IAM=1`, the two policy documents and this evaluation record demonstrate the intended AWS decision.

Evidence: *[Insert Task 4 analyst attempts, or both policy documents and the written evaluation if LocalStack did not enforce IAM.]*

## 4. Session B — Encryption, retention, and disposal

### Task 5: Default SSE-KMS encryption

A customer-managed KMS key was configured as the bucket's default encryption key with `BucketKeyEnabled=true`. The confidential version-2 object was uploaded without encryption options; `head-object` should show `aws:kms`, the configured key ID, and the bucket-key setting. This proves the bucket, rather than the uploader, enforced encryption at rest.

```bash
aws $EP s3api get-bucket-encryption --bucket $BUCKET
aws $EP s3api head-object --bucket $BUCKET --key confidential/record-v2.txt \
  --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]' --output text
```

`BucketKeyEnabled` optimises envelope encryption by reducing KMS calls; it does not weaken the encryption of the objects.

Evidence: *[Insert Task 5 `head-object` screenshot showing `aws:kms` and the key ID.]*

### Task 6: Presigned access and the SecureTransport condition

A presigned URL delegated read access to one object for 60 seconds. Its signature binds the permitted request to the signer and object, while `Expires` or `X-Amz-Expires` limits the period in which it is valid. Anyone who possesses a valid URL before it expires can use the authority it grants, so it must be treated as a secret and shared only through an appropriate channel.

The `aws:SecureTransport=false` deny policy was then applied. Since the LocalStack endpoint uses plain HTTP, the condition matched every local request and the bucket became inaccessible until the policy was deleted. On real AWS S3 HTTPS endpoints, legitimate HTTPS requests set this condition to true and are not denied. This demonstrates that a condition key must be validated in the environment where the policy runs.

Evidence: *[Insert Task 6 presigned-URL result and bucket-wide failure/recovery screenshot.]*

### Task 7: Versioning, delete markers, and remanence

Versioning was enabled and two revisions of `confidential/record.txt` were uploaded. A normal `delete-object` operation then placed a delete marker as the latest version. The ordinary object read failed, but the original version with ID `null` could still be retrieved and contained the original diagnosis. This is object-level data remanence.

```bash
aws $EP s3api list-object-versions --bucket $BUCKET \
  --prefix confidential/record.txt
aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt \
  --version-id null recovered.txt
```

Deleting the `null` version permanently removes that particular old object version, but every other version and delete marker must also be enumerated and deleted before the record is genuinely removed.

Evidence: *[Insert Task 7 version listing, delete marker, and recovered original-record screenshot.]*

### Task 8: Lifecycle, retention, and cryptographic erasure

An auditable lifecycle policy was applied. It expires current confidential objects after 365 days, noncurrent versions after 30 days, and aborts incomplete multipart uploads after 7 days.

| Rule | Scope | Retention action |
| --- | --- | --- |
| `RetireConfidentialRecords` | `confidential/` prefix | Expire current versions after 365 days and noncurrent versions after 30 days |
| `AbortIncompleteUploads` | Entire bucket | Abort incomplete multipart uploads after 7 days |

Finally, the bucket KMS key was disabled and scheduled for deletion after seven days. Because encrypted objects depend on that key to decrypt their data keys, destroying the key makes all ciphertext encrypted under it unrecoverable. This is cryptographic erasure. LocalStack may not re-check a disabled KMS key when S3 reads an object; where that occurred, the KMS encrypt → disable-key → decrypt sequence should instead be recorded as the evidence of failed decryption.

Evidence: *[Insert Task 8 lifecycle-rule output, KMS key state after scheduled deletion, and failed read/decrypt screenshot.]*

## 5. Answers to short-answer questions

### Q1. Which single element caused the Task 2 exposure, and why is it more dangerous on a bucket policy than an over-broad IAM policy?

`"Principal": "*"` caused the exposure because it grants the statement to every principal, including unauthenticated internet users. In a bucket policy it attaches directly to the data resource and can make every matching object public. An over-broad IAM policy is still dangerous, but it is attached to one named identity or role; its scope is normally limited to principals that possess those credentials. A public bucket policy removes that identity boundary altogether.

### Q2. Explain identity-based and resource-based policies. Which policy decided each Task 4 request?

An identity-based policy is attached to an IAM user, group, or role and specifies what that principal can do. A resource-based policy is attached to a resource, such as an S3 bucket, and specifies who can access that resource and under what conditions. For the analyst's internal-object request, the IAM allow and bucket-policy allow both applied, so it was allowed. For the confidential-object request, the resource-based statement `DenyAnalystConfidential` decided the result because an explicit deny overrides the IAM allow.

### Q3. What is the difference between a guardrail and a control, and why does it matter at scale?

A control can detect, permit, deny, or respond to a particular condition. A guardrail is a preventative boundary that constrains what other policies or users can configure. Block Public Access is a guardrail because it prevents a future public policy or ACL from taking effect, even when an engineer makes a mistake. This matters in a large organisation because it provides consistent protection across many engineers and buckets instead of relying on every individual configuration review or a later detective alert.

### Q4. Does default SSE-KMS protect the confidential record from the Task 4 analyst?

Not by itself. SSE-KMS encrypts the object at rest: S3 encrypts the data before storage and decrypts it for an authorised read, using KMS-managed keys and permissions. It protects against unauthorised access to stored media and supports key-control and cryptographic erasure. It does not prevent a principal that S3 authorises to read the object from receiving plaintext. In Task 4, the explicit bucket-policy deny—not SSE-KMS—prevents the analyst from reading the confidential record.

### Q5. Why is `delete-object` alone not compliant with a right-to-erasure request, and what makes deletion provable?

With versioning enabled, `delete-object` creates a delete marker rather than erasing earlier versions. Task 7 proved this because the original `null` version remained retrievable after deletion. One mechanism for provable deletion is to enumerate and permanently delete every object version and delete marker, then retain the deletion audit evidence. A second is cryptographic erasure: destroy or render unusable the customer-managed KMS key that protects the object ciphertext. Lifecycle rules can automate version expiry, but their configuration and completion evidence must be retained.

### Q6. Name three commands whose output is compliance evidence and state the control each proves.

| Command | Compliance control evidenced |
| --- | --- |
| `aws $EP s3api get-public-access-block --bucket $BUCKET` | All four public-access guardrail settings are configured. |
| `aws $EP s3api get-bucket-encryption --bucket $BUCKET` | Default SSE-KMS encryption and the customer-managed key are configured. |
| `aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET` | The formal retention and noncurrent-version disposal rules are configured. |
| `aws $EP s3api list-object-versions --bucket $BUCKET --prefix confidential/record.txt` | Versions and delete markers are visible for retention/erasure verification. |
| `aws $EP kms describe-key --key-id $KEY_ID` | The KMS key state and scheduled cryptographic-erasure status are recorded. |

## 6. Verification command and output

Run the following after completing the tasks, then paste the actual output below it. This is the final evidence of the bucket's security posture.

```bash
echo "=== IKB42603 Lab 6 verification: $BUCKET ==="
aws $EP s3api get-public-access-block --bucket $BUCKET \
  --query 'PublicAccessBlockConfiguration' --output text
aws $EP s3api get-bucket-versioning --bucket $BUCKET --output text
aws $EP s3api get-bucket-encryption --bucket $BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.[SSEAlgorithm,KMSMasterKeyID]' \
  --output text
aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' --output text
aws $EP kms describe-key --key-id $KEY_ID --query 'KeyMetadata.KeyState' --output text
```

```text
=== IKB42603 Lab 6 verification: [paste bucket name] ===
[Paste actual command output here]
```

## 7. Best-practices checklist

- [x] Every stored object was assigned a classification tag.
- [x] The public-policy exposure was tested, removed, and replaced with least-privilege prefix scoping.
- [x] All four Block Public Access flags were enabled.
- [x] IAM and resource-policy evaluation, including explicit deny precedence, was demonstrated.
- [x] Default `aws:kms` encryption was applied using a customer-managed key.
- [x] Delegated sharing used a time-limited presigned URL rather than a permanent public object.
- [x] Versioning and delete-marker remanence were demonstrated.
- [x] Lifecycle rules expressed retention, and KMS key deletion provided cryptographic erasure.

## 8. Conclusion

The lab followed a sensitive patient record from classification and controlled access through encryption, versioning, retention, and disposal. It showed that most object-storage breaches arise from incorrect authorisation, especially public bucket policies, while encryption at rest cannot replace access control. It also showed that deletion in object storage is not necessarily destruction: version history must be managed deliberately, and cryptographic erasure provides the strongest scalable assurance when physical storage media is outside the organisation's control.
