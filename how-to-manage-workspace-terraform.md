
# Manage Azure Machine Learning workspaces using Terraform

In this article, you learn how to create and manage an Azure Machine Learning workspace using Terraform configuration files. [Terraform](/azure/developer/terraform/)'s template-based configuration files enable you to define, create, and configure Azure resources in a repeatable and predictable manner. Terraform tracks resource state and is able to clean up and destroy resources. 

A Terraform configuration is a document that defines the resources that are needed for a deployment. It may also specify deployment variables. Variables are used to provide input values when using the configuration.

## Prerequisites

* An **Azure subscription**. If you don't have one, try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/).
* An installed version of the [Azure CLI](/cli/azure/).
* Configure Terraform: follow the directions in this article and the [Terraform and configure access to Azure](/azure/developer/terraform/get-started-cloud-shell) article.

## Limitations

[!INCLUDE [register-namespace](../../includes/machine-learning-register-namespace.md)]

[!INCLUDE [application-insight](../../includes/machine-learning-application-insight.md)]

## Declare the Azure provider

Create the Terraform configuration file that declares the Azure provider:

1. Create a new file named `main.tf`. If working with Azure Cloud Shell, use bash:

    ```bash
    code main.tf
    ```

1. Paste the following code into the editor:

    **main.tf**:
```terraform
terraform {
  required_version = ">=1.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=2.76.0"
    }
  }
}

provider "azurerm" {
  features {}
}

data "azurerm_client_config" "current" {}

resource "azurerm_resource_group" "default" {
  name     = "rg-${var.name}-${var.environment}"
  location = var.location
}
```

1. Save the file (**&lt;Ctrl>S**) and exit the editor (**&lt;Ctrl>Q**).

## Deploy a workspace

The following Terraform configurations can be used to create an Azure Machine Learning workspace. When you create an Azure Machine Learning workspace, various other services are required as dependencies. The template also specifies these [associated resources to the workspace](./concept-workspace.md#associated-resources). Depending on your needs, you can choose to use the template that creates resources with either public or private network connectivity.

# [Public network connectivity](#tab/publicworkspace)

Some resources in Azure require globally unique names. Before deploying your resources using the following templates, set the `name` variable to a value that is unique.

**variables.tf**:
```terraform
variable "name" {
  type        = string
  description = "Name of the deployment"
}

variable "environment" {
  type        = string
  description = "Name of the environment"
  default     = "dev"
}

variable "location" {
  type        = string
  description = "Location of the resources"
  default     = "East US"
}
```

**workspace.tf**:
```terraform
# Dependent resources for Azure Machine Learning
resource "azurerm_application_insights" "default" {
  name                = "appi-${var.name}-${var.environment}"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  application_type    = "web"
}

resource "azurerm_key_vault" "default" {
  name                     = "kv-${var.name}-${var.environment}"
  location                 = azurerm_resource_group.default.location
  resource_group_name      = azurerm_resource_group.default.name
  tenant_id                = data.azurerm_client_config.current.tenant_id
  sku_name                 = "premium"
  purge_protection_enabled = false
}

resource "azurerm_storage_account" "default" {
  name                            = "st${var.name}${var.environment}"
  location                        = azurerm_resource_group.default.location
  resource_group_name             = azurerm_resource_group.default.name
  account_tier                    = "Standard"
  account_replication_type        = "GRS"
  allow_nested_items_to_be_public = false
}

resource "azurerm_container_registry" "default" {
  name                = "cr${var.name}${var.environment}"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  sku                 = "Premium"
  admin_enabled       = true
}

# Machine Learning workspace
resource "azurerm_machine_learning_workspace" "default" {
  name                    = "mlw-${var.name}-${var.environment}"
  location                = azurerm_resource_group.default.location
  resource_group_name     = azurerm_resource_group.default.name
  application_insights_id = azurerm_application_insights.default.id
  key_vault_id            = azurerm_key_vault.default.id
  storage_account_id      = azurerm_storage_account.default.id
  container_registry_id   = azurerm_container_registry.default.id

  identity {
    type = "SystemAssigned"
  }
}



```

# [Private network connectivity](#tab/privateworkspace)

The configuration below creates a workspace in an isolated network environment using Azure Private Link endpoints. [Private DNS zones](../dns/private-dns-privatednszone.md) are included so domain names can be resolved within the virtual network.

Some resources in Azure require globally unique names. Before deploying your resources using the following templates, set the `resourceprefix` variable to a value that is unique.

When using private link endpoints for both Azure Container Registry and Azure Machine Learning, Azure Container Registry tasks cannot be used for building [environment](/python/api/azure-ai-ml/azure.ai.ml.entities.environment) images. Instead you can build images using an Azure Machine Learning compute cluster. To configure the cluster name of use, set the [image_build_compute_name](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/machine_learning_workspace) argument. You can configure to [allow public access](./how-to-configure-private-link.md?tabs=python#enable-public-access) to a workspace that has a private link endpoint using the [public_network_access_enabled](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/machine_learning_workspace) argument.

**variables.tf**:
```terraform
variable "name" {
  type        = string
  description = "Name of the deployment"
}

variable "environment" {
  type        = string
  description = "Name of the environment"
  default     = "dev"
}

variable "location" {
  type        = string
  description = "Location of the resources"
  default     = "East US"
}

variable "vnet_address_space" {
  type        = list(string)
  description = "Address space of the virtual network"
  default     = ["10.0.0.0/16"]
}

variable "training_subnet_address_space" {
  type        = list(string)
  description = "Address space of the training subnet"
  default     = ["10.0.1.0/24"]
}

variable "aks_subnet_address_space" {
  type        = list(string)
  description = "Address space of the aks subnet"
  default     = ["10.0.2.0/23"]
}

variable "ml_subnet_address_space" {
  type        = list(string)
  description = "Address space of the ML workspace subnet"
  default     = ["10.0.0.0/24"]
}
variable "dsvm_subnet_address_space" {
  type        = list(string)
  description = "Address space of the DSVM subnet"
  default     = ["10.0.4.0/24"]
}

variable "bastion_subnet_address_space" {
  type        = list(string)
  description = "Address space of the bastion subnet"
  default     = ["10.0.5.0/24"]
}

variable "image_build_compute_name" {
  type        = string
  description = "Name of the compute cluster to be created and set to build docker images"
  default     = "image-builder"
}

# DSVM Variables
variable "dsvm_name" {
  type        = string
  description = "Name of the Data Science VM"
  default     = "vmdsvm01"
}
variable "dsvm_admin_username" {
  type        = string
  description = "Admin username of the Data Science VM"
  default     = "azureadmin"
}

variable "dsvm_host_password" {
  type        = string
  description = "Password for the admin username of the Data Science VM"
  sensitive   = true
}
```

**workspace.tf**:
```terraform
# Dependent resources for Azure Machine Learning
resource "azurerm_application_insights" "default" {
  name                = "appi-${var.name}-${var.environment}"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  application_type    = "web"
}

resource "azurerm_key_vault" "default" {
  name                     = "kv-${var.name}-${var.environment}"
  location                 = azurerm_resource_group.default.location
  resource_group_name      = azurerm_resource_group.default.name
  tenant_id                = data.azurerm_client_config.current.tenant_id
  sku_name                 = "premium"
  purge_protection_enabled = true

  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
  }
}

resource "azurerm_storage_account" "default" {
  name                     = "st${var.name}${var.environment}"
  location                 = azurerm_resource_group.default.location
  resource_group_name      = azurerm_resource_group.default.name
  account_tier             = "Standard"
  account_replication_type = "GRS"

  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
  }
}

resource "azurerm_container_registry" "default" {
  name                = "cr${var.name}${var.environment}"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  sku                 = "Premium"
  admin_enabled       = true

  network_rule_set {
    default_action = "Deny"
  }
  public_network_access_enabled = false
}

# Machine Learning workspace
resource "azurerm_machine_learning_workspace" "default" {
  name                    = "mlw-${var.name}-${var.environment}"
  location                = azurerm_resource_group.default.location
  resource_group_name     = azurerm_resource_group.default.name
  application_insights_id = azurerm_application_insights.default.id
  key_vault_id            = azurerm_key_vault.default.id
  storage_account_id      = azurerm_storage_account.default.id
  container_registry_id   = azurerm_container_registry.default.id

  identity {
    type = "SystemAssigned"
  }

  # Args of use when using an Azure Private Link configuration
  public_network_access_enabled = false
  image_build_compute_name      = var.image_build_compute_name
  depends_on = [
    azurerm_private_endpoint.kv_ple,
    azurerm_private_endpoint.st_ple_blob,
    azurerm_private_endpoint.storage_ple_file,
    azurerm_private_endpoint.cr_ple,
    azurerm_subnet.snet-training
  ]

}

# Private endpoints
resource "azurerm_private_endpoint" "kv_ple" {
  name                = "ple-${var.name}-${var.environment}-kv"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  subnet_id           = azurerm_subnet.snet-workspace.id

  private_dns_zone_group {
    name                 = "private-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.dnsvault.id]
  }

  private_service_connection {
    name                           = "psc-${var.name}-kv"
    private_connection_resource_id = azurerm_key_vault.default.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }
}

resource "azurerm_private_endpoint" "st_ple_blob" {
  name                = "ple-${var.name}-${var.environment}-st-blob"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  subnet_id           = azurerm_subnet.snet-workspace.id

  private_dns_zone_group {
    name                 = "private-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.dnsstorageblob.id]
  }

  private_service_connection {
    name                           = "psc-${var.name}-st"
    private_connection_resource_id = azurerm_storage_account.default.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
}

resource "azurerm_private_endpoint" "storage_ple_file" {
  name                = "ple-${var.name}-${var.environment}-st-file"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  subnet_id           = azurerm_subnet.snet-workspace.id

  private_dns_zone_group {
    name                 = "private-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.dnsstoragefile.id]
  }

  private_service_connection {
    name                           = "psc-${var.name}-st"
    private_connection_resource_id = azurerm_storage_account.default.id
    subresource_names              = ["file"]
    is_manual_connection           = false
  }
}

resource "azurerm_private_endpoint" "cr_ple" {
  name                = "ple-${var.name}-${var.environment}-cr"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  subnet_id           = azurerm_subnet.snet-workspace.id

  private_dns_zone_group {
    name                 = "private-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.dnscontainerregistry.id]
  }

  private_service_connection {
    name                           = "psc-${var.name}-cr"
    private_connection_resource_id = azurerm_container_registry.default.id
    subresource_names              = ["registry"]
    is_manual_connection           = false
  }
}

resource "azurerm_private_endpoint" "mlw_ple" {
  name                = "ple-${var.name}-${var.environment}-mlw"
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
  subnet_id           = azurerm_subnet.snet-workspace.id

  private_dns_zone_group {
    name                 = "private-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.dnsazureml.id, azurerm_private_dns_zone.dnsnotebooks.id]
  }

  private_service_connection {
    name                           = "psc-${var.name}-mlw"
    private_connection_resource_id = azurerm_machine_learning_workspace.default.id
    subresource_names              = ["amlworkspace"]
    is_manual_connection           = false
  }
}

# Compute cluster for image building required since the workspace is behind a vnet.
# For more details, see https://docs.microsoft.com/en-us/azure/machine-learning/tutorial-create-secure-workspace#configure-image-builds.
resource "azurerm_machine_learning_compute_cluster" "image-builder" {
  name                          = var.image_build_compute_name
  location                      = azurerm_resource_group.default.location
  vm_priority                   = "LowPriority"
  vm_size                       = "Standard_DS2_v2"
  machine_learning_workspace_id = azurerm_machine_learning_workspace.default.id
  subnet_resource_id            = azurerm_subnet.snet-training.id

  scale_settings {
    min_node_count                       = 0
    max_node_count                       = 3
    scale_down_nodes_after_idle_duration = "PT15M" # 15 minutes
  }

  identity {
    type = "SystemAssigned"
  }
}

```

**network.tf**:
```terraform
# Virtual Network
resource "azurerm_virtual_network" "default" {
  name                = "vnet-${var.name}-${var.environment}"
  address_space       = var.vnet_address_space
  location            = azurerm_resource_group.default.location
  resource_group_name = azurerm_resource_group.default.name
}

resource "azurerm_subnet" "snet-training" {
  name                                           = "snet-training"
  resource_group_name                            = azurerm_resource_group.default.name
  virtual_network_name                           = azurerm_virtual_network.default.name
  address_prefixes                               = var.training_subnet_address_space
  enforce_private_link_endpoint_network_policies = true
}

resource "azurerm_subnet" "snet-aks" {
  name                                           = "snet-aks"
  resource_group_name                            = azurerm_resource_group.default.name
  virtual_network_name                           = azurerm_virtual_network.default.name
  address_prefixes                               = var.aks_subnet_address_space
  enforce_private_link_endpoint_network_policies = true
}

resource "azurerm_subnet" "snet-workspace" {
  name                                           = "snet-workspace"
  resource_group_name                            = azurerm_resource_group.default.name
  virtual_network_name                           = azurerm_virtual_network.default.name
  address_prefixes                               = var.ml_subnet_address_space
  enforce_private_link_endpoint_network_policies = true
}

# ...
# For full reference, see: https://github.com/Azure/terraform/blob/master/quickstart/201-machine-learning-moderately-secure/network.tf
```

There are several options to connect to your private link endpoint workspace. To learn more about these options, refer to [Securely connect to your workspace](./how-to-secure-workspace-vnet.md#securely-connect-to-your-workspace).


## Troubleshooting

### Resource provider errors

[!INCLUDE [machine-learning-resource-provider](../../includes/machine-learning-resource-provider.md)]

## Next steps

* To learn more about Terraform support on Azure, see [Terraform on Azure documentation](/azure/developer/terraform/).
* For details on the Terraform Azure provider and Machine Learning module, see [Terraform Registry Azure Resource Manager Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/machine_learning_workspace).
* To find "quick start" template examples for Terraform, see [Azure Terraform QuickStart Templates](https://github.com/Azure/terraform/tree/master/quickstart):
  
  * [101: Machine learning workspace and compute](https://github.com/Azure/terraform/tree/master/quickstart/101-machine-learning) – the minimal set of resources needed to get started with Azure ML.
  * [201: Machine learning workspace, compute, and a set of network components for network isolation](https://github.com/Azure/terraform/tree/master/quickstart/201-machine-learning-moderately-secure) – all resources that are needed to create a production-pilot environment for use with HBI data.
  * [202: Similar to 201, but with the option to bring existing network components.](https://github.com/Azure/terraform/tree/master/quickstart/202-machine-learning-moderately-secure-existing-VNet).
  * [301:  Machine Learning workspace (Secure Hub and Spoke with Firewall)](https://github.com/azure/terraform/tree/master/quickstart/301-machine-learning-hub-spoke-secure).
  
* To learn more about network configuration options, see [Secure Azure Machine Learning workspace resources using virtual networks (VNets)](./how-to-network-security-overview.md).
* For alternative Azure Resource Manager template-based deployments, see [Deploy resources with Resource Manager templates and Resource Manager REST API](../azure-resource-manager/templates/deploy-rest.md).
* For information on how to keep your Azure ML up to date with the latest security updates, see [Vulnerability management](concept-vulnerability-management.md).