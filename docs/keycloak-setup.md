# Overview

This guide is for Dev Team to deploy Keycloak on **OpenShift Local** for development testing purposes.


## Install Guide
For files that are applied, please refer to the [Github Repo](https://github.com/ninjachicken100/setup-docs/settings)


### Step 1: Installing Keycloak 

**Create new namespace**
```
oc new-project keycloak
```

In your Openshift Local UI, go to OperatorHub/SoftwareCatalog and search for keycloak and install the latest version selected.


### Step 2: Setting up Certificate
There will be two methods shown: A CA issued Certificate and Self-signed Certificate. 

> [!NOTE]
>  For Self-signed certificates, there could be TLS integration issues with your application that the guide does not cover.


#### Option 1: CA Issued Certificate
**Apply cert-manager custom Resource**

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

**Apply ...**
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: my-ca
  secretName: my-ca-secret
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    secretName: my-ca-secret
```



**Create Configmap to store Certificate**
```
kubectl get secret my-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | base64 -d > ca.crt

kubectl create configmap keycloak-ca-bundle --from-file=ca.crt=ca.crt -n agentgateway-system
```

#### Option 2: Self Signed Certificate

**Generate .crt and .key**
```
# Generate .crt and .key
openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt -days 365 -nodes
```

**Create secret container cert and key in cluster**
```
kubectl create secret tls my-tls-secret --cert=tls.crt --key=tls.key -n keycloak
```

### Step 3: Starting a Keycloak Instance

**Apply the following Custom Resources**



Download and apply the files inside the **docs** folder from the [Github Repo](https://github.com/ninjachicken100/setup-docs/settings)
```
oc apply -f keycloak-instance.yaml -f keycloak-db-secrets.yaml -f keycloak-admin-secrets.yaml -f keycloak-sts.yaml -f keycloak-svc.yaml
```

**Expose the Keycloak Route**
```
oc -n keycloak expose svc keycloak-service
```

### Step 4: Integrating with Applications
Create a new Realm for your application, and a new Client. 

> [!NOTE]
> Multiple 'Clients' can be created in a single realm.
