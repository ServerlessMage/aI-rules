# AWS Terraform Project Structure

## Standard Directory Layout

```
terraform/aws/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── README.md
│   ├── ec2/
│   ├── rds/
│   └── s3/
├── shared/
│   ├── data.tf
│   ├── locals.tf
│   └── providers.tf
├── .terraform-version
├── .gitignore
└── README.md
```

## AWS-Specific File Organization

### Provider Configuration
```hcl
# providers.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.8.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.4"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "terraform"
      Owner       = var.owner
    }
  }
}
```

### AWS Data Sources
```hcl
# data.tf
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_partition" "current" {}

data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

## AWS Resource Naming Conventions

### Resource Naming Patterns
```hcl
# VPC Resources
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project_name}-${var.environment}-vpc"
  }
}

# Subnet Naming
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.project_name}-${var.environment}-public-${count.index + 1}"
    Type = "Public"
  }
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "${var.project_name}-${var.environment}-private-${count.index + 1}"
    Type = "Private"
  }
}
```

### Security Group Naming
```hcl
resource "aws_security_group" "web" {
  name_prefix = "${var.project_name}-${var.environment}-web-"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-${var.environment}-web-sg"
  }
}
```

## AWS Account Structure

### Multi-Account Setup
```hcl
# Account-specific configurations
locals {
  account_configs = {
    dev = {
      account_id = "123456789012"
      region     = "us-west-2"
    }
    staging = {
      account_id = "234567890123"
      region     = "us-west-2"
    }
    prod = {
      account_id = "345678901234"
      region     = "us-east-1"
    }
  }
  
  current_account = local.account_configs[var.environment]
}

# Cross-account role assumption
provider "aws" {
  alias  = "target_account"
  region = local.current_account.region
  
  assume_role {
    role_arn = "arn:aws:iam::${local.current_account.account_id}:role/TerraformRole"
  }
}
```

### AWS Organizations Integration
```hcl
# Organization-wide resources
data "aws_organizations_organization" "main" {}

# Account-specific policies
resource "aws_organizations_policy_attachment" "account_policy" {
  policy_id = aws_organizations_policy.security_policy.id
  target_id = local.current_account.account_id
}
```

## AWS Service Integration

### Common AWS Services Structure
```hcl
# S3 Buckets
resource "aws_s3_bucket" "app_data" {
  bucket = "${var.project_name}-${var.environment}-app-data-${random_id.bucket_suffix.hex}"
}

resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  versioning_configuration {
    status = "Enabled"
  }
}

# RDS Instance
resource "aws_db_instance" "main" {
  identifier = "${var.project_name}-${var.environment}-db"
  
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = var.db_instance_class
  
  allocated_storage     = var.db_allocated_storage
  max_allocated_storage = var.db_max_allocated_storage
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.database.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = var.environment == "prod" ? 30 : 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = var.environment != "prod"
  
  tags = {
    Name = "${var.project_name}-${var.environment}-database"
  }
}
```

## AWS Tagging Strategy

### Standard Tags
```hcl
locals {
  common_tags = {
    Environment   = var.environment
    Project       = var.project_name
    Owner         = var.owner
    CostCenter    = var.cost_center
    ManagedBy     = "terraform"
    CreatedDate   = formatdate("YYYY-MM-DD", timestamp())
    
    # AWS-specific tags
    "aws:cloudformation:stack-name" = null  # Exclude CloudFormation
    "backup:schedule"               = var.backup_schedule
    "monitoring:enabled"            = var.monitoring_enabled
  }
  
  # Environment-specific tag overrides
  environment_tags = var.environment == "prod" ? {
    "backup:retention" = "30-days"
    "monitoring:level" = "detailed"
  } : {
    "backup:retention" = "7-days"
    "monitoring:level" = "basic"
  }
  
  final_tags = merge(local.common_tags, local.environment_tags)
}
```

## Best Practices

### AWS-Specific Guidelines
- **Use AWS-native services** for state storage (S3 + DynamoDB)
- **Implement proper IAM roles** instead of access keys
- **Use AWS Organizations** for multi-account management
- **Follow AWS Well-Architected Framework** principles
- **Implement AWS Config** for compliance monitoring
- **Use AWS Secrets Manager** for sensitive data
- **Enable CloudTrail** for audit logging
- **Implement proper VPC design** with public/private subnets

### Resource Organization
- Group resources by AWS service type
- Use consistent naming with environment prefixes
- Implement proper security group hierarchies
- Use AWS resource groups for logical organization
- Follow AWS tagging best practices for cost allocation