# jitsi-multi-shard

## Intro
A multi shard deployment of jitsi-meet based from the [hpi-schul-cloud/jitsi-deployment](https://github.com/hpi-schul-cloud/jitsi-deployment).

## Prerequisites
- Nginx-Ingress Controller
    - Install [kubernetes/ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/#using-helm).
- Metacontroller
    - Install the [aporquez/metacontroller](https://github.com/aporquez/metacontroller).
    - This is use for creating the NodePort type Service for each jvb pods.
    - IMPORTANT: You can only install one copy of this in a given Kubernetes cluster, due to the fact that it has CRDs included in it. You'll get an error if you attempt to install a second copy.
- Label the kubernetes nodes for the zones. Shard-0 for ZONE_0, Shard-1 for ZONE_1, Shard-2 for ZONE_2, etc.
    - e.g. kubectl label nodes node-name topology.kubernetes.io/zone=ZONE_0
- Open the kubernetes node ports 30300-30399 for shard-0, 30400-30499 for shard-1, 30500-30599 for shard-2, etc.
    - The range of ports is dependent on the maximum replica of the jvb.
- Prometheus Adapter
    - Follow the [Monitoring](##Monitoring) guide on how to install.

## Deployment
### Namespace
```bash
kubectl create namespace jitsi
kubectl label namespace jitsi
```

### TURN/STUN
Not configured.

### ENABLE AUTH
Uncomment the environment variables of Auth of the following:
- [prosody-deployment-patch.yaml](yaml-k8s/yaml-overlay/dev/jitsi-base/prosody-deployment-patch.yaml)

Include the following variables in the creation of secret.
- JWT_APP_ID
- JWT_APP_SECRET
- JWT_ACCEPTED_ISSUERS
- JWT_ACCEPTED_AUDIENCES

### Create the k8s secret of jitsi
Using kubectl
```bash
kubectl create secret generic jitsi-config -n jitsi --from-literal=JICOFO_COMPONENT_SECRET=replace --from-literal=JICOFO_AUTH_PASSWORD=replace --from-literal=JVB_AUTH_PASSWORD=replace
```
Using kustomize
```bash
cd yaml-k8s/yaml-overlay/dev
kustomize edit add secret jitsi-config --disableNameSuffixHash --from-literal=JICOFO_COMPONENT_SECRET=replace --from-literal=JICOFO_AUTH_PASSWORD=replace --from-literal=JVB_AUTH_PASSWORD=replace
```

## Update the deployment specific settings of overlays
- PUBLIC_URL
    - This should be same with your frontend hostname that is configured in the [ingress](yaml-k8s/yaml-overlay/dev/loadbalancer/haproxy-ingress-patch.yaml) e.g. https://jitsi.meet.com.
    - Update it in the [common config](yaml-k8s/yaml-overlay/dev/config/common-configmap-patch.yaml).
- HAProxy backend web
    - Update the [backend server](yaml-k8s\yaml-overlay\dev\loadbalancer\haproxy-configmap-patch.yaml) of the haproxy.
    - This is dependent on the namespace name. From _http._tcp.web.jitsi.svc.cluster.local to _http._tcp.web.namespace-name.svc.cluster.local. E.g. _http._tcp.web.jitsi-dev.svc.cluster.local.

## Monitoring
By default, the [jvb-statefulset](yaml-k8s/yaml-base/shard/jvb/jvb-statefulset.yaml), [prosody-deployment](yaml-k8s/yaml-base/shard/prosody-deployment.yaml), and [haproxy-statefulset](yaml-k8s/yaml-base/loadbalancer/haproxy-statefulset.yaml) are using the annotation "prometheus.io/scrape" for monitoring. Select the Option 1 if you are going to use this. If you don't want to use the annotation, remove it and follow the Option 2.
- Option 1: Separate installation of Grafana, Prometheus, Prometheus Node Exporter, Kube State Metrics
    - Install [Kube State Metrics](https://github.com/kubernetes/kube-state-metrics/tree/master/charts/kube-state-metrics)
    - Install [Node Exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-node-exporter)
    - Install [Prometheus](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus)
    - Install [Prometheus Adapter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-adapter)
    - Install [Grafana](https://github.com/grafana/helm-charts)
    - Import the [jitsi-dashboard.json](yaml-k8s/yaml-base/monitoring/jitsi-dashboard.json) to Grafana.
- Option 2: Using Prometheus Stack
    - Install the [Prometheus Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) which includes the prometheus-operator. The PodMonitor is dependent on the CRDs of the prometheus-operator.
    - Uncomment the monitoring in the [overlay/dev/kustomization.yaml](yaml-k8s/yaml-overlay/dev/kustomization.yaml)  and [overlay/prod/kustomization.yaml](yaml-k8s/yaml-overlay/prod/kustomization.yaml).
    - Import the [jitsi-dashboard.json](yaml-k8s/yaml-base/monitoring/jitsi-dashboard.json) to Grafana.

Add the custom metric to the Prometheus Adapter. The [jvb-hpa](yaml-k8s/yaml-base/shard/jvb/jvb-hpa.yaml) depends on the custom metric container_network_transmit_bytes_per_second to autoscale the jvb pods.
```yaml
- seriesQuery: 'container_network_transmit_bytes_total{interface="eth0"}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[3m])) by (<<.GroupBy>>)'
```

## TODO
- Create a helm chart version

## References
- HAProxy
    - https://www.haproxy.com/blog/websockets-load-balancing-with-haproxy/
    - https://gist.github.com/lackac/3767467
    - https://medium.com/@lucjansuski/investigating-websocket-haproxy-reconnecting-and-timeouts-6d19cc0002a1
