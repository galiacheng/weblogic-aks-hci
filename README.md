# Run Oracle WebLogic Server on HCI AKS cluster

## Overview

In this guide, you'll walk through the steps to run Oracle WebLogic Server on Azure Stack HCI AKS cluster. At a high level, this will consist of the following:

* Deploy Nested Virtualization Azure Stack HCI infrastructure, to act as your main Hyper-V host
* Deploy the AKS on Azure Stack HCI management cluster and target clusters for WebLogic workloads
* Deploy WebLogic servers on the AKS cluster with NFS share enabled - you can moderize WebLogic workload via domain on pv and model in image
* Expose WebLoigc Administration Console and applications deployed to WebLogic cluster.


## Contents

- [Overview](#overview)
- [Contents](#contents)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Set up AKS cluster on Azure Stack HCI]()
  - [Deploy Hyper-V host]()
  - [Set up Azure HCI AKS cluster]()
  - [Connect to Azure Arc AKS]()
- [Deploy WebLoigc Server with domain on a PV]()
- [Deploy WebLoigc Server with model in imgae]()
- [Expose WebLogic cluster via Load Balancer]()
- [Expose WebLogic cluster via Ingress Controller]()
- [CI/CD consideration]()

## Architecture

WIP

## Prerequisites

- Azure Active Directory
  - Have a user with **Member** type. The user should be granted with **Application administrator**. The sample will use this user to connect to Azure from HCI.
  - If you are Microsoft employee, please do not use your MSFT account.

- Azure Subscription.
  - The user should have **Owner** role in the subscription. The user will assign a contributor role to an AAD application that Windows Admin Center creat.
 
## Set up AKS cluster on Azure Stack HCI

### Deploy Hyper-V host

Follow this [guide](https://github.com/Azure/aks-hci/blob/main/eval/steps/1_AKSHCI_Azure.md) to deploy your Hyper-V host. This guideline enables your to deploy the resources with two options:
  * [Use ARM template to deploy VM resources](https://github.com/Azure/aks-hci/blob/main/eval/steps/1_AKSHCI_Azure.md#option-1---creating-the-vm-with-an-azure-resource-manager-json-template)
  * [Use PowerShell](https://github.com/Azure/aks-hci/blob/main/eval/steps/1_AKSHCI_Azure.md#option-2---creating-the-azure-vm-with-powershell)

In this sample, we use ARM template to create the resources with default settings:

![Deploy Azure VMs](resources/screenshot-vm-deployment.png)

Access the VM machine using Window Remote Desktop.

In this example, login the machine with user `azureuser` and password that input in the ARM deployment.

### Set up Azure HCI AKS cluster

This sample will use Windows Admin Center to set up Azure connection and AKS cluster, if you prefer to use Powershell, please find steps in this [guide](https://github.com/Azure/aks-hci/blob/main/eval/steps/2b_DeployAKSHCI_PS.md)


Now you should be able to access the VM.

Before launching Windows Admin Center, we have to set trusted sites and allowed popups for browser.

1. [Make sure the default web browser is Edge browser](https://github.com/Azure/aks-hci/blob/main/eval/steps/2a_DeployAKSHCI_WAC.md#set-microsoft-edge-as-default-browser).
2. Set trusted sites. 

    To avoid connections issue in the following steps, we suggest you to enable the follow sites in trusted sites.

    Search for **Internet Options** in the Windows Start Menu, go to the **Security** tab, under the **Trusted Sites** option, click on the sites button and add the URLs in the dialog box that opens. Please add https://login.microsoftonline.com and https://login.live.com.

3. [Allow popups in Edge browser](https://github.com/Azure/aks-hci/blob/main/eval/steps/2a_DeployAKSHCI_WAC.md#allow-popups-in-edge-browser)
   
   Besides the gateway URL (link like https://akshcihost001), please also add https://login.microsoftonline.com and https://login.live.com.

4. [Connect to Azure](https://github.com/Azure/aks-hci/blob/main/eval/steps/2a_DeployAKSHCI_WAC.md#configure-windows-admin-center)

   Open Windows Admin Center and connect to Azure. As the user we are using has permission to create AAD app (it is **Application administrator** role in the tenant), we sugguest your to select **Create New** under **Connect to Azure Active Directory**.

6. Set up AKS cluster in CHI. Follow this [document](https://github.com/Azure/aks-hci/blob/main/eval/steps/2a_DeployAKSHCI_WAC.md#deploying-aks-on-azure-stack-hci-management-cluster) to set up AKS. Please stop before clicking **Apply**.

    After inputing the configurations, in the **Review** page, you should find settings like:

    ![Configure AKS cluster](resources/screenshot-aks-configurations.png)

    Before clicking **Apply**, make sure the AAD application is authorized to call APIs when they are granted permissions by users/admins as part of the consent process. 
    
    Go to Azure portal, click **Azure Active Directory**, click your application name, e.g. `WindowsAdminCenter-https://akshcihost1213`, and click **API permissions**. You will find API status are not authorized.

    Click **Grant admin consent for your subscription** to authorize, find the following screenshot.
    ![Grant Admin Consent](resources/screenshot-grant-admin-consent.png)

    After the deployment finished, the API permission status should be updated, like:
    ![Grant Admin Consent](resources/screenshot-grant-admin-consent2.png)

    Now you can lick **Apply** to apply the configuration and set up AKS cluster.

### Connect to Azure Arc AKS


