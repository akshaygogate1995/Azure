For added safety, enable soft delete for the Azure Storage Account blobs. This way, even if someone accidentally deletes the Terraform state, you have a grace period to recover it.
State locking is not natively supported by Terraform for Azure Storage Accounts. However, you can use external tooling for this purpose. For example, you can use Azure Storage Account's Lease mechanism to implement locking.
Soft-delete in Azure Key Vault is a feature that allows the recovery of a deleted key vault or a deleted key, secret, or certificate within a retention period. During this retention period, the deleted object is retained and can be recovered, preventing accidental data loss.
Use Azure Key Vault task in Azure DevOps pipelines to securely fetch and use secrets during the pipeline execution.
 

# Azure
# ARM template for Virtual Network

{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2022-03-01",
      "name": "yourVnetName",
      "location": "yourLocation",
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "10.0.0.0/16"
          ]
        },
        "subnets": [
          {
            "name": "yourSubnetName",
            "properties": {
              "addressPrefix": "10.0.0.0/24"
            }
          }
        ]
      }
    }
  ]
}

# Key Vault

{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.KeyVault/vaults",
      "apiVersion": "2019-09-01",
      "name": "yourKeyVaultName",
      "location": "yourLocation",
      "properties": {
        "sku": {
          "family": "A",
          "name": "standard"
        },
        "tenantId": "yourTenantId",
        "accessPolicies": []
      }
    }
  ]
}

# AKS
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "apiVersion": "2021-06-01",
      "type": "Microsoft.ContainerService/managedClusters",
      "name": "yourAKSClusterName",
      "location": "yourLocation",
      "properties": {
        "kubernetesVersion": "1.21.2", // Specify the desired Kubernetes version
        "dnsPrefix": "yourAKSDnsPrefix", // DNS prefix for AKS cluster
        "agentPoolProfiles": [
          {
            "name": "agentpool",
            "count": 1, // Number of nodes in the node pool
            "vmSize": "Standard_D2s_v3" // VM size for the node pool
          }
        ],
        "servicePrincipalProfile": {
          "clientId": "yourServicePrincipalClientId",
          "secret": "yourServicePrincipalSecret"
        }
      }
    }
  ]
}

# YAML Pipeline

trigger:
- master

pr:
- '*'

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureSubscriptionEndpoint: 'YourAzureServiceConnectionName'  # Replace with your Azure service connection name
  resourceGroupName: 'YourResourceGroupName'  # Replace with your Azure resource group name
  aksClusterName: 'YourAKSClusterName'  # Replace with your AKS cluster name
  dockerRegistryServiceConnection: 'YourDockerRegistryServiceConnection'  # Replace with your Docker registry service connection name
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  appName: 'YourAppName'  # Replace with your application name

stages:
- stage: Build
  displayName: 'Build and Publish'
  jobs:
  - job: Build
    steps:
    - task: UseJavaVersion@1
      inputs:
        versionSpec: '11'
    - task: Maven@3
      inputs:
        mavenPomFile: 'pom.xml'
        options: '-Xmx3072m'
        goals: 'clean package'
    - task: PublishBuildArtifacts@1
      inputs:
        pathtoPublish: '$(Build.SourcesDirectory)/target'
        artifactName: 'drop'
        publishLocation: 'Container'

- stage: Deploy
  displayName: 'Deploy to AKS'
  jobs:
  - job: Deploy
    steps:
    - task: UseDotNet@2
      inputs:
        packageType: 'sdk'
        version: '2.1.x'
        installationPath: $(Agent.ToolsDirectory)/dotnet
        addToPath: true
    - task: Docker@2
      inputs:
        containerRegistry: $(dockerRegistryServiceConnection)
        repository: $(appName)
        command: 'build'
        Dockerfile: $(dockerfilePath)
        tags: 'latest'
        arguments: '--build-arg JAR_FILE=$(Build.SourcesDirectory)/target/your-artifact-id-1.0-SNAPSHOT.jar .'
    - task: Docker@2
      inputs:
        containerRegistry: $(dockerRegistryServiceConnection)
        repository: $(appName)
        command: 'push'
        tags: 'latest'
    - task: KubernetesManifest@0
      inputs:
        action: 'deploy'
        kubernetesServiceConnection: 'YourKubeServiceConnection'  # Replace with your Kubernetes service connection name
        manifests: '$(Build.SourcesDirectory)/k8s-deployment.yaml'
        containers: '$(appName):latest'

# Terraform

provider "azurerm" {
  features = {}
}

# Create a resource group
resource "azurerm_resource_group" "rg" {
  name     = "myResourceGroup"
  location = "East US"
}

# Create an Azure Key Vault
resource "azurerm_key_vault" "keyvault" {
  name                        = "myKeyVault"
  resource_group_name         = azurerm_resource_group.rg.name
  location                    = azurerm_resource_group.rg.location
  enabled_for_disk_encryption = true
  enabled_for_template_deployment = true

  sku {
    family = "A"
    name   = "standard"
  }
}

# Create a Virtual Network
resource "azurerm_virtual_network" "vnet" {
  name                = "myVNet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create a Subnet within the Virtual Network
resource "azurerm_subnet" "subnet" {
  name                 = "mySubnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Create a Network Interface for the Virtual Machine
resource "azurerm_network_interface" "nic" {
  name                = "myNIC"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "myNICConfig"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

# Create a Virtual Machine
resource "azurerm_virtual_machine" "vm" {
  name                  = "myVM"
  location              = azurerm_resource_group.rg.location
  resource_group_name   = azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.nic.id]

  vm_size = "Standard_DS1_v2"

  storage_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2019-Datacenter"
    version   = "latest"
  }

  storage_os_disk {
    name              = "myOsDisk"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
  }

  os_profile {
    computer_name  = "myVM"
    admin_username = "adminuser"
    admin_password = "Password1234!"
  }

  os_profile_windows_config {
    enable_automatic_upgrades = true
  }
}
