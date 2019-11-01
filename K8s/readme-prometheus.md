# Prometheus for RobotShop

## Setup Prometheus

This uses https://github.com/coreos/kube-prometheus which deploys and setups all necessary elements for monitoring for the namespaces: default & kube-system.

To monitor the robot shop we need to [customize it](https://github.com/coreos/kube-prometheus#customizing-kube-prometheus) and add [Monitoring other Kubernetes Namespaces](https://github.com/coreos/kube-prometheus/blob/master/docs/monitoring-other-namespaces.md).

```robotshop.jsonnet
local kp =
  (import 'kube-prometheus/kube-prometheus.libsonnet') +
  // Uncomment the following imports to enable its patches
  // (import 'kube-prometheus/kube-prometheus-anti-affinity.libsonnet') +
  // (import 'kube-prometheus/kube-prometheus-managed-cluster.libsonnet') +
  // (import 'kube-prometheus/kube-prometheus-node-ports.libsonnet') +
  // (import 'kube-prometheus/kube-prometheus-static-etcd.libsonnet') +
  // (import 'kube-prometheus/kube-prometheus-thanos-sidecar.libsonnet') +
  {
    _config+:: {
      namespace: 'monitoring',

      prometheus+:: {
        namespaces: ["default","kube-system","robot-shop","monitoring"],
      },
    },
  };

{ ['setup/0namespace-' + name]: kp.kubePrometheus[name] for name in std.objectFields(kp.kubePrometheus) } +
{
  ['setup/prometheus-operator-' + name]: kp.prometheusOperator[name]
  for name in std.filter((function(name) name != 'serviceMonitor'), std.objectFields(kp.prometheusOperator))
} +
// serviceMonitor is separated so that it can be created after the CRDs are ready
{ 'prometheus-operator-serviceMonitor': kp.prometheusOperator.serviceMonitor } +
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['prometheus-adapter-' + name]: kp.prometheusAdapter[name] for name in std.objectFields(kp.prometheusAdapter) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) }
```

```
$ ./build.sh robotshop.jsonnet
$ kubectl create -f manifests/setup
$ kubectl apply -f manifests/
```
// I had to label the nodes: "kubernetes.io/os=linux" to make the schedulable.

Check the dashboards: https://github.com/coreos/kube-prometheus#access-the-dashboards

expose the grafana service (pass admin/admin)
kubectl expose deployment grafana --type=LoadBalancer --name=test-grafana -n monitoring

### Create a service monitor:
Create a service monitor for the cart service: 
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cart-service-monitor
spec:
  namespaceSelector:
      matchNames:
      - robot-shop
  selector:
    matchLabels:
      service: cart
  endpoints:
  - port: http
```

$ kubectl create -f cart-service-monitor.yaml -n monitoring

Troubleshooting: 

check http://localhost:9090/config if config was applied by looking for "monitoring/cart-service-monitor/0"

ceck http://localhost:9090/targets if target was found by looking for "cart-service-monitor" and that it lists endpoiunts

kubectl get svc test-cart-svc -n robot-shop
curl 35.226.96.54:8080/metric

Status: Didn't work. Next time watch the operator to see if the service monitors are picked up correctly


----



 curl localhost:32768/metrics
# HELP cart_items_total all items in all carts. Currently dummy
# TYPE cart_items_total counter
cart_items_total{a_dummy_key="a_dummy_label"} 42

com.instana.plugin.prometheus:
  customMetricSources:
  - url: '/metrics' # url path part which expose metrics
    metricNameIncludeRegex: '^cart'