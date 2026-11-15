+++
title = "Local AWS Development Without the Cloud Bill: A Practical Guide to AWS Emulator"
date = 2026-11-15
[taxonomies]
tags = ["aws", "devops", "terraform", "rust", "testing"]

[extra]
mermaid = true
+++

# Local AWS Development Without the Cloud Bill: A Practical Guide to AWS Emulator

Every cloud engineer knows the pain of the "deploy-and-pray" feedback loop. You write application code that interacts with Amazon S3, Amazon DynamoDB, or AWS Cloud Map, and to verify that your API calls, parameter constraints, and cross-service workflows actually behave the way AWS expects, you have historically faced an uncomfortable choice:

1. **Test against real AWS accounts**: dealing with IAM credentials in CI pipelines, per-developer sandbox provisioning, slower network round-trips, and surprise cloud bills from orphaned resources.
2. **Write hand-rolled unit mocks or use third-party emulators**: risking "mock drift" where your tests pass locally because a stub accepted an invalid parameter that real AWS rejects in production.

**AWS Emulator** changes this equation. Released by AWS as an open-source local API emulator under the **CC0 1.0 Universal** public domain dedication, AWS Emulator lets you run a high-fidelity, stateful AWS endpoint right on your laptop, development server, or CI runner, with **no AWS account required** and **no changes to your application code**.

Even more uniquely, AWS Emulator is **continuously updated multiple times a day by AI agents**, verified against real AWS behavior (`us-west-2` and `us-east-1`), and ships with a public fidelity evaluation suite so you always know exactly which APIs match production AWS.

In this deep-dive guide, we will explore:
- How AWS Emulator works under the hood (ephemeral credentials, TLS, SigV4, and in-memory state).
- All four ways to install and run it (AI Coding Agent Skill, Linux Native, Docker, and Rust Embedded Mode).
- How to point the AWS CLI and language SDKs at the emulator with zero code changes.
- Complete, runnable examples (AWS CLI, Python Boto3 + `pytest`, Rust Embedded parallel tests, Terraform, and GitHub Actions CI/CD).
- How AWS Emulator's **Fidelity Scores** work.
- **How AWS Emulator compares to existing tools** like LocalStack, Moto, DynamoDB Local, MinIO, and AWS SAM CLI.
- Key limitations, security caveats, and production best practices.

## 1. What Is AWS Emulator and How Does It Work?

**AWS Emulator** is a local command-line server (and embeddable Rust library) that emulates AWS APIs on your machine. When started as a local process, it listens on a single HTTPS endpoint (by default `https://127.0.0.1:9297`) and serves both AWS API operations and an internal health check route (`GET /__admin/health`).

{% mermaid() %}
flowchart LR
    subgraph DevMachine["Developer Machine / CI Runner (No AWS Account Needed)"]
        direction LR
        CLI["AWS CLI / SDKs\n(Boto3, JS v3, Go, Java, .NET)"]
        EnvFiles["~/.aws/emulator/9297/\n• emulator_credentials\n• config\n• ca.pem / truststore.p12"]
        
        subgraph EmulatorProcess["aws-emulator Process (127.0.0.1:9297)"]
            Router["HTTPS + SigV4 Router\n& /__admin/health"]
            State["In-Memory State Store\n(Account: 000000000000)"]
            S3["Amazon S3"]
            DDB["Amazon DynamoDB"]
            CM["AWS Cloud Map / STS / IAM / ..."]
            
            Router --> State
            State --> S3
            State --> DDB
            State --> CM
        end

        CLI -- "Reads TLS cert & keys" --> EnvFiles
        CLI -- "HTTPS + SigV4\nhttps://127.0.0.1:9297" --> Router
    end

    subgraph RustProcess["Rust Test Process (Embedded Mode)"]
        RustTest["#[tokio::test]\nAWS SDK for Rust"] -- "In-Memory HTTP Client\n(Zero Network Sockets)" --> Embedded["AwsEmulatorEmbedded\n(Isolated Per-Test State)"]
    end
{% end %}

### Key Architectural Highlights

1. **Single Endpoint for All Services**: Every emulated AWS service is served over `https://127.0.0.1:9297` (or a custom port via `-p <port>`).
2. **Per-Run Ephemeral Credentials & TLS Certificates**:
   Each time `aws-emulator` starts on port `9297`, it creates a private directory at `~/.aws/emulator/9297/` with `0600` permissions containing:
   - `emulator_credentials`: Ephemeral per-run SigV4 credentials (a 20-character access key ID starting with `EMULATOR` + 12 uppercase hex chars, and a 40-character hex secret key).
   - `config`: An AWS CLI/SDK profile setting `region = us-east-1`, `endpoint_url`, `ca_bundle`, and S3 `signature_version = s3v4`.
   - `ca.pem`: A freshly generated TLS certificate for HTTPS.
   - `truststore.p12`: A passwordless PKCS12 trust store containing the same certificate for Java JVMs.
3. **Real SigV4 Request Signing by Default**:
   Unlike naive local servers that accept any unauthenticated HTTP traffic (which exposes developers to malicious browser cross-origin or local process requests), AWS Emulator enforces AWS Signature Version 4 (SigV4) by default and validates clock skew within 15 minutes. It rejects requests signed with your real AWS credentials so you immediately catch misconfigurations.
4. **In-Memory Ephemeral State**:
   The emulator uses roughly **160 MiB of RAM when idle** and stores all created resources in memory under emulated account ID `000000000000`. When you stop the emulator (`Ctrl+C`, `SIGTERM`, or `--idle-timeout`), it cleanly deletes `~/.aws/emulator/<port>/` and wipes all state, guaranteeing a clean slate on the next run.

## 2. Four Ways to Get Started

AWS Emulator offers four installation and execution paths depending on your OS and workflow:

### Path A: With an AI Coding Agent (`aws-emulator` Skill)
If you use an AI coding agent (such as GitHub Copilot, Kiro, Cursor, or Claude Code) and have **AWS CLI v2.35.0 or later**, you can install the official `aws-emulator` agent skill:

```bash
# 1. Verify AWS CLI version (>= 2.35.0)
aws --version

# 2. Install the aws-emulator skill for your coding agents
aws agent-toolkit add-skill --skill-name aws-emulator --region us-east-1
```

Once installed, you can prompt your agent:
- *"Install AWS Emulator and point my project at it."*
- *"Will my app in this repo work on AWS Emulator?"* (The agent inspects the AWS API operations your codebase uses and cross-references them against the official fidelity manifest.)

### Path B: Native Linux Package (RPM & Debian)
On 64-bit Linux (`x86_64` or `aarch64` with `glibc >= 2.17`), AWS Emulator runs natively and installs into `/opt/aws-emulator/` with `/usr/bin/aws-emulator` on your `PATH`.

```bash
# RPM-based systems (Amazon Linux 2023, Fedora, RHEL - x86_64)
sudo dnf install -y https://downloads.aws.com/aws-emulator/packages/aws-emulator/LTS/latest/aws-emulator-x86_64.rpm

# Debian/Ubuntu systems (amd64)
curl -fsSLo aws-emulator.deb https://downloads.aws.com/aws-emulator/packages/aws-emulator/LTS/latest/aws-emulator_amd64.deb
sudo apt-get install -y ./aws-emulator.deb

# Verify installation
aws-emulator -v
```

> **Tip: Cap Memory Usage on Linux:** Because resources live in memory, you can run the emulator inside a `systemd` cgroup v2 scope with a hard 2 GiB cap and zero swap:
> ```bash
> systemd-run --user --scope -p MemoryMax=2G -p MemorySwapMax=0 aws-emulator
> ```

### Path C: Docker Container (macOS, Windows, and Linux)
On macOS and Windows (or containerized Linux workflows), you build a lightweight container image from Amazon Linux 2023 Minimal. (See [docker/Dockerfile](https://github.com/aadityapageni/aws-simulator-test/blob/main/docker/Dockerfile) and [docker/run-emulator.sh](https://github.com/aadityapageni/aws-simulator-test/blob/main/docker/run-emulator.sh) in the companion repo.)

```dockerfile
FROM public.ecr.aws/amazonlinux/amazonlinux:2023-minimal
# Replace x86_64 with aarch64 on Apple Silicon / ARM64 machines
ADD https://downloads.aws.com/aws-emulator/packages/aws-emulator/LTS/latest/aws-emulator-x86_64.rpm /tmp/aws-emulator.rpm
RUN rpm -i /tmp/aws-emulator.rpm && rm /tmp/aws-emulator.rpm \
    && mkdir -p /home/emulator && chown 1000 /home/emulator
USER 1000
ENV HOME=/home/emulator
EXPOSE 9297
ENTRYPOINT ["aws-emulator", "-b", "0.0.0.0"]
```

Build and run the container, mounting `~/.aws/emulator` so your host SDKs can read the generated credentials and `ca.pem`:

```bash
docker build -t aws-emulator .
mkdir -p ~/.aws/emulator

PORT=9297
docker run --rm --memory=2g --memory-swap=2g \
  -p 127.0.0.1:$PORT:$PORT \
  --user "$(id -u):$(id -g)" \
  -e HOME="$HOME" \
  -v "$HOME/.aws/emulator:$HOME/.aws/emulator" \
  aws-emulator -p $PORT
```

> **Security Note:** Always publish the port on loopback (`-p 127.0.0.1:$PORT:$PORT`), never `-p 9297:9297`, so other machines on your network cannot reach your local emulator.

### Path D: Embedded Mode for Rust
If you write applications in Rust using the AWS SDK for Rust, you can add the `aws-emulator-embedded` crate to `[dev-dependencies]`. Each test spins up its own isolated, in-process emulator without binding network ports or writing certificate files.

## 3. Pointing the AWS CLI and SDKs at AWS Emulator (Zero Code Changes)

Once `aws-emulator` is running on port `9297`, configure your terminal shell so any AWS CLI command or SDK client talks to the local emulator instead of AWS. (A ready-to-source script is included in [env-setup.sh](https://github.com/aadityapageni/aws-simulator-test/blob/main/env-setup.sh).)

```bash
# 1. Point endpoints, credentials, config, and TLS CA bundle to the emulator's port directory
export AWS_ENDPOINT_URL=https://127.0.0.1:9297
export AWS_SHARED_CREDENTIALS_FILE=~/.aws/emulator/9297/emulator_credentials
export AWS_CONFIG_FILE=~/.aws/emulator/9297/config
export AWS_CA_BUNDLE=~/.aws/emulator/9297/ca.pem

# 2. Prevent SDKs from querying EC2 instance metadata or mutating the endpoint hostname
export AWS_EC2_METADATA_DISABLED=true
export AWS_DISABLE_HOST_PREFIX_INJECTION=true
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1

# 3. CRITICAL: Unset real AWS credential environment variables so they don't override the file
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_PROFILE
```

### Verify You Are Talking to the Emulator
Before running tests, always verify your caller identity:

```bash
$ aws sts get-caller-identity
{
    "UserId": "EMULATORUSERID",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:user/emulator"
}
```

If you see `"Account": "000000000000"`, your shell is safely isolated from real AWS.

### SDK-Specific TLS Certificate & Hostname Nuances

While the AWS CLI, Python (Boto3), and Go v2 automatically trust `AWS_CA_BUNDLE`, other language runtimes need one extra environment variable to trust the emulator's self-signed certificate:

| Client / SDK | Extra HTTPS Certificate Setting | Host Prefix Injection Setting |
| :--- | :--- | :--- |
| **AWS CLI v2** | None (reads `AWS_CA_BUNDLE`) | `export AWS_DISABLE_HOST_PREFIX_INJECTION=true` |
| **Python (Boto3)** | None (reads `AWS_CA_BUNDLE`) | `export AWS_DISABLE_HOST_PREFIX_INJECTION=true` or `Config(inject_host_prefix=False)` |
| **Go SDK v2** | None (reads `AWS_CA_BUNDLE`) | Handled via endpoint resolver |
| **JavaScript v3 (Node.js)** | `export NODE_EXTRA_CA_CERTS=~/.aws/emulator/9297/ca.pem` | Pass `disableHostPrefix: true` to client constructor |
| **Java SDK 2.x** | `export JAVA_TOOL_OPTIONS="-Djavax.net.ssl.trustStore=$HOME/.aws/emulator/9297/truststore.p12 -Djavax.net.ssl.trustStoreType=PKCS12"` | Set `SdkAdvancedClientOption.DISABLE_HOST_PREFIX_INJECTION` to `true` |
| **Rust SDK (Process Mode)** | `export SSL_CERT_FILE=~/.aws/emulator/9297/ca.pem` | Or use `aws-emulator-embedded` (no TLS needed) |
| **.NET SDK** | `export SSL_CERT_FILE=~/.aws/emulator/9297/ca.pem` | Set `DisableHostPrefixInjection = true` in client config |

> **Why `127.0.0.1` Instead of `localhost`?**
> Always use `https://127.0.0.1:9297` in `AWS_ENDPOINT_URL`. Because `127.0.0.1` is an IP address, the AWS SDKs for JavaScript, Java, and Go automatically keep Amazon S3 bucket names in the URL **path** (`https://127.0.0.1:9297/my-bucket`) rather than prepending the bucket name to the hostname (`https://my-bucket.localhost:9297`), which would fail DNS resolution.

## 4. Hands-On Code Examples

All examples below are included as runnable files in the [aadityapageni/aws-simulator-test](https://github.com/aadityapageni/aws-simulator-test) repository.

### Example 1: AWS CLI: Stateful Persistence, Exact Validation Errors & Host Prefixes
*(See [cli-walkthrough.sh](https://github.com/aadityapageni/aws-simulator-test/blob/main/cli-walkthrough.sh))*

First, create an Amazon S3 bucket and query it back. The emulator retains state in memory across commands:

```bash
$ aws s3api create-bucket --bucket amzn-s3-demo-bucket
{
    "Location": "/amzn-s3-demo-bucket"
}

$ aws s3api list-buckets --query 'Buckets[].Name'
[
    "amzn-s3-demo-bucket"
]
```

Next, test how AWS Emulator handles **invalid API inputs**. On real AWS, Amazon DynamoDB table names must be at least 3 characters long. If we try to create a table named `ab`, the emulator rejects the call with the exact `ValidationException` message that real AWS returns:

```bash
$ aws dynamodb create-table --table-name ab \
    --attribute-definitions AttributeName=id,AttributeType=S \
    --key-schema AttributeName=id,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST

An error occurred (ValidationException) when calling the CreateTable operation: 1 validation error detected: Value 'ab' at 'tableName' failed to satisfy constraint: Member must have length greater than or equal to 3
```

Finally, consider operations like AWS Cloud Map `DiscoverInstances`, which normally prepend `data-` to the endpoint hostname (`https://data-127.0.0.1:9297/`). With `AWS_DISABLE_HOST_PREFIX_INJECTION=true` set, the request routes cleanly to the emulator:

```bash
$ export AWS_DISABLE_HOST_PREFIX_INJECTION=true
$ aws servicediscovery discover-instances \
    --namespace-name my-namespace \
    --service-name my-service \
    --query 'Instances[].{InstanceId: InstanceId, Attributes: Attributes}'
[
    {
        "InstanceId": "my-instance",
        "Attributes": {
            "AWS_INSTANCE_IPV4": "10.0.0.1"
        }
    }
]
```

### Example 2: Python (Boto3): Cross-Service S3 + DynamoDB Workflow
*(See [python-boto3/document_store.py](https://github.com/aadityapageni/aws-simulator-test/blob/main/python-boto3/document_store.py) and [python-boto3/test_document_store.py](https://github.com/aadityapageni/aws-simulator-test/blob/main/python-boto3/test_document_store.py))*

Notice that our production Python module contains **no hardcoded endpoints, credentials, or test flags**. It simply creates standard `boto3` clients:

```python
# python-boto3/document_store.py
import hashlib
import json
from datetime import datetime, timezone
import boto3


class DocumentStore:
    """Stores JSON documents in S3 and indexes their metadata in DynamoDB."""

    def __init__(self, bucket_name: str, table_name: str) -> None:
        self.bucket_name = bucket_name
        self.table_name = table_name
        self.s3 = boto3.client("s3")
        self.dynamodb = boto3.resource("dynamodb")
        self.table = self.dynamodb.Table(table_name)

    def provision(self) -> None:
        self.s3.create_bucket(Bucket=self.bucket_name)
        client = self.dynamodb.meta.client
        client.create_table(
            TableName=self.table_name,
            AttributeDefinitions=[{"AttributeName": "doc_id", "AttributeType": "S"}],
            KeySchema=[{"AttributeName": "doc_id", "KeyType": "HASH"}],
            BillingMode="PAY_PER_REQUEST",
        )
        # Just like real AWS, CreateTable starts in CREATING status before becoming ACTIVE
        self.table.wait_until_exists(WaiterConfig={"Delay": 1, "MaxAttempts": 10})


    def save_document(self, doc_id: str, payload: dict) -> dict:
        raw_bytes = json.dumps(payload, sort_keys=True).encode("utf-8")
        sha256_hex = hashlib.sha256(raw_bytes).hexdigest()
        s3_key = f"documents/{doc_id}.json"

        self.s3.put_object(
            Bucket=self.bucket_name,
            Key=s3_key,
            Body=raw_bytes,
            ContentType="application/json",
        )

        item = {
            "doc_id": doc_id,
            "s3_key": s3_key,
            "sha256": sha256_hex,
            "size_bytes": len(raw_bytes),
            "created_at": datetime.now(timezone.utc).isoformat(),
        }
        self.table.put_item(Item=item)
        return item

    def get_document(self, doc_id: str) -> dict:
        meta = self.table.get_item(Key={"doc_id": doc_id}).get("Item")
        if not meta:
            raise KeyError(f"Document {doc_id} not found in index")

        obj = self.s3.get_object(Bucket=self.bucket_name, Key=meta["s3_key"])
        body = json.loads(obj["Body"].read().decode("utf-8"))
        return {"metadata": meta, "content": body}
```

With our emulator environment variables exported (`source env-setup.sh`), our `pytest` suite tests both the happy path across S3 and DynamoDB and the DynamoDB validation constraint without touching the cloud:

```python
# python-boto3/test_document_store.py
import uuid
import pytest
from botocore.exceptions import ClientError
from document_store import DocumentStore


def test_save_and_retrieve_document():
    suffix = uuid.uuid4().hex[:8]
    store = DocumentStore(
        bucket_name=f"docs-bucket-{suffix}",
        table_name=f"docs-index-{suffix}",
    )
    store.provision()

    saved_meta = store.save_document("order-1001", {"customer": "Ada", "total": 42})
    result = store.get_document("order-1001")

    assert result["content"] == {"customer": "Ada", "total": 42}
    assert result["metadata"]["sha256"] == saved_meta["sha256"]


def test_dynamodb_rejects_short_table_name():
    store = DocumentStore(bucket_name="valid-bucket-name", table_name="ab")
    with pytest.raises(ClientError) as exc_info:
        store.provision()

    err = exc_info.value.response["Error"]
    assert err["Code"] == "ValidationException"
    assert "length greater than or equal to 3" in err["Message"]
```

### Example 3: Rust Embedded Mode: Parallel In-Process Testing
*(See [rust-embedded/Cargo.toml](https://github.com/aadityapageni/aws-simulator-test/blob/main/rust-embedded/Cargo.toml) and [rust-embedded/src/lib.rs](https://github.com/aadityapageni/aws-simulator-test/blob/main/rust-embedded/src/lib.rs))*

In Rust projects, you don't even need a background `aws-emulator` daemon. Add `aws-emulator-embedded` to `[dev-dependencies]` pinned to a specific release commit:

```toml
[dependencies]
aws-config = "1"
aws-sdk-s3 = "1"

[dev-dependencies]
aws-emulator-embedded = { git = "https://github.com/aws/aws-emulator", rev = "<release-commit>" }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }

# aws-emulator-embedded links 400+ service crates; stripping debug sections at
# link time prevents >4GiB 32-bit DWARF (.debug_macro) relocation overflow in ld.lld.
[profile.dev]
debug = false
strip = "debuginfo"
```

Each `#[tokio::test]` creates its own `AwsEmulatorEmbedded` instance and passes `emulator.http_client()` and `emulator.aws_credentials_provider()` to `aws_config`. Two tests can even create the exact same bucket name simultaneously without colliding:

```rust
use aws_emulator_embedded::{Authentication, AwsEmulatorEmbedded, EmbeddedOptions};
use aws_sdk_s3::primitives::ByteStream;

#[tokio::test]
async fn put_then_get_returns_the_body() {
    // Dedicated in-memory emulator for this test only (with SigV4 auth enabled)
    let emulator = AwsEmulatorEmbedded::new(
        EmbeddedOptions::default().authentication(Authentication::On),
    );

    let config = aws_config::defaults(aws_config::BehaviorVersion::latest())
        .http_client(emulator.http_client())
        .region(aws_config::Region::new("us-east-1"))
        .credentials_provider(emulator.aws_credentials_provider())
        .load()
        .await;

    let s3 = aws_sdk_s3::Client::new(&config);

    s3.create_bucket()
        .bucket("emulator-demo-bucket")
        .send()
        .await
        .unwrap();

    s3.put_object()
        .bucket("emulator-demo-bucket")
        .key("greeting.txt")
        .body(ByteStream::from_static(b"hello from embedded emulator"))
        .send()
        .await
        .unwrap();

    let got = s3
        .get_object()
        .bucket("emulator-demo-bucket")
        .key("greeting.txt")
        .send()
        .await
        .unwrap();

    assert_eq!(
        got.body.collect().await.unwrap().to_vec(),
        b"hello from embedded emulator"
    );
}
```

### Example 4: CI/CD Pipeline Integration (GitHub Actions)
*(See [ci/github-actions.yml](https://github.com/aadityapageni/aws-simulator-test/blob/main/ci/github-actions.yml))*

In CI/CD pipelines, AWS Emulator eliminates the need for OIDC IAM roles, long-lived AWS secrets, or shared staging accounts for functional tests. Two flags are particularly helpful in CI:
- `--idle-timeout 300`: Automatically shuts down the emulator after 5 minutes of inactivity so hung jobs never linger.
- `--telemetry off`: Disables anonymous usage telemetry if your CI environment restricts outbound calls.

```yaml
name: Integration Tests (AWS Emulator)

on: [push, pull_request]

jobs:
  integration-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install AWS Emulator
        run: |
          curl -fsSLo aws-emulator.deb https://downloads.aws.com/aws-emulator/packages/aws-emulator/LTS/latest/aws-emulator_amd64.deb
          sudo apt-get install -y ./aws-emulator.deb
          aws-emulator -v

      - name: Start AWS Emulator & Run Tests
        run: |
          # Start emulator in background with a 120-second idle timeout
          aws-emulator --idle-timeout 120 --telemetry off &

          # Poll the HTTPS health check endpoint until ready
          for i in $(seq 30); do
            if [ -f ~/.aws/emulator/9297/ca.pem ] && \
               curl -sf --cacert ~/.aws/emulator/9297/ca.pem https://127.0.0.1:9297/__admin/health; then
              echo "AWS Emulator is healthy!"
              break
            fi
            sleep 1
          done

          # Export emulator variables and unset any ambient AWS credentials
          source ./env-setup.sh
          PYTHONPATH=python-boto3 pytest python-boto3/
```

### Example 5: Infrastructure as Code with Terraform (`init` / `plan` / `apply` / `destroy`)
*(See [terraform/main.tf](https://github.com/aadityapageni/aws-simulator-test/blob/main/terraform/main.tf))*

I have also tested Terraform (`hashicorp/aws v5.100.0`) against AWS Emulator. Because the `hashicorp/aws` Terraform provider is built on the **AWS SDK for Go v2**, it natively reads `AWS_ENDPOINT_URL`, `AWS_SHARED_CREDENTIALS_FILE`, `AWS_CONFIG_FILE`, and `AWS_CA_BUNDLE`. Unlike older local emulators that require hardcoding custom `endpoints {}` blocks for every service, your Terraform configuration stays clean:

```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region            = "us-east-1"
  s3_use_path_style = true
}

data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "app_bucket" {
  bucket = "tf-emulator-demo-bucket"
}

resource "aws_dynamodb_table" "app_table" {
  name         = "tf-emulator-orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "order_id"

  attribute {
    name = "order_id"
    type = "S"
  }
}

resource "aws_sqs_queue" "app_queue" {
  name                       = "tf-emulator-events-queue"
  visibility_timeout_seconds = 45
}
```

Run `terraform init`, source the environment variables, and apply against the local emulator:

```bash
$ cd terraform
$ terraform init
$ source ../env-setup.sh 9297
$ terraform apply -auto-approve
...
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

dynamodb_table_arn = "arn:aws:dynamodb:us-east-1:000000000000:table/tf-emulator-orders"
emulator_account_id = "000000000000"
s3_bucket_arn = "arn:aws:s3:::tf-emulator-demo-bucket"
sqs_queue_url = "https://sqs.us-east-1.amazonaws.com/000000000000/tf-emulator-events-queue"
```

> **Terraform TLS Tip:** Use `AWS_CA_BUNDLE` (which is read only by the AWS Go SDK inside `terraform-provider-aws`), and **do not** set `SSL_CERT_FILE` when running `terraform init`. Setting `SSL_CERT_FILE` replaces the system CA bundle for the Terraform CLI itself, which would block `terraform init` from verifying `registry.terraform.io` over HTTPS.

### Example 6: Four Real-World Production Scenarios (Verified in `pytest`)
*(See [python-boto3/test_real_life_scenarios.py](https://github.com/aadityapageni/aws-simulator-test/blob/main/python-boto3/test_real_life_scenarios.py))*

Beyond basic CRUD and Terraform, we also tested four high-value real-world engineering workflows against the live emulator:

1. **In-Process AWS Lambda `.zip` Execution with IAM Role Enforcement**:
   - AWS Emulator embeds Python and Node.js runtimes directly in the binary. When you upload a `.zip` package with `create_function` and call `lambda.invoke()`, it runs your handler in-process and executes any nested `boto3` calls *as the function's IAM execution role*, catching missing role permissions (such as `s3:PutObject`) locally without needing Docker containers.
2. **Event-Driven SNS -> SQS Fan-Out & Queue Policy Validation**:
   - Subscribing an SQS queue to an SNS topic and calling `sns.publish()` delivers the SNS JSON envelope into the SQS queue **only** if you attach an SQS Queue Policy granting `sns.amazonaws.com` permission to call `sqs:SendMessage` (matching real AWS behavior and catching silent drop bugs locally).
3. **S3 Presigned URLs (`PUT` and `GET`) for Frontend Upload Testing**:
   - `s3.generate_presigned_url("put_object", ...)` and `"get_object"` generate SigV4-signed URLs that work with standard HTTP clients (`urllib`, `fetch`, `curl`), making it easy to test browser/mobile direct-to-S3 upload flows locally.
4. **CloudFormation Stack Deployment (`create_stack`)**:
   - Deploying a CloudFormation template with `AWS::S3::Bucket` and `AWS::SQS::Queue` transitions from `CREATE_IN_PROGRESS` to `CREATE_COMPLETE` and materializes the actual bucket and queue in the S3 and SQS services.

## 5. Understanding Fidelity Scores & Checking Your Project

One of the biggest challenges with any cloud emulator is knowing **where the emulation stops**. AWS solves this transparently through **Fidelity Tests** and **Fidelity Scores**.

### How Fidelity Is Measured
- **Supported API Surface**: AWS Emulator supports AWS services that publish an API model in the official [aws/api-models-aws](https://github.com/aws/api-models-aws) repository.
- **Fast Failure on Unimplemented Operations**: If your code calls an operation that AWS Emulator does not implement yet, it immediately returns **HTTP 501 (`NotImplemented`)** rather than silently returning fake empty data.
- **Fidelity Tests (`aws-emulator-evaluation-suite`)**: A fidelity test is an executable SDK workflow verified against real AWS in `us-west-2` (or `us-east-1` for global services). To pass a test, the emulator must match AWS on every call in the workflow: status code, created resources, side effects, and exact error codes/messages on invalid requests.
- **Control Plane vs. Data Plane**: Each test in `manifest.jsonl` is labeled by plane. Today, AWS Emulator has its highest fidelity on **control plane APIs** (create, describe, update, delete, and list resources), while **data plane APIs** and Infrastructure-as-Code (CloudFormation, Terraform, CDK) support vary by service.
- **No Regressions**: Passing fidelity tests are locked in and do not regress across releases.

### How to Check If Your Project Will Work
1. **Automated (Recommended)**: Install the `aws-emulator` skill and ask your AI coding agent: *"Will my app in this repo work on AWS Emulator?"*
2. **Manual Check**:
   - List the AWS service operations your code invokes.
   - Check `manifest.jsonl` in [github.com/aws/aws-emulator-evaluation-suite](https://github.com/aws/aws-emulator-evaluation-suite).
   - Review the per-service fidelity scores published on the [AWS Emulator GitHub Releases page](https://github.com/aws/aws-emulator/releases), or run `fidelity_tests_runner` locally against your installed build.

## 6. How AWS Emulator Compares to Existing Tools

The AWS ecosystem already has several popular local testing tools, most notably **LocalStack**, **Moto**, **single-service official emulators** (DynamoDB Local, Step Functions Local, MinIO/S3Mock), and **AWS SAM CLI**. How does AWS Emulator fit into this landscape?

### Feature Comparison Matrix

| Dimension | **AWS Emulator** | **LocalStack (Community / Pro)** | **Moto (Library / Server)** | **Single-Service Tools (DynamoDB Local, MinIO, etc.)** |
| :--- | :--- | :--- | :--- | :--- |
| **Maintainer & License** | **Official AWS** · Open Source (**CC0 1.0 Public Domain** + Apache 2.0 vendor) | Third-party (LocalStack GmbH) · Apache 2.0 (Community) / Commercial (Pro) | Community Open Source · Apache 2.0 | Mixed (AWS proprietary JAR for DynamoDB Local, AGPLv3 for MinIO) |
| **Cost** | **100% Free** (no gated tiers) | Free tier + Paid Pro/Enterprise tiers for advanced services & features | 100% Free | Free |
| **Runtime & Footprint** | Single native binary (~160 MiB idle RAM) or Rust in-process library | Docker container (Python + JVM + Go runtimes, heavier RAM footprint) | Python in-process decorator/context manager or Flask server | Separate JVM / Go processes per service |
| **Multi-Service Single Endpoint** | **Yes** (`https://127.0.0.1:9297`) | **Yes** (`http://localhost:4566`) | Yes (in server mode) | **No** (separate port & config per service) |
| **In-Process Parallel Testing** | **Yes (Rust SDK)** via `aws-emulator-embedded` | No (network daemon / container) | **Yes (Python Boto3)** via `@mock_aws` | No |
| **Update Cadence & Verification** | **Continuous (multiple times/day via AI agents)** + public `aws-emulator-evaluation-suite` | Frequent releases; internal & community test suites | Community PR-driven | Periodic releases per service team |
| **Security & SigV4 by Default** | **Strict HTTPS + per-run ephemeral SigV4 credentials** by default | Permissive HTTP by default (`test`/`test` credentials unless configured) | Permissive HTTP / mocked botocore hooks | Varies by tool |
| **Control Plane vs. Data Plane** | Strongest on **Control Plane APIs**; explicit HTTP 501 on unimplemented calls | Deep **Data Plane + Execution** (Lambda containers, RDS, EventBridge pipes, CDK/TF) | Broad Python Boto3 coverage across control & basic data planes | Deep data plane for that **single** service only |
| **AI Coding Agent Integration** | **Built-in `aws-emulator` skill** via AWS Agent Toolkit | Community MCP / scripts | None built-in | None built-in |

### Architectural Breakdown: When to Choose Which

#### 1. AWS Emulator vs. LocalStack
- **Where AWS Emulator wins**:
  - **First-party, zero-cost license**: Entirely free under CC0 1.0 public domain with no features locked behind a commercial license.
  - **Lightweight native execution**: Runs directly as a ~160 MiB Linux process or embedded inside Rust tests without requiring a heavy multi-runtime Docker container.
  - **Published, reproducible fidelity scores**: Every release is scored against `aws-emulator-evaluation-suite` verified against real `us-west-2` AWS responses, and unimplemented APIs fail cleanly with `501 NotImplemented`.
  - **Secure-by-default local posture**: Loopback HTTPS + per-run SigV4 keys protect developer laptops from drive-by local requests.
- **Where LocalStack still leads today**:
  - **Complex data-plane execution & IaC**: If your local workflow requires executing containerized Lambda functions triggered by SQS/EventBridge, stateful local RDS Postgres instances, S3 Control operations, or full CloudFormation/CDK stack deployments with IAM policy enforcement, LocalStack Pro has years of specialized runtime integrations.

#### 2. AWS Emulator vs. Moto
- **Where AWS Emulator wins**:
  - **Language-agnostic out of the box**: Because AWS Emulator is a native HTTPS server speaking real AWS wire protocols, the exact same binary tests Python, TypeScript, Go, Java, Rust, .NET, and shell scripts.
  - **Wire-level & TLS realism**: Moto primarily intercepts `botocore` calls in memory inside Python; AWS Emulator tests the full HTTP serialization, SigV4 signing, host-prefix behavior, and TLS configuration of your SDK.
  - **Rust native embedded mode**: `aws-emulator-embedded` brings Moto-style zero-network in-process isolation to the Rust ecosystem.
- **Where Moto is still handy**:
  - Pure Python unit tests on macOS/Windows where you don't want to run Docker and just want a quick `@mock_aws` decorator around a `boto3` function.

#### 3. AWS Emulator vs. DynamoDB Local / S3Mock / MinIO
- Stitching together `amazon/dynamodb-local` on port `8000`, `minio` on port `9000`, and `stepfunctions-local` on port `8083` forces you to override a different `endpoint_url` per service client in your application code.
- **AWS Emulator** serves all emulated services on a single `AWS_ENDPOINT_URL` (`https://127.0.0.1:9297`), so your application code stays completely untouched.

## 7. Important Limitations & Best Practices

AWS Emulator is purpose-built for **building, local testing, CI integration testing, and AI-assisted prototyping**. Keep the following constraints in mind:

1. **Never Use AWS Emulator to Test Security or IAM Policies**:
   The emulator **never denies a request signed with its own run key**, and it does not evaluate IAM policies, permission boundaries, S3 bucket policies, or KMS key policies the way real AWS does. Always validate IAM and security controls in a real AWS environment before production.
2. **One Account (`000000000000`) and One Region per Instance**:
   While the emulator accepts any Region in requests, it does not partition resources cleanly across accounts or Regions (and its behavior is verified against `us-west-2`, or `us-east-1` for global services).
   - *Best Practice*: Use a single Region (`us-east-1` or `us-west-2`) per emulator. If you need to test multi-region behavior, launch two emulator processes on separate ports (`aws-emulator -p 9297` and `aws-emulator -p 9300`).
3. **In-Memory Ephemeral State Only**:
   There is no persistence to disk and no admin reset endpoint. To reset state between test suites, simply stop and restart the emulator process (or use `--idle-timeout`).
4. **Amazon S3 Control Is Not Supported**:
   AWS SDKs always prepend the account ID to the hostname for S3 Control requests (e.g., `000000000000.127.0.0.1`), which cannot be disabled via configuration and therefore cannot reach the emulator.
5. **Managing Telemetry**:
   By default, the emulator sends anonymous aggregate metrics (operation counts, error categories, and startup/memory ranges, never payloads, ARNs, credentials, or free text). You can opt out at any time via:
   - CLI flag: `aws-emulator --telemetry off` (or `-t off`)
   - Environment variable: `export EMULATOR_TELEMETRY=off`
   - Config file (`~/.aws/emulator/config.toml`):
     ```toml
     [telemetry]
     enabled = false
     ```
   - Rust Embedded API: `EmbeddedOptions::default().telemetry(Telemetry::Off)`

## 8. Conclusion & Resources

**AWS Emulator** fills a long-standing gap in the AWS developer experience: an official, fast, zero-cost, open-source local API emulator backed by continuous AI-driven updates and transparent fidelity testing. Whether you are running integration tests in CI without cloud credentials, writing parallel Rust tests in memory, or building new cloud features locally with an AI coding agent, AWS Emulator lets you iterate at local speed with production-grade API validation.

### Official Links
- **Companion Examples Repository**: [github.com/aadityapageni/aws-simulator-test](https://github.com/aadityapageni/aws-simulator-test)
- **Documentation**: [AWS Emulator User Guide](https://docs.aws.amazon.com/emulator/latest/userguide/what-is-service.html)
- **Source Code**: [github.com/aws/aws-emulator](https://github.com/aws/aws-emulator)
- **Releases & Fidelity Scores**: [github.com/aws/aws-emulator/releases](https://github.com/aws/aws-emulator/releases)
- **Fidelity Evaluation Suite**: [github.com/aws/aws-emulator-evaluation-suite](https://github.com/aws/aws-emulator-evaluation-suite)
- **Agent Toolkit Skill Guide**: [Managing skills with the AWS CLI](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/aws-cli.html)
