# Infra — Ultron Embeddings (CDK)

This folder defines the AWS infrastructure for the project using **AWS CDK** (Cloud Development Kit, TypeScript).
It provisions storage, compute, and routing for the pipeline.

---

## What gets created

- **S3 bucket** for raw, chunks, embeddings, and index artifacts.
- **DynamoDB table** for ingest checkpoints and session state.
- **Lambda functions** (Rust + Python, ARM64):
  - `ingest` (Rust) — pulls Marvel API data to S3 (raw).
  - `derive` (Rust) — normalizes raw → canonical entities.
  - `chunker` (Rust) — sentence‑aware chunking.
  - `embedder` (Rust) — Candle embeddings, writes vectors.
  - `retrieve` (Rust) — cosine search over vectors.
  - `orchestrator` (Python) — LangGraph control plane (query analysis, routing).
- **API Gateway (REST)** exposing:
  - `GET /retrieve` → `retrieve` Lambda
  - `POST /chat` → `orchestrator` Lambda
- **EventBridge rule** to schedule `ingest` (and optionally `derive`/`embedder`) nightly.
- **Lambda Layer(s)** (optional) for `llama.cpp` or shared libs.
- **IAM roles/policies** scoped to least privilege.
- **(Optional) VPC** endpoints if you want private S3/DynamoDB access.

> We keep **model weights in S3** and copy them into **/tmp** at cold start to avoid containers.

---

## Pre‑flight (who/where am I)

```bash
aws sts get-caller-identity   # confirm account + IAM (Identity and Access Management)
aws configure get region      # confirm AWS region
```

Stop if the account/role/region is wrong.

---

## Secrets & config

See `docs/settings.md` for options:
- **Local `.env`** kept outside the repo (zsh-friendly `source`).
- **AWS Secrets Manager** for production keys.
- **VS Code Secrets** for editor-only experiments.

At minimum, the stacks expect:
- `MARVEL_PUBLIC_KEY`, `MARVEL_PRIVATE_KEY`
- `BUCKET_NAME` (optional: defaults to `ultron-embeddings-<account>`)
- `CHECKPOINT_TABLE` (optional: auto‑named if absent)

---

## Project layout (infra)

```
infra/
├─ bin/
│  └─ app.ts                 # CDK app entry
├─ lib/
│  ├─ storage-stack.ts       # S3, DynamoDB
│  ├─ compute-stack.ts       # Lambdas, Layers, permissions
│  └─ api-stack.ts           # API Gateway, routes, usage plan
├─ cdk.json
└─ package.json
```

Each stack is standalone but wired together in `bin/app.ts`.

---

## Storage stack sketch (TypeScript)

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export class StorageStack extends cdk.Stack {
  public readonly bucket: s3.Bucket;
  public readonly table: dynamodb.Table;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.bucket = new s3.Bucket(this, 'UltronBucket', {
      bucketName: `ultron-embeddings-${this.account}`,
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      enforceSSL: true,
    });

    this.table = new dynamodb.Table(this, 'CheckpointTable', {
      partitionKey: { name: 'pk', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'sk', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      pointInTimeRecovery: true
    });
  }
}
```

---

## Compute stack sketch (Rust & Python Lambdas)

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

interface ComputeProps extends cdk.StackProps {
  bucket: s3.Bucket;
  table: dynamodb.Table;
}

export class ComputeStack extends cdk.Stack {
  public readonly retrieveFn: lambda.Function;
  public readonly orchestratorFn: lambda.Function;

  constructor(scope: Construct, id: string, props: ComputeProps) {
    super(scope, id, props);

    const env = {
      OUTPUT_BUCKET: props.bucket.bucketName,
      CHECKPOINT_TABLE: props.table.tableName,
      MODEL_DIR: '/tmp/model'
    };

    // Example: Rust Lambda (retrieve)
    this.retrieveFn = new lambda.Function(this, 'RetrieveFn', {
      runtime: lambda.Runtime.PROVIDED_AL2,  // Rust custom runtime
      architecture: lambda.Architecture.ARM_64,
      handler: 'bootstrap',
      code: lambda.Code.fromAsset('../target/lambda/lambda_retrieve'),
      memorySize: 1024,                       // controls CPU on Lambda
      timeout: cdk.Duration.seconds(15),
      ephemeralStorageSize: cdk.Size.mebibytes(2048),
      environment: env
    });

    // Example: Python Lambda (orchestrator)
    this.orchestratorFn = new lambda.Function(this, 'OrchestratorFn', {
      runtime: lambda.Runtime.PYTHON_3_12,
      architecture: lambda.Architecture.ARM_64,
      handler: 'app.handler',
      code: lambda.Code.fromAsset('../orchestrator/dist'), // zipped app
      memorySize: 1024,
      timeout: cdk.Duration.seconds(30),
      environment: env
    });

    // Least‑privilege access
    props.bucket.grantReadWrite(this.retrieveFn);
    props.bucket.grantReadWrite(this.orchestratorFn);
    props.table.grantReadWriteData(this.retrieveFn);
    props.table.grantReadWriteData(this.orchestratorFn);

    // (Optional) allow Secrets Manager read for prod keys
    this.orchestratorFn.addToRolePolicy(new iam.PolicyStatement({
      actions: ['secretsmanager:GetSecretValue'],
      resources: ['*']
    }));
  }
}
```

---

## API stack sketch (routes)

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as apigw from 'aws-cdk-lib/aws-apigateway';
import * as lambda from 'aws-cdk-lib/aws-lambda';

interface ApiProps extends cdk.StackProps {
  retrieveFn: lambda.Function;
  orchestratorFn: lambda.Function;
}

export class ApiStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: ApiProps) {
    super(scope, id, props);

    const api = new apigw.RestApi(this, 'UltronApi', {
      restApiName: 'Ultron Embeddings API',
      deployOptions: { stageName: 'prod' }
    });

    const retrieve = api.root.addResource('retrieve');
    retrieve.addMethod('GET', new apigw.LambdaIntegration(props.retrieveFn));

    const chat = api.root.addResource('chat');
    chat.addMethod('POST', new apigw.LambdaIntegration(props.orchestratorFn));

    // (Optional) API key + usage plan
    const plan = api.addUsagePlan('Plan', { throttle: { rateLimit: 5, burstLimit: 10 } });
    const key = api.addApiKey('Key');
    plan.addApiKey(key);
    plan.addApiStage({ stage: api.deploymentStage });
  }
}
```

---

## Build & deploy

```bash
# Build Rust Lambdas (release, ARM64)
cargo lambda build --release --arm64 -p lambda_retrieve
# Repeat for other Rust crates as needed (ingest, derive, chunker, embedder)

# Install deps and deploy CDK
cd infra
npm ci
cdk bootstrap                 # once per account/region
cdk deploy --all
```

Outputs will include the **REST API URL** and resource ARNs.

---

## Scheduling ingest

Enable a nightly crawl via EventBridge:

```ts
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

new events.Rule(this, 'NightlyIngest', {
  schedule: events.Schedule.cron({ minute: '0', hour: '6' }), // 06:00 UTC
  targets: [new targets.LambdaFunction(ingestFn)]
});
```

---

## Cost & scale notes

- Keep **Lambda memory** just high enough to meet p95; it also scales CPU.
- Favor **Graviton (ARM64)** for cost/perf.
- Shard vectors into **25–200 MB** files to balance cold load and search cost.
- Use **reserved concurrency** on public endpoints to cap spend.
- Prefer **S3 + DynamoDB** over always‑on databases for this workload.

---

## Next steps

- See `docs/state-and-storage.md` for S3/DynamoDB layouts.
- See `docs/retrieve.md` for the search endpoint shape.
- See `docs/orchestrator.md` for control‑plane details.
- See `docs/ingest.md` to wire Marvel API credentials and run the first crawl.
