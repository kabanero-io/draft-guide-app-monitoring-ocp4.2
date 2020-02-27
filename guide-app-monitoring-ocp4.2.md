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

* [service_monitor.yaml](code/service_monitor.yaml)
* [service_account.yaml](code/service_account.yaml)
* [prometheus.yaml](code/prometheus.yaml)


### Install Prometheus Operator Using Operator Lifecycle Manager (OLM)

The following procedure is based on [Using the Operator Lifecycle Manager to deploy Prometheus on OpenShift](https://medium.com/faun/using-the-operator-lifecycle-manager-to-deploy-prometheus-on-openshift-cd2f3abb3511), with the added inclusion of OpenShift commands needed to complete each step.

1. Create a new namespace for our Prometheus Operator deployment
```
oc new-project prometheus-operator
```

1. Go to OpenShift Container Platform web console and Click on Operators > OperatorHub. Using the OLM, Operators can be easily pulled, installed and subscribed on the cluster. Ensure that the Project is set to prometheus-operator. Search for Prometheus Operator and install it. Choose prometheus-operator under *A specific namespace on the cluster* and subscribe.

1. Click on Overview and create a service monitor instance. A ServiceMonitor defines a service endpoint that needs to be monitored by the Prometheus instance.

1. Inside the Service Monitor YAML file, make sure **metadata.namespace** is your monitoring namespace. In this case, it will be prometheus-operator. **spec.namespaceSelector** and **spec.selector** for labels should be configured to match your app deployment's namespace and label. For example, inside the `service_monitor.yaml` file, an application with label **app: example-app** from namespace **myapp** will be monitored by the service monitor. If the metrics endpoint is secured, you can define a secured endpoint with authentication configuration by following the [endpoint](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#endpoint) API documentation of Prometheus Operator.

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

Use these files while working with Grafana:

* [grafana_datasource.yaml](code/grafana_datasource.yaml)
* [grafana.yaml](code/grafana.yaml)
* [grafana_dashboard.yaml](code/grafana_dashboard.yaml)

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
