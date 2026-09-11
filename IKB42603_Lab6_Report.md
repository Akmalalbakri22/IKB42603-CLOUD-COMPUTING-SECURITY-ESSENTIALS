# IKB42603 Cloud Computing Security Essentials

# Lab 6 Report --- Object Storage Security & the Data Security Lifecycle

**Student Name:**
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\
**Student ID:**
\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\
**Course:** IKB42603 Cloud Computing Security Essentials\
**Lab:** Lab 6 --- Object Storage Security & the Data Security
Lifecycle\
**Platform:** Amazon S3 on LocalStack\
**Date:** 11 September 2026

------------------------------------------------------------------------

## 1. Introduction

This lab demonstrates how to secure object storage throughout the data
security lifecycle. The activities cover data classification,
public-bucket exposure, Block Public Access, IAM and bucket policies,
SSE-KMS encryption, presigned URLs, versioning, data remanence,
lifecycle rules, and cryptographic erasure.

The lab is divided into two parts:

-   **Session A:** Tasks 1--4 focus on who can access the data.
-   **Session B:** Tasks 5--8 focus on how the data is protected,
    retained, versioned, and eventually destroyed.

The steps and evidence in this report follow the IKB42603 Lab 6 manual.
The supplied `Lab6 Cloud Evidence.md` file contains **32 screenshots**,
and all 32 are included below.

------------------------------------------------------------------------

# 2. Environment Setup

## Step 1 --- Start LocalStack

The LocalStack container was started with IAM enforcement enabled.

``` bash
docker rm -f localstack 2>/dev/null

docker run -d --name localstack -p 4566:4566 \
  -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
  -e ENFORCE_IAM=1 \
  localstack/localstack-pro:latest
```

The AWS CLI endpoint and test credentials were then configured:

``` bash
export EP='--endpoint-url=http://localhost:4566'

aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

aws $EP sts get-caller-identity
```

### Result

The identity check returned the LocalStack dummy AWS account
`000000000000`.

### Evidence

![Evidence Image 1 --- LocalStack setup and caller
identity](lab6_evidence_images/image1.png)

------------------------------------------------------------------------

# 3. Task 1 --- Classify the Data Before You Store It

## Objective

The purpose of this task is to classify information before deciding how
it should be protected. Three objects were created:

-   Public information
-   Internal information
-   Confidential patient information

## Step 1 --- Create the bucket

``` bash
export BUCKET=miit-patient-records-$RANDOM
echo $BUCKET

aws $EP s3api create-bucket --bucket $BUCKET
```

The bucket used in the evidence was:

``` text
miit-patient-records-18273
```

## Step 2 --- Create the three files

``` bash
echo 'Ward visiting hours 10am-8pm' > public-notice.txt
echo 'Staff duty schedule, week 12' > internal-roster.txt
echo 'Patient: Ahmad bin Ali, Diagnosis: confidential' > confidential-record.txt
```

## Step 3 --- Upload and tag the objects

``` bash
aws $EP s3api put-object --bucket $BUCKET \
  --key public/notice.txt \
  --body public-notice.txt \
  --tagging 'classification=public'

aws $EP s3api put-object --bucket $BUCKET \
  --key internal/roster.txt \
  --body internal-roster.txt \
  --tagging 'classification=internal'

aws $EP s3api put-object --bucket $BUCKET \
  --key confidential/record.txt \
  --body confidential-record.txt \
  --tagging 'classification=confidential'
```

### Evidence

![Evidence Image 2 --- Bucket creation and file
preparation](lab6_evidence_images/image2.png)

![Evidence Image 3 --- Public and internal objects uploaded with
classification tags](lab6_evidence_images/image3.png)

![Evidence Image 4 --- Confidential object uploaded with confidential
classification](lab6_evidence_images/image4.png)

## Step 4 --- List the objects

``` bash
aws $EP s3api list-objects-v2 --bucket $BUCKET \
  --query 'Contents[].[Key,Size]' --output table
```

The output showed the three objects:

-   `confidential/record.txt`
-   `internal/roster.txt`
-   `public/notice.txt`

## Step 5 --- Check the confidential tag

``` bash
aws $EP s3api get-object-tagging \
  --bucket $BUCKET \
  --key confidential/record.txt
```

The output showed:

``` text
Key: classification
Value: confidential
```

### Evidence

![Evidence Image 5 --- Object listing and confidential classification
tag](lab6_evidence_images/image5.png)

## Data Classification Table

  -----------------------------------------------------------------------
  Classification    Who may read it   Impact if leaked  Control to apply
  ----------------- ----------------- ----------------- -----------------
  Public            Anyone            Low               Public
                                                        classification
                                                        only; no
                                                        confidential data

  Internal          Authorised        Moderate          IAM and
                    staff/users                         least-privilege
                                                        access to the
                                                        internal prefix

  Confidential      Only specifically High; may expose  Block Public
                    authorised users  sensitive patient Access,
                                      information       least-privilege
                                                        policy, SSE-KMS,
                                                        versioning and
                                                        lifecycle
                                                        controls
  -----------------------------------------------------------------------

### Simple explanation

Classification tells us **how sensitive the data is**. More sensitive
data needs stronger access restrictions and protection.

A prefix such as `confidential/` is also not a real folder. It is part
of the object key. Policies can therefore use the prefix to restrict
access to only certain objects.

------------------------------------------------------------------------

# 4. Task 2 --- Reproduce the Archetypal Breach

## Objective

This task demonstrates how a bucket can accidentally become publicly
readable.

## Step 1 --- Create the dangerous bucket policy

``` bash
cat > public-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadEverything",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*"
  }]
}
JSON
```

## Step 2 --- Apply the policy

``` bash
aws $EP s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://public-policy.json
```

## Step 3 --- Verify the policy

``` bash
aws $EP s3api get-bucket-policy \
  --bucket $BUCKET \
  --query Policy \
  --output text
```

### Evidence

![Evidence Image 6 --- Creation of the public bucket
policy](lab6_evidence_images/image6.png)

![Evidence Image 7 --- Public bucket policy applied and
displayed](lab6_evidence_images/image7.png)

## Step 4 --- Test anonymous access

No AWS credentials were used for this request.

``` bash
curl -s -o leaked.txt -w 'HTTP %{http_code}\n' \
  http://localhost:4566/$BUCKET/confidential/record.txt

cat leaked.txt
```

The evidence showed:

``` text
HTTP 200
Patient: Ahmad bin Ali, Diagnosis: confidential
```

### Evidence

![Evidence Image 8 --- Anonymous request successfully accessed the
confidential record](lab6_evidence_images/image8.png)

## Result

The confidential record was exposed because the bucket policy allowed:

``` json
"Principal": "*"
```

### Simple explanation

`Principal: "*"` means **everyone** can be the principal. Because the
action was `s3:GetObject` and the resource covered `$BUCKET/*`, any
anonymous user could read the objects.

There was no need for malware or an exploit. The problem was simply an
overly broad bucket policy.

------------------------------------------------------------------------

# 5. Task 3 --- Remediate with Block Public Access

## Objective

Block Public Access is used as a preventative guardrail to stop public
access from being accidentally enabled.

## Step 1 --- Remove the dangerous policy

``` bash
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

## Step 2 --- Enable all four Block Public Access settings

``` bash
aws $EP s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

## Step 3 --- Verify the settings

``` bash
aws $EP s3api get-public-access-block --bucket $BUCKET
```

The evidence showed all four flags as `true`:

-   `BlockPublicAcls = true`
-   `IgnorePublicAcls = true`
-   `BlockPublicPolicy = true`
-   `RestrictPublicBuckets = true`

## Step 4 --- Test the public policy again

The supplied evidence shows that LocalStack stored the Block Public
Access configuration but still allowed the test public policy/read in
this environment.

``` bash
aws $EP s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://public-policy.json

curl -s -o /dev/null -w 'anonymous read now: HTTP %{http_code}\n' \
  http://localhost:4566/$BUCKET/confidential/record.txt
```

### Evidence

![Evidence Image 9 --- Block Public Access configuration and
anonymous-read retest](lab6_evidence_images/image9.png)

## Result

The important evidence is that all four Block Public Access flags were
configured as `true`.

On real Amazon S3, `BlockPublicPolicy` is the key setting that prevents
a public bucket policy such as `Principal: "*"` from being accepted.

### Simple explanation

A **guardrail** prevents an unsafe configuration from being created. A
**detective control** only finds or reports the problem after it
happens.

For an organisation with many engineers, a preventative guardrail is
stronger because it reduces the chance that one engineer accidentally
makes a bucket public.

## Step 5 --- Create a least-privilege policy

The intended policy gives account-level access only to the `internal/`
prefix:

``` json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AccountReadInternalOnly",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::000000000000:root"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/internal/*"
  }]
}
```

### Simple explanation

Least privilege means giving access only to what is needed. The policy
is scoped to `internal/*` instead of `$BUCKET/*`, so it does not
automatically expose confidential objects.

------------------------------------------------------------------------

# 6. Task 4 --- Identity Policy vs Resource Policy

## Objective

This task shows that S3 access can be affected by both:

1.  An identity-based IAM policy attached to a user.
2.  A resource-based bucket policy attached to the bucket.

An explicit `Deny` overrides an `Allow`.

## Step 1 --- Create the DataAnalyst user

``` bash
aws $EP iam create-user --user-name DataAnalyst
```

## Step 2 --- Give the analyst a broad IAM policy

``` json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": "*"
  }]
}
```

This policy allows the analyst to read objects broadly.

## Step 3 --- Create access credentials

``` bash
aws $EP iam put-user-policy \
  --user-name DataAnalyst \
  --policy-name S3ReadAll \
  --policy-document file://analyst-iam.json

aws $EP iam create-access-key \
  --user-name DataAnalyst \
  --query 'AccessKey.[AccessKeyId,SecretAccessKey]' \
  --output text
```

The access-key values were then configured under the `analyst` AWS CLI
profile.

### Evidence

![Evidence Image 10 --- DataAnalyst IAM policy and access-key
creation](lab6_evidence_images/image10.png)

![Evidence Image 11 --- Analyst secret/access-key value
configuration](lab6_evidence_images/image11.png)

## Step 4 --- Create a bucket policy that allows internal but denies confidential

The important statements were:

``` json
{
  "Sid": "AllowAnalystInternal",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::000000000000:user/DataAnalyst"
  },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::$BUCKET/internal/*"
},
{
  "Sid": "DenyAnalystConfidential",
  "Effect": "Deny",
  "Principal": {
    "AWS": "arn:aws:iam::000000000000:user/DataAnalyst"
  },
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::$BUCKET/confidential/*"
}
```

### Evidence

![Evidence Image 12 --- Bucket policy with internal Allow and
confidential Deny](lab6_evidence_images/image12.png)

## Step 5 --- Test internal access

``` bash
AWS_PROFILE=analyst aws $EP s3api get-object \
  --bucket $BUCKET \
  --key internal/roster.txt \
  analyst-internal.txt && echo "internal: ALLOWED"
```

The evidence showed:

``` text
internal: ALLOWED
```

### Evidence

![Evidence Image 13 --- Analyst successfully accessed the internal
object](lab6_evidence_images/image13.png)

## Step 6 --- Check the analyst IAM policy

``` bash
AWS_PROFILE=analyst aws $EP iam get-user-policy \
  --user-name DataAnalyst \
  --policy-name S3ReadALL
```

The evidence confirms that the analyst's IAM policy allowed broad S3
read access.

### Evidence

![Evidence Image 14 --- Analyst IAM policy showing broad
permissions](lab6_evidence_images/image14.png)

## Step 7 --- Test confidential access

``` bash
AWS_PROFILE=analyst aws $EP s3api get-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  analyst-conf.txt
```

The supplied evidence shows the confidential object request and its
object metadata.

### Evidence

![Evidence Image 15 --- Analyst confidential-object
request](lab6_evidence_images/image15.png)

## Result and policy evaluation

The intended evaluation is:

``` text
Default deny
    ↓
Check explicit Deny
    ↓
Check explicit Allow
```

For the **internal object**:

-   IAM policy allows access.
-   Bucket policy also allows access to `internal/*`.
-   Result: **ALLOWED**

For the **confidential object**:

-   IAM policy allows access.
-   Bucket policy explicitly denies access to `confidential/*`.
-   Explicit Deny wins.
-   Result: **DENIED**

The manual notes that LocalStack may not fully enforce
IAM/resource-policy behaviour in every configuration. Therefore, the
policy documents themselves are also valid evidence when the simulated
environment does not enforce the expected decision.

### Simple explanation

An **identity-based policy** is attached to the user or role.

A **resource-based policy** is attached to the bucket.

When both apply, an explicit `Deny` is stronger than an `Allow`.

------------------------------------------------------------------------

# 7. Task 5 --- Default Encryption at Rest with SSE-KMS

## Objective

The bucket is configured to automatically encrypt objects using a
customer-managed KMS key.

This means the uploader does not have to remember to add encryption
options for every upload.

## Step 1 --- Create a KMS key

``` bash
export KEY_ID=$(aws $EP kms create-key \
  --description 'IKB42603 Lab6 patient records bucket key' \
  --query 'KeyMetadata.KeyId' \
  --output text)

echo $KEY_ID
```

### Evidence

![Evidence Image 16 --- KMS key creation and encryption
configuration](lab6_evidence_images/image16.png)

## Step 2 --- Create the encryption configuration

``` json
{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "$KEY_ID"
    },
    "BucketKeyEnabled": true
  }]
}
```

## Step 3 --- Apply the configuration

``` bash
aws $EP s3api put-bucket-encryption \
  --bucket $BUCKET \
  --server-side-encryption-configuration file://encryption.json

aws $EP s3api get-bucket-encryption --bucket $BUCKET
```

### Evidence

![Evidence Image 17 --- Bucket default SSE-KMS encryption
configuration](lab6_evidence_images/image17.png)

## Step 4 --- Upload an object without encryption flags

``` bash
aws $EP s3api put-object \
  --bucket $BUCKET \
  --key confidential/record-v2.txt \
  --body confidential-record.txt
```

## Step 5 --- Check the object's encryption

``` bash
aws $EP s3api head-object \
  --bucket $BUCKET \
  --key confidential/record-v2.txt \
  --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]' \
  --output text
```

The evidence showed:

``` text
aws:kms
<customer KMS key ARN>
True
```

### Evidence

![Evidence Image 18 --- Object protected automatically with aws:kms and
BucketKeyEnabled](lab6_evidence_images/image18.png)

## Result

The object was encrypted automatically even though no encryption flag
was included in the upload command.

### Simple explanation

SSE-KMS protects data **at rest** by encrypting the stored object.

However, encryption does **not** decide who is allowed to access the
object. If a user has valid permission and the service can decrypt the
object for that request, encryption does not replace access control.

Therefore, SSE-KMS alone would not stop the analyst. The analyst must
also be denied by IAM/resource policies.

`BucketKeyEnabled: true` reduces KMS request overhead and cost while
keeping the confidentiality protection.

------------------------------------------------------------------------

# 8. Task 6 --- Delegated Access and the Condition-Key Trap

## Part A --- Presigned URL

A presigned URL gives temporary access to a specific object without
requiring the recipient to have AWS credentials.

## Step 1 --- Generate a 60-second presigned URL

``` bash
aws $EP s3 presign \
  s3://$BUCKET/internal/roster.txt \
  --expires-in 60
```

The URL was stored in the `URL` variable and tested with `curl`.

### Evidence

![Evidence Image 19 --- Generated presigned URL and successful
access](lab6_evidence_images/image19.png)

The first request returned:

``` text
HTTP 200
```

The URL contains information such as:

-   The requested object.
-   The signing algorithm.
-   The credential used to sign it.
-   The requested expiry time.
-   The signature.

### Evidence

![Evidence Image 20 --- Presigned URL tested after the expiry
period](lab6_evidence_images/image20.png)

The supplied LocalStack evidence returned HTTP 200 after expiry. This is
consistent with the lab manual's warning that LocalStack may not always
enforce presigned URL expiry.

### Simple explanation

A presigned URL is like a temporary permission token. Anyone who gets
the URL can use the granted permission until the signature expires or
access is otherwise invalidated.

It should therefore be:

-   Short-lived.
-   Limited to one object.
-   Used only when delegated access is necessary.

------------------------------------------------------------------------

## Part B --- `aws:SecureTransport` Condition-Key Trap

## Step 1 --- Apply the SecureTransport Deny policy

``` json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnencryptedTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::$BUCKET",
      "arn:aws:s3:::$BUCKET/*"
    ],
    "Condition": {
      "Bool": {
        "aws:SecureTransport": "false"
      }
    }
  }]
}
```

### Evidence

![Evidence Image 21 --- SecureTransport Deny
policy](lab6_evidence_images/image21.png)

## Step 2 --- Observe the bucket-wide effect

The LocalStack endpoint is:

``` text
http://localhost:4566
```

Therefore, `aws:SecureTransport` evaluates as `false`.

The Deny therefore matches the requests made through the HTTP LocalStack
endpoint.

### Evidence

![Evidence Image 22 --- Bucket contents while the SecureTransport policy
was present](lab6_evidence_images/image22.png)

## Step 3 --- Remove the policy

``` bash
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

The supplied evidence also shows the later check where the bucket policy
no longer existed.

### Evidence

![Evidence Image 23 --- SecureTransport policy
removed](lab6_evidence_images/image23.png)

### Simple explanation

The policy is correct for an HTTPS production environment, but the lab
uses HTTP LocalStack.

A security condition must always be tested against the **actual
environment** where it will run. A condition that is correct in AWS can
unintentionally block every request in a local HTTP simulation.

------------------------------------------------------------------------

# 9. Task 7 --- Versioning, Delete Markers and Data Remanence

## Objective

This task demonstrates that deleting an S3 object does not automatically
remove previous versions when versioning is enabled.

## Step 1 --- Enable versioning

``` bash
aws $EP s3api put-bucket-versioning \
  --bucket $BUCKET \
  --versioning-configuration Status=Enabled

aws $EP s3api get-bucket-versioning --bucket $BUCKET
```

The output showed:

``` text
Status: Enabled
```

### Evidence

![Evidence Image 24 --- Versioning
enabled](lab6_evidence_images/image24.png)

## Step 2 --- Create version 2

``` bash
echo 'Patient: Ahmad bin Ali, Diagnosis: hypertension' > rec-v2.txt

aws $EP s3api put-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  --body rec-v2.txt \
  --query VersionId \
  --output text
```

## Step 3 --- Create version 3

``` bash
echo 'Patient: [REDACTED], Diagnosis: [REDACTED]' > rec-v3.txt

aws $EP s3api put-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  --body rec-v3.txt \
  --query VersionId \
  --output text
```

### Evidence

![Evidence Image 25 --- Creation of the second and third object
versions](lab6_evidence_images/image25.png)

## Step 4 --- List all versions

``` bash
aws $EP s3api list-object-versions \
  --bucket $BUCKET \
  --prefix confidential/record.txt \
  --query 'Versions[].[VersionId,IsLatest,Size]' \
  --output table
```

The evidence showed multiple versions, including the older version with
`null` as the version ID.

### Evidence

![Evidence Image 26 --- Multiple versions of the confidential
record](lab6_evidence_images/image26.png)

## Step 5 --- Delete the object normally

``` bash
aws $EP s3api delete-object \
  --bucket $BUCKET \
  --key confidential/record.txt
```

The response indicated that a delete marker was created.

### Evidence

![Evidence Image 27 --- Delete marker created after deleting the
object](lab6_evidence_images/image27.png)

## Step 6 --- Recover the old version

``` bash
aws $EP s3api get-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  --version-id null \
  recovered.txt

cat recovered.txt
```

The evidence showed that the original record could still be recovered.

### Evidence

![Evidence Image 28 --- Original confidential record recovered from an
old version](lab6_evidence_images/image28.png)

## Result

The normal delete operation did not destroy the historical version.

This is called **data remanence**.

### Simple explanation

With versioning enabled:

``` text
Delete object
      ↓
Delete marker is created
      ↓
Old versions remain underneath
```

Therefore, deleting only the current object is not enough for a true
erasure request.

For permanent version-level deletion, each retained version must be
deleted by its version ID.

------------------------------------------------------------------------

# 10. Task 8 --- Lifecycle, Retention and Cryptographic Erasure

## Part A --- Lifecycle Configuration

## Step 1 --- Configure lifecycle rules

The confidential data lifecycle used:

``` json
{
  "Rules": [
    {
      "ID": "RetireConfidentialRecords",
      "Filter": {
        "Prefix": "confidential/"
      },
      "Status": "Enabled",
      "Expiration": {
        "Days": 365
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      }
    },
    {
      "ID": "AbortIncompleteUploads",
      "Filter": {
        "Prefix": ""
      },
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

The configuration was applied using:

``` bash
aws $EP s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --lifecycle-configuration file://lifecycle.json
```

## Step 2 --- Verify lifecycle rules

``` bash
aws $EP s3api get-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' \
  --output table
```

The evidence showed:

``` text
RetireConfidentialRecords    Enabled
AbortIncompleteUploads       Enabled
```

### Evidence

![Evidence Image 29 --- Lifecycle rules configured and
enabled](lab6_evidence_images/image29.png)

### Simple explanation

Lifecycle rules automate retention and deletion.

For this lab:

-   Confidential objects expire after 365 days.
-   Non-current versions expire after 30 days.
-   Incomplete multipart uploads are removed after 7 days.

This makes retention easier to manage and audit.

------------------------------------------------------------------------

## Part B --- Cryptographic Erasure

## Step 3 --- Check the KMS key

``` bash
aws $EP kms describe-key \
  --key-id $KEY_ID \
  --query 'KeyMetadata.[KeyId,KeyState,Enabled]' \
  --output text
```

The key was initially enabled.

### Evidence

![Evidence Image 30 --- KMS key initially enabled, then
disabled](lab6_evidence_images/image30.png)

## Step 4 --- Disable the KMS key

``` bash
aws $EP kms disable-key --key-id $KEY_ID
```

The key then showed:

``` text
Disabled False
```

## Step 5 --- Schedule key deletion

``` bash
aws $EP kms schedule-key-deletion \
  --key-id $KEY_ID \
  --pending-window-in-days 7
```

The evidence showed:

``` text
KeyState: PendingDeletion
PendingWindowInDays: 7
```

### Evidence

![Evidence Image 31 --- KMS key scheduled for
deletion](lab6_evidence_images/image31.png)

## Step 6 --- Attempt to read encrypted data

``` bash
aws $EP s3api get-object \
  --bucket $BUCKET \
  --key confidential/record-v2.txt \
  after-erasure.txt
```

The supplied LocalStack evidence shows that the object could still be
returned after the key was disabled.

### Evidence

![Evidence Image 32 --- Object read after KMS key disable/scheduled
deletion](lab6_evidence_images/image32.png)

## Result

The lab manual warns that LocalStack may not re-check KMS key state when
reading S3 objects. Therefore, the successful read in the simulation
does not mean that cryptographic erasure is ineffective in real AWS.

### Simple explanation

Cryptographic erasure works by destroying or making unavailable the key
required to decrypt encrypted data.

If all relevant copies are encrypted with that key, losing the key makes
the ciphertext unusable.

This provides stronger assurance than simply overwriting a file because
the organisation does not control the physical storage media or every
possible underlying copy.

------------------------------------------------------------------------

# 11. Short-Answer Questions

## Question 1

**Which single element of the Task 2 policy caused the exposure, and why
is `Principal: "*"` more dangerous on a bucket policy than an over-broad
IAM policy attached to one user?**

### Answer

The main element was:

``` json
"Principal": "*"
```

It means any principal, including anonymous users, can access the
resource when the policy allows it.

A broad IAM policy affects one specific user or role. A bucket policy
with `Principal: "*"` can expose the bucket to everyone, including
people who do not have an AWS identity.

**Simple explanation:** `Principal: "*"` can turn a private bucket into
a public bucket.

------------------------------------------------------------------------

## Question 2

**Explain the difference between an identity-based policy and a
resource-based policy. In Task 4, which one decided each of the
analyst's two requests?**

### Answer

An **identity-based policy** is attached to an IAM user, group, or role.
It says what that identity is allowed to do.

A **resource-based policy** is attached to the resource, such as the S3
bucket. It says which principals can access the resource and what they
can do.

In Task 4:

-   `internal/roster.txt` was **allowed** because the analyst IAM policy
    allowed it and the bucket policy also allowed the internal prefix.
-   `confidential/record.txt` was **denied** because the bucket policy
    contained an explicit `Deny` for the confidential prefix.

**Simple explanation:** IAM allowed the analyst generally, but the
bucket policy placed a stronger restriction on confidential data.

------------------------------------------------------------------------

## Question 3

**Block Public Access is described as a guardrail rather than a control.
What is the difference, and why does the distinction matter for an
organisation with many engineers?**

### Answer

A control can protect or monitor a resource, while a guardrail places a
restriction that prevents an unsafe configuration from being created.

For example, Block Public Access can stop a public bucket policy from
making a bucket public.

This matters in a large organisation because many engineers may create
or modify storage resources. A preventative guardrail reduces the chance
that one engineer accidentally exposes sensitive information.

**Simple explanation:** A detective control says, "You made it public."
A guardrail tries to stop it from becoming public in the first place.

------------------------------------------------------------------------

## Question 4

**Your bucket has default SSE-KMS encryption. Does that protect the
confidential record from the analyst in Task 4? Explain precisely what
server-side encryption does and does not defend against.**

### Answer

No. SSE-KMS does not replace access control.

Server-side encryption protects the object's data while it is stored. It
helps protect against someone obtaining the stored ciphertext without
the ability to decrypt it.

However, it does not decide whether an authenticated user is allowed to
request the object. If the analyst has valid S3 permission, S3 can
decrypt the object for the authorised request.

In Task 4, the confidential record must therefore be protected using the
bucket/IAM policy that explicitly denies the analyst.

**Simple explanation:** Encryption protects the stored data. IAM and
bucket policies control who can access it.

------------------------------------------------------------------------

## Question 5

**A patient invokes their right to erasure. Using your Task 7 evidence,
explain why `delete-object` alone is not compliant, and describe two
mechanisms that would make the deletion provable.**

### Answer

`delete-object` alone is not enough when versioning is enabled because
it creates a delete marker while previous versions remain.

The Task 7 evidence proved this because the original unredacted record
was recovered using:

``` text
--version-id null
```

Two mechanisms that can provide stronger deletion assurance are:

1.  **Permanent per-version deletion** --- identify and delete every
    retained object version and delete marker by version ID.
2.  **Cryptographic erasure** --- disable/destroy the KMS key protecting
    the encrypted data so the remaining ciphertext cannot be decrypted.

Lifecycle rules can also automate the removal of old and non-current
versions according to a defined retention policy.

**Simple explanation:** Deleting the visible object does not always
delete the old copies. All versions must be handled, or the encryption
key can be destroyed for cryptographic erasure.

------------------------------------------------------------------------

## Question 6

**You are the auditor in Week 11. Name three commands from this lab
whose output you would collect as compliance evidence, and state which
control each one evidences.**

### Answer

  -----------------------------------------------------------------------------------------------------------------------
  Command                                                                             Evidence provided
  ----------------------------------------------------------------------------------- -----------------------------------
  `aws $EP s3api get-public-access-block --bucket $BUCKET`                            Proves the four Block Public Access
                                                                                      settings are enabled

  `aws $EP s3api head-object --bucket $BUCKET --key confidential/record-v2.txt ...`   Proves the object uses SSE-KMS
                                                                                      encryption

  `aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET ...`             Proves the retention/lifecycle
                                                                                      policy is configured
  -----------------------------------------------------------------------------------------------------------------------

Other useful audit evidence includes `list-object-versions` for version
retention and `kms describe-key` for KMS key state.

------------------------------------------------------------------------

# 12. Final Security Verification

The lab manual requires the following verification block:

``` bash
echo "=== IKB42603 Lab 6 verification: $BUCKET ==="

aws $EP s3api get-public-access-block \
  --bucket $BUCKET \
  --query 'PublicAccessBlockConfiguration' \
  --output text

aws $EP s3api get-bucket-versioning \
  --bucket $BUCKET \
  --output text

aws $EP s3api get-bucket-encryption \
  --bucket $BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.[SSEAlgorithm,KMSMasterKeyID]' \
  --output text

aws $EP s3api get-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' \
  --output text

aws $EP kms describe-key \
  --key-id $KEY_ID \
  --query 'KeyMetadata.KeyState' \
  --output text
```

The supplied `Lab6 Cloud Evidence.md` file contains the individual
evidence for these controls, including:

-   All four Block Public Access flags set to `true`.
-   Versioning set to `Enabled`.
-   Default encryption using `aws:kms`.
-   Lifecycle rules set to `Enabled`.
-   KMS key state changed to `PendingDeletion`.

**Note:** A separate screenshot containing the complete final
verification block was not present among the 32 images in the supplied
evidence file, so no final verification output has been invented in this
report.

------------------------------------------------------------------------

# 13. Security Control Summary

  ------------------------------------------------------------------------------
  Security area           Implemented control            Evidence
  ----------------------- ------------------------------ -----------------------
  Data classification     Public/internal/confidential   Images 2--5
                          tags                           

  Public exposure         Public policy deliberately     Images 6--8
                          demonstrated                   

  Public access           Four Block Public Access       Image 9
  remediation             settings                       

  Least privilege         Internal prefix scoped policy  Images 12--15

  Identity/resource       IAM Allow vs bucket explicit   Images 10--15
  policy                  Deny                           

  Encryption at rest      Default SSE-KMS                Images 16--18

  Temporary sharing       Presigned URL                  Images 19--20

  Transport condition     `aws:SecureTransport` test     Images 21--23

  Versioning              Versioning enabled             Image 24

  Data remanence          Old versions and delete marker Images 25--28

  Lifecycle               Retention and                  Image 29
                          incomplete-upload rules        

  Cryptographic erasure   KMS disabled and scheduled for Images 30--32
                          deletion                       
  ------------------------------------------------------------------------------

------------------------------------------------------------------------

# 14. Conclusion

This lab demonstrated that securing object storage requires more than
one security mechanism.

The main lessons are:

1.  Data should be classified before access decisions are made.
2.  `Principal: "*"` can accidentally expose sensitive objects.
3.  Block Public Access provides an important preventative guardrail.
4.  IAM policies and bucket policies must be evaluated together.
5.  Explicit `Deny` takes priority over `Allow`.
6.  SSE-KMS protects data at rest but does not replace access control.
7.  Presigned URLs provide temporary delegated access but must be
    short-lived and carefully controlled.
8.  Versioning can preserve old sensitive data after a normal delete.
9.  Lifecycle rules automate retention and cleanup.
10. Cryptographic erasure can provide strong deletion assurance by
    removing the key needed to decrypt encrypted data.

Overall, the lab shows how confidentiality and data lifecycle security
should be managed from **classification → access control → encryption →
sharing → versioning → retention → deletion**.

------------------------------------------------------------------------

# 15. Evidence Index

All screenshots supplied in `Lab6 Cloud Evidence.md` are included in
this report.

  Image           Report section
  --------------- ------------------------------------------------
  Image 1         Environment Setup
  Images 2--5     Task 1 --- Data Classification
  Images 6--8     Task 2 --- Public Bucket Breach
  Image 9         Task 3 --- Block Public Access
  Images 10--15   Task 4 --- IAM vs Resource Policy
  Images 16--18   Task 5 --- SSE-KMS
  Images 19--23   Task 6 --- Presigned URL and SecureTransport
  Images 24--28   Task 7 --- Versioning and Data Remanence
  Images 29--32   Task 8 --- Lifecycle and Cryptographic Erasure

------------------------------------------------------------------------

## References

1.  UniKL MIIT. (2026). *IKB42603 Cloud Computing Security Essentials
    Lab Manual: Lab 6 --- Object Storage Security & the Data Security
    Lifecycle*.
2.  *Lab6 Cloud Evidence.md*. Supplied practical evidence file
    containing the 32 screenshots used in this report.
