---
name: terraform-aws
description: >-
  AWS provider 5+ patterns for production infrastructure. Use for default_tags, IAM policy
  documents, VPC and EC2, Lambda (zip and container images on ECR), Kinesis streams and fine-grained
  Lambda event source mapping (batching, partial failure), IoT Core (JIT provisioning templates, topic
  rules, thing types, policies), hardened S3, RDS, EKS, EventBridge, Step Functions, and SSM. Includes an
  end-to-end reference: 100 Hz IMU telemetry through IoT/Kinesis into a Python container Lambda with
  tuned batching and ReportBatchItemFailures.
---

# Terraform on AWS (provider 5.x patterns)

## Scope

Use this skill when selecting **AWS resources**, structuring **IAM**, applying **default tags**, and
designing **event-driven** paths. Pair with `terraform-core` for language constructs and
`terraform-state` for remote backends on S3.

## Provider-wide default_tags (AWS provider 5+)

Set organizational tags once; resources that support AWS **resource tagging APIs** inherit them
automatically, reducing drift between teams.

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Project     = var.project
      Environment = var.environment
      CostCenter  = var.cost_center
      ManagedBy   = "terraform"
    }
  }
}
```

Explicit `tags` on a resource **merge** with defaults; collisions should be rare—prefer unique
keys per layer (`Service` on resources, `Project` globally).

## IAM policy documents (data source)

Prefer `aws_iam_policy_document` over heredoc JSON strings for linting and `source_policy_documents`
composition.

```hcl
data "aws_iam_policy_document" "lambda_kinesis" {
  statement {
    sid    = "ReadStream"
    effect = "Allow"
    actions = [
      "kinesis:DescribeStreamSummary",
      "kinesis:DescribeStream",
      "kinesis:GetRecords",
      "kinesis:GetShardIterator",
      "kinesis:ListShards"
    ]
    resources = [aws_kinesis_stream.imu_ingest.arn]
  }

  statement {
    sid    = "WriteTelemetryDLQ"
    effect = "Allow"
    actions = [
      "sqs:SendMessage"
    ]
    resources = [aws_sqs_queue.imu_failures.arn]
  }
}

resource "aws_iam_role_policy" "lambda_kinesis" {
  name   = "imu-processor-kinesis"
  role   = aws_iam_role.lambda_exec.id
  policy = data.aws_iam_policy_document.lambda_kinesis.json
}
```

## VPC baseline (compact example)

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
}

resource "aws_subnet" "private" {
  for_each = toset(var.azs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, index(var.azs, each.key))
  availability_zone = each.key
}

resource "aws_security_group" "lambda" {
  name_prefix = "imu-lambda-"
  vpc_id      = aws_vpc.this.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## ECR repository for Python Lambda container

```hcl
resource "aws_ecr_repository" "imu_processor" {
  name                 = "imu-processor"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }
}

resource "aws_ecr_lifecycle_policy" "imu_processor" {
  repository = aws_ecr_repository.imu_processor.name
  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Retain last 30 images"
      selection = {
        tagStatus   = "any"
        countType   = "imageCountMoreThan"
        countNumber = 30
      }
      action = { type = "expire" }
    }]
  })
}
```

Lambda will reference `${aws_ecr_repository.imu_processor.repository_url}:stable` after your CI
pushes.

## Reference architecture: IoT, Kinesis, container Lambda

**Story:** edge IMU gateways publish **JSON telemetry** (`ax, ay, az, ts`) via MQTT at roughly **100 Hz**
per device to AWS IoT Core. A topic rule filters and forwards to **Kinesis Data Streams**. A **Python**
processor runs as AWS Lambda (**package type Image** from ECR), consumes the stream with **large
micro-batches** and partial failure semantics tuned for noisy sensors.

### IoT thing type and policy

```hcl
resource "aws_iot_thing_type" "imu_gateway" {
  name = "IMUGateway"
}

resource "aws_iot_policy" "imu_gateway_connect" {
  name = "imu-gateway-connect"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "iot:Connect",
          "iot:Publish",
          "iot:Subscribe",
          "iot:Receive"
        ]
        Resource = [
          "arn:aws:iot:${var.region}:${data.aws_caller_identity.current.account_id}:topic/imu/${var.env}/*",
          "arn:aws:iot:${var.region}:${data.aws_caller_identity.current.account_id}:client/$${iot:Connection.Thing.ThingName}"
        ]
      }
    ]
  })
}
```

Use `$${...}` in Terraform strings to emit **literal** `${...}` for IoT policy variables.

### Just-in-time provisioning (JITP)

**JITP** registers uncataloged devices into your registry using a template and IAM role.

```hcl
resource "aws_iot_provisioning_template" "imu_jitp" {
  name                  = "${var.env}-imu-jitp"
  description           = "Bootstrap IMU gateways"
  provisioning_role_arn = aws_iam_role.iot_jitp.arn
  enabled               = true

  template_body = jsonencode({
    Parameters = {
      AWS::IoT::Certificate::Id = { Type = "String" }
      SerialNumber              = { Type = "String" }
    }
    Resources = {
      thing = {
        Type = "AWS::IoT::Thing"
        Properties = {
          ThingName = { Ref = "SerialNumber" }
          ThingTypeName = aws_iot_thing_type.imu_gateway.name
          AttributePayload = {
            Attributes = {
              Provisioning = "JITP"
            }
          }
        }
      }
      cert = {
        Type = "AWS::IoT::Certificate"
        Properties = {
          CertificateId = { Ref = "AWS::IoT::Certificate::Id" }
          Status          = "Active"
          ThingPrincipal  = { Ref = "thing" }
        }
      }
    }
  })
}

data "aws_caller_identity" "current" {}
```

Associate fleet provisioning **in the AWS console or API** to point first-connection devices at this
template; Terraform covers template + IAM role wiring.

### Kinesis stream for high-velocity telemetry

Use **on-demand** mode for variable device fan-out or **provisioned** mode when you model predictable
TPS/shards. For ~100 Hz per device, aggregate across devices when sizing; spikes may need **retention**
and **enhanced fan-out** for multiple consumers.

```hcl
resource "aws_kinesis_stream" "imu_ingest" {
  name            = "${var.env}-imu-ingest"
  stream_mode_details {
    stream_mode = "ON_DEMAND"
  }

  retention_period = 72 # hours
}
```

### IoT topic rule to Kinesis

```hcl
resource "aws_iam_role" "iot_actions" {
  name = "iot-actions-imu"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "iot.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

data "aws_iam_policy_document" "iot_publish_kinesis" {
  statement {
    effect    = "Allow"
    actions   = ["kinesis:PutRecord", "kinesis:PutRecords"]
    resources = [aws_kinesis_stream.imu_ingest.arn]
  }
}

resource "aws_iam_role_policy" "iot_publish_kinesis" {
  role   = aws_iam_role.iot_actions.id
  policy = data.aws_iam_policy_document.iot_publish_kinesis.json
}

resource "aws_iot_topic_rule" "imu_to_kinesis" {
  name        = "${var.env}_imu_to_kinesis"
  description = "Forward IMU MQTT payloads to Kinesis"
  enabled     = true
  sql         = <<EOT
SELECT *, topic() AS mqtt_topic, timestamp() AS arrival_ts
FROM 'imu/${var.env}/#'
EOT
  sql_version = "2016-03-23"

  kinesis {
    stream_name   = aws_kinesis_stream.imu_ingest.name
    role_arn      = aws_iam_role.iot_actions.arn
    partition_key = "$${topic()}"
  }
}
```

### Lambda execution role (image-based)

```hcl
resource "aws_iam_role" "lambda_exec" {
  name = "imu-processor-lambda"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume.json
}

data "aws_iam_policy_document" "lambda_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  count      = var.lambda_in_vpc ? 1 : 0
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}
```

### Lambda function (container) + event source mapping tuning

```hcl
resource "aws_lambda_function" "imu_processor" {
  function_name = "${var.env}-imu-processor"
  role            = aws_iam_role.lambda_exec.arn
  package_type    = "Image"
  image_uri       = "${aws_ecr_repository.imu_processor.repository_url}:${var.image_tag}"
  architectures   = ["x86_64"]

  memory_size = 1024
  timeout     = 60

  environment {
    variables = {
          LOG_LEVEL    = "INFO"
          STREAM_NAME  = aws_kinesis_stream.imu_ingest.name
          FAILURES_QUEUE_URL = aws_sqs_queue.imu_failures.id
    }
  }

  dynamic "vpc_config" {
    for_each = var.lambda_in_vpc ? [1] : []
    content {
      subnet_ids         = [for s in aws_subnet.private : s.id]
      security_group_ids = [aws_security_group.lambda.id]
    }
  }
}

resource "aws_lambda_event_source_mapping" "kinesis_imu" {
  event_source_arn  = aws_kinesis_stream.imu_ingest.arn
  function_name     = aws_lambda_function.imu_processor.arn
  starting_position = "LATEST"

  batch_size                         = 500
  maximum_batching_window_in_seconds = 5
  bisect_batch_on_function_error     = true

  function_response_types = ["ReportBatchItemFailures"]

  destination_config {
    on_failure {
      destination_arn = aws_sqs_queue.imu_failures.arn
    }
  }
}
```

**Python handler** (outside Terraform but required in real systems) must return partial batch
failure JSON for `ReportBatchItemFailures` to behave as designed; wire integration tests in
`terraform-testing`.

### Dead-letter queue for poison batches

```hcl
resource "aws_sqs_queue" "imu_failures" {
  name                       = "${var.env}-imu-failures"
  message_retention_seconds  = 1209600
  kms_master_key_id          = aws_kms_key.sqs.id
}
```

## S3 hardening (audit + telemetry archives)

```hcl
resource "aws_s3_bucket" "imu_archive" {
  bucket = "${var.account_id}-${var.env}-imu-archive"
}

resource "aws_s3_bucket_versioning" "imu_archive" {
  bucket = aws_s3_bucket.imu_archive.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "imu_archive" {
  bucket = aws_s3_bucket.imu_archive.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "imu_archive" {
  bucket = aws_s3_bucket.imu_archive.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Add **bucket policy** denying insecure transport and non-TLS.

## RDS (Postgres, minimal)

```hcl
resource "aws_db_subnet_group" "app" {
  name       = "${var.env}-app"
  subnet_ids = [for s in aws_subnet.private : s.id]
}

resource "aws_db_instance" "app" {
  identifier                 = "${var.env}-imu-meta"
  engine                     = "postgres"
  instance_class             = "db.t4g.medium"
  allocated_storage          = 100
  db_subnet_group_name       = aws_db_subnet_group.app.name
  vpc_security_group_ids     = [aws_security_group.rds.id]
  kms_key_id                 = aws_kms_key.rds.arn
  storage_encrypted          = true
  username                   = var.db_username
  manage_master_user_password = true
  backup_retention_period    = 7
  deletion_protection        = true
}
```

## EKS (control plane + ng stub)

```hcl
resource "aws_eks_cluster" "this" {
  name     = "${var.env}-imu"
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids         = [for s in aws_subnet.private : s.id]
    security_group_ids = [aws_security_group.eks.id]
  }
}

resource "aws_eks_node_group" "default" {
  cluster_name    = aws_eks_cluster.this.name
  node_group_name = "default"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = [for s in aws_subnet.private : s.id]

  scaling_config {
    desired_size = 3
    max_size     = 6
    min_size     = 2
  }
}
```

## EventBridge + Step Functions + SSM (automation cues)

```hcl
resource "aws_cloudwatch_event_rule" "imu_alarm" {
  name        = "${var.env}-imu-errors"
  description = "Route DLQ depth alarms"

  event_pattern = jsonencode({
    source      = ["aws.cloudwatch"]
    detail-type = ["CloudWatch Alarm State Change"]
    resources   = [aws_cloudwatch_metric_alarm.imu_dlq.arn]
  })
}

resource "aws_sfn_state_machine" "replay" {
  name     = "${var.env}-imu-replay"
  role_arn = aws_iam_role.sfn.arn

  definition = jsonencode({
    Comment = "Replay failed IMU batches"
    StartAt = "Notify"
    States = {
      Notify = {
        Type     = "Task"
        Resource = "arn:aws:states:::lambda:invoke"
        End      = true
      }
    }
  })
}
```

Use **SSM Parameter Store** for tunable knobs (batch sizes, feature flags) consumed by Lambda via
IAM-scoped `ssm:GetParameter` when dynamic TF changes are too heavy.

## Operational notes

- **Cold starts** with large container images benefit from **provisioned concurrency** (cost trade-off).
- **Kinesis iterator age** alarms catch processor lag before data loss.
- **IoT rule errors** publish to CloudWatch metrics—monitor `Success` vs `Failure`.
- Apply **SCPs** at org level for region denial and service guards—see `terraform-multi-account`.

## When not to use this skill

Cross-cloud patterns → `terraform-azure`, `terraform-gcp`. Kubernetes manifests inside clusters →
`terraform-kubernetes`.

## CI/CD handoff

Build and push the container image in the pipeline **before** `terraform apply` updates `image_uri`.
For immutable tags, reference digests (`image_uri = ...@sha256:...`) when your release process
supports them—this prevents Lambda from silently running stale layers after a failed promotion step.
