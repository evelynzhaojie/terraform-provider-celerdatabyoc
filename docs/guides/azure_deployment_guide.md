---
# generated by https://github.com/hashicorp/terraform-plugin-docs
page_title: "Provision CelerData Cloud BYOC on Azure"
subcategory: ""
description: |-
  
---

# Provision CelerData Cloud BYOC on Azure

This article walks you through the following steps necessary to deploy a CelerData Cloud BYOC cluster on Azure:

- [Preparations](#preparations)
- [Configure providers](#configure-providers)
- [Configure Azure objects](#configure-azure-objects)
- [Describe infrastructure](#describe-infrastructure)
- [Apply configurations](#apply-configurations)

Read this article before you start a Terraform configuration for your cluster deployment on Azure.

## Preparations

Before using the CelerData Cloud BYOC provider to create infrastructure at the Azure account level for the first time, you must complete the following preparations:

### For Azure

For Azure, you need to:

1. Have an Azure service account with administrative privileges.
2. Install the Azure CLI. For more information, see [How to install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

### For CelerData

For CelerData, you need to obtain the credentials with which you can authenticate into the CelerData Cloud BYOC platform. For details, see [Authentication](../index.md#authentication).

### For Terraform

For Terraform, you need to:

1. Install [Terraform](https://developer.hashicorp.com/terraform/downloads) in your terminal.

2. Have a Terraform project. In your terminal, create an empty directory (for example, `terraform`) and then switch to it. (Each separate set of Terraform configuration files must be in its own directory, which is called a Terraform project.)

## Configure providers

This section assumes that you have [completed the preparations](#preparations).

Create a **.tf** file (for example, **main.tf**) in your Terraform project. Then, add the following code snippet to the **.tf** file:

```terraform
terraform {
  required_providers {
    celerdatabyoc = {
      source  = "CelerData/celerdatabyoc"
      version = "<provider_version>"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "2.47.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=3.0.0"
    }
  }
}

provider "celerdatabyoc" {
  client_id = "<client_id>"
  client_secret = "<client_secret>"
}

locals {
  cluster_region = "<Azure_region_ID>"
  azure_region   = "<Azure_region_name>"
}
```

The parameters you need to specify are as follows:

- `provider_version`: Enter the CelerData provider version of your choice. We recommend that you select the latest provider version. You can view the provider versions offered by CelerData Cloud BYOC from the [CelerData Cloud BYOC provider](https://registry.terraform.io/providers/CelerData/celerdatabyoc/latest/docs) page.
- `client_id` and `client_secret`: Enter the **Client ID** and **Secret** of your application key. See [For CelerData](#for-celerdata).
- `cluster_region` and `azure_region`: Enter the ID (for example, `eastus`) and name (for example, `East US`), respectively, of the Azure region in which you want your CelerData cluster to run. See [Supported cloud platforms and regions](https://docs.celerdata.com/private/main/get_started/cloud_platforms_and_regions#azure). The Azure region you specify here must be the same as the Azure region of the resource group you create in [Configure Azure objects](#configure-azure-objects). By setting these region elements as local values, you can then directly set the arguments for these region elements in your Terraform configuration to `local.cluster_region` and `local.azure_region` to save time.

## Configure Azure objects

This section assumes that you have [completed the preparations](#preparations) and have [configured the providers](#configure-providers).

The cluster deployment on Azure depends on the following Azure objects:

- A resource group, which will host all the Azure resources required for the cluster deployment, including a storage account and a container within the storage account, a managed identity, a virtual network and a subnet within the virtual network, a security group, and an SSH public key.
- A storage account and a container within the storage account, which will be used to store your data.
- A managed identity, to which you also need to grant the required permissions, so the cluster will be able to store query profiles to the container.
- An app registration to authorize Terraform as a service principal, and a client secret for the registered application. You also need to add role assignments to the application, so Terraform can launch the resources necessary to deploy the cluster within your Azure service account.
- An SSH public key, which gives access to your virtual machines (VMs) for automatic deployment, so Terraform can deploy the required service processes on your VMs.

  You need to create an SSH public key on your local computer, because you will need to fill the path in the `public_key` element of this object.

- A virtual network and a subnet within the virtual network for the VMs on which the cluster depends.
- A security group to which the subnet is assigned.

To create these Azure objects, you need to declare the following resources in the **.tf** file (for example, **main.tf**) in which you have configured the providers:

```terraform
provider "azuread" {}

provider "azurerm" {
  features {}
  subscription_id = "<Microsoft_subscription_ID>"
}

# Create a resource group
resource "azurerm_resource_group" "example" {
  name     = "<resource_group_name>"
  location = local.azure_region
}

# Create a managed identity
resource "azurerm_user_assigned_identity" "example" {
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  name                = "<managed_identity_name>"
}

# Assign permissions to the managed identity
locals {
  managed_identity_roles = [
    "Reader",
    "Storage Blob Data Contributor",
  ]
}
resource "azurerm_role_assignment" "assignment_identity_roles" {
  count                = length(local.managed_identity_roles)
  role_definition_name = local.managed_identity_roles[count.index]
  scope                = azurerm_resource_group.example.id
  principal_id         = azurerm_user_assigned_identity.example.principal_id
}

# Create a storage account
resource "azurerm_storage_account" "example" {
  name                     = "<storage_account_name>"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
}

# Create a storage container
resource "azurerm_storage_container" "example" {
  name                  = "<storage_container_name>"
  storage_account_name  = azurerm_storage_account.example.name
  container_access_type = "private"
}

# Create an app registration and a client secret for it
resource "azuread_application_registration" "example" {
  display_name     = "<app_registration_name>"
  description      = "My example application"
  sign_in_audience = "AzureADMyOrg"
}
resource "azuread_application_password" "example" {
  application_id = azuread_application_registration.example.id
  display_name   = "<app_secret_name>"
}
resource "azuread_service_principal" "app_service_principal" {
  client_id = azuread_application_registration.example.client_id
}

# Add role assignments to the app application
locals {
  app_registration_roles = [
    "Reader",
    "Virtual Machine Contributor",
    "Network Contributor",
    "Managed Identity Operator"
  ]
}
resource "azurerm_role_assignment" "assignment_app_roles" {
  count                = length(local.app_registration_roles)
  role_definition_name = local.app_registration_roles[count.index]
  scope                = azurerm_resource_group.example.id
  principal_id         = azuread_service_principal.app_service_principal.object_id
}

# Create an SSH public key
resource "azurerm_ssh_public_key" "example" {
  name                = "<ssh_key_name>"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  public_key          = file("~/.ssh/id_rsa.pub")
}

# Create a virtual network
resource "azurerm_virtual_network" "example" {
  name                = "<network_name>"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  address_space       = ["10.0.0.0/16"]
}

# Create a subnet
resource "azurerm_subnet" "example" {
  name                 = "<subnet_name>"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Create a network security group
resource "azurerm_network_security_group" "example" {
  name                = "<security_group_name>"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
}

# Assign the subnet to the security group
resource "azurerm_subnet_network_security_group_association" "example" {
  subnet_id                 = azurerm_subnet.example.id
  network_security_group_id = azurerm_network_security_group.example.id
}
```

See the following documents for more information:

- [azurerm_resource_group](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group)
- [azurerm_user_assigned_identity](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/user_assigned_identity)
- [azurerm_role_assignment](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/role_assignment)
- [azurerm_storage_account](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_account)
- [azurerm_storage_container](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_container)
- [azuread_application_registration](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/application_registration)
- [azuread_service_principal](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/service_principal)
- [azuread_application_password](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/application_password)
- [azurerm_ssh_public_key](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/ssh_public_key)
- [azurerm_virtual_network](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_network)
- [azurerm_subnet](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet)
- [azurerm_subnet_network_security_group_association](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet_network_security_group_association)

## Describe infrastructure

This section provides a sample infrastructure configuration that automates the deployment of a classic CelerData cluster on Azure to help you understand how you can work with the CelerData Cloud BYOC provider. It assumes that you have [completed the preparations](#preparations), [configured the providers](#configure-providers), and [configured the Azure objects](#configure-azure-objects).

To create a classic CelerData cluster, you need to declare the following resources, which represent the infrastructure to be built, in the **.tf** file (for example, **main.tf**) in which you have configured the providers and Azure objects.

### [celerdatabyoc_azure_data_credential](../resources/azure_data_credential.md)

```terraform
resource "celerdatabyoc_azure_data_credential" "example" {
  name                         = "<data_credential_name>"
  managed_identity_resource_id = azurerm_user_assigned_identity.example.id
  storage_account_name         = azurerm_storage_account.example.name
  container_name               = azurerm_storage_container.example.name
}
```

This resource contains the following required arguments:

- `name`: (Forces new resource) The name of the data credential. Enter a unique name.

- `managed_identity_resource_id`: (Forces new resource) The ID of the managed identity. Set this argument to `azurerm_user_assigned_identity.example.id`.

- `storage_account_name`: (Forces new resource) The name of the storage account. Set this argument to `azurerm_storage_account.example.name`.

- `container_name`: (Forces new resource) The name of the container. Set this argument to `azurerm_storage_container.example.name`.

### [celerdatabyoc_azure_deployment_credential](../resources/azure_deployment_credential.md)

```terraform
resource "celerdatabyoc_azure_deployment_credential" "example" {
  name                = "<deployment_credential_name>"
  application_id      = azuread_application_registration.example.client_id
  directory_id        = azuread_service_principal.app_service_principal.application_tenant_id
  client_secret_value = azuread_application_password.example.value
  ssh_key_resource_id = azurerm_ssh_public_key.example.id
}
```

This resource contains the following required arguments:

- `name`: (Forces new resource) The name of the deployment credential. Enter a unique name.

- `application_id`: (Forces new resource) The application (client) ID of the registered application. Set this argument to `azuread_application_registration.example.client_id`.

- `directory_id`: (Forces new resource) The directory (tenant) ID of the registered application. Set this argument to `azuread_service_principal.app_service_principal.application_tenant_id`.

- `client_secret_value`: (Forces new resource) The value of the client secret of the registered application. Set this argument to `azuread_application_password.example.value`.

- `ssh_key_resource_id`: (Forces new resource) The ID of the SSH public key. Set this argument to `azurerm_ssh_public_key.example.id`.

### [celerdatabyoc_azure_network](../resources/azure_network.md)

```terraform
resource "celerdatabyoc_azure_network" "example" {
  name                        = "<network_credential_name>"
  deployment_credential_id    = celerdatabyoc_azure_deployment_credential.example.id
  virtual_network_resource_id = azurerm_virtual_network.example.id
  subnet_name                 = azurerm_subnet.example.name
  region                      = local.cluster_region
  public_accessible           = true
}
```

This resource contains the following required arguments and optional arguments:

**Required:**

- `name`: (Forces new resource) The name of the network configuration. Enter a unique name.

- `deployment_credential_id`: (Forces new resource) The ID of the deployment credential. Set this argument to `celerdatabyoc_azure_deployment_credential.example.id`.

- `virtual_network_resource_id`: (Forces new resource) The resource ID of the Azure virtual network. Set this argument to `azurerm_virtual_network.example.id`.

- `subnet_name`: (Forces new resource) The name of the subnet. Set this argument to `azurerm_subnet.example.name`.

- `region`: (Forces new resource) The ID of the Azure region. Set this argument to `local.cluster_region`, as we recommend that you set the region element as a local value in your Terraform configuration. See [Local Values](https://developer.hashicorp.com/terraform/language/values/locals).

**Optional:**

- `public_accessible`: Whether the cluster can be accessed from public networks. Valid values: `true` and `false`. If you set this argument to `true`, CelerData will attach a load balancer to the cluster to distribute incoming queries, and will assign a public domain name to the cluster so you can access the cluster over a public network. If you set this argument to `false`, the cluster is accessible only through a private domain name.

### [celerdatabyoc_classic_cluster](../resources/classic_cluster.md)

```terraform
resource "celerdatabyoc_classic_cluster" "azure_terraform_test" {
  cluster_name             = "<cluster_name>"
  fe_instance_type         = "<fe_node_instance_type>"
  fe_node_count            = 1
  deployment_credential_id = celerdatabyoc_azure_deployment_credential.example.id
  data_credential_id       = celerdatabyoc_azure_data_credential.example.id
  network_id               = celerdatabyoc_azure_network.example.id
  be_instance_type         = "<be_node_instance_type>"
  be_node_count            = 2
  be_disk_number           = 2
  be_disk_per_size         = 100
  default_admin_password   = "<SQL_user_initial_password>"

  expected_cluster_state = "Running"
  resource_tags = {
    flag = "terraform-test"
  }
  csp    = "azure"
  region = local.cluster_region
}
```

The `celerdatabyoc_classic_cluster` resource contains the following required arguments and optional arguments:

**Required:**

- `cluster_name`: (Forces new resource) The desired name for the cluster.

- `fe_instance_type`: The instance type for FE nodes in the cluster. Select an FE instance type from the table "[Supported instance types](../resources/classic_cluster.md#for-azure)".

- `deployment_credential_id`: (Forces new resource) The ID of the deployment credential. Set the value to `celerdatabyoc_azure_deployment_credential.example.id`.

- `data_credential_id`: (Forces new resource) The ID of the data credential. Set the value to `celerdatabyoc_azure_data_credential.example.id`.

- `network_id`: (Forces new resource) The ID of the network configuration. Set the value to `celerdatabyoc_azure_network.example.id`.

- `be_instance_type`: The instance type for BE nodes in the cluster. Select a BE instance type from the table "[Supported instance types](../resources/classic_cluster.md#for-azure)".

- `default_admin_password`: The initial password of the cluster `admin` user.

- `expected_cluster_state`: When creating a cluster, you need to declare the status of the cluster you are creating. Cluster states are categorized as `Suspended` and `Running`. If you want the cluster to start after provisioning, set this argument to `Running`. If you do not do so, the cluster will be suspended after provisioning.

- `csp`: The cloud service provider of the cluster. Set this argument to `azure`.

- `region`: The ID of the Azure region to which the AWS VPC hosting the cluster belongs. See [Supported cloud platforms and regions](https://docs.celerdata.com/private/main/get_started/cloud_platforms_and_regions#aws). Set this argument to `local.cluster_region`, as we recommend that you set the bucket element as a local value `cluster_region` in your Terraform configuration. See [Local Values](https://developer.hashicorp.com/terraform/language/values/locals).

**Optional:**

- `fe_node_count`: The number of FE nodes in the cluster. Valid values: `1`, `3`, and `5`. Default value: `1`.

- `be_node_count`: The number of BE nodes in the cluster. Valid values: any non-zero positive integer. Default value: `3`.

- `be_disk_number`: (Forces new resource) The maximum number of disks that are allowed for each BE. Valid values: [1,24]. Default value: `2`.

- `be_disk_per_size`: The size per disk for each BE. Unit: GB. Maximum value: `16000`. Default value: `100`. You can only increase the value of this parameter, and the time interval between two value changes must be greater than 6 hours.

- `resource_tags`: The tags to be attached to the cluster.

## Apply configurations

After you finish [configuring the providers](#configure-providers) and [describing the infrastructure objects](#describe-infrastructure) in your Terraform configuration, follow these steps to apply the configuration in your Terraform project:

1. Initialize and install the providers defined in the Terraform configuration:

   ```SQL
   terraform init
   ```

2. Verify that your Terraform project has been properly configured:

   ```SQL
   terraform plan
   ```

   If there are any errors, edit the Terraform configuration and re-run the preceding command.

3. Apply the Terraform configuration:

   ```SQL
   terraform apply
   ```

When the system returns a "Apply complete!" message, the Terraform configuration has been successfully applied.

~> After you change the provider versions in the Terraform configuration, you must run `terraform init -upgrade` to initialize the providers and then run `terraform apply` again to apply the Terraform configuration.

## Delete configurations

You can delete your Terraform configuration if you no longer need it.

Deleting a Terraform configuration means destroying all resources created by the CelerData Cloud BYOC provider.

To delete a Terraform configuration, run the following command in your Terraform project:

```SQL
terraform destroy
```

When the system returns a "Destroy complete!" message, the Terraform configuration has been successfully deleted and the cluster created by the Terraform configuration is also released.