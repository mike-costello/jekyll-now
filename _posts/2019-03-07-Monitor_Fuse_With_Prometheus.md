---
layout: post
title: Monitoring Fuse MicroServices With the Prometheus Operator in Openshift
tags: Fuse Camel Prometheus Monitoring 
eye_catch: /assets/img/prometheus/logo_prometheus.png
---

## Monitoring MicroServices and Prometheus 
Prometheus has established itself as mainstay for application monitoring, alerting, and safe keeping of our key operational behaviours by maintaining a time series store of metrics from our application containers. As its lightweight, and focuses on high availability, Prometheus when deployed to a container platform can provide a cloud native means to monitoring application runtimes in our containers.  

For distributed integration platforms, this need becomes even greater as our MicroService based architectures may imply many pods choreograph according to the dictates of our integration. As a result, being able to asses health of our applications, operational behaviour, and longitudinal trending across our enterprise and act on those indicators becomes a mission critical factor in our MicroService orchestration in container platforms. 

### The Setup 
It is worth noting, the following techniques as described in this article, do not require use of anything but Kubernetes (k8s), the CoreOS Prometheus Operator, and a Grafana instance.

For the purposes of this demonstration; however, we will use:
* Minishift With the latest CDK 3.11.0 installed (OKD 3.11.0)
* Red Hat Fuse Springboot Camel images 
* Productized resource templates from Red Hat for preparation of our Prometheus Operator 
* The Prometheus Operator as released by CoreOS

#### Minishift configuration    
As our purposes are for demonstration purposes only, we're going to use Minishift, and as our resources will be fairly lightweight, we'll simply use a default profile for minishift and allow it to size itself accordingly. Please note, to extend this example out further, or to really get a better feel for what many microservices would look like in your container platform and monitored by Prometheus it is advised to establish more resources for the Minishift VM: https://docs.okd.io/latest/minishift/using/profiles.html 

Upon starting up Minishift

```python 

```
Our prometheus install is going to consist of the following: 
* A Prometheus instance with 3 replicas 
* A Fuse based MicroService (using Camel's Rest component). We've got one here: 

```yaml
test.test=test
```

### The Prometheus Operator 

* List of different things to accomplish
* bar
    * indentation
        * nesting indentation
    * indentation
* buz

<!--more-->

## Minishift Time 

> quote
>
> > nesting quote
>
> quote

## Strikethrough

~~Mistaken text.~~

## Syntax highlighting

```php
<?php
echo 'Hello, World!';
```

## Tables

First Header  | Second Header
------------- | -------------
Content Cell  | Content Cell
Content Cell  | Content Cell

## Emoji

You can use GitHub flavored emoji :+1:

> **Note**  
> It's not a very good idea to use emoji before `<!--more-->` because jekyll can't render emoji in the excerpted content.

## See also

[Markdown](http://daringfireball.net/projects/markdown/syntax)
