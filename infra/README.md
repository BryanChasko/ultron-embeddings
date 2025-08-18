# infra/README.md

Here’s the file:

---

## 🏗️ Infra Guide — Ultron Embeddings

Infrastructure for storing **Ultron embeddings** in AWS S3 using AWS CDK.

---

### Goals

* Verify AWS account and IAM identity before deploying.
* Create (or re-use) an S3 bucket for JSONL embeddings.
* Upload JSONL test files from Rust to that bucket.
* Always include Marvel attribution with Marvel data.

---

### 1. Pre-Flight Checks

```bash
aws sts get-caller-identity     # Confirm account + IAM
aws configure get region        # Confirm region
```

Stop if account/role is wrong.

---

### 2. Initialize CDK Project

```bash
cd infra
cdk init app --language typescript
```

---

### 3. Add S3 Bucket

Edit `infra/lib/infra-stack.ts`:

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class InfraStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new s3.Bucket(this, 'UltronEmbeddingsBucket', {
      bucketName: `ultron-embeddings-${this.account}`,
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });
  }
}
```

---

## 4. Deploy Infra

```bash
cdk bootstrap   # only once per account/region
cdk deploy
```

---

### 5. Upload Embeddings

```bash
aws s3 cp ./embeddings/test.jsonl s3://ultron-embeddings-<account-id>/ --region <region>
```

---

✅ With this setup:

* **Rust** can save embeddings locally, then push to S3.
* **S3 bucket** is created/verified via CDK.
* **IAM check** ensures we don’t deploy to the wrong account.
