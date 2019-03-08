---
layout: post
title: Monitoring Fuse MicroServices With the Prometheus Operator in Openshift
tags: Fuse Camel Prometheus Monitoring 
eye_catch: /assets/img/prometheus/logo_prometheus.png
---

# Monitoring MicroServices and Prometheus 
Prometheus has established itself as mainstay for application monitoring, alerting, and safe keeping of our key operational behaviours by maintaining a time series store of metrics from our application containers. As its lightweight, and focuses on high availability, Prometheus when deployed to a container platform can provide a cloud native means to monitoring application runtimes in our containers.  

For distributed integration platforms like Red Hat Fuse, this need becomes even greater as our MicroService based architectures may imply many pods choreograph according to the dictates of our integration which may span many integration endpoints across many pods. As a result, being able to asses health of our applications, operational behaviour, and longitudinal trending across our enterprise and act on those indicators becomes a mission critical factor in our MicroService orchestration in container platforms. 

## The Setup 
It is worth noting, the following techniques as described in this article, do not require use of anything but Kubernetes (k8s), the CoreOS Prometheus Operator, and a Grafana instance.

For the purposes of this demonstration; however, we will use:
* Minishift With the latest CDK 3.11.0 installed (OKD 3.11.0)
* Red Hat Fuse Springboot Camel images (https://access.redhat.com/containers/?tab=images&platform=openshift#/registry.access.redhat.com/fuse7/fuse-java-openshift) 
* Productized resource templates from Red Hat for preparation of our Prometheus Operator (https://github.com/jboss-fuse/application-templates)
* The Prometheus Operator as released by CoreOS (https://coreos.com/operators/prometheus/docs/latest/)

#### Minishift configuration    
As our purposes are for demonstration only, we'll use Minishift, and as our resources will be fairly lightweight, we'll simply use a default profile for minishift and allow it to size itself accordingly. Please note, to extend this example out further, or to really get a better feel for what many microservices would look like in your container platform and monitored by Prometheus it is advised to establish more resources for the Minishift VM: https://docs.okd.io/latest/minishift/using/profiles.html 

## The Prometheus Operator 
The Prometheus Operator leverages the Operator SDK "as a class of software that operates other software, putting operational knowledge collected by humans into software" (https://coreos.com/blog/introducing-operators.html). Through the use of the operator framework, the Prometheus Operator is able to install a Service Account (prometheus), install a replicated cluster of time series databases, and install itself to manage state of Prometheus over its lifecycle. 

#### Prometheus CRD's
Custom Resource Definitions (CRD's) in Kubernetes are a means to extend out Kubernetes API resources to introduce into a project or cluster. The Prometheus Operator uses CRD's as they are introduced as new resources into the cluster to perform operations such as adding service monitors for new types of applications, to configure rules to run against metrics collected from service monitors, or to configure alerts to send to the Prometheus Alert Manager. 

For use of the prometheus operator, we'll extend Kubernetes to install the following crd's https://raw.githubusercontent.com/jboss-fuse/application-templates/master/fuse-prometheus-crd.yml: 
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: prometheusrules.monitoring.coreos.com
spec:
  group: monitoring.coreos.com
  names:
    kind: PrometheusRule
    listKind: PrometheusRuleList
    plural: prometheusrules
    singular: prometheusrule
  scope: Namespaced
  version: v1
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: servicemonitors.monitoring.coreos.com
spec:
  group: monitoring.coreos.com
  names:
    kind: ServiceMonitor
    listKind: ServiceMonitorList
    plural: servicemonitors
    singular: servicemonitor
  scope: Namespaced
  version: v1
--- 
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition 
metadata:
  name: prometheuses.monitoring.coreos.com
spec:    
  group: monitoring.coreos.com
  names:
    kind: Prometheus
    listKind: PrometheusList
    plural: prometheuses
    singular: prometheus
  scope: Namespaced
  version: v1
--- 
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: alertmanagers.monitoring.coreos.com
spec:
  group: monitoring.coreos.com
  names:
    kind: Alertmanager
    listKind: AlertmanagerList
    plural: alertmanagers
    singular: alertmanager
  scope: Namespaced
  version: v1
```
These resource definitions allow our Prometheus Operator to use Kubernetes resources to establish an imperative model for managing Kubernetes resources. 

#### Installing the Prometheus CRD's 
As we will need a cluster admin role binding to install CRD's across the cluster, we will login to our minishift cluster as follows: 
```py
oc login -u system:admin
```
We'll create a new namespace in OpenShift for our Prometheus deployment
```py 
oc new-project prometheus-dev
```

We'll use the Red Hat productized yaml resources to create the CRD's and our Operator; however, it is worth noting, that these yaml templates do not require use of Red Hat software, and simply install the CoreOS Operator. As result, we will issue the following command to the oc cluster: 
```py
oc create -f https://raw.githubusercontent.com/jboss-fuse/application-templates/master/fuse-prometheus-crd.yml
```

This will install the above custom resources and enable our Prometheus Operator to install Prometheus and other operational capabilities Prometheus requires over its lifecycle. To install the Prometheus Operator we'll apply another Red Hat Fuse yaml template which will introduce a Service Account for our Prometheus runtime, a Service Account for the Prometheus operator, and everything required to install Prometheus. 

```py 
oc process -f https://raw.githubusercontent.com/jboss-fuse/application-templates/master/fuse-prometheus-operator.yml | oc create -f -
```

After a couple of minutes (all depending), we should see a few things installed into the "prometheus-dev" project in our cluster: 

```py
[mcostell@work /]$ oc get pods -w 
NAME                                  READY     STATUS    RESTARTS   AGE
prometheus-operator-8586c4688-8m2kk   1/1       Running   0          4h
prometheus-prometheus-0               3/3       Running   1          4h
```

We'll also see 2 new Service Accounts installed in the "prometheus-dev" namespace: 

```py
[mcostell@work /]$ oc get serviceaccounts -n prometheus-dev
NAME                  SECRETS   AGE
builder               2         4h
default               2         4h
deployer              2         4h
prometheus            2         4h
prometheus-operator   2         4h
```
At this point, we'll also notice the Prometheus Operator has installed a route for the Prometheus console. Upon issuing this command: 

```py
[mcostell@work /]$ oc get routes 
NAME         HOST/PORT                                         PATH      SERVICES     PORT      TERMINATION   WILDCARD
prometheus   prometheus-prometheus-dev.192.168.42.118.nip.io             prometheus   <all>                   None

```
Will deliver the following Prometheus Console that currently does not have anything under management. 

![Prometheus Console](/assets/img/prometheus/prometheus.initial.png "Prometheus Console")

## Configuring ServiceMonitors