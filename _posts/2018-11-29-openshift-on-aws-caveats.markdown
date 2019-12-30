---
layout: post
title: "Openshift on AWS caveats"
date: 2018-11-29 14:15:45 -0500
comments: true
categories:
---

Cloud versus on-premises based Openshift deployments have their own unique set of challenges. From a consulting perspective, I generally view cloud as easier in terms of orchestration, but with the possibility of deeper technical issues.

The main challenges people seem to face with OCP on AWS are integration with the cloud plugin, registry storage, DNS, and successfully managing the AWS and Openshift layers in harmony:

![Openshift on AWS architecture](/images/OCP_AWS.png)

<!-- more -->


<br>
## Kubernetes Cloud Provider Plugin
***
*Don't talk to me about Azure*
<br>

[Cloud Provider](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#aws) plugins allow for Kubernetes to integrate with the platform hosting it. The general objective of these plugins are to add features and increase reliability. At the time of writing this, the AWS Kubernetes plugin adds two features: Creating [Elastic Load Balancers (ELBs)](https://aws.amazon.com/elasticloadbalancing/) and dynamic storage (If you create a PV in Kubernetes, requests a disk with that amount of storage and attaches it) via [Elastic Block Storage (EBS).](https://aws.amazon.com/ebs/)

This plugin is currently pretty underutilized, but integration is still recommended because of features that are planned for the future. The use for provisioning ELBs is nullified by the [Openshift Router](https://docs.openshift.com/container-platform/3.11/install_config/router/index.html).

<br>
## Storage, Stateful Applications, limitations.
***
*Stateless apps are easy*
<br>

Elastic *Block* Volumes are block devices. They are not shared storage and are bound to their respective Availability Zones. These limitations need to be kept in mind. The first thing this effects is the internal docker registry if there are multiple replicas of the pod. The recommended work around is to use an [S3 Bucket](https://aws.amazon.com/s3/) as registry storage. This practice has pretty solid performance, so even if you have another storage solution in place for OCP on AWS, this is still the recommended practice.

To escape the possible limitations of EBS, you could use NFS (not recommended for anything significant, but fine in a lab) or something more reliable like [Openshift Container Storage](https://redhatstorage.redhat.com/2018/09/18/running-openshift-container-storage-3-10-with-red-hat-openshift-container-platform-3-10/) (Containerized or external)

<br>

## DNS
***
*The vast majority of installs in new environments you will run into DNS issues. Cloud providers are no different.*
<br>

DNS is so painful for users new to OCP/AWS to troubleshoot. Especially in environments with deviations from standard procedure.

Most guidelines I see online always assume [Route53](https://aws.amazon.com/route53/) is being utilized for DNS. If you're using [GovCloud](https://aws.amazon.com/govcloud-us/), there is no Route53 available, making problem solving even more interesting. Route53 is easy to manage; branching away from this is where we start running into problems.

Most (Including AWS) Cloud Provider plugins require the Kubernetes `NodeName` to match whatever the cloud provider has those node registered as. In Amazon, this is often `ip-x-y-z-q.ec2.internal`. Most people don't care for this because `oc get nodes` isn't quite as pretty as most clusters, and it's harder to keep track of nodes:

```
[root@ip-x-y-z-f~#] oc get nodes
NAME                           STATUS    ROLES     AGE       VERSION
ip-10-240-1-13.ec2.internal   Ready     infra     3d        v1.11.0+d4cacc0
ip-10-240-1-22.ec2.internal   Ready     compute   3d        v1.11.0+d4cacc0
ip-10-240-1-44.ec2.internal   Ready     infra     3d        v1.11.0+d4cacc0
ip-10-240-2-55.ec2.internal    Ready     compute   3d        v1.11.0+d4cacc0
ip-10-240-2-66.ec2.internal   Ready     master    3d        v1.11.0+d4cacc0
ip-10-240-3-23.ec2.internal   Ready     compute   3d        v1.11.0+d4cacc0
ip-10-240-3-15.ec2.internal    Ready     master    3d        v1.11.0+d4cacc0
ip-10-240-3-38.ec2.internal    Ready     master    3d        v1.11.0+d4cacc0
ip-10-240-3-61.ec2.internal    Ready     infra     3d        v1.11.0+d4cacc0
```

To check your meta-data hostname:
`curl http://169.254.169.254/latest/meta-data/hostname`

In VPCs the second IP after the network IP is reserved for DNS. `169.254.169.253` is also available, but only returns default values (not usable if custom FQDNs are configured)

So: The installer _needs_ to resolve the nodes via Amazon Private DNS. The hostname needs to be set to what Amazon knows it as. If you use custom DNS, but change the hostname, the control plane will fail to come up because that ID is based off the name in the Ansible inventory. If you change the hostname, to account for this, the cloud provider plugin fail to initialize. If you use _only_ private AWS DNS, the install will fail because the masters cannot verify the install, because that requires successfully resolving the loadbalancer.

There are two solutions to this:

1. Add the private resolutions to your non-amazon DNS.

2. Configure dnsmaq to fallback on the Amazon DNS server for private (ec2.internal) routes.

This is a pretty cool workaround a coworker showed me:

```
# ansible-playbook aws_custom_route_dns.yml -i openshift_inventory

- hosts: all
  become: yes
  tasks:

  - name: Add Amazon hostnames and FQDN to /etc/hosts
    lineinfile:
      line: '{{ ansible_default_ipv4.address }} {{ inventory_hostname }} {{ inventory_hostname_short }} {{ ansible_nodename }} {{ ansible_hostname }}'
      regexp: '^{{ ansible_default_ipv4.address }}'
      state: present
      path: /etc/hosts
      backrefs: yes

  - name: Create ec2 dns file
    lineinfile:
      line: 'server=/ec2.internal/169.254.169.253'
      state: present
      path: /etc/dnsmasq.d/aws-dns.conf
      create: true
      owner: root
      group: root
      mode: 0644
    notify: restart_dnsmasq_service

  handlers:
  - name: restart_dnsmasq_service
    service:
      name: dnsmasq
      state: restarted
```
TL;DR:

![Openshift on AWS architecture](/images/OCP_AWS_DNS_Flowchart.png)

<br>
## Registry Storage via S3 Bucket
***
*This feels weird, but it's pretty cool*
<br>

This is supported out of the box and can be stood up automatically via the Openshift installer, provided the S3 exists and you provide the key or have the correct IAM roles in place
```
[OSEv3:vars]
# AWS Registry Configuration
openshift_hosted_manage_registry=true
openshift_hosted_registry_storage_kind=object
openshift_hosted_registry_storage_provider=s3
openshift_hosted_registry_storage_s3_accesskey=AKIAJ6VLREDHATSPBUA # Delete this line if using IAM Roles
openshift_hosted_registry_storage_s3_secretkey=g/8PmTYDQVGssFWWFvfawHpDbZyGkjGNZhbWQpjH # Delete this line if using IAM Roles
openshift_hosted_registry_storage_s3_bucket=openshift-registry-storage
openshift_hosted_registry_storage_s3_region=us-east-1
openshift_hosted_registry_storage_s3_chunksize=26214400
openshift_hosted_registry_storage_s3_rootdirectory=/registry
openshift_hosted_registry_pullthrough=true
openshift_hosted_registry_acceptschema2=true
openshift_hosted_registry_enforcequota=true
openshift_hosted_registry_replicas=3
```

The generated storage section of the registry configuration looks like this:
```
storage:
  delete:
    enabled: true
  cache:
    blobdescriptor: inmemory
  s3:
    accesskey: AKLOLOMGBBQSPBUA
    secretkey: g/8PmTYDQVGssFWWFvfawdaleislongkjGNZhbWQpjH
    region: us-east-1
    bucket: openshift-registry-storage
    encrypt: False
    secure: true
    v4auth: true
    rootdirectory: /registry
    chunksize: "26214400"
```

This is kind of confusing on the Kubernetes side because this stored as a secret. `oc describe dc docker-registry -n default` gives no insight that S3 storage is being used (It shows EmptyDir) The only way to confirm it using kubectl/oc is:

```
oc get secret registry-config \
    -o jsonpath='{.data.config\.yml}' -n default | base64 -d
```

Or you can just view your bucket via the AWS console and you'll see the registry files show up in /registry.

<br>
## IAM Roles
***
IAM roles allow/deny access to AWS resources. In this context, we use IAM roles to grant Kubernetes the permission to request EBS volumes, and connect to the S3 registry.

The role to connect to the registry, attach to the Infra Nodes:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": "arn:aws:s3:::S3_BUCKET_NAME"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListMultipartUploadParts",
        "s3:AbortMultipartUpload"
      ],
      "Resource": "arn:aws:s3:::S3_BUCKET_NAME/*"
    }
  ]
}
```

For the cloud provider plugin, attach this role to Masters:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ec2:DescribeVolume*",
                "ec2:CreateVolume",
                "ec2:CreateTags",
                "ec2:DescribeInstances",
                "ec2:AttachVolume",
                "ec2:DetachVolume",
                "ec2:DeleteVolume",
                "ec2:DescribeSubnets",
                "ec2:CreateSecurityGroup",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeRouteTables",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupIngress",
                "elasticloadbalancing:DescribeTags",
                "elasticloadbalancing:CreateLoadBalancerListeners",
                "elasticloadbalancing:ConfigureHealthCheck",
                "elasticloadbalancing:DeleteLoadBalancerListeners",
                "elasticloadbalancing:RegisterInstancesWithLoadBalancer",
                "elasticloadbalancing:DescribeLoadBalancers",
                "elasticloadbalancing:CreateLoadBalancer",
                "elasticloadbalancing:DeleteLoadBalancer",
                "elasticloadbalancing:ModifyLoadBalancerAttributes",
                "elasticloadbalancing:DescribeLoadBalancerAttributes"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "1"
        }
    ]
}
```

All other nodes need:


```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ec2:DescribeInstances",
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "1"
        }
    ]
}
```

### Implementation Knowledge Gap
***
*There are tons of well written Ansible Playbooks that build all of the infrastructure from scratch. You just give them a key and they work. They assume 100% AWS components, are not flexible, and could be depreciated overnight.*

The largest challenge we face with the Operations side of cloud provider hosted instances of Openshift are knowledge gaps sustained by how fast and how many directions things can change. It is crucial to be able to effectively react and adapt to changes that could come to Openshift, Kubernetes, AWS, or your organization's architecture.
