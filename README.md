# Multi-tiered expense application

## Overview

This blog demonstrates a multi-tier application deployment on to Azure Kubernetes Service along with several other Azure managed services such as Azure Database for MySQL, Azure Functions, etc.
Note: There may be few features that are used in this blog such as Azure Active Directory Pod Identity are still in preview, these features are not recommended for production deployment.

## Architecture

![Architecture](Architecture.jpg)

## Setup

We will create and setup the infrastructure including the following services:

1. Azure Container Registry for storing images
2. AAD Enabled, Managed AKS Cluster with the below addons and components
   1. Application Gateway Ingress Controller Addon
   2. Monitoring Addon
   3. LetsEncrypt for Certificate authority
   4. KEDA runtime for Azure Functions on Kubernetes clusters
3. Azure Database for MySQL Service
4. Azure Storage Queues
5. DNS Zone for custom domain
6. SendGrid Account for email service

### Cluster Creation

#### Clone the repo

```bash
git clone https://github.com/ssarwa/multitiered-app-on-azure
cd multitiered-app-on-azure

# You could use script.azcli as your working file. Don't run the script as is!
```

#### Initialize variables

```bash
# Add variables (sample values below change as required)
resourcegroupName='CNCF-Azure-RG'
clusterName='myaksCluster'
location='westus'
appGtwyName='AKSAppGtwy'
acrName='cncfazure'
domainName='sarwascloud.com'
dnsRG='dns-rg'
subDomain='expense'
mysqlSvr='expensedbserver'
adminUser='expenseadmin'
mysqlPwd=''
keyvaultName='expensesvault'
# Must be lower case
identityName='exppoidentity'
# Must be globally unique
storageAcc='expensesqueue'
subscriptionId='12bb4e89-4f7a-41e0-a38f-b22f079248b4'
tenantId='72f988bf-86f1-41af-91ab-2d7cd011db47'
```

#### Login to Azure

```bash
az login

az account set -s $subscriptionId
```

#### Register to AKS preview features

```bash
# Follow https://docs.microsoft.com/en-us/azure/aks/use-azure-ad-pod-identity
az feature register --name EnablePodIdentityPreview --namespace Microsoft.ContainerService
az provider register -n Microsoft.ContainerService
az extension add --name aks-preview
az extension update --name aks-preview
```

#### Create Resource Group

```bash
az group create --name $resourcegroupName --location $location
```

#### Create ACR

```bash
az acr create --resource-group $resourcegroupName --name $acrName --sku Standard
```

#### Get Object ID of the AAD Group (create AAD Group and add the members, in this case: expenses-ad-azure)
This group is needed as admin group for the cluster to grant cluster admin permissions. You can use an existing Azure AD group, or create a new one. Record the object ID of your Azure AD group.
```bash
az ad group create --display-name expenses-ad-azure --mail-nickname expenses-ad-azure --description 'Group for Managing AAD based AKS cluster'
objectId=$(az ad group list --filter "displayname eq 'expenses-ad-azure'" --query '[].objectId' -o tsv)
```

#### Create an AKS-managed Azure AD cluster with AGIC addon and AAD Pod Identity

```bash
az aks create \
    -n $clusterName \
    -g $resourcegroupName \
    --network-plugin azure \
    --enable-managed-identity \
    -a ingress-appgw --appgw-name $appGtwyName \
    --appgw-subnet-cidr "10.2.0.0/16" \
    --enable-aad \
    --enable-pod-identity \
    --aad-admin-group-object-ids $objectId \
    --generate-ssh-keys \
    --attach-acr $acrName

# Enable monitoring on the cluster
az aks enable-addons -a monitoring -n $clusterName -g $resourcegroupName
```

#### Add Public IP to custom domain

```bash
# Get Node Resource Group
nodeRG=$(az aks show --resource-group $resourcegroupName --name $clusterName --query nodeResourceGroup -o tsv)

# Get Public IP created by App Gtwy in AKS created cluster
appIP=$(az network public-ip show -g $nodeRG -n $appGtwyName-appgwpip --query ipAddress -o tsv)

# Create DNS zone, if not created
az network dns zone create -g $dnsRG -n $domainName

# Once created, add Nameservers in the domain provider (eg go daddy, may take sometime to update the name servers)
az network dns record-set a add-record --resource-group $resourcegroupName --zone-name $domainName --record-set-name $subDomain --ipv4-address $appIP

# For more details see the tutorial on registering custom domain: https://docs.microsoft.com/en-us/azure/dns/dns-delegate-domain-azure-dns
```

### Connect to the Cluster

#### Merge Kubeconfig

```bash
az aks get-credentials --resource-group $resourcegroupName --name $clusterName --admin
```

#### Install Cert Manager

```bash
# Install the CustomResourceDefinition resources separately
# Note: --validate=false is required per https://github.com/jetstack/cert-manager/issues/2208#issuecomment-541311021
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.13/deploy/manifests/00-crds.yaml --validate=false

kubectl create namespace cert-manager
kubectl label namespace cert-manager cert-manager.io/disable-validation=true
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager --namespace cert-manager --version v0.13.0 jetstack/cert-manager
kubectl apply -f yml/clusterissuer.yaml

# Test a sample application. The below command will deploy a Pod, Service and Ingress resource. Application Gateway will be configured with the associated rules.
sed -i "s/<custom domain name>/$domainName/g" yml/Test-App-Ingress.yaml
sed -i "s/<Sub Domain>/$subDomain/g" yml/Test-App-Ingress.yaml
kubectl apply -f yml/Test-App-Ingress.yaml

# Clean up after successfully verifying AGIC
kubectl delete -f yml/Test-App-Ingress.yaml
```

#### Install KEDA runtime

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda
```

#### Install CSI Provider for Azure Keyvault

```bash
helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
helm repo update
kubectl create namespace csi
helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --namespace csi
```

#### Assign managed identity

```bash
clientId=$(az aks show -n $clusterName -g $resourcegroupName --query identityProfile.kubeletidentity.clientId -o tsv)

scope=$(az group show -g $nodeRG --query id -o tsv)

az role assignment create --role "Managed Identity Operator" --assignee $clientId --scope $scope
```

#### Create Azure Keyvault for saving secrets and assign identity

```bash
az keyvault create --location $location --name $keyvaultName --resource-group $resourcegroupName

kvscope=$(az keyvault show -g $resourcegroupName -n $keyvaultName --query id -o tsv)

az identity create -g $resourcegroupName -n $identityName

idClientid=$(az identity show -n $identityName -g $resourcegroupName --query clientId -o tsv)

idPrincipalid=$(az identity show -n $identityName -g $resourcegroupName --query principalId -o tsv)

identityId=$(az identity show -n $identityName -g $resourcegroupName --query id -o tsv)

az role assignment create --role "Reader" --assignee $idPrincipalid --scope $kvscope

az keyvault set-policy -n $keyvaultName --secret-permissions get --spn $idClientid

az aks pod-identity add --resource-group $resourcegroupName --cluster-name $clusterName --namespace default --name $identityName --identity-resource-id $identityId
```

#### Create MySQL managed service (basic sku) and add Kubernetes Load Balancer's public ip in it firewall rules

```bash
aksPublicIpName=$(az network lb show -n kubernetes -g $nodeRG --query "frontendIpConfigurations[0].name" -o tsv)
aksPublicIpAddress=$(az network public-ip show -n $aksPublicIpName -g $nodeRG --query ipAddress -o tsv)
az mysql server create --resource-group $resourcegroupName --name $mysqlSvr --location $location --admin-user $adminUser --admin-password $mysqlPwd --sku-name B_Gen5_2
az mysql server firewall-rule create --name allowip --resource-group $resourcegroupName --server-name $mysqlSvr --start-ip-address $aksPublicIpAddress --end-ip-address $aksPublicIpAddress

# Replace <Dev station ip> with your Local Machine IP. You can use: https://www.whatsmyip.org/
az mysql server firewall-rule create --name devbox --resource-group $resourcegroupName --server-name $mysqlSvr --start-ip-address <Dev station ip> --end-ip-address <Dev station ip>
```

#### Login to MySQL (you may need to add you ip to firewall rules as well)

Install MySQL Client your your local dev box: sudo apt install mysql-client-core-8.0

```bash
mysql -h $mysqlSvr.mysql.database.azure.com -u $adminUser@$mysqlSvr -p
show databases;

CREATE DATABASE conexpweb;

CREATE DATABASE conexpapi;
USE conexpapi;

CREATE TABLE CostCenters(
   CostCenterId int(11)  NOT NULL,
   SubmitterEmail text NOT NULL,
   ApproverEmail text NOT NULL,
   CostCenterName text NOT NULL,
   PRIMARY KEY ( CostCenterId )
);

# Insert example records
INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (1, 'submitter@email.com', 'approver@email.com','123E42');
INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (2, 'submitter@email.com', 'approver@email.com','456C14');
INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (3, 'submitter@email.com', 'approver@email.com','789A28');

# Verify records
SELECT * FROM CostCenters;
quit
```

#### Create Storage queue

```bash
az storage account create -n $storageAcc -g $resourcegroupName -l $location --sku Standard_LRS

# Do not change queue name of contosoexpenses. KedaFunction queue trigger relies on this queue name.
az storage queue create -n contosoexpenses --account-name $storageAcc
```

#### Add corresponding secrets to the create KeyVault

1. MySQL Connection strings (choose ADO.NET) - both for API and Web
   1. mysqlconnapi
   2. mysqlconnweb
2. Storage Connection strings
   1. storageconn
3. Sendgrid Key
   1. sendgridapi

```bash
az keyvault secret set --vault-name $keyvaultName --name mysqlconnapi --value '<replace>Connection strings for MySQL API connection</replace>'
az keyvault secret set --vault-name $keyvaultName --name mysqlconnweb --value '<replace>Connection strings for MySQL Web connection</replace>'
az keyvault secret set --vault-name $keyvaultName --name storageconn --value '<replace>Connection strings for Storage account</replace>'
# Make sure to register for free SendGrid Account and verify identity. [Visit Sendgrid](https://sendgrid.com).
az keyvault secret set --vault-name $keyvaultName --name sendgridapi --value '<replace>Sendgrid Key</replace>'
az keyvault secret set --vault-name $keyvaultName --name funcruntime --value 'dotnet'
```

#### Application Deployment

```bash
registryHost=$(az acr show -n $acrName --query loginServer -o tsv)

az acr login -n $acrName

cd source/Contoso.Expenses.API
docker build -t $registryHost/conexp/api:latest .
docker push $registryHost/conexp/api:latest

cd ..
docker build -t $registryHost/conexp/web:latest -f Contoso.Expenses.Web/Dockerfile .
docker push $registryHost/conexp/web:latest

docker build -t $registryHost/conexp/emaildispatcher:latest -f Contoso.Expenses.KedaFunctions/Dockerfile .
docker push $registryHost/conexp/emaildispatcher:latest

cd ..

# Update yamls files and change identity name, keyvault name, queue name and image used refer values between <> in all files
# Create CSI Provider Class
# Use gsed for MacOS
sed -i "s/<Tenant ID>/$tenantId/g" yml/csi-sync.yaml
sed -i "s/<Cluster RG Name>/$resourcegroupName/g" yml/csi-sync.yaml
sed -i "s/<Subscription ID>/$subscriptionId/g" yml/csi-sync.yaml
sed -i "s/<Keyvault Name>/$keyvaultName/g" yml/csi-sync.yaml
kubectl apply -f yml/csi-sync.yaml

# Create API
sed -i "s/<identity name created>/$identityName/g" yml/backend.yaml
sed -i "s/<Backend image built>/$registryHost\/conexp\/api:latest/g" yml/backend.yaml
sed -i "s/<Keyvault Name>/$keyvaultName/g" yml/backend.yaml
kubectl apply -f yml/backend.yaml

# Create frontend
sed -i "s/<identity name created>/$identityName/g" yml/frontend.yaml
sed -i "s/<frontend image built>/$registryHost\/conexp\/web:latest/g" yml/frontend.yaml
sed -i "s/<Keyvault Name>/$keyvaultName/g" yml/frontend.yaml
kubectl apply -f yml/frontend.yaml

# Create ingress resource
sed -i "s/<custom domain name>/$domainName/g" yml/ingress.yaml
sed -i "s/<Sub Domain>/$subDomain/g" yml/ingress.yaml
kubectl apply -f yml/ingress.yaml

# Create KEDA function
sed -i "s/<identity name created>/$identityName/g" yml/function.yaml
sed -i "s/<function image built>/$registryHost\/conexp\/emaildispatcher:latest/g" yml/function.yaml
kubectl apply -f yml/function.yaml
```

Once the ingress controller updates with new frontend deployed it may take a min for Application gateway to update.
