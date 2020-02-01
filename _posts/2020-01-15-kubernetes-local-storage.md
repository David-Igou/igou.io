---
layout: post
title: Kubernetes Local Storage
date: 2020-01-15
categories: [storage, bare metal, flash, fast, zoom]
---

Here is how to start-to-finish use local storage on a kubernetes pod.

Local storage simply means that you create a PV with a path parameter that is a local directory on the host filesystem. You do not hand Kubernetes the device itself, meaning you will have to partition and format the disk on your own from the host machine. In this example, it is a flash drive I plugged into my Raspberry Pi 4 running Prometheus.

```bash
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    1 119.5G  0 disk 
└─sda1        8:1    1 119.5G  0 part 
mmcblk0     179:0    0  59.5G  0 disk 
├─mmcblk0p1 179:1    0   256M  0 part /boot
└─mmcblk0p2 179:2    0  59.2G  0 part /
$ sudo fdisk /dev/sda
[ ... ]
$ sync
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    1 119.5G  0 disk 
├─sda1        8:1    1    80G  0 part 
└─sda2        8:2    1     2G  0 part 
mmcblk0     179:0    0  59.5G  0 disk 
├─mmcblk0p1 179:1    0   256M  0 part /boot
└─mmcblk0p2 179:2    0  59.2G  0 part /
$ mkfs.ext4 /dev/sda1
$ mkfs.ext4 /dev/sda2
```

Add entries to fstab:

```bash
/dev/sda1 /mnt/prometheus ext4 defaults 0 0
/dev/sda2 /mnt/mediawikidb ext4 defaults 0 0
```

Do a `mount -a` to make sure everything mounts fine. That's all you have to do from the node.

Now we create a PV with this path, and add it to our Prometheus custom resource:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-persistent-local-storage
  namespace: monitoring
  labels:
    app: prometheus-persistent-local-storage
spec:
  capacity:
    storage: 80Gi
  accessModes:
  - ReadWriteOnce
  local:
    path: /mnt/prometheus
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - pi-worker
```

Update the Prometheus object:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  [...]
  storage:
    volumeClaimTemplate:
      spec:
        selector:
          matchLabels:
            app: prometheus-persistent-local-storage
        resources:
          requests:
            storage: 80Gi
  tolerations:
  - effect: NoSchedule
    key: arm64
    operator: Equal
    value: "true"
```


Check your output and, success:

```shell
NAMESPACE    NAME                                               STATUS   VOLUME                                CAPACITY   ACCESS MODES   STORAGECLASS   AGE
monitoring   prometheus-prometheus-db-prometheus-prometheus-0   Bound    prometheus-persistent-local-storage   80Gi       RWO                           26m
```

