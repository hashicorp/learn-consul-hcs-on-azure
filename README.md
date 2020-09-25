# Deploy Production Workloads with Managed Consul and Kubernetes Take Home Lab

## Prerequisites

- Your own Azure subscription
- The following binaries installed on the development host
  - jq
  - kubectl 1.18.0+
  - helm 3.2.1+
  - Azure CLI
  - [Consul 1.8.x](https://www.consul.io/downloads)

## Download the Helm repository

`helm repo add hashicorp https://helm.releases.hashicorp.com && helm repo update`

## Install the HCS Azure CLI extension

`az extension add --source https://releases.hashicorp.com/hcs/0.3.0/hcs-0.3.0-py2.py3-none-any.whl`

## Login to Azure

`az login`

## Set an environment variable for your resource group name

`export RESOURCE_GROUP=<your-resource-group-name> && echo $RESOURCE_GROUP`

## Create a resource group

`az group create -l westus2 -n $RESOURCE_GROUP`

## Create an AKS cluster (this may take up to 10 minutes)

`az aks create -g $RESOURCE_GROUP -n $RESOURCE_GROUP-aks`

## Create an HCS Datacenter (This may take up to 15 minutes)

`az hcs create -g $RESOURCE_GROUP --name $RESOURCE_GROUP-managed-hcs --datacenter-name dc1 --email your@email.com --external-endpoint enabled`

## Create a VNet

`az network vnet create -g $RESOURCE_GROUP -n $RESOURCE_GROUP-vnet`

## Confirm your resources

`az resource list --resource-group $RESOURCE_GROUP -o table`

## Set an environment variable to the name of your AKS cluster

`export AKS_CLUSTER=$(az aks list --resource-group $RESOURCE_GROUP | jq -r '.[] | .name') && echo $AKS_CLUSTER`

## Set an environment variable to the name of your HCS managed app

`export HCS_MANAGED_APP=$(az hcs list --resource-group $RESOURCE_GROUP | jq -r '.[] | .name') && echo $HCS_MANAGED_APP`

## Set an environment variable to the name of your HCS managed app's resource group

`export HCS_MANAGED_RESOURCE_GROUP=${$(az hcs list --resource-group $RESOURCE_GROUP | jq -r '.[] | .managedResourceGroupId')##*/} && echo $HCS_MANAGED_RESOURCE_GROUP`

## Add remote AKS kubeconfig to local kubconfig

`az aks get-credentials --name $AKS_CLUSTER --resource-group $RESOURCE_GROUP`

## Bootstrap ACLs and store the token as a Kubernetes secret

`az hcs create-token --name $HCS_MANAGED_APP --resource-group $RESOURCE_GROUP --output-kubernetes-secret | kubectl apply -f -`

## Generate Consul key/cert and store as a Kubernetes secret

`az hcs generate-kubernetes-secret --name $HCS_MANAGED_APP --resource-group $RESOURCE_GROUP | kubectl apply -f -`

## Export the config file to pass to helm during install

`az hcs generate-helm-values --name $HCS_MANAGED_APP --resource-group $RESOURCE_GROUP --aks-cluster-name $AKS_CLUSTER > config.yaml`

## Enable the AKS specific setting `exposeGossipPorts`

`sed -i -e 's/^  # \(exposeGossipPorts\)/  \1/' config.yaml`

## Configure the development host to talk to the public endpoint

`export CONSUL_HTTP_ADDR=$(az hcs show --name $HCS_MANAGED_APP --resource-group $RESOURCE_GROUP | jq -r .properties.consulExternalEndpointUrl) && echo $CONSUL_HTTP_ADDR`

## Set the CONSUL_HTTP_TOKEN on the development host to authorize the CLI

`export CONSUL_HTTP_TOKEN=$(kubectl get secret $HCS_MANAGED_APP-bootstrap-token -o jsonpath={.data.token} | base64 -d) && echo $CONSUL_HTTP_TOKEN`

## Set the CONSUL_HTTP_SSL_VERIFY flag to false on the development host

`export CONSUL_HTTP_SSL_VERIFY=false && echo $CONSUL_HTTP_SSL_VERIFY`

## Verify that the development host can see the Consul servers

`consul members`

## Create a peering from the HCS Datacenter's vnet to the AKS Cluster's vnet

```shell-session
az network vnet peering create \
  -g $HCS_MANAGED_RESOURCE_GROUP \
  -n hcs-to-aks \
  --vnet-name $(az network vnet list \
    --resource-group $HCS_MANAGED_RESOURCE_GROUP | jq -r '.[0].name') \
  --remote-vnet $(az network vnet list \
    --resource-group $RESOURCE_GROUP | jq -r '.[0].id') \
  --allow-vnet-access
```

## Create a peering from the AKS Cluster's vnet to the HCS Datacenter's vnet

```shell-session
az network vnet peering create \
  -g $RESOURCE_GROUP \
  -n aks-to-hcs \
  --vnet-name $(az network vnet list \
    --resource-group $RESOURCE_GROUP | jq -r '.[0].name') \
  --remote-vnet $(az network vnet list \
    --resource-group $HCS_MANAGED_RESOURCE_GROUP | jq -r '.[0].id') \
  --allow-vnet-access
```

## Install the Consul clients to the AKS Cluster

`helm install hcs hashicorp/consul -f config.yaml --wait`

## Verify the installation

`consul members`

## Deploy the application to AKS

`kubectl apply -f hashicups/ --wait`

## Create a config entry for an ingress gateway

`consul config write hashicups/ingress-gateway.hcl`

## Add the ingress gateway to the helm configuration file

```shell-session
sudo tee -a ./config.yaml <<EOF
ingressGateways:
  enabled: true
  defaults:
    replicas: 1
  gateways:
    - name: ingress-gateway
      service:
        type: LoadBalancer
EOF
```

## Upgrade the installation to deploy the ingress gateway

`helm upgrade -f ./config.yaml hcs hashicorp/consul --wait`

## Create all necessary inter-service intentions

```shell-session
consul intention create ingress-gateway frontend && \
consul intention create frontend public-api && \
consul intention create public-api products-api && \
consul intention create products-api postgres
```

## Retrieve the public IP/Port for the ingress gateway

`kubectl get svc`
