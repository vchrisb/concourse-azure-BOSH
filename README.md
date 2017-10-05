# Concourse on Azure with BOSH

The repos contains the necessary files and steps to deploy [concourse](https://concourse.ci) to Azure using BOSH.

## requirements

* [jq](https://stedolan.github.io/jq/)
* [azure cli](https://github.com/Azure/azure-cli)

## initial setup

update `CLIENT_SECRET` with a password and `CLIENT_SECRET` with some unique string e.g. `a3c56`

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

## create storage

This will create a storage account and create necessary containers for BOSH:

```
az storage account create --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --sku Standard_LRS --kind Storage --location $LOCATION
export CONNECTION_STRING=$(az storage account show-connection-string --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP | jq -r ".connectionString")

az storage container create --name bosh --connection-string $CONNECTION_STRING
az storage container create --name stemcell --public-access blob --connection-string $CONNECTION_STRING

az storage table create --name stemcells --connection-string $CONNECTION_STRING
```

## create jumpbox ssh keys

This will deploy a Ubuntu jumpbox with BOSH Cli installed:

```
ssh-keygen -t rsa -f jumpbox -C ubuntu
```

## create azure objects

```
az group deployment create --template-file azure-deploy.json --parameters adminSSHKey="$(cat jumpbox.pub)" --parameters CustomData="$(cat cloud-init-jumpbox.txt)" --resource-group $RESOURCE_GROUP --name boshdeploy
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
bosh upload-release https://github.com/concourse/concourse/releases/download/v3.5.0/concourse-3.5.0.tgz
bosh upload-release https://github.com/concourse/concourse/releases/download/v3.5.0/garden-runc-1.6.0.tgz
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-azure-hyperv-ubuntu-trusty-go_agent
```

Finally start to deploy concourse:

```
bosh -d concourse deploy concourse-azure-BOSH/concourse.yml \
  --vars-store deployment-vars.yml \
  -v concourse_domain=<concourse_domain>
```

## Concourse

Login to concourse with cli called `fly`:

```
fly -t lite login --concourse-url https://<concourse_domain> -k login -u admin -p $(bosh int ./deployment-vars.yml --path /concourse_admin_password)
```

Deploy [sample pipelines](https://concourse.ci/hello-world.html):

```
fly -t lite set-pipeline -p hello-world -c concourse-azure-BOSH/hello.yml
fly -t lite unpause-pipeline -p hello-world
fly -t lite set-pipeline -p navi -c concourse-azure-BOSH/navi-pipeline.yml
fly -t lite unpause-pipeline -p navi
```
## clean up

To destroy all deployed ressources issue following commands:
```
az login
az group delete --name $RESOURCE_GROUP
az ad app delete --id "http://BOSHAzureCPI$IDENTIFIER"
```
