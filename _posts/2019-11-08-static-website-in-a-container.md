---
layout: post
title: "Running a static website on Kubernetes"
date: 2019-11-09
categories: [html, static website, kubernetes, cert-manager, k3s, traefik]
---


Jekyll generates my site to a directory `_site`

```bash
[igou]$ ls _site/
archives  atom.xml  blog  contact  dale  david  favicon.png  font  fonts  images  index.html  javascripts  robots.txt  sitemap.xml  stylesheets
```

Building this into a container is pretty simple:

```dockerfile
FROM nginx:alpine
COPY _site /usr/share/nginx/html
```

To test/push:

```bash
[igou]$ docker build -t igou.io-nginx .
[igou]$ docker run --name igou.io-nginx -d -p 8080:80 igou.io-nginx
[igou]$ docker ps
[igou]$ docker commit [id] quay.io/igou/igou.io-nginx
[igou]$ docker push quay.io/igou/igou.io-nginx
```

To deploy it on Kubernetes:

Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: igou
  name: igou-website
  labels:
    igou-app: website
spec:
  replicas: 3 #Don't talk to me unless your static website is HA
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

```

Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    igou-app: website
  name: igou-website
  namespace: igou
spec:
  clusterIP: 10.43.227.80
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    igou-app: website
  sessionAffinity: None
  type: ClusterIP

```

Ingress:

```yaml
apiVersion: v1
items:
- apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    labels:
      igou-app: website
    name: igou-website
    namespace: igou
  spec:
    rules:
    - host: websitetest.igou.io
      http:
        paths:
        - backend:
            serviceName: igou-website
            servicePort: 80
```

So let's create them

```
[igou]$ kubectl create igou-blog-deployment.yml
[igou]$ kubectl create igou-blog-service.yml
[igou]$ kubectl create igou-blog-ingress.yml
```

A lazy hack to recreate the pods and pull the images again:

```
kubectl patch deploy igou-website -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"
```

Ideally, some CI/CD will be in place. You can only overengineer a static website so much in a week. It's on my to-do list.

# Certificates with cert-manager

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: igou-website-certificate
  namespace: igou
spec:
  commonName: igou.io
  dnsNames:
  - igou.io
  - www.igou.io
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-prod
  secretName: igou-blog-acme-certificate
```

Create the manifest and wait until it's READY

```shell
[igou@igou-rh manifests]$ kubectl get cert -n igou
NAME                       READY   SECRET                       AGE
igou-website-certificate   True    igou-blog-acme-certificate   17m
[igou@igou-rh manifests]$ kubectl describe cert -n igou
Name:         igou-website-certificate
Namespace:    igou
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1alpha2
Kind:         Certificate
Metadata:
  Creation Timestamp:  2019-11-08T16:29:44Z
  Generation:          1
  Resource Version:    313630
  Self Link:           /apis/cert-manager.io/v1alpha2/namespaces/igou/certificates/igou-website-certificate
  UID:                 ed8ad018-5791-4da2-9890-ff4343
Spec:
  Common Name:  igou.io
  Dns Names:
    igou.io
    www.igou.io
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  igou-blog-acme-certificate
Status:
  Conditions:
    Last Transition Time:  2019-11-08T16:30:13Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2020-02-06T15:30:12Z
Events:
  Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  GeneratedKey  17m   cert-manager  Generated a new private key
  Normal  Requested     17m   cert-manager  Created new CertificateRequest resource "igou-website-certificate-2819406579"
  Normal  Issued        17m   cert-manager  Certificate issued successfully
```

Make an ingress using the secret generated:

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


