---
layout: post
title: "Prometheus"
date: 2018-10-26 8:27:13 -0400
comments: true
categories:
---

Some time last year Prometheus became a [technical preview](https://docs.openshift.com/container-platform/3.7/release_notes/ocp_3_7_release_notes.html) for Openshift. That same month I rolled onto a project with some pretty steep architectural layouts of handling metrics. I spent several sprints hacking out of the box features and configurations into Openshift's Prometheus deployment. All of this changed towards the end with the introduction of the [Monitoring Operator](https://github.com/openshift/cluster-monitoring-operator). I'll be writing about that at a later date, I'm still working out some kinks in my home lab.

>Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. Since its inception in 2012, many companies and organizations have adopted Prometheus, and the project has a very active developer and user community. It is now a standalone open source project and maintained independently of any company. To emphasize this, and to clarify the project's governance structure, Prometheus joined the Cloud Native Computing Foundation in 2016 as the second hosted project, after Kubernetes.

<!-- more -->

Pretty juicy toolset. Ships with alerting, scales well, and focuses on reliability. There are some caveats consumers need to be aware of, though. In particular certain aspects are left in their responsibility.

__Use__: I want to take detailed metrics of this ec2 instance. I have a Kubernetes cluster running on an array of Raspberry Pi's in my apartment. This has its own dedicated storage (Containerized Gluster). Metrics are pretty storage intensive, and storage is expensive in the cloud.

__Architecture__: Prometheus can scrape another instance of Prometheus and apply labels. I can have a small data retention policy on the cloud instance, and a large one on my on site instance.

__Problem__: Prometheus isn't secured out of the box. I need my metrics behind some sort of gateway, and it's best I use HTTPS.

First install Prometheus. This varies by OS. The docs recommend a manual installation, but I was able to apt-get them on Ubuntu.

``` bash
apt-get install prometheus prometheus-node-exporter
```

There are additional exporters listed [here.](https://prometheus.io/docs/instrumenting/exporters/)

The Ubuntu package comes as an init.d script so I quickly wrote a Service for Prometheus so I could pass some CLI flags. I didn't bother for the node exporter since it did what I wanted out of the box.

``` text /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --web.external-url="http://igou.io/prometheus" #To enlighten upon
    --storage.tsdb.retention=8h #This can be lower. This data won't live on my ec2 very long.

[Install]
WantedBy=multi-user.target
```

I can fire this up now and expose a port of my choice but it'd be open to the public. Let's instead reverse proxy with HTTPS and password auth. Inside my virtualhost I added:

``` text Virtualhost
ProxyPreserveHost On
ProxyPass /prometheus http://localhost:9090/prometheus
ProxyPassReverse /prometheus http://localhost:9090/prometheus
ServerName igou.io
SSLCertificateFile /path/to/certfile.pem
SSLCertificateKeyFile /path/to/privatekey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
<Location /prometheus>
  AuthType Basic
  AuthName "Wrapper auth"
  AuthBasicProvider file
  AuthUserFile "/path/to/file"
  Require valid-user
</Location>
```

Last, let Prometheus know where to find itself to gather metrics on itself.
``` text /etc/prometheus/prometheus/yml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    scrape_timeout: 10s

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    metrics_path: /prometheus/metrics # I changed this
    target_groups:
      - targets: ['localhost:9090']
```



The web URL parameter let Prometheus know to redirect properly. Hit the URL to impress your project managers. ![zoom](/images/prometheus_graph.png)
