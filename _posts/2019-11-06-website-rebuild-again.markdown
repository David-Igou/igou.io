---
layout: post
title: "Buildin this site again .. again"
date: 2019-11-06 20:38:38 -0500
comments: true
categories: 
---


My website started as an Ubuntu server that I managed via Ansible. Early on I actually generated blogs using `sed` but I quickly moved onto Octopress. I hacked random features into my site and eventually it became littered in cronjobs and junk.

I've decided to just use Jekyll, but run the site in a container.

I break things a lot so I wanted to make this easy to rebuild. Oh, and cheap.

## k3s

[k3s](https://k3s.io/) has a great topology map about how things work under the OS level. I used terraform to build the infrastructure, and Ansible to handle everything under the OS. The only prerequisite currently is an elastic IP you add to `variables.tf` - I did this so I could tear down/up the infrastructure and my wildcard dns never broke. 


## k3s-ansible

[Repo](https://github.com/David-Igou/k3s-aws-ansible)

Separate from the terraform, because I'm eventually going to run this against my Raspberry Pi clusters.

It's incredibly easy to deploy k3s itself:

```yaml
---
- hosts: k3s_master
  remote_user: ubuntu
  become: True
  gather_facts: True

  tasks:
    - name: Install on masters
      shell: "curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC=\"server --bind-address 0.0.0.0\" sh -"

    - name: Get token from master
      shell: "cat /var/lib/rancher/k3s/server/node-token"
      register: k3s_node_token

- hosts: k3s_worker
  remote_user: ubuntu
  become: True
  gather_facts: True

  tasks:
    - set_fact:
        k3s_master_host: "{{ groups['k3s_master'][0] }}"

    - set_fact:
        k3s_master_token: "{{ hostvars[k3s_master_host]['k3s_node_token'].stdout }}"

    - name: Connect to masters
      shell: "curl -sfL https://get.k3s.io | K3S_URL=https://{{ groups['k3s_master'][0] }}:6443 K3S_TOKEN={{ k3s_master_token }} sh -"
```

Is there a cleaner way to do this than shell modules? Probably. But if I attempted that to start out, I'd of burned out the first week of this and accomplished nothing.

```
./terraform-inventory --inventory . > inventory.ini #There seems to be a bug in terraform-inventory right now breaking taking the script in as the inventory
ansible-playbook playbooks/full.yml 
export KUBECONFIG=~/terraform-k3s/creds/k3s.yaml 
kubectl get pods -A
```

```
NAMESPACE              NAME                                          READY   STATUS      RESTARTS   AGE
kube-system            local-path-provisioner-58fb86bdfd-jrhvf       1/1     Running     0          164m
kube-system            coredns-57d8bbb86-nwlcc                       1/1     Running     0          164m
kube-system            svclb-traefik-h6r2x                           3/3     Running     0          164m
kube-system            svclb-traefik-kk5zc                           3/3     Running     0          164m
kube-system            svclb-traefik-9qtz5                           3/3     Running     0          164m
kube-system            svclb-traefik-wsm79                           3/3     Running     0          164m
kube-system            helm-install-traefik-9tsnf                    0/1     Completed   0          164m
kube-system            helm-install-traefik-cb8wj                    0/1     Completed   0          163m
kube-system            traefik-5db488975-6f7mm                       1/1     Running     0          163m
kubernetes-dashboard   kubernetes-metrics-scraper-6b97c6d857-xn4br   1/1     Running     0          162m
kubernetes-dashboard   kubernetes-dashboard-bf855c94d-w97lm          1/1     Running     0          162m
cert-manager           cert-manager-cainjector-576978ffc8-rxhcp      1/1     Running     0          162m
cert-manager           cert-manager-69b4f77ffc-jxkgz                 1/1     Running     0          162m
cert-manager           cert-manager-webhook-c67fbc858-88292          1/1     Running     2          162m
```

It's beautiful.

The entire deployment takes about 5 minutes on 4 t2.micros.

K3s ships with [Traefik](https://traefik.io), and a local storage provisioner. Local storage isn't very useful here, getting the EBS plugin working is on the checklist down the road. I added [cert-manager](https://github.com/jetstack/cert-manager) a tool that unexpectedly blew my mind. How easy handling certificates has become is incredible.

## Deploying the dashboard using cert-manager and traefik ingress

I am absolutely terrified of man in the middle attacks afflicting my static website. Let us make sure we are safe.

```yaml
- hosts: k3s_master
  become: True
  gather_facts: True

  tasks:
    - name: Install cert-manager crds
      shell: "k3s kubectl create namespace cert-manager"

    - name: Install cert-manager
      shell: "kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml --validate=false"
```

Create this manifest with your email:
```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: heeroyuy@wingzero.net
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    # An empty 'selector' means that this solver matches all domains
    - selector: {}
      http01:
        ingress: {}
```

Okay, I want a secure ~route~ ingress for the dashboard. Let's register a certificate for the url we want to use. I have a wildcard DNS entry for `*.igou.io` going to the master so it just has to be within that subdomain.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: dashboard-certificate
  namespace: kubernetes-dashboard
spec:
  secretName: kubernetes-dashboard-acme-certificate
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: dashboard.igou.io
  dnsNames:
  - dashboard.igou.io
```

To check if it got made correctly:

```
$ kubectl describe cert -n kubernetes-dashboard
Name:         dashboard-certificate
Namespace:    kubernetes-dashboard
[...]
Spec:
  Common Name:  dashboard.igou.io
  Dns Names:
    dashboard.igou.io
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       letsencrypt-staging
  Secret Name:  kubernetes-dashboard-acme-certificate
Status:
  Conditions:
    Last Transition Time:  2019-11-07T00:47:13Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2020-02-04T23:47:12Z
Events:                    <none>
```

If this hangs for like 5 minutes, check the logs via `$ kubectl logs cert-manager-xyzbbq -n cert-manage`

Don't do this over and over against LetsEncrypt's PROD testing your automation like I did. You can only request 5 certificates for a domain per week. 

To make the ~route~ ingress:

```
$ kubectl get svc kubernetes-dashboard -n kubernetes-dashboard
NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   ClusterIP   10.43.48.74   <none>        443/TCP   3h35m
```

Okay. The service is listening on 443. Let's make an ingress:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/protocol: https
    ingress.kubernetes.io/redirect-entry-point: https
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  rules:
  - host: dashboard.igou.io
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
  tls:
  - secretName: kubernetes-dashboard-acme-certificate
```

This should work. I'm still working on figuring out whitelists, to make this a little more secure.
