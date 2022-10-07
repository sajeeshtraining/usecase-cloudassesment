Introduction
There are two usecases to be showcased for demo

# Usecase -1 
1. Provision a virtual machine on your preferred cloud provider (AWS, Azure, GCP)
2. Install the components necessary to run your favourite webserver on it
3. Write the code to push/install new application code onto the server

# Usecase -2

1. Network only allows secured HTTP access from the outside.
2. Versioning, audit trails are enabled on file/object stores
3. Logs are sent to an appropriate location
4. All data at rest for any databases/volumes are encrypted


# Solution

The above usecases are achieved and will be demonstrated using Microsoft Azure ,Terraform and Azure Powershell Scripts.

# Usecase 1 : VM  provisioning with webserver/application deployment

For the 1st Usecase, Microsoft Azure has been chosen as the target cloud platform and Terraform as the Provisioning/configuration management solution.

The flow for the Usecase 1 ( 1,2 and 3 Sub items) will be as follows.There are Six terraform modules created for provisioning Azure resources along with the Webpackage deployment inside the Azure VM

![TerraformFlow](https://user-images.githubusercontent.com/64534032/194174621-5a71fac4-af6d-4ac7-b7cc-b204b1f1dc0e.png)

Each modules are constructed with a main.tf ,variable.tf and outputs.tf which are called in the consolidated main.tf file for end to end deployment

**Module 1 : Resource Group Creation**


resource "azurerm_resource_group" "demorsg"{
  name = var.group_name
  location = var.location
  tags = local.rsgtags
}


**Module 2 : Virtual Network Creation**

resource "azurerm_virtual_network" "demovnet" {
  name                = var.virtual_network_name
  address_space       = var.virtual_address_space
  resource_group_name = var.resource_group_name
  location            = var.location
  tags = local.vnettags
}

**Module 3 : Subnet Creation**

resource "azurerm_subnet" "demosbt" {
  name                 = var.subnet_name
  address_prefixes     = var.address_prefixes
  virtual_network_name = var.virtual_network_name
  resource_group_name  = var.resource_group_name
}

**Module 4 : Security Group Creation**

resource "azurerm_network_security_group" "demonsg" {
  name                = var.network_security_group
  location            = var.location
  resource_group_name = var.resource_group_name
  tags = local.sectags
}

resource "azurerm_subnet_network_security_group_association" "demosubasso" {

  subnet_id                 = var.subnet_id
  network_security_group_id = var.security_id
}


resource "time_sleep" "wait_seconds3" {
  create_duration = "250s"
}

resource "azurerm_network_security_rule" "nsgrule" {

  for_each                    = local.nsgrules
  name                        = each.key
  direction                   = each.value.direction
  access                      = each.value.access
  priority                    = each.value.priority
  protocol                    = each.value.protocol
  source_port_range           = each.value.source_port_range
  destination_port_range      = each.value.destination_port_range
  source_address_prefix       = each.value.source_address_prefix
  destination_address_prefix  = each.value.destination_address_prefix
  resource_group_name         = var.resource_group_name
  network_security_group_name = var.network_security_group
  depends_on                  = [time_sleep.wait_seconds3]
}



**Module 5 : Virtual Machine creation**

**Use existing Azure Keyvault Secrets value to supply the password for VM creation and access**

data "azurerm_key_vault" "keyvault" {

  name                = var.keyvault_name
  resource_group_name = var.keyvault_rsg
}

data "azurerm_key_vault_secret" "mySecret" {
  name         = "adminuser"
  key_vault_id = data.azurerm_key_vault.keyvault.id

}

**Create VM in Azure Cloud**
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

**Create Public IP for the VM to access WebSite from Internet**

resource "azurerm_public_ip" "public_ip" {
  name                = var.vm_pipname
  resource_group_name = var.resource_group_name
  location            = var.location
  allocation_method   = "Dynamic"
}

**Create Storage acccount for VM Boot Diagnostics**
resource "azurerm_storage_account" "bootdiagsta" {
  name                     = var.storage_name
  location                 = var.location
  resource_group_name      = var.resource_group_name
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

**Set Disk Encryption for VM Disk**
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

**Module 6 : Webpackage Deployment (IIS install and Configure)
                        
 **Install IIS Web Services and Puch the Code**
resource "azurerm_virtual_machine_extension" "vm_extension" {
  name                       = "iisandcodedeploy"
  virtual_machine_id         = var.vm_id
  publisher                  = "Microsoft.Compute"
  type                       = "CustomScriptExtension"
  type_handler_version       = "1.10"
  auto_upgrade_minor_version = true

  settings = <<SETTINGS
            
    {
        "fileUris":["https://dmowepsta.blob.core.windows.net/data/IISandCodePush.ps1"] ,
        "commandToExecute": "powershell.exe -ExecutionPolicy Unrestricted -File IISandCodePush.ps1"
    }
SETTINGS

}

The Consolidated main.tf file will be as below


module "resource_group" {
  source = "./modules/resource_group"
}
module "virtual_network" {
  source              = "./modules/virtual_network"
  resource_group_name = module.resource_group.rsg_name
  location            = module.resource_group.rsg_location
  }

module "subnet" {

  source               = "./modules/subnet"
  virtual_network_name = module.virtual_network.vnet_name
  resource_group_name  = module.resource_group.rsg_name
}

module "security_group" {

  source              = "./modules/security_group"
  resource_group_name = module.resource_group.rsg_name
  location            = module.resource_group.rsg_location
  subnet_id           = module.subnet.sbt_id
  security_id         = module.security_group.sec_id
}

module "virtual_machine" {

  source              = "./modules/virtual_machine"
  resource_group_name = module.resource_group.rsg_name
  location            = module.resource_group.rsg_location
  subnet_id           = module.subnet.sbt_id
  }

resource "time_sleep" "wait_seconds1" {
  create_duration = "120s"
}
module "app_deployment" {

  source     = "./modules/app_code_deployment"
  vm_id      = module.virtual_machine.vms_id
  depends_on = [time_sleep.wait_seconds1]
}
                       
                       
# Use Case 2 : Security Compliance
                       
-  Using tfsec tool to scan the terraform infra code from each modules and ensure the security compliance such as password, storage account settings, network security group configuration before you start deployment.              
                       
-  IIS web logs to send existing log analytics workspace.

-  Allow 443 port, Source & Destinatin ANY in Network security group created as part of the use case-1. This will ensure to access the web site using secure http(s)port 443 from internet.
                       
-  PowerShell script to check compliance on the VM disk encryption , network security group allowed only http(s) , IIS logs redirected to Log analytics work space,Storage account versioning & logging 
                       
                    
