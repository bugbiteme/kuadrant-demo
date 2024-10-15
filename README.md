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
oc create -f k8/cert-manager 
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

**Note** If `PHASE` shows as `Failed` troubleshoot