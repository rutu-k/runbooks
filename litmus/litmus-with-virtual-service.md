# Steps to deploy LitmusChaos with Istio Virtual Service

## Install Litmus Core or Control Plane

- To install the Litmus Core use Litmus Helm Chart
```sh
helm upgrade --install litmus litmus-2.12.0.tgz -nlitmus --values <pathtovaluesfile>
```

#### **`valuesfile.yaml`**
```yaml
customLabels:
  sidecar.istio.io/inject: "true"

portal:
  frontend:
    virtualService:
      enabled: true
      hosts: ["litmuschaos.somedomain.com"]
      gateways: ["istio-system/gateway"]
      pathPrefixEnabled: false
```

- Login in to the Litmus Portal with default creds

- As soon as you login in to the portal, Litmus will deploy Self Agents. For Self Agents to be successful and active, you need to patch `agent-config` configMap and `litmusportal-frontend-service` virtualService

#### **`agent-config`**
```yaml
apiVersion: v1
data:
  AGENT_SCOPE: cluster
  COMPONENTS: |
    DEPLOYMENTS: ["app=chaos-exporter", "name=chaos-operator", "app=event-tracker", "app=workflow-controller"]
  CUSTOM_TLS_CERT: ""
  IS_CLUSTER_CONFIRMED: "true"
  SERVER_ADDR: http(s)://litmuschaos.somedomain.com/backend/query
  SKIP_SSL_VERIFY: "false"
  START_TIME: "1673939061"
  VERSION: 2.11.0
kind: ConfigMap
```

#### **`litmusportal-frontend-service`**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    meta.helm.sh/release-name: litmus
    meta.helm.sh/release-namespace: litmus
  labels:
    app.kubernetes.io/component: litmus-server
    app.kubernetes.io/instance: litmus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: litmus
    app.kubernetes.io/part-of: litmus
    app.kubernetes.io/version: 2.12.0
    helm.sh/chart: litmus-2.12.0
    litmuschaos.io/version: 2.11.0
    sidecar.istio.io/inject: "true"
  name: litmusportal-frontend-service
  namespace: litmus
spec:
  gateways:
  - istio-system/gateway
  hosts:
  - litmuschaos.somedomain.com
  http:
  - match:
    - uri:
        prefix: /backend/
    - uri:
        prefix: /backend
    name: backend
    rewrite:
      uri: /
    route:
    - destination:
        host: litmus-server-service
        port:
          number: 9002
  - match:
    - uri:
        prefix: /
    name: frontend
    route:
    - destination:
        host: litmus-frontend-service
        port:
          number: 9091
```

> Note: We are adding hhtp rewrite functionality in virtualService for /backend/(.*)

> Restart the Subscriber pod if necessary.

> use http or https in SERVER_ADDR url in agent-config.yaml


## Install Litmus External Agents

External Agents can be deployed in two ways:
    1. Using litmusctl
    2. Using Litmus-Agents Helm chart

### Using litmusctl

- Install litmusctl using this link

- Set Account by pointing out to the correct Litmus Core Deployment

```sh
itmusctl config set-account  --endpoint "https://litmuschaos-vs.stage.cnxloyalty.com" --password "litmus" --username "admin"
```

- Install Litmus Agents into the required EKS cluster by giving the necessary kubeconfig 

```sh
itmusctl connect chaos-delegate \
--name="external-agent-1" \
--project-id="a91xxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx" \
--non-interactive \
--kubeconfig <kubeconfig>.yaml \
--node-selector "topology.kubernetes.io/zone=us-east-1b,vpc.amazonaws.com/eniConfig=us-east-1b"
```

- Patch SERVER_ADDR in `agent-config.yaml` {http(s)://litmuschaos.somedomain.com/backend/query}




