# Prometheus Operator Testing

## Overview 

This repository will test the [CoreOS Prometheus Operator](https://github.com/coreos/prometheus-operator) for monitoring the metrics of the [Elastic Stack centralized logging pipeline](https://github.com/cheuklau/elastic-stack-logging). The Elastic Stack pipeline consists of the microservice, Filebeat, Logstash, Elasticsearch and Kibana.

### What is an Operator?

From the [CoreOS blog](https://coreos.com/blog/introducing-operators.html), an Operator is an application-specific controller that extends Kubernetes APIs to create, configure and manage the instances of a complex stateful application on behalf of the Kubernetes user. Stateful applications (e.g., a monitoring system) require application-specific knowledge in order to function. An Operator builds upon the Kubernetes resource and controller concepts, and adds a set of application-specific knowledge that allows the Operator to perform common application tasks. Some general Operator guidelines include:

1. Operators should install as a single deployment.
2. Operators should create a 3rd-party type, and application instances are created using this type.
3. Operators should use built-in Kubernetes primitives e.g., `Services` and `ReplicaSets`.
4. Application instances should continue to run even if the Operator is removed.
5. Operators should orchestrate application upgrades.

### The Prometheus Operator

The [CoresOS Prometheus operator](https://github.com/coreos/prometheus-operator) creates, configures and manages Prometheus monitoring instances. Once installed, the user can:

1. Easily create and destroy Prometheus instances for a Kubernetes namespace or application.
2. Configure Prometheus (e.g., version, number of replicas) from a native Kubernetes resource.
3. Generate monitoring target configurations based on Kubernetes labels.

Prometheus Operator decouples Prometheus instances from the services they are monitoring by defining four custom resources:

1. `Prometheus`
    * Creates Prometheus servers with user-defined configurations e.g., monitoring targets, number of replicas, etc
    * Operator deploys a `StatefulSet` for each `Prometheus` resource created
    * Prometheus pods are configured to mount a `Secret` containing Prometheus configuration
    * Label selection determines which `ServiceMonitors` are covered by Prometheus instances
    * If no `ServiceMonitor` is specified, Operator leaves management of `Secret` to the user
2. `ServiceMonitor` 
    * Defines how services should be monitored using label selections
    * `Endpoint` objects (i.e., list of IP addresses) are needed for Prometheus to monitor an application
    * `Endpoint` objects populated by a `Service` object
    * `Service` object discovers pods using label selectors
    * `ServiceMonitor` discovers `Endpoint` objects and configures Prometheus to monitor those pods
    * `endpoints` in `ServiceMonitorSpec` used to configure which ports `EndPoint` objects are scraped
    * `ServiceMonitorNamespaceSelector` in `PrometheusSpec` used to restrict namespaces for `ServiceMonitors`
3. `AlertManager`
    * Defines a desired AlertManger setup
    * Operator deploys a `StatefulSet` for each `AlertManager` resource created
    * AlertManger pods configured with `Secret` which holds user configuration file in `alertmanager.yaml`
4. `PrometheusRule`
    * Defines a desired Prometheus rule to be consumed by one or more Prometheus instances
    * Alerts and recording rules saved as YAML files and can be dynamically loaded

## Build Instructions

First, deploy the Elastic Stack logging pipeline:

```
cd Prometheus_Testing/elk_k8s/
kubectl create -f ./microservice-filebeat
kubectl create -f ./logstash
kubectl create -f ./elasticsearch
kubectl create -f ./kibana
```

This will deploy the following into the Kubernetes cluster:
* Pod running microservice and Filebeat containers sharing a mounted volume where the logs are written
* Pod running Logstash reading in the data from Filebeat on Port 5044
* Pod running Elasticsearch reading in the data from Logstash on Port 9200
* Pod running Kibana reading in the data from Elasticsearch
* Kibana service is exposed on Port 30000

Next, we deploy the full monitoring stack in `prometheus-operator/contrib/kube-prometheus`. This package includes:

1. The Prometheus Operator
2. Highly available Prometheus
3. Highly available Alertmanager
4. Prometheus node-exporter (https://github.com/prometheus/node_exporter) which collects metrics on the machine level
5. kube-state-metrics (https://github.com/kubernetes/kube-state-metrics) which listens to the Kubernetes API server and generates metrics about the state of objects e.g., deployments, nodes and pods
6. Grafana (https://github.com/grafana/grafana) for visualization

To deploy the full monitoring stack:

```
cd Prometheus_Testing/prometheus-operator/contrib/kube-prometheus
kubectl create -f ./manifests
```

To access the dashboards, we need to port-forward from the node to each pod:

```
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
kubectl --namespace monitoring port-forward svc/grafana 3000
kubectl --namespace monitoring port-forward svc/alertmanager-main 9093
```

The dashboards for each service can now be visited using the links below:

* [Prometheus](http://localhost:9090)
* [Grafana](http://localhost:3000)
* [AlertManager](http://localhost:9093)

To teardown the monitoring stack:

```
cd Prometheus_Testing/prometheus-operator/contrib/kube-prometheus
kubectl delete -f ./manifests
```

## Customizing Kube-Prometheus

Install Go (https://dl.google.com/go/go1.11.2.darwin-amd64.pkg) then install `jsonnet` and `gojsontoyaml`:

```
go get github.com/google/go-jsonnet/jsonnet
go get github.com/brancz/gojsontoyaml
```

The default `example.jsonnet` is:

```
local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',
  },
};

{ ['00namespace-' + name]: kp.kubePrometheus[name] for name in std.objectFields(kp.kubePrometheus) } +
{ ['0prometheus-operator-' + name]: kp.prometheusOperator[name] for name in std.objectFields(kp.prometheusOperator) } +
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['prometheus-adapter-' + name]: kp.prometheusAdapter[name] for name in std.objectFields(kp.prometheusAdapter) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) }
```

The default settings below can be changed in `example.jsonnet`:

```
{
	_config+:: {
        namespace: "default",
        versions+:: {
            alertmanager: "v0.15.3",
            nodeExporter: "v0.16.0",
            kubeStateMetrics: "v1.3.1",
            kubeRbacProxy: "v0.3.1",
            addonResizer: "1.0",
            prometheusOperator: "v0.24.0",
            prometheus: "v2.4.3",
        },
        imageRepos+:: {
            prometheus: "quay.io/prometheus/prometheus",
            alertmanager: "quay.io/prometheus/alertmanager",
            kubeStateMetrics: "quay.io/coreos/kube-state-metrics",
            kubeRbacProxy: "quay.io/coreos/kube-rbac-proxy",
            addonResizer: "quay.io/coreos/addon-resizer",
            nodeExporter: "quay.io/prometheus/node-exporter",
            prometheusOperator: "quay.io/coreos/prometheus-operator",
        },
        prometheus+:: {
            names: 'k8s',
            replicas: 2,
            rules: {},
        },
        alertmanager+:: {
            name: 'main',
            config: |||
                global:
                resolve_timeout: 5m
                route:
                group_by: ['job']
                group_wait: 30s
                group_interval: 5m
                repeat_interval: 12h
                receiver: 'null'
                routes:
                - match:
                    alertname: DeadMansSwitch
                    receiver: 'null'
                receivers:
                - name: 'null'
            |||,
            replicas: 3,
        },
        kubeStateMetrics+:: {
            collectors: '',  // empty string gets a default set
            scrapeInterval: '30s',
            scrapeTimeout: '30s',
            baseCPU: '100m',
            baseMemory: '150Mi',
            cpuPerNode: '2m',
            memoryPerNode: '30Mi',
        },
	},
}
```

To create additional Prometheus alerts in `example.jsonnet`:

```
local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',
  },
  prometheusAlerts+:: {
    groups+: [
      {
        name: 'example-group',
        rules: [
          {
            alert: 'DeadMansSwitch',
            expr: 'vector(1)',
            labels: {
              severity: 'none',
            },
            annotations: {
              description: 'This is a DeadMansSwitch meant to ensure that the entire alerting pipeline is functional.',
            },
          },
        ],
      },
    ],
  },
};
```

To create additional Prometheus recording rules in `example.jsonnet`:

```
local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',
  },
  prometheusRules+:: {
    groups+: [
      {
        name: 'example-group',
        rules: [
          {
            record: 'some_recording_rule_name',
            expr: 'vector(1)',
          },
        ],
      },
    ],
  },
};
```

To add pre-rendered Prometeus rules in `example.jsonnet`:

```
local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',
    prometheus+:: {
      renderedRules: {
        'example.rules.yaml': (importstr 'example.rules.yaml'),
      },
    },
  },
};
```

To add a Grafana dashboard in `example.jsonnet`:

```
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local prometheus = grafana.prometheus;
local template = grafana.template;
local graphPanel = grafana.graphPanel;

local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',
  },
  grafanaDashboards+:: {
    'my-dashboard.json':
      dashboard.new('My Dashboard')
      .addTemplate(
        {
          current: {
            text: 'Prometheus',
            value: 'Prometheus',
          },
          hide: 0,
          label: null,
          name: 'datasource',
          options: [],
          query: 'prometheus',
          refresh: 1,
          regex: '',
          type: 'datasource',
        },
      )
      .addRow(
        row.new()
        .addPanel(graphPanel.new('My Panel', span=6, datasource='$datasource')
                  .addTarget(prometheus.target('vector(1)')))
      ),
  },
};
```

To add pre-rendered Grafana dashboards in `example.jsonnet`:

```
local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',
  },
  grafanaDashboards+:: {
    'my-dashboard.json': (import 'example-grafana-dashboard.json'),
  },
};
```

To compile the manifests and deploy the configured monitoring pipeline:

```
./build.sh example.jsonnet
kubectl create -f ./manifests
```