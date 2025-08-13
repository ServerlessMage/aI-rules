# Azure Terraform State Management

## Azure Storage Backend Configuration

### Azure Storage Backend with State Locking
```hcl
# backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state-prod"
    storage_account_name = "sttfstateprod"
    container_name       = "tfstate"
    key                  = "azure/environments/prod/terraform.tfstate"
    
    # Azure-specific configurations
    use_msi              = true  # Use Managed Service Identity
    subscription_id      = "12345678-1234-1234-1234-123456789012"
    tenant_id           = "87654321-4321-4321-4321-210987654321"
  }
}
```

### Azure Storage Account Setup for State
```hcl
# Resource group for Terraform state
resource "azurerm_resource_group" "terraform_state" {
  name     = "rg-terraform-state-${var.environment}"
  location = var.location
  
  tags = {
    Purpose     = "terraform-state"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Storage account for Terraform state
resource "azurerm_storage_account" "terraform_state" {
  name                = "sttfstate${var.environment}${random_string.suffix.result}"
  resource_group_name = azurerm_resource_group.terraform_state.name
  location            = azurerm_resource_group.terraform_state.location
  
  account_tier             = "Standard"
  account_replication_type = var.environment == "prod" ? "GRS" : "LRS"
  
  # Security configurations
  min_tls_version                = "TLS1_2"
  allow_nested_items_to_be_public = false
  
  # Enable versioning for state history
  blob_properties {
    versioning_enabled = true
    
    delete_retention_policy {
      days = var.environment == "prod" ? 90 : 30
    }
    
    container_delete_retention_policy {
      days = var.environment == "prod" ? 90 : 30
    }
  }
  
  # Network access restrictions
  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
    ip_rules       = var.allowed_ip_ranges
    
    virtual_network_subnet_ids = var.allowed_subnet_ids
  }
  
  tags = local.common_tags
}

# Container for state files
resource "azurerm_storage_container" "terraform_state" {
  name                  = "tfstate"
  storage_account_name  = azurerm_storage_account.terraform_state.name
  container_access_type = "private"
}

# Enable soft delete for blobs
resource "azurerm_storage_management_policy" "terraform_state" {
  storage_account_id = azurerm_storage_account.terraform_state.id

  rule {
    name    = "state-lifecycle"
    enabled = true
    
    filters {
      prefix_match = ["tfstate/"]
      blob_types   = ["blockBlob"]
    }
    
    actions {
      base_blob {
        tier_to_cool_after_days_since_modification_greater_than    = 30
        tier_to_archive_after_days_since_modification_greater_than = 90
        delete_after_days_since_modification_greater_than          = 365
      }
      
      version {
        delete_after_days_since_creation = 90
      }
    }
  }
}
```

### Customer-Managed Key Encryption
```hcl
# Key Vault for state encryption
resource "azurerm_key_vault" "terraform_state" {
  name                = "kv-tfstate-${var.environment}-${random_string.suffix.result}"
  location            = azurerm_resource_group.terraform_state.location
  resource_group_name = azurerm_resource_group.terraform_state.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
  
  purge_protection_enabled   = var.environment == "prod"
  soft_delete_retention_days = var.environment == "prod" ? 90 : 7
  
  tags = local.common_tags
}

# Key for storage encryption
resource "azurerm_key_vault_key" "terraform_state" {
  name         = "terraform-state-encryption-key"
  key_vault_id = azurerm_key_vault.terraform_state.id
  key_type     = "RSA"
  key_size     = 2048
  
  key_opts = [
    "decrypt", "encrypt", "unwrapKey", "wrapKey"
  ]
  
  tags = local.common_tags
}

# Managed identity for storage encryption
resource "azurerm_user_assigned_identity" "terraform_state" {
  name                = "id-terraform-state-${var.environment}"
  location            = azurerm_resource_group.terraform_state.location
  resource_group_name = azurerm_resource_group.terraform_state.name
  
  tags = local.common_tags
}

# Grant Key Vault access to managed identity
resource "azurerm_key_vault_access_policy" "terraform_state" {
  key_vault_id = azurerm_key_vault.terraform_state.id
  tenant_id    = azurerm_user_assigned_identity.terraform_state.tenant_id
  object_id    = azurerm_user_assigned_identity.terraform_state.principal_id
  
  key_permissions = [
    "Get", "WrapKey", "UnwrapKey"
  ]
}

# Configure customer-managed key encryption
resource "azurerm_storage_account_customer_managed_key" "terraform_state" {
  storage_account_id = azurerm_storage_account.terraform_state.id
  key_vault_key_id   = azurerm_key_vault_key.terraform_state.id
  user_assigned_identity_id = azurerm_user_assigned_identity.terraform_state.id
}
```

## Azure Multi-Subscription State Organization

### Subscription-Based State Structure
```
Storage Account: sttfstateprod
Container: tfstate
├── azure/
│   ├── subscriptions/
│   │   ├── dev-subscription/
│   │   │   ├── networking/terraform.tfstate
│   │   │   ├── security/terraform.tfstate
│   │   │   └── compute/terraform.tfstate
│   │   ├── staging-subscription/
│   │   └── prod-subscription/
│   ├── shared/
│   │   ├── management-groups/terraform.tfstate
│   │   ├── dns/terraform.tfstate
│   │   └── monitoring/terraform.tfstate
│   └── regions/
│       ├── eastus/
│       └── westus2/
```

### Cross-Subscription State Access
```hcl
# Reference state from different Azure subscription
data "terraform_remote_state" "shared_networking" {
  backend = "azurerm"
  config = {
    resource_group_name  = "rg-terraform-state-shared"
    storage_account_name = "sttfstateshared"
    container_name       = "tfstate"
    key                  = "azure/shared/networking/terraform.tfstate"
    subscription_id      = var.shared_subscription_id
  }
}

# Use cross-subscription VNet peering
resource "azurerm_virtual_network_peering" "cross_subscription" {
  name                = "peer-to-shared-vnet"
  resource_group_name = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  remote_virtual_network_id = data.terraform_remote_state.shared_networking.outputs.vnet_id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways         = false
}
```

## Azure AD Integration for State Access

### Service Principal for Terraform
```hcl
# Azure AD application for Terraform
resource "azuread_application" "terraform" {
  display_name = "terraform-${var.environment}"
  owners       = [data.azurerm_client_config.current.object_id]
}

resource "azuread_service_principal" "terraform" {
  application_id = azuread_application.terraform.application_id
  owners         = [data.azurerm_client_config.current.object_id]
}

# Service principal password
resource "azuread_service_principal_password" "terraform" {
  service_principal_id = azuread_service_principal.terraform.object_id
  end_date            = timeadd(timestamp(), "8760h") # 1 year
}

# Store service principal credentials in Key Vault
resource "azurerm_key_vault_secret" "terraform_client_id" {
  name         = "terraform-client-id"
  value        = azuread_application.terraform.application_id
  key_vault_id = azurerm_key_vault.terraform_state.id
  
  tags = local.common_tags
}

resource "azurerm_key_vault_secret" "terraform_client_secret" {
  name         = "terraform-client-secret"
  value        = azuread_service_principal_password.terraform.value
  key_vault_id = azurerm_key_vault.terraform_state.id
  
  tags = local.common_tags
}
```

### RBAC for State Access
```hcl
# Custom role for Terraform state management
resource "azurerm_role_definition" "terraform_state" {
  name  = "Terraform State Manager"
  scope = azurerm_resource_group.terraform_state.id
  
  description = "Custom role for Terraform state management"
  
  permissions {
    actions = [
      "Microsoft.Storage/storageAccounts/blobServices/containers/read",
      "Microsoft.Storage/storageAccounts/blobServices/containers/write",
      "Microsoft.Storage/storageAccounts/blobServices/containers/delete",
      "Microsoft.Storage/storageAccounts/blobServices/generateUserDelegationKey/action"
    ]
    
    data_actions = [
      "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read",
      "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/write",
      "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/delete",
      "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/add/action"
    ]
  }
  
  assignable_scopes = [
    azurerm_resource_group.terraform_state.id
  ]
}

# Assign role to service principal
resource "azurerm_role_assignment" "terraform_state" {
  scope              = azurerm_storage_account.terraform_state.id
  role_definition_id = azurerm_role_definition.terraform_state.role_definition_resource_id
  principal_id       = azuread_service_principal.terraform.object_id
}

# Storage Blob Data Contributor for state access
resource "azurerm_role_assignment" "terraform_blob_contributor" {
  scope                = azurerm_storage_account.terraform_state.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azuread_service_principal.terraform.object_id
}
```

## Azure State Security

### Private Endpoints for State Storage
```hcl
# Private endpoint for storage account
resource "azurerm_private_endpoint" "terraform_state" {
  name                = "pe-terraform-state-${var.environment}"
  location            = azurerm_resource_group.terraform_state.location
  resource_group_name = azurerm_resource_group.terraform_state.name
  subnet_id           = var.private_endpoint_subnet_id
  
  private_service_connection {
    name                           = "psc-terraform-state"
    private_connection_resource_id = azurerm_storage_account.terraform_state.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name                 = "pdz-group-terraform-state"
    private_dns_zone_ids = [azurerm_private_dns_zone.blob.id]
  }
  
  tags = local.common_tags
}

# Private DNS zone for blob storage
resource "azurerm_private_dns_zone" "blob" {
  name                = "privatelink.blob.core.windows.net"
  resource_group_name = azurerm_resource_group.terraform_state.name
  
  tags = local.common_tags
}

# Link private DNS zone to VNet
resource "azurerm_private_dns_zone_virtual_network_link" "blob" {
  name                  = "pdz-link-blob"
  resource_group_name   = azurerm_resource_group.terraform_state.name
  private_dns_zone_name = azurerm_private_dns_zone.blob.name
  virtual_network_id    = var.vnet_id
  
  tags = local.common_tags
}
```

### Azure Policy for State Compliance
```hcl
# Policy to require encryption for storage accounts
resource "azurerm_policy_definition" "require_storage_encryption" {
  name         = "require-storage-encryption"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Require Storage Account Encryption"
  description  = "This policy ensures that storage accounts have encryption enabled"
  
  policy_rule = jsonencode({
    if = {
      allOf = [
        {
          field  = "type"
          equals = "Microsoft.Storage/storageAccounts"
        },
        {
          field  = "Microsoft.Storage/storageAccounts/encryption.services.blob.enabled"
          equals = "false"
        }
      ]
    }
    then = {
      effect = "deny"
    }
  })
}

# Assign policy to resource group
resource "azurerm_resource_group_policy_assignment" "require_encryption" {
  name                 = "require-storage-encryption"
  resource_group_id    = azurerm_resource_group.terraform_state.id
  policy_definition_id = azurerm_policy_definition.require_storage_encryption.id
  display_name         = "Require Storage Encryption"
}
```

## Azure State Monitoring

### Azure Monitor Integration
```hcl
# Log Analytics workspace for monitoring
resource "azurerm_log_analytics_workspace" "terraform_state" {
  name                = "log-terraform-state-${var.environment}"
  location            = azurerm_resource_group.terraform_state.location
  resource_group_name = azurerm_resource_group.terraform_state.name
  sku                 = "PerGB2018"
  retention_in_days   = var.environment == "prod" ? 90 : 30
  
  tags = local.common_tags
}

# Diagnostic settings for storage account
resource "azurerm_monitor_diagnostic_setting" "terraform_state" {
  name               = "terraform-state-diagnostics"
  target_resource_id = azurerm_storage_account.terraform_state.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.terraform_state.id
  
  enabled_log {
    category = "StorageRead"
  }
  
  enabled_log {
    category = "StorageWrite"
  }
  
  enabled_log {
    category = "StorageDelete"
  }
  
  metric {
    category = "Transaction"
    enabled  = true
  }
  
  metric {
    category = "Capacity"
    enabled  = true
  }
}

# Alert for unusual state access
resource "azurerm_monitor_metric_alert" "state_access_alert" {
  name                = "terraform-state-unusual-access"
  resource_group_name = azurerm_resource_group.terraform_state.name
  scopes              = [azurerm_storage_account.terraform_state.id]
  description         = "Alert when Terraform state has unusual access patterns"
  
  criteria {
    metric_namespace = "Microsoft.Storage/storageAccounts"
    metric_name      = "Transactions"
    aggregation      = "Total"
    operator         = "GreaterThan"
    threshold        = 100
    
    dimension {
      name     = "ResponseType"
      operator = "Include"
      values   = ["Success"]
    }
  }
  
  action {
    action_group_id = azurerm_monitor_action_group.terraform_alerts.id
  }
  
  tags = local.common_tags
}
```

## Azure State Operations

### State Management with Azure CLI
```bash
#!/bin/bash
# Azure-specific state management script

# Set Azure subscription and resource group
export ARM_SUBSCRIPTION_ID=${ARM_SUBSCRIPTION_ID}
export ARM_TENANT_ID=${ARM_TENANT_ID}
export ARM_CLIENT_ID=${ARM_CLIENT_ID}
export ARM_CLIENT_SECRET=${ARM_CLIENT_SECRET}

# Login to Azure (for interactive use)
az login --service-principal \
  --username $ARM_CLIENT_ID \
  --password $ARM_CLIENT_SECRET \
  --tenant $ARM_TENANT_ID

# Initialize with Azure backend
terraform init \
  -backend-config="resource_group_name=rg-terraform-state-${ENVIRONMENT}" \
  -backend-config="storage_account_name=sttfstate${ENVIRONMENT}" \
  -backend-config="container_name=tfstate" \
  -backend-config="key=azure/environments/${ENVIRONMENT}/terraform.tfstate"

# Verify Azure access
az account show
az storage account show \
  --name "sttfstate${ENVIRONMENT}" \
  --resource-group "rg-terraform-state-${ENVIRONMENT}" || {
  echo "Error: Cannot access state storage account"
  exit 1
}

# Run terraform operations
terraform plan -out=tfplan
terraform apply tfplan
```

### State Migration Between Regions
```bash
#!/bin/bash
# Migrate state from one Azure region to another

OLD_REGION="East US"
NEW_REGION="West US 2"
RESOURCE_GROUP="rg-terraform-state-prod"

# Create new storage account in target region
az storage account create \
  --name "sttfstateprod${NEW_REGION//[^a-zA-Z0-9]/}" \
  --resource-group $RESOURCE_GROUP \
  --location "$NEW_REGION" \
  --sku Standard_GRS

# Copy state files to new storage account
az storage blob copy start-batch \
  --source-account-name "sttfstateprod${OLD_REGION//[^a-zA-Z0-9]/}" \
  --source-container tfstate \
  --destination-account-name "sttfstateprod${NEW_REGION//[^a-zA-Z0-9]/}" \
  --destination-container tfstate

# Update backend configuration
cat > backend_new.tf << EOF
terraform {
  backend "azurerm" {
    resource_group_name  = "${RESOURCE_GROUP}"
    storage_account_name = "sttfstateprod${NEW_REGION//[^a-zA-Z0-9]/}"
    container_name       = "tfstate"
    key                  = "azure/environments/prod/terraform.tfstate"
  }
}
EOF

# Migrate state
terraform init -migrate-state
```

## Best Practices

### Azure State Management Guidelines
- **Use separate storage accounts** for different environments
- **Enable blob versioning** for state history
- **Implement customer-managed keys** for encryption
- **Use private endpoints** for secure access
- **Configure network access restrictions** on storage accounts
- **Implement RBAC** with least privilege principles
- **Use managed identities** instead of service principals when possible
- **Monitor state operations** with Azure Monitor
- **Use Azure Policy** for compliance enforcement
- **Implement backup strategies** for critical state files

### Security Considerations
- Never store state files in public containers
- Use managed identities or service principals for authentication
- Implement network restrictions on storage accounts
- Enable soft delete and versioning for state recovery
- Use separate Azure subscriptions for different environments
- Regularly audit state file access permissions
- Enable diagnostic logging for state storage accounts