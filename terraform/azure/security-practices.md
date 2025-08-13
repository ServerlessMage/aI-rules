# Azure Terraform Security Practices

## Azure Secrets Management

### Azure Key Vault Integration
```hcl
# Key Vault for secrets management
resource "azurerm_key_vault" "main" {
  name                = "kv-${var.project_name}-${var.environment}-${random_string.kv_suffix.result}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
  
  # Enable soft delete and purge protection for production
  soft_delete_retention_days = var.environment == "prod" ? 90 : 7
  purge_protection_enabled   = var.environment == "prod"
  
  # Network access restrictions
  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
    
    ip_rules = var.allowed_ip_ranges
    
    virtual_network_subnet_ids = [
      azurerm_subnet.private[0].id
    ]
  }
  
  tags = local.common_tags
}

# Access policy for current service principal
resource "azurerm_key_vault_access_policy" "terraform" {
  key_vault_id = azurerm_key_vault.main.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = data.azurerm_client_config.current.object_id
  
  secret_permissions = [
    "Get", "List", "Set", "Delete", "Recover", "Backup", "Restore", "Purge"
  ]
  
  key_permissions = [
    "Get", "List", "Create", "Delete", "Recover", "Backup", "Restore", "Purge",
    "Decrypt", "Encrypt", "Sign", "Verify", "WrapKey", "UnwrapKey"
  ]
  
  certificate_permissions = [
    "Get", "List", "Create", "Delete", "Recover", "Backup", "Restore", "Purge"
  ]
}

# Store database password in Key Vault
resource "random_password" "sql_admin_password" {
  length  = 32
  special = true
}

resource "azurerm_key_vault_secret" "sql_admin_password" {
  name         = "sql-admin-password"
  value        = random_password.sql_admin_password.result
  key_vault_id = azurerm_key_vault.main.id
  
  depends_on = [azurerm_key_vault_access_policy.terraform]
  
  tags = local.common_tags
}

# Reference secret in SQL Server
data "azurerm_key_vault_secret" "sql_admin_password" {
  name         = azurerm_key_vault_secret.sql_admin_password.name
  key_vault_id = azurerm_key_vault.main.id
}

resource "azurerm_mssql_server" "main" {
  name                         = "sql-${var.project_name}-${var.environment}-${var.location_short}"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = var.sql_admin_username
  administrator_login_password = data.azurerm_key_vault_secret.sql_admin_password.value
  
  # Enable Azure AD authentication
  azuread_administrator {
    login_username = var.azuread_admin_login
    object_id      = var.azuread_admin_object_id
  }
  
  tags = local.common_tags
}
```

### Managed Identity Integration
```hcl
# User-assigned managed identity
resource "azurerm_user_assigned_identity" "main" {
  name                = "id-${var.project_name}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  
  tags = local.common_tags
}

# Grant Key Vault access to managed identity
resource "azurerm_key_vault_access_policy" "managed_identity" {
  key_vault_id = azurerm_key_vault.main.id
  tenant_id    = azurerm_user_assigned_identity.main.tenant_id
  object_id    = azurerm_user_assigned_identity.main.principal_id
  
  secret_permissions = [
    "Get", "List"
  ]
}

# Virtual Machine with managed identity
resource "azurerm_linux_virtual_machine" "app" {
  name                = "vm-${var.project_name}-${var.environment}-app"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = var.vm_size
  admin_username      = var.admin_username
  
  disable_password_authentication = true
  
  # Assign managed identity
  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.main.id]
  }
  
  network_interface_ids = [azurerm_network_interface.app.id]
  
  admin_ssh_key {
    username   = var.admin_username
    public_key = file(var.public_key_path)
  }
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    
    # Enable disk encryption
    disk_encryption_set_id = azurerm_disk_encryption_set.main.id
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

## Azure Identity and Access Management

### Azure AD Integration
```hcl
# Azure AD application for service authentication
resource "azuread_application" "main" {
  display_name = "${var.project_name}-${var.environment}-app"
  owners       = [data.azurerm_client_config.current.object_id]
  
  required_resource_access {
    resource_app_id = "00000003-0000-0000-c000-000000000000" # Microsoft Graph
    
    resource_access {
      id   = "e1fe6dd8-ba31-4d61-89e7-88639da4683d" # User.Read
      type = "Scope"
    }
  }
}

resource "azuread_service_principal" "main" {
  application_id = azuread_application.main.application_id
  owners         = [data.azurerm_client_config.current.object_id]
}

# Custom role definition
resource "azurerm_role_definition" "app_role" {
  name  = "${var.project_name}-${var.environment}-app-role"
  scope = azurerm_resource_group.main.id
  
  description = "Custom role for ${var.project_name} application"
  
  permissions {
    actions = [
      "Microsoft.Storage/storageAccounts/blobServices/containers/read",
      "Microsoft.Storage/storageAccounts/blobServices/containers/write",
      "Microsoft.KeyVault/vaults/secrets/read"
    ]
    
    not_actions = []
  }
  
  assignable_scopes = [
    azurerm_resource_group.main.id
  ]
}

# Role assignment
resource "azurerm_role_assignment" "app_role" {
  scope              = azurerm_resource_group.main.id
  role_definition_id = azurerm_role_definition.app_role.role_definition_resource_id
  principal_id       = azurerm_user_assigned_identity.main.principal_id
}
```

### RBAC Configuration
```hcl
# Built-in role assignments
resource "azurerm_role_assignment" "storage_blob_data_contributor" {
  scope                = azurerm_storage_account.main.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_user_assigned_identity.main.principal_id
}

resource "azurerm_role_assignment" "key_vault_secrets_user" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.main.principal_id
}

# Conditional access for production
resource "azurerm_role_assignment" "prod_admin" {
  count = var.environment == "prod" ? 1 : 0
  
  scope                = azurerm_resource_group.main.id
  role_definition_name = "Contributor"
  principal_id         = var.prod_admin_group_id
  
  condition         = "((!(ActionMatches{'Microsoft.Authorization/roleAssignments/write'})) OR (@Request[Microsoft.Authorization/roleAssignments:RoleDefinitionId] ForAnyOfAnyValues:GuidEquals {b24988ac-6180-42a0-ab88-20f7382dd24c}))"
  condition_version = "2.0"
}
```

## Azure Encryption

### Disk Encryption
```hcl
# Key Vault for disk encryption
resource "azurerm_key_vault_key" "disk_encryption" {
  name         = "disk-encryption-key"
  key_vault_id = azurerm_key_vault.main.id
  key_type     = "RSA"
  key_size     = 2048
  
  key_opts = [
    "decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"
  ]
  
  depends_on = [azurerm_key_vault_access_policy.terraform]
  
  tags = local.common_tags
}

# Disk encryption set
resource "azurerm_disk_encryption_set" "main" {
  name                = "des-${var.project_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  key_vault_key_id    = azurerm_key_vault_key.disk_encryption.id
  
  identity {
    type = "SystemAssigned"
  }
  
  tags = local.common_tags
}

# Grant disk encryption set access to Key Vault
resource "azurerm_key_vault_access_policy" "disk_encryption" {
  key_vault_id = azurerm_key_vault.main.id
  tenant_id    = azurerm_disk_encryption_set.main.identity[0].tenant_id
  object_id    = azurerm_disk_encryption_set.main.identity[0].principal_id
  
  key_permissions = [
    "Get", "WrapKey", "UnwrapKey"
  ]
}
```

### Storage Encryption
```hcl
# Customer-managed key for storage encryption
resource "azurerm_key_vault_key" "storage_encryption" {
  name         = "storage-encryption-key"
  key_vault_id = azurerm_key_vault.main.id
  key_type     = "RSA"
  key_size     = 2048
  
  key_opts = [
    "decrypt", "encrypt", "unwrapKey", "wrapKey"
  ]
  
  depends_on = [azurerm_key_vault_access_policy.terraform]
  
  tags = local.common_tags
}

resource "azurerm_storage_account" "secure" {
  name                = "st${var.project_name}${var.environment}sec${random_string.storage_suffix.result}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  
  account_tier             = "Standard"
  account_replication_type = var.environment == "prod" ? "GRS" : "LRS"
  
  # Enable infrastructure encryption
  infrastructure_encryption_enabled = true
  
  # Customer-managed key encryption
  customer_managed_key {
    key_vault_key_id          = azurerm_key_vault_key.storage_encryption.id
    user_assigned_identity_id = azurerm_user_assigned_identity.storage.id
  }
  
  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.storage.id]
  }
  
  # Network restrictions
  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
    ip_rules       = var.allowed_ip_ranges
    
    virtual_network_subnet_ids = [
      azurerm_subnet.private[0].id
    ]
  }
  
  tags = local.common_tags
}

# Managed identity for storage encryption
resource "azurerm_user_assigned_identity" "storage" {
  name                = "id-${var.project_name}-${var.environment}-storage"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  
  tags = local.common_tags
}

# Grant storage identity access to Key Vault
resource "azurerm_key_vault_access_policy" "storage" {
  key_vault_id = azurerm_key_vault.main.id
  tenant_id    = azurerm_user_assigned_identity.storage.tenant_id
  object_id    = azurerm_user_assigned_identity.storage.principal_id
  
  key_permissions = [
    "Get", "WrapKey", "UnwrapKey"
  ]
}
```

## Azure Network Security

### Network Security Groups
```hcl
# Web tier NSG
resource "azurerm_network_security_group" "web" {
  name                = "nsg-${var.project_name}-${var.environment}-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  
  # Allow HTTP/HTTPS from Application Gateway only
  security_rule {
    name                         = "AllowApplicationGatewayHTTP"
    priority                     = 1000
    direction                    = "Inbound"
    access                       = "Allow"
    protocol                     = "Tcp"
    source_port_range            = "*"
    destination_port_range       = "80"
    source_address_prefix        = azurerm_subnet.gateway[0].address_prefixes[0]
    destination_address_prefix   = "*"
  }
  
  security_rule {
    name                         = "AllowApplicationGatewayHTTPS"
    priority                     = 1010
    direction                    = "Inbound"
    access                       = "Allow"
    protocol                     = "Tcp"
    source_port_range            = "*"
    destination_port_range       = "443"
    source_address_prefix        = azurerm_subnet.gateway[0].address_prefixes[0]
    destination_address_prefix   = "*"
  }
  
  # Deny all other inbound traffic
  security_rule {
    name                       = "DenyAllInbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  
  tags = local.common_tags
}

# Database tier NSG
resource "azurerm_network_security_group" "database" {
  name                = "nsg-${var.project_name}-${var.environment}-db"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  
  # Allow SQL from app tier only
  security_rule {
    name                         = "AllowSQLFromApp"
    priority                     = 1000
    direction                    = "Inbound"
    access                       = "Allow"
    protocol                     = "Tcp"
    source_port_range            = "*"
    destination_port_range       = "1433"
    source_address_prefix        = azurerm_subnet.private[0].address_prefixes[0]
    destination_address_prefix   = "*"
  }
  
  # Deny all other traffic
  security_rule {
    name                       = "DenyAllInbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  
  tags = local.common_tags
}

# Associate NSGs with subnets
resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.public[0].id
  network_security_group_id = azurerm_network_security_group.web.id
}

resource "azurerm_subnet_network_security_group_association" "database" {
  subnet_id                 = azurerm_subnet.database[0].id
  network_security_group_id = azurerm_network_security_group.database.id
}
```

### Azure Firewall
```hcl
# Azure Firewall for advanced network security
resource "azurerm_subnet" "firewall" {
  name                 = "AzureFirewallSubnet"  # Must be exactly this name
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.firewall_subnet_prefix]
}

resource "azurerm_public_ip" "firewall" {
  name                = "pip-${var.project_name}-${var.environment}-fw"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
  
  tags = local.common_tags
}

resource "azurerm_firewall" "main" {
  name                = "fw-${var.project_name}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Standard"
  
  ip_configuration {
    name                 = "configuration"
    subnet_id            = azurerm_subnet.firewall.id
    public_ip_address_id = azurerm_public_ip.firewall.id
  }
  
  tags = local.common_tags
}

# Firewall rules
resource "azurerm_firewall_application_rule_collection" "web_traffic" {
  name                = "web-traffic"
  azure_firewall_name = azurerm_firewall.main.name
  resource_group_name = azurerm_resource_group.main.name
  priority            = 100
  action              = "Allow"
  
  rule {
    name = "allow-web-traffic"
    
    source_addresses = [
      azurerm_subnet.private[0].address_prefixes[0]
    ]
    
    target_fqdns = [
      "*.microsoft.com",
      "*.ubuntu.com",
      "*.docker.com"
    ]
    
    protocol {
      port = "443"
      type = "Https"
    }
    
    protocol {
      port = "80"
      type = "Http"
    }
  }
}
```

## Azure Compliance and Monitoring

### Azure Policy
```hcl
# Custom policy definition
resource "azurerm_policy_definition" "require_encryption" {
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

# Policy assignment
resource "azurerm_resource_group_policy_assignment" "require_encryption" {
  name                 = "require-encryption"
  resource_group_id    = azurerm_resource_group.main.id
  policy_definition_id = azurerm_policy_definition.require_encryption.id
  display_name         = "Require Storage Encryption"
  description          = "Ensures all storage accounts have encryption enabled"
}
```

### Azure Monitor and Log Analytics
```hcl
# Log Analytics workspace
resource "azurerm_log_analytics_workspace" "main" {
  name                = "log-${var.project_name}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = var.environment == "prod" ? "PerGB2018" : "Free"
  retention_in_days   = var.environment == "prod" ? 90 : 30
  
  tags = local.common_tags
}

# Security Center workspace connection
resource "azurerm_security_center_workspace" "main" {
  scope        = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
  workspace_id = azurerm_log_analytics_workspace.main.id
}

# Diagnostic settings for Key Vault
resource "azurerm_monitor_diagnostic_setting" "key_vault" {
  name               = "key-vault-diagnostics"
  target_resource_id = azurerm_key_vault.main.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  
  enabled_log {
    category = "AuditEvent"
  }
  
  metric {
    category = "AllMetrics"
    enabled  = true
  }
}
```

## Azure Security Best Practices

### Security Checklist
- [ ] Use Azure AD for identity management
- [ ] Implement managed identities instead of service principals
- [ ] Enable Azure Key Vault for secrets management
- [ ] Use customer-managed keys for encryption
- [ ] Configure Network Security Groups with least privilege
- [ ] Enable Azure Firewall for advanced network security
- [ ] Implement Azure Policy for governance
- [ ] Enable Azure Monitor and Log Analytics
- [ ] Configure Azure Security Center
- [ ] Use Azure Backup for data protection
- [ ] Enable Azure AD Conditional Access
- [ ] Implement Just-In-Time VM access

### Monitoring and Alerting
```hcl
# Action group for security alerts
resource "azurerm_monitor_action_group" "security" {
  name                = "ag-${var.project_name}-${var.environment}-security"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "security"
  
  email_receiver {
    name          = "security-team"
    email_address = var.security_team_email
  }
  
  tags = local.common_tags
}

# Alert for Key Vault access
resource "azurerm_monitor_metric_alert" "key_vault_access" {
  name                = "key-vault-unusual-access"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_key_vault.main.id]
  description         = "Alert when Key Vault has unusual access patterns"
  
  criteria {
    metric_namespace = "Microsoft.KeyVault/vaults"
    metric_name      = "ServiceApiHit"
    aggregation      = "Count"
    operator         = "GreaterThan"
    threshold        = 100
  }
  
  action {
    action_group_id = azurerm_monitor_action_group.security.id
  }
  
  tags = local.common_tags
}
```