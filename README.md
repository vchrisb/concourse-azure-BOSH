# Concourse on Azure with BOSH

The repos contains the necessary files and steps to deploy [concourse](https://concourse.ci) to Azure using BOSH.

## requirements

* [jq](https://stedolan.github.io/jq/)
* [azure cli](https://github.com/Azure/azure-cli)

## initial setup

```
az cloud set --name AzureCloud
az login

export CLIENT_SECRET="<CLIENT_SECRET>"
export IDENTIFIER="<IDENTIFIER>"
export SUBSCRIPTION_ID=$(az account list | jq -r ".[0].id")
export TENANT_ID=$(az account list | jq -r ".[0].tenantId")
export RESOURCE_GROUP="bosh_resource_group"
export LOCATION="westeurope"
export STORAGE_ACCOUNT="boshstorage$IDENTIFIER"

az account set --subscription $SUBSCRIPTION_ID
```

### create a service principal
```
az ad app create --display-name "Service Principal for BOSH" --password $CLIENT_SECRET --homepage "http://BOSHAzureCPI" --identifier-uris "http://BOSHAzureCPI$IDENTIFIER"

export CLIENT_ID=$(az ad app show --id "http://BOSHAzureCPI$IDENTIFIER" | jq -r ".appId")

az ad sp create --id $CLIENT_ID
az role assignment create --assignee "http://BOSHAzureCPI$IDENTIFIER" --role "Contributor" --scope "/subscriptions/$SUBSCRIPTION_ID"
```

### create resource group
```
az login --username $CLIENT_ID --password $CLIENT_SECRET --service-principal --tenant $TENANT_ID

az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Compute

az group create --name $RESOURCE_GROUP --location $LOCATION
```

## create networks

This will create the necessary network security groups and network objects:

```
az network nsg create --name nsg-bosh --resource-group $RESOURCE_GROUP --location $LOCATION
az network nsg rule create --name ssh --nsg-name nsg-bosh --resource-group $RESOURCE_GROUP --protocol Tcp --priority 300 --source-address-prefix Internet --source-port-range "*" --destination-address-prefix "*" --destination-port-range 22
az network nsg rule create --name https-concourse --nsg-name nsg-bosh --resource-group $RESOURCE_GROUP --protocol Tcp --priority 400 --source-address-prefix Internet --source-port-range "*" --destination-address-prefix "*" --destination-port-range 4443

az network vnet create --name boshnet --resource-group $RESOURCE_GROUP --location $LOCATION --address-prefixes 10.0.0.0/16
az network vnet subnet create --name boshsub --vnet-name boshnet --resource-group $RESOURCE_GROUP --address-prefix 10.0.0.0/24 --network-security-group nsg-bosh
```

## create load balancer

This will create a Azure Loadbalancer, which will later be consumed by concourse:

```
az network lb create --name concourse-lb --resource-group $RESOURCE_GROUP --location $LOCATION --backend-pool-name concourse-web-vms --frontend-ip-name concourse-fe-ip --public-ip-address concourse-lb-ip --public-ip-address-allocation static
az network lb probe create --lb-name concourse-lb --name tcp4443 --resource-group $RESOURCE_GROUP --protocol Tcp --port 4443
az network lb rule create --lb-name concourse-lb --name https --resource-group $RESOURCE_GROUP --protocol Tcp --frontend-port 443 --backend-port 4443 --frontend-ip-name concourse-fe-ip --backend-pool-name concourse-web-vms
```

## create storage

This will create a storage account and create necessary containers for BOSH:

```
az storage account create --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --sku Standard_LRS --kind Storage --location $LOCATION
export CONNECTION_STRING=$(az storage account show-connection-string --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP | jq -r ".connectionString")

az storage container create --name bosh --connection-string $CONNECTION_STRING
az storage container create --name stemcell --public-access blob --connection-string $CONNECTION_STRING

az storage table create --name stemcells --connection-string $CONNECTION_STRING
```

## create jumpbox

This will deploy a Ubuntu jumpbox with BOSH Cli installed:

```
ssh-keygen -t rsa -f jumpbox -C ubuntu

az vm create --resource-group $RESOURCE_GROUP --name jumpbox --image UbuntuLTS --vnet-name boshnet --subnet boshsub --nsg "" --size Standard_DS2_v2 --private-ip-address 10.0.0.5 --public-ip-address-dns-name "jumpbox$IDENTIFIER" --os-disk-name jumpboxOSdisk --admin-username ubuntu --ssh-key-value ./jumpbox.pub --custom-data cloud-init-jumpbox.txt --public-ip-address-allocation static
```

## bootstrap BOSH Director

Issue following command on your local machine to generate the `bosh create-env` command:

```
echo "bosh create-env bosh-deployment/bosh.yml \\
    --state=state.json \\
    --vars-store=creds.yml \\
    -o bosh-deployment/azure/cpi.yml \\
    -v director_name=bosh-1 \\
    -v internal_cidr=10.0.0.0/24 \\
    -v internal_gw=10.0.0.1 \\
    -v internal_ip=10.0.0.6 \\
    -v vnet_name=boshnet \\
    -v subnet_name=boshsub \\
    -v subscription_id=$SUBSCRIPTION_ID \\
    -v tenant_id=$TENANT_ID \\
    -v client_id=$CLIENT_ID \\
    -v client_secret=\"$CLIENT_SECRET\" \\ 
    -v resource_group_name=$RESOURCE_GROUP \\
    -v storage_account_name=$STORAGE_ACCOUNT \\
    -v default_security_group=nsg-bosh"
```

Login to the jumpbox and prepare the bosh deployment:

```
ssh -i jumpbox ubuntu@<jumpbox fqdn>

mkdir bosh-1 && cd bosh-1

git clone https://github.com/cloudfoundry/bosh-deployment
git clone https://github.com/vchrisb/concourse-azure-BOSH
```

Paste the `bosh create-env` command from above to bootstrap the BOSH director!

Login to the BOSH director and upload the cloud-config:

```	
bosh alias-env bosh-1 -e 10.0.0.6 --ca-cert <(bosh int ./creds.yml --path /director_ssl/ca)
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET=`bosh int ./creds.yml --path /admin_password`
export BOSH_ENVIRONMENT=bosh-1

bosh update-cloud-config concourse-azure-BOSH/cloud-config.yml \
  -v internal_cidr=10.0.0.0/24 \
  -v internal_gw=10.0.0.1 \
  -v vnet_name=boshnet \
  -v subnet_name=boshsub \
  -v security_group=nsg-bosh \
  -v concourse_load_balancer=concourse-lb
```

## deploy concourse

Upload the concourse release and latest stemcell:

```
bosh upload-release https://github.com/concourse/concourse/releases/download/v3.3.4/concourse-3.3.4.tgz
bosh upload-stemcell https://s3.amazonaws.com/bosh-core-stemcells/azure/bosh-stemcell-3431.10-azure-hyperv-ubuntu-trusty-go_agent.tgz
bosh upload-release https://bosh.io/d/github.com/cloudfoundry/garden-runc-release?v=1.9.0 --sha1 77bfe8bdb2c3daec5b40f5116a6216badabd196c
```

Finally start to deploy concourse:

```
bosh -d concourse deploy concourse-azure-BOSH/concourse.yml \
  --vars-store deployment-vars.yml \
  -v concourse_domain=<concourse_domain>
```

## Concourse

Install concourse cli called `fly`:
```
wget  https://github.com/concourse/concourse/releases/download/v3.3.4/fly_linux_amd64 -O fly
chmod +x fly
sudo mv fly /usr/local/bin/
```

Login :

```
fly -t lite login --concourse-url https://<concourse_domain> -k login -u admin -p $(bosh int ./deployment-vars.yml --path /concourse_admin_password)
```

Deploy [sample pipelines-6(https://concourse.ci/hello-world.html):

```
fly -t lite set-pipeline -p hello-world -c concourse-azure-BOSH/hello.yml
fly -t lite unpause-pipeline -p hello-world
fly -t lite set-pipeline -p hello-world -c oncourse-azure-BOSH/navi-pipeline.yml
```