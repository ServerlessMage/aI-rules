# Terraform Testing Strategies

## Testing Pyramid

### Unit Tests (Module Level)
```hcl
# Example module structure for testing
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
├── examples/
│   ├── basic/
│   │   ├── main.tf
│   │   └── outputs.tf
│   └── complete/
│       ├── main.tf
│       └── outputs.tf
└── tests/
    ├── vpc_basic_test.go
    └── vpc_complete_test.go
```

### Integration Tests (Environment Level)
```hcl
# Test complete environment deployments
environments/
├── dev/
│   ├── main.tf
│   └── test/
│       └── integration_test.go
├── staging/
└── prod/
```

## Terratest Framework

### Basic Test Structure
```go
// tests/vpc_basic_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCBasic(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/basic",
        Vars: map[string]interface{}{
            "vpc_cidr": "10.0.0.0/16",
            "environment": "test",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Validate outputs
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)

    vpcCidr := terraform.Output(t, terraformOptions, "vpc_cidr_block")
    assert.Equal(t, "10.0.0.0/16", vpcCidr)
}
```

### Advanced Testing Patterns
```go
// Test with AWS SDK validation
func TestVPCWithAWSValidation(t *testing.T) {
    t.Parallel()

    awsRegion := "us-west-2"
    
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/complete",
        Vars: map[string]interface{}{
            "region": awsRegion,
            "vpc_cidr": "10.0.0.0/16",
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Get VPC ID from Terraform output
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")

    // Validate with AWS SDK
    awsSession := aws.NewSession(&aws.Config{Region: aws.String(awsRegion)})
    ec2Client := ec2.New(awsSession)

    vpc := aws.GetVpcById(t, ec2Client, vpcId)
    assert.Equal(t, "10.0.0.0/16", *vpc.CidrBlock)
    assert.Equal(t, "available", *vpc.State)
}
```

## Static Analysis

### Terraform Validate
```bash
# Basic syntax validation
terraform validate

# Validate all configurations in directory
find . -name "*.tf" -exec dirname {} \; | sort -u | xargs -I {} terraform -chdir={} validate
```

### TFSec Security Scanning
```bash
# Install tfsec
brew install tfsec

# Run security scan
tfsec .

# Generate report
tfsec --format json --out tfsec-report.json .

# Custom checks
tfsec --config-file .tfsec.yml .
```

```yaml
# .tfsec.yml
severity_overrides:
  aws-s3-enable-bucket-encryption: ERROR
  aws-s3-enable-bucket-logging: WARNING

exclude:
  - aws-s3-enable-versioning  # Exclude specific checks
```

### Checkov Policy Scanning
```bash
# Install checkov
pip install checkov

# Run policy scan
checkov -d .

# Generate report
checkov -d . --output json --output-file checkov-report.json

# Custom policies
checkov -d . --external-checks-dir ./custom-policies
```

### TFLint Code Quality
```bash
# Install tflint
brew install tflint

# Initialize with plugins
tflint --init

# Run linting
tflint

# Configuration file
```

```hcl
# .tflint.hcl
plugin "aws" {
  enabled = true
  version = "0.21.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_deprecated_interpolation" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}

rule "terraform_comment_syntax" {
  enabled = true
}
```

## Test Examples and Fixtures

### Test Data Management
```hcl
# tests/fixtures/variables.tf
variable "test_name" {
  description = "Name for test resources"
  type        = string
  default     = "terratest"
}

variable "test_environment" {
  description = "Test environment identifier"
  type        = string
  default     = "test"
}

locals {
  test_tags = {
    Environment = var.test_environment
    TestName    = var.test_name
    ManagedBy   = "terratest"
    CreatedAt   = timestamp()
  }
}
```

### Reusable Test Modules
```hcl
# tests/fixtures/vpc/main.tf
module "test_vpc" {
  source = "../../../modules/vpc"
  
  vpc_cidr    = var.vpc_cidr
  environment = var.test_environment
  
  availability_zones = data.aws_availability_zones.available.names
  
  tags = local.test_tags
}

data "aws_availability_zones" "available" {
  state = "available"
}

output "vpc_id" {
  value = module.test_vpc.vpc_id
}

output "private_subnet_ids" {
  value = module.test_vpc.private_subnet_ids
}
```

## Continuous Integration

### GitHub Actions Workflow
```yaml
# .github/workflows/terraform-test.yml
name: Terraform Tests

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'
      - 'tests/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
      
      - name: Terraform Validate
        run: |
          find . -name "*.tf" -exec dirname {} \; | sort -u | while read dir; do
            echo "Validating $dir"
            terraform -chdir="$dir" init -backend=false
            terraform -chdir="$dir" validate
          done

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: true
      
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          soft_fail: true

  test:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Go
        uses: actions/setup-go@v3
        with:
          go-version: 1.19
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-west-2
      
      - name: Run Terratest
        run: |
          cd tests
          go mod download
          go test -v -timeout 30m
```

### Pre-commit Hooks
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.81.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
      - id: terraform_tflint
      - id: terraform_tfsec
      - id: terraform_checkov
```

## Test Organization

### Test Structure Best Practices
```
tests/
├── unit/                    # Module-level tests
│   ├── vpc_test.go
│   ├── security_group_test.go
│   └── fixtures/
│       ├── vpc/
│       └── security_group/
├── integration/             # Environment-level tests
│   ├── dev_environment_test.go
│   └── fixtures/
│       └── dev/
├── e2e/                     # End-to-end tests
│   └── full_stack_test.go
└── helpers/                 # Test utilities
    ├── aws.go
    └── terraform.go
```

### Test Utilities
```go
// tests/helpers/terraform.go
package helpers

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
)

func GetTerraformOptions(t *testing.T, terraformDir string, vars map[string]interface{}) *terraform.Options {
    return terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: terraformDir,
        Vars:         vars,
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "us-west-2",
        },
    })
}

func ApplyAndValidateBasicOutputs(t *testing.T, options *terraform.Options) {
    defer terraform.Destroy(t, options)
    terraform.InitAndApply(t, options)
    
    // Common validations
    assert.NotEmpty(t, terraform.Output(t, options, "vpc_id"))
    assert.NotEmpty(t, terraform.Output(t, options, "vpc_cidr_block"))
}
```

## Performance Testing

### Load Testing Infrastructure
```go
func TestInfrastructureUnderLoad(t *testing.T) {
    // Deploy infrastructure
    terraformOptions := GetTerraformOptions(t, "../examples/load-test", map[string]interface{}{
        "instance_count": 10,
        "instance_type":  "t3.medium",
    })
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    // Get load balancer URL
    lbUrl := terraform.Output(t, terraformOptions, "load_balancer_url")
    
    // Run load test
    loadtest.RunLoadTest(t, lbUrl, loadtest.Options{
        Duration:     time.Minute * 5,
        RequestsPerSecond: 100,
        MaxResponseTime:   time.Second * 2,
    })
}
```

## Best Practices

### Testing Guidelines
- **Test at multiple levels** - Unit, integration, and end-to-end
- **Use parallel execution** - Speed up test runs with `t.Parallel()`
- **Clean up resources** - Always use `defer terraform.Destroy()`
- **Use unique naming** - Avoid resource conflicts with random suffixes
- **Test failure scenarios** - Include negative test cases
- **Validate with cloud APIs** - Don't just test Terraform outputs

### Test Data Management
- Use fixtures for reusable test configurations
- Generate unique resource names to avoid conflicts
- Use separate AWS accounts for testing when possible
- Clean up resources even if tests fail

### CI/CD Integration
- Run tests on every pull request
- Use different test suites for different triggers
- Cache dependencies to speed up builds
- Set appropriate timeouts for long-running tests
- Use matrix builds for multiple Terraform versions