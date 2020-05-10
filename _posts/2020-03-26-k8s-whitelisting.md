---
layout: post
title: Kubernetes ingress whitelisting behind a loadbalancer
date: 2020-03-26
categories: [haproxy, whitelisting, security, cyber, secret squirrel]
---

I'm stuck in my house indefinitely so it looks like I have no excuse not to write blogs. Here is how I got white/blacklisting to work on my "Personal Hybrid Cloud" Kubernetes cluster using Traefik.

Currently how applications are resolved in my cluster the network flow follows: `resolve app.subdomain -> haproxy -> kubernetes node running traefik svclb -> traefik pod -> pods mapped via service` Out of the box, if you look at the source IP for incoming traffic from the pod, it will always appear as the IP mapped to the traefik pod.

Here's a log entry for the pod this blog runs in:

`2020-03-26T14:55:39.417321686Z 10.42.13.34 - - [26/Mar/2020:14:55:39 +0000] "GET /images/OCP_AWS.png HTTP/1.1" 304 0 "-" "Mozilla/5.0 (compatible; YandexImages/3.0; +http://yandex.com/bots)"`

Verify this IP:

```shell
[igou@igou-rh ~]$ k get pods -o wide -n kube-system | grep traefik
traefik-6b545dd9dc-2z9k9                1/1     Running     0          3d14h   10.42.13.34   worker-2
```

This isn't super useful. What featureset allows us to see where traffic is coming from?

# Configuring the Proxy Protocol

The [Proxy Protocol](http://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) allows us to circumvent this. In a nutshell this is just a handshake between the loadbalancer and server that allows the loadbalancer to pass client information. Without this, the server would only know that its traffic was coming from the loadbalancer IP and nothing else. 

Traefik needs to know it is receiving traffic via the Proxy Protocol, the loadbalancer IP needs to be whitelisted, and loadbalancer needs to be configured to send Proxy traffic.

For Traefik deployed via Helm Chart:

```yaml
kind: HelmChart
apiVersion: helm.cattle.io/v1
metadata:
  finalizers:
    - wrangler.cattle.io/helm-controller
  name: traefik
  namespace: kube-system
spec:
  chart: 'https://%{KUBERNETES_API}%/static/charts/traefik-1.81.0.tgz'
  set:
    externalIP: <Loadbalancer Public IP>
    kubernetes.ingressEndpoint.useDefaultPublishedService: 'true'
  valuesContent: |-
    proxyProtocol:
      enabled: true
      trustedIPs:
      - <Loadbalancer Private IP>
    forwardedHeaders:
      enabled: true
      trustedIPs:
      - 10.0.0.0/8
```

On the HAProxy side:

```
frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server <node running svclb pod private ip> <node running svclb pod private ip>:443 check send-proxy 
```

Restarting the services/rolling out Traefik again is required. There might be some downtime here (I observed some) but eventually it figured itself out and my ingress routes all came back up. You might have to clear your browser cache.

If this works correctly, in the context of nginx, the original address should now show up in your logs:

```bash
10.42.12.35 - - [06/Apr/2020:21:59:14 +0000] "GET /images/dale.jpg HTTP/1.1" 200 436176 "https://igou.io/dale/" "Mozilla/5.0 (X11; Fedora; Linux x86_64; rv:74.0) Gecko/20100101 Firefox/74.0" "73.42.22.33"
```

Now that Traefik knows the source of traffic, we can set whitelists.

Now add an annotation to your ingress to whitelist based on x-forwarded-for address:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/whitelist-x-forwarded-for: "true"
    traefik.ingress.kubernetes.io/whitelist-source-range: 13.37.13.17/32
  name: secret-ingress
  namespace: secret-namespace
spec:
  rules:
  - host: secret.igou.io
    http:
      paths:
      - backend:
          serviceName: secret-service
          servicePort: 80
```


