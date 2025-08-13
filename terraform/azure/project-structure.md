# Azure Terraform Project Structure

## Standard Directory Layout

```
terraform/azure/
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
│   ├── resource-group/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── README.md
│   ├── virtual-network/
│   ├── virtual-machine/
│   └── storage-account/
├── shared/
│   ├── data.tf
│   ├── locals.tf
│   └── providers.tf
├── .terraform-version
├── .gitignore
└── README.md
```

## Azure-Specific File Organization

### Provider Configuration
```hcl
# providers.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.39.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.4"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
    
    key_vault {
      purge_soft_delete_on_destroy    = true
      recover_soft_deleted_key_vaults = true
    }
    
    virtual_machine {
      delete_os_disk_on_deletion     = true
      graceful_shutdown              = false
      skip_shutdown_and_force_delete = false
    }
  }
  
  subscription_id = var.azure_subscription_id
  tenant_id       = var.azure_tenant_id
}
```

### Azure Data Sources
```hcl
# data.tf
data "azurerm_client_config" "current" {}
data "azurerm_subscription" "current" {}

data "azurerm_resource_group" "existing" {
  count = var.use_existing_resource_group ? 1 : 0
  name  = var.existing_resource_group_name
}

# Get available VM sizes for the region
data "azurerm_virtual_machine_sizes" "available" {
  location = var.location
}
```

## Azure Resource Naming Conventions

### Resource Group Structure
```hcl
# Resource Group - Foundation of Azure resources
resource "azurerm_resource_group" "main" {
  name     = "rg-${var.project_name}-${var.environment}-${var.location_short}"
  location = var.location
  
  tags = local.common_tags
}

# Example: rg-myapp-prod-eus (Resource Group - MyApp - Production - East US)
```

### Virtual Network Naming
```hcl
resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.project_name}-${var.environment}-${var.location_short}"
  address_space       = [var.vnet_address_space]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  
  tags = local.common_tags
}

resource "azurerm_subnet" "public" {
  count = length(var.public_subnet_prefixes)
  
  name                 = "snet-${var.project_name}-${var.environment}-public-${count.index + 1}"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.public_subnet_prefixes[count.index]]
}

resource "azurerm_subnet" "private" {
  count = length(var.private_subnet_prefixes)
  
  name                 = "snet-${var.project_name}-${var.environment}-private-${count.index + 1}"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.private_subnet_prefixes[count.index]]
}
```

### Network Security Group Naming
```hcl
resource "azurerm_network_security_group" "web" {
  name                = "nsg-${var.project_name}-${var.environment}-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  
  security_rule {
    name                       = "AllowHTTP"
    priority                   = 1001
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  
  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 1002
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  
  tags = local.common_tags
}
```

## Azure Subscription Structure

### Multi-Subscription Setup
```hcl
# Subscription-specific configurations
locals {
  subscription_configs = {
    dev = {
      subscription_id = "12345678-1234-1234-1234-123456789012"
      location        = "East US 2"
      location_short  = "eus2"
    }
    staging = {
      subscription_id = "23456789-2345-2345-2345-234567890123"
      location        = "East US 2"
      location_short  = "eus2"
    }
    prod = {
      subscription_id = "34567890-3456-3456-3456-345678901234"
      location        = "East US"
      location_short  = "eus"
    }
  }
  
  current_subscription = local.subscription_configs[var.environment]
}

# Cross-subscription provider
provider "azurerm" {
  alias           = "target_subscription"
  subscription_id = local.current_subscription.subscription_id
  tenant_id       = var.azure_tenant_id
  
  features {}
}
```

### Azure Management Groups Integration
```hcl
# Management group structure
data "azurerm_management_group" "root" {
  name = var.root_management_group_name
}

resource "azurerm_management_group_subscription_association" "main" {
  management_group_id = data.azurerm_management_group.root.id
  subscription_id     = "/subscriptions/${local.current_subscription.subscription_id}"
}
```

## Azure Service Integration

### Common Azure Services Structure
```hcl
# Storage Account
resource "azurerm_storage_account" "main" {
  name                     = "st${var.project_name}${var.environment}${random_string.storage_suffix.result}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = var.environment == "prod" ? "GRS" : "LRS"
  
  blob_properties {
    versioning_enabled = true
    
    delete_retention_policy {
      days = var.environment == "prod" ? 30 : 7
    }
  }
  
  tags = local.common_tags
}

# Azure SQL Database
resource "azurerm_mssql_server" "main" {
  name                         = "sql-${var.project_name}-${var.environment}-${var.location_short}"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = var.sql_admin_username
  administrator_login_password = var.sql_admin_password
  
  tags = local.common_tags
}

resource "azurerm_mssql_database" "main" {
  name           = "sqldb-${var.project_name}-${var.environment}"
  server_id      = azurerm_mssql_server.main.id
  collation      = "SQL_Latin1_General_CP1_CI_AS"
  license_type   = "LicenseIncluded"
  max_size_gb    = var.database_max_size_gb
  sku_name       = var.database_sku_name
  zone_redundant = var.environment == "prod"
  
  tags = local.common_tags
}

# Virtual Machine
resource "azurerm_linux_virtual_machine" "web" {
  name                = "vm-${var.project_name}-${var.environment}-web"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = var.vm_size
  admin_username      = var.admin_username
  
  disable_password_authentication = true
  
  network_interface_ids = [
    azurerm_network_interface.web.id,
  ]
  
  admin_ssh_key {
    username   = var.admin_username
    public_key = file(var.public_key_path)
  }
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts-gen2"
    version   = "latest"
  }
  
  tags = local.common_tags
}
```

## Azure Tagging Strategy

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
    
    # Azure-specific tags
    "Application"     = var.application_name
    "BusinessUnit"    = var.business_unit
    "DataClass"       = var.data_classification
    "Criticality"     = var.criticality_level
  }
  
  # Environment-specific tag overrides
  environment_tags = var.environment == "prod" ? {
    "backup-policy"    = "daily"
    "monitoring-level" = "enhanced"
    "sla-tier"        = "premium"
  } : {
    "backup-policy"    = "weekly"
    "monitoring-level" = "basic"
    "sla-tier"        = "standard"
  }
  
  final_tags = merge(local.common_tags, local.environment_tags)
}
```

## Azure Naming Convention Reference

### Resource Abbreviations
```hcl
locals {
  # Azure resource type abbreviations (Microsoft recommended)
  resource_abbreviations = {
    resource_group              = "rg"
    virtual_network            = "vnet"
    subnet                     = "snet"
    network_security_group     = "nsg"
    virtual_machine            = "vm"
    storage_account            = "st"
    key_vault                  = "kv"
    app_service                = "app"
    sql_server                 = "sql"
    sql_database               = "sqldb"
    cosmos_db                  = "cosmos"
    application_gateway        = "agw"
    load_balancer              = "lb"
    public_ip                  = "pip"
    network_interface          = "nic"
  }
  
  # Location abbreviations
  location_abbreviations = {
    "East US"        = "eus"
    "East US 2"      = "eus2"
    "West US"        = "wus"
    "West US 2"      = "wus2"
    "Central US"     = "cus"
    "North Europe"   = "neu"
    "West Europe"    = "weu"
    "Southeast Asia" = "sea"
  }
}
```

## Best Practices

### Azure-Specific Guidelines
- **Use Resource Groups** as the primary organizational unit
- **Implement Azure Policy** for governance and compliance
- **Use Azure Key Vault** for secrets management
- **Enable Azure Monitor** and Log Analytics
- **Implement Azure Backup** for data protection
- **Use Azure Active Directory** for identity management
- **Follow Azure Well-Architected Framework** principles
- **Use Azure Resource Manager templates** alongside Terraform when needed

### Resource Organization
- Group related resources in the same Resource Group
- Use consistent naming with approved abbreviations
- Implement proper Network Security Group hierarchies
- Use Azure Resource Tags for cost allocation and management
- Follow Azure subscription and management group structure
- Use Azure Blueprints for repeatable deployments