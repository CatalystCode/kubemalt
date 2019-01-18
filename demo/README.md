# kubemalt - Spring Boot Application Demo

### Simple Application to test KubeMALT components

Deploy a spring boot based application to a KubeMALT supporting cluster.

Application is built from the [Microsoft/containers-rest-cosmos-appservice-java Project](https://github.com/Microsoft/containers-rest-cosmos-appservice-java)

These steps were cobbled together from Walkthrough documents [here](https://github.com/Microsoft/containers-rest-cosmos-appservice-java/issues/59) and adjusted for kubernetes deployments.

## Steps

### Create docker images and deploy to private docker repository
Skipping these docs for now.

### Create MALT Kubernetes Cluster
Through [bedrock](https://github.com/Microsoft/bedrock) or [Fabrikate](https://github.com/Microsoft/fabrikate)

### Configure `kubectl` to point to above cluster
Through [kubectx & kubens](https://github.com/ahmetb/kubectx) or for Azure:
```
az login
az account set --subscription <subscription-guid>
az aks get-credentials --resource-group <resource-group-name> --name <kubernetes-cluster-name>
```
for the last command put the names of the resource group and cluster, not uuid/guid.

### Create rbac service principal to connect to private image repository
`az ad sp create-for-rbac --subscription <subscription-guid>`
Take note of the response (appId and password) for the next step.

### Create `regcred` in kubectl with above rbac-sp creds.
`kubectl create secret docker-registry regcred --docker-server=<private-docker-repo> --docker-username <rbac-appId> --docker-password <rbac-password>`

### Configure deployment manifests
- Update private docker repository `<private docker repo>` in `ui-deployment.yaml`.
- Update private docker repository `<private docker repo>` in `api-deployment.yaml`.
- Update `DB_CONNSTR` and `DB_NAME` to point to the CosmosDB/MondoDB created for this project in `api-deployment.yaml`.
- Optionally, change the number of `replicas` in `ui-deployment.yaml` and `api-deployment.yaml`.

### Deploy manifests to cluster
`kubectl apply -f ui-deployment.yaml && kubectl apply -f api-deployment.yaml`

### Deploy load balancer services
`kubectl apply -f ui-service.yaml && kubectl apply -f ui-service.yaml`

### Other references and notes
  - https://itnext.io/migrating-a-spring-boot-service-to-kubernetes-in-5-steps-7c1702da81b6
  - https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
  - https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest#create-the-service-principal
  - https://www.callicoder.com/spring-boot-docker-example/
  - 