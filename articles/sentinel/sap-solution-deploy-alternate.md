---
title: Microsoft Sentinel SAP data connector expert configuration options, on-premises deployment, and SAPControl log sources  | Microsoft Docs
description: Learn how to deploy the Microsoft Sentinel data connector for SAP environments using expert configuration options and an on-premises machine. Also learn more about SAPControl log sources.
author: batamig
ms.author: bagol
ms.topic: how-to
ms.custom: mvc, ignite-fall-2021
ms.date: 11/09/2021
---

# Expert configuration options, on-premises deployment, and SAPControl log sources

[!INCLUDE [Banner for top of topics](./includes/banner.md)]

This article describes how to deploy the Microsoft Sentinel SAP data connector in an expert or custom process, such as using an on-premises machine and an Azure Key Vault to store your credentials.

> [!NOTE]
> The default, and most recommended process for deploying the Microsoft Sentinel SAP data connector is by [using an Azure VM](sap-deploy-solution.md). This article is intended for advanced users.

> [!IMPORTANT]
> The Microsoft Sentinel SAP solution is currently in PREVIEW. The [Azure Preview Supplemental Terms](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) include additional legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.
>

## Prerequisites

The basic prerequisites for deploying your Microsoft Sentinel SAP data connector are the same regardless of your deployment method.

Make sure that your system complies with the prerequisites documented in the main [SAP data connector deployment procedure](sap-deploy-solution.md#prerequisites) before you start.

For more information, see [Microsoft Sentinel SAP solution detailed SAP requirements (public preview)](sap-solution-detailed-requirements.md).

## Create your Azure key vault

Create an Azure key vault that you can dedicate to your Microsoft Sentinel SAP data connector.

Run the following command to create your Azure key vault and grant access to an Azure service principal:

``` azurecli
kvgp=<KVResourceGroup>

kvname=<keyvaultname>

spname=<sp-name>

kvname=<keyvaultname>
# Optional when Azure MI not enabled - Create sp user for AZ cli connection, save details for env.list file
az ad sp create-for-rbac –name $spname --role Contributor

SpID=$(az ad sp list –display-name $spname –query “[].appId” --output tsv

#Create key vault
az keyvault create \
  --name $kvname \
  --resource-group $kvgp
  
# Add access to SP
az keyvault set-policy --name $kvname --resource-group $kvgp --object-id $spID --secret-permissions get list set
```

For more information, see [Quickstart: Create a key vault using the Azure CLI](../key-vault/general/quick-create-cli.md).

## Add Azure Key Vault secrets

To add Azure Key Vault secrets, run the following script, with your own system ID and the credentials you want to add:

```azurecli
#Add Abap username
az keyvault secret set \
  --name <SID>-ABAPUSER \
  --value "<abapuser>" \
  --description SECRET_ABAP_USER --vault-name $kvname

#Add Abap Username password
az keyvault secret set \
  --name <SID>-ABAPPASS \
  --value "<abapuserpass>" \
  --description SECRET_ABAP_PASSWORD --vault-name $kvname

#Add java Username
az keyvault secret set \
  --name <SID>-JAVAOSUSER \
  --value "<javauser>" \
  --description SECRET_JAVAOS_USER --vault-name $kvname

#Add java Username password
az keyvault secret set \
  --name <SID>-JAVAOSPASS \
  --value "<javauserpass>" \
  --description SECRET_JAVAOS_PASSWORD --vault-name $kvname

#Add abapos username
az keyvault secret set \
  --name <SID>-ABAPOSUSER \
  --value "<abaposuser>" \
  --description SECRET_ABAPOS_USER --vault-name $kvname

#Add abapos username password
az keyvault secret set \
  --name <SID>-ABAPOSPASS \
  --value "<abaposuserpass>" \
  --description SECRET_ABAPOS_PASSWORD --vault-name $kvname

#Add Azure Log ws ID
az keyvault secret set \
  --name <SID>-LOG_WS_ID \
  --value "<logwsod>" \
  --description SECRET_AZURE_LOG_WS_ID --vault-name $kvname

#Add Azure Log ws public key
az keyvault secret set \
  --name <SID>-LOG_WS_PUBLICKEY \
  --value "<loswspubkey>" \
  --description SECRET_AZURE_LOG_WS_PUBLIC_KEY --vault-name $kvname
```

For more information, see the [az keyvault secret](/cli/azure/keyvault/secret) CLI documentation.

## Perform an expert / custom installation

This procedure describes how to deploy the SAP data connector using an expert or custom installation, such as when installing on-premises.

We recommend that you perform this procedure after you have a key vault ready with your SAP credentials.

**To deploy the SAP data connector**:

1. On your on-premises machine, download the latest SAP NW RFC SDK from the [SAP Launchpad site](https://support.sap.com) > **SAP NW RFC SDK** > **SAP NW RFC SDK 7.50** > **nwrfc750X_X-xxxxxxx.zip**.

    > [!NOTE]
    > You'll need your SAP user sign-in information in order to access the SDK, and you must download the SDK that matches your operating system.
    >
    > Make sure to select the **LINUX ON X86_64** option.

1. On your on-premises machine, create a new folder with a meaningful name, and copy the SDK zip file into your new folder.

1. Clone the Microsoft Sentinel solution GitHub repository onto your on-premises machine, and copy Microsoft Sentinel SAP solution **systemconfig.ini** file into your new folder.

    For example:

    ```bash
    mkdir /home/$(pwd)/sapcon/<sap-sid>/
    cd /home/$(pwd)/sapcon/<sap-sid>/
    wget  https://raw.githubusercontent.com/Azure/Azure-Sentinel/master/Solutions/SAP/template/systemconfig.ini 
    cp <**nwrfc750X_X-xxxxxxx.zip**> /home/$(pwd)/sapcon/<sap-sid>/
    ```

1. Edit the **systemconfig.ini** file as needed, using the embedded comments as a guide. For more information, see [Manually configure the SAP data connector](#manually-configure-the-sap-data-connector).

    To test your configuration, you may want to add the user and password directly to the **systemconfig.ini** configuration file. While we recommend that you use [Azure Key vault](#add-azure-key-vault-secrets) to store your credentials, you can also use an **env.list** file, [Docker secrets](#manually-configure-the-sap-data-connector), or you can add your credentials directly to the **systemconfig.ini** file.

1. Define the logs that you want to ingest into Microsoft Sentinel using the instructions in the **systemconfig.ini** file. For example, see [Define the SAP logs that are sent to Microsoft Sentinel](#define-the-sap-logs-that-are-sent-to-microsoft-sentinel).

1. Define the following configurations using the instructions in the **systemconfig.ini** file:

    - Whether to include user email addresses in audit logs
    - Whether to retry failed API calls
    - Whether to include cexal audit logs
    - Whether to wait an interval of time between data extractions, especially for large extractions

    For more information, see [SAL logs connector configurations](#sal-logs-connector-settings).

1. Save your updated **systemconfig.ini** file in the **sapcon** directory on your machine.

1. If you have chosen to use an **env.list** file for your credentials, create a temporary **env.list** file with the required credentials. Once your Docker container is running correctly, make sure to delete this file.

    > [!NOTE]
    > The following script has each Docker container connecting to a specific ABAP system. Modify your script as needed for your environment.
    >

    Run:

    ```bash
    ##############################################################
    # Include the following section if you're using user authentication
    ##############################################################
    # env.list template for Credentials
    SAPADMUSER=<SET_SAPCONTROL_USER>
    SAPADMPASSWORD=<SET_SAPCONTROL_PASS>
    LOGWSID=<SET SENTINEL WORKSPACE id>
    LOGWSPUBLICKEY=<SET SENTINEL WORKSPACE KEY>
    ABAPUSER=SET_ABAP_USER>
    ABAPPASS=<SET_ABAP_PASS>
    JAVAUSER=<SET_JAVA_OS_USER>
    JAVAPASS=<SET_JAVA_OS_USER>
    ##############################################################
    # Include the following section if you are using Azure Keyvault
    ##############################################################
    # env.list template for AZ Cli when MI is not enabled
    AZURE_TENANT_ID=<your tenant id>
    AZURE_CLIENT_ID=<your client/app id>
    AZURE_CLIENT_SECRET=<your password/secret for the service principal>
    ##############################################################
    ```

1. Download and run the pre-defined Docker image with the SAP data connector installed.  Run:

    ```bash
    docker pull docker pull mcr.microsoft.com/azure-sentinel/solutions/sapcon:latest-preview
    docker run --env-file=<env.list_location> -d --restart unless-stopped -v /home/$(pwd)/sapcon/<sap-sid>/:/sapcon-app/sapcon/config/system --name sapcon-<sid> sapcon
    rm -f <env.list_location>
    ```

1. Verify that the Docker container is running correctly. Run:

    ```bash
    docker logs –f sapcon-[SID]
    ```

1. Continue with deploying the **Microsoft Sentinel - Continuous Threat Monitoring for SAP** solution.

    Deploying the solution enables the SAP data connector to display in Microsoft Sentinel and deploys the SAP workbook and analytics rules. When you're done, manually add and customize your SAP watchlists.

    For more information, see [Deploy SAP security content](sap-deploy-solution.md#deploy-sap-security-content).

## Manually configure the SAP data connector

The Microsoft Sentinel SAP solution data connector is configured in the **systemconfig.ini** file, which you cloned to your SAP data connector machine as part of the [deployment procedure](#perform-an-expert--custom-installation).

The following code shows a sample **systemconfig.ini** file:

```python
[Secrets Source]
secrets = '<DOCKER_RUNTIME/AZURE_KEY_VAULT/DOCKER_SECRETS/DOCKER_FIXED>'
keyvault = '<SET_YOUR_AZURE_KEYVAULT>'
intprefix = '<SET_YOUR_PREFIX>'

[ABAP Central Instance]
##############################################################
# Define the following values according to your server configuration.
ashost = <SET_YOUR_APPLICATION_SERVER_HOST>
mshost = <SET_YOUR_MESSAGE_SERVER_HOST> - #In case different then App
##############################################################
group = <SET_YOUR_LOGON_GROUP>
msserv = <SET_YOUR_MS_SERVICE> - #Required only if the message server service is not defined as sapms<SYSID> in /etc/services
sysnr = <SET_YOUR_SYS_NUMBER>
user = <SET_YOUR_USER>
##############################################################
# Enter your password OR your X509 SNC parameters
passwd = <SET_YOUR_PASSWORD>
snc_partnername = <SET_YOUR_SNC_PARTNER_NAME>
snc_lib = <SET_YOUR_SNC_LIBRARY_PATH>
x509cert = <SET_YOUR_X509_CERTIFICATE>
##############################################################
sysid = <SET_YOUR_SYSTEM_ID>
client = <SET_YOUR_CLIENT>

[Azure Credentials]
loganalyticswsid = <SET_YOUR_LOG_ANALYTICS_WORKSPACE_ID>
publickey = <SET_YOUR_PUBLIC_KEY>

[File Extraction ABAP]
osuser = <SET_YOUR_SAPADM_LIKE_USER>
##############################################################
# Enter your password OR your X509 SNC parameters
ospasswd = <SET_YOUR_SAPADM_PASS>
x509pkicert = <SET_YOUR_X509_PKI_CERTIFICATE>
##############################################################
appserver = <SET_YOUR_SAPCTRL_SERVER IP OR FQDN>
instance = <SET_YOUR_SAP_INSTANCE NUMBER, example 10>
abapseverity = <SET_ABAP_SEVERITY 0 = All logs ; 1 = Warning ; 2 = Error>
abaptz = <SET_ABAP_TZ --Use ONLY GMT FORMAT-- example - For OS Timezone = NZST use abaptz = GMT+12>

[File Extraction JAVA]
javaosuser = <SET_YOUR_JAVAADM_LIKE_USER>
##############################################################
# Enter your password OR your X509 SNC parameters
javaospasswd = <SET_YOUR_JAVAADM_PASS>
javax509pkicert = <SET_YOUR_X509_PKI_CERTIFICATE>
##############################################################
javaappserver = <SET_YOUR_JAVA_SAPCTRL_SERVER IP ADDRESS OR FQDN>
javainstance = <SET_YOUR_JAVA_SAP_INSTANCE for example 10>
javaseverity = <SET_JAVA_SEVERITY  0 = All logs ; 1 = Warning ; 2 = Error>
javatz = <SET_JAVA_TZ --Use ONLY GMT FORMAT-- example - For OS Timezone = NZST use javatz = GMT+12>
```

### Define the SAP logs that are sent to Microsoft Sentinel

Add the following code to the Microsoft Sentinel SAP solution **systemconfig.ini** file to define the logs that are sent to Microsoft Sentinel.

For more information, see [Microsoft Sentinel SAP solution logs reference (public preview)](sap-solution-log-reference.md).

```python
##############################################################
# Enter True OR False for each log to send those logs to Microsoft Sentinel
[Logs Activation Status]
ABAPAuditLog = True
ABAPJobLog = True
ABAPSpoolLog = True
ABAPSpoolOutputLog = True
ABAPChangeDocsLog = True
ABAPAppLog = True
ABAPWorkflowLog = True
ABAPCRLog = True
ABAPTableDataLog = False
# ABAP SAP Control Logs - Retrieved by using SAP Conntrol interface and OS Login
ABAPFilesLogs = False
SysLog = False
ICM = False
WP = False
GW = False
# Java SAP Control Logs - Retrieved by using SAP Conntrol interface and OS Login
JAVAFilesLogs = False
##############################################################
```

### SAL logs connector settings

Add the following code to the Microsoft Sentinel SAP data connector **systemconfig.ini** file to define other settings for SAP logs ingested into Microsoft Sentinel.

For more information, see [Perform an expert / custom SAP data connector installation](#perform-an-expert--custom-installation).

```python
##############################################################
[Connector Configuration]
extractuseremail = True
apiretry = True
auditlogforcexal = False
auditlogforcelegacyfiles = False
timechunk = 60
##############################################################
```

This section enables you to configure the following parameters:

|Parameter name  |Description  |
|---------|---------|
|**extractuseremail**     |  Determines whether user email addresses are included in audit logs.       |
|**apiretry**     |   Determines whether API calls are retried as a failover mechanism.      |
|**auditlogforcexal**     |  Determines whether the system forces the use of audit logs for non-SAL systems, such as SAP BASIS version 7.4.       |
|**auditlogforcelegacyfiles**     |  Determines whether the system forces the use of audit logs with legacy system capabilities, such as from SAP BASIS version 7.4 with lower patch levels.|
|**timechunk**     |   Determines that the system waits a specific number of minutes as an interval between data extractions. Use this parameter if you have a large amount of data expected. <br><br>For example, during the initial data load during your first 24 hours, you might want to have the data extraction running only every 30 minutes to give each data extraction enough time. In such cases, set this value to **30**.  |
|     |         |

### Configuring an ABAP SAP Control instance

To ingest all ABAP logs into Microsoft Sentinel, including both NW RFC and SAP Control Web Service-based logs, configure the following ABAP SAP Control details:

|Setting  |Description  |
|---------|---------|
|**javaappserver**     |Enter your SAP Control ABAP server host. <br>For example: `contoso-erp.appserver.com`         |
|**javainstance**     |Enter your SAP Control ABAP instance number. <br>For example: `00`         |
|**abaptz**     |Enter the time zone configured on your SAP Control ABAP server, in GMT format. <br>For example: `GMT+3`         |
|**abapseverity**     |Enter the lowest, inclusive, severity level for which you want to ingest ABAP logs into Microsoft Sentinel.  Values include: <br><br>- **0** = All logs <br>- **1** = Warning <br>- **2** = Error     |

### Configuring a Java SAP Control instance

To ingest SAP Control Web Service logs into Microsoft Sentinel, configure the following JAVA SAP Control instance details:

|Parameter  |Description  |
|---------|---------|
|**javaappserver**     |Enter your SAP Control Java server host. <br>For example: `contoso-java.server.com`         |
|**javainstance**     |Enter your SAP Control ABAP instance number. <br>For example: `10`         |
|**javatz**     |Enter the time zone configured on your SAP Control Java server, in GMT format. <br>For example: `GMT+3`         |
|**javaseverity**     |Enter the lowest, inclusive, severity level for which you want to ingest Web Service logs into Microsoft Sentinel.  Values include: <br><br>- **0** = All logs <br>- **1** = Warning <br>- **2** = Error     |

## Next steps

After you have your SAP data connector installed, you can add the SAP-related security content.

For more information, see [Deploy the SAP solution](sap-deploy-solution.md#deploy-sap-security-content).

For more information, see:

- [Deploy the Microsoft Sentinel SAP data connector with SNC](sap-solution-deploy-snc.md)
- [Microsoft Sentinel SAP solution detailed SAP requirements](sap-solution-detailed-requirements.md)
- [Microsoft Sentinel SAP solution logs reference](sap-solution-log-reference.md)
- [Microsoft Sentinel SAP solution: security content reference](sap-solution-security-content.md)
- [Troubleshooting your Microsoft Sentinel SAP solution deployment](sap-deploy-troubleshoot.md)
