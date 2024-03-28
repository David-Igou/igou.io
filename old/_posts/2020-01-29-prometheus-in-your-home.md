---
layout: post
title: Prometheus in Your Home
date: 2020-01-29
categories: [Prometheus, monitoring, labs, raspberry pi]
---

All metrics of nodes and services in my home lab are scraped by Prometheus. Here are some challenges I ran into, and how I worked around them.

# Why use Prometheus in your Homelab

I'm approaching ~15 nodes in my home and Prometheus is a great way to get insight on my resource usage and performance. I have alerts configured for when things go down (or I break things), and I can get visibility on what other users are doing in my cluster. There are tons of useful exporters I consume - I have blackbox exporters that check if my site is up, have a temperature exporter for my Raspberry Pi's, and I get utilization metrics from my file servers.

[I keep some example configurations here.](https://github.com/David-Igou/kubernetes-lab-manifests/tree/master/monitoring)

# Trial #1: Cheap ec2's

The shortest path to getting a running instance in the beginning for me was doing a `kubectl create -f bundle.yaml -n monitoring` from the official [prometheus-operator](https://github.com/coreos/prometheus-operator). My worker nodes were originally t2.nanos, and I allocated storage via the ebs-csi driver. Originally this appeared fine and scraping a couple node-exporters and some small bundled metrics endpoints didn't seem to cause issues. 

Eventually, I noticed the memory usage would slowly become too much for the instance size. The CPU spikes from saving blocks to the TSDB also were too much, and my node would become unresponsive. Eventually, the operator would reschedule the pod on another node and the cycle would repeat where it slowly deathballed through my cluster. Upgrading to t2.micros did not fix this.

Setting limits sort of helped, but there were gaps in metrics from Kubernetes killing the pod when it eventually hit said limit. Limiting scrape intervals and having a node that I _only_ scheduled Prometheus on would only delay the inevitable. I also tired switching to [k3OS](https://github.com/rancher/k3os) to reduce the OS footprint, but to no avail. It became clear I wasn't going to see success on an EC2

Recently I was able to pinpoint some performance issues I experienced on cert-manager and the ebs-csi controllers. These projects are still Alpha status and things like that are to be expected. If you're reading this in 2021, this might now be possible.

# Trial #2: Unused hardware in my office

![k3s_pis](/images/k3s_pis_censored.jpg)

When it became clear I needed more power than I was willing to pay AWS bills for, I built a [VPN Gateway](https://igou.io/blog/20200107-aws-vpc-vpn-gateway/) into my VPC, added my Raspberry Pis to the cluster, and created a new Prometheus object backed by NFS storage. I used images from [this Prometheus ARM project.](https://github.com/carlosedp/prometheus-ARM)

This worked for a bit, but I ran into more performance issues with NFS. Eventually the database got corrupted, and I saw more performance issues linked to [this issue](https://github.com/prometheus/prometheus/issues/4392). The cause was related to the OS running 32 bit.

So I needed a 64 Bit OS, and better storage performance..

# Trial #3: This is getting ridiculous

![Science](/images/science.jpg)

So here we are at 4x the memory we started at on an aarch64 CPU. The Pi 4's cooling issues are a real thing, so the sink isn't entirely a joke.

Out of the box, Raspbian does not boot a 64bit kernel, so we will need to enable it.

```bash
rpi-update
echo 'arm_64bit=1' >> /boot/config.txt
```
Reboot and throw a `uname -a`

```
Linux pi-x 4.19.97-v8+ #1293 SMP PREEMPT Wed Jan 22 17:22:12 GMT 2020 aarch64 GNU/Linux
```

For storage, I used local storage via the fastest USB 3.0 flash drive I could find. I have a [previous blog](https://igou.io/blog/20200115-kubernetes-local-storage/) on configuring a PV backed by it. 

# Conclusions

When working with constraints, innovation truly shines.
