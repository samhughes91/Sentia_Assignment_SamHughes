# Sentia Assignment - Sam Hughes

As part of a Sentia Assessment I have been tasked with using ARM templates and best practices to create the following: 
- Azure Datalake Gen 2 storage account
- Azure Key Vault with secret
- Azure Virtual Network
- A web app capable of accessing the Datalake and the Key Vault
- A Load balancer that sits between the web app and the public facing internet
- Use tags to group resources

## Assumptions
- You need to automate provisioning of the above stated requirements in Azure.
- You do not want to use the Azure App Service (or ASE)
- Your web app is targeted at .NET Framework 3.0
- This tutorial goes beyond the required ARM-templates to prove the environment is configured properly to host a .NET 3.0 web app
- You want your solution based in a European location

## Prerequisites
- Azure Account
- Powershell (including Azure modules)
- Visual Studio
  - ensure you have 'ASP.NET and web development' & 'Azure Development installed'
  
## Installation
These instructions will enable you to deploy the solution to Azure and explain how to configure the virtual machines to enable web app hosting.

Open Powershell and log into your Azure account by using ````Connect-AzAccount````


### Create a resource group

```
# set resourceGroup name (lower case letters)
$resourceGroup = '<yourresourcegroupname>'

# Create resourceGroup
New-AzResourceGroup $resourceGroup -location 'westeurope'
```
### Create an Azure Policy
Add Azure Policy to allow resources to be provisioned only in Europe. Not required for the assignment but it is good practice to set policies.
````
# Create the Policy Definition (Subscription scope)
$definition = New-AzPolicyDefinition -Name "allowed-locations" -DisplayName "Allowed locations" -description "This policy enables you to restrict the locations your organization can specify when deploying resources. Use to enforce your geo-compliance requirements. Excludes resource groups, Microsoft.AzureActiveDirectory/b2cDirectories, and resources that use the 'global' region." -Policy 'https://raw.githubusercontent.com/Azure/azure-policy/master/samples/built-in-policy/allowed-locations/azurepolicy.rules.json' -Parameter 'https://raw.githubusercontent.com/Azure/azure-policy/master/samples/built-in-policy/allowed-locations/azurepolicy.parameters.json' -Mode Indexed
# Set the scope to a resource group; may also be a resource, subscription, or management group
$scope = Get-AzResourceGroup -Name 'YourResourceGroup'
# Set the Policy Parameter (JSON format)
$policyparam = '{ "listOfAllowedLocations": { "value": [ "westeurope", "northeurope" ] } }'
# Create the Policy Assignment
$assignment = New-AzPolicyAssignment -Name 'allowed-locations-assignment' -DisplayName 'Allowed locations Assignment' -Scope $scope.ResourceId -PolicyDefinition $definition -PolicyParameter $policyparam
````

### Change parameters.json file
Navigate to the AzureResourceManager directory and open the parameters.json file, edit the parameters that contain a null value.
- resourceName
  - Will be used as a prefix for all provisioned resource names. Use the same name as your ````$resourceGroup```` (example: odin)
- objectId
  - Refers to the user ObjectID, can be retrieved by running following command in Powershell:
    ````
    az ad signed-in-user show --query objectId -o tsv
    ````
- adminUserName
  - Credentials for your VMs
- adminPassword
  - Credentials for your VMs
  
 ### Deploy solution to Azure 
 The next step is to deploy your solution to Azure. ```` cd ```` into the AzureResourceManager directory and run the following command to validate your deployment
 ````
 az group deployment validate -g $resourceGroup --template-file .\template.json --parameters .\parameters.json
 ````
 If no errors are thrown run the following to provision the solution
 ````
 az group deployment create -g $resourceGroup --template-file .\template.json --parameters .\parameters.json
 ````
 ### Configuring the Azure Virtual Machines
 From the Azure Portal download the RDP files for both your VMs. Log into a VM with the credentials from the parameters.json file. Once you're logged in complete the following steps:
- Open Server Manager and click on Add Roles and Features
- Under 'Server Roles' scroll down to Web Server (IIS) and click on 'Add features'
- Under 'Features' add '.NET Framework 3.5 features' 
- Under 'Role Services' add 'Management Tools --> Management Service'
- Under 'Confirmation' click on 'Specify an alternative path' and enter the following path:
  \Sources\SxS\
- Click on Install

To enable downloading Web Deploy and .NET runtime hosting bundles, configure the following:
- Open Server Manager --> Local Server
- Click on IE Enhanced Security Configuration
- For Administrators select the 'Off' option

Web Deploy enables you to deploy the web app from your local machine. To download Web Deploy follow these steps:
- Open Internet Explorer
- Navigate to the following URL
	https://www.iis.net/downloads/microsoft/web-deploy
- Install the Extension, click on download and select 
	WebDeploy_amd64_en-US.msi
- Allow the pop-up and run the installer
- Choose set-up type Complete
- Finish the wizard

The Azure Virtual Machine needs to have ASP.NET Core runtime installed to host your web app. Follow these steps:
- Navigate to the following URL:
	https://dotnet.microsoft.com/download/dotnet-core/thank-you/runtime-aspnetcore-3.1.1-windows-hosting-bundle-installer
- Run the installer
- Finish the wizard

NOTE: Make sure you configure BOTH VM's!

### Publish your web app to your virtual machine
Now it's time to deploy the web app to the Azure Virtual Machine. I've chosen to use the Publish method in Visual Studio. 
- Open the web app in Visual Studio (ensure you have ASP.NET and web development & Azure Development installed)
- In the solution explorer, navigate to the dotnetcore3_0, right-click and select publish
- Select Azure virtual machines, click on Browse, and select your VM (make sure it's th
- Click on 'Create Profile'
- In the Solution Explorer, open the properties directory and select the newly created publish profile (*.pubxml) document
- Add the following to ````<PropertyGroup>````. 
  This fix will enable you to publish without needing to configure certificates, please note this is not best practice.
  
````
    <UseMSDeployExe>true</UseMSDeployExe>
		<AllowUntrustedCertificate>true</AllowUntrustedCertificate>
````
- Click Save
- Update the appsettings.json file. 
	- AccountName (Name of your ADL Gen 2)
	- AccountKey (Access Key for your ADL Gen 2)
	- VaultName (Name of your Key Vault)
- Save the file
- Navigate back to the dotnetcore3_0 tab and hit publish
- When prompted, enter the password as stated in the parameters.json file

After a few minutes Visual Studio should invoke a web browser navigating to your published web app. 

Follow the above steps to publish the web app to the second vm

## Testing
- Both VMs have DNS configured, you can navigate to the VMs separately to ensure both are working
- The VM's have been placed in a load balancer back end pool. The public facing ip address decides which VM you will navigate to.

## Additional
- Azure Policy incorporated into solution
- Availability set incorporated into solution. The VMs are configured in 2 Fault domains and 5 update domains

## References
 - Web app
  - I have used the code from the following link https://github.com/Azure-Samples/storage-blob-upload-from-webapp
  - I updated the code to run on .NET core 3.0
  - I used code from the following module https://docs.microsoft.com/en-us/learn/modules/manage-secrets-with-azure-key-vault/
  - Incorporated the code and updated it to .NET core 3.0








 
 
  

