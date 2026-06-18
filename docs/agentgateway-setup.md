# Overview

This guide is for Dev Team to deploy AgentGateway on **OpenShift Local** as part of evaluating MCP Gateway before RFO/RFD.



## Install Guide
 For files that are applied, please refer to the [Github Repo](https://github.com/ninjachicken100/setup-docs/settings)

### Step 1: agentgateway Control Plane
We will be Installing the AgentGateway Control Plane which is required for ...

**Create new namespace**
```
oc new-project agentgateway-system
```


**Pull required Custom Resources to be applied to cluster**
```
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v1.2.0 agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds
```


Expected Output:
```
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created

```

**Pull Helm chart and deploy helm chart**

```
helm pull oci://cr.agentgateway.dev/charts/agentgateway --version v1.2.0

tar -xvf agentgateway-v1.2.0.tgz

open agentgateway/values.yaml

```

```
helm upgrade -i -n agentgateway-system agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
--version v1.2.0
```

At the end of this section, you should see a agentgateway pod spun up in the agentgateway-system namespace

> [!NOTE]
> Please refer to the [guide](https://agentgateway.dev/docs/kubernetes/latest/install/helm/) for more information.

### Step 2: agentgateway Proxy

Apply the agentgateway proxy CR

```
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - protocol: HTTP
    port: 80
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

You may access the agentgateway UI by running a portforward
```
kubectl port-forward -n agentgateway-system pod/agentgateway-proxy-5d947f8657-rl5zs 15000:15000

http://localhost:15000/ui
```

> [!NOTE]
> Please refer to the [guide](https://agentgateway.dev/docs/kubernetes/latest/setup/gateway/) for more information.

## Integration with vLLM 
This section assumes that you already have at least one vLLm instance spun up and running in the vllm namespace


We will be applying the following resource for each instance of vLLM:

- **agentgateway-backend** - to define endpoint of vLLM


- **agentgateway-proxy-http-route** to expose model route


**agentgatewaybackend Custom Resource**

```
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: <vllm-release-name>-agentgateway-backend
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai:
        model: <vllm-release-name>
        host: <service-name>
        port: 80
```

> [!NOTE]
> Obtain the release-name by running `helm list -n <namespace>`

**Apply HTTP Route Custom Resource**

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: agentgateway-proxy-http-route
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
      headers:
        - type: RegularExpression
          name: x-model
          value: <vllm-release-name> 
    backendRefs:
    - name: <vllm-release-name>-agentgateway-backend
      namespace: agentgateway-system
      group: agentgateway.dev
      kind: AgentgatewayBackend
```

Next, apply the following **extract-model** Custom Resource
- It extracts the model name from the request body and promotes it to an x-model header so the HTTPRoutes can match on it and send the request to the right backend.


```
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: extract-model
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: agentgateway-proxy
  traffic:
    phase: PreRouting
    transformation:
      request:
        set:
        - name: "x-model"
          value: 'json(request.body).model'
```


After applying the custom resource, Run test curls from your Terminal to the vLLM endpoint.


Port Forward to simulate request passing through MCP Gateway

```
kubectl port-forward deployment/agentgateway-proxy -n agentgateway-system 8080:80
```
Run Test Curl
```
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dummy" \
  -H "x-model: <vllm-release-name> \
  -d '{
    "model": vllm-release-name,
    "messages": [
      {"role": "user", "content": "Explain the benefits of vLLM."}
    ]
  }' | jq
```

## Integration with Keycloak
This section assumes that you already have keycloak running.

Apply the following Custom Resource certificate to the keycloak namespace

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: keycloak-tls
  namespace: keycloak
spec:
  secretName: my-tls-secret
  issuerRef:
    name: my-ca-issuer
    kind: ClusterIssuer
  dnsNames:
	  - keycloak-service-keycloak.apps-crc.testing
    - keycloak-service.keycloak.svc.cluster.local
```
> [!TIP]
> Please refer to the [guide](https://rigorous-allspice-3f4.notion.site/Setup-Guides-32c522040daf80648de6c2f6c6e4d63f?source=copy_link) for more information on setting up keycloak
