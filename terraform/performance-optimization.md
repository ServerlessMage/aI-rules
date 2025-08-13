# Terraform Performance Optimization

## State Management Performance

### State File Optimization
```hcl
# Split large state files into smaller, focused ones
# Instead of one monolithic state file:
# ❌ BAD: Everything in one state
terraform/
└── main.tf  # 500+ resources

# ✅ GOOD: Logical separation
terraform/
├── networking/     # VPC, subnets, routing
├── security/       # IAM, security groups, KMS
├── compute/        # EC2, ASG, ALB
└── data/          # RDS, ElastiCache, S3
```

### Remote State Performance
```hcl
# Use S3 with proper configuration for performance
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "networking/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    
    # Performance optimizations
    max_retries         = 5
    skip_region_validation = true
    skip_credentials_validation = true
    skip_metadata_api_check = true
  }
}
```

## Resource Creation Optimization

### Parallel Resource Creation
```hcl
# Terraform automatically parallelizes independent resources
# Ensure resources don't have unnecessary dependencies

# ❌ BAD: Unnecessary dependency chain
resource "aws_security_group" "web" {
  # Configuration
}

resource "aws_security_group" "app" {
  # This doesn't need to depend on web SG
  depends_on = [aws_security_group.web]
}

# ✅ GOOD: Independent resources
resource "aws_security_group" "web" {
  # Configuration
}

resource "aws_security_group" "app" {
  # No unnecessary dependencies
}
```

### Efficient Resource Loops
```hcl
# Use for_each instead of count for better performance
# ❌ BAD: Using count
resource "aws_instance" "web" {
  count         = length(var.instance_configs)
  ami           = var.ami_id
  instance_type = var.instance_configs[count.index].type
  
  tags = {
    Name = "web-${count.index}"
  }
}

# ✅ GOOD: Using for_each
resource "aws_instance" "web" {
  for_each = var.instance_configs
  
  ami           = var.ami_id
  instance_type = each.value.type
  
  tags = {
    Name = "web-${each.key}"
  }
}

# Variable definition
variable "instance_configs" {
  type = map(object({
    type = string
    az   = string
  }))
  
  default = {
    web1 = { type = "t3.micro", az = "us-west-2a" }
    web2 = { type = "t3.micro", az = "us-west-2b" }
  }
}
```

### Data Source Optimization
```hcl
# Cache data source results using locals
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
}

# Use locals to avoid repeated data source calls
locals {
  availability_zones = data.aws_availability_zones.available.names
  ami_id            = data.aws_ami.amazon_linux.id
  
  # Compute subnet configurations once
  subnet_configs = {
    for i, az in local.availability_zones : "subnet-${i}" => {
      cidr_block        = cidrsubnet(var.vpc_cidr, 8, i)
      availability_zone = az
    }
  }
}

# Reference locals instead of data sources
resource "aws_subnet" "private" {
  for_each = local.subnet_configs
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.availability_zone
}
```

## Plan and Apply Optimization

### Targeted Operations
```bash
# Target specific resources for faster operations
terraform plan -target=aws_instance.web
terraform apply -target=module.vpc

# Target multiple resources
terraform apply -target=aws_instance.web -target=aws_security_group.web

# Use refresh=false when state is known to be current
terraform plan -refresh=false
terraform apply -refresh=false
```

### Parallelism Configuration
```bash
# Increase parallelism for faster operations (default is 10)
terraform apply -parallelism=20

# Set via environment variable
export TF_CLI_ARGS_apply="-parallelism=15"
export TF_CLI_ARGS_plan="-parallelism=15"

# In terraform configuration
terraform {
  # This doesn't exist - parallelism is CLI-only
  # Just showing for reference
}
```

### Plan File Usage
```bash
# Save plan to avoid re-planning
terraform plan -out=tfplan
terraform apply tfplan

# For CI/CD pipelines
terraform plan -out=tfplan -detailed-exitcode
if [ $? -eq 2 ]; then
  terraform apply tfplan
fi
```

## Module Performance

### Module Composition
```hcl
# ❌ BAD: Monolithic module
module "everything" {
  source = "./modules/monolith"
  
  # 100+ variables
  vpc_cidr = "10.0.0.0/16"
  # ... many more variables
}

# ✅ GOOD: Focused modules
module "vpc" {
  source = "./modules/vpc"
  
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = local.availability_zones
}

module "security_groups" {
  source = "./modules/security-groups"
  
  vpc_id      = module.vpc.vpc_id
  environment = var.environment
}

module "compute" {
  source = "./modules/compute"
  
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  security_group_ids = module.security_groups.app_security_group_ids
}
```

### Module Caching
```hcl
# Use module registry for caching
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  # Configuration
}

# Local modules with version pinning
module "custom_vpc" {
  source = "git::https://github.com/company/terraform-modules.git//vpc?ref=v1.2.0"
  
  # Configuration
}
```

## Provider Performance

### Provider Configuration
```hcl
# Optimize provider configuration
provider "aws" {
  region = var.region
  
  # Skip unnecessary validations for performance
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_region_validation      = true
  
  # Increase retry attempts for reliability
  max_retries = 10
}
```

### Multiple Provider Instances
```hcl
# Use provider aliases efficiently
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west_2"
  region = "us-west-2"
}

# Group resources by provider to minimize context switching
resource "aws_s3_bucket" "east_bucket" {
  provider = aws.us_east_1
  bucket   = "my-east-bucket"
}

resource "aws_s3_bucket" "west_bucket" {
  provider = aws.us_west_2
  bucket   = "my-west-bucket"
}
```

## Large-Scale Deployments

### Workspace Management
```bash
# Use workspaces for environment isolation
terraform workspace new prod
terraform workspace new staging
terraform workspace new dev

# Switch between workspaces
terraform workspace select prod

# List workspaces
terraform workspace list
```

### State Splitting Strategies
```hcl
# Split by lifecycle
terraform/
├── foundation/     # Long-lived infrastructure (VPC, IAM)
├── platform/       # Medium-lived (EKS, RDS)
└── applications/   # Short-lived (deployments, configs)

# Split by team ownership
terraform/
├── networking/     # Network team
├── security/       # Security team
├── platform/       # Platform team
└── applications/   # Application teams
```

### Automation Optimization
```bash
# CI/CD pipeline optimization
# Use terraform fmt -check instead of fmt -diff
terraform fmt -check -recursive

# Use terraform validate with minimal output
terraform validate -json

# Cache .terraform directory in CI
cache:
  paths:
    - .terraform/
    - .terraform.lock.hcl
```

## Monitoring and Debugging

### Performance Monitoring
```bash
# Enable debug logging for performance analysis
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log

# Analyze timing in logs
grep "timing" terraform.log

# Use time command for overall timing
time terraform apply
```

### Resource Graph Analysis
```bash
# Generate dependency graph
terraform graph | dot -Tpng > graph.png

# Analyze for unnecessary dependencies
terraform graph -type=plan | grep -E "(depends_on|implicit)"
```

### Memory Usage Optimization
```bash
# Monitor memory usage during operations
# Use system monitoring tools
top -p $(pgrep terraform)

# For large states, consider splitting
# Monitor state file size
ls -lh terraform.tfstate
```

## Best Practices

### Performance Guidelines
- **Split large state files** into logical components
- **Use for_each instead of count** for better resource management
- **Minimize data source calls** by caching results in locals
- **Use targeted operations** when working with specific resources
- **Implement proper module boundaries** to avoid unnecessary dependencies
- **Cache provider configurations** and avoid repeated validations
- **Use workspaces** for environment isolation
- **Monitor state file size** and split when necessary

### Optimization Checklist
- [ ] State files are logically separated and manageable in size
- [ ] Resources use for_each instead of count where appropriate
- [ ] Data sources are cached in locals when used multiple times
- [ ] Modules have clear boundaries and minimal dependencies
- [ ] Provider configurations are optimized for performance
- [ ] CI/CD pipelines use plan files and caching
- [ ] Resource dependencies are explicit and minimal
- [ ] Large deployments are split into phases

### Troubleshooting Performance Issues
- Use `TF_LOG=DEBUG` to identify bottlenecks
- Analyze resource dependency graphs
- Monitor API rate limits and throttling
- Check network connectivity and latency
- Review state file size and complexity
- Identify unnecessary resource dependencies
- Consider provider-specific performance optimizations