# arm-oraclelinux-ohs
This page documents how Oracle HTTP Sever can be deployed for existing WebLogic cluster and dynamic cluster offers.

## Prerequisites

### Environment for Setup

* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure), use `az --version` to test if `az` works.

### WebLogic Server Instance

The template will be applied to an existing WebLogic cluster or WebLogic dynamic cluster instance.  If you don't have one, please create a new instance from the Azure portal, by following the link to the offer [Oracle WebLogic Offers](https://azuremarketplace.microsoft.com/en-us/marketplace/apps?search=oracle%20weblogic).

Oracle HTTP Server deployment requires following details from existing WebLogic deployment instance.

* #### WebLogic adminserver VM name

Go to your existing WebLogic Azure deployment resource group, and check adminserver VM hostname. This should be computer name and not public IP or public DNS name. 

For example, in below image WebLogic adminserver VM name is `adminVM` .

![Admin vm](https://user-images.githubusercontent.com/36834780/114751513-dcbae300-9d72-11eb-88c1-51fc39cd4f62.png)


* #### WebLogic deployed storage account name

Go to your existing WebLogic Azure deployment resource group, and note down Storage account resource name.

For example, in below image Storage account name is `79fefbolvm`.

![Storage account name](https://user-images.githubusercontent.com/36834780/114751871-5226b380-9d73-11eb-8efe-b02140d3aa6d.png)


* #### WebLogic deployed Virtual Network name

Go to your existing WebLogic Azure deployment resource group, and note down Virtual network resource name.

For example, in below image Virtual network name is `wlsd_VNET`.

![Virtual network](https://user-images.githubusercontent.com/36834780/114751992-77b3bd00-9d73-11eb-989f-e27e93a96c3e.png)

 
* #### WebLogic deployed Network Security Group name

Go to your existing WebLogic Azure deployment resource group, and note down Network security group resource name.

For example, in below image Network security group name is `wls-nsg`.

![Network security group](https://user-images.githubusercontent.com/36834780/114752103-93b75e80-9d73-11eb-81fa-0aba0aa9bc50.png)


### Certificate for SSL Termination

Oracle HTTP Server serves as the front end load balancer for the WebLogic cluster or WebLogic dynamic cluster, hence it must be provided with a certificate to allow browsers to connect via SSL.

##### Create TLS/SSL certificate

This section shows how to create a self-signed SSL certificate in a format suitable for use by Oracle HTTP Server deployed with WebLogic on Azure. The example provided below is one of the ways to create self-signed certificates for JKS and PKCS12 format.

`Note: Self-signed certificates should only be used for testing purpose and it is not recommended for production purpose.`

You need to choose one of the formats JKS or PKCS12 format and note down which format is selected. You’ll need it later when deploying Oracle HTTP Server.

##### JKS format certificate
 
Create JKS format file using following $JAVA_HOME/bin/keytool command.

<code> keytool -genkey -keyalg RSA -alias selfsigned -keystore mycert.jks -storepass password -validity 360 -keysize 2048 -keypass password -storetype jks </code>

Provide all information prompted and store in a file, mycert.jks.


##### PKCS12 format certificate

 Create an RSA PRIVATE KEY.

<code> openssl genrsa 2048 > private.pem </code>

 Create a corresponding public key.

<code> openssl req -x509 -new -key private.pem -out public.pem </code>
 
 Provide all information prompted.
 
 Export the certificate as a “. p12” file.
 
 <code> openssl pkcs12 -export -in public.pem -inkey private.pem -out mycert.p12 </code>
 
 Note down the Export Password provided.

### SSL Configuration options

Oracle HTTP Server deployment provides following two ways to provide SSL configuration details. 

#### 1) Uploading existing certificates

| Parameter Name | Explanation |
|----------------|-------------|
|`ohsSSLConfigAccessOption`| Select `uploadConfig` option for supplying certificate data and password as arguments|
|`uploadedKeyStoreData`| Base64 encoded SSL Certificate Data|
|`uploadedKeyStorePassword`| Password of the SSL Certificate Data|
|`keyType`| Provide Key type is JKS or PKCS12 signed certificates |

#### 2) Use KeyStores stored in Azure Key Vault

| Parameter Name | Explanation |
|----------------|-------------|
|`ohsSSLConfigAccessOption` | Select `keyVaultStoredConfig` if certificates are available as part of Azure keyvault|
|`keyVaultName` | KeyVault Name where certificates are stored|
|`keyVaultResourceGroup` | Resource group name in current subscription containing the KeyVault |
|`keyVaultSSLCertDataSecretName`| The name of the secret in the specified KeyVault whose value is the SSL Certificate Data |
|`keyVaultSSLCertPasswordSecretName`| The name of the secret in the specified KeyVault whose value is the password for the SSL Certificate|



## Prepare the Parameters JSON file

You must construct a parameters JSON file containing the parameters to the Oracle HTTP Server ARM template.  See [Create Resource Manager parameter file](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/parameter-files) for background information about parameter files.   We must specify the information of the existing SSL certificate. This section shows how to obtain the values for the following required properties.

| Parameter Name | Explanation |
|----------------|-------------|
|`_artifactsLocation`| See below for details. |
|`adminPasswordOrKey`|Password of administration account for the new Virtual Machine that hosts Oracle HTTP Server.|
|`adminUsername`| User name of administration account for the new Virtual Machine that hosts Oracle HTTP Server.|
|`dnsLabelPrefix`| Unique DNS Name for the Public IP used to access the Virtual Machine. |
|`ohsSSLConfigAccessOption` | Provide options uploadConfig or keyVaultStoredConfig based on how certificates are supplied|
|`keyType` | Provide SSL certificate key type. Available options are JKS and PKCS12 |
|`keyVaultName` | Azure KeyVault Name where certificates are stored|
|`keyVaultResourceGroup` | Resource group name in current subscription containing the KeyVault |
|`keyVaultSSLCertDataSecretName` | The name of the secret in the specified KeyVault whose value is the SSL Certificate Data |
|`keyVaultSSLCertPasswordSecretName` |The name of the secret in the specified KeyVault whose value is the password for the SSL Certificate |
|`ohsComponentName`| Oracle HTTP Server component name to be configured|
|`ohsDomainName` | Oracle HTTP Server domain name to be configured|
|`ohsNMPassword` | Oracle HTTP Server nodemanager password|
|`ohsNMUser`| Oracle HTTP Server nodemanager username|
|`ohsSkuUrnVersion` | Oracle HTTP Server base image. Refer [Oracle HTTP Server Offers](https://azuremarketplace.microsoft.com/en-us/marketplace/apps?search=oracle%20ohs).
|`ohshttpPort` |Oracle HTTP Server configured with HTTP port|
|`ohshttpsPort`|Oracle HTTP Server configured with HTTPS port|
|`oracleVaultPswd`| Password to configure SSL Oracle vault store|
|`uploadedKeyStoreData` | Base64 encoded SSL Certificate Data. See below for details|
|`uploadedKeyStorePassword`|Password of the SSL Certificate Data|
|`storageAccountName` | Existing WebLogic deployed resource group storage account name|
|`virtualNetworkName`| Existing WebLogic deployed resource group , virtual network name|
|`networkSecurityGroup`|Existing WebLogic deployed resource group , network security group name|
|`wlsAdminVMName` | Existing WebLogic deployed admin server computer or VM name|
|`wlsPassword` | Password for existing deployed WebLogic domain name|
|`wlsUserName` | User name for existing deployed WebLogic domain name|


### `_artifactsLocation`

This value must be the following.

```bash
{{ armTemplateBasePath }}
```

### `uploadedKeyStoreData`
Use base64 to encode your existing SSL certificate. 

<code> base64 your-JKS/PKCS12-certificate-contents -w 0 > temp.txt </code>

Use temp.txt contents to set the value for uploadedKeyStoreData


#### Example Parameters JSON for uploading certificates

Here is a fully filled out parameters file.   Note that we did not include any optional parameters, assuming that deployment is done with default values.

```json
{
  "adminPasswordOrKey": {
    "value": "Azure123456!"
  },
  "adminUsername": {
    "value": "azureuser"
  },
  "authenticationType": {
    "value": "password"
  },
  "wlsPassword": {
    "value": "Welcome1234567"
  },
  "wlsUserName": {
    "value": "weblogic"
  },
  "ohsSkuUrnVersion": {
    "value": "ohs-122140-jdk8-ol73;ohs-122140-jdk8-ol73;latest"
  },
  "ohsDomainName": {
    "value": "ohsStandaloneDomain"
  },
  "ohsComponentName": {
    "value": "ohs_component"
  },
  "ohsNMUser": {
    "value": "weblogic"
  },
  "ohsNMPassword": {
    "value": "Nmpswd1234567"
  },
  "ohshttpPort": {
    "value": "7777"
  },
  "ohshttpsPort": {
    "value": "4444"
  },
  "oracleVaultPswd": {
    "value": "Gumby12340987"
  },
  "ohsSSLConfigAccessOption": {
    "value": "uploadConfig"
  },
  "keyType": {
    "value": "PKCS12"
  },
  "uploadedKeyStoreData": {
    "value": "/u3+7QAAAAIAAAABAAAAAQAKc2VsZnNpZ25lZAAAAX ...."
  },
  "uploadedKeyStorePassword": {
    "value": "azure123!"
  },
  "storageAccountName": {
    "value": "8350c6olvm"  
  },
  "virtualNetworkName": {
    "value": "wlsd_VNET"  
  },
  "wlsAdminVMName": {
    "value": "adminVM"    
  },
  "networkSecurityGroup": {
    "value": "wls-nsg"
  },
  "vmSizeSelect": {
   "value": "Standard_B1s"
  }
}
```

#### Example Parameters JSON for keystores stored in Azure Key Vault


```json
{
  "adminPasswordOrKey": {
    "value": "Azure123456!"
  },
  "adminUsername": {
    "value": "azureuser"
  },
  "authenticationType": {
    "value": "password"
  },
  "wlsPassword": {
    "value": "Welcome1234567"
  },
  "wlsUserName": {
    "value": "weblogic"
  },
  "ohsSkuUrnVersion": {
    "value": "ohs-122140-jdk8-ol73;ohs-122140-jdk8-ol73;latest"
  },
  "ohsDomainName": {
    "value": "ohsStandaloneDomain"
  },
  "ohsComponentName": {
    "value": "ohs_component"
  },
  "ohsNMUser": {
    "value": "weblogic"
  },
  "ohsNMPassword": {
    "value": "Nmpswd1234567"
  },
  "ohshttpPort": {
    "value": "7777"
  },
  "ohshttpsPort": {
    "value": "4444"
  },
  "oracleVaultPswd": {
    "value": "Gumby12340987"
  },
  "ohsSSLConfigAccessOption": {
    "value": "uploadConfig"
  },
  "keyType": {
    "value": "JKS"
  },
  "keyVaultResourceGroup": {
    "value": "keyvaultRG"
  },
  "keyVaultName": {
    "value": "keyvault"
  },
  "keyVaultSSLCertDataSecretName": {
    "value": "certData"
  },
  "keyVaultSSLCertPasswordSecretName": {
    "value": "certPassword"
  },
  "storageAccountName": {
    "value": "8350c6olvm"  
  },
  "virtualNetworkName": {
    "value": "wlsd_VNET"  
  },
  "wlsAdminVMName": {
    "value": "adminVM"    
  },
  "networkSecurityGroup": {
    "value": "wls-nsg"
  },
  "vmSizeSelect": {
   "value": "Standard_B1s"
  }
}
```

### Invoke the ARM template
Assume your parameters file is available in the current directory and is named parameters.json. This section shows the commands to configure your deployment with a Oracle HTTP Server. Replace yourResourceGroup with the Azure resource group in which the WebLogic instance is deployed.

`Note: Provide same Azure resource group name with which WebLogic instance is deployed.`

#### First, validate your parameters file

The az deployment group validate command is very useful to validate your parameters file is syntactically correct.

```bash
az deployment group validate --verbose --resource-group `yourWebLogicResourceGroup` --parameters @parameters.json --template-uri {{ armTemplateBasePath }}/mainTemplate.json
```
If the command returns with an exit status other than `0`, inspect the output and resolve the problem before proceeding.  You can check the exit status by executing the commad `echo $?` immediately after the `az` command.

#### Next, execute the template

After successfully validating the template invocation, change `validate` to `create` to invoke the template.
```bash
az deployment group create --verbose --resource-group `yourWebLogicResourceGroup` --parameters @parameters.json --template-uri {{ armTemplateBasePath }}nestedtemplates/ohsNestedTemplate.json
```

As with the validate command, if the command returns with an exit status other than 0, inspect the output and resolve the problem.

This is an example output of successful deployment.  Look for `"provisioningState": "Succeeded"` in your output.

```bash
    "provisioningState": "Succeeded",
    "template": null,
    "templateHash": "13760326614657528322",
```

## Verify Oracle HTTP Server setup

Successful deployment provides Oracle HTTP Server access url in your output, similar to below.

```json
      "ohsAccessURL": {
        "type": "String",
        "value": "http://wls-5ff4cab395-loadbalancer.eastus.cloudapp.azure.com:7777"
      },
      "ohsSecureAccessURL": {
        "type": "String",
        "value": "https://wls-5ff4cab395-loadbalancer.eastus.cloudapp.azure.com:4444"
      }
```

## How to use Oracle HTTP Server as load balancer

### Deploy application on WebLogic cluster or WebLogic dynamic cluster

This section describes how to deploy sample application to the WebLogic cluster or WebLogic dynamic cluster running instance.This section makes use of sample application browsestore.war web application, as an example. 
Download [browsestore.war](https://www.oracle.com/webfolder/technetwork/tutorials/obe/fmw/wls/12c/12_2_1/004-13-001-ohsCP/files/browsestore.war) web application and keep it locally.

* Access WebLogic admin console 
* Login to WebLogic admin console by entering Usernameand Password
* Click on `Deployments` title to navigate to the `Deployments` page
* In the `Change Center`, click the `Lock & Edit`
* Under `Deployments` page, click on `Install` and then click on `Upload your file(s)`
* At `Upload a deployment to the Administration Server`,click on `Browse` and upload the sample application browsestore.war which is available locally
* Click on `Next`, keeping all options default, till you reach the below page.
  Select `cluster1` under `Clusters` as `Available targets` for your application
* Click on `Next`, keeping all options default and finally click on `Finish`
* In the `Change Center`, click `Activate Changes`
* Start the application by selecting the applicationbrowsestore in the `Deployments` table, then clicking `Start > Servicing all requests` from the `Control` menu
* On the Confirmation window, click `Yes`
* Make sure your deployed application state is `Active` under `Summary of Deployments`

### Accessing deployed application using Oracle HTTP Server

Note down `ohsAccessURL` and `ohsSecureAccessURL` after successful completion of Oracle HTTP Server deployment.
You can access your deployed application using following ways.
* Using ohsAccessURL
  
  `ohsAccessURL/your application path`
  
  In this case http://wls-5ff4cab395-loadbalancer.eastus.cloudapp.azure.com:7777/browsestore/

* Using ohsAccessURL
  
  `ohsSecureAccessURL/your application path`
 
  In this case https://wls-5ff4cab395-loadbalancer.eastus.cloudapp.azure.com:4444/browsestore/

### 
