# kubemalt - Spring Boot Application Demo

### Simple Application to test KubeMALT components

Deploy a spring boot based application to a KubeMALT supporting cluster.

Application is built from the [CatalystCode/containers-rest-cosmos-appservice-java Project](https://github.com/CatalystCode/containers-rest-cosmos-appservice-java)

These steps were cobbled together from Walkthrough documents [here](https://github.com/Microsoft/containers-rest-cosmos-appservice-java/issues/59) and adjusted for kubernetes deployments.

## Steps

### Create MALT Kubernetes Cluster
Deploy infrastructure through [~~bedrock~~](https://github.com/Microsoft/bedrock) and deploy MALT stack through [Fabrikate](https://github.com/Microsoft/fabrikate).

Currently using [Cloud Native Stack](https://github.com/timfpark/fabrikate-cloud-native/) from [Fabrikate Components](https://github.com/CatalystCode/kubemalt/wiki/Fabrikate-Components). Deployment scripts can be found in the [Wiki](https://github.com/CatalystCode/kubemalt/wiki/Various-helpful-docs-and-bash-scripts-for-Kubernetes-and-Docker-deployments#fabrikate-deployment-commands).

Recommended number of AKS nodes are 7 or 8.


### Configure `kubectl` to point to above cluster
Through [kubectx & kubens](https://github.com/ahmetb/kubectx) or for Azure:
```
az login
az account set --subscription <subscription-guid>
az aks get-credentials --resource-group <resource-group-name> --name <kubernetes-cluster-name>
```
For the last command put the names of the resource group and cluster, not uuid/guid.


### Create MongoDB

These are condensed steps taken from the [source repo here](https://github.com/Microsoft/containers-rest-cosmos-appservice-java/tree/master/infrastructure/global-resources).

Replace the `<changeme>` values and then execute this script.

```bash
export RESOURCE_GROUP=<changeme>
export COSMOSDB_NAME=<changeme>
export LOCATION=<changeme>
az group create -n ${RESOURCE_GROUP} -l ${LOCATION}
az cosmosdb create -n ${COSMOSDB_NAME} -g ${RESOURCE_GROUP} --kind MongoDB
export DB_CONNSTR=$(az cosmosdb list-connection-strings -n ${COSMOSDB_NAME} -g ${RESOURCE_GROUP} -o tsv --query connectionStrings[0].connectionString)

kubectl create secret generic jackson-secrets \
--from-literal=DB_CONNSTR=${DB_CONNSTR} \
--from-literal=DB_NAME=${COSMOSDB_NAME} \
--from-literal=OAUTH_KEYSET_URI='https://login.microsoftonline.com/common/discovery/keys' \
--from-literal=OAUTH_RES_ID='notused'
```

#### Load the data to CosmosDB

```bash
export COSMOSDB_PASSWORD=$(az cosmosdb list-keys -n ${COSMOSDB_NAME} -g ${RESOURCE_GROUP} -o tsv --query primaryMasterKey)
echo $COSMOSDB_PASSWORD
# This downloads fairly quickly the datasets, but takes a LONG time to scrub the data of the \N.  In fact the data scrub seemed to hang for me.
curl https://raw.githubusercontent.com/Microsoft/containers-rest-cosmos-appservice-java/master/data/getdata.sh >getdata.sh
sudo chmod +x ./getdata.sh
./getdata.sh
# Import of the data works even if the scrub hasn't finished as long as the TSV files are present.  This also takes a LONG time to upload ~2+GB data
curl https://raw.githubusercontent.com/Microsoft/containers-rest-cosmos-appservice-java/master/data/importdata.sh >importdata.sh
sudo chmod +x ./importdata.sh
./importdata.sh
```

**NOTE** - For the "custom" api endpoints to work, you will need to enable the Preview Feature: `Aggregation Pipeline` in CosmosDB. [Link](https://azure.microsoft.com/en-us/blog/azure-cosmosdb-extends-support-for-mongodb-aggregation-pipeline-unique-indexes-and-more/)

### Create and deploy kube pod with Secrets (Keyvault TBD)
Run the below script with the proper subsitutions for DB and AzureAD components.
```
kubectl create secret generic jackson-secrets \
--from-literal=DB_CONNSTR='<MongoDB Connection String>' \
--from-literal=DB_NAME='<MongoDB Database Name>' \
--from-literal=OAUTH_KEYSET_URI='https://login.microsoftonline.com/common/discovery/keys' \
--from-literal=OAUTH_RES_ID='<AzureAD App ID URI>'
```

TBD - Create Keyvault solution with various secrets and env vars:
- Repository URL
- Various AAD Security env vars
- UI env vars (if we get dynamic injection supported)

### Install helm and add Istio

```
helm init
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com
helm repo update
```
https://github.com/helm/charts/tree/master/incubator/istio#tldr

### Deploy manifests to cluster
`kubectl apply --recursive -f .`

### Test deployment by visiting site
Visit the Istio Ingress Gateway External-IP: `kubectl get svc -n istio-system`

### Monitoring Endpoints
```
kubectl port-forward -n kibana svc/kibana 5601:443
```
[Kibana](http://localhost:5601/)
```
kubectl port-forward -n prometheus svc/prometheus-server 9090:80
```
[Prometheus](http://localhost:9090/)
```
kubectl port-forward -n jaeger svc/jaeger-query 4000:80
```
[Jaeger](http://localhost:4000/)
```
kubectl port-forward -n grafana svc/grafana 3000:80
kubectl get secret -n grafana grafana -o yaml
echo '<admin-password>' | base64 --decode
```
[Grafana](http://localhost:3000/)
`username: admin`
`password: <found above>`

### Other references and notes
- Currently having to hardcode some env vars for the `ui` component, dynamic injection is not supported currently.
- https://itnext.io/migrating-a-spring-boot-service-to-kubernetes-in-5-steps-7c1702da81b6
- https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
- https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest#create-the-service-principal
- https://www.callicoder.com/spring-boot-docker-example/
- [Managing Computing Resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)
- [Assign Memory Resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
- [Using a private repository](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry)
- [Kubernetes - Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes - Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
- For Jaeger and Debugging help check project wiki.

## Things that are TODO/TBD
- Ingress engine for routing and shared IP addresses/urls.
  - Hopefully assists in removing hard coding the `api` IP address.
  - SSL Support, needed for Azure AD. 
- Kube Secrets Pod manifest
- Security / Azure AD writeup in this readme.
