---
layout: post
title: Simple Anti-affinity example
date: 2020-12-26
categories: [containers, kubernetes, antiaffinity]
---

Kubernetes anti-affinity rules can become fairly complex. Here is a very simple example in preventing all of your pods from a single deployment to end up on the same node.

My cluster has 3 RPI 4GB workers + 1 that is 8GB. Sometimes if I create a deployment with 3 replicas, all 3 land on the 8GB Pi. That doesn't provide me any extra resiliency and just bogs down my 8GB Pi.

Kubernetes Affinity/AntiAffinity groups ensure pods are/aren't scheduled together based on a label, and a topology (also a label, but on a node). This can be as simple as having a small deployment not schedule every pod on the same worker node, or something more complicated like keeping applications/caching pods grouped together while also ensuring instances are separated across data centers.

Given a simple deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: igou-website
  namespace: igou
  labels:
    igou-app: website
spec:
  replicas: 4
  selector:
    matchLabels:
      igou-app: website
  template:
    metadata:
      labels:
        igou-app: website
    spec:
      containers:
      - env:
        image: quay.io/igou/igou.io:latest
        imagePullPolicy: Always
        name: igou-website
[etc]
```

If you create this deployment, there are no protections in place to keep all your replicas off one node. Let's `kubectl explain` the field that we need:

```
$ k explain pod.spec.affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution
KIND:     Pod
VERSION:  v1

RESOURCE: requiredDuringSchedulingIgnoredDuringExecution <[]Object>

DESCRIPTION:
     If the anti-affinity requirements specified by this field are not met at
     scheduling time, the pod will not be scheduled onto the node. If the
     anti-affinity requirements specified by this field cease to be met at some
     point during pod execution (e.g. due to a pod label update), the system may
     or may not try to eventually evict the pod from its node. When there are
     multiple elements, the lists of nodes corresponding to each podAffinityTerm
     are intersected, i.e. all terms must be satisfied.

     Defines a set of pods (namely those matching the labelSelector relative to
     the given namespace(s)) that this pod should be co-located (affinity) or
     not co-located (anti-affinity) with, where co-located is defined as running
     on a node whose value of the label with key <topologyKey> matches that of
     any node on which a pod of the set of pods is running

FIELDS:
   labelSelector	<Object>
     A label query over a set of resources, in this case pods.

   namespaces	<[]string>
     namespaces specifies which namespaces the labelSelector applies to (matches
     against); null or empty list means "this pod's namespace"

   topologyKey	<string> -required-
     This pod should be co-located (affinity) or not co-located (anti-affinity)
     with the pods matching the labelSelector in the specified namespaces, where
     co-located is defined as running on a node whose value of the label with
     key topologyKey matches that of any node on which any of the selected pods
     is running. Empty topologyKey is not allowed.
```

Okay, so we need an expression that matches a label that the pods will have; pods already have the label `igou-app: website`, so we can use that. We also need and a "topologyKey" which is just a label key to use for separation. An easy one is just `kubernetes.io/hostname` - separation across hostname.

Our affinity field should look something like this:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - topologyKey: kubernetes.io/hostname
      labelSelector:
        matchExpressions:
        - key: igou-app
          operator: In
          values:
          - website
```

Or, inside our new deployment object:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: igou-website
  namespace: igou
  labels:
    igou-app: website
spec:
  replicas: 4
  selector:
    matchLabels:
      igou-app: website
  template:
    metadata:
      labels:
        igou-app: website
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchExpressions:
              - key: igou-app
                operator: In
                values:
                - website
      containers:
      - env:
        image: quay.io/igou/igou.io:latest
        imagePullPolicy: Always
        name: igou-website
        ports:
        - containerPort: 80
[etc]
```
