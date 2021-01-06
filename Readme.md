                                            Manage Azure Event Hubs with ASO on Kubernetes
                                                     
                                                     
In this Demo:-

How to set it up and use it to provision Azure Event Hubs
Deploy apps to Kubernetes which use the Azure Event Hubs cluster.

                                            Azure Service Operator — manage your Azure resources with Kubernetes
                                                    
What Azure Service Operator is and how it works, Link below.

https://cloudblogs.microsoft.com/opensource/2020/06/25/announcing-azure-service-operator-kubernetes/

Beside these Azure Service Operator also depends on:

Cert-manager to manage internal certificates.
Cert-manager is a Kubernetes add-on to automate the management and issuance of TLS certificates from various issuing sources.
Cert-manager is not part of the Azure Service Operator and needs to be installed upfront.

Azure AD (AAD) Pod Identity is used to manage authentication against Azure when managed identities are used.
AAD Pod Identity is part of the Azure Service Operator and is provided as a Helm subchart.

So far Azure Service Operator supports the following Azure resources:

Resource Group
Event Hubs
Azure SQL
Azure Database for PostgreSQL
Azure Database for MySQL
Azure Key Vault
Azure Cache for Redis
Storage Account
Blob Storage
Virtual Network
Application Insights
API Management
Cosmos DB
Virtual Machine
Virtual Machine Scale Set

Get started with Azure Service Operator

Pre-requisites

Start by getting an Azure account if you don’t have one already — you can get for FREE! Please make sure you’ve kubectl and Helm 3 installed as well.

Although the steps outlined in this blog should work with any Kubernetes cluster (including minikube etc.), I used Azure Kubernetes Service (AKS). You can setup a cluster using Azure CLI, Azure portal or even an ARM template. 

Once that's done, simply configure kubectl to point to it

az aks get-credentials --resource-group <CLUSTER_RESOURCE_GROUP> --name <CLUSTER_NAME>
 
                                                        Install Azure Service Operator
                                                        
Start by installing cert-manager     

https://cert-manager.io/docs/installation/kubernetes/

Steps:-

kubectl create namespace cert-manager

kubectl label namespace cert-manager cert-manager.io/disable-validation=true

kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.12.0/cert-manager.yaml

//make sure cert manager is up and running
kubectl rollout status -n cert-manager deploy/cert-manager-webhook

Authentication

Since the operator will create resource on Azure, we need to authorize it to do so by providing the appropriate credentials. Currently, you can use Managed Identity or Service Principal
I will be using a Service Principal, so let’s start by creating one (with Azure CLI) using the az ad sp create-for-rbac command

az ad sp create-for-rbac -n "aso-rbac-sp"
//JSON output
{
  "appId": "xxxx",
  "displayName": "aso-rbac-sp",
  "name": "http://aso-rbac-sp",
  "password": "xxxxx",
  "tenant": "xxxx"
}

Setup required environment variables:

export AZURE_SUBSCRIPTION_ID=<enter Azure subscription ID>
export AZURE_TENANT_ID=<enter value from the "tenant" attribute in the JSON payload above>
export AZURE_CLIENT_ID=<enter value from the "appId" attribute in the JSON payload above>
export AZURE_CLIENT_SECRET=<enter value from the "password" attribute in the JSON payload above>
export AZURE_SERVICE_OPERATOR_NAMESPACE=<name of the namespace into which ASO will be installed>

Add the repo, create namespace

helm repo add azureserviceoperator https://raw.githubusercontent.com/Azure/azure-service-operator/master/charts
kubectl create namespace $AZURE_SERVICE_OPERATOR_NAMESPACE

Use helm upgrade to initiate setup:
helm upgrade --install aso azureserviceoperator/azure-service-operator \
-n $AZURE_SERVICE_OPERATOR_NAMESPACE \
--set azureSubscriptionID=$AZURE_SUBSCRIPTION_ID \
--set azureTenantID=$AZURE_TENANT_ID \
--set azureClientID=$AZURE_CLIENT_ID \
--set azureClientSecret=$AZURE_CLIENT_SECRET

Before you proceed, wait for the Azure Service Operator Pod to startup

kubectl get pods -n $AZURE_SERVICE_OPERATOR_NAMESPACE
NAME                                              READY   STATUS    RESTARTS   AGE
azureoperator-controller-manager-68f44fd4-cm6wl   2/2     Running   0          15m

Setup Azure Event Hubs components.

Start by cloning the repo:

git clone https://github.com/akmurugan/eventhubs-using-aso-on-k8svc.git
cd eventhubs-using-aso-on-k8s

Create an Azure Resource Group

I have used the centralindia location. Please update eh-resource-group.yaml if you need to use a different one.

kubectl apply -f deploy/eh-resource-group.yaml
//confirm that its created
kubectl get resourcegroups/eh-aso-rg

Create Event Hubs namespace

I have used the centralindia location. Please update eh-namespace.yaml if you need to use a different one.

kubectl apply -f deploy/eh-namespace.yaml
//wait for creation
kubectl get eventhubnamespaces -w

Once done, you should see this:

NAME        PROVISIONED   MESSAGE
eh-aso-ns   true          successfully provisioned

You can get details with kubectl describe eventhubnamespacesand also double-check using az eventhubs namespace show

The namespace is ready, we can now create an Event Hub

kubectl apply -f deploy/eh-hub.yaml
kubectl get eventhubs/eh-aso-hub
//once done...
NAME        PROVISIONED   MESSAGE
eh-aso-hub  true          successfully provisioned

You can get details with kubectl describe eventhub and also double-check using az eventhubs eventhub show

This is addition to the default consumer group (appropriately named $Default)

kubectl apply -f deploy/eh-consumer-group.yaml
kubectl get consumergroups/eh-aso-cg
NAME        PROVISIONED   MESSAGE
eh-aso-cg  true          successfully provisioned

You can get details with kubectl describe consumergroup

Next Steps Consumer & Producer deployment

consumer app:

kubectl apply -f deploy/consumer.yaml
//wait for it to start
kubectl get pods -l=app=eh-consumer -w

Keep a track of the logs for the consumer app:

kubectl logs -f $(kubectl get pods -l=app=eh-consumer --output=jsonpath={.items..metadata.name})

You should see something similar to:

Event Hubs broker [eh-aso-ns.servicebus.windows.net:9093]
Sarama client consumer group ID eh-aso-cg
new consumer group created
Event Hubs topic eh-aso-hub
Waiting for program to exit
Partition allocation - map[eh-aso-hub:[0 1 2]]


Using another terminal, deploy the producer app:

kubectl apply -f deploy/producer.yaml

Once the producer app is up and running, the consumer should kick in, start consumer the messages and print them to the console. So you’ll see logs similar to this:

...
Message topic:"eh-aso-hub" partition:0 offset:6
Message content value-2020-07-06 15:37:06.116674866 +0000 UTC m=+67.450171692
Message topic:"eh-aso-hub" partition:0 offset:7
Message content value-2020-07-06 15:37:09.133115988 +0000 UTC m=+70.466612714
Message topic:"eh-aso-hub" partition:0 offset:8
Message content value-2020-07-06 15:37:12.149068005 +0000 UTC m=+73.482564831

In case you want to check producer logs as well: kubectl logs -f $(kubectl get pods -l=app=eh-producer --output=jsonpath={.items..metadata.name})

Awesome, it worked!

We created an Event Hubs namespace, Event Hub along and a consumer group.. all using kubectl (and YAMLs of course)

Deployed a simple producer and consumer for testing







