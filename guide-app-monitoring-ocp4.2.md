---
permalink: /guides/app-monitoring-ocp4.2/
layout: guide-markdown
title: Application Monitoring on Red Hat OpenShift Container Platform (RHOCP) 4.2 with Prometheus and Grafana
duration: 60 minutes
releasedate: 2020-02-02
description: Learn how to monitor applications on RHOCP 4.2 with Prometheus and Grafana.
tags: ['monitoring', 'prometheus', 'grafana']
guide-category: monitoring
---

__This guide has been tested with RHOCP 4.2/Kabanero 0.3.0.__


For application monitoring on RHOCP, you need to set up your own Prometheus and Grafana deployments. Both Prometheus and Grafana can be set up via Operator Lifecycle Manager (OLM).

## Prerequisites

Prior to deploying Prometheus, ensure that there is a running application that has a service endpoint for outputting metrics in Prometheus format.

It is assumed such a running application has been deployed to the RHOCP cluster inside a project/namespace called `myapp`, and that the Prometheus metrics endpoint is exposed on path `/metrics`.

Git clone this repo to get going right away:
```
git clone https://github.com/Kabanero-io/guide-app-monitoring-ocp4.2.git
```

## Deploy Prometheus - Prometheus Operator

The Prometheus Operator is an open-source project originating from CoreOS and exists as a part of their Kubernetes Operator framework. The Kubernetes Operator framework is the preferred way to deploy Prometheus on a Kubernetes system. When the Prometheus Operator is installed on the Kubernetes system, you no longer need to hand-configure the Prometheus configuration. Instead, you create CoreOS ServiceMonitor resources for each of the service endpoints that needs to be monitored: this makes daily maintenance of the Prometheus server a lot easier. An architecture overview of the Prometheus Operator is shown below:

![Prometheus Operator](/img/guide/prometheusOperator.png)

Using Operator Lifecycle Manager (OLM), Prometheus operator can be easily installed and configured in RHOCP Web Console.

Use these files while working through the guide:

service_monitor.yaml:
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  labels:
    k8s-app: myapp-monitor
  namespace: prometheus-operator
spec:
  namespaceSelector:
    matchNames:
      - myapp
  selector:
    matchLabels:
      app: example-app
  endpoints:
    - interval: 30s
      path: /metrics
      port: 9080-tcp
```

service_account.yaml:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: prometheus-operator
spec:
  type: NodePort
  ports:
  - name: web
    port: 9090
    protocol: TCP
    targetPort: web
  selector:
    prometheus: prometheus
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: prometheus-operator
```

prometheus.yaml:
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  labels:
    prometheus: k8s
  namespace: prometheus-operator
spec:
  replicas: 2
  serviceAccountName: prometheus
  securityContext: {}
  serviceMonitorSelector:
    matchExpressions:
      - key: k8s-app
        operator: Exists
  ruleSelector:
    matchLabels:
      role: prometheus-rulefiles
      prometheus: k8s
  alerting:
    alertmanagers:
      - namespace: prometheus-operator
        name: alertmanager-main
        port: web
```  

### Install Prometheus Operator Using Operator Lifecycle Manager (OLM)

The following procedure is based on https://medium.com/faun/using-the-operator-lifecycle-manager-to-deploy-prometheus-on-openshift-cd2f3abb3511[Using the Operator Lifecycle Manager to deploy Prometheus on OpenShift], with the added inclusion of OpenShift commands needed to complete each step.

1. Create a new namespace for our Prometheus Operator deployment
```
oc new-project prometheus-operator
```

1. Go to OpenShift Container Platform web console and Click on Operators > OperatorHub. Using the OLM, Operators can be easily pulled, installed and subscribed on the cluster. Ensure that the Project is set to prometheus-operator. Search for Prometheus Operator and install it. Choose prometheus-operator under *A specific namespace on the cluster* and subscribe.

1. Click on Overview and create a service monitor instance. A ServiceMonitor defines a service endpoint that needs to be monitored by the Prometheus instance.

1. Inside the Service Monitor YAML file, make sure **metadata.namespace** is your monitoring namespace. In this case, it will be prometheus-operator. **spec.namespaceSelector** and **spec.selector** for labels should be configured to match your app deployment's namespace and label. For example, inside the `service_monitor.yaml` file, an application with label **app: example-app** from namespace **myapp** will be monitored by the service monitor. If the metrics endpoint is secured, you can define a secured endpoint with authentication configuration by following the https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#endpoint[endpoint] API documentation of Prometheus Operator.

1. Create a Service Account with Cluster role and Cluster role binding to ensure you have the permission to get nodes and pods in other namespaces at the cluster scope. Refer to the `service_account.yaml` file. Create the YAML file and apply it.
```
oc apply -f service_account.yaml
```

1. Click Overview and create a Prometheus instance. A Prometheus resource can scrape the targets defined in the ServiceMonitor resource.

1. Inside the Prometheus YAML file, make sure **metadata.namespace** is prometheus-operator. Ensure **spec.serviceAccountName** is the Service Account's name that you have applied in the previous step. You can set the match expression to select which Service Monitors you are interested in under **spec.serviceMonitorSelector.matchExpressions** as in the `prometheus.yaml` file.

1. Verify that the Prometheus services have successfully started.
```
[root@rhel7-ocp]# oc get svc -n prometheus-operator
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
prometheus-operated   ClusterIP   None             <none>        9090/TCP         19h
```

1. Check the server logs from one of the target pods to see if the services are running properly.
```
[root@rhel7-ocp]# oc get pods -n prometheus-operator
NAME                                   READY     STATUS    RESTARTS   AGE
prometheus-operator-7fccbd7c74-48m6v   1/1       Running   0          19h
prometheus-prometheus-0                3/3       Running   1          19h
prometheus-prometheus-1                3/3       Running   1          19h
[root@rhel7-ocp]# oc logs prometheus-prometheus-0 -c prometheus -n prometheus-operator
```

1. Expose the prometheus-operated service to use the Prometheus console externally.
```
[root@rhel7-ocp]# oc expose svc/prometheus-operated -n prometheus-operator
route.route.openshift.io/prometheus-operated exposed
[root@rhel7-ocp]# oc get route -n prometheus-operator
NAME         HOST/PORT                                                 PATH      SERVICES     PORT      TERMINATION   WILDCARD
prometheus   prometheus-prometheus-operator.apps.9.37.135.153.nip.io             prometheus   web                     None
```

1. Visit the Prometheus route and go to the Prometheus targets page.
Check to see that the Prometheus targets page is picking up the target endpoints.

![Prometheus Target Page](/img/guide/prometheus_endpoints.png)


## Deploy Grafana

Refer to these files while working with Grafana:

grafana_datasource.yaml
```
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDataSource
metadata:
  name: grafana-datasource
  namespace: prometheus-operator
spec:
  datasources:
    - access: proxy
      editable: true
      isDefault: true
      jsonData:
        timeInterval: 5s
      name: Prometheus
      type: prometheus
      url: 'http://prometheus-operated:9090'
      version: 1
  name: grafana-datasources.yaml
```

grafana.yaml
```
apiVersion: integreatly.org/v1alpha1
kind: Grafana
metadata:
  name: grafana
  namespace: prometheus-operator
spec:
  ingress:
    enabled: true
  config:
    auth:
      disable_signout_menu: true
    auth.anonymous:
      enabled: true
    log:
      level: warn
      mode: console
  dashboardLabelSelector:
    - matchExpressions:
        - key: app
          operator: In
          values:
            - grafana
```

grafana_dashboard.yaml
```
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDashboard
metadata:
  labels:
    app: grafana
  name: template-dashboard
  namespace: prometheus-operator
spec:
  json: |
    {
      "__inputs": [
         {
           "name": "Prometheus",
           "label": "Prometheus",
           "description": "",
           "type": "datasource",
           "pluginId": "prometheus",
           "pluginName": "Prometheus"
         }
       ],
      "__requires": [
         {
           "type": "grafana",
           "id": "grafana",
           "name": "Grafana",
           "version": "5.2.0"
         },
         {
           "type": "panel",
           "id": "graph",
           "name": "Graph",
           "version": "5.0.0"
         },
         {
           "type": "datasource",
           "id": "prometheus",
           "name": "Prometheus",
           "version": "5.0.0"
         },
         {
           "type": "panel",
           "id": "table",
           "name": "Table",
           "version": "5.0.0"
         }
       ],
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": 1,
      "iteration": 1569353980677,
      "links": [],
      "panels": [
        {
          "columns": [],
          "datasource": "Prometheus",
          "fontSize": "100%",
          "gridPos": {
            "h": 6,
            "w": 24,
            "x": 0,
            "y": 2
          },
          "id": 26,
          "links": [],
          "options": {},
          "pageSize": null,
          "scroll": true,
          "showHeader": true,
          "sort": {
            "col": 0,
            "desc": true
          },
          "styles": [
            {
              "alias": "Time",
              "dateFormat": "YYYY-MM-DD HH:mm:ss",
              "pattern": "Time",
              "type": "date"
            },
            {
              "alias": "Namespace",
              "pattern": "namespace",
              "type": "custom"
            },
            {
              "alias": "Service",
              "pattern": "service",
              "type": "custom"
            },
            {
              "alias": "Endpoint",
              "pattern": "endpoint",
              "type": "custom"
            },
            {
              "alias": "",
              "mappingType": 1,
              "pattern": "Value",
              "type": "hidden",
              "unit": "short"
            }
          ],
          "targets": [
            {
              "expr": "count(up) by (namespace, service, endpoint)",
              "format": "table",
              "instant": true,
              "intervalFactor": 1,
              "refId": "A"
            }
          ],
          "title": "Endpoints",
          "transform": "table",
          "type": "table"
        }
      ],
      "refresh": "10s",
      "schemaVersion": 19,
      "style": "dark",
      "tags": [],
      "time": {
        "from": "now-15m",
        "to": "now"
      },
      "timepicker": {
        "refresh_intervals": [
          "1m",
          "5m",
          "15m",
          "1h",
          "1d"
        ]
      },
      "title": "Template-Dashboard",
      "uid": "Template-Dashboard"
    }
  name: template-dashboard.json
```

Use Grafana dashboards to visualize the metrics. Perform the following steps to deploy Grafana and ensure that Prometheus endpoints are reachable as a data source in Grafana.

1. Choose the *same namespace* as Prometheus Operator deployment.
```
oc project prometheus-operator
```

1. Go to OpenShift Container Platform web console and Click on Operators > OperatorHub. Search for Grafana Operator and install it. Choose prometheus-operator under **A specific namespace on the cluster** and subscribe.

1. Click on Overview and create a Grafana Data Source instance.

1. Inside the Grafana Data Source YAML file, make sure **metadata.namespace** is prometheus-operator. Set **spec.datasources.url** to the URL of the target datasource. For example, inside the `grafana_datasource.yaml` file, the Prometheus service is **prometheus-operated** on port **9090**, so the URL is set to `http://prometheus-operated:9090`.

1. Click Overview and create a Grafana instance.

1. Inside the Grafana YAML file, make sure **metadata.namespace** is prometheus-operator. You can define the match expression to select which Dashboards you are interested in under **spec.dashboardLabelSelector.matchExpressions**. For example, inside the `grafana.yaml` file, the Grafana will discover dashboards with app labels having a value of **grafana**.

1. Click Overview and create a Grafana Dashboard instance.

1. Copy the `grafana_dashboard.yaml` file to Grafana Dashboard YAML file to check the Data Source is connected and Prometheus endpoints are discoverable.

1. Click Networking > Routes and go to Grafana's location to see the template dashboard. You can now consume all the application metrics gathered by Prometheus on the Grafana dashboard.

![Template Dashboard](/img/guide/template_grafana_dashboard.png)

1. When importing your own Grafana dashboard, your dashboard should be configured under **spec.json** in Grafana Dashboard YAML file. Make sure under **"__inputs"**, the name matches with your Grafana Data Source's **spec.datasources**. For example, inside the `grafana_dashboard.yaml` file, **name** is set to "Prometheus".
