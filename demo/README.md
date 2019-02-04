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

### Create rbac service principal to connect to private image repository
`az ad sp create-for-rbac --subscription <subscription-guid>`
Take note of the response (appId and password) for the next step.

### Create `regcred` in kubectl with above rbac-sp creds.
`kubectl create secret docker-registry regcred --docker-server=<private-docker-repo> --docker-username <rbac-appId> --docker-password <rbac-password>`

### Create MongoDB
TBD - Currently creating CosmosDB through azure portal.

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
- Optionally, change the number of `replicas` in `ui-deployment.yaml` and `api-deployment.yaml`.

### Deploy manifests to cluster
`kubectl apply -f ui-deployment.yaml && kubectl apply -f api-deployment.yaml`

### Test deploy by visiting site
Get the URL/IP from the ui-service: `kubectl get services ui-lb-service`

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