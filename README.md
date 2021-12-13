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
- [Set up AKS cluster on Azure Stack HCI](#set-up-aks-cluster-on-azure-stack-hci)
  - [Deploy Hyper-V host](#deploy-hyper-v-host)
  - [Set up Azure HCI AKS cluster](#set-up-azure-hci-aks-cluster)
  - [Connect to Azure Arc AKS](#connect-to-azure-arc-aks)
- [Deploy WebLoigc Server with domain on a PV]()
  - [Enable NFS server for storage]()
  - [Install Oracle WebLogic Server Kubernetes Operator]()
  - [Deploy Weblogic cluster]()
  - [Expose WebLogic cluster via Load Balancer]()
  - [Deploy application]()
- [Deploy WebLoigc Server with model in imgae]()
  - [Build WebLogic image and push to ACR]()
  - [Install Oracle WebLogic Server Kubernetes Operator]()
  - [Deploy Weblogic cluster]()
  - [Expose application via Ingress Controller]()
- [CI/CD consideration]()

## Architecture

WIP

## Prerequisites

- Azure Active Directory
  - Have a user with **Member** type. The user should be granted with **Application administrator**. The sample will use this user to connect to Azure from HCI.
  - If you are Microsoft employee, please do not use your MSFT account.

- Azure Subscription.
  - The user should have **Owner** role in the subscription. The user will assign a contributor role to an AAD application that Windows Admin Center creates.
 
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

5. Set up AKS cluster in CHI. Follow this [document](https://github.com/Azure/aks-hci/blob/main/eval/steps/2a_DeployAKSHCI_WAC.md#deploying-aks-on-azure-stack-hci-management-cluster) to set up AKS. Please stop before clicking **Apply**.

    After inputing the configurations, in the **Review** page, you should find settings like:

    ![Configure AKS cluster](resources/screenshot-aks-configurations.png)

    Before clicking **Apply**, make sure the AAD application is authorized to call APIs when they are granted permissions by users/admins as part of the consent process. 
    
    Go to Azure portal, click **Azure Active Directory**, click your application name, e.g. `WindowsAdminCenter-https://akshcihost1213`, and click **API permissions**. You will find API status are not authorized.

    Click **Grant admin consent for your subscription** to authorize, find the following screenshot.
    ![Grant Admin Consent](resources/screenshot-grant-admin-consent.png)

    After the deployment finished, the API permission status should be updated, like:
    ![Grant Admin Consent](resources/screenshot-grant-admin-consent2.png)

    Now you can lick **Apply** to apply the configuration and set up AKS cluster. It takes about 20min for the deployment. After the process finishes, you should find:

    ![Set up AKS completed](resources/screenshot-complete-aks-setup.png)

6. Create target cluster. Now you have set up management cluster in your HCI environment, but you still need to create AKS cluster for WebLogic.

    Follow step in [Create Target Cluster](https://github.com/Azure/aks-hci/blob/main/eval/steps/2a_DeployAKSHCI_WAC.md#create-a-kubernetes-cluster-target-cluster) to create an AKS cluster.

    This sample use the following settings:

    ![Set up AKS target cluster](resources/screenshot-aks-cluster.png)

    It takes about 20 min to set up target cluster.

### Connect to Azure Arc AKS

Now, you should have Azure Arc-enabled AKS cluster, you can managed the cluster outside the VM machine.

Let's have a look at the Arc AKS from Azure portal, your aks name should be `my-workload-cluster` if you didn't change the cluster name:

![Azure Arc-enabled AKS cluster](resources/screenshot-azure-arc-aks.png)

While when you click **Workloads**, you are asked to sign in to view your Kubernetets resources.

Document [Use Cluster Connect to connect to Azure Arc-enabled Kubernetes clusters](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/cluster-connect#prerequisites) introduces two options to connect the cluster.

As we need admin role to create the WebLogic operator and deploy WebLogic cluster, this sample will guide you to connect to the cluster using service account token.

Open Windows Admin Center, click the server you are working on, here is `akshcihost1213.akshci.local`. Click **PowerShell** on the left, the Tools Navigation.

After the PowerShell console launches, you are connected to the machine successfully if you find text like:

```text
Connecting to akshcihost1213.akshci.local, Logon user AKSHCIHost1213\AzureUser
[akshcihost1213.akshci.local]: PS C:\Users\AzureUser\Documents>
```

Import HCI modules by running the following commands:

```powershell

Import-Module AksHci
Get-Command -Module AksHci
```

Query cluster information:

```powershell

Get-AksHciCluster
```

Output of this sample:

```text
Status                : {ProvisioningState, Details}
ProvisioningState     : Deployed
KubernetesVersion     : v1.21.2
PackageVersion        : v1.21.2-kvapkg.2
NodePools             : linuxnodepool
WindowsNodeCount      : 0
LinuxNodeCount        : 3
ControlPlaneNodeCount : 1
Name                  : my-workload-cluster
```

To retrieve the kubeconfig file for the cluster, you'll need to run the following command:

```powershell
Get-AksHciCredential -Name akshciclus001 -Confirm:$false
dir $env:USERPROFILE\.kube
```

Now, you can use `kubectl` to manage the cluster. The HCI Hyper-V host has installed `kubectl`, you can use it in the PowerShell console.

We need a service account to connect to the AKS cluster. Create a service account in any namespace and grant it with `cluster-admin` role, see this [document](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#kubectl-create-rolebinding) for more information. Run the following command in the PowerShell console.

```powershell
kubectl create serviceaccount admin-user
kubectl create clusterrolebinding admin-user-binding --clusterrole cluster-admin --serviceaccount default:admin-user

$SECRET_NAME=$(kubectl get serviceaccount admin-user -o jsonpath='{$.secrets[0].name}')
$RAWTOKEN=$(kubectl get secret $SECRET_NAME -o jsonpath='{$.data.token}')
$TOKEN=[Text.Encoding]::Utf8.GetString([Convert]::FromBase64String($RAWTOKEN))

echo $TOKEN
```

Now you have to token to access the AKS cluster. Copy the token to a file, make sure the string is one line string, you may have to remove the new lines manually.

Open Azure Arc-enabled AKS cluster, input the token to sign in the cluster. 

![Sign in AKS cluster](resources/screenshot-sign-in-aks-cluster.png)

You should be able to access the AKS resource from Azure portal.

![Sign in AKS cluster](resources/screenshot-aks-namespaces.png)


With the token, we can manage the AKS cluster resource in the development machine, you are required to use a Linux machine, WSL in Windows machine, or Azure Cloud Shell.

This sample set up WebLogic cluster using WSL terminal from Windows 11 machine.

Run the shell commands to connect to AKS outside the HCI host.

Install `connectedk8s` extension and enable the Cluster Connect the Arc-enabled Kubernetes cluster.

```bash
# Install or upgrade Azure CLI to version >= 2.16.0 and <= 2.29.0
az extension add --name connectedk8s
az extension update --name connectedk8s
```

Connect to AKS cluster.

```bash
CLUSTER_NAME=<my-workload-cluster>
RESOURCE_GROUP=<resource-group-name>
$TOKEN=<the-token-from-above-powershell-command>

az connectedk8s proxy -n $CLUSTER_NAME -g $RESOURCE_GROUP --token $TOKEN
```

You will find output like:

```text
$ az connectedk8s proxy -n $CLUSTER_NAME -g $RESOURCE_GROUP --token $TOKEN
Proxy is listening on port 47011
Merged "my-workload-cluster" as current context in /home/bamboo/.kube/config
Start sending kubectl requests on 'my-workload-cluster' context using kubeconfig at /home/bamboo/.kube/config
Press Ctrl+C to close proxy.
```

Please do not close the proxy, we will deploy the WebLogic cluster in the next steps.

## Deploy WebLoigc Server with domain on a PV

Now the AKS cluster is ready to deploy WebLogic workloads. The previous steps have connected the AKS cluster in your local development machine, which has `git`, `helm`, `kubectl`, AZ CLI installed.

You are allowed to manage the AKS resources using `kubectl`.

Let's test by getting the namespace, you should find the following output:

```bash
$ kubectl get ns
NAME              STATUS   AGE
azure-arc         Active   103m
default           Active   113m
kube-node-lease   Active   113m
kube-public       Active   113m
kube-system       Active   113m
```

Before moving forward, we still have to create storage for the domain, as we will store the domain configuration to a PV.

This sample will enable the NFS server in the Hyper-V host, the Windows Server 2022 Azure VM. Follow this [guide](https://docs.microsoft.com/en-us/windows-server/storage/nfs/deploy-nfs) to eable NFS server and create NFS share.

This sample use Server Manager to create NFS share in the Azure VM. 

1. Connect to the Azure VM with user `azureuser`
2. Select Start, type servermanager.exe, and then select Server Manager.
3. In the Quick Start, click Add roles and features.
4. Installation Type: Role-based or feature-based installation.
5. Server Roles: File and Storage Service -> File and iSCSI SErvice -> check **Server for NFS**
6. Wait for the installation process finished.

Then create NFS share.

1. Open Server Manager again.
2. On the left, select File and Storage Services, and then select Shares.
3. Select To create a file share, start the New Share Wizard.
4. On the Select Profile page, select either NFS Share - Quick or NFS Share - Advanced, then select Next.
5. On the Share Location page, select a server and select **volume V**, and select Next.
6. On the Share Name page, specify a name for the new share, here is `NFSShare`, and select Next.
7. On the Authentication page, specify all the authentication method. Enable **Allow unmapped user access by UID/GID**
8. On the Share Permissions page, select All Machines and enable Allow root access.
9. In Permissions, configure the type of access control you want the users to have, and select OK.
10. On the Confirmation page, review your configuration, and select Create to create the NFS file share.

![Create NSF Share](resources/screenshot-nfsshare.png)

After the process finished, you should be able to find the file share from Windows Admin Center -> Files & file sharing.

![Create NSF Share](resources/screenshot-nfsshare2.png)

Now, we cn move on to create WebLogic cluster.




