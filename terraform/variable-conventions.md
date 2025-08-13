# Terraform Variable Conventions

## Variable Declaration Standards

### Basic Variable Structure
```hcl
variable "variable_name" {
  description = "Clear description of what this variable does"
  type        = string
  default     = "default_value"  # Optional
  sensitive   = false           # Optional, default is false
  nullable    = false           # Optional, default is true
  
  validation {
    condition     = length(var.variable_name) > 0
    error_message = "Variable cannot be empty."
  }
}
```

### Naming Conventions
```hcl
# Use snake_case for all variable names
variable "instance_type" {}
variable "vpc_cidr_block" {}
variable "enable_monitoring" {}

# Boolean variables should be prefixed appropriately
variable "enable_encryption" {
  description = "Enable encryption for the resource"
  type        = bool
  default     = true
}

variable "disable_api_termination" {
  description = "Disable API termination for EC2 instances"
  type        = bool
  default     = false
}

# Use descriptive names that indicate purpose
variable "database_subnet_group_name" {
  description = "Name of the database subnet group"
  type        = string
}

# Avoid abbreviations unless universally understood
variable "availability_zones" {}  # Good
variable "azs" {}                 # Avoid
```

## Variable Types

### Primitive Types
```hcl
# String variables
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  
  validation {
    condition = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Number variables
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 1
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

# Boolean variables
variable "enable_detailed_monitoring" {
  description = "Enable detailed CloudWatch monitoring"
  type        = bool
  default     = false
}
```

### Collection Types
```hcl
# List variables
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-west-2a", "us-west-2b", "us-west-2c"]
}

variable "allowed_cidr_blocks" {
  description = "List of CIDR blocks allowed to access the resource"
  type        = list(string)
  default     = []
  
  validation {
    condition = alltrue([
      for cidr in var.allowed_cidr_blocks : can(cidrhost(cidr, 0))
    ])
    error_message = "All CIDR blocks must be valid IPv4 CIDR notation."
  }
}

# Map variables
variable "tags" {
  description = "A map of tags to assign to the resource"
  type        = map(string)
  default     = {}
}

variable "instance_types_by_env" {
  description = "Instance types for different environments"
  type        = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
}

# Set variables (for unique values)
variable "security_group_ids" {
  description = "Set of security group IDs"
  type        = set(string)
  default     = []
}
```

### Object Types
```hcl
# Simple object
variable "database_config" {
  description = "Database configuration"
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    allocated_storage = number
  })
  
  default = {
    engine         = "postgres"
    engine_version = "13.7"
    instance_class = "db.t3.micro"
    allocated_storage = 20
  }
}

# Complex nested object
variable "vpc_config" {
  description = "VPC configuration with subnets"
  type = object({
    cidr_block = string
    enable_dns_hostnames = bool
    enable_dns_support   = bool
    
    public_subnets = list(object({
      cidr_block        = string
      availability_zone = string
    }))
    
    private_subnets = list(object({
      cidr_block        = string
      availability_zone = string
    }))
  })
}

# Optional object attributes (Terraform 1.3+)
variable "server_config" {
  description = "Server configuration with optional attributes"
  type = object({
    name         = string
    instance_type = string
    monitoring   = optional(bool, false)
    backup_retention = optional(number, 7)
    tags         = optional(map(string), {})
  })
}
```

## Variable Validation

### Common Validation Patterns
```hcl
# CIDR block validation
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid IPv4 CIDR block."
  }
}

# Email validation
variable "notification_email" {
  description = "Email address for notifications"
  type        = string
  
  validation {
    condition     = can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", var.notification_email))
    error_message = "Must be a valid email address."
  }
}

# Enum validation
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  
  validation {
    condition = contains([
      "t3.micro", "t3.small", "t3.medium", "t3.large",
      "m5.large", "m5.xlarge", "m5.2xlarge"
    ], var.instance_type)
    error_message = "Instance type must be a supported type."
  }
}

# Range validation
variable "min_size" {
  description = "Minimum number of instances in ASG"
  type        = number
  
  validation {
    condition     = var.min_size >= 1 && var.min_size <= 100
    error_message = "Minimum size must be between 1 and 100."
  }
}

# String length validation
variable "project_name" {
  description = "Name of the project"
  type        = string
  
  validation {
    condition     = length(var.project_name) >= 3 && length(var.project_name) <= 20
    error_message = "Project name must be between 3 and 20 characters."
  }
}

# Complex validation with multiple conditions
variable "database_config" {
  description = "Database configuration"
  type = object({
    engine            = string
    allocated_storage = number
    max_allocated_storage = number
  })
  
  validation {
    condition = contains(["mysql", "postgres", "mariadb"], var.database_config.engine)
    error_message = "Database engine must be mysql, postgres, or mariadb."
  }
  
  validation {
    condition = var.database_config.allocated_storage >= 20
    error_message = "Allocated storage must be at least 20 GB."
  }
  
  validation {
    condition = var.database_config.max_allocated_storage >= var.database_config.allocated_storage
    error_message = "Max allocated storage must be greater than or equal to allocated storage."
  }
}
```

## Variable Organization

### File Structure
```hcl
# variables.tf - Main variable declarations
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project_name" {
  description = "Name of the project"
  type        = string
}

# Group related variables together
# Network variables
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

# Compute variables
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "key_pair_name" {
  description = "Name of the EC2 key pair"
  type        = string
}
```

### Environment-Specific Values
```hcl
# terraform.tfvars (or environment-specific .tfvars files)
environment    = "prod"
project_name   = "my-application"
instance_type  = "t3.medium"
key_pair_name  = "prod-keypair"

availability_zones = [
  "us-west-2a",
  "us-west-2b",
  "us-west-2c"
]

database_config = {
  engine            = "postgres"
  engine_version    = "13.7"
  instance_class    = "db.t3.small"
  allocated_storage = 100
}

tags = {
  Environment = "prod"
  Project     = "my-application"
  Owner       = "platform-team"
  CostCenter  = "engineering"
}
```

## Sensitive Variables

### Handling Sensitive Data
```hcl
# Mark sensitive variables
variable "database_password" {
  description = "Password for the database"
  type        = string
  sensitive   = true
}

variable "api_keys" {
  description = "Map of API keys"
  type        = map(string)
  sensitive   = true
  default     = {}
}

# Use environment variables for sensitive values
# TF_VAR_database_password=secret_password
# TF_VAR_api_keys='{"service1":"key1","service2":"key2"}'
```

### External Secret References
```hcl
# Reference secrets from external systems
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/password"
}

locals {
  db_password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}
```

## Best Practices

### Variable Design Guidelines
- **Always include descriptions** - Make them clear and comprehensive
- **Use appropriate types** - Be specific with object structures
- **Provide sensible defaults** - When appropriate for the use case
- **Implement validation** - Catch errors early with validation rules
- **Group related variables** - Organize logically in files
- **Use consistent naming** - Follow snake_case convention
- **Mark sensitive data** - Use the sensitive flag appropriately

### Documentation Standards
```hcl
variable "complex_config" {
  description = <<-EOT
    Complex configuration object for the application.
    
    Fields:
    - name: Application name (required)
    - version: Application version (required)
    - replicas: Number of replicas (optional, default: 1)
    - resources: Resource limits and requests
      - cpu: CPU limit (e.g., "500m")
      - memory: Memory limit (e.g., "512Mi")
    
    Example:
    {
      name     = "my-app"
      version  = "1.0.0"
      replicas = 3
      resources = {
        cpu    = "1000m"
        memory = "1Gi"
      }
    }
  EOT
  
  type = object({
    name     = string
    version  = string
    replicas = optional(number, 1)
    resources = object({
      cpu    = string
      memory = string
    })
  })
}
```

### Common Patterns
- Use `locals` for computed values based on variables
- Combine user-provided tags with standard tags
- Use conditional expressions for environment-specific values
- Implement cross-variable validation when needed
- Use `try()` function for optional nested attributes