﻿
# ODA CA Lab installation

## Motivation

Improve develops work by simplifying, getting rid of manual steps using new ways of doing things not available when it was developed
Most of the work can be done nowadays using just Helm

## Software Versions

The environment where the chart has been tested has the following
|Software|Version  |
|--|--|
|Istio  | 1.16.1  |
|kubernetes | 1.25.6
|Helm | 3.10

The helm chart installs the following updated versions of third party to

|Software|Version  |
|--|--|
|Cert-Manager  |1.20  |
|Keycloak  |  20.0.3|
|Postgress| 15.0.1 |

## Changes

The Helm chart has been refactored to move all the different subcharts to the same level to improve rreadabilityA new chart, oda-ca has been create as an umbrella for others allowing to have a centralised configuration
|OLD| NEW | DESCRIPTION
|--|--|--|
| shell script  | oda-ca  | Chart of chart.
| shell script | cert-manager-init  | Install cert-manager Deploy Issuer and generate Certificate used by CRD webhook
| canvas/chart/keycloak | Bitnami/keycloak  | Direct remote dependency  on oda-ca
| canvas/| canvas-namespaces  | Namespaces
| canvas/chart/controller| controller  | ODA ingress controller
| canvas/chart/crds| oda-crds| ODA crds
| canvas/chart/weebhooks | oda-webhook| ODA mutating webhook to handle conversion among versions

## Configuration values

The values used [here](canvas-oda/README.md)

## Environment installation

### 1. Azure Kubernetes Service

Change the variable RESOURCENAME in the script below:-

```

RESOURCENAME=nameofmyaksresource

# Create Resource Group
az group create -l WestEurope -n $RESOURCENAME-rg

# Deploy template with in-line parameters
# The below AKS configuration can be amended to your needs by visiting the AKS Construction Set helper:- https://azure.github.io/AKS-Construction/
# But worth noting that not all configuration setup may be compatible with ODA

az deployment group create -g $RESOURCENAME-rg  --template-uri https://github.com/Azure/AKS-Construction/releases/download/0.9.10/main.json --parameters \
	resourceName=$RESOURCENAME \
	agentCount=1 \
	upgradeChannel=stable \
	AksPaidSkuForSLA=true \
	agentCountMax=2 \
	registries_sku=Basic \
	acrPushRolePrincipalId=$(az ad signed-in-user show --query id --out tsv) \
	omsagent=true \
	retentionInDays=30 \
	ingressApplicationGateway=true
	
# Install Azure Grafana

az grafana create --name grafana-$RESOURCENAME --resource-group $RESOURCENAME-rg

 
 ```
 
 #### Enable the collection of Prometheus metrics via Azure Monitor (Preview)
 
 **Prerequisites**: 
 
- The following resource providers must be registered in the subscription of the AKS cluster and the Azure Monitor Workspace.
-- Microsoft.ContainerService
-- Microsoft.Insights
-- Microsoft.AlertsManagement

- Register the AKS-PrometheusAddonPreview feature flag in the Azure Kubernetes clusters subscription with the following command in Azure CLI: az feature register --namespace Microsoft.ContainerService --name AKS-PrometheusAddonPreview.
- The aks-preview extension needs to be installed using the command az extension add --name aks-preview. For more information on how to install a CLI extension, see Use and manage extensions with the Azure CLI.
- Aks-preview version 0.5.122 or higher is required for this feature. You can check the aks-preview version using the az version command.

```
# RESOURCENAME=nameofmyaksresource # You will need to uncomment this line if the variable is no longer set from the previous step

GRAFANARESOURCEID=$(az grafana show -n grafana-$RESOURCENAME -g $RESOURCENAME-rg   --query id --output tsv)
MONITORWORKSPACERESOURCEID=$(az monitor log-analytics workspace show -n log-$RESOURCENAME -g $RESOURCENAME-rg --query id --output tsv)

az aks update --enable-azuremonitormetrics -n aks-$RESOURCENAME -g $RESOURCENAME-rg --grafana-resource-id  $GRAFANARESOURCEID

# Logging into AKS

az aks get-credentials -n aks-$RESOURCENAME -g $RESOURCENAME-rg

# Applying the Prometheus ConfigMap configuration for Azure Monitor managed service for Prometheus
# This can be modified to your needs as per docs:- 
# https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-scrape-configuration

kubectl apply -f https://raw.githubusercontent.com/samaea/oda-canvas-charts/master/azure-assets/ama-metrics-prometheus-config.yaml

```
 

### 2. Helm

A Helm 3.0+ installation is needed. Depending on your
<https://helm.sh/docs/intro/install/#through-package-managers>

Helm currently has an issue with the dependencies declared, the **helm dependency update** command only takes care of the dependencies at the first level preventing the correct installation. It supposes to be addressed in a future (May'23) 3.12 version

Until that version is released, we can use a plugin to sort it out this
<https://github.com/Noksa/helm-resolve-deps>

````bash
helm plugin install --version "main" https://github.com/Noksa/helm-resolve-deps.git
````

The charts used need the following repositories

```
helm repo add jetstack https://charts.jetstack.io
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### 3. Istio

We follow the helm steps provided by [Istio](https://istio.io/latest/docs/setup/install/helm/)

``` bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system --wait
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install istio-ingressgateway istio/gateway -n istio-ingress --wait
```

This way of installing Istio sets a **istio=ingress** label in the istio-ingress service.
The  apiOperatorIsito rely on this component to have a **istio=ingressgateway**
Check if it's the case in your installation.

````bash
kubectl get svc istio-ingressgateway -n istio-ingress --show-labels
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                      AGE     LABELS
istio-ingressgateway   LoadBalancer   10.43.218.202   172.28.58.9   15021:31154/TCP,80:31497/TCP,443:30230/TCP   3d22h   app.kubernetes.io/managed-by=Helm,app.kubernetes.io/name=istio-ingress,app.kubernetes.io/version=1.16.2,app=istio-ingress,helm.sh/chart=gateway-1.16.2,istio=ingress 
````

If so, execute this command to set the label to what it's expected.

````
kubectl label svc istio-ingressgateway -n istio-ingress istio=ingressgateway --overwrite
service/istio-ingress labeled
````

### 4. Reference implementation

1. Move to *canvas-oda*
2. Update the dependencies using the plugin installed

````bash
$ helm resolve-deps
Fetching updates from all helm repositories, attempt #1 ...
  * Updates have been fetched, took 2.146s
Resolving dependencies in canvas-oda chart ...
  * Dependencies have been resolved, took 8.672s
````

If we prefer not to use the plugin, we have to manually update the subchart which has dependencies, in this case *cert-manager-init*

````bash
cd ..\cert-manager-init
helm dependency update
````

and then do the same with the umbrella helm *canvas-oda*

````bash
cd ..\canvas-oda
helm dependency update
````

3. Install the reference implementation

Install the canvas using the following command.

````bash
helm install canvas -n canvas --create-namespace . 
NAME: canvas
 Feb  7 09:35:38 2023
NAMESPACE: canvas
STATUS: deployed
REVISION: 1
TEST SUITE: None
````

4. Apply the canvas-controller-configmap for PrometheusAnnotation to be configured. This ensures application metrics are merged into Istio's metrics.

```bash
kubectl patch configmap canvas-controller-configmap --patch-file ./configmap/canvas-controller-configmap.yaml -n canvas
```

### 5. Demo application (ProductCatalog)
 **Prerequisites**: 
 - Git
 1. Sample Product Catalog application based on this [tutorial](https://tmforum-oda.github.io/oda-ca-docs/caDocs/Observability-Tutorial/README.html) that leverages Azure Application Gateway for load balancing and using pod annotations to instruct Prometheus metrics to be collected via Azure Monitor. Visit [here](https://github.com/samaea/oda-ca-docs/blob/master/examples/ProductCatalog/productcatalog/templates/deployment-metricsapi.yaml) to see how that is done. For Application Gateway config, you can view this [here](https://github.com/samaea/oda-ca-docs/blob/master/examples/ProductCatalog/productcatalog/templates/ingress-partyroleapi.yaml) and [here](https://github.com/samaea/oda-ca-docs/blob/master/examples/ProductCatalog/productcatalog/templates/deployment-productcatalogapi.yaml).
````bash
git clone https://github.com/samaea/oda-ca-docs
cd oda-ca-docs/examples/ProductCatalog
helm install r1 productcatalog -n components
````

## Troubleshooting

### Error instaling: BackoffLimitExceeded

The installation can fail with an error

````bash
Error: INSTALLATION FAILED: failed post-install: job failed: BackoffLimitExceeded
````

There are two major causes of this error

1. An error on the Job for configuring keycloak

````bash
 kubectl get pods -n canvas
NAME                                        READY   STATUS      RESTARTS   AGE
canvas-keycloak-0                           1/1     Running     0          4m43s
canvas-keycloak-keycloak-config-cli-5k6h7   0/1     Error       0          2m50s
canvas-keycloak-keycloak-config-cli-fq5ph   1/1     Running     0          30s
canvas-postgresql-0                         1/1     Running     0          4m43s
compcrdwebhook-658f4868b8-48cvx             1/1     Running     0          4m43s
job-hook-postinstall-6bm99                  0/1     Completed   0          4m43s
oda-controller-ingress-d5c495bbb-crt4t      2/2     Running     0          4m43s
````

Checking the logs of the failed Job

````bash
2023-02-01 15:23:19.488  INFO 1 --- [           main] d.a.k.config.provider.KeycloakProvider   : Wait 120 seconds until http://canvas-keycloak-headless:8080/auth/ is available ...
2023-02-01 15:25:19.511 ERROR 1 --- [           main] d.a.k.config.KeycloakConfigRunner        : Could not connect to keycloak in 120 seconds: HTTP 403 Forbidden
````

That means that your k8s cluster assign IPs to PODs that [Keycloak consider public ones and forced to use HTTPS](https://www.keycloak.org/docs/latest/server_admin/#_ssl_modes)
The ranges valid are the following
`localhost`, `127.0.0.1`, `10.x.x.x`, `192.168.x.x`, and `172.16.x.x`

2. An Error in the Job but caused because the canvas-keycloak-0 that is in CrashLoopBackOff

````bash
$ kubectl get pods -A
NAMESPACE       NAME                                              READY   STATUS             RESTARTS      AGE
canvas          canvas-keycloak-0                                 0/1     CrashLoopBackOff   4 (89s ago)   6m11s
canvas          canvas-keycloak-keycloak-config-cli-9ks9d         0/1     Error              0             2m28s
canvas          canvas-keycloak-keycloak-config-cli-cd2gv         0/1     Error              0             4m38s
canvas          canvas-postgresql-0                               1/1     Running            0             6m11s
canvas          compcrdwebhook-658f4868b8-v9sc2                   1/1     Running            0             6m11s
canvas          job-hook-postinstall-v56pt                        0/1     Completed          0             6m10
````

Checking the logs `kubectl logs -n canvas sts/canvas-postgresql`  we can see an error

````bash
 FATAL:  password authentication failed for user "bn_keycloak"
 ````

In that case, a previous installation left a PVC reused by the Postgress
To solve that issue

- Uninstall the helm chart
- Delete the PVC with `kubectl delete pvc -n canvas data-canvas-postgresql-0`
- reinstall the canvas

### Error installing : Failed post-install

The installation could fail with this error

````bash
failed post-install: warning: Hook post-install canvas-oda/charts/cert-manager-init/templates/issuer.yaml failed: Internal error occurred:
failed calling webhook "webhook.cert-manager.io": failed to call webhook: Post "https://canvas-cert-manager-webhook.cert-manager.svc:443/mutate?timeout=10s":
x509: certificate signed by unknown authority
````

That error arises when Cert-Manager is not ready to accept Issuers
The installation has a configurable wait time
*cert-manager.leaseWaitTimeonStartup*
Increase the time over 70s if it fails
Uninstall the chart and reinstall it with the new time.
