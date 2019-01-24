# kubemalt - Spring Boot Application Demo

### Simple Application to test KubeMALT components

Deploy a spring boot based application to a KubeMALT supporting cluster.

Application is built from the [CatalystCode/containers-rest-cosmos-appservice-java Project](https://github.com/CatalystCode/containers-rest-cosmos-appservice-java)

These steps were cobbled together from Walkthrough documents [here](https://github.com/Microsoft/containers-rest-cosmos-appservice-java/issues/59) and adjusted for kubernetes deployments.

## Steps

### Create MALT Kubernetes Cluster
Through [~~bedrock~~](https://github.com/Microsoft/bedrock) or [Fabrikate](https://github.com/Microsoft/fabrikate).

### Configure `kubectl` to point to above cluster
Through [kubectx & kubens](https://github.com/ahmetb/kubectx) or for Azure:
```
az login
az account set --subscription <subscription-guid>
az aks get-credentials --resource-group <resource-group-name> --name <kubernetes-cluster-name>
```
For the last command put the names of the resource group and cluster, not uuid/guid.

### Create rbac service principal to connect to private image repository
`az ad sp create-for-rbac --subscription <subscription-guid>`
Take note of the response (appId and password) for the next step.

### Create `regcred` in kubectl with above rbac-sp creds.
`kubectl create secret docker-registry regcred --docker-server=<private-docker-repo> --docker-username <rbac-appId> --docker-password <rbac-password>`

### Create and deploy kube pod with Secrets (TBD)
TBD - Create pod with various secrets so sensitive env vars etc. do not need to be copy-pasted all the time.
- Repository URL
- `DB_CONNSTR`
- `DB_NAME`
- Various AAD Security env vars
- UI env vars (if we get dynamic injection supported)

### Deploy load balancer services
`kubectl apply -f ui-service.yaml && kubectl apply -f ui-service.yaml`
This step is currently needed so we can hardcode the `api` IP and expose the `UI` layer.

### Create docker images
Build the `ui` and `api` components from the `containers-rest-cosmos-appservice-java`, aka. `Jackson`. You should be able to clone and simply run `Docker build .` in the `api` directory.

`ui` requires creating an `.env` file and populating it with relevant env var components needed for Webpack to subsitute in during the build.
Sample `.env` file:
```
WEBPACK_PROP_AAD_CLIENT_ID=<AAD GUID>
WEBPACK_PROP_API_BASE_URL=http://1.1.1.1:8080
WEBPACK_PROP_UI_BASEPATH=ui
```
WEBPACK_PROP_API_BASE_URL will come from the external IP for the API: `kubectl get services api-lb-service`

### Deploy images to docker repository
TBD: Private Azure container registry stuff.
Helpful scripts:
```
docker build -t tag-name .
docker tag tag-name <private docker repo>/ui:<version>
docker push <private docker repo>/ui
```

### Configure deployment manifests
- Update private docker repository `<private docker repo>` in `ui-deployment.yaml`.
  - Also update the `ui` image tag/version
- Update private docker repository `<private docker repo>` in `api-deployment.yaml`.
  - Also update the `api` image tag/version
- Update `DB_CONNSTR` and `DB_NAME` to point to the CosmosDB/MondoDB created for this project in `api-deployment.yaml`.
- Optionally, change the number of `replicas` in `ui-deployment.yaml` and `api-deployment.yaml`.

### Deploy manifests to cluster
`kubectl apply -f ui-deployment.yaml && kubectl apply -f api-deployment.yaml`

### Test deploy by visiting site
Get the URL/IP from the ui-service: `kubectl get services ui-lb-service`

### Other references and notes
- Currently having to hardcode some env vars for the `ui` component, dynamic injection is not supported currently.
- https://itnext.io/migrating-a-spring-boot-service-to-kubernetes-in-5-steps-7c1702da81b6
- https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
- https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest#create-the-service-principal
- https://www.callicoder.com/spring-boot-docker-example/

## Things that are TODO/TBD
- Ingress engine for routing and shared IP addresses/urls.
  - Hopefully assists in removing hard coding the `api` IP address.
  - SSL Support, needed for Azure AD. 
- Kube Secrets Pod manifest
- Security / Azure AD writeup in this readme.