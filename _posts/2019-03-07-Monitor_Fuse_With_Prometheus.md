---
layout: post
title: Monitoring Fuse MicroServices With the Prometheus Operator in Openshift
tags: Fuse Camel Prometheus Monitoring 
eye_catch: /assets/img/prometheus/logo_prometheus.png
---

## Monitoring MicroServices and Prometheus 
Prometheus has established itself as mainstay for application monitoring, alerting, and safe keeping of our key operational behaviours by maintaining a time series store of metrics from our application containers. As its lightweight, and focuses on high availability, Prometheus when deployed to a container platform can provide a cloud native means to monitoring application runtimes in our containers.  

For distributed integration platforms like Red Hat Fuse, this need becomes even greater as our MicroService based architectures may imply many pods choreograph according to the dictates of our integration which may span many integration endpoints across many pods. As a result, being able to asses health of our applications, operational behaviour, and longitudinal trending across our enterprise and act on those indicators becomes a mission critical factor in our MicroService orchestration in container platforms. 

### The Setup 
It is worth noting, the following techniques as described in this article, do not require use of anything but Kubernetes (k8s), the CoreOS Prometheus Operator, and a Grafana instance.

For the purposes of this demonstration; however, we will use:
* Minishift With the latest CDK 3.11.0 installed (OKD 3.11.0)
* Red Hat Fuse Springboot Camel images (https://access.redhat.com/containers/?tab=images&platform=openshift#/registry.access.redhat.com/fuse7/fuse-java-openshift) 
* Productized resource templates from Red Hat for preparation of our Prometheus Operator (https://github.com/jboss-fuse/application-templates)
* The Prometheus Operator as released by CoreOS (https://coreos.com/operators/prometheus/docs/latest/)

#### Minishift configuration    
As our purposes are for demonstration only, we'll use Minishift, and as our resources will be fairly lightweight, we'll simply use a default profile for minishift and allow it to size itself accordingly. Please note, to extend this example out further, or to really get a better feel for what many microservices would look like in your container platform and monitored by Prometheus it is advised to establish more resources for the Minishift VM: https://docs.okd.io/latest/minishift/using/profiles.html 

### The Prometheus Operator 
The Prometheus Operator leverages the Operator SDK "as a class of software that operates other software, putting operational knowledge collected by humans into software" (https://coreos.com/blog/introducing-operators.html). Through the use of the operator framework, the Prometheus Operator is able to install a Service Account (prometheus), install a replicated cluster of time series databases, and install itself to manage state of Prometheus over its lifecycle. 

#### Prometheus CRD's
Custom Resource Definitions (CRD's) in Kubernetes are a means to extend out Kubernetes API resources to introduce into a project or cluster. The Prometheus Operator uses CRD's as they are introduced as new resources into the cluster to perform operations such as adding service monitors for new types of applications, to configure rules to run against metrics collected from service monitors, or to configure alerts to send to the Prometheus Alert Manager.  

