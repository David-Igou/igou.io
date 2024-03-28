---
layout: post
date: 2019-12-31
title: "Nginx metrics via exporter sidecar"
categories: [nginx, prometheus, kubernetes, grafana]
---


If your static website runs on Kubernetes, here is how to export metrics and have them scraped into Prometheus.

Metrics need to be enabled on the nginx side, I do it via mounting a configmap into /etc/nginx/conf.d. 

This kind of implementation warrants discussion - From what I can see, there are three ways to go about this:

* Exporter and nginx in the same container
* Exporter and nginx in the same pod (this)
* Exporter and nginx in different pods, /nginx_status shared via a separate service

Each have pros and cons in terms of reliability and performance. But it doesn't really matter in this context due to how simple everything is. 


{% codetabs %}

{% codetab Deployment %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: igou-website
  namespace: igou
  labels:
    igou-app: website
spec:
  replicas: 3
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
        image: quay.io/igou/igou.io-nginx:latest
        imagePullPolicy: Always
        name: igou-website
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            scheme: HTTP
            path: /
            port: 80
          initialDelaySeconds: 30
          timeoutSeconds: 30
        volumeMounts:
        - mountPath: /etc/nginx/conf.d/nginx-status.conf
          name: nginx-status-conf
          readOnly: true
          subPath: nginx.status.conf
      - name: nginx-exporter
        image: 'nginx/nginx-prometheus-exporter:0.3.0'
        args:
          - '-nginx.scrape-uri=http://localhost:8090/nginx_status'
        ports:
          - name: nginx-ex-port
            containerPort: 9113
            protocol: TCP
        imagePullPolicy: Always
      volumes:
      - configMap:
          defaultMode: 420
          name: nginx-status-conf
        name: nginx-status-conf
```
{% endcodetab %}

{% codetab Configmap %}
```yaml
apiVersion: v1
data:
  nginx.status.conf: |
    server {
        listen       8090 default_server;
        location /nginx_status {
            stub_status;
            access_log off;
        }
    }
kind: ConfigMap
metadata:
  name: nginx-status-conf
  namespace: igou
```
{% endcodetab %}

{% codetab Service %}
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    igou-app: website
  name: igou-website
  namespace: igou
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
  - port: 9113
    protocol: TCP
    targetPort: 9113
    name: metrics
  selector:
    igou-app: website
  sessionAffinity: None
  type: ClusterIP
```
{% endcodetab %}

{% codetab Servicemonitor %}
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: igou
  namespace: igou
spec:
  selector:
    matchLabels:
      igou-app: website
  endpoints:
  - port: metrics
    interval: 30s
```
{% endcodetab %}

{% codetab  Ingress %}
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    traefik.ingress.kubernetes.io/redirect-entry-point: https
  labels:
    igou-app: website
  name: igou-website
  namespace: igou
spec:
  rules:
  - host: igou.io
    http:
      paths:
      - backend:
          serviceName: igou-website
          servicePort: 80
  - host: www.igou.io
    http:
      paths:
      - backend:
          serviceName: igou-website
          servicePort: 80
  tls:
  - secretName: igou-blog-acme-certificate
```
{% endcodetab %}
{% endcodetabs %}


