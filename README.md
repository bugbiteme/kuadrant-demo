# kuadrant-demo

Instructions based on kuadrant installation documentation:
https://docs.kuadrant.io/0.11.0/kuadrant-operator/doc/install/install-openshift/

## Prereqs
- OpenShift Container Platform 4.16.x or later with community Operator catalog available.
  - RHDH: `Red Hat OpenShift Container Platform Cluster (AWS)`
- AWS/Azure or GCP with DNS capabilities.
- Accessible Redis instance.
- log into openshift via the CLI ('oc login ...`)


## Step 1 - Set up your environment

Environment Variable (you can get these from the AWS console)


```bash
export AWS_ACCESS_KEY_ID=xxxxxxx # Key ID from AWS with Route 53 access
export AWS_SECRET_ACCESS_KEY=xxxxxxx # Access key from AWS with Route 53 access
export REDIS_URL=redis://user:xxxxxx@some-redis.com:10340 # A Redis cluster URL
```

## Step 2 - Install Gateway API v1

```
oc apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

Expected output:
```
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
```

## Step 3 - Install cert-manager
(from: https://docs.openshift.com/container-platform/4.16/security/cert_manager_operator/cert-manager-operator-install.html#cert-manager-install-cli_cert-manager-operator-install)

Installation

```
apply create -f k8/cert-manager 
```

Verify that the OLM subscription is created by running the following command:

```
oc get subscription -n cert-manager-operator
```

Output
```
NAME                              PACKAGE                           SOURCE             CHANNEL
openshift-cert-manager-operator   openshift-cert-manager-operator   redhat-operators   stable-v1
```

Verify whether the Operator is successfully installed by running the following command:

```
oc get csv -n cert-manager-operator
```

Output
```
NAME                            DISPLAY                                       VERSION   REPLACES                        PHASE
cert-manager-operator.v1.14.0   cert-manager Operator for Red Hat OpenShift   1.14.0    cert-manager-operator.v1.13.1   Succeeded
```

**Note** If `PHASE` shows as `Failed` , delete these resources and try again

Delete:
```
oc delete -f k8/cert-manager
```

Verify that the status cert-manager Operator for Red Hat OpenShift is Running by running the following command:

```
oc get pods -n cert-manager-operator
```

Output:
```
NAME                                                       READY   STATUS    RESTARTS   AGE
cert-manager-operator-controller-manager-xxx               2/2     Running   0          3m10s
```

Verify that the status of cert-manager pods is Running by running the following command:

```
oc get pods -n cert-manager
```

Output:
```
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-7bbf4bdb5b-ncgcv             1/1     Running   0          4m58s
cert-manager-cainjector-694ddbdd8-6rbsd   1/1     Running   0          4m58s
cert-manager-webhook-5b599554ff-g6pdz     1/1     Running   0          4m58s
```

## Step 4 - Install and configure Istio with the Sail Operator 
(Assumes OSSM 3.x rather than Envoy GW with Gateway API provider)

```
oc create -f k8/sail-operator
```
Output:
```
namespace/gateway-system created
operatorgroup.operators.coreos.com/sail created
subscription.operators.coreos.com/sailoperator created
```

Check the status of the installation as follows:

```
oc get installplan -n gateway-system -o=jsonpath='{.items[0].status.phase}'
```

When ready, the status will change from `installing` to `complete`

### Configure Istio as a gateway

```
oc apply -f k8/gateway-system
```

Output:
```
istio.operator.istio.io/default created
```
Wait for Istio to be ready as follows:

```
oc wait istio/default -n gateway-system --for="condition=Ready=true"
```

Output:

```
istio.operator.istio.io/default condition met
```

## Step 6 - Optional: Configure observability and metrics (TODO)

## Step 7 - Setup the catalogsource and Kuandrant operator

According Kuadrant documentation, before installing the Kuadrant Operator you
need to create a `CatalogSource` CR in order to set up secrets that will be 
used later.

I have combined creating `CatalogSource` and installing the Kuadrant operator in one step:

```
oc apply -f k8/kuadrant-system
```

Output:

```
namespace/kuadrant-system created
catalogsource.operators.coreos.com/kuadrant-operator-catalog created
subscription.operators.coreos.com/kuadrant-operator created
operatorgroup.operators.coreos.com/kuadrant created
```
Wait for the Kuadrant Operators to be installed as follows:

```
oc  get installplan -n kuadrant-system -o=jsonpath='{.items[0].status.phase}'    
```
Output:
```                       
Complete
```

### Set up a DNSProvider

The example here is for `AWS Route 53`. It is important the secret for the DNSProvider 
is setup in the same namespace as the gateway.

Namespace:
```
oc create -f k8/ingress-gateway/                                             
```

Output:
```
namespace/ingress-gateway created
```
  
Secret creation:
```
oc -n ingress-gateway create secret generic aws-credentials \
  --type=kuadrant.io/aws \
  --from-literal=AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  --from-literal=AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY 
```

Output:
```
secret/aws-credentials created
```

## Step 9 - Install Kuadrant Components

To trigger your Kuadrant deployment

```
oc apply -f k8/kuadrant-system-2
```

Output:
```
kuadrant.kuadrant.io/kuadrant created
```

This will setup and configure a number of Kuadrant subcomponents. Some of these can also take additional configuration:

- Authorino (Enforcement Component for AuthPolicy)
- Learn More: (Authorino CRD)[https://docs.kuadrant.io/latest/authorino-operator/#the-authorino-custom-resource-definition-crd]
- Limitador (Enforcement Component for RateLimitPolicy)
- Learn More:(Limitador CRD)[https://docs.kuadrant.io/latest/limitador-operator/#features]
- DNS Operator (Enforcement Component for DNSPOlicy)

```bash
oc get pods      
NAME                                                              READY   STATUS      RESTARTS   AGE
47d9bae1bba2b665c8d7af92af72f5f5e40d6a312e18cd3520efa5444cj4s6j   0/1     Completed   0          14m
8c6622a121e44f73764d24640c6635ade6d1eed22a16d3824c4a56b101vr8xr   0/1     Completed   0          14m
a2b8922a63dde9fbc3c98056388b4d5803a6a7373b972f7ca2304e97389shck   0/1     Completed   0          14m
a71e1d850f27946bc9eed3d71f12b9c3747c4d485278f71baab7db0c1ct9vqm   0/1     Completed   0          14m
authorino-848b779d9f-rfvrc                                        1/1     Running     0          2m2s
authorino-operator-85df9f449f-bc4lb                               1/1     Running     0          13m
authorino-webhooks-86cfc9d5d6-g27cx                               1/1     Running     0          13m
dns-operator-controller-manager-5bdccfdd4b-kvl6q                  1/1     Running     0          13m
kuadrant-operator-catalog-d5g66                                   1/1     Running     0          14m
kuadrant-operator-controller-manager-84f87448c7-jzmdv             1/1     Running     0          13m
limitador-limitador-5cbb99d8d6-gjz9p                              1/1     Running     0          2m3s
limitador-operator-controller-manager-68598db75b-nzj7h            1/1     Running     0          13m
```