# Velero-In-AKS
Velero is an open source tool that helps backup and restore Kubernetes resources. It also helps with migrating Kubernetes resources from one cluster to another. Also, it can help backup/restore data in persistent volumes

`1) Login to Portal.azure.com and open Cloud shell `

`2) az login`

## Setup AKS Cluster
#### A) Create Resource Group 
```
Velero_Resource_Group="aks-test"
region=eastus
az group create --location $region --name $Velero_Resource_Group
```
#### B) Create AKS Cluster
```
az aks get-versions -l $region
AKS_Cluster_Name="aks-cluster-1"
az aks create --resource-group $Velero_Resource_Group --name $AKS_Cluster_Name --node-count 1 --kubernetes-version 1.19.11
```
#### C) Verify the AKS Cluster which we created
```
az aks show --name $AKS_Cluster_Name --resource-group $Velero_Resource_Group
```
#### D) Configure kubeconfig
```
az aks get-credentials --resource-group $Velero_Resource_Group --name $AKS_Cluster_Name
kubectl get nodes 
```
## Create Storage account
```
Velero_Storage_Account="testnewakstorageaditya"
Velero_SA_blob_Container="testakscontaineraditya"
az storage account create --name $Velero_Storage_Account --resource-group $Velero_Resource_Group --location $region --kind StorageV2 --sku Standard_LRS --encryption-services blob --https-only true --access-tier Hot
az storage container create --name $Velero_SA_blob_Container --public-access off --account-name $Velero_Storage_Account
```
## Velero Installation

#### - In Azure CLoud Shell
 ```
 wget https://github.com/vmware-tanzu/velero/releases/download/v1.7.0/velero-v1.7.0-linux-amd64.tar.gz
 tar -xvf velero-v1.7.0-linux-amd64.tar.gz
 Move the extracted velero binary to somewhere in your $PATH (/usr/local/bin for most users). [ Skip this in Cloud Shell ]
  ```
#### Install Velero Server in AKS Cluster
```
AZURE_SUBSCRIPTION_ID=`az account list --query '[?isDefault].id' -o tsv`
AZURE_TENANT_ID=`az account list --query '[?isDefault].tenantId' -o tsv`
AZURE_CLIENT_SECRET=$(az ad sp create-for-rbac -n $Velero_Storage_Account --role contributor --query password --output tsv)
AZURE_AKS_RESOURCE_GROUP=$(az aks show --query nodeResourceGroup --name $AKS_Cluster_Name --resource-group $Velero_Resource_Group --output tsv)
AZURE_CLIENT_ID=$(az ad sp show --id http://$Velero_Storage_Account --query appId --output tsv)

```
##### NOTE : In Azure change this Above AZURE_CLIENT_ID with this : Azure AD > APP Registrations > All Applications > Storage Account > Copy that Application Client ID and Paste it in above AZURE_CLIENT_ID
```
cat << EOF > ./credentials-velero
AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID
AZURE_TENANT_ID=$AZURE_TENANT_ID
AZURE_CLIENT_ID=$AZURE_CLIENT_ID
AZURE_CLIENT_SECRET=$AZURE_CLIENT_SECRET
AZURE_RESOURCE_GROUP=$AZURE_AKS_RESOURCE_GROUP
AZURE_CLOUD_NAME=AzurePublicCloud
EOF
```
```
In Cloud Shell COPY credentials-velero and paste to velero-v1.7.0-linux-amd64 & go to velero-v1.7.0-linux-amd64 and use velero like ./velero 
```
```
./velero install --provider azure \ 
--plugins velero/velero-plugin-for-microsoft-azure:v1.0.1 \
--bucket $Velero_SA_blob_Container \ 
--secret-file ./credentials-velero \ 
--backup-location-config resourceGroup=$Velero_Resource_Group,storageAccount=$Velero_Storage_Account,subscriptionId=$AZURE_SUBSCRIPTION_ID \ 
--snapshot-location-config apiTimeout=5m,resourceGroup=$Velero_Resource_Group,subscriptionId=$AZURE_SUBSCRIPTION_ID
```
```
kubectl get all -n velero
```
#### Now velero is Installed and running So now deploy some your deployments & Pods

### Backup
```
./velero backup create newbackup
./velero get backup
./velero backup describe newbackup
./velero backup logs newbackup
```
### Restore 
```
./velero restore create --from-backup newbackup
```
### NOTE : If want to schedule a backup every 24 hrs then use below command.
```
./velero create schedule nginx-test  --schedule ="@every 24h"
```
##### References : 
- https://velero.io/docs/v1.7/basic-install/
- https://stackoverflow.com/questions/68899098/access-azure-active-directory-sso-from-an-app-outside-the-tenant
- https://github.com/Nagendran2807/Velero-k8s/tree/main/Azure-AKS
- https://www.youtube.com/watch?v=WMrBs2JN57k&t=1669s

Thanks & Regards

Aditya Mandil
