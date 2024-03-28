---
layout: post
title: "Securing a Kubernetes ingress with htpasswd"
date: 2019-11-09
categories: [kubernetes, ingress, route, secure, htpasswd, basic-auth]
---

Here's a pretty easy example for adding basic password auth to a Kubernetes ingress

```shell
$ htpasswd -nb david stinkysocks
david:$apr1$dxwaFeYS$Tt3D4YsaFyja1W1zPPXUh0
$ echo -n `david:$apr1$dxwaFeYS$Tt3D4YsaFyja1W1zPPXUh0` |base64
ZGF2aWQ6JGFwcjEkZHh3ekZlFUMEzULTRAc2FGeGREATSWORDxelBQWFVoMA==
```

Stick it in a secret

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: ingress-auth
data:
  auth: ZGF2aWQ6JGFwcjEkZHh3ekZlFUMEzULTRAc2FGeGREATSWORDxelBQWFVoMA==
```

Add these annotations to your ingress:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/redirect-entry-point: https
    ingress.kubernetes.io/auth-secret: ingress-auth
    ingress.kubernetes.io/auth-type: basic
  name: grafana-ingress
  namespace: monitoring
spec:
  rules:
  - host: dashboard.myorg.com
    http:
      paths:
      - backend:
          serviceName: grafana
          servicePort: 3000
  tls:
  - secretName: grafana-acme-certificate
```



