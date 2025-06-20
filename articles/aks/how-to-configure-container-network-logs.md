---
title: "Set up Container Network Logs with Advanced Container Networking Services (Preview)"
description: Enabling Container Network Flow logs with storage in AKS"
author: shaifaligargmsft
ms.author: shaifaligarg
ms.service: azure-kubernetes-service
ms.subservice: aks-networking
ms.topic: how-to
ms.date: 05/09/2025
ms.custom: template-how-to-pattern, devx-track-azurecli
---

# Set up Container Network Logs with Advanced Container Networking Services

This document is designed to provide clear steps for configuring and utilizing Container Network Logs feature using Advanced Container Networking Services. These logs offer persistent network flow monitoring tailored to enhance visibility within containerized environments. By capturing Container Network Logs, you can effectively track network traffic, detect anomalies, optimize performance, and ensure compliance with established policies. Follow the detailed instructions provided to set up and integrate Container Network Logs for your system. For more information about Container Network Logs Feature, see 
 [Overview of Container Network Logs](container-network-observability-logs.md)

## Prerequisites

* An Azure account with an active subscription. If you don't have one, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.
[!INCLUDE [azure-CLI-prepare-your-environment-no-header.md](~/reusable-content/azure-cli/azure-cli-prepare-your-environment-no-header.md)]

*  The minimum version of Azure CLI required for the steps in this article is 2.73.0. To find the version, Run `az --version` . If you need to install or upgrade, see [Install Azure CLI](/cli/azure/install-azure-cli).

* Container Network Log with Stored logs mode works only for Cilium data planes. 

* Container Network logs with On-demand mode works for both Cilium and non-Cilium data planes. 

* If existing cluster is <= 1.32, upgrade the cluster to the latest available Kubernetes version.

### Install the `aks-preview` Azure CLI extension

Install or update the Azure CLI preview extension using the [`az extension add`](/cli/azure/extension#az_extension_add) or [`az extension update`](/cli/azure/extension#az_extension_update) command.

 The minimum version of the aks-preview Azure CLI extension is `18.0.0b2`.

```azurecli-interactive
# Install the aks-preview extension
az extension add --name aks-preview
# Update the extension to make sure you have the latest version installed
az extension update --name aks-preview
```
## Configuring Stored logs mode for Container Network Logs

### Register the `AdvancedNetworkingFlowLogsPreview' feature flag

Register the `AdvancedNetworkingFlowLogsPreview` feature flag using the  [`az feature register`](/cli/azure/feature#az_feature_register) command.

```azurecli-interactive 
az feature register --namespace "Microsoft.ContainerService" --name "AdvancedNetworkingFlowLogsPreview"
```
Verify successful registration using the [`az feature show`](/cli/azure/feature#az_feature_show) command. It takes a few minutes for the registration to complete.

```azurecli-interactive
az feature show --namespace "Microsoft.ContainerService" --name "AdvancedNetworkingFlowLogsPreview"
```

Once the feature shows `Registered`, refresh the registration of the `Microsoft.ContainerService` resource provider using the [`az provider register`](/cli/azure/provider#az_provider_register) command.


### Enable Advanced Container Networking Services on a new cluster

The `az aks create` command with the Advanced Container Networking Services flag, `--enable-acns`, creates a new AKS cluster with all Advanced Container Networking Services features. These features encompass:
* **Container Network Observability:**  Provides insights into your network traffic. To learn more visit [Container Network Observability](./advanced-container-networking-services-overview.md#container-network-observability).

* **Container Network Security:** Offers security features like Fully Qualified Domain filtering(FQDN). To learn more visit  [Container Network Security](./advanced-container-networking-services-overview.md#container-network-security).


```azurecli-interactive
# Set an environment variable for the AKS cluster name. Make sure to replace the placeholder with your own value.
export CLUSTER_NAME="<aks-cluster-name>"
export RESOURCE_GROUP="<aks-resource-group>"

# Create an AKS cluster
az aks create \
    --name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --generate-ssh-keys \
    --location uksouth \
    --max-pods 250 \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --network-dataplane cilium \
    --node-count 2 \
    --pod-cidr 192.168.0.0/16 \
    --kubernetes-version 1.32 \
    --enable-acns
```
#### Configuring Custom Resource for Log filtering  
Configuring Container Network Logs in stored logs mode requires defining specific Custom Resources to set filters for log collection. When at least one Custom Resource is defined, logs are collected and stored on the host node at "/var/log/acns/hubble/events.log". To configure logging, users must define and apply Custom Resources of the "RetinaNetworkFlowLog" type, specifying filters such as namespace, pod, service, port, protocol, or verdict. Multiple Custom Resources can exist in a cluster simultaneously. If no Custom Resource is defined with nonempty filters, no logs are saved in the designated location.
This sample definition of Custom resource demonstrates how to configure Retina network flow logs:
#### RetinaNetworkFlowLog Template 
```azurecli-interactive
apiVersion: acn.azure.com/v1alpha1
kind: RetinaNetworkFlowLog
metadata:
  name: sample-retinanetworkflowlog # Cluster scoped
spec:
  includefilters: # List of filters
    - name: sample-filter # Filter name
      from:
        namespacedPod: # List of source namespace/pods. Prepend namespace with /
          - sample-namespace/sample-pod
        labelSelector: # Standard k8s label selector
          matchLabels:
            app: frontend
            k8s.io/namespace: sample-namespace
          matchExpressions:
            - key: environment
              operator: In
              values:
                - production
                - staging
        ip: # List of source IPs; can be CIDR
          - "192.168.1.10"
          - "10.0.0.1"
      to:
        namespacedPod:
          - sample-namespace2/sample-pod2
        labelSelector:
          matchLabels:
            app: backend
            k8s.io/namespace: sample-namespace2
          matchExpressions:
            - key: tier
              operator: NotIn
              values:
                - dev
        ip:
          - "192.168.1.20"
          - "10.0.1.1"
      protocol: # List of protocols; can be tcp, udp, dns
        - tcp
        - udp
        - dns
      verdict: # List of verdicts; can be forwarded, dropped
        - forwarded
        - dropped
```

Following is the description of fields in this Custom resource definition.

| **Field**                        | **Type**         | **Description**                                                                                                                         | **Required** |
|----------------------------------|------------------|-----------------------------------------------------------------------------------------------------------------------------------------|--------------|
| `includefilters`                 | []filter | A list of filters defining network flows to include. Each filter specifies the source, destination, protocol, and other matching criteria. include filters can't be empty and should have at least one filter. | Mandatory    |
| `filters.name`            | String           | The name of the filter.                                                                                                                | Optional    |
| `filters.protocol`        | []string | The protocols to match for this filter. Valid values are `tcp`, `udp`, and `dns`.  As It's an optional parameter, if it isn't specified, logs with all protocols would be included.                                                      | Optional     |
| `filters.verdict`         | []string | The verdict of the flow to match. Valid values are `forwarded` and `dropped`.  As It's an optional parameter, if it isn't specified, logs with all verdicts would be included.                                                        | Optional     |
| `filters.from`            | Endpoint          | Specifies the source of the network flow. Can include IP addresses, label selectors, or namespace-pod pairs.                           | Optional    |
| `Endpoint.ip`         | []string | It can be single IP or CIDR.                                                                                                         | Optional     |
| `Endpoint.labelSelector` | Object           | A Label Selector is a mechanism to filter and query resources based on labels, allowing users to identify specific subsets of resources.A label selector can include two components: matchLabels and matchExpressions. Use matchLabels for straightforward matching by specifying a key-value pair (for example, {"app": "frontend"}). For more advanced criteria, use matchExpressions, where you define a label key, an operator (such as In, NotIn, Exists, or DoesNotExist), and an optional list of values. Ensure that the conditions in both matchLabels and matchExpressions are met, as they're logically ANDed. If no conditions are specified, the selector matches all resources. To match none, leave the selector null. Carefully define your label selector to target the correct set of resources.  | Optional     |
| `Endpoint.namespacedPod` | []string | A list of namespace and pod pairs (formatted as `namespace/pod`) for matching the source. name should match regex pattern ^.+$                                              | Optional     |
| `filters.to`              | Endpoint           | Specifies the destination of the network flow. Can include IP addresses, label selectors, or namespace-pod pairs.                      | Optional    |
| `Endpoint.ip`           | []string | It can be single IP or CIDR.         | Optional     |
| `Endpoint.labelSelector` | Object           | A label selector to match resources based on their labels.                                                                             | Optional     |
| `Endpoint.namespacedPod` | []string | A list of namespace and pod pairs (formatted as `namespace/pod`) for matching the destination.                                         | Optional     |

- Apply RetinaNetworkFlowLog CR to enable log collection at cluster with this command:

```azurecli-interactive
kubectl apply -f <crd.yaml>
```
Logs stored Locally on host nodes are temporary, as the host or node itself isn't a persistent storage solution. Furthermore, logs on host nodes are rotated upon reaching 50 MB in size. For longer-term storage and analysis,  recommendation is to configure the Azure Monitor Agent on the cluster to collect and retain logs into the Log analytics workspace. Alternatively, third-party logging services an OpenTelemetry collector can be integrated for additional log management options. 

#### Configuring Azure Monitor agent to scrape logs in Azure log analytics workspace for new cluster (Managed storage)

```azurecli-interactive
# Set an environment variable for the AKS cluster name. Make sure to replace the placeholder with your own value.
  export CLUSTER_NAME="<aks-cluster-name>"
  export RESOURCE_GROUP="<aks-resource-group>"

# Enable azure monitor with high log scale mode
    ### Use default Log Analytics workspace
    az aks enable-addons -a monitoring --enable-high-log-scale-mode -g $RESOURCE_GROUP -n $CLUSTER_NAME 
    ### Use existing Log Analytics workspace
    az aks enable-addons -a monitoring --enable-high-log-scale-mode -g $RESOURCE_GROUP -n $CLUSTER_NAME --workspace-resource-id <workspace-resource-id>

# Update the aks cluster with enable retina flow log flag. 
  az aks update --enable-acns \
    --enable-retina-flow-logs \
    -g $RESOURCE_GROUP \
    -n $CLUSTER_NAME
```
> [!NOTE]
> When enabled, container network flow logs are written to "/var/log/acns/hubble/events.log" when custom resource RetinaNetworkFlowLog is applied. If Log Analytics integration is enabled later, the Azure Monitor Agent (AMA) begins collecting logs from that point onward. Logs older than two minutes aren't ingested; only new entries appended after monitoring begins are collected in log analytics workspace. 

#### Configuring existing cluster to store logs at Azure Log analytics workspace

To enable the container network logs on existing cluster, follow these steps:
1. Check if monitoring addon is already enabled on that cluster with following command.
    ```azurecli-interactive
     az aks addon list -g $RESOURCE_GROUP -n $CLUSTER_NAME
    ```
1. Disable monitoring addon with following command, if monitoring addon is already enabled.
    ```azurecli-interactive
     az aks disable-addons -a monitoring -g $RESOURCE_GROUP -n $CLUSTER_NAME
    ```
   The reason for disabling is - it might already have monitoring enabled but not the high scale. For more information, see [High Scale mode](/azure/azure-monitor/containers/container-insights-high-scale).

1. Enable Azure monitor with high log scale mode.
    ```azurecli-interactive
     ### Use default Log Analytics workspace
     az aks enable-addons -a monitoring --enable-high-log-scale-mode -g $RESOURCE_GROUP -n $CLUSTER_NAME 
     ### Use existing Log Analytics workspace
     az aks enable-addons -a monitoring --enable-high-log-scale-mode -g $RESOURCE_GROUP -n $CLUSTER_NAME --workspace-resource-id <workspace-resource-id>
    ```
1. Update the aks cluster with enable retina flow log flag 
    ```azurecli-interactive
     az aks update --enable-acns \
         --enable-retina-flow-logs \
         -g $RESOURCE_GROUP \
         -n $CLUSTER_NAME
    ```
---    

### Get cluster credentials 

Once you have Get your cluster credentials using the [`az aks get-credentials`](/cli/azure/aks#az_aks_get_credentials) command.

```azurecli-interactive
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP
```
### Validate the Setup
Validate if Retina Network flow log capability is enabled with following command –
```azurecli-interactive
   az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME
```
Expected Output from command above is:
```shell
"netowrkProfile":{
	"advancedNwtworking": {
	 "enabled": true,
	 "observability":{
	   "enabled": true
	    }
	}
}
----------------------------
"osmagent":{
	"config":{
		"enableRetinaNetworkFlags": "True"
	}
}
```

Validate if CRD of type RetinaNetworkFlowLog is applied 
```azurecli-interactive
   kubectl describe retinanetworkflowlogs <cr-name>
```
Expect to see a Spec node that contains 'Include filters' & Status node. Status. State should be 'CONFIGURED' not 'FAILED'
```shell
Spec:
  Includefilters:
    From:
      Namespaced Pod:
        namespace/pod-
    Name:  sample-filter
    Protocol:
      tcp
    To:
      Namespaced Pod:
        namespace/pod-
    Verdict:
      dropped
Status:
  State:      CONFIGURED
  Timestamp:  2025-05-01T11:24:48Z
``` 

### Azure managed Grafana

#### Create Azure Managed Grafana instance

Use [az grafana create](/cli/azure/grafana#az-grafana-create) to create a Grafana instance. The name of the Grafana instance must be unique.

```azurecli-interactive
# Set an environment variable for the Grafana name. Make sure to replace the placeholder with your own value.
export GRAFANA_NAME="<grafana-name>"

# Create Grafana instance
az grafana create \
    --name $GRAFANA_NAME \
    --resource-group $RESOURCE_GROUP 
```
Verify data source for grafana instance- The user can verify subscription for data source for grafana dashboards vy checking data source tab in grafana instance as shown below -
  :::image type="content" source="./media/advanced-container-networking-services/check-datasource-grafana.png" alt-text="Screenshot of checking data source for grafana instance." lightbox="./media/advanced-container-networking-services/check-datasource-grafana.png":::
  
> [!NOTE]
> By default, the managed identity for Azure Managed Grafana (AMG) has read access to the subscription in which it was created. This means no additional configuration is needed if both AMG and the Log Analytics workspace are in the same subscription. However, if they are in different subscriptions, the user must manually assign the 'Monitoring Reader' role to the Grafana managed identity on the Log Analytics workspace. Refer following link to know more ([How to modify access permissions](/azure/managed-grafana/how-to-permissions))

### Visualization in Grafana dashboards

User can visualize Container Network Flow log for analysis with two prebuilt Grafana dashboards. Customers have several options to access these dashboards

#### Visualization using Azure Managed Grafana

1. Make sure the Azure logs pods are running using the `kubectl get pods` command.

    ```azurecli-interactive
    kubectl get pods -o wide -n kube-system | grep ama-logs
    ```
    
    Your output should look similar to the following example output:
    
    ```output
    ama-logs-9bxc6                                   3/3     Running   1 (39m ago)   44m
    ama-logs-fd568                                   3/3     Running   1 (40m ago)   44m
    ama-logs-rs-65bdd98f75-hqnd2                     2/2     Running   1 (43m ago)   22h
    
    ```
2. To simplify the analysis of logs, we provide preconfigured two Azure Managed Grafana dashboards. You can find them as 
    - **Azure / Insights / Containers / Networking / Flow Logs** - This dashboard provides visualizations into which Kubernetes workloads are communicating with each other, including network requests, responses, drops, and errors. Currently user needs to import those dashboards with ID.
    (ID: [23155](https://grafana.com/grafana/dashboards/23155-azure-insights-containers-networking-flow-logs//))
    :::image type="content" source="./media/advanced-container-networking-services/container-network-logs-dashboard.png" alt-text="Screenshot of Flow log Grafana dashboard in grafana instance." lightbox="./media/advanced-container-networking-services/container-network-logs-dashboard.png":::

    - **Azure / Insights / Containers / Networking / Flow Logs (External Traffic)** - This dashboard provides visualizations into which Kubernetes workloads are sending/receiving communications from outside a Kubernetes cluster, including network requests, responses, drops, and errors.
     (ID: [23156](https://grafana.com/grafana/dashboards/23156-azure-insights-containers-networking-flow-logs-external-traffic//))
    :::image type="content" source="./media/advanced-container-networking-services/container-network-logs-dashboard-external.png" alt-text="Screenshot of Flow log (external) Grafana dashboard in grafana instance." lightbox="./media/advanced-container-networking-services/container-network-logs-dashboard-external.png":::
    For more information about usage of this dashboard, refer [Overview of Container Network Logs](container-network-observability-logs.md)       

#### Visualization of container network logs in Azure portal. 

The user can visualize, query, and analyze Flow logs in Azure portal in Azure log analytics workspace of their cluster:
  :::image type="content" source="./media/advanced-container-networking-services/azure-log-analytics.png" alt-text="Screenshot of Container Network Logs in Azure log analytics." lightbox="./media/advanced-container-networking-services/azure-log-analytics.png":::

## Configuring on-demand mode

On-demand mode for network flows work with both Cilium and Non-Cilium Data planes.
To proceed, you must have an AKS cluster with [Advanced Container Networking Services](./advanced-container-networking-services-overview.md) enabled.

The `az aks create` command with the Advanced Container Networking Services flag, `--enable-acns`, creates a new AKS cluster with all Advanced Container Networking Services features. These features encompass:
* **Container Network Observability:**  Provides insights into your network traffic. To learn more visit [Container Network Observability](./advanced-container-networking-services-overview.md#container-network-observability).

* **Container Network Security:** Offers security features like FQDN filtering. To learn more visit  [Container Network Security](./advanced-container-networking-services-overview.md#container-network-security).

#### [**Cilium**](#tab/cilium)

> [!NOTE]
> Clusters with the Cilium data plane support Container Network Observability and Container Network security starting with Kubernetes version 1.29.

```azurecli-interactive
# Set an environment variable for the AKS cluster name. Make sure to replace the placeholder with your own value.
export CLUSTER_NAME="<aks-cluster-name>"

# Create an AKS cluster
az aks create \
    --name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --generate-ssh-keys \
    --location eastus \
    --max-pods 250 \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --network-dataplane cilium \
    --node-count 2 \
    --pod-cidr 192.168.0.0/16 \
    --kubernetes-version 1.29 \
    --enable-acns
```

#### [**Non-Cilium**](#tab/non-cilium)

> [!NOTE]
> [Container Network Security](./advanced-container-networking-services-overview.md#container-network-security) feature is not available for Non-cilium clusters.

```azurecli-interactive
# Set an environment variable for the AKS cluster name. Make sure to replace the placeholder with your own value.
export CLUSTER_NAME="<aks-cluster-name>"

# Create an AKS cluster
az aks create \
    --name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --generate-ssh-keys \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --pod-cidr 192.168.0.0/16 \
    --enable-acns
```

---

### Enable Advanced Container Networking Services on an existing cluster

The [`az aks update`](/cli/azure/aks#az_aks_update) command with the Advanced Container Networking Services flag, `--enable-acns`, updates an existing AKS cluster with all Advanced Container Networking Services features which includes [Container Network Observability](./advanced-container-networking-services-overview.md#container-network-observability) and the [Container Network Security](./advanced-container-networking-services-overview.md#container-network-security) feature.


> [!NOTE]
> Only clusters with the Cilium data plane support Container Network Security features of Advanced Container Networking Services.

```azurecli-interactive
az aks update \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --enable-acns
```    

## Get cluster credentials 

Once you have Get your cluster credentials using the [`az aks get-credentials`](/cli/azure/aks#az_aks_get_credentials) command.

```azurecli-interactive
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP
```

### Install Hubble CLI
Install the Hubble CLI to access the data it collects using the following commands:

```azurecli-interactive
# Set environment variables
export HUBBLE_VERSION=v1.16.3
export HUBBLE_ARCH=amd64

#Install Hubble CLI
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

### Visualize the Hubble Flows

1. Make sure the Hubble pods are running using the `kubectl get pods` command.

    ```azurecli-interactive
    kubectl get pods -o wide -n kube-system -l k8s-app=hubble-relay
    ```
    
    Your output should look similar to the following example output:
    
    ```output
    hubble-relay-7ddd887cdb-h6khj     1/1  Running     0       23h 
    ```
    
1. Port forward Hubble Relay using the `kubectl port-forward` command.
    
    ```bash
    kubectl port-forward -n kube-system svc/hubble-relay --address 127.0.0.1 4245:443
    ```
    
1. Mutual TLS (mTLS) ensures the security of the Hubble Relay server. To enable the Hubble client to retrieve flows, you need to get the appropriate certificates and configure the client with them. Apply the certificates using the following commands:

    ```azurecli-interactive
    #!/usr/bin/env bash
    
    set -euo pipefail
    set -x
    
    # Directory where certificates will be stored
    CERT_DIR="$(pwd)/.certs"
    mkdir -p "$CERT_DIR"
    
    declare -A CERT_FILES=(
      ["tls.crt"]="tls-client-cert-file"
      ["tls.key"]="tls-client-key-file"
      ["ca.crt"]="tls-ca-cert-files"
    )
    
    for FILE in "${!CERT_FILES[@]}"; do
      KEY="${CERT_FILES[$FILE]}"
      JSONPATH="{.data['${FILE//./\\.}']}"
    
      # Retrieve the secret and decode it
      kubectl get secret hubble-relay-client-certs -n kube-system \
        -o jsonpath="${JSONPATH}" | \
        base64 -d > "$CERT_DIR/$FILE"
    
      # Set the appropriate hubble CLI config
      hubble config set "$KEY" "$CERT_DIR/$FILE"
    done
        
    hubble config set tls true
    hubble config set tls-server-name instance.hubble-relay.cilium.io
    ```

1. Verify the secrets were generated using the following `kubectl get secrets` command:
    ```azurecli-interactive
    kubectl get secrets -n kube-system | grep hubble-
    ```

    Your output should look similar to the following example output:

    ```output
    kube-system     hubble-relay-client-certs     kubernetes.io/tls     3     9d
    
    kube-system     hubble-relay-server-certs     kubernetes.io/tls     3     9d
    
    kube-system     hubble-server-certs           kubernetes.io/tls     3     9d    
    ```

2. Make sure the Hubble Relay pod is running using the `hubble observe` command.

    ```azurecli-interactive
    hubble observe --pod hubble-relay-7ddd887cdb-h6khj
    ```

### Visualize using Hubble UI

1. To use Hubble UI, save following into hubble-ui.yaml
    ```yml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: hubble-ui
      namespace: kube-system
    ---
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: hubble-ui
      labels:
        app.kubernetes.io/part-of: retina
    rules:
      - apiGroups:
          - networking.k8s.io
        resources:
          - networkpolicies
        verbs:
          - get
          - list
          - watch
      - apiGroups:
          - ""
        resources:
          - componentstatuses
          - endpoints
          - namespaces
          - nodes
          - pods
          - services
        verbs:
          - get
          - list
          - watch
      - apiGroups:
          - apiextensions.k8s.io
        resources:
          - customresourcedefinitions
        verbs:
          - get
          - list
          - watch
      - apiGroups:
          - cilium.io
        resources:
          - "*"
        verbs:
          - get
          - list
          - watch
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: hubble-ui
      labels:
        app.kubernetes.io/part-of: retina
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: hubble-ui
    subjects:
      - kind: ServiceAccount
        name: hubble-ui
        namespace: kube-system
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: hubble-ui-nginx
      namespace: kube-system
    data:
      nginx.conf: |
        server {
            listen       8081;
            server_name  localhost;
            root /app;
            index index.html;
            client_max_body_size 1G;
            location / {
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                # CORS
                add_header Access-Control-Allow-Methods "GET, POST, PUT, HEAD, DELETE, OPTIONS";
                add_header Access-Control-Allow-Origin *;
                add_header Access-Control-Max-Age 1728000;
                add_header Access-Control-Expose-Headers content-length,grpc-status,grpc-message;
                add_header Access-Control-Allow-Headers range,keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout;
                if ($request_method = OPTIONS) {
                    return 204;
                }
                # /CORS
                location /api {
                    proxy_http_version 1.1;
                    proxy_pass_request_headers on;
                    proxy_hide_header Access-Control-Allow-Origin;
                    proxy_pass http://127.0.0.1:8090;
                }
                location / {
                    try_files $uri $uri/ /index.html /index.html;
                }
                # Liveness probe
                location /healthz {
                    access_log off;
                    add_header Content-Type text/plain;
                    return 200 'ok';
                }
            }
        }
    ---
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: hubble-ui
      namespace: kube-system
      labels:
        k8s-app: hubble-ui
        app.kubernetes.io/name: hubble-ui
        app.kubernetes.io/part-of: retina
    spec:
      replicas: 1
      selector:
        matchLabels:
          k8s-app: hubble-ui
      template:
        metadata:
          labels:
            k8s-app: hubble-ui
            app.kubernetes.io/name: hubble-ui
            app.kubernetes.io/part-of: retina
        spec:
          serviceAccountName: hubble-ui
          automountServiceAccountToken: true
          containers:
          - name: frontend
            image: mcr.microsoft.com/oss/cilium/hubble-ui:v0.12.2   
            imagePullPolicy: Always
            ports:
            - name: http
              containerPort: 8081
            livenessProbe:
              httpGet:
                path: /healthz
                port: 8081
            readinessProbe:
              httpGet:
                path: /
                port: 8081
            resources: {}
            volumeMounts:
            - name: hubble-ui-nginx-conf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: nginx.conf
            - name: tmp-dir
              mountPath: /tmp
            terminationMessagePolicy: FallbackToLogsOnError
            securityContext: {}
          - name: backend
            image: mcr.microsoft.com/oss/cilium/hubble-ui-backend:v0.12.2
            imagePullPolicy: Always
            env:
            - name: EVENTS_SERVER_PORT
              value: "8090"
            - name: FLOWS_API_ADDR
              value: "hubble-relay:443"
            - name: TLS_TO_RELAY_ENABLED
              value: "true"
            - name: TLS_RELAY_SERVER_NAME
              value: ui.hubble-relay.cilium.io
            - name: TLS_RELAY_CA_CERT_FILES
              value: /var/lib/hubble-ui/certs/hubble-relay-ca.crt
            - name: TLS_RELAY_CLIENT_CERT_FILE
              value: /var/lib/hubble-ui/certs/client.crt
            - name: TLS_RELAY_CLIENT_KEY_FILE
              value: /var/lib/hubble-ui/certs/client.key
            livenessProbe:
              httpGet:
                path: /healthz
                port: 8090
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8090
            ports:
            - name: grpc
              containerPort: 8090
            resources: {}
            volumeMounts:
            - name: hubble-ui-client-certs
              mountPath: /var/lib/hubble-ui/certs
              readOnly: true
            terminationMessagePolicy: FallbackToLogsOnError
            securityContext: {}
          nodeSelector:
            kubernetes.io/os: linux 
          volumes:
          - configMap:
              defaultMode: 420
              name: hubble-ui-nginx
            name: hubble-ui-nginx-conf
          - emptyDir: {}
            name: tmp-dir
          - name: hubble-ui-client-certs
            projected:
              defaultMode: 0400
              sources:
              - secret:
                  name: hubble-relay-client-certs
                  items:
                    - key: tls.crt
                      path: client.crt
                    - key: tls.key
                      path: client.key
                    - key: ca.crt
                      path: hubble-relay-ca.crt
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: hubble-ui
      namespace: kube-system
      labels:
        k8s-app: hubble-ui
        app.kubernetes.io/name: hubble-ui
        app.kubernetes.io/part-of: retina
    spec:
      type: ClusterIP
      selector:
        k8s-app: hubble-ui
      ports:
        - name: http
          port: 80
          targetPort: 8081
    ```

1. Apply the hubble-ui.yaml manifest to your cluster, using the following command. 
    ```azurecli-interactive
    kubectl apply -f hubble-ui.yaml
    ```

1. Set up port forwarding for Hubble UI using the `kubectl port-forward` command.

    ```azurecli-interactive
    kubectl -n kube-system port-forward svc/hubble-ui 12000:80
    ```

1. Access Hubble UI by entering `http://localhost:12000/` into your web browser.

### Basic troubleshooting 

- Advanced Container Networking Services is a prerequisite for enabling AMA log collection feature:
   Trying to enable container network flow logs capability on a cluster without enabling Advanced Container Networking Services:

    ```az aks update -g test-rg -n test-cluster --enable-retina-flow-logs```

   Would result in an error message:

    ```Flow logs requires '--enable-acns', advanced networking to be enabled, and the monitoring addon to be enabled.```

- If the cluster Kubernetes version is below 1.32.0, trying to enable '--enable-retina-flow-logs':

    ```The specified orchestrator version %s is not valid. Advanced Networking Flow Logs is only supported on Kubernetes version 1.32.0 or later.```

    Where the %s is replaced by the customer's K8s version

- If a customer tries to enable '--enable-retina-flow-logs' on a subscription where AFEC flag isn't enabled:

    ```Feature Microsoft.ContainerService/AdvancedNetworkingFlowLogsPreview is not enabled. Please see https://aka.ms/aks/previews for how to enable features.```

- If a customer tries to apply a RetinaNetworkFlowLog CR on a cluster where Advanced Container Networking Services isn't enabled:
    
   ```error: resource mapping not found for <....>": no matches for kind "RetinaNetworkFlowLog" in version "acn.azure.com/v1alpha1"```
   ```ensure CRDs are installed first```
 
### Disable container network logs: stored logs mode on existing cluster 
If all the CRDs get deleted, Flow log collection would stop as there would be no filters defined for collection. 
To disable retina flow log collection by Azure monitor agent, use following command 
  ```azurecli-interactive
   az aks update -n $CLUSTER_NAME -g $RESOURCE_GROUP --disable-retina-flow-logs

```

## Clean up resources

If you don't plan on using this application, delete the other resources you created in this article using the [`az group delete`](/cli/azure/#az_group_delete) command.

```azurecli-interactive
  az group delete --name $RESOURCE_GROUP
```

## Next steps

In this how-to article, you learned how to enable Container Network logs with Advanced Container Networking Services for your AKS cluster.

* For more information about Advanced Container Networking Services for Azure Kubernetes Service (AKS), see [What is Advanced Container Networking Services for Azure Kubernetes Service (AKS)?](advanced-container-networking-services-overview.md).
* Explore Container Network Observability features in Advanced Container Networking Services in [What is Container Network Observability](./advanced-container-networking-services-overview.md#container-network-observability)
