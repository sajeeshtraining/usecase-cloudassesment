Introduction
There are two usecases to be showcased for demo

Usecase -1 
1. Provision a virtual machine on your preferred cloud provider (AWS, Azure, GCP)
2. Install the components necessary to run your favourite webserver on it
3. Write the code to push/install new application code onto the server

Usecase -2

1. Network only allows secured HTTP access from the outside.
2. Versioning, audit trails are enabled on file/object stores
3. Logs are sent to an appropriate location
4. All data at rest for any databases/volumes are encrypted


Solution

The above usecases are achieved and will be demonstrated using Microsoft Azure ,Terraform and Azure Powershell Scripts.

Usecase 1: For the 1st Usecase, Microsoft Azure has been chosen as the target cloud platform and Terraform as the Provisioning/configuration management solution.

The flow for the Usecase 1 ( 1,2 and 3 Sub items) will be as follows.There are Six terraform modules created for provisioning Azure resources along with the Webpackage deployment inside the Azure VM

![TerraformFlow](https://user-images.githubusercontent.com/64534032/194174621-5a71fac4-af6d-4ac7-b7cc-b204b1f1dc0e.png)

Each modules are constructed with a main.tf ,variable.tf and outputs.tf which are called in the consolidated main.tf file for end to end deployment

The Consolidated main.tf file will be as below

# Terraform Script for deploying Azure VM along with IIS webpackage
data "azurerm_key_vault" "keyvault" {

  name                = var.keyvault_name
  resource_group_name = var.keyvault_rsg
}

data "azurerm_key_vault_secret" "mySecret" {
  name         = "adminuser"
  key_vault_id = data.azurerm_key_vault.keyvault.id

}
# Create VM in Azure Cloud
resource "azurerm_windows_virtual_machine" "dmowep-win2016" {
  name                  = var.vmname1
  resource_group_name   = var.resource_group_name
  location              = var.location
  size                  = var.size
  admin_username        = var.admin_username
  admin_password        = data.azurerm_key_vault_secret.mySecret.value
  network_interface_ids = [azurerm_network_interface.demonic.id]

  os_disk {
    name                 = var.vmname1
    caching              = "ReadWrite"
    storage_account_type = var.storage_account_type
    
  }

  source_image_reference {
    publisher = var.publishertype
    offer     = var.offer
    sku       = var.sku
    version   = var.Windowsversion
  }

  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.bootdiagsta.primary_blob_endpoint
  }

   tags = local.vmtags
}
resource "azurerm_network_interface" "demonic" {
  name                = var.network_interface_ids
  location            = var.location
  resource_group_name = var.resource_group_name

  ip_configuration {
    name                          = "democonfiguration1"
    subnet_id                     = var.subnet_id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.public_ip.id

  }

  
}
# Create Public IP for the VM to access WebSite from Internet
resource "azurerm_public_ip" "public_ip" {
  name                = var.vm_pipname
  resource_group_name = var.resource_group_name
  location            = var.location
  allocation_method   = "Dynamic"
}

# Create Storage acccount for VM Boot Diagnostics
resource "azurerm_storage_account" "bootdiagsta" {
  name                     = var.storage_name
  location                 = var.location
  resource_group_name      = var.resource_group_name
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

# Set Disk Encryption for VM Disk
resource "azurerm_virtual_machine_extension" "disk-encryption" {
  name                 = "DiskEncryption"
  virtual_machine_id   = azurerm_windows_virtual_machine.dmowep-win2016.id
  publisher            = "Microsoft.Azure.Security"
  type                 = "AzureDiskEncryption"
  type_handler_version = "2.2"

  settings = <<SETTINGS
{
   
  "EncryptionOperation": "${var.encrypt_operation}",
        "KeyVaultURL": "${data.azurerm_key_vault.keyvault.vault_uri}",
        "KeyVaultResourceId": "${data.azurerm_key_vault.keyvault.id}",                   
        "KeyEncryptionAlgorithm": "RSA-OAEP",
        "VolumeType": "All"
}
SETTINGS

}
