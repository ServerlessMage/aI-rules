# AWS Terraform State Management

## S3 Backend Configuration

### S3 Backend with DynamoDB Locking
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state-mycompany-prod"
    key            = "aws/environments/prod/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    
    # AWS-specific optimizations
    max_retries                 = 5
    skip_region_validation      = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    
    # Cross-account state access
    role_arn = "arn:aws:iam::ACCOUNT-ID:role/TerraformStateRole"
  }
}
```

### S3 State Bucket Setup
```hcl
# State bucket with AWS best practices
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-${var.company_name}-${var.environment}"
  
  tags = {
    Name        = "Terraform State Bucket"
    Environment = var.environment
    Purpose     = "terraform-state"
  }
}

# Enable versioning for state history
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.terraform_state.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle configuration for cost optimization
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "state_lifecycle"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 90
    }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_transition {
      noncurrent_days = 60
      storage_class   = "GLACIER"
    }
  }
}

# KMS key for state encryption
resource "aws_kms_key" "terraform_state" {
  description             = "KMS key for Terraform state encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow Terraform State Access"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/TerraformRole"
        }
        Action = [
          "kms:Decrypt",
          "kms:Encrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })

  tags = {
    Name = "terraform-state-key"
  }
}

resource "aws_kms_alias" "terraform_state" {
  name          = "alias/terraform-state"
  target_key_id = aws_kms_key.terraform_state.key_id
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-state-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  # Enable point-in-time recovery
  point_in_time_recovery {
    enabled = true
  }

  # Server-side encryption
  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.terraform_state.arn
  }

  tags = {
    Name = "Terraform State Lock Table"
  }
}
```

## AWS Multi-Account State Organization

### Account-Based State Structure
```
s3://terraform-state-mycompany-prod/
├── aws/
│   ├── accounts/
│   │   ├── dev-account/
│   │   │   ├── networking/terraform.tfstate
│   │   │   ├── security/terraform.tfstate
│   │   │   └── compute/terraform.tfstate
│   │   ├── staging-account/
│   │   └── prod-account/
│   ├── shared/
│   │   ├── organizations/terraform.tfstate
│   │   ├── dns/terraform.tfstate
│   │   └── logging/terraform.tfstate
│   └── regions/
│       ├── us-west-2/
│       └── us-east-1/
```

### Cross-Account State Access
```hcl
# Reference state from different AWS account
data "terraform_remote_state" "shared_networking" {
  backend = "s3"
  config = {
    bucket   = "terraform-state-mycompany-prod"
    key      = "aws/shared/networking/terraform.tfstate"
    region   = "us-west-2"
    role_arn = "arn:aws:iam::${var.shared_account_id}:role/TerraformStateReadRole"
  }
}

# Use cross-account VPC peering
resource "aws_vpc_peering_connection" "cross_account" {
  vpc_id        = aws_vpc.main.id
  peer_vpc_id   = data.terraform_remote_state.shared_networking.outputs.vpc_id
  peer_owner_id = var.shared_account_id
  peer_region   = var.shared_region
  auto_accept   = false

  tags = {
    Name = "Cross-account VPC peering"
  }
}
```

## AWS Organizations Integration

### Organization-Wide State Management
```hcl
# Organization master account state
data "terraform_remote_state" "organization" {
  backend = "s3"
  config = {
    bucket = "terraform-state-mycompany-master"
    key    = "aws/organization/terraform.tfstate"
    region = "us-east-1"
  }
}

# Reference organization resources
locals {
  organization_id = data.terraform_remote_state.organization.outputs.organization_id
  master_account_id = data.terraform_remote_state.organization.outputs.master_account_id
}

# Account-specific resources
resource "aws_organizations_account" "workload_account" {
  name  = "${var.project_name}-${var.environment}"
  email = "${var.project_name}-${var.environment}@${var.company_domain}"
  
  parent_id = data.terraform_remote_state.organization.outputs.workloads_ou_id
  
  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

## AWS State Security

### IAM Policies for State Access
```hcl
# Terraform execution role
resource "aws_iam_role" "terraform_role" {
  name = "TerraformExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/terraform-ci",
            "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/GitHubActionsRole"
          ]
        }
      }
    ]
  })
}

# State bucket access policy
resource "aws_iam_role_policy" "terraform_state_access" {
  name = "TerraformStateAccess"
  role = aws_iam_role.terraform_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${aws_s3_bucket.terraform_state.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket"
        ]
        Resource = aws_s3_bucket.terraform_state.arn
      },
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:DeleteItem"
        ]
        Resource = aws_dynamodb_table.terraform_locks.arn
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:Encrypt",
          "kms:GenerateDataKey"
        ]
        Resource = aws_kms_key.terraform_state.arn
      }
    ]
  })
}
```

### Cross-Account State Access Roles
```hcl
# Read-only state access for other accounts
resource "aws_iam_role" "terraform_state_read" {
  name = "TerraformStateReadRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          AWS = [
            for account_id in var.trusted_account_ids :
            "arn:aws:iam::${account_id}:role/TerraformRole"
          ]
        }
        Condition = {
          StringEquals = {
            "sts:ExternalId" = var.external_id
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "terraform_state_read" {
  name = "TerraformStateReadAccess"
  role = aws_iam_role.terraform_state_read.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject"
        ]
        Resource = "${aws_s3_bucket.terraform_state.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket"
        ]
        Resource = aws_s3_bucket.terraform_state.arn
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt"
        ]
        Resource = aws_kms_key.terraform_state.arn
      }
    ]
  })
}
```

## AWS State Monitoring

### CloudWatch Monitoring
```hcl
# CloudWatch alarms for state bucket
resource "aws_cloudwatch_metric_alarm" "state_bucket_errors" {
  alarm_name          = "terraform-state-bucket-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "4xxErrors"
  namespace           = "AWS/S3"
  period              = "300"
  statistic           = "Sum"
  threshold           = "5"
  alarm_description   = "This metric monitors S3 4xx errors for Terraform state bucket"

  dimensions = {
    BucketName = aws_s3_bucket.terraform_state.bucket
  }

  alarm_actions = [aws_sns_topic.terraform_alerts.arn]
}

# DynamoDB lock table monitoring
resource "aws_cloudwatch_metric_alarm" "lock_table_throttles" {
  alarm_name          = "terraform-lock-table-throttles"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "ThrottledRequests"
  namespace           = "AWS/DynamoDB"
  period              = "300"
  statistic           = "Sum"
  threshold           = "0"
  alarm_description   = "This metric monitors DynamoDB throttles for Terraform lock table"

  dimensions = {
    TableName = aws_dynamodb_table.terraform_locks.name
  }

  alarm_actions = [aws_sns_topic.terraform_alerts.arn]
}
```

### AWS Config for State Compliance
```hcl
# Config rule for S3 bucket encryption
resource "aws_config_config_rule" "s3_bucket_server_side_encryption_enabled" {
  name = "s3-bucket-server-side-encryption-enabled"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

# Config rule for S3 bucket public access
resource "aws_config_config_rule" "s3_bucket_public_access_prohibited" {
  name = "s3-bucket-public-access-prohibited"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_ACCESS_PROHIBITED"
  }

  depends_on = [aws_config_configuration_recorder.main]
}
```

## AWS State Operations

### State Management Scripts
```bash
#!/bin/bash
# AWS-specific state management script

# Set AWS profile and region
export AWS_PROFILE=${AWS_PROFILE:-default}
export AWS_REGION=${AWS_REGION:-us-west-2}

# Initialize with S3 backend
terraform init \
  -backend-config="bucket=terraform-state-mycompany-${ENVIRONMENT}" \
  -backend-config="key=aws/environments/${ENVIRONMENT}/terraform.tfstate" \
  -backend-config="region=${AWS_REGION}" \
  -backend-config="dynamodb_table=terraform-state-locks" \
  -backend-config="encrypt=true"

# Verify AWS credentials and permissions
aws sts get-caller-identity
aws s3 ls s3://terraform-state-mycompany-${ENVIRONMENT}/ || {
  echo "Error: Cannot access state bucket"
  exit 1
}

# Run terraform operations
terraform plan -out=tfplan
terraform apply tfplan
```

### State Migration Between Regions
```bash
#!/bin/bash
# Migrate state from one AWS region to another

OLD_REGION="us-west-2"
NEW_REGION="us-east-1"
BUCKET_NAME="terraform-state-mycompany-prod"

# Create new backend configuration
cat > backend_new.tf << EOF
terraform {
  backend "s3" {
    bucket = "${BUCKET_NAME}"
    key    = "aws/environments/prod/terraform.tfstate"
    region = "${NEW_REGION}"
    encrypt = true
    dynamodb_table = "terraform-state-locks"
  }
}
EOF

# Migrate state
terraform init -migrate-state -backend-config="region=${NEW_REGION}"
```

## Best Practices

### AWS State Management Guidelines
- **Use separate S3 buckets** for different environments
- **Enable S3 versioning** for state history
- **Implement KMS encryption** for state files
- **Use DynamoDB** for state locking
- **Configure lifecycle policies** for cost optimization
- **Implement cross-account access** with IAM roles
- **Monitor state operations** with CloudWatch
- **Use AWS Config** for compliance monitoring
- **Implement backup strategies** for critical state files
- **Document state file dependencies** and access patterns

### Security Considerations
- Never store state files in public repositories
- Use IAM roles instead of access keys for CI/CD
- Implement least privilege access to state resources
- Enable CloudTrail logging for state bucket access
- Use separate AWS accounts for different environments
- Regularly audit state file access permissions